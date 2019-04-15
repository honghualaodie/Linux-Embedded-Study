# start_armboot 函数解析

---

#### gd bd
`DECLARE_GLOBAL_DATA_PTR;`  
- 本函数上面有这样的宏定义  
`#define DECLARE_GLOBAL_DATA_PTR register volatile gd_t *gd asm ("r8")`  
- 上面的定义是在 include\asm-arm\global_data.h 中
- 定义了一个全局变量名字叫 gd ，是一个指针类型，占4个字节。`register`修饰表示这个变量要尽量放到寄存器中，用`volatile`修饰表示这是可变的，asm("r8") 是 gcc 支持的一种语法，意思是要把 gd 放到寄存器 r8 中
- `DECLARE_GLOBAL_DATA_PTR;`就是定义了一个要放在寄存器 r8 中的全局变量，名为 gd ，类型是一个指向 gd_t 类型变量的指针
- 使用`register`修饰的原因是： gd 在 uboot 中很重要，它指向了一个定义的名为 gd_t 结构体，内容是 uboot 常用的全局变量，而且在运行过程中是要被经常访问的，使用`register`修饰是为了提升效率

```C
typedef	struct	global_data {
	volatile bd_t		*bd;
	volatile unsigned long	flags;
	volatile unsigned long	baudrate;
	volatile unsigned long	have_console;	/* serial_init() was called */
	volatile unsigned long	env_addr;	/* Address  of Environment struct */
	volatile unsigned long	env_valid;	/* Checksum of Environment valid? */
	volatile unsigned long	fb_base;	/* base address of frame buffer */
#ifdef CONFIG_VFD
	volatile unsigned char	vfd_type;	/* display type */
#endif
#ifdef CONFIG_FSL_ESDHC
	volatile unsigned long	sdhc_clk;
#endif
#if 0
	unsigned long	cpu_clk;	/* CPU clock in Hz!		*/
	unsigned long	bus_clk;
	phys_size_t	ram_size;	/* RAM size */
	unsigned long	reset_status;	/* reset status register at boot */
#endif
	volatile void		**jt;		/* jump table */
} gd_t;
```
解析：  
- 同样在目录 include\asm-arm\global_data.h 中定义
- `volatile bd_t *bd;`:开发板硬件相关信息  
`volatile unsigned long	flags;`：存放相关标志位 
`volatile unsigned long	baudrate;`：存放控制台串口波特率  
`volatile unsigned long	have_console;`：串口控制台是否建立标志位，其实是一个布尔变量  
`volatile unsigned long	env_addr;`：环境变量结构体地址  
`volatile unsigned long	env_valid;`：内存中环境变量是否能够使用，也是个布尔变量，1为可以使用。启动之初若从 SD 卡启动，环境变量还没有拷贝到 DDR 内存中，则需要到 SD 卡中读取，若已经在 DDR 中运行，则需要在 DDR 中读取环境变量    
`volatile unsigned long	fb_base;`：frame buffer 基地址，即 lcd 基地址  

```C
typedef struct bd_info {
    int			    bi_baudrate;	/* serial console baudrate */
    unsigned long	bi_ip_addr;	/* IP Address */
    struct environment_s *bi_env;
    ulong	        bi_arch_number;	/* unique id for this board */
    ulong	        bi_boot_params;	/* where this board expects params */
    struct			/* RAM configuration */
    {
	ulong start;
	ulong size;
    } bi_dram[CONFIG_NR_DRAM_BANKS];
} bd_t;
```
解析：  
- 在目录 include\asm-arm\u-boot.h 中定义
- `int bi_baudrate;`：定义了开发板硬件的波特率，与 gd_t 中定义的控制台波特率定位不同，但实际是一样的，因为控制台波特率是建立在硬件基础上的
- `unsigned long	bi_ip_addr;`：开发板 IP 地址  
- `struct environment_s *bi_env;`：
- `ulong bi_arch_number;`：机器码
- `ulong bi_boot_params;`：启动参数地址，是 uboot 启动内核是传给 kernel 的参数的地址
- `bi_dram[CONFIG_NR_DRAM_BANKS]` ：保存有 DDR 信息，开发板根据这个结构体知道板子上面有几片 DDR ，以及 DDR 容量和地址

#### 内存分配
```C
/* Pointer is writable since we allocated a register for it */
gd = (gd_t*)(_armboot_start - CONFIG_SYS_MALLOC_LEN - sizeof(gd_t));
/* compiler optimization barrier needed for GCC >= 3.4 */
__asm__ __volatile__("": : :"memory");

memset ((void*)gd, 0, sizeof (gd_t));
gd->bd = (bd_t*)((char*)gd - sizeof(bd_t));
memset (gd->bd, 0, sizeof (bd_t));
```
解析：  
- 内存分配待分析
- `__asm__`是 gcc 内嵌汇编，即 C 和汇编混合汇编的关键字。`__volatile__`为 C 语言的 volatile 。此句代码意义是：当 gcc 编译器版本大于 3.4 时，为了防止程序优化导致不可预知错误
- 使用 `memset`对 gd 和 bd 进行复位清零，防止后面使用时出现不可预知的问题。新申请的内存不一定是干净的，清零是必须的

#### 硬件初始化

`init_fnc_t **init_fnc_ptr;`  
- 在本函数上面有定义：`typedef int (init_fnc_t) (void);`，这样的定义为函数类型定义， init_fnc_ptr 是一个二重函数指针，用来指向一重指针，以及用来指向指针数组，即可以指向一个保存有函数指针的数组，这样操作就可以很方便的通过这个指针来访问不同的函数，这么搞可以方便后续添加具体的实现函数而不需要修改其它部分的代码  

```C
for (init_fnc_ptr = init_sequence; *init_fnc_ptr; ++init_fnc_ptr) {
	if ((*init_fnc_ptr)() != 0) {
		hang ();
	}
}
```
```C
void hang (void)
{
	puts ("### ERROR ### Please RESET the board ###\n");
	for (;;);
}

```
解析：  
- 按顺序执行 init_fnc_ptr 指针指向的 init_sequence 函数指针数组元素（即数组中每个元素即为函数地址指针），逐个执行硬件（开发板上其它硬件）初始化
- for(;;) 循环中 `*init_fnc_ptr` 也可写为 `*init_fnc_ptr != NULL`，即若 init_sequence 数组中元素为 NULL ，则跳出 for(;;) 循环  
- 若`*init_fnc_ptr`指向的指针函数执行返回非0值，则执行`hang ()`，即执行初始化失败，则 uboot 启动失败  

```C
init_fnc_t *init_sequence[] = {
#if defined(CONFIG_ARCH_CPU_INIT)
	arch_cpu_init,		/* basic arch cpu dependent setup */
#endif
	board_init,		    /* basic board dependent setup */
//#if defined(CONFIG_USE_IRQ)
	interrupt_init,		/* set up exceptions */
//#endif
	//timer_init,		/* initialize timer */
#ifdef CONFIG_FSL_ESDHC
	//get_clocks,
#endif
	env_init,		    /* initialize environment */
	init_baudrate,		/* initialze baudrate settings */
	serial_init,		/* serial communications setup */
	console_init_f,		/* stage 1 init of console */
	off_charge,		    // xiebin.wang @ 20110531,for charger&power off device.

	display_banner,		/* say that we are here */
#if defined(CONFIG_DISPLAY_CPUINFO)
	print_cpuinfo,		/* display cpu info (and speed) */
#endif
#if defined(CONFIG_DISPLAY_BOARDINFO)
	checkboard,		    /* display board info */
#endif
#if defined(CONFIG_HARD_I2C) || defined(CONFIG_SOFT_I2C)
	//init_func_i2c,
#endif
	dram_init,		    /* configure available RAM banks */
#if defined(CONFIG_CMD_PCI) || defined (CONFIG_PCI)
	//arm_pci_init,
#endif
	display_dram_config,
	NULL,
};
```
解析：  
- 此数组同样保存在 board.c 中
- 本数组中保存的元素均为 init_fnc_t 类型函数指针，返回为 int 类型，参数为 void
- `arch_cpu_init`：目录(cpu\arm_cortexa9\s5pc210\cpu_info.c)，由于关于 CPU 初始化已经在汇编部分执行过了，本函数实际并未做任何实际的工作
- `board_init`：目录 board\samsung\smdkc210\smdkc210.c 中  

#### 背景：  
#### 关于DDR的配置：  
```C
#define MEMORY_BASE_ADDRESS	0x40000000

#define CONFIG_NR_DRAM_BANKS    8          	/* 8 banks of DRAM at maximum */

//dg change for kinds of coreboard 2015-08-04
#if  defined(CONFIG_SCP_1GDDR_Ubuntu) || defined(CONFIG_POP_2GDDR_Ubuntu)
   #define SDRAM_BANK_SIZE   0x10000000/2	/* each bank has 128 MB */
#elif defined(CONFIG_SCP_2GDDR_Ubuntu) ||  defined(CONFIG_POP_1GDDR_Ubuntu)
   #define SDRAM_BANK_SIZE   0x10000000	    /* each bank has 256 MB */
#endif

#define PHYS_SDRAM_1            (unsigned long)MEMORY_BASE_ADDRESS /* SDRAM Bank #1 */
#define PHYS_SDRAM_1_SIZE       (unsigned long)SDRAM_BANK_SIZE
#define PHYS_SDRAM_2            (unsigned long)(MEMORY_BASE_ADDRESS + SDRAM_BANK_SIZE) /* SDRAM Bank #2 */
#define PHYS_SDRAM_2_SIZE       (unsigned long)SDRAM_BANK_SIZE
#define PHYS_SDRAM_3            (unsigned long)(MEMORY_BASE_ADDRESS + 2 * SDRAM_BANK_SIZE) /* SDRAM Bank #2 */
#define PHYS_SDRAM_3_SIZE       (unsigned long)SDRAM_BANK_SIZE
#define PHYS_SDRAM_4            (unsigned long)(MEMORY_BASE_ADDRESS + 3 * SDRAM_BANK_SIZE) /* SDRAM Bank #2 */
#define PHYS_SDRAM_4_SIZE       (unsigned long)SDRAM_BANK_SIZE
#define PHYS_SDRAM_5            (unsigned long)(MEMORY_BASE_ADDRESS + 4 * SDRAM_BANK_SIZE) /* SDRAM Bank #2 */
#define PHYS_SDRAM_5_SIZE       (unsigned long)SDRAM_BANK_SIZE
#define PHYS_SDRAM_6            (unsigned long)(MEMORY_BASE_ADDRESS + 5 * SDRAM_BANK_SIZE) /* SDRAM Bank #2 */
#define PHYS_SDRAM_6_SIZE       (unsigned long)SDRAM_BANK_SIZE
#define PHYS_SDRAM_7            (unsigned long)(MEMORY_BASE_ADDRESS + 6 * SDRAM_BANK_SIZE) /* SDRAM Bank #2 */
#define PHYS_SDRAM_7_SIZE       (unsigned long)SDRAM_BANK_SIZE
#define PHYS_SDRAM_8            (unsigned long)(MEMORY_BASE_ADDRESS + 7 * SDRAM_BANK_SIZE) /* SDRAM Bank #2 */
#define PHYS_SDRAM_8_SIZE       (unsigned long)SDRAM_BANK_SIZE
```
(1)注意：这里的初始化DDR和汇编阶段lowlevel_init中初始化DDR是不同的。当时是硬件的初始化，目的是让DDR可以开始工作。现在是软件结构中一些DDR相关的属性配置、地址设置的初始化，是纯软件层面的。
(2)软件层次初始化DDR的原因：对于uboot来说，他怎么知道开发板上到底有几片DDR内存，每一片的起始地址、长度这些信息呢？在uboot的设计中采用了一种简单直接有效的方式：程序员在移植uboot到一个开发板时，程序员自己在 include\configs\itop_4412_ubuntu.h 中使用宏定义去配置出来板子上DDR内存的信息，然后uboot只要读取这些信息即可。（实际上还有另外一条思路：就是uboot通过代码读取硬件信息来知道DDR配置，但是uboot没有这样。实际上PC的BIOS采用的是这种）
(3)include\configs\itop_4412_ubuntu.h 使用了标准的宏定义来配置DDR相关的参数。主要配置了这么几个信息：有几片DDR内存、每一片DDR的起始地址、长度。这里的配置信息我们在uboot代码中使用到内存时就可以从这里提取使用（想象uboot中使用到内存的地方都不是直接用地址数字的，都是用宏定义的）

```C
int board_init(void)
{
DECLARE_GLOBAL_DATA_PTR;
#ifdef CONFIG_DRIVER_SMC911X
	smc9115_pre_init();
#endif

#ifdef CONFIG_DRIVER_DM9000
//	dm9000_pre_init();
#endif

	gd->bd->bi_arch_number = MACH_TYPE;
	gd->bd->bi_boot_params = (PHYS_SDRAM_1+0x100);

	return 0;
}
```
(1) `DECLARE_GLOBAL_DATA_PTR`定义为宏是为了后面使用 gd 更方便。这个宏相当于 C 语言中的全局变量，在工程其它地方使用时，必须做声明才能使用
(2)`dm9000_pre_init()`是用来对网卡 GPIO 和端口进行初始化，具体网卡的驱动是现成的，这部分只对板级 GPIO 和端口进行初始化，用于应对后面的驱动使用
(3)`gd->bd->bi_boot_params = (PHYS_SDRAM_1+0x100)`即`0x40000100`，表示是 uboot 给 linux kernel启动时的传参的内存地址，uboot 事先将参数（字符串，即 bootargs）准备好，并存放到内存的 gd->bd->bi_boot_params 处，之后 uboot 启动内核（uboot 在启动内核时通过寄存器 r0 r1 r2直接传递参数，其中一个寄存器就是 bi_boot_params），内核启动并从寄存器中读取 bi_boot_params 就知道了 uboot 传递的参数到底在内存的哪里，然后取内存指定的地址中找 bootargs  
(4)`gd->bd->bi_arch_number = MACH_TYPE`是开发板的机器码，是 uboot 给这个开发板定义的一个唯一编号  
机器码主要作用就是在 uboot 和 linux 内核之间进行比对和适配  
嵌入式设备每一个设备的硬件都是定制化的，不能通用。Linux给每个开发板做了唯一编号（机器码），然后子啊 uboot 和 linux 内核中都有一个软件维护的机器码，然后开发板、uboot、linux三者取比对机器码，若机器码比对成功则启动  
uboot 机器码在 include\configs\itop_4412_ubuntu.h 中定义：`#define MACH_TYPE 2838`，此编号理论上应该提交给 uboot 官方审核并发放机器码，现实中其实是很少有申请的。做产品时只需要保证 uboot 和 linux kernel 编号一致就不影响自己板子的启动  
另外需要说明的是：机器码会作为参数一部分传给 linux 用于比对  

- `interrupt_init`，目录 cpu\arm_cortexa9\s5pc210\interrupts.c，实际上本函数是用来初始化定时器的（实际使用的是 Timer4，本定时器被设计用来做计时使用）  
Timer4用来做计时时要使用到2个寄存器：TCNTB4、TCNTO4。TCNTB中存了一个数，这个数就是定时次数（每一次时间是由时钟决定的，其实就是由2级时钟分频器决定的）。我们定时时只需要把定时时间/基准时间=数，将这个数放入TCNTB中即可；我们通过TCNTO寄存器即可读取时间有没有减到0，读取到0后就知道定的时间已经到了。
使用Timer4来定时，因为没有中断支持，所以CPU不能做其他事情同时定时，CPU只能使用轮询方式来不断查看TCNTO寄存器才能知道定时时间到了没。因为Timer4的定时是不能实现微观上的并行。uboot中定时就是通过Timer4来实现定时的。所以uboot中定时时不能做其他事（考虑下，典型的就是bootdelay，bootdelay中实现定时并且检查用户输入是用轮询方式实现的，原理类似轮询方式处理按键）
interrupt_init函数将timer4设置为定时10ms。关键部位就是get_PCLK函数获取系统设置的PCLK_PSYS时钟频率，然后设置TCFG0和TCFG1进行分频，然后计算出设置为10ms时需要向TCNTB中写入的值，将其写入TCNTB，然后设置为auto reload模式，然后开定时器开始计时就没了。
- `env_init`
