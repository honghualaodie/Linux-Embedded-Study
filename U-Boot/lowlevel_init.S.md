# lowlevel_init 解析 

---

```ASM
#if 1//*****ly
	/* use iROM stack in bl2 */
	ldr	sp, =0x02060000
#endif
	push	{lr}
```
- 定义栈空间内存地址，且设置为内部 SRAM 中，并保存当前地址，为方便后面返回start.S（为何不在start.S中定义 sp？？？）  

```ASM
	/* check reset status  */
	ldr     r0, =(INF_REG_BASE + INF_REG1_OFFSET)
    ldr     r1, [r0]

	/* AFTR wakeup reset */
	ldr	r2, =S5P_CHECK_DIDLE
	cmp	r1, r2
	beq	exit_wakeup

	/* Sleep wakeup reset */
	ldr	r2, =S5P_CHECK_SLEEP
	cmp	r1, r2
	beq	wakeup_reset
```
- 判断时哪种复位情况-冷上电、热启动、睡眠状态下的唤醒等，这些情况都属于复位  
判断启动情况目的在于：冷上电是需要 DDR 初始化才能使用的，而热启动或睡眠唤醒则不需要再次初始化 DDR（并未关机，且之前已经初始化过了）  

```ASM
    /* PS-Hold high */
    ldr r0, =0x1002330c
    ldr r1, [r0]
    orr r1, r1, #0x300
    str r1, [r0]

    ldr     r0, =0x11000c08
    ldr r1, =0x0
    str r1, [r0]

    /* Clear  MASK_WDT_RESET_REQUEST  */
    ldr r0, =0x1002040c
    ldr r1, =0x00
    str r1, [r0]
```
- 对电源保持引脚进行设置，保持供电锁存  
- 关看门狗

```ASM
#ifdef check_mem /*liyang 20110822*/
	/* when we already run in ram, we don't need to relocate U-Boot.
	 * and actually, memory controller must be configured before U-Boot
	 * is running in ram.
	 */
	ldr	r0, =0xff000fff
	bic	r1, pc, r0		/* r0 <- current base addr of code */
	ldr	r2, _TEXT_BASE  /* r1 <- original base addr in ram */
	bic	r2, r2, r0		/* r0 <- current base addr of code */
	cmp     r1, r2      /* compare r0, r1                  */
	beq     1f			/* r0 == r1 then skip sdram init   */
#endif
```
- 判断是否需要代码重定位，即判断代码是在哪里执行的（SRAM or DDR？），因为 CPU 可能是冷启动也可能是热启动，冷启动必然是 SRAM 中运行，热启动当前是 DDR 中运行。若冷启动情况，则执行时钟以及DDR初始化，反之则不用初始化，直接跳过  
- r1 是当前执行代码的程序指针 PC 的地址值与  0xff000fff 执行与操作，只取(12-24)bit并保存到 r1  
r2 是 `_TEXT_BASE`-链接地址，即 /board/samsung/smdkc210 目录下的 conifg.mk 中的 `TEXT_BASE = 0xc3e00000`，然后此值与 0xff000fff进行与操作，同样取(12-24)bit并保存到r2  
取出链接地址以及当前程序运行地址进行比较，若两个地址不相等，则在 SRAM 中运行，否则在 DDR 中运行  
####为何要和 0xff000fff 做与操作呢？####
若在 DDR 中运行，程序已经运行了一小段，当前地址，即 PC 指向的地址并不会完全等于链接地址（链接地址是程序在 DDR 中一开始运行的地址），此处做与操作就是要屏蔽或忽略掉程序运行一小段这种情况的影响，也就是说，程序大体上地址可以比对的上，则代表是在DDR中运行的  
0xff000fff 低位屏蔽了12bit，即 4096（4K），直接跳转屏蔽了4K的程序  

```ASM
/* Memory initialize */
bl mem_ctrl_asm_init

/* init system clock */
bl system_clock_init

bl tzpc_init
```
- 内存和系统时钟初始化  
- `system_clock_init`在本文件中定义，并且初始化过程调用了很多`configs/itop_4412_ubuntu.h`中的宏
- `mem_ctrl_asm_init`位置在`cpu/arm_cortexa9/s5pc210`目录下的 cpu_init_POP.S 文件中定义
- `tzpc_init`
`bl uart_asm_init`，本初始化在本文件中定义，用于初始化串口，并且执行完毕会向串口打印 0x4f4f4f4f ('O')，用于方便调试  
#####当执行完 lowlevel_init 生于代码后，会通过串口打印0x4b4b4b4b('K')，即执行完毕 lowlevel_init 后会通过串口打印'OK'#####
