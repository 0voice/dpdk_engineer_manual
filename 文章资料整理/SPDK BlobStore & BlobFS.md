# SPDK BlobStore & BlobFS

blobstore可看作是SPDK对外提供的一个对象存储系统，存储引擎构建在SPDK所管理的块设备之上，所采用的磁盘布局结构如图所示。

![img](https://pic3.zhimg.com/80/v2-97dc55205bce919ea5a3d418b4c28a4e_720w.webp)

superblock主要记录了如下属性信息
(1) base info
信息包括：cluster_size(blob的块存储单元大小，默认1M)，size(blobstore的容量)，io_unit_size(硬盘支持的原子in-place-update大小，md_page大小采用该阈值)
(2) 各种bitmap信息
其中mask_start记录了bitmap数据的起始lba，mask_len为lba个数(参考bs_load_read_used_pages)

与Ext4的磁盘布局结构相比较，used_blobids相当于定义了inode槽位，而used_clusters则相当于定义了block槽位。所不同的是Ext4的inode长度是固定不变的，扩展属性还有文件块信息都是通过额外的block进行存储的，inode只是记录对应块的存储位置。而blobstore则将所有的元数据信息全部保存到了inode区域(blob内部)，因此其空间长度是不确定的，需要多个md_page进行保存，对应的数据组织结构如图所示。

![img](https://pic3.zhimg.com/80/v2-2a953a8c3873e3ddc75e360f869c2c92_720w.webp)

每个blob的元数据由多个md_page组成，部分md_page以链表的方式进行关联(比如图片中通过md_page_0#next可定位到md_page_2)，头部md_page的物理位置可通过blobid进行定位(类似XFS可通过inodeId来确定inode的存储位置)，blobid的命名规则与used_blobids的槽位信息相关，可通过扫描used_blobids区域来确定当前blobstore都部署了哪些blob，以及对应的blobid是什么。

当现有md_page的存储容量不够用的时候(比如blob的扩展属性比较多，或者文件内容比较大)，可通过申请额外的md_page来做空间扩容(将其加入链表)，每一个md_page的使用情况是通过used_md_pages进行标识的，对应的bit位代表metaRegion中的一个槽位。

md_page内部主要记录了blob的各种描述符信息，包括FLAGS、XATTR(保存blob扩展属性)、EXTENT_TABLE(记录每个EXTENT_PAGE的存储位置)和EXTENT_PAGE，其中EXTENT_PAGE比较特殊，需要采用单一的md_page进行保存，不能和其他描述符进行混部，其内部主要记录了每个cluster的物理位置。

cluster相当于是每个blob的块存储单元，默认占用1MB的连续空间，以此来规避磁盘空间的外部碎片问题。不过针对小文件场景，该阈值的设置会带来比较严重的空间浪费，因为每个文件至少要占据1M的硬盘空间。

与Ext4相比，blobstore的磁盘布局并没有采用划分group的方式进行管理，而是将所有元数据统一集中到了metaRegion区域，这与blobstore的线程工作模型有关，其对硬盘空间的访问采用的是单线程RTC方式进行的，基于无锁编程模型来对IO进行异步处理，因此不需要通过拆分group来降低元数据的锁操作粒度。

另外，blobstore在执行块分配的时候，相关bit位信息只需要在内存中进行维护，而无需实时的持久化到硬盘，由于blobstore通过用户态进程来托管，我们可以将状态保存操作放到signalHandler中去做实现，在进程退出的时候，对superblock所记录的bitmap区域进行统一的sync处理，以此来降低有关操作的磁盘IO次数。而如果机器发生了断电重启，或者进程采用了kill -9强制退出(此时superblock的clean状态为false)，则可通过扫描metaRegion来将这部分元数据做重新恢复。

blobstore只提供了对象存储功能，SPDK在此基础之上又新增了blobfs来做文件存储，每个文件对应一个blob，并将文件名保存到blob的扩展属性中。不过并没有提供目录树管理功能，因此在blobfs启动的时候需要将所有的blob全部加载到内存，以便在内存中维护filename与blob的映射关系，从而便于文件的lookup检索。

作为文件系统来讲，blobfs所能提供的能力支持还是比较有限的，除了没有目录树结构以及不支持posix语义之外，还存在以下问题短板。
(1) 文件只能做追加写而不能随机写
(2) rename操作保证不了原子性
(3) 一致性语义比较弱，需要在上层自己引入journal来保证一致性
(4) 元数据采用in-place-update方式更新，难以适配ZNS
(5) cache机制较弱，数据驱逐没有采用LRU

综合来看，比较适用于大文件、少量元数据、没有随机写的应用场景、并能通过外部WAL来保证数据的一致性，因此作为LSM树的存储引擎还是可以满足需求的。







原文链接：https://zhuanlan.zhihu.com/p/531713792  原文作者：爱分享的码农