---
title: 系统调用和SYSCALL_DEFINE宏定义解析
date: 2018-03-07 14:06:29
tags: [操作系统]
---
## 用户态和内核态

为了减少有限资源的访问和使用冲突，Unix/Linux的设计哲学之一就是：对不同的操作赋予不同的执行等级，就是所谓**特权**的概念。
与系统相关的一些特别关键的操作必须由最高特权的程序来完成。Intel的X86架构的CPU提供了0到3四个特权级，数字越小，特权越高。

Linux操作系统中主要采用了0和3两个特权级，分别对应的就是**内核态**和**用户态**（用户空间和内核）。

- **内核**本质上看是一种软件--控制计算机的硬件资源，并提供商场应用程序运行环境。运行在内核态的进程则可以执行任何操作并且在资源的使用上没有限制。
- **用户态**即上层应用程序的活动空间，应用程序的执行必须依托于内核提供的资源 。用户态的进程可执行操作和访问的资源受到极大限制
- 为了使上层应用能够访问到这些资源，内核必须为上层应用提供访问的接口：即系统调用。**系统调用是操作系统最小的功能单位**

<!--more-->

当用户态的进程调用一个系统调用时，CPU切换到内核态并开始执行一个内核函数。在80x86体系结构中，可以用两种不同的方式调用Linux的系统调用，最终结果都是跳转到所谓的系统调用处理程序的汇编语言函数。
- 执行 int $0x80 汇编语言指令。
- 执行 sysenter 汇编语言指令。


退出系统调用：
- 执行 iret 汇编语言指令
- 执行 sysexit 汇编语言指令

从用户态到内核态的切换，一般存在以下三种情况：

1. 系统调用

2. 异常事件： 当CPU正在执行运行在用户态的程序时，突然发生某些预先不可知的异常事件，这个时候就会触发从当前用户态执行的进程转向内核态执行相关的异常事件，典型的如缺页异常。

3. 外围设备的中断：当外围设备完成用户的请求操作后，会像CPU发出中断信号，此时，CPU就会暂停执行下一条即将要执行的指令，转而去执行中断信号对应的处理程序，如果先前执行的指令是在用户态下，则自然就发生从用户态到内核态的转换。

**系统调用的本质其实也是中断，相对于外围设备的硬中断，这种中断称为软中断**

## SYSCALL_DEFINE
系统调用在内核中的入口都是sys_xxx，然而在阅读 epoll 源码的过程中，发现都是通过 **SYSCALL_DEFINE** 宏定义实现。故查阅相关资料如下：

```c
#define SYSCALL_DEFINE0(name)      asmlinkage long sys_##name(void)  
#define SYSCALL_DEFINE1(name, ...) SYSCALL_DEFINEx(1, _##name, __VA_ARGS__)  
#define SYSCALL_DEFINE2(name, ...) SYSCALL_DEFINEx(2, _##name, __VA_ARGS__)  
#define SYSCALL_DEFINE3(name, ...) SYSCALL_DEFINEx(3, _##name, __VA_ARGS__)  
#define SYSCALL_DEFINE4(name, ...) SYSCALL_DEFINEx(4, _##name, __VA_ARGS__)  
#define SYSCALL_DEFINE5(name, ...) SYSCALL_DEFINEx(5, _##name, __VA_ARGS__)  
#define SYSCALL_DEFINE6(name, ...) SYSCALL_DEFINEx(6, _##name, __VA_ARGS__) 
```

```c
#define __SYSCALL_DEFINEx(x, name, ...)                                 \
        asmlinkage long sys##name(__SC_DECL##x(__VA_ARGS__));           \
        static inline long SYSC##name(__SC_DECL##x(__VA_ARGS__));       \
        asmlinkage long SyS##name(__SC_LONG##x(__VA_ARGS__))            \
        {                                                               \
                __SC_TEST##x(__VA_ARGS__);                              \
                return (long) SYSC##name(__SC_CAST##x(__VA_ARGS__));    \
        }                                                               \
        SYSCALL_ALIAS(sys##name, SyS##name);                            \
        static inline long SYSC##name(__SC_DECL##x(__VA_ARGS__))

```

**SYSCALL_DEFINEx** 里面的x代表的是系统调用参数个数。sys_epoll_create1 的宏定义为 
```c
SYSCALL_DEFINE1(epoll_create1, int, flag)

//展开后得：
SYSCALL_DEFINEx(1, _epoll_create1, int, flag)

//继续展开得：        

    asmlinkage long sys_epoll_create1(__SC_DECL1(int, flag));       \  
    static inline long SYSC_epoll_create1(__SC_DECL1(int, flag));   \  
    asmlinkage long SyS_epoll_create1(__SC_LONG1(int ,flag))        \  
    {                               \  
        __SC_TEST1(int ,flag);              \  
        return (long) SYSC_epoll_create1(__SC_CAST1(int ,flag));    \  
    }                               \  
    SYSCALL_ALIAS(sys_epoll_create1, SyS_epoll_create1);                \  
    static inline long SYSC_epoll_create1(__SC_DECL1(int ,flag)) 

```
##是连接符，__VA_ARGS__代表前面...里面的可变参数

其中 **__SC_DECLx**,**__SC_LONGx**,**_SC_CASTx**,**_SC_CASTx** 等宏定义如下：
```c

//设置别名 
#define SYSCALL_ALIAS(alias, name)                  \  
    asm ("\t.globl " #alias "\n\t.set " #alias ", " #name "\n"  \  
         "\t.globl ." #alias "\n\t.set ." #alias ", ." #name) 
         
//转换为定义形参形式
#define __SC_DECL1(t1, a1)  t1 a1   
#define __SC_DECL2(t2, a2, ...) t2 a2, __SC_DECL1(__VA_ARGS__)  
#define __SC_DECL3(t3, a3, ...) t3 a3, __SC_DECL2(__VA_ARGS__)  
... 
// 全部转换为 long 类型
#define __SC_LONG1(t1, a1)  long a1  
#define __SC_LONG2(t2, a2, ...) long a2, __SC_LONG1(__VA_ARGS__)  
#define __SC_LONG3(t3, a3, ...) long a3, __SC_LONG2(__VA_ARGS__)  
... 
// 根据 tx 转换为对应类型 
#define __SC_CAST1(t1, a1)  (t1) a1  
#define __SC_CAST2(t2, a2, ...) (t2) a2, __SC_CAST1(__VA_ARGS__)  
#define __SC_CAST3(t3, a3, ...) (t3) a3, __SC_CAST2(__VA_ARGS__)  
...

#define __SC_TEST(type)     BUILD_BUG_ON(sizeof(type) > sizeof(long))  
#define __SC_TEST1(t1, a1)  __SC_TEST(t1)  
#define __SC_TEST2(t2, a2, ...) __SC_TEST(t2); __SC_TEST1(__VA_ARGS__)  
#define __SC_TEST3(t3, a3, ...) __SC_TEST(t3); __SC_TEST2(__VA_ARGS__)  
...

//其作用是在编译的时候如果condition为真，则编译char[1-2]出错
#define BUILD_BUG_ON(condition) ((void)sizeof(char[1 - 2*!!(condition)]))  
```
**将系统调用的参数统一变为了使用long型来接收，再强转转为int，也就是系统调用本来传下来的参数类型**

**多一次强制转换的意义在于Linux CVE-2009-0029漏洞，64位平台的ABI要求在系统调用时，用户空间程序将系统调用中32位的参数存放在64位的寄存器中要做到正确的符号扩展，但是用户空间程序却不能保证做到这点，这样就会可以通过向有漏洞的系统调用传送特制参数便可以导致系统崩溃或获得权限提升。**

参考链接：https://access.redhat.com/security/cve/cve-2009-0029

### asmlinkage的作用

asmlinkage 的定义 (/usr/include/asm/linkage.h里面)：
```c
#define asmlinkage CPP_ASMLINKAGE __attribute__((regparm(0)))
/* __attribute__是关键字，是gcc的C语言扩展，regparm(0)表示不从寄存器传递参数;

如果是__attribute__((regparm(3)))，那么调用函数的时候参数不是通过栈传递，而是直接放到寄存器里，被调用函数直接从寄存器取参数。 */

```

汇编语言的主程序与子程序之间的参数传递方式:
- **寄存器法**：寄存器法就是将入口参数和出口参数存放在约定的寄存器中。
- **约定单元法**：约定单元法顾名思义是吧入口参数和出口参数都放在事先约定好的单元中
- **堆栈法**：堆栈法是利用堆栈来传递参数的。
- **地址表法**：这种方法是把参数组成的一张参数表放在某个存储区中，然后只要主程序和子程序约定好这个存储区的首地址和存放的内容，在主程序中将参数传递给地址表，在子程序中根据地址表给定的参数就可以完成操作。