## uboot 2014-07 SPL编译过程分析

> author: cedar
>
> date 2019-05-07



首先，对于uboot来说，编译过程是：

1. 编译ATF(arm-trusted-firmware)
2. 编译uboot
3. 编译spl版本

以上过程在build.sh文件build_uboot函数，如下所示

``` bash
# brandy/build.sh
build_uboot()
{
	if [ "x${PLATFORM}" = "xsun50iw1p1" ]; then

		prepare_toolchain
		#make atf
		cd arm-trusted-firmware-1.0/
		#make clean
		make PLAT=sun50iw1p1
		cd ..
        fi
	if [ "x${PLATFORM}" = "xsun50iw1p1" ] || [ "x${PLATFORM}" = "xsun8iw10p1" ]; then
                cd u-boot-2014.07/
	else
		cd u-boot-2011.09/
	fi

	#make distclean
	if [ "x$MODE" = "xota_test" ] ; then
		export "SUNXI_MODE=ota_test"
	fi
	make ${PLATFORM}_config
	#remake --trace -j16
	make -j16         #编译uboot

    if [ ${PLATFORM} = "sun50iw1p1" ] || [ ${PLATFORM} = "sun8iw6p1" ] || [ ${PLATFORM} = "sun8iw7p1" ] || [ ${PLATFORM} = "sun8iw8p1" ] || [ ${PLATFORM} = "sun9iw1p1" ] || [ ${PLATFORM} = "sun8iw5p1" ] || [ "x${PLATFORM}" = "xsun8iw10p1" ]; then
        make spl #编译spl版本
    fi

	cd - 1>/dev/null
}
```

本文重点分析spl的编译过程，所以根据line 28打开makefile文件

``` makefile
#/brandy/u-boot-2014.07/Makefile Line 862
boot0:
	make -f spl_make boot0
fes:
	make -f spl_make fes
sboot:
	make -f spl_make sboot

ifeq ($(CONFIG_SUNXI_SECURE_SYSTEM),y)
spl:    fes boot0 sboot
else
spl:    fes boot0
endif
```

可以看到，先spl依赖于`fes` `boot0` ，实际执行的命令是

``` makefile
spl_make fes
spl_make boot0
```

那我们打开spl_make文件，查看fes目标和boot0目标

``` makefile
#/brandy/u-boot-2014.07/spl_make Line 27
fes:
		$(MAKE) -C sunxi_spl/fes_init all
		@$(TOPDIR)/tools/gen_check_sum sunxi_spl/fes_init/fes1.bin fes1_$(CONFIG_TARGET_NAME).bin > /dev/null
		@cp -v fes1_$(CONFIG_TARGET_NAME).bin $(TOPDIR)/../../tools/pack/chips/$(CONFIG_TARGET_NAME)/bin/fes1_$(CONFIG_TARGET_NAME).bin
```

fes生成步骤如下

1. 根据`sunxi_spl/fes_init`目录下的makefile编译出fes1.bin文件
2. 计算bin校验和，并生成新的bin文件
3. 复制新bin到结果目录`$(TOPDIR)/../../tools/pack/chips/$(CONFIG_TARGET_NAME)/bin/`

我们稍后再深入分析实现原理，接下来，我们看boot0目标

``` makefile
#/brandy/u-boot-2014.07/spl_make Line 32

boot0:
		$(MAKE)  -C  sunxi_spl/boot0 all
ifdef CONFIG_STORAGE_MEDIA_NAND
		@git show HEAD --pretty=format:"%H" | head -n 1 > cur.log
		@[ -s cur.log ] || echo "1234567891234567891234567891234567891234" >cur.log
		@../add_hash.sh -f sunxi_spl/boot0/boot0_nand.bin -m boot0
		@$(TOPDIR)/tools/gen_check_sum sunxi_spl/boot0/boot0_nand.bin boot0_nand_$(CONFIG_TARGET_NAME).bin > /dev/null
		@cp -v boot0_nand_$(CONFIG_TARGET_NAME).bin $(TOPDIR)/../../tools/pack/chips/$(CONFIG_TARGET_NAME)/bin/boot0_nand_$(CONFIG_TARGET_NAME).bin
endif
ifdef CONFIG_STORAGE_MEDIA_MMC
		@git show HEAD --pretty=format:"%H" | head -n 1 > cur.log
		@[ -s cur.log ] || echo "1234567891234567891234567891234567891234" >cur.log
		@../add_hash.sh -f sunxi_spl/boot0/boot0_sdcard.bin -m boot0
		@$(TOPDIR)/tools/gen_check_sum sunxi_spl/boot0/boot0_sdcard.bin boot0_sdcard_$(CONFIG_TARGET_NAME).bin > /dev/null
		@cp -v boot0_sdcard_$(CONFIG_TARGET_NAME).bin $(TOPDIR)/../../tools/pack/chips/$(CONFIG_TARGET_NAME)/bin/boot0_sdcard_$(CONFIG_TARGET_NAME).bin
endif
ifdef CONFIG_STORAGE_MEDIA_SPINOR
		@git show HEAD --pretty=format:"%H" | head -n 1 > cur.log
		@[ -s cur.log ] || echo "1234567891234567891234567891234567891234" >cur.log
		@../add_hash.sh -f sunxi_spl/boot0/boot0_spinor.bin -m boot0
		@$(TOPDIR)/tools/gen_check_sum sunxi_spl/boot0/boot0_spinor.bin boot0_spinor_$(CONFIG_TARGET_NAME).bin > /dev/null
		@cp -v boot0_spinor_$(CONFIG_TARGET_NAME).bin $(TOPDIR)/../../tools/pack/chips/$(CONFIG_TARGET_NAME)/bin/boot0_spinor_$(CONFIG_TARGET_NAME).bin
endif
```

boot0的生成依赖于`CONFIG_STORAGE_MEDIA_NAND` `CONFIG_STORAGE_MEDIA_MMC` `CONFIG_STORAGE_MEDIA_SPINOR` 这三个配置宏，对于A64来说，仅仅使能了NAND和MMC宏，从编译log中也可以看到这点

``` 
[xxx] make libnand start
make -C /home/cedar/Desktop/boot_bank/brandy/u-boot-2014.07/sunxi_spl/boot0/load_nand/
 CC      load_Boot1_from_nand.c ...
[xxx] make libspinor stop
/home/cedar/Desktop/boot_bank/brandy/u-boot-2014.07/../toolchain/gcc-arm/bin/arm-linux-gnueabihf-ld /home/cedar/Desktop/boot_bank/brandy/u-boot-2014.07/arch/arm/cpu/armv7/sun50iw1p1/dram/libchipid.o /home/cedar/Desktop/boot_bank/brandy/u-boot-2014.07/arch/arm/cpu/armv7/sun50iw1p1/dram/libdram.o /home/cedar/Desktop/boot_bank/brandy/u-boot-2014.07/sunxi_spl/boot0/libs/libgeneric.o /home/cedar/Desktop/boot_bank/brandy/u-boot-2014.07/sunxi_spl/boot0/main/libmain.o /home/cedar/Desktop/boot_bank/brandy/u-boot-2014.07/sunxi_spl/boot0/spl/libsource_spl.o /home/cedar/Desktop/boot_bank/brandy/u-boot-2014.07/sunxi_spl/spl/lib/libgeneric.o /home/cedar/Desktop/boot_bank/brandy/u-boot-2014.07/arch/arm/cpu/armv7/sun50iw1p1/nand/libnand.o /home/cedar/Desktop/boot_bank/brandy/u-boot-2014.07/sunxi_spl/boot0/load_nand/libloadnand.o -L /home/cedar/Desktop/boot_bank/brandy/toolchain/gcc-arm/bin/../lib/gcc/arm-linux-gnueabihf/4.9.2 -lgcc   -T/home/cedar/Desktop/boot_bank/brandy/u-boot-2014.07/sunxi_spl/boot0/boot0.lds -o boot0_nand.axf -Map boot0_nand.map
bootaddr is (0x10000)
/home/cedar/Desktop/boot_bank/brandy/u-boot-2014.07/../toolchain/gcc-arm/bin/arm-linux-gnueabihf-objcopy  -O binary  /home/cedar/Desktop/boot_bank/brandy/u-boot-2014.07/sunxi_spl/boot0/boot0_nand.axf /home/cedar/Desktop/boot_bank/brandy/u-boot-2014.07/sunxi_spl/boot0/boot0_nand.bin
[xxx] make libmmc start
make -C /home/cedar/Desktop/boot_bank/brandy/u-boot-2014.07/arch/arm/cpu/armv7/sun50iw1p1/mmc/
 CC      mmc_bsp.c ...
 CC      mmc.c ...
[xxx] make libspinor stop
[xxx] make libmmc start
make -C /home/cedar/Desktop/boot_bank/brandy/u-boot-2014.07/sunxi_spl/boot0/load_mmc/
 CC      load_boot1_from_sdmmc.c ...
[xxx] make libspinor stop
```

所以，编译boot0的流程是

1. 执行boot0目录下Makefile，编译全部代码
2. 生成NAND bin文件
3. 生成MMC bin文件

那接下来详细分析实现细节

### 详细分析-fes目标生成

根据上面的分析fes第一步是生成bin文件，具体命令是

``` makefile
#/brandy/u-boot-2014.07/spl_make line 28
$(MAKE) -C sunxi_spl/fes_init all
```

等同于

``` makefile
cd /brandy/u-boot-2014.07/sunxi_spl/fes_init
make all
```

所以，我们看fes_init目录下makefile的内容

``` makefile
#/brandy/u-boot-2014.07/sunxi_spl/fes_init/Makefile

#
# (C) Copyright 2000-2011
# Wolfgang Denk, DENX Software Engineering, wd@denx.de.
#
# (C) Copyright 2011
# Daniel Schwierzeck, daniel.schwierzeck@googlemail.com.
#
# (C) Copyright 2011
# Texas Instruments Incorporated - http://www.ti.com/
# Aneesh V <aneesh@ti.com>
#
# This file is released under the terms of GPL v2 and any later version.
# See the file COPYING in the root directory of the source tree for details.
#
# Based on top-level Makefile.
#

include $(SPLDIR)/config.mk
include $(TOPDIR)/include/autoconf.mk
include $(TOPDIR)/include/autoconf.mk.dep

CONFIG_SPL := y
export CONFIG_SPL

FES_LDSCRIPT := $(SPLDIR)/fes_init/main/fes_init.lds


# We want the final binaries in this directory
obj := $(OBJTREE)/sunxi_spl/fes_init/


LIBS-y += sunxi_spl/fes_init/spl/libsource_spl.o
LIBS-y += sunxi_spl/fes_init/main/libmain.o
LIBS-y += sunxi_spl/spl/lib/libgeneric.o
LIBS-y += arch/$(ARCH)/cpu/$(CPU)/$(SOC)/dram/libdram.o
LIBS-$(CONFIG_SUNXI_CHIPID) += arch/$(ARCH)/cpu/$(CPU)/$(SOC)/dram/libchipid.o

LIBS := $(addprefix $(OBJTREE)/,$(sort $(LIBS-y)))


# Special flags for CPP when processing the linker script.
# Pass the version down so we can handle backwards compatibility
# on the fly.
LDPPFLAGS += \
	-include $(TOPDIR)/include/u-boot/u-boot.lds.h \
	-DFES1ADDR=$(CONFIG_FES1_RUN_ADDR)	 \
	$(shell $(LD) --version | \
	  sed -ne 's/GNU ld version \([0-9][0-9]*\)\.\([0-9][0-9]*\).*/-DLD_MAJOR=\1 -DLD_MINOR=\2/p')

ALL-y	+= $(obj)fes1.bin

all: $(ALL-y)

$(obj)fes1.bin:	$(obj)fes1.axf
	$(OBJCOPY) $(OBJCFLAGS) -O binary $< $@

$(obj)fes1.axf:  $(LIBS) $(obj)fes_init.lds
	$(LD) $(LIBS) $(PLATFORM_LIBGCC) $(LDFLAGS) -T$(obj)fes_init.lds -o fes1.axf -Map fes1.map

$(LIBS): depend
	$(MAKE) -C $(SRCTREE)$(dir $(subst $(OBJTREE),,$@))

$(obj)fes_init.lds: $(FES_LDSCRIPT)
	@$(CPP) $(ALL_CFLAGS) $(LDPPFLAGS) -ansi -D__ASSEMBLY__ -P - <$^ >$@

depend:.depend

#########################################################################

# defines $(obj).depend target
include $(SRCTREE)/rules.mk

sinclude .depend

#########################################################################

```



Makefile比较简单，首先是all依赖ALL-y，而ALL-y实际上就是fes1.bin

所以执行line 55

```makefile
$(obj)fes1.bin:	$(obj)fes1.axf
```

而fes1.axf依赖libs和fes_init.lds

``` makefile
$(obj)fes1.axf:  $(LIBS) $(obj)fes_init.lds
```

首先看LIBS目标

``` makefile
$(LIBS): depend
	$(MAKE) -C $(SRCTREE)$(dir $(subst $(OBJTREE),,$@))
```

不好分析，所以我们添加一些echo日志语句

``` makefile
$(LIBS): depend
	$(MAKE) -C $(SRCTREE)$(dir $(subst $(OBJTREE),,$@))
	@echo "[yyyy] libs   =$(LIBS))"
	@echo "[yyyy] lib_dep=$(SRCTREE)$(dir $(subst $(OBJTREE),,$@))"
```

分析build日志，发现4处有效信息

```
[yyyy] libs   =/home/cedar/Desktop/boot_bank/brandy/u-boot-2014.07/arch/arm/cpu/armv7/sun50iw1p1/dram/libchipid.o 
/home/cedar/Desktop/boot_bank/brandy/u-boot-2014.07/arch/arm/cpu/armv7/sun50iw1p1/dram/libdram.o /home/cedar/Desktop/boot_bank/brandy/u-boot-2014.07/sunxi_spl/fes_init/main/libmain.o /home/cedar/Desktop/boot_bank/brandy/u-boot-2014.07/sunxi_spl/fes_init/spl/libsource_spl.o /home/cedar/Desktop/boot_bank/brandy/u-boot-2014.07/sunxi_spl/spl/lib/libgeneric.o)

[yyyy] lib_dep=/home/cedar/Desktop/boot_bank/brandy/u-boot-2014.07/arch/arm/cpu/armv7/sun50iw1p1/dram/
[yyyy] lib_dep=/home/cedar/Desktop/boot_bank/brandy/u-boot-2014.07/arch/arm/cpu/armv7/sun50iw1p1/dram/
[yyyy] lib_dep=/home/cedar/Desktop/boot_bank/brandy/u-boot-2014.07/sunxi_spl/fes_init/main/
[yyyy] lib_dep=/home/cedar/Desktop/boot_bank/brandy/u-boot-2014.07/sunxi_spl/fes_init/spl/
[yyyy] lib_dep=/home/cedar/Desktop/boot_bank/brandy/u-boot-2014.07/sunxi_spl/spl/lib/
```

首先，libs变量源自于

``` makefile
LIBS-y += sunxi_spl/fes_init/spl/libsource_spl.o
LIBS-y += sunxi_spl/fes_init/main/libmain.o
LIBS-y += sunxi_spl/spl/lib/libgeneric.o
LIBS-y += arch/$(ARCH)/cpu/$(CPU)/$(SOC)/dram/libdram.o
LIBS-$(CONFIG_SUNXI_CHIPID) += arch/$(ARCH)/cpu/$(CPU)/$(SOC)/dram/libchipid.o

LIBS := $(addprefix $(OBJTREE)/,$(sort $(LIBS-y)))
```

line 7的含义是排序LIBS-y，然后添加前缀OBJTREE，也就是`/home/cedar/Desktop/boot_bank/brandy/u-boot-2014.07/`

所以，我们知道了，下面一句的意思是，从libs变量中取出目录部分，然后执行目录下的makefile文件

```
$(MAKE) -C $(SRCTREE)$(dir $(subst $(OBJTREE),,$@))
```

另外，libs依赖于depend，而depend又依赖于.depend

编译系统中，rules.mk用于生成.depend文件，同时`sinclude .depend`负责引入自动依赖关系

那我们继续分析libs中相关目录的makefile

```
/home/cedar/Desktop/boot_bank/brandy/u-boot-2014.07/arch/arm/cpu/armv7/sun50iw1p1/dram/
/home/cedar/Desktop/boot_bank/brandy/u-boot-2014.07/arch/arm/cpu/armv7/sun50iw1p1/dram/
/home/cedar/Desktop/boot_bank/brandy/u-boot-2014.07/sunxi_spl/fes_init/main/
/home/cedar/Desktop/boot_bank/brandy/u-boot-2014.07/sunxi_spl/fes_init/spl/
/home/cedar/Desktop/boot_bank/brandy/u-boot-2014.07/sunxi_spl/spl/lib/
```

``` makefile
#brandy/u-boot-2014.07/arch/arm/cpu/armv7/sun50iw1p1/dram/makefile 
# 主要用来拷贝lib文件，重命名为libxxx.o

include $(TOPDIR)/config.mk

all:
ifeq ($(notdir $(shell find ./ -name lib-dram)), lib-dram)
	make -C lib-dram
else
	@echo "libdram exist"
endif

ifeq ($(notdir $(shell find ./ -name lib-chipid)), lib-chipid)
	make -C lib-chipid
else
	@echo "lib-chipid exist"
endif

	cp ./libdram ./libdram.o
	cp ./libchipid ./libchipid.o

#########################################################################

# defines $(obj).depend target
include $(SRCTREE)/rules.mk

sinclude $(obj).depend

#########################################################################
```

``` makefile
#/brandy/u-boot-2014.07/sunxi_spl/fes_init/main/makefile
#主要生成main目录下的*.o文件

include $(SPLDIR)/config.mk

LIB	:= $(obj)libmain.o

HEAD    := fes_head.o

START	:= fes1_entry.o

COBJS   += fes1_main.o

SRCS	:= $(START:.o=.S) $(COBJS:.o=.c) $(HEAD:.o=.c)

OBJS =  $(COBJS)

all:	 .depend $(HEAD) $(START) $(LIB)


$(LIB):	$(OBJS)
	$(call cmd_link_o_target, $(OBJS))

#########################################################################

# defines $(obj).depend target
include $(SRCTREE)/rules.mk

sinclude $(obj).depend

#########################################################################
```



``` makefile
#/brandy/u-boot-2014.07/sunxi_spl/fes_init/spl/makefile
#主要生成spl下的libxxx.o文件

include $(SPLDIR)/config.mk

LIB	=  $(obj)libsource_spl.o

COBJS-y	+= $(TOPDIR)/arch/$(ARCH)/cpu/$(CPU)/$(SOC)/spl/timer_spl.o
COBJS-y	+= $(TOPDIR)/arch/$(ARCH)/cpu/$(CPU)/$(SOC)/spl/gpio_spl.o
COBJS-y	+= $(TOPDIR)/arch/$(ARCH)/cpu/$(CPU)/$(SOC)/spl/serial_spl.o
COBJS-y += $(TOPDIR)/arch/$(ARCH)/cpu/$(CPU)/$(SOC)/spl/clock_spl.o
COBJS-$(CONFIG_SUNXI_RSB) += $(TOPDIR)/arch/$(ARCH)/cpu/$(CPU)/$(SOC)/spl/rsb_spl.o
COBJS-$(CONFIG_SUNXI_I2C) += $(TOPDIR)/arch/$(ARCH)/cpu/$(CPU)/$(SOC)/spl/sunxi_i2c_spl.o

#COBJS-$(CONFIG_XXXX)	+= xxxx.o
COBJS 	:= $(COBJS-y)

SRCS	:= $(COBJS:.o=.c)
OBJS	:= $(addprefix $(obj),$(COBJS))

all:	$(obj).depend $(LIB)

$(LIB):	$(OBJS)
	$(call cmd_link_o_target, $(OBJS))

#########################################################################

# defines $(obj).depend target
include $(SRCTREE)/rules.mk

sinclude $(obj).depend

#########################################################################
```



``` makefile
#/brandy/u-boot-2014.07/sunxi_spl/spl/lib/makefile
#主要生成lib下的libxxx.o文件

include $(SPLDIR)/config.mk
LIB	=  $(obj)libgeneric.o

COBJS	+= console.o
COBJS	+= string.o
COBJS	+= jmp.o

#COBJS-$(CONFIG_XXXX)	+= xxxx.o

SRCS	:= $(COBJS:.o=.c)
OBJS	:= $(addprefix $(obj),$(COBJS))

all:	 $(LIB)

$(LIB):	$(OBJS)
	$(call cmd_link_o_target, $(OBJS))

#########################################################################

# defines $(obj).depend target
include $(SRCTREE)/rules.mk

sinclude $(obj).depend

#########################################################################
```

那生成完4个目录下的所有*.o文件，我们回到fes_init/makefile，看到line 57

``` makefile
#/brandy/u-boot-2014.07/sunxi_spl/fes_init/Makefile line 57

$(obj)fes1.axf:  $(LIBS) $(obj)fes_init.lds
	$(LD) $(LIBS) $(PLATFORM_LIBGCC) $(LDFLAGS) -T$(obj)fes_init.lds -o fes1.axf -Map fes1.map
```

用ld工具链接所有的*.o文件，生成fes1.axf和map文件

在这过程中，makefile需要更新链接脚本fes_init.lds，相关命令是

``` makefile
#/brandy/u-boot-2014.07/sunxi_spl/fes_init/Makefile line 63

$(obj)fes_init.lds: $(FES_LDSCRIPT)
	@$(CPP) $(ALL_CFLAGS) $(LDPPFLAGS) -ansi -D__ASSEMBLY__ -P - <$^ >$@
```



最后，对于rules.mk和.depend的分析，我们放在文章最后来进行

让我们回到boot0目标

### 详细分析-boot0目标生成

根据以上分析，boot0首先通过下句编译boot0

``` makefile
#/brandy/u-boot-2014.07/spl_make Line 33

$(MAKE)  -C  sunxi_spl/boot0 all
```

让我们来看下boot0目录下的makefile all目标

``` makefile
#/brandy/u-boot-2014.07/sunxi_spl/boot0/makefile Line 77

all:	 $(ALL-y)  #这里，ALL-y 为$(obj)boot0_nand.bin，$(obj)boot0_sdcard.bin
```

可以知道，all的终极目标是生成两个bin文件，而bin依赖于axf，axf的依赖于libs, libnand/libmmc，*.lds

所以我们从libs开始分析

``` makefile
#/brandy/u-boot-2014.07/sunxi_spl/boot0/makefile Line 99

$(LIBS): depend
	@echo "[xxx] make libs start"
	$(MAKE) -C $(SRCTREE)$(dir $(subst $(OBJTREE),,$@))
	@echo "[xxx] make libs stop"
```

根据以上fes的分析，libs的生成也是依次进入不同目录执行makefile

``` makefile
#/brandy/u-boot-2014.07/sunxi_spl/boot0/makefile Line 35

LIBS-y += sunxi_spl/boot0/spl/libsource_spl.o
LIBS-y += sunxi_spl/boot0/main/libmain.o
LIBS-y += sunxi_spl/boot0/libs/libgeneric.o
LIBS-y += sunxi_spl/spl/lib/libgeneric.o
LIBS-y += arch/$(ARCH)/cpu/$(CPU)/$(SOC)/dram/libdram.o
LIBS-$(CONFIG_SUNXI_CHIPID) += arch/$(ARCH)/cpu/$(CPU)/$(SOC)/dram/libchipid.o
LIBS := $(addprefix $(OBJTREE)/,$(sort $(LIBS-y)))
```

因为在fes过程中，已经生成libdram.o，libchipid.o，libgeneric.o，所以这里主要就是依次进入boot0/spl，boot0/main，boot0/libs目录执行makefile

这里的过程不做过多分析

跟fes一样，boot0 libs依然依赖.depend，到最后再解释这一点

对于boot0_sdcard.axf来说，还要依赖LIBMMC，命令如下

``` makefile
#/brandy/u-boot-2014.07/sunxi_spl/boot0/makefile Line 109

$(LIBMMC): depend
	@echo "[xxx] make libmmc start"
	$(MAKE) -C $(SRCTREE)$(dir $(subst $(OBJTREE),,$@))
	@echo "[xxx] make libspinor stop"
```

依次进入LIBMMC目录执行makefile

``` makefile
#/brandy/u-boot-2014.07/sunxi_spl/boot0/makefile Line 48

LIBMMC-$(CONFIG_STORAGE_MEDIA_MMC) += sunxi_spl/boot0/load_mmc/libloadmmc.o
LIBMMC-$(CONFIG_STORAGE_MEDIA_MMC) += arch/$(ARCH)/cpu/$(CPU)/$(SOC)/mmc/libmmc.o

LIBMMC := $(addprefix $(OBJTREE)/,$(sort $(LIBMMC-y)))
```

最后，同fes一样，boot0依然依赖于lds链接文件，不做过多解释

当所有的*.o准备完成之后，axf文件的生成命令如下

``` makefile
#/brandy/u-boot-2014.07/sunxi_spl/boot0/makefile Line 91
$(LD) $(LIBS) $(LIBMMC)  $(PLATFORM_LIBGCC) $(LDFLAGS) -T$(obj)boot0.lds -o boot0_sdcard.axf -Map boot0_sdcard.map

```



### fes和boot0编译流程总结

整体流程基本一致

1. 编译对应目录下的*.c *.s
2. 根据目录下makefile链接成一个*.o文件
3. 配置lds链接脚本
4. 通过ld工具链接所有*.o lib，生成axf文件
5. 通过objcopy，从axf文件中生成bin

那最后还有几个遗留问题

1. 每个编译目录下makefile都会调用`cmd_link_o_target`，它怎么工作的？
2. .depend文件是怎样生成的？有什么作用？

### `cmd_link_o_target`深入分析

我们进入sunxi_spl/boot0/libs/makefile中，发现第一行引入了config.mk文件

那看一下config.mk的内容

``` makefile
#/brandy/u-boot-2014.07/sunxi_spl/config.mk

# Load generated board configuration
sinclude $(OBJTREE)/include/autoconf.mk   #包含CONFIG_XXX配置宏

CROSS_COMPILE := $(TOPDIR)/../toolchain/gcc-arm/bin/arm-linux-gnueabihf-
#CROSS_COMPILE := $(TOPDIR)/../gcc-linaro/bin/arm-linux-gnueabi-

AS		= $(CROSS_COMPILE)as
LD		= $(CROSS_COMPILE)ld
CC		= $(CROSS_COMPILE)gcc
CPP		= $(CC) -E
AR		= $(CROSS_COMPILE)ar
NM		= $(CROSS_COMPILE)nm
LDR		= $(CROSS_COMPILE)ldr
STRIP		= $(CROSS_COMPILE)strip
OBJCOPY		= $(CROSS_COMPILE)objcopy
OBJDUMP		= $(CROSS_COMPILE)objdump

##########################################################
COMPILEINC :=  -isystem $(shell dirname `$(CC)  -print-libgcc-file-name`)/include
SPLINCLUDE    := \
		-I$(SRCTREE)/include \
		-I$(SRCTREE)/arch/arm/include \
		-I$(SPLDIR)/include           \
		-I$(SRCTREE)/include/openssl

PLATFORM_RELFLAGS +=  -march=armv7-a

# 通用编译器和链接器命令
COMM_FLAGS := -nostdinc  $(COMPILEINC) \
	-g  -Os   -fno-common -msoft-float -mfpu=neon  \
	-fno-builtin -ffreestanding \
	-D__KERNEL__  \
	-DCONFIG_ARM -D__ARM__ \
	-D__NEON_SIMD__  \
	-mabi=aapcs-linux \
	-mthumb-interwork \
	-fno-stack-protector \
	-Wall \
	-Wstrict-prototypes \
	-Wno-format-security \
	-Wno-format-nonliteral \
	-pipe




C_FLAGS += $(SPLINCLUDE)   $(COMM_FLAGS)
S_FLAGS += $(SPLINCLUDE)   -D__ASSEMBLY__  $(COMM_FLAGS)
#LDFLAGS += --gap-fill=0xff
###########################################################

###########################################################
PLATFORM_LIBGCC = -L $(shell dirname `$(CC) $(CFLAGS) -print-libgcc-file-name`) -lgcc
export PLATFORM_LIBGCC
###########################################################

# Allow boards to use custom optimize flags on a per dir/file basis
ALL_AFLAGS = $(AFLAGS)  $(PLATFORM_RELFLAGS) $(S_FLAGS)
ALL_CFLAGS = $(CFLAGS)  $(PLATFORM_RELFLAGS) $(C_FLAGS)
export ALL_CFLAGS ALL_AFLAGS

# *.o *.c编译规则
$(obj)%.o:	%.S
	@$(CC)  $(ALL_AFLAGS) -o $@ $< -c
	@echo " CC      "$< ...
$(obj)%.o:	%.c
	@$(CC)  $(ALL_CFLAGS) -o $@ $< -c
	@echo " CC      "$< ...

#########################################################################

# If the list of objects to link is empty, just create an empty built-in.o
cmd_link_o_target = $(if $(strip $1),\
		      @$(LD) $(LDFLAGS) -r -o $@ $1,\
		      rm -f $@; $(AR) rcs $@ )

#########################################################################
```

makefile主要包括3部分

1. 环境变量的配置
2. C/S文件编译规则
3. 链接命令函数

前两部分很简单，不做解释

对于下面调用命令来说

``` makefile
#/brandy/u-boot-2014.07/sunxi_spl/boot0/main/makefile Line 27

$(LIB):	$(OBJS)
	$(call cmd_link_o_target, $(OBJS))
```

展开函数，变成

``` makefile
$(LIB):	$(OBJS)
	 $(if $(strip $(OBJS)),\                     #OBJS是否非空
		  @$(LD) $(LDFLAGS) -r -o $@ $(OBJS),\  #如果非空，则链接*.o，输出libxxx.o目标文件
		  rm -f $@; $(AR) rcs $@ )              #删除目录文件，创建一个空libxxx.o
```

整个函数的作用是：链接所有传入的.o文件，如果为空，就生成一个默认的libxxx.o



### 自动生成依赖

最关键的两句是

```makefile
include $(SRCTREE)/rules.mk
sinclude $(obj).depend # 如果没有找到.depend也不报错
```

那我们看一下rules.mk文件内容

``` makefile
#/brandy/u-boot-2014.07/rules.mk

_depend: .depend

.depend: $(TOPDIR)/spl_make $(SPLDIR)/config.mk $(SRCS)
		@rm -f $@	#删除.depend
		@touch $@   #创建.depend
		@for f in $(SRCS); do \ #遍历.depend
			g=`basename $$f | sed -e 's/\(.*\)\.[[:alnum:]_]/\1.o/'`; \
			$(CC) -M $(ALL_CFLAGS) -MQ $(obj)$$g $$f >> $@ ; \
		done
```

第9行不太好理解，其中sed应用了反向引用的技巧，`\( XXX \)`其中的xxx被标记为`\1`，所以sed的意思是替换XXX.nnn_为XXX.o

第10行是将多个源文件的依赖内容写入.depend文件里面

同时，这里有个关键变量$(SRCS)，而其定义是

```makefile
SRCS	:= $(SOBJS:.o=.S) $(COBJS:.o=.c)      #根据*.o生成*.c文件列表
```

所以表示当前目录下的所有源文件


至此，整个uboot-spl编译流程已经分析完
