# Development Environment Setup

This guide reproduces the initial development environment for the **Operating System** project. It assumes **Windows 11** with **WSL2 + Ubuntu**, using **Cursor** as the editor. Most build commands run **inside Ubuntu**, not in PowerShell or Git Bash.

> **Scope:** These steps install and verify the tools. They **do not yet build the project's OS**; a kernel, linker script, Limine configuration, and build rules must first be implemented.

## 1. How the tools fit together

| Tool           | Responsibility                                     |
| -------------- | -------------------------------------------------- |
| Windows 11     | Host desktop and editor                            |
| WSL2 / Ubuntu  | Linux environment for building and testing         |
| Cursor         | Code editing, terminal, language tooling           |
| Clang          | Compile freestanding C into kernel object files    |
| LLD (`ld.lld`) | Link kernel objects into an ELF executable         |
| GNU Make       | Run reproducible build commands                    |
| Limine         | Load the kernel and hand over control              |
| Xorriso        | Assemble a bootable ISO image                      |
| QEMU           | Emulate an x86-64 machine                          |
| GDB            | Debug kernel code (eventually via QEMU's GDB stub) |

**Why WSL2?** It provides a real Linux userland/kernel environment and makes Linux-oriented build tooling easier to use. **Git Bash and Cygwin are not required** for these steps. A Cygwin-targeting GCC is not automatically an appropriate kernel compiler. You can keep Git Bash for ordinary Windows Git operations.

## 2. Install Ubuntu with WSL2

Open **Windows PowerShell** (as Administrator when prompted) and inspect current distributions:

```powershell
wsl --list --verbose
```

If `docker-desktop` is the only entry, WSL2 is available but you still need a general-purpose Linux distribution. Install Ubuntu:

```powershell
wsl --install -d Ubuntu
```

Restart Windows if prompted. On first launch, create a **UNIX username and password**. The password is used for `sudo`; input is not displayed as you type.

Verify Ubuntu is running as WSL version **2**:

```powershell
wsl --list --verbose
```

If necessary, make it the default distribution:

```powershell
wsl --set-default Ubuntu
```

### Opening Ubuntu later

Use any of these methods:

- Search for **Ubuntu** in the Windows Start menu.
- Select the **Ubuntu** profile in Windows Terminal.
- From Git Bash, PowerShell, or CMD, run `wsl.exe -d Ubuntu`.
- Open a WSL-connected terminal in Cursor.

Type `exit` to leave a WSL shell.

## 3. Update Ubuntu and install tools

All commands in this section run in an **Ubuntu terminal**:

```bash
sudo apt update
sudo apt upgrade -y
sudo apt install -y \
    build-essential clang lld gdb git curl make \
    qemu-system-x86 xorriso ovmf clang-format
```

Explanation:

- `apt update` refreshes available package information.
- `apt upgrade` updates already installed Ubuntu packages.
- `apt install` adds the compilers, linker, debugger, emulator, formatter, and image-build tools.
- `build-essential` supplies GCC and common host build utilities; **Clang/LLD** are the intended kernel compiler/linker in this project.
- `ovmf` supplies UEFI firmware for relevant QEMU boot configurations; it is not essential for every BIOS-only smoke test.

Verify the command-line tools:

```bash
clang --version
ld.lld --version
make --version
gdb --version
qemu-system-x86_64 --version
xorriso -version
clang-format --version
```

A version or help banner generally indicates the command is available. Some tools print version information to standard error; that is normal.

> **Kernel compilation note:** `clang hello.c -o hello` produces an ordinary Linux application, **not a bootable kernel**. A freestanding kernel needs architecture-appropriate compilation flags, a linker script and compatible ELF output. See the official Limine C template linked below for a working reference.

## 4. Obtain the Limine binary distribution

Use Limine's **binary release** rather than the `limine-<version>.tar.gz` source release. The binary archive contains compiled bootloader components. The portable host utility can be built locally with `make`.

From Ubuntu:

```bash
mkdir -p ~/osdev/tools
cd ~/osdev/tools

curl -fL -o limine-binary.tar.gz \
    https://github.com/Limine-Bootloader/Limine/releases/download/v12.9.4/limine-binary.tar.gz

tar -xzf limine-binary.tar.gz
make -C limine-binary
```

Verify the files (these checks assume the archive extracts to `limine-binary/`):

```bash
ls limine-binary/
test -f limine-binary/BOOTX64.EFI && echo "UEFI bootloader found"
test -x limine-binary/limine && echo "Limine host utility built"
```

This keeps Limine under `~/osdev/tools/limine-binary/`; **a system-wide Limine installation is unnecessary**. When the OS's build scripts are written, they can reference this directory or manage Limine as a pinned project dependency. For reproducible builds, **pin an exact Limine version** rather than relying indefinitely on `latest`.

Official sources:

- [Limine releases](https://github.com/limine-bootloader/limine/releases)
- [Limine repository and binary-release notes](https://github.com/limine-bootloader/limine)

## 5. Configure Cursor and code formatting

1. Open Cursor and connect to Ubuntu using its **WSL/Remote WSL** workflow, if available for your version. Alternatively, open a local Windows project folder and select an Ubuntu terminal.
2. Choose **one primary C/C++ language service** (for example, C/C++ by Anysphere **or** the LLVM `clangd` extension), to avoid duplicate diagnostics.
3. For formatting on save, configure a formatter that actually supports **Format Document** in your current Cursor environment. The LLVM clangd extension can provide formatting; do not assume Anysphere C/C++ alone registers a document formatter.
4. Open a `.c` file and use **Format Document With...** to verify which formatter is available.

For the **LLVM clangd extension** (`llvm-vs-code-extensions.vscode-clangd`), the following is a possible addition to Cursor's **User Settings (JSON)**, provided that extension is installed and active in the current environment:

```json
{
	"[c]": {
		"editor.tabSize": 4,
		"editor.insertSpaces": true,
		"editor.detectIndentation": false,
		"editor.defaultFormatter": "llvm-vs-code-extensions.vscode-clangd",
		"editor.formatOnSave": true
	},
	"[cpp]": {
		"editor.tabSize": 4,
		"editor.insertSpaces": true,
		"editor.detectIndentation": false,
		"editor.defaultFormatter": "llvm-vs-code-extensions.vscode-clangd",
		"editor.formatOnSave": true
	}
}
```

Merge the fields into any existing settings object; do not overwrite unrelated settings.

Create a `.clang-format` file in the repository root for consistent four-space indentation:

```yaml
BasedOnStyle: LLVM
IndentWidth: 4
UseTab: Never
ColumnLimit: 100
PointerAlignment: Right
```

Formatting can also be performed without any editor extension:

```bash
clang-format -i path/to/file.c
```

`-i` replaces the file contents with formatted output. Commit only the changes you intend to keep.

## 6. Locate and test source files

In Ubuntu, Windows drives appear under `/mnt/`:

| Windows                            | Ubuntu WSL                             |
| ---------------------------------- | -------------------------------------- |
| `C:\Users\<windows-user>\Projects` | `/mnt/c/Users/<windows-user>/Projects` |
| `C:\`                              | `/mnt/c/`                              |

For a quick ordinary C-program smoke test (this is **not** kernel compilation):

```bash
cat > /tmp/hello.c <<'C'
#include <stdio.h>

int main(void) {
    puts("Hello from Ubuntu!");
    return 0;
}
C

clang -Wall -Wextra -g /tmp/hello.c -o /tmp/hello
/tmp/hello
```

You should see `Hello from Ubuntu!`.

For Linux-heavy work, placing the repository under `~/projects/` in Ubuntu generally gives better filesystem performance than building under `/mnt/c`. If your repository remains on Windows, WSL can still access and compile its files. To open the WSL directory in Windows Explorer:

```bash
explorer.exe .
```

## 7. Next step: boot a kernel in QEMU

Once the project includes a kernel source file, linker script, Limine boot configuration, and Makefile, the intended pipeline is:

```text
C/assembly kernel source
    -> Clang (ELF object files)
    -> LLD (kernel ELF)
    -> Limine files + Xorriso (bootable ISO)
    -> QEMU (virtual x86-64 computer)
    -> GDB (debugging)
```

A future project command might look like `make run`, but **this repository does not yet promise that target**.

For a reference implementation, consult the [official Limine x86-64 C template](https://github.com/limine-bootloader/limine-c-template-x86-64). Its Makefile supports `make TOOLCHAIN=llvm` and provides build/run targets; those commands apply **to that template**, not automatically to this repository.

> **QEMU on WSL:** Windows 11 normally supports Linux graphical applications through WSLg. If QEMU's graphical window does not appear, first confirm that WSLg/display integration is available. A headless QEMU mode is possible once the kernel supports serial output.

## 8. Troubleshooting

| Problem                                                    | What to check                                                                                                                          |
| ---------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| `wsl --list --verbose` lists only `docker-desktop`         | Install Ubuntu with `wsl --install -d Ubuntu`; do not develop inside Docker Desktop's internal WSL distribution.                       |
| `clang`, `gdb`, or `qemu-system-x86_64`: command not found | Ensure the relevant `apt install` command completed in **Ubuntu** (not Git Bash/Cygwin).                                               |
| `gcc: fatal error: no input files`                         | Supply the source filename before `-o`: `gcc HelloWorld.c -o main`.                                                                    |
| A Linux binary won't launch by double-clicking in Windows  | A WSL-built ELF executable must run inside Linux/WSL; it is not a Windows `.exe`.                                                      |
| Limine archive extraction fails                            | Ensure you downloaded `limine-binary.tar.gz`, not an HTML error page; `curl -fL` reports HTTP failures.                                |
| Cursor does not format on `Ctrl+S`                         | Check the active formatter using **Format Document With...**, the extension's environment (Windows vs WSL), and `editor.formatOnSave`. |
| Build is slower under `/mnt/c`                             | Consider moving Linux-heavy build files into the WSL filesystem under `~/projects/`.                                                   |

## References

- [Microsoft: Install WSL](https://learn.microsoft.com/en-us/windows/wsl/install)
- [Microsoft: Filesystems with WSL](https://learn.microsoft.com/en-us/windows/wsl/filesystems)
- [Limine official repository](https://github.com/limine-bootloader/limine)
- [Limine official x86-64 C template](https://github.com/limine-bootloader/limine-c-template-x86-64)
- [QEMU documentation](https://www.qemu.org/docs/master/)
- [Clang documentation](https://clang.llvm.org/docs/)
- [clang-format documentation](https://clang.llvm.org/docs/ClangFormat.html)
