---
title: 启动
---

# 内核启动

当计算机开机时，CPU会初始化自身并开始执行操作系统。操作系统初始化硬件并启动应用程序。这个过程被称为"启动"。

在操作系统启动之前发生了什么？在PC中，BIOS（或现代PC中的UEFI）初始化硬件，显示启动画面，并从磁盘加载操作系统。在QEMU的`virt`机器中，OpenSBI相当于BIOS/UEFI的角色。

## 监管者二进制接口（SBI）

监管者二进制接口（SBI）是一个面向操作系统内核的API，它定义了固件（OpenSBI）为操作系统提供的功能。

SBI规范在[GitHub上发布](https://github.com/riscv-non-isa/riscv-sbi-doc/releases)。它定义了一些有用的功能，如在调试控制台（例如串口）上显示字符、重启/关机和定时器设置等。

一个著名的SBI实现是[OpenSBI](https://github.com/riscv-software-src/opensbi)。在QEMU中，OpenSBI默认启动，执行硬件特定的初始化，然后启动内核。

## 让我们启动OpenSBI

首先，让我们看看OpenSBI是如何启动的。创建一个名为`run.sh`的shell脚本：

```
$ touch run.sh
$ chmod +x run.sh
```

```bash [run.sh]
#!/bin/bash
set -xue

# QEMU文件路径
QEMU=qemu-system-riscv32

# 启动QEMU
$QEMU -machine virt -bios default -nographic -serial mon:stdio --no-reboot
```

QEMU使用各种选项来启动虚拟机。以下是脚本中使用的选项：

- `-machine virt`：启动一个`virt`机器。你可以使用`-machine '?'`选项查看其他支持的机器。
- `-bios default`：使用默认固件（在这种情况下是OpenSBI）。
- `-nographic`：启动QEMU时不使用GUI窗口。
- `-serial mon:stdio`：将QEMU的标准输入/输出连接到虚拟机的串口。指定`mon:`允许通过按<kbd>Ctrl</kbd>+<kbd>A</kbd>然后按<kbd>C</kbd>切换到QEMU监视器。
- `--no-reboot`：如果虚拟机崩溃，停止模拟器而不重启（对调试有用）。

> [!TIP]
>
> 在macOS中，你可以使用以下命令查看Homebrew的QEMU路径：
>
> ```
> $ ls $(brew --prefix)/bin/qemu-system-riscv32
> /opt/homebrew/bin/qemu-system-riscv32
> ```

运行脚本，你将看到以下横幅：

```
$ ./run.sh

OpenSBI v1.2
   ____                    _____ ____ _____
  / __ \                  / ____|  _ \_   _|
 | |  | |_ __   ___ _ __ | (___ | |_) || |
 | |  | | '_ \ / _ \ '_ \ \___ \|  _ < | |
 | |__| | |_) |  __/ | | |____) | |_) || |_
  \____/| .__/ \___|_| |_|_____/|____/_____|
        | |
        |_|

Platform Name             : riscv-virtio,qemu
Platform Features         : medeleg
Platform HART Count       : 1
Platform IPI Device       : aclint-mswi
Platform Timer Device     : aclint-mtimer @ 10000000Hz
...
```

OpenSBI显示OpenSBI版本、平台名称、功能、HART（CPU核心）数量等调试信息。

当你按任何键时，什么都不会发生。这是因为QEMU的标准输入/输出已连接到虚拟机的串口，你输入的字符被发送到OpenSBI。但是没有人读取输入的字符。

按<kbd>Ctrl</kbd>+<kbd>A</kbd>然后按<kbd>C</kbd>切换到QEMU调试控制台（QEMU监视器）。你可以在监视器中使用`q`命令退出QEMU：

```
QEMU 8.0.2 monitor - type 'help' for more information
(qemu) q
```

> [!TIP]
>
> <kbd>Ctrl</kbd>+<kbd>A</kbd>除了切换到QEMU监视器（<kbd>C</kbd>键）外还有其他功能。例如，按<kbd>X</kbd>键将立即退出QEMU。
>
> ```
> C-a h    打印此帮助
> C-a x    退出模拟器
> C-a s    将磁盘数据保存回文件（如果使用-snapshot）
> C-a t    切换控制台时间戳
> C-a b    发送中断（magic sysrq）
> C-a c    在控制台和监视器之间切换
> C-a C-a  发送C-a
> ```

## 链接器脚本

链接器脚本是一个定义可执行文件内存布局的文件。根据布局，链接器为函数和变量分配内存地址。

让我们创建一个名为`kernel.ld`的新文件：

```ld [kernel.ld]
ENTRY(boot)

SECTIONS {
    . = 0x80200000;

    .text :{
        KEEP(*(.text.boot));
        *(.text .text.*);
    }

    .rodata : ALIGN(4) {
        *(.rodata .rodata.*);
    }

    .data : ALIGN(4) {
        *(.data .data.*);
    }

    .bss : ALIGN(4) {
        __bss = .;
        *(.bss .bss.* .sbss .sbss.*);
        __bss_end = .;
    }

    . = ALIGN(4);
    . += 128 * 1024; /* 128KB */
    __stack_top = .;
}
```

以下是链接器脚本的关键点：

- 内核的入口点是`boot`函数。
- 基地址是`0x80200000`。
- `.text.boot`段始终放在开头。
- 各个段按`.text`、`.rodata`、`.data`和`.bss`的顺序放置。
- 内核栈位于`.bss`段之后，大小为128KB。

这里提到的`.text`、`.rodata`、`.data`和`.bss`段是具有特定作用的数据区域：

| 段        | 描述                                   |
| --------- | -------------------------------------- |
| `.text`   | 此段包含程序的代码。                   |
| `.rodata` | 此段包含只读的常量数据。               |
| `.data`   | 此段包含可读写的数据。                 |
| `.bss`    | 此段包含初始值为零的可读写数据。       |

让我们仔细看看链接器脚本的语法。首先，`ENTRY(boot)`声明`boot`函数是程序的入口点。然后，在`SECTIONS`块内定义各个段的放置位置。

`*(.text .text.*)`指令将所有文件（`*`）中的`.text`段和任何以`.text.`开头的段放在该位置。

`.`符号表示当前地址。它会随着数据的放置（如`*(.text)`）自动增加。语句`. += 128 * 1024`表示"将当前地址前进128KB"。`ALIGN(4)`指令确保当前地址调整到4字节边界。

最后，`__bss = .`将当前地址赋值给符号`__bss`。在C语言中，你可以使用`extern char symbol_name`引用定义的符号。

> [!TIP]
>
> 链接器脚本提供了许多便利的功能，特别是对于内核开发。你可以在GitHub上找到真实世界的例子！

## 最小内核

现在我们准备开始编写内核了。让我们从一个最小的内核开始！创建一个名为`kernel.c`的C语言源代码文件：

```c [kernel.c]
typedef unsigned char uint8_t;
typedef unsigned int uint32_t;
typedef uint32_t size_t;

extern char __bss[], __bss_end[], __stack_top[];

void *memset(void *buf, char c, size_t n) {
    uint8_t *p = (uint8_t *) buf;
    while (n--)
        *p++ = c;
    return buf;
}

void kernel_main(void) {
    memset(__bss, 0, (size_t) __bss_end - (size_t) __bss);

    for (;;);
}

__attribute__((section(".text.boot")))
__attribute__((naked))
void boot(void) {
    __asm__ __volatile__(
        "mv sp, %[stack_top]\n" // 设置栈指针
        "j kernel_main\n"       // 跳转到内核主函数
        :
        : [stack_top] "r" (__stack_top) // 将栈顶地址作为%[stack_top]传递
    );
}
```

让我们一个一个地探讨关键点：

### 内核入口点

内核的执行从`boot`函数开始，这在链接器脚本中被指定为入口点。在这个函数中，栈指针（`sp`）被设置为链接器脚本中定义的栈区域的结束地址。然后，它跳转到`kernel_main`函数。需要注意的是，栈是向零方向增长的，也就是说，它在使用时会递减。因此，必须设置栈区域的结束地址（而不是起始地址）。

### `boot`函数属性

`boot`函数有两个特殊属性。`__attribute__((naked))`属性指示编译器不要在函数体前后生成不必要的代码，如返回指令。这确保内联汇编代码就是确切的函数体。

`boot`函数还有`__attribute__((section(".text.boot")))`属性，它控制函数在链接器脚本中的放置位置。由于OpenSBI只是简单地跳转到`0x80200000`而不知道入口点，所以`boot`函数需要放在`0x80200000`。

### 使用`extern char`获取链接器脚本符号

在文件开头，使用`extern char`声明链接器脚本中定义的每个符号。在这里，我们只关心获取符号的地址，所以使用`char`类型并不那么重要。

我们也可以声明为`extern char __bss;`，但单独的`__bss`意味着*"`.bss`段第0字节的值"*而不是*"`.bss`段的起始地址"*。因此，建议添加`[]`以确保`__bss`返回地址并防止任何粗心的错误。

### `.bss`段初始化

在`kernel_main`函数中，首先使用`memset`函数将`.bss`段初始化为零。虽然某些引导加载程序可能会识别并清零`.bss`段，但我们为了以防万一手动初始化它。最后，函数进入无限循环，内核终止。

## 让我们运行！

在`run.sh`中添加内核构建命令和新的QEMU选项（`-kernel kernel.elf`）：

```bash [run.sh] {6-13,17}
#!/bin/bash
set -xue

QEMU=qemu-system-riscv32

# 新增：clang路径和编译器标志
CC=/opt/homebrew/opt/llvm/bin/clang  # Ubuntu用户：使用CC=clang
CFLAGS="-std=c11 -O2 -g3 -Wall -Wextra --target=riscv32 -ffreestanding -nostdlib"

# 新增：构建内核
$CC $CFLAGS -Wl,-Tkernel.ld -Wl,-Map=kernel.map -o kernel.elf \
    kernel.c

# 启动QEMU
$QEMU -machine virt -bios default -nographic -serial mon:stdio --no-reboot \
    -kernel kernel.elf # 新增：加载内核
```

> [!TIP]
>
> 你可以使用以下命令在macOS上检查Homebrew版本的clang的文件路径：
>
> ```
> $ ls $(brew --prefix)/opt/llvm/bin/clang
> /opt/homebrew/opt/llvm/bin/clang
> ```

指定的clang选项（`CFLAGS`变量）如下：

| 选项 | 描述 |
| ------ | ----------- |
| `-std=c11` | 使用C11。 |
| `-O2` | 启用优化以生成高效的机器代码。 |
| `-g3` | 生成最大量的调试信息。 |
| `-Wall` | 启用主要警告。 |
| `-Wextra` | 启用额外警告。 |
| `--target=riscv32` | 为32位RISC-V编译。 |
| `-ffreestanding` | 不使用主机环境（你的开发环境）的标准库。 |
| `-nostdlib` | 不链接标准库。 |
| `-Wl,-Tkernel.ld` | 指定链接器脚本。 |
| `-Wl,-Map=kernel.map` | 输出映射文件（链接器分配结果）。 |

`-Wl,`表示将选项传递给链接器而不是C编译器。`clang`命令执行C编译并在内部执行链接器。

## 你的第一次内核调试

当你运行`run.sh`时，内核进入无限循环。没有任何迹象表明内核正在正确运行。但别担心，这在底层开发中很常见！这就是QEMU的调试功能派上用场的地方。

要获取有关CPU寄存器的更多信息，打开QEMU监视器并执行`info registers`命令：

```
QEMU 8.0.2 monitor - type 'help' for more information
(qemu) info registers

CPU#0
 V      =   0
 pc       80200014  ← 要执行的指令的地址（程序计数器）
 ...
 x0/zero  00000000 x1/ra    8000a084 x2/sp    80220018 x3/gp    00000000  ← 每个寄存器的值
 x4/tp    80033000 x5/t0    00000001 x6/t1    00000002 x7/t2    00000000
 x8/s0    80032f50 x9/s1    00000001 x10/a0   80220018 x11/a1   87e00000
 x12/a2   00000007 x13/a3   00000019 x14/a4   00000000 x15/a5   00000001
 x16/a6   00000001 x17/a7   00000005 x18/s2   80200000 x19/s3   00000000
 x20/s4   87e00000 x21/s5   00000000 x22/s6   80006800 x23/s7   8001c020
 x24/s8   00002000 x25/s9   8002b4e4 x26/s10  00000000 x27/s11  00000000
 x28/t3   616d6569 x29/t4   8001a5a1 x30/t5   000000b4 x31/t6   00000000
```

> [!TIP]
>
> 具体值可能因clang和QEMU的版本而异。

`pc 80200014`显示当前程序计数器，即正在执行的指令的地址。让我们使用反汇编器（`llvm-objdump`）来确定具体的代码行：

```
$ llvm-objdump -d kernel.elf

kernel.elf:     file format elf32-littleriscv

Disassembly of section .text:

80200000 <boot>:  ← boot函数
80200000: 37 05 22 80   lui     a0, 524832
80200004: 13 05 85 01   addi    a0, a0, 24
80200008: 2a 81         mv      sp, a0
8020000a: 6f 00 60 00   j       0x80200010 <kernel_main>
8020000e: 00 00         unimp

80200010 <kernel_main>:  ← kernel_main函数
80200010: 73 00 50 10   wfi
80200014: f5 bf         j       0x80200010 <kernel_main>  ← pc在这里
```

每行对应一条指令。每列表示：

- 指令的地址。
- 机器代码的十六进制转储。
- 反汇编的指令。

`pc 80200014`意味着当前执行的指令是`j 0x80200010`。这确认QEMU已正确到达`kernel_main`函数。

让我们也检查一下栈指针（sp寄存器）是否设置为链接器脚本中定义的`__stack_top`值。寄存器转储显示`x2/sp 80220018`。要查看链接器将`__stack_top`放在哪里，请检查`kernel.map`文件：

```
     VMA      LMA     Size Align Out     In      Symbol
       0        0 80200000     1 . = 0x80200000
80200000 80200000       16     4 .text
...
80200016 80200016        2     1 . = ALIGN ( 4 )
80200018 80200018    20000     1 . += 128 * 1024
80220018 80220018        0     1 __stack_top = .
```

或者，你也可以使用`llvm-nm`检查函数/变量的地址：

```
$ llvm-nm kernel.elf
80200010 t .LBB0_1
00000000 N .Lline_table_start0
80220018 T __stack_top
80200000 T boot
80200010 T kernel_main
```

第一列是它们被放置的地址（VMA）。你可以看到`__stack_top`被放置在`0x80220018`。这确认了栈指针在`boot`函数中被正确设置。很好！

随着执行的进行，`info registers`的结果会发生变化。如果你想暂时停止模拟，可以在QEMU监视器中使用`stop`命令：

```
(qemu) stop             ← 进程停止
(qemu) info registers   ← 你可以观察停止时的状态
(qemu) cont             ← 进程恢复
```

现在你已经成功编写了你的第一个内核！
