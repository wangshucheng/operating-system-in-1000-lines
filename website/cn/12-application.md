---
title: 应用程序
---

# 应用程序

在本章中，我们将准备在内核上运行的第一个应用程序可执行文件。

## 内存布局

在上一章中，我们使用分页机制实现了隔离的虚拟地址空间。让我们考虑在地址空间中的什么位置放置应用程序。

创建一个新的链接器脚本（`user.ld`），定义应用程序在内存中的位置：

```ld [user.ld]
ENTRY(start)

SECTIONS {
    . = 0x1000000;

    /* 机器代码 */
    .text :{
        KEEP(*(.text.start));
        *(.text .text.*);
    }

    /* 只读数据 */
    .rodata : ALIGN(4) {
        *(.rodata .rodata.*);
    }

    /* 带有初始值的数据 */
    .data : ALIGN(4) {
        *(.data .data.*);
    }

    /* 启动时应该用零填充的数据 */
    .bss : ALIGN(4) {
        *(.bss .bss.* .sbss .sbss.*);

        . = ALIGN(16);
        . += 64 * 1024; /* 64KB */
        __stack_top = .;

       ASSERT(. < 0x1800000, "可执行文件太大");
    }
}
```

它看起来和内核的链接器脚本很相似，对吧？关键的区别在于基地址（`0x1000000`），这样应用程序就不会与内核的地址空间重叠。

`ASSERT` 是一个断言，如果第一个参数中的条件不满足，链接器就会中止。在这里，它确保 `.bss` 段的结束位置（也就是应用程序内存的结束位置）不超过 `0x1800000`。这是为了确保可执行文件不会意外变得太大。

## 用户空间库

接下来，让我们为用户程序创建一个库。为了简单起见，我们将从最小的功能集开始，以启动应用程序：

```c [user.c]
#include "user.h"

extern char __stack_top[];

__attribute__((noreturn)) void exit(void) {
    for (;;);
}

void putchar(char c) {
    /* TODO */
}

__attribute__((section(".text.start")))
__attribute__((naked))
void start(void) {
    __asm__ __volatile__(
        "mv sp, %[stack_top] \n"
        "call main           \n"
        "call exit           \n"
        :: [stack_top] "r" (__stack_top)
    );
}
```

应用程序的执行从 `start` 函数开始。与内核的启动过程类似，它设置栈指针并调用应用程序的 `main` 函数。

我们准备了 `exit` 函数来终止应用程序。不过，目前我们只是让它执行一个无限循环。

另外，我们定义了 `putchar` 函数，这是 `common.c` 中的 `printf` 函数所引用的。我们稍后会实现它。

与内核的初始化过程不同，我们不会用零清除 `.bss` 段。这是因为内核保证已经用零填充了它（在 `alloc_pages` 函数中）。

> [!TIP]
>
> 在典型的操作系统中，分配的内存区域也已经用零填充。否则，内存可能包含其他进程的敏感信息（例如凭证），这可能导致严重的安全问题。

最后，为用户空间库准备一个头文件（`user.h`）：

```c [user.h]
#pragma once
#include "common.h"

__attribute__((noreturn)) void exit(void);
void putchar(char ch);
```

## 第一个应用程序

是时候创建第一个应用程序了！不幸的是，我们还没有显示字符的方法，所以不能从 "Hello, World!" 程序开始。相反，我们将创建一个简单的无限循环：

```c [shell.c]
#include "user.h"

void main(void) {
    for (;;);
}
```

## 构建应用程序

应用程序将与内核分开构建。让我们创建一个新的脚本（`run.sh`）来构建应用程序：

```bash [run.sh] {1,3-6,10}
OBJCOPY=/opt/homebrew/opt/llvm/bin/llvm-objcopy

# 构建 shell（应用程序）
$CC $CFLAGS -Wl,-Tuser.ld -Wl,-Map=shell.map -o shell.elf shell.c user.c common.c
$OBJCOPY --set-section-flags .bss=alloc,contents -O binary shell.elf shell.bin
$OBJCOPY -Ibinary -Oelf32-littleriscv shell.bin shell.bin.o

# 构建内核
$CC $CFLAGS -Wl,-Tkernel.ld -Wl,-Map=kernel.map -o kernel.elf \
    kernel.c common.c shell.bin.o
```

第一个 `$CC` 调用与内核构建脚本非常相似。编译 C 文件并使用 `user.ld` 链接器脚本链接它们。

第一个 `$OBJCOPY` 命令将可执行文件（ELF 格式）转换为原始二进制格式。原始二进制是从基地址（在本例中为 `0x1000000`）在内存中展开的实际内容。操作系统可以通过简单地复制原始二进制的内容来在内存中准备应用程序。常见的操作系统使用像 ELF 这样的格式，其中内存内容和它们的映射信息是分开的，但在本书中，我们将为了简单起见使用原始二进制。

第二个 `$OBJCOPY` 命令将原始二进制执行映像转换为可以嵌入 C 语言的格式。让我们使用 `llvm-nm` 命令看看里面有什么：

```
$ llvm-nm shell.bin.o
00000000 D _binary_shell_bin_start
00010260 D _binary_shell_bin_end
00010260 A _binary_shell_bin_size
```

前缀 `_binary_` 后面跟着文件名，然后是 `start`、`end` 和 `size`。这些是表示执行映像的开始、结束和大小的符号。在实践中，它们的使用方式如下：

```c
extern char _binary_shell_bin_start[];
extern char _binary_shell_bin_size[];

void main(void) {
    uint8_t *shell_bin = (uint8_t *) _binary_shell_bin_start;
    printf("shell_bin size = %d\n", (int) _binary_shell_bin_size);
    printf("shell_bin[0] = %x (%d bytes)\n", shell_bin[0]);
}
```

这个程序输出 `shell.bin` 的文件大小和其内容的第一个字节。换句话说，你可以把 `_binary_shell_bin_start` 变量当作它包含文件内容，就像：

```c
char _binary_shell_bin_start[] = "<shell.bin 内容在这里>";
```

`_binary_shell_bin_size` 变量包含文件大小。然而，它的使用方式有点不寻常。让我们再次用 `llvm-nm` 检查：

```
$ llvm-nm shell.bin.o | grep _binary_shell_bin_size
00010454 A _binary_shell_bin_size

$ ls -al shell.bin   ← 注意：不要与 shell.bin.o 混淆！
-rwxr-xr-x 1 seiya staff 66644 Oct 24 13:35 shell.bin

$ python3 -c 'print(0x10454)'
66644
```

`llvm-nm` 输出中的第一列是符号的*地址*。这个 `10260` 值与文件大小匹配，但这不是巧合。通常，`.o` 文件中每个地址的值都由链接器决定。然而，`_binary_shell_bin_size` 是特殊的。

第二列中的 `A` 表示 `_binary_shell_bin_size` 的地址是一种不应该被链接器更改的符号类型（绝对的）。也就是说，它将文件大小作为地址嵌入。

通过将其定义为任意类型的数组，如 `char _binary_shell_bin_size[]`，`_binary_shell_bin_size` 将被视为存储其*地址*的指针。然而，由于我们在这里将文件大小作为地址嵌入，所以转换它将得到文件大小。这是一个常见的技巧（或者说是一个不太优雅的技巧），它利用了目标文件格式。

最后，我们在内核编译的 `clang` 参数中添加了 `shell.bin.o`。它将第一个应用程序的可执行文件嵌入到内核映像中。

## 反汇编可执行文件

在反汇编中，我们可以看到 `.text.start` 段被放置在可执行文件的开头。`start` 函数应该放在 `0x1000000` 处，如下所示：

```
$ llvm-objdump -d shell.elf

shell.elf:	file format elf32-littleriscv

Disassembly of section .text:

01000000 <start>:
 1000000: 37 05 01 01  	lui	a0, 4112
 1000004: 13 05 05 26  	addi	a0, a0, 608
 1000008: 2a 81        	mv	sp, a0
 100000a: 19 20        	jal	0x1000010 <main>
 100000c: 29 20        	jal	0x1000016 <exit>
 100000e: 00 00        	unimp

01000010 <main>:
 1000010: 01 a0        	j	0x1000010 <main>
 1000012: 00 00        	unimp

01000016 <exit>:
 1000016: 01 a0        	j	0x1000016 <exit>
```
