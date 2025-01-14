---
title: 进程
---

# 进程

进程是应用程序的一个实例。每个进程都有自己独立的执行上下文和资源，比如虚拟地址空间。

> [!NOTE]
>
> 实际的操作系统将执行上下文作为一个单独的概念，称为*"线程"*。为了简单起见，在本书中我们将每个进程视为只有一个线程。

## 进程控制块

以下`process`结构定义了一个进程对象。它也被称为_"进程控制块(Process Control Block, PCB)"_。

```c
#define PROCS_MAX 8       // Maximum number of processes

#define PROC_UNUSED   0   // Unused process control structure
#define PROC_RUNNABLE 1   // Runnable process

struct process {
    int pid;             // Process ID
    int state;           // Process state: PROC_UNUSED or PROC_RUNNABLE
    vaddr_t sp;          // Stack pointer
    uint8_t stack[8192]; // Kernel stack
};
```

内核栈包含保存的CPU寄存器、返回地址（从哪里调用）以及局部变量。通过为每个进程准备一个内核栈，我们可以通过保存和恢复CPU寄存器以及切换栈指针来实现上下文切换。

> [!TIP]
>
> 还有另一种方法叫做*"单内核栈"*。不是为每个进程（或线程）都有一个内核栈，而是每个CPU只有一个栈。[seL4采用这种方法](https://trustworthy.systems/publications/theses_public/05/Warton%3Abe.abstract)。
>
> 这个*"在哪里存储程序上下文"*的问题在Go和Rust等编程语言的异步运行时中也是一个讨论的话题。如果你感兴趣，可以搜索*"stackless async"*。

## 上下文切换

切换进程执行上下文被称为*"上下文切换"*。以下`switch_context`函数是上下文切换的实现：

```c [kernel.c]
__attribute__((naked)) void switch_context(uint32_t *prev_sp,
                                           uint32_t *next_sp) {
    __asm__ __volatile__(
        // Save callee-saved registers onto the current process's stack.
        "addi sp, sp, -13 * 4\n" // Allocate stack space for 13 4-byte registers
        "sw ra,  0  * 4(sp)\n"   // Save callee-saved registers only
        "sw s0,  1  * 4(sp)\n"
        "sw s1,  2  * 4(sp)\n"
        "sw s2,  3  * 4(sp)\n"
        "sw s3,  4  * 4(sp)\n"
        "sw s4,  5  * 4(sp)\n"
        "sw s5,  6  * 4(sp)\n"
        "sw s6,  7  * 4(sp)\n"
        "sw s7,  8  * 4(sp)\n"
        "sw s8,  9  * 4(sp)\n"
        "sw s9,  10 * 4(sp)\n"
        "sw s10, 11 * 4(sp)\n"
        "sw s11, 12 * 4(sp)\n"

        // Switch the stack pointer.
        "sw sp, (a0)\n"         // *prev_sp = sp;
        "lw sp, (a1)\n"         // Switch stack pointer (sp) here

        // Restore callee-saved registers from the next process's stack.
        "lw ra,  0  * 4(sp)\n"  // Restore callee-saved registers only
        "lw s0,  1  * 4(sp)\n"
        "lw s1,  2  * 4(sp)\n"
        "lw s2,  3  * 4(sp)\n"
        "lw s3,  4  * 4(sp)\n"
        "lw s4,  5  * 4(sp)\n"
        "lw s5,  6  * 4(sp)\n"
        "lw s6,  7  * 4(sp)\n"
        "lw s7,  8  * 4(sp)\n"
        "lw s8,  9  * 4(sp)\n"
        "lw s9,  10 * 4(sp)\n"
        "lw s10, 11 * 4(sp)\n"
        "lw s11, 12 * 4(sp)\n"
        "addi sp, sp, 13 * 4\n"  // We've popped 13 4-byte registers from the stack
        "ret\n"
    );
}
```

`switch_context`将被调用者保存的寄存器保存到栈上，切换栈指针，然后从栈中恢复被调用者保存的寄存器。换句话说，执行上下文作为临时局部变量存储在栈上。或者，你也可以将上下文保存在`struct process`中，但这种基于栈的方法不是很简单吗？

被调用者保存的寄存器是被调用函数在返回前必须恢复的寄存器。在RISC-V中，`s0`到`s11`是被调用者保存的寄存器。其他寄存器如`a0`是调用者保存的寄存器，已经由调用者保存在栈上。这就是为什么`switch_context`只处理部分寄存器。

`naked`属性告诉编译器不要生成除内联汇编之外的任何其他代码。没有这个属性也应该可以工作，但使用它是一个好习惯，特别是当你手动修改栈指针时，可以避免意外行为。

> [!TIP]
>
> 被调用者/调用者保存的寄存器在[调用约定](https://riscv.org/wp-content/uploads/2015/01/riscv-calling.pdf)中定义。编译器根据这个约定生成代码。

接下来，让我们实现进程初始化函数`create_process`。它接受入口点作为参数，并返回指向创建的`process`结构的指针：

```c
struct process procs[PROCS_MAX]; // All process control structures.

struct process *create_process(uint32_t pc) {
    // Find an unused process control structure.
    struct process *proc = NULL;
    int i;
    for (i = 0; i < PROCS_MAX; i++) {
        if (procs[i].state == PROC_UNUSED) {
            proc = &procs[i];
            break;
        }
    }

    if (!proc)
        PANIC("no free process slots");

    // Stack callee-saved registers. These register values will be restored in
    // the first context switch in switch_context.
    uint32_t *sp = (uint32_t *) &proc->stack[sizeof(proc->stack)];
    *--sp = 0;                      // s11
    *--sp = 0;                      // s10
    *--sp = 0;                      // s9
    *--sp = 0;                      // s8
    *--sp = 0;                      // s7
    *--sp = 0;                      // s6
    *--sp = 0;                      // s5
    *--sp = 0;                      // s4
    *--sp = 0;                      // s3
    *--sp = 0;                      // s2
    *--sp = 0;                      // s1
    *--sp = 0;                      // s0
    *--sp = (uint32_t) pc;          // ra

    // Initialize fields.
    proc->pid = i + 1;
    proc->state = PROC_RUNNABLE;
    proc->sp = (uint32_t) sp;
    return proc;
}
```

## 测试上下文切换

我们已经实现了进程的最基本功能 - 多个程序的并发执行。让我们创建两个进程：

```c [kernel.c] {1-25,32-34}
void delay(void) {
    for (int i = 0; i < 30000000; i++)
        __asm__ __volatile__("nop"); // do nothing
}

struct process *proc_a;
struct process *proc_b;

void proc_a_entry(void) {
    printf("starting process A\n");
    while (1) {
        putchar('A');
        switch_context(&proc_a->sp, &proc_b->sp);
        delay();
    }
}

void proc_b_entry(void) {
    printf("starting process B\n");
    while (1) {
        putchar('B');
        switch_context(&proc_b->sp, &proc_a->sp);
        delay();
    }
}

void kernel_main(void) {
    memset(__bss, 0, (size_t) __bss_end - (size_t) __bss);

    WRITE_CSR(stvec, (uint32_t) kernel_entry);

    proc_a = create_process((uint32_t) proc_a_entry);
    proc_b = create_process((uint32_t) proc_b_entry);
    proc_a_entry();

    PANIC("unreachable here!");
}
```

`proc_a_entry`函数和`proc_b_entry`函数分别是进程A和进程B的入口点。在使用`putchar`函数显示单个字符后，它们使用`switch_context`函数切换到另一个进程。

`delay`函数实现了一个忙等待，以防止字符输出变得太快，这会使你的终端无响应。`nop`指令是一个"什么都不做"的指令。添加它是为了防止编译器优化删除循环。

现在，让我们试试！启动消息将各显示一次，然后"ABABAB..."将永远持续：

```
$ ./run.sh

starting process A
Astarting process B
BABABABABABABABABABABABABABABABABABABABABABABABABABABABABABABABABABABABABABABABABABABABABABABABABABABABABABABABABABABABABABABABABABABABABABABABABABABABABABABABABABABABABABABABABABABAQE
```

## 调度器

在前面的实验中，我们直接调用`switch_context`函数来指定"下一个要执行的进程"。然而，随着进程数量的增加，这种方法在确定下一个要切换到哪个进程时变得复杂。为了解决这个问题，让我们实现一个*"调度器"*，这是一个决定下一个进程的内核程序。

以下`yield`函数是调度器的实现：

> [!TIP]
>
> "yield"这个词经常被用作允许进程自愿放弃CPU给另一个进程的API的名称。

```c [kernel.c]
struct process *current_proc; // Currently running process
struct process *idle_proc;    // Idle process

void yield(void) {
    // Search for a runnable process
    struct process *next = idle_proc;
    for (int i = 0; i < PROCS_MAX; i++) {
        struct process *proc = &procs[(current_proc->pid + i) % PROCS_MAX];
        if (proc->state == PROC_RUNNABLE && proc->pid > 0) {
            next = proc;
            break;
        }
    }

    // If there's no runnable process other than the current one, return and continue processing
    if (next == current_proc)
        return;

    // Context switch
    struct process *prev = current_proc;
    current_proc = next;
    switch_context(&prev->sp, &next->sp);
}
```

这里，我们引入了两个全局变量。`current_proc`指向当前正在运行的进程。`idle_proc`指的是空闲进程，即"当没有可运行的进程时要运行的进程"。`idle_proc`在启动时作为进程ID为`-1`的进程创建，如下所示：

```c [kernel.c] {8-10,15-16}
void kernel_main(void) {
    memset(__bss, 0, (size_t) __bss_end - (size_t) __bss);

    printf("\n\n");

    WRITE_CSR(stvec, (uint32_t) kernel_entry);

    idle_proc = create_process((uint32_t) NULL);
    idle_proc->pid = -1; // idle
    current_proc = idle_proc;

    proc_a = create_process((uint32_t) proc_a_entry);
    proc_b = create_process((uint32_t) proc_b_entry);

    yield();
    PANIC("switched to idle process");
}
```

这个初始化过程的关键点是`current_proc = idle_proc`。这确保了引导进程的执行上下文被保存并作为空闲进程的上下文恢复。在第一次调用`yield`函数时，它从空闲进程切换到进程A，当切换回空闲进程时，它的行为就像从这个`yield`函数调用返回一样。

最后，修改`proc_a_entry`和`proc_b_entry`如下，调用`yield`函数而不是直接调用`switch_context`函数：

```c [kernel.c] {5,14}
void proc_a_entry(void) {
    printf("starting process A\n");
    while (1) {
        putchar('A');
        yield();
    }
}

void proc_b_entry(void) {
    printf("starting process B\n");
    while (1) {
        putchar('B');
        yield();
    }
}
```

如果"A"和"B"像以前一样打印，那就完美运行了！

## 异常处理程序的变化

在异常处理程序中，它将执行状态保存到栈上。然而，由于我们现在为每个进程使用单独的内核栈，我们需要稍微更新它。

首先，在进程切换期间，在`sscratch`寄存器中设置当前执行进程的内核栈的初始值。

```c [kernel.c] {4-8}
void yield(void) {
    /* omitted */

    __asm__ __volatile__(
        "csrw sscratch, %[sscratch]\n"
        :
        : [sscratch] "r" ((uint32_t) &next->stack[sizeof(next->stack)])
    );

    // Context switch
    struct process *prev = current_proc;
    current_proc = next;
    switch_context(&prev->sp, &next->sp);
}
```

由于栈指针向较低地址扩展，我们将第`sizeof(next->stack)`字节处的地址设置为内核栈的初始值。

对异常处理程序的修改如下：

```c [kernel.c] {3-4,38-44}
void kernel_entry(void) {
    __asm__ __volatile__(
        // Retrieve the kernel stack of the running process from sscratch.
        "csrrw sp, sscratch, sp\n"

        "addi sp, sp, -4 * 31\n"
        "sw ra,  4 * 0(sp)\n"
        "sw gp,  4 * 1(sp)\n"
        "sw tp,  4 * 2(sp)\n"
        "sw t0,  4 * 3(sp)\n"
        "sw t1,  4 * 4(sp)\n"
        "sw t2,  4 * 5(sp)\n"
        "sw t3,  4 * 6(sp)\n"
        "sw t4,  4 * 7(sp)\n"
        "sw t5,  4 * 8(sp)\n"
        "sw t6,  4 * 9(sp)\n"
        "sw a0,  4 * 10(sp)\n"
        "sw a1,  4 * 11(sp)\n"
        "sw a2,  4 * 12(sp)\n"
        "sw a3,  4 * 13(sp)\n"
        "sw a4,  4 * 14(sp)\n"
        "sw a5,  4 * 15(sp)\n"
        "sw a6,  4 * 16(sp)\n"
        "sw a7,  4 * 17(sp)\n"
        "sw s0,  4 * 18(sp)\n"
        "sw s1,  4 * 19(sp)\n"
        "sw s2,  4 * 20(sp)\n"
        "sw s3,  4 * 21(sp)\n"
        "sw s4,  4 * 22(sp)\n"
        "sw s5,  4 * 23(sp)\n"
        "sw s6,  4 * 24(sp)\n"
        "sw s7,  4 * 25(sp)\n"
        "sw s8,  4 * 26(sp)\n"
        "sw s9,  4 * 27(sp)\n"
        "sw s10, 4 * 28(sp)\n"
        "sw s11, 4 * 29(sp)\n"

        // Retrieve and save the sp at the time of exception.
        "csrr a0, sscratch\n"
        "sw a0,  4 * 30(sp)\n"

        // Reset the kernel stack.
        "addi a0, sp, 4 * 31\n"
        "csrw sscratch, a0\n"

        "mv a0, sp\n"
        "call handle_trap\n"
```

第一个`csrrw`指令简单来说是一个交换操作：

```
tmp = sp;
sp = sscratch;
sscratch = tmp;
```

因此，`sp`现在指向当前运行进程的*内核*（不是*用户*）栈。同时，`sscratch`现在保存了异常发生时`sp`（用户栈）的原始值。

在将其他寄存器保存到内核栈后，我们从`sscratch`恢复原始的`sp`值并将其保存到内核栈上。然后，计算`sscratch`的初始值并恢复它。

这里的关键点是每个进程都有自己独立的内核栈。通过在上下文切换期间切换`sscratch`的内容，我们可以从中断点恢复进程的执行，就像什么都没发生一样。

> [!TIP]
>
> 我们已经实现了"内核"栈的上下文切换机制。应用程序使用的栈（所谓的*用户栈*）将与内核栈分开分配。这将在后面的章节中实现。

## 附录：为什么我们需要重置栈指针？

在前一节中，你可能想知道为什么我们需要通过调整`sscratch`来切换到内核栈。

这是因为我们不能信任异常发生时的栈指针。在异常处理程序中，我们需要考虑以下三种模式：

1. 在内核模式下发生异常。
2. 在处理另一个异常时在内核模式下发生异常（嵌套异常）。
3. 在用户模式下发生异常。

在情况(1)中，即使我们不重置栈指针，通常也没有问题。在情况(2)中，我们会覆盖保存区域，但我们的实现在嵌套异常时会触发内核恐慌，所以没问题。

问题出在情况(3)。在这种情况下，`sp`指向"用户（应用程序）栈区域"。如果我们实现为按原样使用（信任）`sp`，可能会导致使内核崩溃的漏洞。

让我们通过在完成本书第17章的所有实现后运行以下应用程序来进行实验：

```c
// An example of applications
#include "user.h"

void main(void) {
    __asm__ __volatile__(
        "li sp, 0xdeadbeef\n"  // Set an invalid address to sp
        "unimp"                // Trigger an exception
    );
}
```

如果我们在不应用本章的修改（即从`sscratch`恢复内核栈）的情况下运行这个程序，内核会挂起而不显示任何内容，你会在QEMU的日志中看到以下输出：

```
epc:0x0100004e, tval:0x00000000, desc=illegal_instruction <- unimp triggers the trap handler
epc:0x802009dc, tval:0xdeadbe73, desc=store_page_fault <- an aborted write to the stack  (0xdeadbeef)
epc:0x802009dc, tval:0xdeadbdf7, desc=store_page_fault <- an aborted write to the stack  (0xdeadbeef) (2)
epc:0x802009dc, tval:0xdeadbd7b, desc=store_page_fault <- an aborted write to the stack  (0xdeadbeef) (3)
epc:0x802009dc, tval:0xdeadbcff, desc=store_page_fault <- an aborted write to the stack  (0xdeadbeef) (4)
...
```

首先，使用`unimp`伪指令发生无效指令异常，转换到内核的陷阱处理程序。然而，因为栈指针指向未映射的地址（`0xdeadbeef`），在尝试保存寄存器时发生异常，跳回到陷阱处理程序的开始。这变成了一个无限循环，导致内核挂起。为了防止这种情况，我们需要从`sscratch`获取一个可信的栈区域。

另一个解决方案是有多个异常处理程序。在RISC-V版本的xv6（一个著名的教育用UNIX类操作系统）中，有单独的异常处理程序用于情况(1)和(2)（[`kernelvec`](https://github.com/mit-pdos/xv6-riscv/blob/f5b93ef12f7159f74f80f94729ee4faabe42c360/kernel/kernelvec.S#L13-L14)）以及情况(3)（[`uservec`](https://github.com/mit-pdos/xv6-riscv/blob/f5b93ef12f7159f74f80f94729ee4faabe42c360/kernel/trampoline.S#L74-L75)）。在前一种情况下，它继承异常发生时的栈指针，在后一种情况下，它获取一个单独的内核栈。在进入和退出内核时[切换](https://github.com/mit-pdos/xv6-riscv/blob/f5b93ef12f7159f74f80f94729ee4faabe42c360/kernel/trap.c#L44-L46)陷阱处理程序。

> [!TIP]
>
> 在Google开发的操作系统Fuchsia中，有一个允许从用户设置任意程序计数器值的API[成为了一个漏洞](https://blog.quarkslab.com/playing-around-with-the-fuchsia-operating-system.html)。不信任来自用户（应用程序）的输入是内核开发中一个极其重要的习惯。

## 下一步

我们现在已经实现了同时运行多个进程的能力，实现了一个多任务操作系统。

然而，就目前而言，进程可以自由地读写内核的内存空间。这非常不安全！在接下来的章节中，我们将看看如何安全地运行应用程序，换句话说，如何隔离内核和应用程序。
