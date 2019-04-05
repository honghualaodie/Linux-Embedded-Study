# start.S 分析

---

#### 为何整个 uboot 程序的第一个运行的代码是 start.S 文件？  
整个程序入口取决于连接脚本中ENTRY声明的地方:`ENTRY(_start)`。因此`_start`符号所在的文件就是整个程序的起始文件，_start所在处的代码就是整个程序的起始代码。  而在start.S文件中有这样的定义：  
```ASM
.globl _start
_start: b reset
```

---

#### 包含的头文件分析
```C
#include <config.h>
#include <version.h>
#if defined(CONFIG_ENABLE_MMU)
#include <asm/proc/domain.h>
#endif
#if defined(CONFIG_S5PV310)
#include <s5pv310.h>
#endif
#if defined(CONFIG_S5PC210)
#include <s5pc210.h>
#endif

#ifndef CONFIG_ENABLE_MMU
#ifndef CFG_PHY_UBOOT_BASE
#define CFG_PHY_UBOOT_BASE	CFG_UBOOT_BASE
#endif
#endif
```
