# QEMU

> QEMU does not have a high level design description document - only the source code tells the full story. (From QEMU wiki)

QEMU 中已经有大量 API 的文档，主要位于 `docs` 目录下。

## x86 复现

启动 Linux 遇到问题，和 xcd 交流后发现可能 Qemu 和 Linux 需要切换到合适的分支上才能正常工作，经过尝试后，目前采用的分支为：

- [QEMU](https://github.com/OS-F-4/qemu-uintr/commit/8077f13cc8d37d229ced48755084563ec94b94c6)
- [Linux](https://github.com/OS-F-4/uintr-linux-kernel/commit/6015080e9ea64f9304f17f610d736cb1ed52924f)

已有文档非常详细，主要分为以下几个步骤：

- [编译 QEMU](https://github.com/OS-F-4/usr-intr/blob/main/ppt/%E5%B1%95%E7%A4%BA%E6%96%87%E6%A1%A3/qemu.md)
- [编译 Linux](https://github.com/OS-F-4/usr-intr/blob/main/ppt/%E5%B1%95%E7%A4%BA%E6%96%87%E6%A1%A3/linux-kernel.md)
- [构建文件系统](https://github.com/OS-F-4/usr-intr/blob/main/ppt/how-to-build-a-ubuntu-rootfs.md)

目前可运行的测例：Linux 自带的 uipi_sample

阅读 Linux 关于 UINTR 系统调用的实现可以注意到，接收方在满足以下条件时可以响应中断：

- 正在被当前核调度，直接进入注册好的用户态中断处理函数
- 通过系统调用 `sys_uintr_wait` 让权后，接收方进程等待被内核调度后再进入中断处理函数
- 时间片用尽后被内核重新放入调度队列，接收方等待重新被内核调度后再进入中断处理函数

```c
int uintr_receiver_wait(void)
{
    struct uintr_upid_ctx *upid_ctx;
    unsigned long flags;

    if (!is_uintr_receiver(current))
        return -EOPNOTSUPP;

    upid_ctx = current->thread.ui_recv->upid_ctx;
    // 发送方将中断发给内核
    upid_ctx->upid->nc.nv = UINTR_KERNEL_VECTOR;
    // 接收方进入 waiting 状态
    upid_ctx->waiting = true;
    // 交给统一的调度队列 uintr_wait_list 进行管理，内核收到用户态中断后会遍历队列进行处理
    spin_lock_irqsave(&uintr_wait_lock, flags);
    list_add(&upid_ctx->node, &uintr_wait_list);
    spin_unlock_irqrestore(&uintr_wait_lock, flags);
    // 修改当前 task 的状态为 INTERRUPTIBLE
    set_current_state(TASK_INTERRUPTIBLE);
    // 运行下一个 task
    schedule();

    return -EINTR;
}
```

目前定义了两种 User Interrupt Notification：

- `UINTR_NOTIFICATION_VECTOR`: 0xec
- `UINTR_KERNEL_VECTOR`: 0xeb

关于 `UINTR_KERNEL_VECTOR` 的用途，注意到：

```c
/*
 * Handler for UINTR_KERNEL_VECTOR.
 */
DEFINE_IDTENTRY_SYSVEC(sysvec_uintr_kernel_notification)
{
    /* TODO: Add entry-exit tracepoints */
    ack_APIC_irq();
    inc_irq_stat(uintr_kernel_notifications);

    uintr_wake_up_process();
}
/*
 * Runs in interrupt context.
 * Scan through all UPIDs to check if any interrupt is on going.
 */
void uintr_wake_up_process(void)
{
    struct uintr_upid_ctx *upid_ctx, *tmp;
    unsigned long flags;

    // 遍历 uintr_wait_list
    spin_lock_irqsave(&uintr_wait_lock, flags);
    list_for_each_entry_safe(upid_ctx, tmp, &uintr_wait_list, node) {
        if (test_bit(UPID_ON, (unsigned long *)&upid_ctx->upid->nc.status)) {
            set_bit(UPID_SN, (unsigned long *)&upid_ctx->upid->nc.status);
            upid_ctx->upid->nc.nv = UINTR_NOTIFICATION_VECTOR;
            upid_ctx->waiting = false;
            wake_up_process(upid_ctx->task);
            list_del(&upid_ctx->node);
        }
    }
    spin_unlock_irqrestore(&uintr_wait_lock, flags);
}
```

每当 task 被重新调度并返回 user space 前，执行以下函数：

```c
void switch_uintr_return(void)
{
    struct uintr_upid *upid;
    u64 misc_msr;

    if (is_uintr_receiver(current)) {
        WARN_ON_ONCE(test_thread_flag(TIF_NEED_FPU_LOAD));

        /* Modify only the relevant bits of the MISC MSR */
        rdmsrl(MSR_IA32_UINTR_MISC, misc_msr);
        if (!(misc_msr & GENMASK_ULL(39, 32))) {
            misc_msr |= (u64)UINTR_NOTIFICATION_VECTOR << 32;
            wrmsrl(MSR_IA32_UINTR_MISC, misc_msr);
        }

        /*
        因为此时 task 被重新调度，需要更新 ndst 对应的 APIC ID
        同时需要清空 SN 来允许接收中断
        */
        upid = current->thread.ui_recv->upid_ctx->upid;
        upid->nc.ndst = cpu_to_ndst(smp_processor_id());
        clear_bit(UPID_SN, (unsigned long *)&upid->nc.status);

        /*
        UPID_SN 已经被清空，此时可以接收新的中断
        为了让 task 能知道自己在等待的过程中收到了发送方发过来的中断，直接调用 send_IPI_self 触发硬件处理流程： User-Interrupt Notification Identification 和 User-Interrupt Notification Processing；
        另一种办法是软件进行处理来触发中断，即清空 UPID.PUIR 和 写入 UIRR 寄存器，代码注释提示软件触发中断需要处理和硬件修改 UPID 之间的竞争；
        根据 intel 文档的描述，以下任何一种情况都可以触发 recognition of pending user interrupt:
        1. 写入 IA32_UINTR_RR_MSR
        4. User-interrupt notification processing: 也就是收到中断后硬件处理
        */
        if (READ_ONCE(upid->puir))
            apic->send_IPI_self(UINTR_NOTIFICATION_VECTOR);
    }
}

```

在 manpages 中注意到这样一段描述：

```txt
A receiver can choose to share the same uintr_fd with multiple senders.
Since an interrupt with the same vector number would be delivered,  the
receiver  would  need  to  use  other  mechanisms to identify the exact
source of the interrupt.
```

大致意思是说不同的 sender 可能是拿同一个 fd 注册的 uitte ，需要 receiver 应用其他方法来区分 sender 。
处于同一个中断优先级的 sender 彼此之间无法通过已有机制加以区分，那么这种区分是否是必要的？

## RISC-V 实现

QEMU RISC-V 的实现架构和 x86 的有很大不同。

编译：`./configure --target-list=riscv64-softmmu && make`。

关于 GDB 调试，之前写 os 的调试方法可以直接在这里应用。

主要关注几个方面：

- 指令翻译
- CPU 状态
- 内存读写
- 核间中断

主要参考 [QEMU RISC-V N扩展](https://github.com/duskmoon314/qemu) 的相关实现。

### 指令翻译（Code Generation）

QEMU 指令翻译的过程：`Guest Instructions -> TCG (Tiny Code Generator) ops -> Host Instructions`

主要关注 `TCG Frontend`，也就是 QEMU 将 Guest 的指令翻译成 TCG 中间码的部分。

代码核心部分位于 `traget/riscv/translate.c`，开头部分定义了 `DisasContext`，该文件给出了一些常用的工具函数。

RISC-V 指令集扩展模块化，QEMU 在翻译指令的时候也按照这个逻辑处理。

QEMU 有一套自己的翻译机制，类似于 CPU 的 decoder，需要修改模式串来帮助 QEMU 在执行到指令时进行匹配并调用 helper 。

以 uret 指令为例，在 `target/riscv/insn_trans` 目录中有各种指令的翻译过程，主要用来将指令解析的结果（寄存器，立即数等）传递给 QEMU 内部的函数，将 guest 的指令拆解为 host 的指令来模拟目标指令的功能。

对于 uret 等指令来说，指令的执行涉及到较多 CPU 状态的变化，会对 pc ，CSR 等产生影响，因此需要利用封装后的 helper 函数来辅助完成。 helper 的定义位于 `target/riscv/helper.h`，通过宏定义 `DEF_HELPER_x` 来声明 helper 函数，例如：

```c
DEF_HELPER_1(uret, tl, env)
DEF_HELPER_4(csrrw, tl, env, int, tl, tl)
```

其中第一个参数对应 `helper_<name>` 中的 name ，第二参数代表函数的返回值，`tl` 表示 `target_ulong` ，后面的参数都是 helper 函数传入的参数。有了以上的参考，我们可以定义其他 helper：

```c
DEF_HELPER_2(uipi_write, void, env, tl)

void helper_uipi_write(CPURISCVState *env, target_ulong data) {
    if (uipi_enabled(env, env->suirs)) {
        target_ulong uintc_addr = UINTC_HIGH(env->suicfg, SUIRS_INDEX(env->suirs));
        qemu_log("UIPI WRITE %lx %lx\n", uintc_addr, data);
    }
}
```

第四个参数传入 `UIPI WRITE` 写入的 `data`。

考虑到 UIPI 指令包含读内存和写回寄存器的功能，且指令需要绕过地址翻译直接用物理地址对内存和外设进行访问，实现 `helper_uipi` 函数：

```c
target_ulong helper_uipi(CPURISCVState *env, int op, target_ulong src) {
    if (op == UIPI_SEND) {
        // Sender
        if (uipi_enabled(env, env->suist)) {
            target_ulong uist_end = SUIST_BASE(env->suist) + SUIST_SIZE(env->suist);
            target_ulong uiste_addr = SUIST_BASE(env->suist) + (src << 6);
            if (uiste_addr < uist_end) {
                uint64_t uiste;
                cpu_physical_memory_read(uiste_addr, &uiste, 16);
                uint64_t uirs_index = (uiste >> 48) & 0xffff;
                uint64_t sender_vec = (uiste >> 16) & 0xffff;
                if (uiste & 0x1) {
                    uint64_t uintc_addr = UINTC_REG_SEND(env->suicfg, uirs_index);
                    cpu_physical_memory_write(uintc_addr, &sender_vec, 8);
                }
            }
        }
    } else if (uipi_enabled(env, env->suirs)) {
        // Receiver
        if (env->xl == MXL_RV64) {
            uint64_t data, addr;
            switch (op) {
                case UIPI_READ:
                    addr = UINTC_REG_HIGH(env->suicfg, SUIRS_INDEX(env->suirs));
                    cpu_physical_memory_read(addr, &data, 8);
                    return data;
                case UIPI_WRITE:
                    addr = UINTC_REG_HIGH(env->suicfg, SUIRS_INDEX(env->suirs));
                    cpu_physical_memory_write(addr, &src, 8);
                    break;
                case UIPI_ACTIVATE:
                    data = 0x1;
                    addr = UINTC_REG_ACTIVE(env->suicfg, SUIRS_INDEX(env->suirs));
                    cpu_physical_memory_write(addr, &data, 8);
                    break;
                case UIPI_DEACTIVATE:
                    data = 0x0;
                    addr = UINTC_REG_ACTIVE(env->suicfg, SUIRS_INDEX(env->suirs));
                    cpu_physical_memory_write(addr, &data, 8);
                    break;
            }
        } else if (env->xl == MXL_RV32) {
            // TODO
        }
    }
    return 0;
}

static bool trans_uipi(DisasContext *ctx, arg_uipi *a)
{

    if (has_ext(ctx, RVN)) {
        TCGv data = temp_new(ctx);
        TCGv src = get_gpr(ctx, a->rs1, EXT_NONE);
        // 注意 opcode 要从 imm 中获得
        TCGv_i32 op = tcg_constant_i32(a->imm);
        gen_helper_uipi(data, cpu_env, op, src);
        if (a->imm == UIPI_READ) {
            // 这里还需要将读到的结果写回到寄存器
            TCGv dest = dest_gpr(ctx, a->rd);
            tcg_gen_mov_tl(dest, data);
        }
        return true;
    }
    return false;
}
```

目前只实现并测试了 RV64 下的功能。


### CPU 状态（Target Emulation）

中断异常、CSR、CSR bits 等定义位于 `target/riscv/cpu_bits.h` 。

`struct CPUArchState`：CPU 状态结构体位于 `target/riscv/cpu.h`。这个结构同时考虑了 RV32、RV64、RV128 的情况，具体可以参考结构体内部的注释。与在 FPGA 上开发硬件类似，这些寄存器都是 CPU 运行时必要的状态。包括但不限于：

- PC
- 整数、浮点寄存器堆
- CSR 特权寄存器，有些寄存器是 M 态和 S 态复用的，例如 `mstatus`、 `mip` 等
- PMP 寄存器堆 `pmp_table_t`
- `QEMUTimer *timer`：计时器
- 通过 `kernel_addr`、`fdt_addr` 等从指定位置加载镜像

`struct RISCVCPUConfig`：主要包含各种 CPU 特性的开关，包括但不限于：

- 扩展类型
- 是否开启 MMU
- 是否开启 PMP

`struct ArchCPU`：对 CPU 的进一步封装。

```c
struct ArchCPU {
    // 全局的 CPU 状态，类似于继承关系，可以暂时不考虑
    CPUState parent_obj;
    CPUNegativeOffsetState neg;
    // 主要修改这一部分内容
    CPURISCVState env;
    char *dyn_csr_xml;
    char *dyn_vreg_xml;
    // 可以加入我们自己设计的扩展选项
    RISCVCPUConfig cfg;
};
```

CSR 的修改主要位于 csr.c 。

```c
riscv_csr_operations csr_ops[CSR_TABLE_SIZE];

typedef struct {
    const char *name;
    riscv_csr_predicate_fn predicate;
    riscv_csr_read_fn read;
    riscv_csr_write_fn write;
    riscv_csr_op_fn op;
    riscv_csr_read128_fn read128;
    riscv_csr_write128_fn write128;
} riscv_csr_operation s;
```

这个结构体给出了针对 CSR 寄存器进行的操作。

读写 csr 寄存器定义在 `target/riscv/cpu.h` ，实现在 `target/riscv/csr.c`。

```c
/*
 * riscv_csrrw - read and/or update control and status register
 *
 * csrr   <->  riscv_csrrw(env, csrno, ret_value, 0, 0);
 * csrrw  <->  riscv_csrrw(env, csrno, ret_value, value, -1);
 * csrrs  <->  riscv_csrrw(env, csrno, ret_value, -1, value);
 * csrrc  <->  riscv_csrrw(env, csrno, ret_value, 0, value);
 */
RISCVException riscv_csrrw(CPURISCVState *env, int csrno,
                           target_ulong *ret_value,
                           target_ulong new_value, target_ulong write_mask)
{
    RISCVCPU *cpu = env_archcpu(env);

    // 检查是否尝试写入只读 CSR，是否开启 CSR
    // 这个函数最后会返回 csr_ops[csrno].predicate(env, csrno)
    // 12.14：暂时没明白 riscv_csr_predicate_fn 是干什么的，可能是一些指令集相关的检查
    RISCVException ret = riscv_csrrw_check(env, csrno, write_mask, cpu);
    if (ret != RISCV_EXCP_NONE) {
        return ret;
    }

    // 先检查该 CSR 是否存在特殊的 csrrw 处理函数
    // CSR 必须注册 riscv_csr_read_fn
    // 尝试读旧值（可能触发异常），根据注释中描述的参数情况，如果 write mask 不为 0,将新值 mask 运算之后写入（可能触发异常）
    // 根据 csrrw 语义，写入新值，返回旧值
    return riscv_csrrw_do64(env, csrno, ret_value, new_value, write_mask);
}
```

`target/riscv/cpu.h` 文件末尾有一个大表，里面列出了每个 CSR 的操作函数，直接在这里定义即可。

在 `target/riscv/cpu_helper.c` 中有这样一个函数将 pending 的 interrupt 转换成 irq 来触发 CPU 中断处理机制：

```c
static int riscv_cpu_all_pending(CPURISCVState *env) {
    return env->mip & env->mie;
}
/* 
主要关注两个参数 extirq_def_prio 和 iprio
先判断是否开启 AIA ，如果没有开启，则直接计算尾 0， 然后返回 irq
如果开启 AIA ，先计算特权级
    默认拿到 iprio[irq] 索引，若为 0
        如果是外部中断 extirq ，就赋值为 extirq_def_prio，否则根据 default_iprio 表中的信息拿到
按以上的方法，从右向左遍历，过程中获取最合适的 irq 返回处理
*/
static int riscv_cpu_pending_to_irq(CPURISCVState *env,
                                    int extirq, unsigned int extirq_def_prio,
                                    uint64_t pending, uint8_t *iprio);
/*
先获取对应特权态的 status 中断位，确认中断是否开启
然后按特权态由高到低的顺序对 riscv_cpu_all_pending 获得的 pending 进行处理
最后调用 riscv_cpu_pending_to_irq 返回中断请求 irqs
*/
static int riscv_cpu_local_irq_pending(CPURISCVState *env);
```

有关 AIA ，详见 [RISC-V AIA](https://github.com/riscv/riscv-aia)。

CPU 中断异常处理函数位于 `target/riscv/cpu_helper.c` 的最后，这个函数只给出了 M 态和 S 态的中断异常处理，之后在此处加入委托给 U 态的中断异常处理，也就是读写 RISC-V N 扩展中提到的 `ustatus`，`ucause`，`uepc` 等寄存器。

```c
/*
计算 async 来判断是中断还是异常，以及中断或异常委托
根据 env->priv 进一步区分中断异常原因，例如 ECALL 类型
根据不同中断异常原因，设置 CPUArchState 中的状态寄存器，例如 `pc`， `stval`，`sepc` 等
根据 env->priv 设置不同的特权态对应的寄存器，例如需要区分 mstatus 和 sstatus
*/
void riscv_cpu_do_interrupt(CPUState *cs);
```

### 内存读写（Memory Emulation）

在 `target/riscv/cpu_helper.c` 中发现地址翻译的函数：

```c
/* get_physical_address - get the physical address for this virtual address
 *
 * Do a page table walk to obtain the physical address corresponding to a
 * virtual address. Returns 0 if the translation was successful
 *
 * Adapted from Spike's mmu_t::translate and mmu_t::walk
 *
 * @env: CPURISCVState
 * @physical: This will be set to the calculated physical address
 * @prot: The returned protection attributes
 * @addr: The virtual address to be translated
 * @fault_pte_addr: If not NULL, this will be set to fault pte address
 *                  when a error occurs on pte address translation.
 *                  This will already be shifted to match htval.
 * @access_type: The type of MMU access
 * @mmu_idx: Indicates current privilege level
 * @first_stage: Are we in first stage translation?
 *               Second stage is used for hypervisor guest translation
 * @two_stage: Are we going to perform two stage translation
 * @is_debug: Is this access from a debugger or the monitor?
 */
static int get_physical_address(CPURISCVState *env, hwaddr *physical,
                                int *prot, target_ulong addr,
                                target_ulong *fault_pte_addr,
                                int access_type, int mmu_idx,
                                bool first_stage, bool two_stage,
                                bool is_debug) {
/*
two stage 涉及更富杂的虚拟化的内容，也就是两级 MMU ，暂时先不考虑
这里的物理地址映射交给 QEMU 进行管理
先根据 env->satp 拿到页表基址
和内核查页表的逻辑差不多，把虚拟地址切分开，然后一级一级循环查下去
主要的复杂性在于根据 CPU 的配置区分不同的 RV 虚存机制，检查各种标志位的合法性
查询失败返回 TRANSLATE_FAIL
*/                              
}
```

x86 的设计中 `SENDUIPI index` 这个指令会根据 `UITTADDR` 寄存器，索引到内存中 `UITTADDR + index * sizeof(UITTE)` 的位置。这条指令在执行时，会直接拿到地址向内存发起读写请求，不会经过也没必要经过 MMU。

QEMU 内存读写函数 `void cpu_physical_memory_rw(hwaddr addr, void *buf, hwaddr len, bool is_write)` 位于 `softmmu/physmem.c`。

### 核间中断 (Hardware Emulation)

#### RISC-V AIA，ACLINT

有关 MSI (Message Signalled Interrupts):

- 传统发中断 pin-based out-of-band ：外设有单独引脚，独立于数据总线
- MSI：处理器和外设之间存在中断控制器，外设通过数据总线给控制器发送更丰富的中断信息，控制器进行处理后再发给处理器，这些信息可以帮助外设和控制器更好地决策发送中断的时机、目标等。
- 可以增加中断数量
- pin-based interrupt 和 posted-write 之间的竞争问题：PCI 内存控制器可能会推迟写入 DMA，导致处理器收到中断后立即尝试通过 DMA 读取旧的数据，所以中断控制器需要读取 PCI 内存控制器来判读写入是否完成。MSI write 和 DMA write 之间共用总线，所以不会出现这种异步的竞争问题。

AIA （RISC-V Advanced Interrupt Architecture）

官方文档给出了设计目标

> This RISC-V ACLINT specification defines a set of memory mapped devices which provide inter-processor interrupts (IPI) and timer functionalities for each HART on a multi-HART RISC-V platform.

QEMU 中也给出了相关实现，主要在 `hw/intc` 目录下，有关 RISC-V 的两个几个文件 `riscv_aclint.c`, `riscv_aplic.c` 和 `riscv_imsic.c` 。

```c
typedef struct RISCVAclintSwiState {
    /*< private >*/
    SysBusDevice parent_obj;

    /*< public >*/
    MemoryRegion mmio;
    // 起始 hartid
    uint32_t hartid_base;
    // hart 总数量
    uint32_t num_harts;
    uint32_t sswi;
    qemu_irq *soft_irqs;
} RISCVAclintSwiState;
```

CPU 读 SWI 寄存器的函数如下：

```c
static uint64_t riscv_aclint_swi_read(void *opaque, hwaddr addr,
    unsigned size)
{
    RISCVAclintSwiState *swi = opaque;
    // 首先保证读地址在对应 hart 范围内
    if (addr < (swi->num_harts << 2)) {
        // 获取对应 hartid
        size_t hartid = swi->hartid_base + (addr >> 2);
        // 获取对应 CPU 状态
        CPUState *cpu = qemu_get_cpu(hartid);
        CPURISCVState *env = cpu ? cpu->env_ptr : NULL;
        if (!env) {
            qemu_log_mask(LOG_GUEST_ERROR,
                          "aclint-swi: invalid hartid: %zu", hartid);
        } else if ((addr & 0x3) == 0) {
            // 返回 SWI 寄存器状态
            return (swi->sswi) ? 0 : ((env->mip & MIP_MSIP) > 0);
        }
    }

    qemu_log_mask(LOG_UNIMP,
                  "aclint-swi: invalid read: %08x", (uint32_t)addr);
    return 0;
}
```

CPU 写 SWI 寄存器的函数如下：

```c
static void riscv_aclint_swi_write(void *opaque, hwaddr addr, uint64_t value,
        unsigned size)
{
    RISCVAclintSwiState *swi = opaque;
    // 首先保证读地址在对应 hart 范围内
    if (addr < (swi->num_harts << 2)) {
        // 获取对应 hartid
        size_t hartid = swi->hartid_base + (addr >> 2);
        // 获取对应 CPU 状态
        CPUState *cpu = qemu_get_cpu(hartid);
        CPURISCVState *env = cpu ? cpu->env_ptr : NULL;
        if (!env) {
            qemu_log_mask(LOG_GUEST_ERROR,
                          "aclint-swi: invalid hartid: %zu", hartid);
        } else if ((addr & 0x3) == 0) {
            if (value & 0x1) {
                // 触发 IRQ ，调用对应 handler 对中断事务进行处理
                qemu_irq_raise(swi->soft_irqs[hartid - swi->hartid_base]);
            } else {
                if (!swi->sswi) {
                    // 清空 IRQ
                    qemu_irq_lower(swi->soft_irqs[hartid - swi->hartid_base]);
                }
            }
            return;
        }
    }

    qemu_log_mask(LOG_UNIMP,
                  "aclint-swi: invalid write: %08x", (uint32_t)addr);
}
```

QEMU 中每个 memory mapped device 都要对应到 `MemoryRegionOps`：

```c
static const MemoryRegionOps riscv_aclint_swi_ops = {
    .read = riscv_aclint_swi_read,
    .write = riscv_aclint_swi_write,
    .endianness = DEVICE_LITTLE_ENDIAN,
    .valid = {
        .min_access_size = 4,
        .max_access_size = 4
    }
};
```

调用 `memory_region_init_io`后，对该内存区域的读写就会转发给注册后的函数进行处理。

#### QEMU RISC-V VirtIO Board

通过 `-machine virt` 选中，通过 VirtIO 模拟硬件环境，代码位置 `hw/riscv/virt.c`。

从代码中可以看出一系列外设对应的地址：

```c
static const MemMapEntry virt_memmap[] = {
    [VIRT_DEBUG] =        {        0x0,         0x100 },
    [VIRT_MROM] =         {     0x1000,        0xf000 },
    [VIRT_TEST] =         {   0x100000,        0x1000 },
    [VIRT_RTC] =          {   0x101000,        0x1000 },
    [VIRT_CLINT] =        {  0x2000000,       0x10000 },
    [VIRT_ACLINT_SSWI] =  {  0x2F00000,        0x4000 },
    [VIRT_PCIE_PIO] =     {  0x3000000,       0x10000 },
    [VIRT_PLATFORM_BUS] = {  0x4000000,     0x2000000 },
    [VIRT_PLIC] =         {  0xc000000, VIRT_PLIC_SIZE(VIRT_CPUS_MAX * 2) },
    [VIRT_APLIC_M] =      {  0xc000000, APLIC_SIZE(VIRT_CPUS_MAX) },
    [VIRT_APLIC_S] =      {  0xd000000, APLIC_SIZE(VIRT_CPUS_MAX) },
    [VIRT_UART0] =        { 0x10000000,         0x100 },
    [VIRT_VIRTIO] =       { 0x10001000,        0x1000 },
    [VIRT_FW_CFG] =       { 0x10100000,          0x18 },
    [VIRT_FLASH] =        { 0x20000000,     0x4000000 },
    [VIRT_IMSIC_M] =      { 0x24000000, VIRT_IMSIC_MAX_SIZE },
    [VIRT_IMSIC_S] =      { 0x28000000, VIRT_IMSIC_MAX_SIZE },
    [VIRT_PCIE_ECAM] =    { 0x30000000,    0x10000000 },
    [VIRT_PCIE_MMIO] =    { 0x40000000,    0x40000000 },
    [VIRT_DRAM] =         { 0x80000000,           0x0 },
};
```

其中涉及核间中断的外设为 `VIRT_PLIC` ，`VIRT_CLINT` 和 `VIRT_ACLINT_SSWI` ，当然也可以自己注册某些外设。其中可以看到我们比较熟悉的 `VIRT_VIRTIO` 地址映射 `0x10001000, 0x1000` 。

`virt_machine_init` 函数负责初始化 CPU ，外设等。

```c
static void virt_machine_init(MachineState *machine) {
/*
预分配一块内存用于存储设备的地址映射
    MemoryRegion *system_memory = get_system_memory();
根据 CPU 插槽数进行遍历，实际 hart 数量由 smp 参数指定，关于这一部分在 hw/riscv/numa.c：
调用 riscv_aclint_swi_create 建立对 VIRT_ACLINT_SSWI 和 VIRT_CLINT 的映射，并初始化 mtimer
根据 s->aia_type 进行判断，如果是 VIRT_AIA_TYPE_NONE 就调用 virt_create_plic 初始化 PLIC
否则调用 virt_create_aia 初始化 AIA

接下来调用一系列函数进行外设初始化：
virt_create_aia 初始化中断控制器
memory_region_add_subregion 初始化 RAM 和 Boot ROM
sysbus_create_simple 将 VirtIO 外设接入
gpex_pcie_init 初始化 VIRT_PCIE_ECAM 和 VIRT_PCIE_MMIO
serial_mm_init 初始化串口 VIRT_UART0
virt_flash_create, virt_flash_map 初始化 flash

最后根据地址映射创建设备树交给上层应用 create_fdt
*/
}
```

#### UINTC 实现

参考 `riscv_aclint` 实现 **UINTC**。

观察到 aclint，aplic 等设备会调用 `riscv_cpu_claim_interrupts`，用来将中断信号唯一绑定至某中断控制器，为了简化代码实现，直接将 UINTC 连接到 USIP 。

```c
static void riscv_uintc_realize(DeviceState *dev, Error **errp)
{
    int i;
    RISCVUintcState *uintc = RISCV_UINTC(dev);

    memory_region_init_io(&uintc->mmio, OBJECT(dev), &riscv_uintc_ops, uintc,
                          TYPE_RISCV_UINTC, RISCV_UINTC_SIZE);
    sysbus_init_mmio(SYS_BUS_DEVICE(dev), &uintc->mmio);

    uintc->uirs = g_new0(RISCVUintcUIRS, uintc->num_harts);

    /* Create output IRQ lines */
    uintc->soft_irqs = g_new(qemu_irq, uintc->num_harts);
    qdev_init_gpio_out(dev, uintc->soft_irqs, uintc->num_harts);

    for (i = 0; i < uintc->num_harts; i++) {
        RISCVCPU *cpu = RISCV_CPU(qemu_get_cpu(uintc->hartid_base + i));
        
        /* Claim software interrupt bits */
        if (riscv_cpu_claim_interrupts(cpu, MIP_USIP) < 0) {
            error_report("USIP already claimed");
            exit(1);
        }
    }
}
```

`riscv_uintc_realize` 完成 `uirs` 表、中断状态 `qemu_irq` 的内存分配。调用 `memory_region_init_io` 和 `sysbus_init_mmio` 对 UINTC 占用的地址空间进行初始化，绑定读写函数。

```c
DeviceState *riscv_uintc_create(hwaddr addr, uint32_t hartid_base, uint32_t num_harts)
{
    int i;
    DeviceState *dev = qdev_new(TYPE_RISCV_UINTC);

    assert(num_harts <= RISCV_UINTC_MAX_HARTS);
    assert(!(addr & 0x1f));

    qdev_prop_set_uint32(dev, "hartid-base", hartid_base);
    qdev_prop_set_uint32(dev, "num-harts", num_harts);
    sysbus_realize_and_unref(SYS_BUS_DEVICE(dev), &error_fatal);
    sysbus_mmio_map(SYS_BUS_DEVICE(dev), 0, addr);

    for (i = 0; i < num_harts; i++) {
        CPUState *cpu = qemu_get_cpu(hartid_base + i);
        RISCVCPU *rvcpu = RISCV_CPU(cpu);

        qdev_connect_gpio_out(dev, i, qdev_get_gpio_in(DEVICE(rvcpu), IRQ_U_SOFT));
    }

    return dev;
}
```

`riscv_uintc_create` 由外部 `board` 进行调用，负责将 GPIO 端口与 CPU 内部 IRQ_U_SOFT 连接起来，根据外部传入的地址完成地址空间的映射。

关于 IRQ 的注册：

```c
struct IRQState {
    Object parent_obj;

    qemu_irq_handler handler;
    void *opaque;
    int n;
};

typedef struct IRQState *qemu_irq;

typedef void (*qemu_irq_handler)(void *opaque, int n, int level);

void qemu_set_irq(qemu_irq irq, int level)
{
    if (!irq)
        return;

    irq->handler(irq->opaque, irq->n, level);
}

static inline void qemu_irq_raise(qemu_irq irq)
{
    qemu_set_irq(irq, 1);
}

static inline void qemu_irq_lower(qemu_irq irq)
{
    qemu_set_irq(irq, 0);
}
```

定义 `IRQState` 包含不同设备注册的 `qemu_irq_handler` ，中断高电平触发。`qemu_irq_handler` 已经预先在 CPU 中注册，GPIO 的输出引脚只需要将指针指向对应 `IRQState` 实例即可。

```c
static void riscv_cpu_init(Object *obj)
{
    RISCVCPU *cpu = RISCV_CPU(obj);

    cpu_set_cpustate_pointers(cpu);

#ifndef CONFIG_USER_ONLY
    qdev_init_gpio_in(DEVICE(cpu), riscv_cpu_set_irq,
                      IRQ_LOCAL_MAX + IRQ_LOCAL_GUEST_MAX);
#endif /* CONFIG_USER_ONLY */
}
```

在 `virt.c` 中添加 UINTC 之后，发现在 S 态访问会报 Store Fault，查看 OpenSBI 源码和文档后，发现 OpenSBI 存在 Domain 机制：

- 一个 Hart 只能运行在某个 Domain 中
- SBI 调用只能访问或影响同一个 Domain 中的 Hart
- 运行在 S 态或 U 态的 Hart 只能访问 Domain 中分配好的地址空间

默认将 ROOT Domain 分配给所有 Hart， 主要存在两类地址空间：S 态和 U 态不可访问，用于保护 SBI 管理的硬件设备；`order=__riscv_xlen`，S 态和 U 态可以访问的全部地址空间。

运行 QEMU 指定参数 `-bios default` 后，打印关于 Domain 的信息：

```txt
Domain0 Name              : root
Domain0 Boot HART         : 0
Domain0 HARTs             : 0*,1*,2*,3*
Domain0 Region00          : 0x0000000002000000-0x000000000200ffff (I)
Domain0 Region01          : 0x0000000080000000-0x000000008007ffff ()
Domain0 Region02          : 0x0000000000000000-0xffffffffffffffff (R,W,X)
Domain0 Next Address      : 0x0000000080200000
Domain0 Next Arg1         : 0x00000000bf000000
Domain0 Next Mode         : S-mode
Domain0 SysReset          : yes
```

相关函数位于 OpenSBI 的 `sbi_domain_dump` 函数，可以看出 `Domain Region00` 刚好为 `VIRT_CLINT` 占用的地址空间，这一部分不可以被 S 态或 U 态访问。

OpenSBI 通过 `sbi_hart_pmp_configure` 读取 Domain 信息并设置相关寄存器来限制 S 态和 U 态对某些物理地址的访问。

```c
int pmp_set(unsigned int n, unsigned long prot, unsigned long addr,
        unsigned long log2len)
{
    int pmpcfg_csr, pmpcfg_shift, pmpaddr_csr;
    unsigned long cfgmask, pmpcfg;
    unsigned long addrmask, pmpaddr;

    /* check parameters */
    if (n >= PMP_COUNT || log2len > __riscv_xlen || log2len < PMP_SHIFT)
        return SBI_EINVAL;

    /* calculate PMP register and offset */
    #if __riscv_xlen == 32
    pmpcfg_csr   = CSR_PMPCFG0 + (n >> 2);
    pmpcfg_shift = (n & 3) << 3;
    #elif __riscv_xlen == 64
    pmpcfg_csr   = (CSR_PMPCFG0 + (n >> 2)) & ~1;
    pmpcfg_shift = (n & 7) << 3;
    #else
    # error "Unexpected __riscv_xlen"
    #endif
    pmpaddr_csr = CSR_PMPADDR0 + n;

    /* encode PMP config */
    prot &= ~PMP_A;
    prot |= (log2len == PMP_SHIFT) ? PMP_A_NA4 : PMP_A_NAPOT;
    cfgmask = ~(0xffUL << pmpcfg_shift);
    pmpcfg = (csr_read_num(pmpcfg_csr) & cfgmask);
    pmpcfg |= ((prot << pmpcfg_shift) & ~cfgmask);

    /* encode PMP address */
    if (log2len == PMP_SHIFT) {
        pmpaddr = (addr >> PMP_SHIFT);
    } else {
        if (log2len == __riscv_xlen) {
            pmpaddr = -1UL;
        } else {
            addrmask = (1UL << (log2len - PMP_SHIFT)) - 1;
            pmpaddr	= ((addr >> PMP_SHIFT) & ~addrmask);
            pmpaddr |= (addrmask >> 1);
        }
    }

    /* write csrs */
    csr_write_num(pmpaddr_csr, pmpaddr);
    csr_write_num(pmpcfg_csr, pmpcfg);

    return 0;
}

int sbi_hart_pmp_configure(struct sbi_scratch *scratch)
{
    ...
/*
* If permissions are to be enforced for all modes on this
* region, the lock bit should be set.
*/
if (reg->flags & SBI_DOMAIN_MEMREGION_ENF_PERMISSIONS)
    pmp_flags |= PMP_L;

if (reg->flags & SBI_DOMAIN_MEMREGION_SU_READABLE)
    pmp_flags |= PMP_R;
if (reg->flags & SBI_DOMAIN_MEMREGION_SU_WRITABLE)
    pmp_flags |= PMP_W;
if (reg->flags & SBI_DOMAIN_MEMREGION_SU_EXECUTABLE)
    pmp_flags |= PMP_X;

pmp_addr =  reg->base >> PMP_SHIFT;
if (pmp_gran_log2 <= reg->order && pmp_addr < pmp_addr_max)
    pmp_set(pmp_idx++, pmp_flags, reg->base, reg->order);
    ...
}
```

通过阅读 OpenSBI 中对 PMP 的处理可以进一步验证，当 `log2len == __riscv_xlen` 时，地址寄存器被设置为最大值，该寄存器为 PMP 地址寄存器序列中的最后一个。由于在初始化的过程中，OpenSBI 将自己占用的代码和数据空间设置了保护，导致在不修改 OpenSBI 的前提下，低于 `fw_addr` 的所有未分配给地址都无法被访问到，所以解决问题的关键就变成了找一段没有被 OpenSBI 设置保护的地址空间。为了简化实现，暂时将 UINTC 放在内存之后的区域，也就是 `0x9000_0000` 后面。

通过 GDB 查看 PMP 内容：

```txt
(gdb) info reg pmpcfg0 pmpaddr0 pmpaddr1 pmpaddr2
pmpcfg0        0x1f1818 2037784
pmpaddr0       0x801fff 8396799
pmpaddr1       0x20007fff       536903679
pmpaddr2       0xffffffffffffffff       -1
```

以上貌似并没有对访问区间加以限制，于是从异常着手开始研究：

发现抛出的异常位于 `riscv_cpu_do_transaction_failed`，这是一个用来注册 `cpu_transaction_failed` 的回调函数，这一函数位于 `io_writex`，发现错误来源为 `memory_region_dispatch_write` 没有返回 `MEMTX_OK`。

进入 `memory_region_dispatch_write` 发现是 `memory_region_access_valid` 返回了 false 。这一函数会根据错误输出一系列 `LOG_GUEST_ERROR` ，因此在运行 `qemu-system-riscv64` 指定参数 `-D <log file> -d guest_errors` 定位输出位置和错误原因为：

```txt
virtio_mmio_write: attempt to write guest features with guest_features_sel > 0 in legacy mode
Invalid write at addr 0x20, size 8, region 'riscv.uintc', reason: invalid size (min:32 max:32)
!memory_region_access_valid(mr, addr, size, true, attrs) 
io_writex cpu_transaction_failed: physaddr=0xc0000020
riscv_cpu_do_transaction_failed!
```

可以发现是 UINTC 在实现的过程中参数给错了。

另外，通过 `qemu-system-riscv64 -d ?` 查看支持的 `loglevel`。

修复问题后，观察输出发现，read 和 write 操作会默认将读写拆分为粒度为 4 字节的若干次读写，需要对代码进行修改和处理。

读取 sip 寄存器，发现 USIP 位并没有置为 1,查看 QEMU 代码发现：

```c
static RISCVException rmw_sip64(CPURISCVState *env, int csrno,
                                uint64_t *ret_val,
                                uint64_t new_val, uint64_t wr_mask)
{
    ...
    if (ret_val) {
        *ret_val &= env->mideleg & (S_MODE_INTERRUPTS | U_MODE_INTERRUPTS);
    }
    ...
}
```

`mideleg` 当前值为 `0x0000000000001666`，因此读出来的结果是零。

一种可能的解决办法是修改 OpenSBI 来支持 N 扩展，具体实现位于 `lib/sbi/sbi_hart.c`。

在 `static int delegate_traps(struct sbi_scratch *scratch)` 函数内添加一行 `interrupts |= MIP_USIP | MIP_UTIP | MIP_UEIP;` 即可。

执行以下命令编译 OpenSBI

```sh
export CROSS_COMPILE=riscv64-unknown-linux-gnu-
make all PLATFORM=generic PLATFORM_RISCV_XLEN=64
```

注意 QEMU 默认使用的 bios 位于 `pc-bios/opensbi-riscv64-generic-fw_dynamic.bin` ，因此我们也采用新编译好的 `fw_dynamic.bin` ，使用其他的会出错。

**2023.1.28**：目前可以在 S 态通过 UINTC 发送用户态中断并看到 USIP 被设置为 1 。

**2023.2.19**：目前 UIPI 指令也可以正常工作，至此针对 QEMU 的改动基本完成。

## 功能测试

在多核 os 中给出了大致的功能测试，测试运行在 S 态，不包含对 U 态执行流切换的测试：

```rust
// Test UIPI
unsafe {
    suicfg::write(UINTC_BASE);
    assert_eq!(suicfg::read(), UINTC_BASE);

    
    // Enable receiver status.
    let uirs_index = 2;
    // Receiver on hart 3
    *((UINTC_BASE + uirs_index * 0x20 + 8) as *mut u64) = ((3 << 16) as u64) | 3;
    suirs::write((1 << 63) | uirs_index);
    assert_eq!(suirs::read().bits(), (1 << 63) | uirs_index);
    // Write to high bits
    uipi_write(0x00010003);
    assert!(uipi_read() == 0x00010003);
    
    // Enable sender status.
    let frame = AllocatedFrame::new(true).unwrap();
    suist::write((1 << 63) | (1 << 44) | frame.number());
    assert_eq!(suist::read().bits(), (1 << 63) | (1 << 44) | frame.number());
    // valid entry, uirs index = 2, sender vector = 3
    *(frame.start_address().value() as *mut u64) = (2 << 48) | (3 << 16) | 1;
    // Send uipi with first uist entry
    info!("Send UIPI!");
    uipi_send(0);

    loop {
        if uintr::sip::read().usoft() {
            info!("Receive UINT!");
            uintr::sip::clear_usoft();
            break;
        }
    }
}
```

主核（一般是 0 号核）进行一系列的配置，例如 CSR 寄存器，Sender Table 等，对 3 号核发送用户态中断 `uipi_send()`。各个核判断收到用户态中断的依据是 USIP 位被设置为 1 。

```rust
// Test UIPI
unsafe {
    suicfg::write(UINTC_BASE);
    loop {
        if uintr::sip::read().usoft() {
            info!("Receive UINT!");
            if (hartid == 3) {
                let uirs_index = 2;
                suirs::write((1 << 63) | uirs_index);
                assert_eq!(uipi_read(), 0x0001000b);
                uintr::sip::clear_usoft();

                info!("Send UIPI!");
                for i in 0..3 {
                    *((UINTC_BASE + (i + 3) * 0x20 + 8) as *mut u64) = ((i << 16) as u64) | 3;
                    *((UINTC_BASE + (i + 3) * 0x20) as *mut u64) = 1;
                }
            }
            break;
        }
    }
}
```

3 号核接收到中断后再对其他核发送用户态中断，注意这里采用的是写外设方式，其他核收到中断后继续运行。

![func-test-1](imgs/func-test-1.png)