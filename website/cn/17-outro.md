---
title: 结语
---

# 结语

恭喜！你已经完成了本书的学习。你已经学会了如何从零开始实现一个简单的操作系统内核。你学习了操作系统的基本概念，如 CPU 引导、上下文切换、页表、用户模式、系统调用、磁盘 I/O 和文件系统。

虽然代码不到 1000 行，但这一定是一个相当具有挑战性的过程。这是因为你构建了内核中最核心的核心部分。

对于那些还想继续深入学习的人，以下是一些后续步骤：

## 添加新功能

在本书中，我们实现了内核的基本功能。然而，还有许多功能可以实现。例如，以下这些功能都很有趣：

- 一个支持内存释放的完整内存分配器。
- 中断处理。不要对磁盘 I/O 使用忙等待。
- 一个功能完备的文件系统。实现 ext2 将是一个很好的起点。
- 网络通信（TCP/IP）。实现 UDP/IP 并不难（TCP 相对较高级）。Virtio-net 的实现与 virtio-blk 非常相似！

## 阅读其他操作系统的实现

最推荐的下一步是阅读现有操作系统的实现。将你的实现与其他实现进行比较，学习他人的实现方式是非常有教育意义的。

我最喜欢的是 [RISC-V 版本的 xv6](https://github.com/mit-pdos/xv6-riscv)。这是一个用于教育目的的类 UNIX 操作系统，它附带了一本[解释性的书籍（英文）](https://pdos.csail.mit.edu/6.828/2022/)。对于那些想要学习 UNIX 特定功能（如 `fork(2)`）的人来说，这是很推荐的资源。

另一个是我的项目 [Starina](https://starina.dev)，这是一个用 Rust 编写的基于微内核的操作系统。这个项目仍然处于实验阶段，但对于那些想要学习微内核架构以及 Rust 在操作系统开发中的优势的人来说，这将是一个有趣的参考。

## 欢迎反馈！

如果你有任何问题或反馈，请随时在 [GitHub](https://github.com/nuta/operating-system-in-1000-lines/issues) 上提出，或者如果你愿意，也可以[给我发邮件](https://seiya.me)。祝你在操作系统编程的无尽旅程中快乐！
