# dpdk静态库编译+动态库编译

## 构建环境

```text
g++ (GCC) 9.1.1 20190605 (Red Hat 9.1.1-2)
Linux localhost.localdomain 5.7.10-1.el7.elrepo.x86_64 #1 SMP Wed Jul 22 08:50:52 EDT 2020 x86_64 x86_64 x86_64 GNU/Linux
```

g++/gcc 9.x 版本对avx512有支持，

g++/gcc 8.x 对avx512不支持，编译时会报告警，是g++的bug

不建议编译为动态库使用，会出现线程变量访问cpu占比高的情况

以下编译 均指定 T=x86_64-native-linuxapp-gcc

## 静态库编译

dpdk源码包中，默认编译结果就是静态库：

```text
make install T=x86_64-native-linuxapp-gcc DESTDIR=build -j
```

## 动态库编译

修改相关mk

```text
	echo 'CONFIG_RTE_LINK_SHARED_LIB=y' >> ./config/defconfig_x86_64-native-linuxapp-gcc
	sed -i '/# Copyright(c) /a\ifeq ($(CONFIG_RTE_LINK_SHARED_LIB),y)\nCFLAGS += -fPIC\nendif'  ./mk/rte.lib.mk
	sed -i '/-include /a\ifeq ($(CONFIG_RTE_LINK_SHARED_LIB),y)\ncombined_mk=rte.combinedlib-shared.mk\nelse\n combined_mk=rte.combinedlib.mk\nendif' ./mk/rte.sdkbuild.mk 
	sed -i 's/mk\/rte.combinedlib.mk/mk\/$\(combined_mk\)/' ./mk/rte.sdkbuild.mk 
```

生成 [rte.combinedlib-shared.mk](https://link.zhihu.com/?target=http%3A//rte.combinedlib-shared.mk) ，下面命令最好贴在脚本中执行，在all:FORCE、clean:下一行注意有一个\t符号，为makefile规则，否则编译时会报错mk/rte.combinedlib-shared.mk:20: *** missing separator. Stop.

```text
genCombinedMk()
{
shellContent='
# SPDX-License-Identifier: BSD-3-Clause
# Copyright(c) 2010-2015 Intel Corporation

include $(RTE_SDK)/mk/rte.vars.mk

default: all

ifeq ($(CONFIG_RTE_BUILD_SHARED_LIB),y)
EXT:=.so
else
EXT:=.a
endif

COMBINEDLIB := libdpdk.so

LIBS := $(filter-out $(COMBINEDLIB), $(sort $(wildcard $(RTE_OUTPUT)/lib/*$(EXT))))

all: FORCE
	$(CC) -fPIC -shared -Wl,--whole-archive $(LIBS) -o $(RTE_OUTPUT)/lib/$(COMBINEDLIB) -Wl,--no-whole-archive
    
#
# Clean all generated files
#
.PHONY: clean
clean:
	$(Q)rm -f $(RTE_OUTPUT)/lib/$(COMBINEDLIB)

.PHONY: FORCE
FORCE:
'

echo  "$shellContent" > $1
}
	genCombinedMk ./mk/rte.combinedlib-shared.mk 
```

### 编译

```text
make install T=x86_64-native-linuxapp-gcc DESTDIR=build -j
```

得到结果在 build目录中

## 交叉编译

自定义makefile，传入指定目标服务器cpu架构及kernel版本进行编译，参考：intel，交叉编译

gcc -c -Q -march=native --help=target|grep "^\s*-march"得到目标服务器的cpu架构，不同gcc执行结果不同

以下整理出最后的makefile脚本，其内容包括：

> 解压源码包
> 校验得到指定的MACHINE及KERNEL
> 修改mk
> 编译，并安装到指定目录中

目录结构如下：

```text
.
├── 3rd
│   ├── dpdk
│   │   └── x86_64-cascadelake-linuxapp-gcc
├── 3rdsrc
│   └── dpdk
│       ├── dpdk-19.11.3.tar.gz
│       └── makefile
├── makefile
├── makefile.3rd
└── makefile.common
```

顶层makefile内容：

makefile：顶层makefile，编译时，先调用check，再确定是否调用install，防止重复编译。

```text
MAKE:=make

SUBDIRS := 3rdsrc/dpdk 

# all 3rd install dir
export DESTDIR

all: subdirs

subdirs:
	for n in $(SUBDIRS); do [ ! -e $$n ] && echo "$$n not exist, ignore" && continue; echo $$n; cd $$n; $(MAKE) check || $(MAKE)||exit 1; cd -; done

clean:
	for n in $(SUBDIRS); do [ ! -e $$n ] && continue; cd $$n; $(MAKE) clean ; cd -; done

clear:
	for n in $(SUBDIRS); do [ ! -e $$n ] && continue; cd $$n; $(MAKE) clear ; cd -; done
```

makefile.comm：该makefile中包含一些公共参数，在三方编译时include使用，在/3rdsrc/dpdk/makefile中被引用

```cpp
MKFILE_ABSPATH := $(abspath $(lastword $(MAKEFILE_LIST)))
SRC_DIR ?= $(shell dirname $(MKFILE_ABSPATH))/../
DESTDIR := $(SRC_DIR)/build/3rd
CFLAGS:=
CPPFLAGS:=
```

makefile.3rd：该makefile主要提供给其他编译模块使用，在编译dpdk时 /3rdsrc/dpdk/makefile，会将编译后的目标目录写入到此文件中

```text
# the dpdk include and lib dir
INC_DIRS+=-I$(SRC_DIR)/build/3rd/dpdk/x86_64-cascadelake-linuxapp-gcc/include/dpdk
LIB_DIRS+=-L$(SRC_DIR)/build/3rd/dpdk/x86_64-cascadelake-linuxapp-gcc/lib
DPDK_MODULES=$(SRC_DIR)/build/3rd/dpdk/x86_64-cascadelake-linuxapp-gcc/lib/modules/3.10.0-1062.18.1.el7.x86_64
# maybe copy
export DPDK_MODULES
```

3rdsrc/dpdk/makefile：此为重新编写的makefile，编译为静态库文件。

```text
include ../../makefile.common
# setup vars
RET=$(shell lscpu|grep "^CPU(s):")
CPUS=$(word 2,$(RET))
DIR_SRC=dpdk-src
TMP=$(shell $(CC) -c -Q -march=native --help=target|grep "^\s*-march")
MACHINE?=$(word 2,$(TMP))
# $(shell uname -r)
KERNEL?=
BUILD_TARGET=x86_64-$(MACHINE)-linuxapp-gcc
MACHINE_DIR=$(DIR_SRC)/mk/machine/$(MACHINE)
TCONFIG=$(DIR_SRC)/config/defconfig_$(BUILD_TARGET)

ifeq ($(KERNEL), )
    MAKE=make -j $(CPUS)
    KVER?=$(shell uname -r)
else
    exist=$(shell if [ -d $(KERNEL) ]; then echo "exist"; else echo "notexist"; fi )
    ifeq ($(exist), exist)
        # the kernel is dir
        WORDS=$(subst /, ,$(KERNEL))
        num:=$(words $(WORDS))
        LASTWORD=$(word $(num),$(WORDS))
        KPATH=$(KERNEL)
        ifeq ($(LASTWORD), build)
            num:=$(shell echo $$(($(num)-1)))
            KVER=$(word $(num),$(WORDS))
        else
            KVER=$(LASTWORD)
            KPATH+=/build
            KPATH=$(strip $(KPATH)) 
        endif
    else
        # try to kernel from /lib/modules/$(KERNEL)
        KPATH=/lib/modules/$(KERNEL)/build
        KVER=$(KERNEL)
        #$(error not found kernel KERNEL=$(KERNEL), please check)
    endif
    MAKE=make -j $(CPUS) RTE_KERNELDIR=$(KPATH)
endif

ifeq ($(DESTDIR), )
    INSTALLDIR=$(BUILD_TARGET)
else 
    INSTALLDIR=$(DESTDIR)/dpdk/$(BUILD_TARGET)
endif

MK3RD=../../makefile.3rd

# template dir and files
TEMPLATE_MACHINE=$(DIR_SRC)/mk/machine/hsw
TEMPLATE_CONFIG=$(DIR_SRC)/config/defconfig_x86_64-native-linuxapp-gcc

TAR_FILE?=$(shell ls *.tar.gz|head -n 1)

.PHONY: install objs clean clear show help

install: modify
	@echo "complie dpdk source do..."
	@cd $(DIR_SRC) && $(MAKE) install T=$(BUILD_TARGET) O=build DESTDIR=$(INSTALLDIR) > /dev/null
	@echo "complie dpdk source done"
	$(shell [ -f $(MK3RD) ] && sed -i 's#\(INC_DIRS+=.*/3rd/dpdk\)/.*include/dpdk#\1/$(BUILD_TARGET)/include/dpdk#g' $(MK3RD) )
	$(shell [ -f $(MK3RD) ] && sed -i 's#\(LIB_DIRS+=.*/3rd/dpdk\)/.*lib#\1/$(BUILD_TARGET)/lib#g' $(MK3RD) )
	$(shell [ -f $(MK3RD) ] && sed -i 's#\(MODULES=.*/3rd/dpdk\)/.*modules.*#\1/$(BUILD_TARGET)/lib/modules/$(KVER)#g' $(MK3RD) )
	@echo "replace $(MK3RD) done"
	#$(shell [ ! -f $(MK3RD) ] && echo -e "# dpdk includes $(BUILD_TARGET)\nINC_DIRS+=$$ \b(DEPLOY_DIR)/3rd/dpdk/$(BUILD_TARGET)/include\nLIB_DIRS+=$$ \b(DEPLOY_DIR)/3rd/dpdk/$(BUILD_TARGET)/lib" > $(MK3RD) )

# step 1: modify mk/machine/xxx
# step 2: modify config/defconfig_xxx
# step 3: modify makefiles
modify:
	$(shell [ ! -d $(DIR_SRC) ] && mkdir -p $(DIR_SRC) )
	$(shell tar xzf $(TAR_FILE) -C $(DIR_SRC) --strip-components 1 )
	$(shell [ ! -d $(MACHINE_DIR) ] && cp -fra $(TEMPLATE_MACHINE) $(MACHINE_DIR) && sed -i 's/\(^\s*MACHINE_CFLAGS\s*\)=.*/\1= -march=$(MACHINE)/g' $(MACHINE_DIR)/rte.vars.mk)
	$(shell [ ! -f $(TCONFIG) ] && cp -fa $(TEMPLATE_CONFIG) $(TCONFIG) && sed -i 's/\(^\s*CONFIG_RTE_MACHINE\s*\)=.*/\1="$(MACHINE)"/g' $(TCONFIG) )
	$(shell sed -i 's/CONFIG_RTE_LIBRTE_PMD_PCAP=n/CONFIG_RTE_LIBRTE_PMD_PCAP=y/' $(DIR_SRC)/config/common_base                       )
	$(shell sed -i 's/CONFIG_RTE_PORT_PCAP=n/CONFIG_RTE_PORT_PCAP=y/' $(DIR_SRC)/config/common_base                                   )
	$(shell sed -i 's/CONFIG_RTE_ENABLE_AVX512=n/CONFIG_RTE_ENABLE_AVX512=y/' $(DIR_SRC)/config/common_base                           )
	$(shell sed -i 's/CONFIG_RTE_LIBRTE_I40E_16BYTE_RX_DESC=n/CONFIG_RTE_LIBRTE_I40E_16BYTE_RX_DESC=y/' $(DIR_SRC)/config/common_base )
	$(shell sed -i 's/CONFIG_RTE_LIBRTE_PMD_AF_PACKET=n/CONFIG_RTE_LIBRTE_PMD_AF_PACKET=y/' $(DIR_SRC)/config/common_linux            )
	$(shell sed -i 's/-Werror/-Werror -Wno-error=missing-attributes/' $(DIR_SRC)/kernel/linux/igb_uio/Makefile                        )
	$(shell sed -i 's/-Werror/-Werror -Wno-error=missing-attributes/' $(DIR_SRC)/kernel/linux/kni/Makefile                            )

clean:
	@rm -rf $(DIR_SRC)
	@rm -rf $(INSTALLDIR)
    
show:
	@echo "MAKE         : $(MAKE)"
	@echo "CPUS         : $(CPUS)"
	@echo "TAR_FILE     : $(TAR_FILE)"
	@echo "DIR_SRC      : $(DIR_SRC)"
	@echo "MACHINE      : $(MACHINE)"
	@echo "KERNEL       : $(KERNEL)"
	@echo "KVER         : $(KVER)"
	@echo "KPATH        : $(KPATH)"
	@echo "BUILD_TARGET : $(BUILD_TARGET)"
	@echo "MACHINE_DIR  : $(MACHINE_DIR)"
	@echo "TCONFIG      : $(TCONFIG)"
	@echo "DESTDIR      : $(DESTDIR)"
	@echo "TEMPLATE_MACHINE : $(TEMPLATE_MACHINE)"
	@echo "TEMPLATE_CONFIG  : $(TEMPLATE_CONFIG)"

# check
exist=$(shell if [ -d $(INSTALLDIR) ] && [ -d $(INSTALLDIR)/lib/modules/$(KVER) ]; then echo "exist"; else echo "notexist"; fi )
check:
ifeq ($(exist), notexist)
	$(error "it ($(INSTALLDIR)) or ($(INSTALLDIR)/lib/modules/$(KVER)) doesn't exist and needs to be rebuilt")
else
	@echo "already exists: $(INSTALLDIR) && ($(INSTALLDIR)/lib/modules/$(KVER))"
	$(shell [ -f $(MK3RD) ] && sed -i 's#\(INC_DIRS+=.*/3rd/dpdk\)/.*include/dpdk#\1/$(BUILD_TARGET)/include/dpdk#g' $(MK3RD) )
	$(shell [ -f $(MK3RD) ] && sed -i 's#\(LIB_DIRS+=.*/3rd/dpdk\)/.*lib#\1/$(BUILD_TARGET)/lib#g' $(MK3RD) )
	$(shell [ -f $(MK3RD) ] && sed -i 's#\(MODULES=.*/3rd/dpdk\)/.*modules.*#\1/$(BUILD_TARGET)/lib/modules/$(KVER)#g' $(MK3RD) )
endif
```

原文链接：[https://blog.csdn.net/u01245383](