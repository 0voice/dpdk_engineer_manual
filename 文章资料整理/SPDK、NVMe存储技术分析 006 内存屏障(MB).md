# [SPDK/NVMe存储技术分析\]006 内存屏障(MB)

在多核(SMP)多线程的情况下，如果不知道CPU乱序执行的话，将会是一场噩梦，因为无论怎么进行代码Review也不可能发现跟内存屏障(MB)相关的Bug。内存屏障分为两类：

- 跟编译有关的内存屏障: 告诉编译器，不要优化我，俺不需要
- 跟CPU有关的内存屏障: 告诉CPU, 不要乱序执行，谢谢

## **1. NVMeDirect中的内存屏障**

```
/* nvmedirect/include/lib_nvmed.h */

38 #define COMPILER_BARRIER() asm volatile("" ::: "memory")
```

由于NVMeDirect依赖于Linux内核的NVMe驱动(nvme.ko)实现，所以NVMeDirect并不需要实现它自己的与CPU相关的内存屏障。

## **2. SPDK中的内存屏障**

```
/* src/spdk-17.07.1/include/spdk/barrier.h */

47 /** Compiler memory barrier */
48 #define spdk_compiler_barrier() __asm volatile("" ::: "memory")
49
50 /** Write memory barrier */
51 #define spdk_wmb()              __asm volatile("sfence" ::: "memory")
52
53 /** Full read/write memory barrier */
54 #define spdk_mb()               __asm volatile("mfence" ::: "memory")
```

在SPDK中，不仅实现了与编译相关的内存屏障，还实现了与CPU有关的内存屏障。 但是， 在与CPU有关的MB中， 读内存屏障(Read memory barrier)并没有实现。

## **3. DPDK中的内存屏障**

在DPDK中，内存屏障的实现要复杂一点，因为支持x86, ARM和PowerPC三种平台。 以x86为例，代码实现如下：

- 与编译相关的MB

```
/* src/dpdk-17.08/lib/librte_eal/common/include/generic/rte_atomic.h */

132 /**
133  * Compiler barrier.
134  *
135  * Guarantees that operation reordering does not occur at compile time
136  * for operations directly before and after the barrier.
137  */
138 #define rte_compiler_barrier() do {             \
139         asm volatile ("" : : : "memory");       \
140 } while(0)
```

- 与CPU相关的MB

```
/* src/dpdk-17.08/lib/librte_eal/common/include/arch/x86/rte_atomic.h */

52 #define rte_mb()             _mm_mfence()
54 #define rte_wmb()            _mm_sfence()
56 #define rte_rmb()            _mm_lfence()

58 #define rte_smp_mb()         rte_mb()
60 #define rte_smp_wmb()        rte_compiler_barrier()
62 #define rte_smp_rmb()        rte_compiler_barrier()

64 #define rte_io_mb()          rte_mb()
66 #define rte_io_wmb()         rte_compiler_barrier()
68 #define rte_io_rmb()         rte_compiler_barrier()
```

另外，DPDK在对ARM32的MB支持中，使用了gcc的内嵌函数__sync_synchronize(), 例如：

```
/* src/dpdk-17.08/lib/librte_eal/common/include/arch/arm/rte_atomic_32.h */

52 #define rte_mb()  __sync_synchronize()
60 #define rte_wmb() do { asm volatile ("dmb st" : : : "memory"); } while (0)
68 #define rte_rmb() __sync_synchronize()
```

于是，让我们反汇编看看gcc的__sync_synchronize()到底是怎么回事。

```
$ cat -n foo.c
     1  int main(int argc, char *argv[])
     2  {
     3          int n = 0x1;
     4          __sync_synchronize();
     5          return ++n;
     6  }
$ gcc -g -Wall -m32 -o foo foo.c
$ gdb foo
...<snip>...
(gdb) disas /m main
Dump of assembler code for function main:
2       {
   0x080483ed <+0>:     push   %ebp
   0x080483ee <+1>:     mov    %esp,%ebp
   0x080483f0 <+3>:     sub    $0x10,%esp

3               int n = 0x1;
   0x080483f3 <+6>:     movl   $0x1,-0x4(%ebp)

4               __sync_synchronize();
   0x080483fa <+13>:    lock orl $0x0,(%esp)

5               return ++n;
   0x080483ff <+18>:    addl   $0x1,-0x4(%ebp)
   0x08048403 <+22>:    mov    -0x4(%ebp),%eax

6       }
   0x08048406 <+25>:    leave
   0x08048407 <+26>:    ret

End of assembler dump.

$ gcc -g -Wall -m64 -o foo foo.c
$ gdb foo
...<snip>...
(gdb) disas /m main
Dump of assembler code for function main:
2       {
   0x00000000004004d6 <+0>:     push   %rbp
   0x00000000004004d7 <+1>:     mov    %rsp,%rbp
   0x00000000004004da <+4>:     mov    %edi,-0x14(%rbp)
   0x00000000004004dd <+7>:     mov    %rsi,-0x20(%rbp)

3               int n = 0x1;
   0x00000000004004e1 <+11>:    movl   $0x1,-0x4(%rbp)

4               __sync_synchronize();
   0x00000000004004e8 <+18>:    mfence

5               return ++n;
   0x00000000004004eb <+21>:    addl   $0x1,-0x4(%rbp)
   0x00000000004004ef <+25>:    mov    -0x4(%rbp),%eax

6       }
   0x00000000004004f2 <+28>:    pop    %rbp
   0x00000000004004f3 <+29>:    retq

End of assembler dump.
```

因为没有ARM平台，就在x86上分别进行32位和64位的编译，于是发现__sync_synchronize()对应的汇编指令是

- 32位

```
4               __sync_synchronize();
   0x080483fa <+13>:    lock orl $0x0,(%esp)
```

- 64位

```
4               __sync_synchronize();
   0x00000000004004e8 <+18>:    mfence
```

关于**lock**指令前缀和**mfence**指令，后面再讲。

## **4. Linux内核中的内存屏障**

Linux内核支持很多种平台，这里仅以x86为例：

```
/* linux-4.11.3/arch/x86/include/asm/barrier.h */

13 #ifdef CONFIG_X86_32
14 #define mb()  asm volatile(ALTERNATIVE("lock; addl $0,0(%%esp)", "mfence", \
15                                        X86_FEATURE_XMM2) ::: "memory", "cc")
16 #define rmb() asm volatile(ALTERNATIVE("lock; addl $0,0(%%esp)", "lfence", \
17                                        X86_FEATURE_XMM2) ::: "memory", "cc")
18 #define wmb() asm volatile(ALTERNATIVE("lock; addl $0,0(%%esp)", "sfence", \
19                                        X86_FEATURE_XMM2) ::: "memory", "cc")
20 #else
21 #define mb()    asm volatile("mfence" ::: "memory")
22 #define rmb()   asm volatile("lfence" ::: "memory")
23 #define wmb()   asm volatile("sfence" ::: "memory")
24 #endif
```

## **5. 总结**

### **5.1 在x86_64平台上实现内存屏障（MB）**

从NVMeDirect到SPDK, 再到DPDK和Linux内核, 我们可以得出在x86_64平台上，与内存屏障（MB）有关的实现可归纳为：

- 与编译有关的MB实现

```
#define XXX_compiler_barrier()          asm volatile(""       ::: "memory")
```

- 与CPU有关的MB实现

```
#define XXX_mb                          asm volatile("mfence" ::: "memory")
#define XXX_rmb                         asm volatile("lfence" ::: "memory")
#define XXX_wmb                         asm volatile("sfence" ::: "memory")
```

其中，

- volatile是C语言的关键字，主要目的是告诉编译器不要做优化。 关于volatile的说明， 请参考[这里](http://www.cnblogs.com/idorax/p/7561793.html)。
- mfence是汇编指令，用于设定读写屏障（**M**emory）。有关mfence指令，请参考[这里](http://x86.renejeschke.de/html/file_module_x86_id_170.html)。
- lfence是汇编指令，用于设定读屏障 (**L**oad)。
- sfence也是汇编指令, 用于设定写屏障 (**S**tore)。

### **5.2 lock指令前缀**

lock指令前缀与原子操作有关。对于Lock指令前缀的总线锁，早期CPU芯片上有一条引线#HLOCK pin, 如果汇编语言的程序中在一条指令前面加上前缀"lock"(表示锁总线)，经过汇编以后的机器码就使CPU在执行这条指令的时候把#HLOCK pin的电平拉低，持续到这条指令结束时放开，从而把总线锁住，这样同一总线上的别的CPU就暂时不能通过总线访问内存了，保证了这条指令在多CPU环境中的原子性。

### **5.3 使用CPU内存屏障的根本原因**

在SMP(对称多处理器)中，CPU是多核的，每个核有自己的cache，读写内存都先通过cache。然而内存只有一个，核有多个，也就是说，同一份数据在内存中只有一份，但却可能同时存在于多个cache line中。那么，如何进行同步？ 答案就是原子操作，注意原子操作的前提是独占。假如一个变量X同时存在于核1和核2的cache line中，那么当核1想要对X进行"原子加(atomic add)"的时候必须先独占这个变量X，也就是告诉核2变量X的值在你的cache line已经失效了，以后想要操作X的时候到哥哥我这里来取最新的值。这看起来非常像锁，但是没有用到锁。(P.S.: 无锁队列的实现其实都离不开原子操作) 因此，我们可以这么认为，**内存屏障(mb, wmb, rmb)的本质是用来在CPU各个核的cache line中进行通信，保证内存数据的更新具有原子性**。

**扩展阅读：**

- Paper: [Memory Barriers: a Hardware View for Software Hackers](http://www.rdrop.com/users/paulmck/scalability/paper/whymb.2010.06.07c.pdf)
- Paper: [Mathematizing C++ Concurrency](http://www.cl.cam.ac.uk/~pes20/cpp/popl085ap-sewell.pdf)
- Wikipedia: https://en.wikipedia.org/wiki/Memory_barrier
- Blog: [巧夺天工的kfifo(修订版)](http://blog.csdn.net/linyt/article/details/53355355)
- Blog: [Linux 2.6内核中新的锁机制--RCU](https://www.ibm.com/developerworks/cn/linux/l-rcu/)





**转载自：**  **作者** [vlhn](https://home.cnblogs.com/u/vlhn/)    **本文链接**https://www.cnblogs.com/vlhn/p/7764908.html

