---
title: 开始入门
---

# 开始入门

本书假设你使用的是 UNIX 或类 UNIX 操作系统，如 macOS 或 Ubuntu。如果你使用的是 Windows，请安装 Windows Subsystem for Linux (WSL2)，然后按照 Ubuntu 的说明进行操作。

## 安装开发工具

### macOS 

安装 [Homebrew](https://brew.sh)，然后运行以下命令来获取所需的所有工具：

```
brew install llvm lld qemu
```

### Ubuntu

使用 `apt` 安装软件包：

```
sudo apt update && sudo apt install -y clang llvm lld qemu-system-riscv32 curl
```

另外，下载 OpenSBI（可以将其理解为 PC 上的 BIOS/UEFI）：

```
curl -LO https://github.com/qemu/qemu/raw/v8.0.4/pc-bios/opensbi-riscv32-generic-fw_dynamic.bin
```

> [!WARNING]
>
> 运行 QEMU 时，请确保 `opensbi-riscv32-generic-fw_dynamic.bin` 在你的当前目录中。如果不在，你会看到这个错误：
>
> ```
> qemu-system-riscv32: Unable to load the RISC-V firmware "opensbi-riscv32-generic-fw_dynamic.bin"
> ```

### 其他操作系统用户

如果你使用其他操作系统，请获取以下工具：

- `bash`：命令行 shell。通常是预装的。
- `tar`：通常是预装的。推荐使用 GNU 版本，而不是 BSD 版本。
- `clang`：C 编译器。确保它支持 32 位 RISC-V CPU（见下文）。
- `lld`：LLVM 链接器，用于将编译后的目标文件打包成可执行文件。
- `llvm-objcopy`：目标文件编辑器。它随 LLVM 包一起提供（通常是 `llvm` 包）。
- `llvm-objdump`：反汇编器。与 `llvm-objcopy` 相同。
- `llvm-readelf`：ELF 文件读取器。与 `llvm-objcopy` 相同。
- `qemu-system-riscv32`：32 位 RISC-V CPU 模拟器。它是 QEMU 包的一部分（通常是 `qemu` 包）。

> [!TIP]
>
> 要检查你的 `clang` 是否支持 32 位 RISC-V CPU，运行以下命令：
>
> ```
> $ clang -print-targets | grep riscv32
>     riscv32     - 32-bit RISC-V
> ```
>
> 你应该能看到 `riscv32`。注意，macOS 上预装的 clang 不会显示这个。这就是为什么你需要在 Homebrew 的 `llvm` 包中安装另一个 `clang`。

## 设置 Git 仓库（可选）

如果你使用 Git 仓库，请使用以下 `.gitignore` 文件：

```gitignore [.gitignore]
/disk/*
!/disk/.gitkeep
*.map
*.tar
*.o
*.elf
*.bin
*.log
*.pcap
```

一切准备就绪！让我们开始构建你的第一个操作系统吧！
