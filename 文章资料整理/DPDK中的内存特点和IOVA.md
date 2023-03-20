# DPDK中的内存特点和IOVA

## 1.**大页**

 DPDK通常是使用大页（hugepage）内存的，无论是2M的大页还是1G的大页，本质上都是为了减少TLB miss，通过更大的page size来提升TLB的命中率，而TLB就是用来缓存页表的高速缓存。

## **2.DMA**

我们知道计算机的设备，如网卡硬件是不能处理用户空间的虚拟地址（只有CPU通过页表转换MMU才能识别虚拟地址），因为它不能感知任何用户态的进程和其所分配到的用户空间虚拟地址。相反，它只能访问真实的物理地址上的内存。

出于对效率的考量，现代硬件几乎总是使用直接内存存取（DMA）事务。通常，为了执行一个DMA事务，内核需要参与创建一个支持DMA的存储区域，将进程内虚拟地址转换成硬件能够理解的真实物理地址，并启动DMA事务。这是大多数现代操作系统中输入输出的工作方式；然而，这是一个耗时的过程，需要上下文切换、转换和查找操作，这不利于高性能输入/输出。 

DPDK的内存管理以一种简单的方式解决了这个问题。每当一个内存区域可供DPDK使用时，DPDK就通过询问内核来计算它的物理地址，即DPDK维护了虚拟地址和物理地址的一个映射关系，由于DPDK使用锁定（如vfio_pin_map_dma）内存（防止物理内存和虚拟内存的映射关系变化），来使底层内存区域的物理地址不会改变，因此硬件可以依赖这些物理地址始终有效，即使内存本身有一段时间没有使用。然后，DPDK会在准备由硬件完成的输入/输出事务时使用这些物理地址，并以允许硬件自己启动DMA事务的方式配置硬件。这使DPDK避免不必要的开销，并且完全从用户空间执行输入/输出。 

## 3.**IOMMU和IOVA**

默认情况下，任何硬件都可以访问整个系统的物理内存，因此它可以在任何地方执行DMA 事务。这有许多安全隐患。例如，流氓和/或不可信进程(包括在VM (虚拟机)内运行的进程)可能使用硬件设备来读写内核空间，和几乎其他任何存储位置。为了解决这个问题，现代系统配备了输入输出内存管理单元(IOMMU)。这是一种硬件设备，提供DMA地址转换和设备隔离功能，因此只允许特定设备执行进出特定内存区域(由IOMMU指定)的DMA 事务，而不能访问指定访问之外的系统内存地址空间。  

由于IOMMU的参与，**硬件使用的物理地址可能不是真实的物理地址，而是IOMMU分配给硬件的(完全任意的)输入输出虚拟地址(IOVA)**。一般来说，DPDK社区可以互换使用物理地址和IOVA这两个术语，但是根据上下文，这两者之间的区别可能很重要。例如，DPDK 17.11和更新的DPDK长期支持(LTS)版本在某些情况下可能根本不使用实际的物理地址，而是使用用户空间虚拟地址来实现DMA。IOMMU负责地址转换，因此硬件永远不会注意到两者之间的差异。 

![图片](https://mmbiz.qpic.cn/mmbiz_png/nrgibEGYIoVIfmvxSqvyLogNGTvDO8HXrbDib2uBdneEiaiamHnkHgodHVKqXQyTJ97ib2ZQC36kbjiaWZBbV4A3wiaVw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

 如上图所示，P1和P2进程分别用虚拟地址P进行DMA，由于IOMMU将虚拟地址P映射到不同的物理地址，所以这并不会冲突，而P3访问的地址P并没有在IOMMU中建立映射关系，所以DMA访问会导致失败。

## **4.IO虚拟地址（IOVA）模式**

DPDK是一个用户态应用框架，使用DPDK的软件可以像其他软件一样使用常规虚拟地址。但除此之外，DPDK还提供了用户态PMD和一组API，以实现完全以用户态执行IO操作。前文也已经提到过，硬件不能识别用户空间虚拟地址;它使用的是IO地址——物理地址（PA）或IO虚拟地址（IOVA）。 

DPDK API对物理和IO虚拟地址不作区分，即使不是由IO内存管理单元（IOMMU）提供VA部分，也都以IOVA来代表两种地址。但DPDK却会区分物理地址作为IOVA的情况，和用户空间虚拟地址作为IOVA的情况。它们在DPDK API中被称为IOVA模式，可分为两种：PA作为IOVA模式，和VA作为IOVA模式。 

## **5.PA作为IOVA模式**

PA作为IOVA的模式下，分配到整个DPDK存储区的IOVA地址都是实际的物理地址，而虚拟内存的分配与物理内存的分配相匹配。该模式的一大优点就是它很简单：它适用于所有硬件（也就是说，不需要IOMMU），并且它适用于内核空间（将真实物理地址转换为内核空间地址的开销是微不足道的）。实际上，这就是DPDK长期以来的运作方式，在很多方面它都被认为是默认的选项。 

然而，作为PA的IOVA模式也存在一些缺点。其中一个就是它需要根用户特权——如果无法访问系统的页面映射（因为DPDK要维护虚拟地址和物理地址IOVA的对应关系），DPDK就无法获取内存区域的真实物理地址。因此，如果系统中没有root权限，就无法以该模式运行。 

![图片](https://mmbiz.qpic.cn/mmbiz_png/nrgibEGYIoVIfmvxSqvyLogNGTvDO8HXrAnPe6icpR87EWqfCJryW102FwVughj6Zxzt3Rn2uiazcZGeIdW7O7VHQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

PA作为IOVA的模式

PA作为IOVA的模式还有另外一个值得一提的限制——**虚拟内存分配要遵循物理内存分配。这意味着如果物理内存空间被分段（被分成许多小段而不是几个大段）时，虚拟内存空间也要遵循同样的分段。极端情况下，分段可能过于严重，导致被分割出来物理上连续的片段数量过多，耗尽DPDK用于存储这些片段相关信息的内部数据结构，就会让DPDK初始化失败**。

![图片](https://mmbiz.qpic.cn/mmbiz_png/nrgibEGYIoVIfmvxSqvyLogNGTvDO8HXr0KSic7aYKcBtrszAMQXRYBBlFDvfKMT9fcMQ3aias8G55tfs46VPx3sw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

作为PA的IOVA模式下PA分段示例。

应对这些问题，DPDK社区提出了解决方法。举例来说，一种减少分段影响的方式是使用更大的分页——问题虽然没被解决，但是单独的1千兆字节（GB）段比独立的2兆字节（MB）段能大幅度减小分段的数量。另外一种广泛使用的解决方式则是在启动时引导系统并保留大页，而不是在运行时。但上述的解决方法都不能根本地解决问题，而且整个DPDK社区都习惯了要去解决这些问题，每个DPDK用户（有意或无意）在使用时都会采取相同的思维模式——“我需要X MB内存，但以防万一，我要保留X + Y MB！”

## **6.VA作为IOVA模式**

相比之下，VA作为IOVA的模式不需遵循底层物理内存的分布。而是重新分配物理内存，与虚拟内存的分配匹配。DPDK EAL依靠内核基础设施来实现这一点。内核基础设施又反过来使用IOMMU重新映射物理内存。 

![图片](https://mmbiz.qpic.cn/mmbiz_png/nrgibEGYIoVIfmvxSqvyLogNGTvDO8HXr6RC3zFz9V7DDLs9fVd8vGfMpGcPf48IM5P7ziaa3jx2Biapem3w92vkA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

作为VA的IOVA模式。

这种方式的优点显而易见：VA作为IOVA的模式下，所有内存都是VA和IOVA连续的。这意味着所有需要大量IOVA连续内存的内存分配更有可能成功，因为对硬件来说，即使底层物理内存可能不存在，内存看上去还是IOVA连续的。由于重新映射，IOVA空间片段化的问题就变得无关紧要。不管物理内存被分段得多么严重，它总能被重新映射为IOVA-连续的大块内存。  

![图片](https://mmbiz.qpic.cn/mmbiz_png/nrgibEGYIoVIfmvxSqvyLogNGTvDO8HXrW1XEeRicD45hpFn8AAibxKIvZgGCn04AZE7rN9HYicQrY8BlLxv3fxARQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)



作为VA的IOVA模式下的分段示例。

VA作为IOVA的模式还有另一个优点，它不需要任何权限。这是因为它不需要访问系统页面映射。这样就可以允许以非root用户身份运行DPDK，而且在特权访问不受欢迎的环境中，如云原生环境就可以更加容易地使用DPDK。

当然, 作为VA的IOVA模式也有一个缺点。出于各种原因，有时候可能不能选择使用IOMMU。这种情况可能包括： 

\1) 硬件不支持IOMMU；

\2) 平台可能本身就没有IOMMU（比如没有IOMMU模拟的VM）；

\3) 软件设备（例如，DPDK的内核网络接口（KNI）PMD）不支持作为VA的IOVA模式；

\4) 一些IOMMU（通常是模拟的IOMMU）的地址宽度可能有限，虽然这不妨碍用作VA的IOVA模式，但限制了其有效性；

\5) 在非Linux *的操作系统上使用DPDK；

但是，这些情况还是相对较少，绝大多数情况下，VA作为IOVA的模式都可以正常工作。

## 7.**IOVA模式的选择**

很多情况下，DPDK默认选择作为PA的IOVA模式，因为从硬件角度这是最安全的模式。所有给定的硬件（或软件）PMD至少都可以保证支持作为PA的IOVA模式。尽管如此，如果条件允许，还是强烈建议所有DPDK用户使用作为VA的IOVA模式，毕竟此模式具有不可否认的优势。 

但是，用户不必非要在两者中选择一个。可以自动检测出最合适的IOVA模式，而且默认选项绝对适用于大多数情况，因此不需要用户来做此选择。如果默认选项并不合适，用户可以使用--iova-mode EAL命令行参数尝试使用EAL标志（适用于DPDK 17.11及更高版本）来代替IOVA模式： 

1 ./app --iova-mode=pa # use IOVA as PA mode

2./app --iova-mode=va # use IOVA as VA mode

 大多数情况下，VA和PA模式不会互相排斥，可以使用任一模式，但在某些情况下，PA作为IOVA的模式是唯一可用的选择。当不能使用作为VA模式的IOVA时，即使EAL参数要求使用作为VA模式的IOVA，DPDK也会自动切换为作为PA模式的IOVA。DPDK还提供了一个API，可查询运行时正在使用的IOVA模式，但通常这不会在用户应用中使用，因为只有像是DPDK PMD和总线驱动程序才会要求获取这种信息。

## 8.**UIO和VFIO对IOVA的支持**

**l** **Igb_uio**

DPDK代码库中最早的内核驱动程序是igb_uio驱动程序。在DPDK最初的发展阶段，这个驱动程序就已经存在了，因此它是DPDK开发人员使用最广泛也是最熟悉的驱动程序。

此驱动程序依赖内核用户空间IO（UIO）基础结构运作，并为所有中断类型（INT、消息信号中断（MSI）和MSI-X）提供支持，以及创建虚拟功能。它还公开硬件设备通过/dev/uio文件系统注册和中断句柄，然后DPDK EAL将它们用于将它们映射到用户空间并使它们可用于DPDK PMD。

  gb_uio驱动程序非常简单，能做的也并不多，因此它不支持使用IOMMU。或者，更确切地说，它确实支持IOMMU，但仅在传输模式下，它在IOVA和物理内存地址之间建立1：1映射。igb_uio不支持使用完整的IOMMU模式。因此，**igb_uio驱动程序仅支持****PA作为I****OVA模式，并且根本无法在IOVA中作为VA模式工作**。 

类似于igb_uio的驱动程序在内核中可用：uio_pci_generic。它的工作方式与igb_uio非常相似，只是它的功能更加有限。例如，igb_uio支持所有中断类型（传统，MSI和MSI -X），而uio_pci_generic只支持传统中断。更重要的是，igb_uio可以创建虚拟函数(Virtual Function, VF)，而uio_pci_generic则不能;因此，如果在使用DPDK物理函数(Physical Function, PF)驱动程序时创建VF是必需的一步，igb_uio是唯一的选择。  

因此，在大多数情况下，igb_uio与uio_pci_generic相同或更可取。关于使用IOMMU的所有限制同样适用于igb_uio和uio_pci_generic驱动程序 - 它们不能使用完整的IOMMU功能，因此仅支持IOVA作为PA模式。

**l** **vfio**

vfio-pci驱动是虚拟功能I / O（VFIO）内核基础结构的一部分，并在Linux 3.6版中引入。VFIO使设备寄存器和设备中断可供用户空间应用程序使用，并可使用IOMMU设置IOVA映射以从用户空间执行IO。后一部分至关重要，此驱动程序专为与IOMMU一起使用而开发，在较旧的内核上，如果没有启用IOMMU，它甚至都无法工作。 

与直观看法相反，**使用VFIO驱动程序允许使用PA****作为IOVA****和****VA作为****IOVA模式**。这是因为，虽然建议使用IOVA作为VA模式来利用该模式的所有好处，但没有什么能阻止DPDK的设置IOMMU映射的EAL以遵循物理内存布局1：1的方式;毕竟IOVA映射是任意的。在这种情况下，即使使用IOMMU，DPDK也可以在IOVA中作为PA模式工作，从而允许DPDK KNI等工作。但是，仍然需要root权限才能将IOVA用作PA模式。

在更新的内核（4.5+，向后移植到一些旧版本）上，有一enable_unsafe_noiommu_mode选项，允许在没有IOMMU的情况下使用VFIO。这种模式适用于与基于UIO的驱动程序相同的所有意图和目的，并具有所有相同的优点与限制。

## **9.及早期版本**

使用DPDK库17.11版本或更早版本的任何应用程序必须事先知道其内存要求。这是因为，对于这些版本的DPDK，在初始化之后不可能再申请额外的大页内存，或者将其释放回系统。因此，DPDK应用程序可能使用的任何内存都必须在应用程序初始化时预留，并在应用程序的整个生命周期都由DPDK保留。

在决定要保留的内存量时，留出一些余量通常是个好主意。在各种内部分配中，某些DPDK内存将在不同的内部分配中被“浪费”，数量会因您的配置而异（DPDK将使用的设备数量，启用功能等）。

此外，DPDK17.11中的大多数API都需要大量的IOVA连续内存。这是因为在DPDK 17.11中，虚拟内存布局总是与物理内存布局相匹配。换句话说，如果使用PA作为IOVA，则要求PA是连续的。这是DPDK17.11内存管理中众所周知的问题之一：实际上很少有应用程序需要PA连续内存，由于缺少足够的IOVA连续内存，分配大量内存可能会失败。

![图片](https://mmbiz.qpic.cn/mmbiz_png/nrgibEGYIoVIfmvxSqvyLogNGTvDO8HXrVicZxaiaTS0M89f9YxGgt1hEQCxZFUFBICDLqZ4aTkBZD1tibXUyv0Zyw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

IOVA模式的比较

上述限制当然仅适用于作为物理地址（PA）模式的IOVA，因为在该模式下，DPDK的虚拟地址（VA）空间遵循PA空间的布局。在PA模式的IOVA中，可用IOVA连续内存量取决于DPDK控制之外的许多因素，尽管DPDK将尝试保留尽可能多的IOVA连续内存，具体取决于可用内存量和系统配置，可能没有足够的IOVA连续内存来满足所有分配。

在VA模式的IOVA中，这不是问题，因为在这种情况下，IOVA空间布局将与VA空间的布局相匹配（而不是相反），并且所有物理内存都被重新映射为IOVA连续到硬件。

## **10.及之后版本**

## 11.**动态内存管理**

**DPDK 18.11的最大变化是可以在运行时增加和减少内存使用量**，从而消除了DPDK内存映射为静态的情况，这带来了许多可用性方面的改进。

在DPDK 17.11中，运行没有任何环境抽象层（EAL）参数的DPDK应用会保留所有可用的大页内存供其使用，且不会为其他应用程序或其他DPDK实例留下任何大页内存。对于DPDK18.11而言情况有所不同，DPDK仅保留应用运行所必须的内存量。在这种意义上，DPDK现在性能更好，并且使DPDK与其他应用程序完美配合所需的工作也更少。

同样，不再需要事先知道应用程序的内存需求，DPDK的内存映射可以动态增加和减少，因此DPDK内存子系统可以根据需要自动增加其内存使用量，并在不再需要时将内存返回给系统。这意味着部署DPDK应用程序所需的工作更少，因为现在DPDK可以自行管理其内存需求。**DPDK 18.11和IOVA连续性**

DPDK 18.11中的一项基本后台更改是，不能保证虚拟地址（VA）连续内存是IOVA连续的；两者之间不再有任何关联。**在VA作为IOVA的模式下，IOVA布局仍然像以前一样遵循VA布局，但是在PA作为IOVA的模式下，PA布局是任意的（物理页面要向后映射并不罕见）**。

![图片](https://mmbiz.qpic.cn/mmbiz_png/nrgibEGYIoVIfmvxSqvyLogNGTvDO8HXrYPGn4nK6QTcicYpHex2wv4ngJic9yX7m267K6PPV8MrnEd0BxbN7kdIw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

DPDK版本之间的IOVA（PA模式）布局比较

这种改变并不像看起来那样具有颠覆性，因为实际上没有多少数据结构需要IOVA连续内存。所有软件数据结构（环，内存池，哈希表等）仅需要VA连续内存，而不在意底层物理内存布局。这使得在早期DPDK版本上工作的大多数用户应用程序可以无缝过渡到新版本，而无需更改代码。

尽管如此，某些数据结构确实需要IOVA连续内存（例如，硬件队列结构），对于这些情况，引入了新的分配器标志。使用此标志，可以使memzone分配器尝试分配IOVA连续的内存。

## 12.**单个文件段**

较旧的DPDK版本在hugetlbfs文件系统中的每个大页上存储一个文件，这适用于大多数用例，但有时会出现问题，特别是，vhost-user后端的Virtio将与后端共享文件，并且有可共享文件描述符数量的硬性限制。当使用大页（例如1 GB的页面）时，它可以很好地工作，但是在页面大小较小的情况下，文件数量会很快超过文件描述符限制。

为了解决此问题，版本18.11中引入了一种新模式，即单文件段模式，该模式通过--single-file-segments EAL命令行标志启用，这使得EAL在hugetlbfs中创建的文件更少，并且使具有vhost-user后端的Virtio甚至可以在最小页面大小下工作。

原文链接：http://blog.chinaunix.net/uid-28541347-id-5834805.html

原文作者：ieee