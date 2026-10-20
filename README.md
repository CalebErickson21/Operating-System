# Operating System

An educational, from-scratch **x86-64 operating system** written primarily in C, with assembly where hardware-level control is necessary.

The purpose of this project is to learn how operating systems work **by implementing one**, progressing from a bootable kernel to memory management, interrupts, multitasking, and basic user-facing functionality. This is a learning project, not a production-ready operating system.

## Goals

- **Boot a custom kernel:** Load a freestanding ELF kernel using the [Limine bootloader](https://github.com/limine-bootloader/limine) and run it in QEMU.
- **Understand the hardware boundary:** Work with CPU state, interrupts, exceptions, timers, and basic device I/O.
- **Manage memory:** Implement physical-page allocation, virtual memory/paging, and eventually a kernel heap.
- **Manage execution:** Explore task switching, scheduling, and kernel-to-user transitions.
- **Add basic OS services:** Introduce system calls and experiment with filesystems and a simple shell as the kernel matures.
- **Develop a repeatable workflow:** Automate builds, use GDB to debug, and document design decisions and experiments.

## Development environment

| Component               | Choice                  | Role                                             |
| ----------------------- | ----------------------- | ------------------------------------------------ |
| Host OS                 | Windows 11              | Desktop environment                              |
| Development environment | Ubuntu on WSL2          | Linux build tools and terminal                   |
| Editor                  | Cursor                  | Writing and navigating code                      |
| Language/toolchain      | C, Clang/LLVM, LLD      | Kernel compilation and linking                   |
| Bootloader              | Limine (binary release) | Loads the kernel                                 |
| Emulator                | QEMU (x86-64)           | Boots and tests the OS without physical hardware |
| Debugger                | GDB                     | Inspect kernel execution                         |
| Build utilities         | GNU Make, Xorriso       | Build automation and ISO generation              |

**[Development environment setup guide →](docs/SETUP.md)** — prerequisites, installation, verification, Cursor configuration, and troubleshooting.

## Planned milestones

The milestones below are **planned**, not claims of completed functionality.

1. **Environment and boot:** Install the toolchain, integrate Limine, create a bootable image, and launch the first kernel in QEMU.
2. **Kernel diagnostics:** Display output, add a panic handler, and debug with GDB.
3. **CPU and interrupts:** Configure descriptor tables, handle exceptions, and set up a timer.
4. **Memory management:** Implement physical memory allocation, paging, and kernel dynamic allocation.
5. **Execution and scheduling:** Implement threads/tasks, context switches, and a basic scheduler.
6. **User-facing services:** Experiment with user mode, system calls, a minimal filesystem interface, and a simple shell.

The immediate goal is **booting a minimal freestanding C kernel in QEMU**. Later milestones may change as the project evolves.

## Building and running

The kernel build system and bootable image are **not implemented yet**. Once the Makefile, linker script, Limine configuration, and kernel source are in place, this section will document the exact commands used by this repository.

For now, follow the [setup guide](docs/SETUP.md) to prepare the development environment.

## References

- [OSDev Wiki](https://wiki.osdev.org/)
- [Limine bootloader](https://github.com/limine-bootloader/limine)
- [Official Limine x86-64 C template](https://github.com/limine-bootloader/limine-c-template-x86-64)
- [QEMU](https://www.qemu.org/)
- [LLVM/Clang](https://clang.llvm.org/)

## License

See [LICENSE](LICENSE) for the project's license.
