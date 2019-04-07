# 根目录mkconfig文件分析

---
#### 在主Makefile文件中的定义
在主Makefile文件中有类似下面这样的定义：

```bash
itop_4412_ubuntu_config_scp_2GDDR:	unconfig
	@$(MKCONFIG) $(@:_config=) arm arm_cortexa9 smdkc210 samsung s5pc210 POP_1GDDR_Ubuntu
```

```
unconfig:
	@rm -f $(obj)include/config.h $(obj)include/config.mk \
		$(obj)board/*/config.tmp $(obj)board/*/*/config.tmp \
		$(obj)include/autoconf.mk $(obj)include/autoconf.mk.dep
```
### 
`MKCONFIG`在主Makefile中是下面这样定义的
```bash
MKCONFIG	:= $(SRCTREE)/mkconfig
export MKCONFIG
```
- 即`MKCONFIG`为根目录下的mkconfig脚本，并且`MKCONFIG`声明了全局变量，方便后面全局使用

当在uboot根目录运行`make itop_4412_ubuntu_config_scp_2GDDR`命令，会运行上面的命令， `itop_4412_ubuntu_config_scp_2GDDR` 依赖于unconfig，并且执行  
`@$(MKCONFIG) $(@:_config=) arm arm_cortexa9 smdkc210 samsung s5pc210`  
注释：  
- `@`:代表该指令需要静默执行(不打印输出到控制台)  
- `MKCONFIG`:根据上面描述为根目录mkconfig脚本
-  `$(@:_config=)`:`$@`为目标，此处即`$@` = `itop_4412_ubuntu_config_scp_2GDDR`。`：`代表前面的字符串需要进行处理。`=`是将前面字符串`_config`替换成后面的字符串`空`。  
本命令意思是：将目标的`_config`部分替换成`空`: `itop_4412_ubuntu_scp_2GDDR`

根据注释中的描述，可以将上面的命令解释为：  
```bash
@rm -f $(obj)include/config.h $(obj)include/config.mk \
		$(obj)board/*/config.tmp $(obj)board/*/*/config.tmp \
		$(obj)include/autoconf.mk $(obj)include/autoconf.mk.dep \
		@$(SRCTREE)/mkconfig itop_4412_ubuntu_scp_2GDDR arm arm_cortexa9 smdkc210 samsung s5pc210
```

---

执行上面指令，会向mkconfig脚本传入参数：  

`$(@:_config=) arm arm_cortexa9 smdkc210 samsung s5pc210 POP_1GDDR_Ubuntu`  

$1: itop_4412_ubuntu_scp_2GDDR → $(@:_config=)  
$2: arm  
$3: arm_cortexa9  
$4: smdkc210  
$5: samsung  
$6: s5pc210  
$7: POP_1GDDR_Ubuntu

$# = 6(共传入六个参数)

---

预备知识
```bash
-eq 	//等于  
-ne 	//不等于  
-gt 	//大于 （greater ）  
-lt 	//小于  （less）  
-ge 	//大于等于  
-le 	//小于等于  
```

#### 参数数量合法性判断
```bash
while [ $# -gt 0 ] ; do
	case "$1" in
	--) shift ; break ;;
	-a) shift ; APPEND=yes ;;
#	-n) shift ; BOARD_NAME="${1%%_config}" ; shift ;;
	-t) shift ; TARGETS="`echo $1 | sed 's:_: :g'` ${TARGETS}" ; shift ;;
	*)  break ;;
	esac
done
```
解释：  
当传入的参数大于0时，执行下面的语句。  
--) 匹配--  
-a) 匹配-a  
*)  相当于别的语言的条件分支的default  
shift就是用掉一个位置参数，那么原来的$2就变成现在的$1了

实际传入的7个参数中在这个case条件选择中并不能找到匹配项，则会执行`*)  break ;;`条件分支，则break直接跳出这个while循环

###  

`[ "${BOARD_NAME}" ] || BOARD_NAME="$1"`  
解释：  
若`BOARD_NAME`定义了，即为真，则不执行后面的`BOARD_NAME="$1"`。  
因为这里使用的是`||`或运算，前面指令若为真，则不执行后面的语句。若使用`&&`与运算，则前面指令若为真，后面指令执行 

```bash
[ $# -lt 4 ] && exit 1
[ $# -gt 7 ] && exit 1
```
解释：  
这里显然在判断传入的参数个数是否满足大于4及小于7的情况，也就是参数个数合法性判断，若判断为非法情况，则直接执行`exit 1`结束后面的指令执行返回错误退出  

---

#### 创建符号链接文件

注：创建符号链接文件作用是给头文件包含等过程提供指向性链接，目的是让uboot具有可以执行。由于uboot要适配不同开发板及不同cpu，在实际使用时需要指定一个cpu及开发板，这时，创建符号链接就起到了作用。说到底，使用这种方式就是为了兼容不同开发板和cpu而设计的。  
实现方式就是根据主Makefile中传入的7个参数在include/目录下选择提前适配好的各个开发板文件夹，类似C语言指针一样，需要使用哪个开发板，根据传入的7个参数配置指针指向对应的开发板的文件夹，后面在编写代码时，只需要统一使用符号链接文件即可，因为符号链接文件目录名称都是相同的，这样就做到了兼容  

```bash
if [ "$SRCTREE" != "$OBJTREE" ] ; then
	mkdir -p ${OBJTREE}/include
	mkdir -p ${OBJTREE}/include2
	cd ${OBJTREE}/include2
	rm -f asm
	ln -s ${SRCTREE}/include/asm-$2 asm
	LNPREFIX="../../include2/asm/"
	cd ../include
	rm -rf asm-$2
	rm -f asm
	mkdir asm-$2
	ln -s asm-$2 asm
else
	cd ./include
	rm -f asm
	ln -s asm-$2 asm
fi
rm -f asm-$2/arch
```
解释：  
由于一般使用的是本地编译，即不使用指定编译目录方式，则`SRCTREE`与`OBJTREE`两个变量目录是相同的，即均为uboot根目录，则本段代码直接执行else部分  
- 进入./include目录
- 删除asm文件夹
- 创建名称为asm的链接，并指向到asm-arm文件夹
- 删除asm-arm/arch

```bash
if [ -z "$6" -o "$6" = "NULL" ] ; then
	ln -s ${LNPREFIX}arch-$3 asm-$2/arch
else
	ln -s ${LNPREFIX}arch-$6 asm-$2/arch
fi

if [ "$2" = "arm" ] ; then
	rm -f asm-$2/proc
	ln -s ${LNPREFIX}proc-armv asm-$2/proc
fi
```
解释：  
- 若$6(即s5pc210)长度为零或等于NULL为真。  
显然$6长度是非零和非NULL情况，则直接执行else分支
- `LNPREFIX`到目前为止未定义，为空，则变成这样的语句`ln -s arch-s5pc210 asm-arm/arch`，即在asm-arm/目录下创建名称为arch的链接，并指向到arch-s5pc210文件夹

- 若$2(arm)为arm，则执行后面的语句
- 删除asm-arm/proc  
在asm-arm/目录下创建名称为proc的链接，并指向到proc-armv文件夹

---
#### 创建config.mk文件
```bash
echo "ARCH   = $2" >  config.mk
echo "CPU    = $3" >> config.mk
echo "BOARD  = $4" >> config.mk

[ "$5" ] && [ "$5" != "NULL" ] && echo "VENDOR = $5" >> config.mk

[ "$6" ] && [ "$6" != "NULL" ] && echo "SOC    = $6" >> config.mk
```
解释：  
此处将$2 $3 $4 $5 $6输出到config/目录下的config.mk文件，原因是在主Makefile中对这个文件有包含并且包含本文件后，将包含到的这些变量变为全局变量，如下：  
```bash
include $(obj)include/config.mk
export	ARCH CPU BOARD VENDOR SOC
```
注：这样做的目的同样是为了兼容性

```bash
if [ -z "$5" -o "$5" = "NULL" ] ; then
    BOARDDIR=$4
else
    BOARDDIR=$5/$4
fi
```
解释：若$5(为samsung)长度为0或为NULL则执行`BOARDDIR=$4`，显然此处执行的是`BOARDDIR=$5/$4`，即`BOARDDIR=samsung/smdkc210`

```bash
if [ "$APPEND" = "yes" ]	# Append to existing config file
then
	echo >> config.h
else
	> config.h		# Create new config file
fi
```
解释：若`APPEND`值为`yes`，则执行`echo >> config.h`，在此文件中一开始就定义了下面的指令，则此处执行`> config.h`，创建config.h文件  
`APPEND=no	# Default: Create new config file`

`[ "$7" ] && [ "$7" != "NULL" ] && echo "#define CONFIG_$7" >> config.h`  
解释：若$7(为 POP_1GDDR_Ubuntu)长度大于零并且不为NULL，则向config.h文件中写入字符串  
此处应该写入config.h的字符串为：`#define CONFIG_POP_1GDDR_Ubuntu`

```bash
cat << EOF >> config.h
#define CONFIG_BOARDDIR board/$BOARDDIR
#include <config_defaults.h>
#include <configs/$BOARD_NAME.h>
#include <asm/config.h>
EOF
```
解释：  
- `cat << EOF`意思是告诉shell脚本，从现在开始执行命令，当遇到EOF时结束
- `>> config.h`意思是执行命令时所有操作都追加到config.h文件中

注：#include <configs/$BOARD_NAME.h>这个文件很重要，需要细致分析