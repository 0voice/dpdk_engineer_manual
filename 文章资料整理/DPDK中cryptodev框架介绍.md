# DPDK中cryptodev框架介绍

## 1.前言

数据平面开发套件(DPDK, Data Plane Development Kit)是由6WIND,Intel等多家公司开发，主要基于Linux系统运行，用于快速数据包处理的函数库与驱动集合，可以极大提高数据处理性能和吞吐量，提高数据平面应用程序的工作效率。
CryptoDev库是DPDK中的一个软件库，提供的管理和配置Crypto poll mode drivers的软件，定义了统一的操作接口。支持加密、认证、链式加密认证、AEAD等对称类算法操作和非对称类算法操作。

![图片](https://mmbiz.qpic.cn/mmbiz_png/bgTy2qKBXROwPSOr5C32FlPcMHXHicjk2ObbQKSNPhrC8OhP6SBrRJnVk3ff9XFppu4JlHuOnzZ0x9Ocf0pSicHQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

cryptodev库的文件大概十几个，代码仅3千行：

```
root@keep-VirtualBox:/media/sf_VM_SHARE/code/dpdk/lib# tree cryptodev/
cryptodev/
├── cryptodev_pmd.c
├── cryptodev_pmd.h
├── cryptodev_trace_points.c
├── meson.build
├── rte_crypto_asym.h
├── rte_cryptodev.c
├── rte_cryptodev_core.h
├── rte_cryptodev.h
├── rte_cryptodev_trace_fp.h
├── rte_cryptodev_trace.h
├── rte_crypto.h
├── rte_crypto_sym.h
└── version.map

0 directories, 13 files
```

### 1.1功能介绍

CryptoDev库管理不同的crypto poll mode drivers， 对上提供统一的接口，把pmd抽象成不同的设备。主要功能模块如下图：

![图片](https://mmbiz.qpic.cn/mmbiz_png/bgTy2qKBXROwPSOr5C32FlPcMHXHicjk2PjibdfiacQO1fxYGsHp2aRmD04DeEAFc156M3ibrVxAU8BlWrmCibGxKdg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)



#### 1.2设备管理

类似linux的驱动管理，CryptoDev把每一种实现算法的硬件或者软件库当做一个crypto设备，定义了crypto设备对应的结构体rte_cryptodev。目前最大支持64个crypto设备，这些信息都会存在一个全局变量cryptodev_globals中。

```
static struct rte_cryptodev rte_crypto_devices[RTE_CRYPTO_MAX_DEVS];

struct rte_cryptodev *rte_cryptodevs = rte_crypto_devices;
static struct rte_cryptodev_global cryptodev_globals = {
        .devs           = rte_crypto_devices,
        .data           = { NULL },
        .nb_devs        = 0
};
```

在创建完设备之后，使用设备之前需要调用相关的接口对设备相关的资源进行配置和初始化，后面才开始正常处理。

#### 1.3设备创建

物理加密设备在DPDK初始化阶段的PCI探测到。虚拟设备可以在程序执行时指定vdev或者在代码中硬编码创建。

```
--vdev  'crypto_aesni_mb0,max_nb_queue_pairs=2,socket_id=0'
rte_vdev_init("crypto_aesni_mb",
                  "max_nb_queue_pairs=2,socket_id=0")
```

在PMD中调用RTE_PMD_REGISTER_VDEV注册为给定vdev的驱动。

#### 1.4设备描述标识

描述一个设备有两种标识。一个是设备的索引，另外一个是设备的名称。可以类比接口名和接口索引。
设备索引主要在于设备创建的顺序，名称是PMD注册时指定的。

#### 1.5设备配置

配置主要就是资源的配置(如：队列数目)，默认状态，计数器初始化。

#### 1.6队列对配置

pmd最主要的模式就是针对一个队列对入队出队，通过rte_cryptodev_queue_pair_setup接口分别设置crypto设备的队列对，从指定的socket申请资源。

## 2.设备特性及能力

Crypto设备定义全局的feature和算法能力，来描述crypto设备支持的功能。

在设备创建时，定义设备支持的特性。

```
// openssl pmd 创建时设置 feature

dev->feature_flags = RTE_CRYPTODEV_FF_SYMMETRIC_CRYPTO |
            RTE_CRYPTODEV_FF_SYM_OPERATION_CHAINING |
            RTE_CRYPTODEV_FF_CPU_AESNI |
            RTE_CRYPTODEV_FF_IN_PLACE_SGL |
            RTE_CRYPTODEV_FF_OOP_SGL_IN_LB_OUT |
            RTE_CRYPTODEV_FF_OOP_LB_IN_LB_OUT |
            RTE_CRYPTODEV_FF_ASYMMETRIC_CRYPTO |
            RTE_CRYPTODEV_FF_RSA_PRIV_OP_KEY_EXP |
            RTE_CRYPTODEV_FF_RSA_PRIV_OP_KEY_QT |
            RTE_CRYPTODEV_FF_SYM_SESSIONLESS;
```

每个PMD会提供各自的算法能力数组，类型为`struct rte_cryptodev_capabilities`。一般在算法执行前，使用者通过rte_cryptodev_info_get接口调用到具体PMD提供的接口，获取到PMD的能力，再决定能否继续操作。

```
       rte_cryptodev_info_get(dev_id, &dev_info);

    if (!(dev_info.feature_flags & RTE_CRYPTODEV_FF_SYMMETRIC_CRYPTO)) {
        RTE_LOG(INFO, USER1, "Feature flag requirements for Snow3G "
                "testsuite not met\n");
        return TEST_SKIPPED;
    }


    ... 

    static const struct rte_cryptodev_capabilities openssl_pmd_capabilities[] = {
    {   /* MD5 HMAC */
        .op = RTE_CRYPTO_OP_TYPE_SYMMETRIC,
        {.sym = {
            .xform_type = RTE_CRYPTO_SYM_XFORM_AUTH,
            {.auth = {
                .algo = RTE_CRYPTO_AUTH_MD5_HMAC,
                .block_size = 64,
                .key_size = {
                    .min = 1,
                    .max = 64,
                    .increment = 1
                },
                .digest_size = {
                    .min = 1,
                    .max = 16,
                    .increment = 1
                },
                .iv_size = { 0 }
            }, }
        }, }
    },
    {   /* MD5 */
        .op = RTE_CRYPTO_OP_TYPE_SYMMETRIC,
        {.sym = {
            .xform_type = RTE_CRYPTO_SYM_XFORM_AUTH,
            {.auth = {
                .algo = RTE_CRYPTO_AUTH_MD5,
                .block_size = 64,
                .key_size = {
                    .min = 0,
                    .max = 0,
                    .increment = 0
                },
                .digest_size = {
                    .min = 16,
                    .max = 16,
                    .increment = 0
                },
                .iv_size = { 0 }
            }, }
        }, }
    },
    ... // 省略
```

### 2.1操作处理

Crypto Operation使用入队出队的接口集实现操作处理。使用enqueue_burst接口接收待处理的operations，软件pmd一般会直接进行算法执行，硬件pmd会把数据放到硬件的input queue。使用dequeue_burst接口获取已经处理完成的operations，软件pmd一般从软件的已完成的队列中获取完成的operations，硬件pmd从硬件的output/processed queue获取。

#### 2.1.1私有数据

提供set和get接口API设置算法计算时需要的用户数据到crypto session数据结构中。对于session-less的操作，私有数据可以存放到operation内存尾部。所谓的私有数据一般是各个pmd内部识别算法使用的数据结构。

#### 2.1.2回调API

支持注册回调函数，在关联的队列对调用enqueue_burst之后，或者调用dequeue_burst之前，执行额外的操作。
这些回调函数，需要用户自己控制注销，以免造成内存泄漏。

感觉这个没有多大用处。

#### 2.1.3Enqueue/Dequeue Burst 接口

提供接口往指定设备的指定队列队塞或者拉取operations。

```
uint16_t rte_cryptodev_enqueue_burst(uint8_t dev_id, uint16_t qp_id,
                                     struct rte_crypto_op **ops, uint16_t nb_ops)
uint16_t rte_cryptodev_dequeue_burst(uint8_t dev_id, uint16_t qp_id,
                                     struct rte_crypto_op **ops, uint16_t nb_ops)
```

#### 2.1.4Operation结构定义

定义了operation的结构。一部分通用的区域，然后再分别跟着sym或者asym的operation，最后是私有的数据。开发者需要填充好各个字段，后续PMD才能正常执行。不同的算法都有一个约定好的数据格式，各个pmd按照约定去解析数据。

#### 2.1.5Operation分配管理

对Operation空间的分配管理提供了一套API接口。
定义一个mempool来分配operation空间， 以及如何从pool申请/释放空间：

```
extern struct rte_mempool *
rte_crypto_op_pool_create(const char *name, enum rte_crypto_op_type type,
                          unsigned nb_elts, unsigned cache_size, uint16_t priv_size,
                          int socket_id);
struct rte_crypto_op *rte_crypto_op_alloc(struct rte_mempool *mempool,
                                          enum rte_crypto_op_type type)

unsigned rte_crypto_op_bulk_alloc(struct rte_mempool *mempool,
                                  enum rte_crypto_op_type type,
                                  struct rte_crypto_op **ops, uint16_t nb_ops)

void rte_crypto_op_free(struct rte_crypto_op *op)
```

#### 2.1.6对称算法支持

主要用图描述一下对称算法的使用。

![图片](https://mmbiz.qpic.cn/mmbiz_png/bgTy2qKBXROwPSOr5C32FlPcMHXHicjk2ic7T376VLfBibA0phHhUOM3Pk0Ud9TLToG4LnicDBvc5Tq2V8jfJTmg0g/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

#### 2.1.7非对称算法支持

主要用图描述一下非对称算法的使用。

![图片](https://mmbiz.qpic.cn/mmbiz_png/bgTy2qKBXROwPSOr5C32FlPcMHXHicjk2DK74iaCZewGSLiaLVdfTxZz41kaDlT71PXKibwxCBao2wbEW4ZibQVHkBg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

#### 2.1.8同步模式

部分pmd支持同步模式，可以不使用enqueue_burst和dequeue_burst接口，而直接使用rte_cryptodev_sym_cpu_crypto_process。此时，需要设置feature 标记RTE_CRYPTODEV_FF_SYM_CPU_CRYPTO。

#### 2.1.9Raw Data-path API

不使用DPDK提供的数据结构（rte_mbuf, rte_crypto_op等），可以使用用户定义的结构直接转换。待深入了解。

## 3.设备信息

可以使用工具dpdk-telemetry.py查看设备信息。

![图片](https://mmbiz.qpic.cn/mmbiz_png/bgTy2qKBXROwPSOr5C32FlPcMHXHicjk2fef63ByszBhSIoVZjmj3K6iblGhj1XeNsbdxcw1dlqe3QUkVDVzSEZw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

DPDK还提供了许多tools来作大页内存配置/设备绑定等。

## 4.更多

CryptoDev库的代码使用了面向对象编程的方法，抽象出许多实体，使用指针的方式关联这些实体。

随着软件行业的发展，面向对象编程使用的也更加广泛了。现在很多使用C语言编写的库都应用了面向对象编程的设计。包括Linux内核代码也一样，比如Linux Crypto框架，把算法抽象成了对象，还使用了模板，使得代码更加形象化。

这周给组内分享了一下DPDK CryptoDev框架的基础，星期一的时候花了一整天的时间全面地看了一下cryptodev的主要代码。今天小结一下。

原文链接：https://mp.weixin.qq.com/s/FsGodvdX6EEOUBShvfVdxw