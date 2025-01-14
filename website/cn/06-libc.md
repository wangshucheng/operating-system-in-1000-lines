---
title: C标准库
---

# C标准库

在本章中，我们将实现基本类型和内存操作，以及字符串操作函数。在本书中，为了学习目的，我们将从头开始创建这些功能，而不是使用C标准库。

> [!TIP]
>
> 本章介绍的概念在C编程中非常常见，所以ChatGPT能提供可靠的答案。如果你在实现或理解任何部分时遇到困难，可以随时尝试询问它或联系我。

## 基本类型

首先，让我们在`common.h`中定义一些基本类型和便捷的宏：

```c [common.h] {1-15,21-24}
typedef int bool;
typedef unsigned char uint8_t;
typedef unsigned short uint16_t;
typedef unsigned int uint32_t;
typedef unsigned long long uint64_t;
typedef uint32_t size_t;
typedef uint32_t paddr_t;
typedef uint32_t vaddr_t;

#define true  1
#define false 0
#define NULL  ((void *) 0)
#define align_up(value, align)   __builtin_align_up(value, align)
#define is_aligned(value, align) __builtin_is_aligned(value, align)
#define offsetof(type, member)   __builtin_offsetof(type, member)
#define va_list  __builtin_va_list
#define va_start __builtin_va_start
#define va_end   __builtin_va_end
#define va_arg   __builtin_va_arg

void *memset(void *buf, char c, size_t n);
void *memcpy(void *dst, const void *src, size_t n);
char *strcpy(char *dst, const char *src);
int strcmp(const char *s1, const char *s2);
void printf(const char *fmt, ...);
```

这些大多数在标准库中都有，但我们添加了一些有用的类型：

- `paddr_t`：表示物理内存地址的类型。
- `vaddr_t`：表示虚拟内存地址的类型。相当于标准库中的`uintptr_t`。
- `align_up`：将`value`向上舍入到`align`的最近倍数。`align`必须是2的幂。
- `is_aligned`：检查`value`是否是`align`的倍数。`align`必须是2的幂。
- `offsetof`：返回结构体中成员的偏移量（从结构体开始的字节数）。

`align_up`和`is_aligned`在处理内存对齐时很有用。例如，`align_up(0x1234, 0x1000)`返回`0x2000`。同样，`is_aligned(0x2000, 0x1000)`返回true，但`is_aligned(0x2f00, 0x1000)`返回false。

每个宏中使用的以`__builtin_`开头的函数是Clang特定的扩展（内置函数）。参见[Clang内置函数和宏](https://clang.llvm.org/docs/LanguageExtensions.html)。

> [!TIP]
>
> 这些宏也可以在不使用内置函数的情况下用C实现。纯C实现的`offsetof`特别有趣 ;)

## 内存操作

接下来，我们实现以下内存操作函数。

`memcpy`函数将`src`中的`n`个字节复制到`dst`：

```c [common.c]
void *memcpy(void *dst, const void *src, size_t n) {
    uint8_t *d = (uint8_t *) dst;
    const uint8_t *s = (const uint8_t *) src;
    while (n--)
        *d++ = *s++;
    return dst;
}
```

`memset`函数用`c`填充`buf`的前`n`个字节。这个函数在第4章中已经实现过，用于初始化bss段。让我们把它从`kernel.c`移到`common.c`：

```c [common.c]
void *memset(void *buf, char c, size_t n) {
    uint8_t *p = (uint8_t *) buf;
    while (n--)
        *p++ = c;
    return buf;
}
```

> [!TIP]
>
> `*p++ = c;`在一个语句中完成指针解引用和指针操作。为了清晰起见，它相当于：
>
> ```c
> *p = c;    // 解引用指针
> p = p + 1; // 赋值后移动指针
> ```
>
> 这是C语言中的一种惯用写法。

## 字符串操作

让我们从`strcpy`开始。这个函数将字符串从`src`复制到`dst`：

```c [common.c]
char *strcpy(char *dst, const char *src) {
    char *d = dst;
    while (*src)
        *d++ = *src++;
    *d = '\0';
    return dst;
}
```

> [!WARNING]
>
> 即使`src`比`dst`的内存区域长，`strcpy`函数也会继续复制。这很容易导致bug和漏洞，所以通常建议使用其他替代函数而不是`strcpy`。永远不要在生产环境中使用它！
>
> 为了简单起见，我们在本书中使用`strcpy`，但如果你有能力，可以尝试实现并使用替代函数（`strcpy_s`）。

下一个函数是`strcmp`函数。它比较`s1`和`s2`并返回：

| 条件 | 结果 |
| --------- | ------ |
| `s1` == `s2` | 0 |
| `s1` > `s2` | 正值 |
| `s1` < `s2` | 负值 |

```c [common.c]
int strcmp(const char *s1, const char *s2) {
    while (*s1 && *s2) {
        if (*s1 != *s2)
            break;
        s1++;
        s2++;
    }

    return *(unsigned char *)s1 - *(unsigned char *)s2;
}
```

> [!TIP]
>
> 在比较时转换为`unsigned char *`是为了符合[POSIX规范](https://www.man7.org/linux/man-pages/man3/strcmp.3.html#:~:text=both%20interpreted%20as%20type%20unsigned%20char)。

`strcmp`函数通常用于检查两个字符串是否相同。这可能有点反直觉，但当`!strcmp(s1, s2)`为真（即函数返回零）时，字符串是相同的：

```c
if (!strcmp(s1, s2))
    printf("s1 == s2\n");
else
    printf("s1 != s2\n");
```
