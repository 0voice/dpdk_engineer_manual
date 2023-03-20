# AddressSanitizer，增强DPDK内存检测

## 1.**前言**

当我们使用C/C++构建我们程序的时候，经常会遇到这类问题：问题难以复现；大压力下运行几天才能出现；大并发问题下，出现问题现场无法分析定位问题。经过大量的分析总结，发现这类难以解决的问题的原因大量集中在内存越界、内存重复释放等内存问题。



这类问题大致由于分配给一个对象的内存，可能被其他程序重写，或者是由于某些计算错误，用了不属于你的内存。更崩溃的是如果它带有一定的随机性重现，当错误发生的时候，系统没有检测到异常，而是在以一定概率被其他程序重写或复用，问题发生了，这时候问题发生的地方与真正出问题的地方相隔很远。内存方面的问题是出了名的难题，而且一旦发生，影响巨大，甚至触发安全问题。



现有的DPDK已经有内存管理的机制，是通过检测可用内存前后区域是否被修改来判断是否越界，但是现有的DPDK实现只有在rte_free的时候才做内存检测，甚至发生内存问题的时候, 非常少且有效的调试信息输出来帮助定位问题，如果问题带有一定的随机性且很难重现，让人很崩溃。这时候会迫切地需要一种工具来帮助定位调试问题，网上有很多内存管理的工具，五花八门，但是没有一个可用并且好用的工具可以在DPDK中直接使用。



## 2.**内存检测机制**

大多数的内存检测工具采用以下步骤进行检测，比如通过在被保护的堆，栈、全局变量周围建立标记为中毒状态的红区（redzones），如**图1**所示，在绿色区域表示可访问的进程内存，红色区域表示在它周边插入的红区。



检测工具会检测进程中的所有位置，通过影子内存的状态值来存储真实内存中的每个字节的访问是否安全，这样的影子内存需要一大块连续的虚拟地址空间，如图1所示蓝色的影子区域（shadow region），在程序初始化的时候申请好这片空间。当然了查找影子内存需要非常快才行，一般会构建查找表(lookup table)来记录快速查找每块区域的访问权限。这个查找表并未真正分配，而是在进程启动的时候保存，在需要的时候访问。编译的时候，在每一次内存访问前编译器会帮我们采用代码插桩技术来针对应用程序的每次load和store对影子内存进行检查，它会影响每一次内存访问，并在代码前缀加上检测，检测对应的影子区域的状态值是否是可访问状态。



动态库可以在程序运行的时候做实时检测，如果影子内存的状态值是在可访问范围内, 则允许你继续，如果是在不可访问范围内, 则内存中毒，它会跟踪程序，并生成诊断报告。 



![图片](https://mmbiz.qpic.cn/mmbiz_png/4amQNRETabmdE6JiaeCD9iblPEJuNSibqiahAuskBdrKkaUlTcApceZ5vEXjibZGzGX63HVvZRmzb7v8jIKhTcp9GSA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

图1



### 2.1**AddressSanitizer原理及在DPDK中实现**

AddressSanitizer(ASan)是google开发的内存检测工具，检测方法与上述类似，突出的是它使用非常高效的映射机制和紧凑的影子编码方法来提高效率。



注意到malloc函数的返回地址基本是以8字节对齐的, 所以对于堆上的8字节内存空间, 只有9种状态, 而9种状态只需要1个字节就可以表示, 也就是说只使用1/8的虚拟地址空间的影子内存就可以描述所有的地址。如**图2**所示，左边是进程内存，右边是影子区域内存，绿色表示可以访问，红色表示不可以访问。 



- 当进程中的8个字节都是绿色的，也就是全部可以访问，那在影子区域存储的状态值是0；
- 如果是进程中的8个字节都是红色，如最后一行所示，它是都不能访问的，那么它就是负值。采用不同的负值来表示不同类型的不可访问地址，程序中是以“f”开头表示负数，比如“fa”是堆，“f1”是栈；
- 还有部分区域可以访问，部分不可以访问，也就是图2所示的中间部分，前N个字节可以访问，那么它的影子状态值是N，剩下的8-N就是不可以访问的。



这种映射机制可以构建高效的查找方式，通过原始指针的值除以8，再添加offset，也就是在内存影子的位置添加。计算公式是ShadowAddr = (Addr >> 3) + offset。



![图片](https://mmbiz.qpic.cn/mmbiz_png/4amQNRETabmdE6JiaeCD9iblPEJuNSibqiahboI25IvM8F2heqUUm8K8fCXadynL2iaFNSgibd2F5PfDjAqHTicv1xkUg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

图2



### 2.2**插桩**

前面介绍的在每一次内存访问前编译器会帮我们在代码前缀加上检测，下面是8 bytes内存访问代码。



原始代码如蓝色所示，红色代码是编译器插入的指令。既然是访问8 bytes，必须要保证对应的影子内存的状态值必须是0，否则肯定是有问题。



那么如果访问的是1,2 或4 bytes，该如何检查呢？如**图3**所示，只需要修改一下if判断条件即可，这里不做过多算法推算。



if (*shadow && *shadow < ((unsigned long)addr & 7) + N); //N = 1,2,4



![图片](https://mmbiz.qpic.cn/mmbiz_png/4amQNRETabmdE6JiaeCD9iblPEJuNSibqiahLRl34dWHGgLQeNnYfqxBwgRwF9rkcqoZfhZaGwPPH2hlG6UZ5VkpMw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

图3



### 2.3**动态运行库(libasan.so)**

在应用程序启动时，整个影子空间将会被分配，以保证程序其他部分无法使用它。每种平台，定义了推荐的默认的常数补偿，在linux x86_64上，shadow的offset是0x00007fff8000。



运行时，会对glibc中的函数进行替换以便检测内存错误，即运行时插桩，malloc和free将会被特定的实现替换掉，malloc会在返回的内存区域周围，设置红区，红区将会被标记为不可访问的或者说是有毒的(poisoned)。



Free函数会对整个内存区域染毒，同时将它放到隔离区(quarantine)。



默认情况下，malloc和free会记录当前的调用栈以为问题发生的时候提供更多日志信息，它可以快速定位到内存是什么时候怎么样被malloc或free的。



当发现内存出错，会使用启发法，来关联错误地址到有效地址的对象，记录并汇报错误信息。



### 2.4**ASan在DPDK中实现**

Glibc是linux 系统中最底层的API，ASan是基于glibc 实现，但向系统申请的内存DPDK和glibc采用不同的管理方式，对外接口完全不一样。所以动态运行库无法挂住DPDK的内存申请接口，就无法设置红区，并无法将其加入影子内存。



对内存是否可访问是编译器在插桩阶段完成，DPDK常用的编译器GCC 和CLANG 默认可以支持ASan，只要编译时添加ASan编译选项即可，而且ASan支持Linux i386/x86_64, MacOS, Android, Windows…多种架构和系统，但是目前我们只在Linux x86_64 平台上做过开发和验证。DPDK需要做的是在申请内存时在内存前后添加红区，将其加入影子区域，即在malloc和free相关函数中实现，将其设置为不可访问，配置并编译出DPDK专属的动态运行库。ASan相关代码已经合入[**DPDK 21.11**](http://mp.weixin.qq.com/s?__biz=MzI3NDA4ODY4MA==&mid=2653338297&idx=1&sn=779ce0d754b8e71ca86247c92dd2169e&chksm=f0cb4d3ec7bcc42892614467845309372b4a59f7e674896144ed7590eda275b89a6ad48ce6aa&scene=21#wechat_redirect)，详情请下载代码查看。



![图片](https://mmbiz.qpic.cn/mmbiz_gif/ibI6CAibeeVqJBfic2TQPEhYJ9njd1shnwPj7kIoZXG62tu9spDFqe34twFibrfdlsNuNSMM3ILupZ1ibCPQEwj6icTA/640?wx_fmt=gif&wxfrom=5&wx_lazy=1)

### 2.5**如何在DPDK中启用ASan工具检测内存？**

这个应该是大家最关心的问题。



Meson 方式编译DPDK代码的时候，添加“-Db_sanitize=address”编译选项便可开启ASan工具。



由于ASan集成到CLANG编译器时，要求链接的代码允许加上undefined symbols，因此CLANG要求加上“-Db_lundef=false”选项, 如果采用GCC编译方式可以不加这个选项。



“-Dbuildtype=debug”选项可以帮助输出更多的日志信息帮助定位调试，建议加上这个选项。 



总而言之，meson编译DPDK代码时，加上“-Dbuildtype=debug -Db_lundef=false -Db_sanitize=address”编译选项，便可启用ASan工具。



目前DPDK ASan支持检测缓存溢出(堆，栈，全局变量)，释放后使用(堆，栈)等内存问题。以下是heap-buffer-overflow的检测信息，问题发生的时候， 它会打印出哪个文件，哪行代码发生溢出，并有详细的函数调用过程、位置信息等帮助定位并高效解决问题。



![图片](https://mmbiz.qpic.cn/mmbiz_png/4amQNRETabmdE6JiaeCD9iblPEJuNSibqiahValX9xdSsQHLicP8xvibM1YQxZWmf9OPuVRrwK28vaLkZLnjRJ88E6sw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

图4



当然了，ASan启用运行会带来一定的性能下降和内存的开销，建议在代码调试和集成测试的时候使用。Intel DPDK测试部门已经启用ASan工具并部署到平常的自动化测试中，发现了许多静态工具无法检测的内存问题，目前这些问题已经全部记录并且修复。CI系统也将ASan内存检测加入到DPDK代码的测试中。



还等什么？赶紧行动起来吧！体验ASan工具带来的高效、便捷~ ~

原文链接：https://mp.weixin.qq.com/s/3YHEv2o_i7r4JFRt8EuXiQ

原文作者：林雪琴