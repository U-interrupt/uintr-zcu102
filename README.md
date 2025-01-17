# User Interrupt

## [undergraduate-thesis](https://github.com/tkf2019/undergraduate-thesis)

## [RISC-V User-Interrupt Specification](./doc/spec.md)

![uintr](./doc/imgs/uintr.png)

## QEMU

- [QEMU support](https://github.com/U-interrupt/qemu)
- [QEMU development log](./doc/qemu-journal.md)

## tCore

- [tCore kernel](https://github.com/tkf2019/tCore)
- [tCore uintr support](https://github.com/tkf2019/tCore/blob/main/kernel/src/arch/riscv64/uintr.rs)
- [tCore uintr development log](./doc/kernel-journal.md)

## Testcases

![uintr-app](./doc/imgs/uintr2.png)

- [libc support](https://github.com/tkf2019/tCore-test/blob/main/libc/src/uintr/uintr.h)
- [uipi_sample.c](https://github.com/tkf2019/tCore-test/blob/main/libc/src/uintr/uipi_sample.c)

## Rocket Chip

![uipi-state-machine](./doc/imgs/uintr3.png)

- [Rocket Chip Support](https://github.com/U-interrupt/uintr-rocket-chip)
