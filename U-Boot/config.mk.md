# 根目录`conifg.mk`文件解析

```
AS	= $(CROSS_COMPILE)as
LD	= $(CROSS_COMPILE)ld
CC	= $(CROSS_COMPILE)gcc
CPP	= $(CC) -E
AR	= $(CROSS_COMPILE)ar
NM	= $(CROSS_COMPILE)nm
LDR	= $(CROSS_COMPILE)ldr
STRIP	= $(CROSS_COMPILE)strip
OBJCOPY = $(CROSS_COMPILE)objcopy
OBJDUMP = $(CROSS_COMPILE)objdump
RANLIB	= $(CROSS_COMPILE)RANLIB
```
- 根据主Makefile中`CROSS_COMPILE`定义，默认情况下`CROSS_COMPILE = /usr/local/arm/arm-2009q3/bin/arm-none-linux-gnueabi-`，执行上面语句相当于在当前变量基础上添加交叉编译后缀  
即：  

```
AS = /usr/local/arm/arm-2009q3/bin/arm-none-linux-gnueabi-as
LD = /usr/local/arm/arm-2009q3/bin/arm-none-linux-gnueabi-ld
CC = /usr/local/arm/arm-2009q3/bin/arm-none-linux-gnueabi-gcc
CPP = /usr/local/arm/arm-2009q3/bin/arm-none-linux-gnueabi-gcc -E
AR = /usr/local/arm/arm-2009q3/bin/arm-none-linux-gnueabi-ar
...
...
```
---
```
# Load generated board configuration
sinclude $(OBJTREE)/include/autoconf.mk
```
- 这个文件作用是用来指导整个uboot编译过程。内容是很多CONFIG_开头的宏（可理解为变量），这些宏/变量会影响uboot编译过程走向（原理是条件编译）。在uboot代码中有很多地方使用条件编译进行编写，这个条件编译是用来实现可移植性的。（可以说uboot的源码通过这些条件编译对不同的开发板进行适配）
- 经过`make`后，在./include目录下自动生成autoconf.mk脚本。这些文件中的内容来源于`include/configs/xxx.h`(开发板的移植文件,此目录下每个文件对应于一个开发板)

---
```
# Load generated board configuration
sinclude $(OBJTREE)/include/autoconf.mk

ifdef	ARCH
sinclude $(TOPDIR)/lib_$(ARCH)/config.mk	# include architecture dependend rules
endif
ifdef	CPU
sinclude $(TOPDIR)/cpu/$(CPU)/config.mk		# include  CPU	specific rules
endif
ifdef	SOC
sinclude $(TOPDIR)/cpu/$(CPU)/$(SOC)/config.mk	# include  SoC	specific rules
endif
ifdef	VENDOR
BOARDDIR = $(VENDOR)/$(BOARD)
else
BOARDDIR = $(BOARD)
endif
ifdef	BOARD
sinclude $(TOPDIR)/board/$(BOARDDIR)/config.mk	# include board specific rules
endif
```
- 根据Makefile文件中配置的`ARCH CPU SOC VENDOR BOARD`添加相关目录下的`config.mk`脚本

---
#### TEXT_BASE  
```
ifndef LDSCRIPT
#LDSCRIPT := $(TOPDIR)/board/$(BOARDDIR)/u-boot.lds.debug
ifeq ($(CONFIG_NAND_U_BOOT),y)
LDSCRIPT := $(TOPDIR)/board/$(BOARDDIR)/u-boot-nand.lds
else
LDSCRIPT := $(TOPDIR)/board/$(BOARDDIR)/u-boot.lds
endif
endif
```
- 执行 `make sbc2410x_config`时，在board/xxx/xxx目录下生成了config.mk，内容是：`TEXT_BASE = 0x.....`，相当于一个变量
	`TEXT_BASE`是将来我们整个uboot链接时指定的链接地址。因为uboot中启用了虚拟地址映射，因此，这个地址要取决于uboot中做的虚拟地址映射关系
- 若没有定义`LDSCRIPT`，则判断是否`CONFIG_NAND_U_BOOT`为`y`。根据`CONFIG_NAND_U_BOOT`决定LDSCRIPT取哪个链接文件作为自身的值

---
- 自动推导规则