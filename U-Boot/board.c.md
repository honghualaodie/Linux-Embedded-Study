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
- `ulong bi_boot_params;`：启动参数地址，是 uboot 启动内核时传给 kernel 的参数的地址
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

---
---

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
- `env_init`，一般从哪里启动就会把环境变量 env 放到哪里，各种介质存取操作 env 方法都不一样，因此 uboot 支持了各种不同介质中 env 的操作方法，这需要根据自己实际使用的存储介质来选择，而选择则通过配置文件 include\configs\itop_4412_ubuntu.h 来进行配置  
本函数内容是对内存中维护的那份 uboot 的 env 做了基本初始化（判定指定地址的内存中是否有可用的环境变量），当前代码阶段，我们还没有从 SD 卡到 DDR 中的 relocate 重定位，则内存中的环境变量是不能够使用的  
在 start_armboot 函数中的后面会通过调用 env_relocate() 函数对环境变量进行重定位，重定位后即可从指定的 DDR 中获取环境变量了
- `init_baudrate`是初始化窗口波特率，且是在本文件中定义  
```C
static int init_baudrate (void)
{
	char tmp[64];	/* long enough for environment variables */
	int i = getenv_r ("baudrate", tmp, sizeof (tmp));
	gd->bd->bi_baudrate = gd->baudrate = (i > 0)
			? (int) simple_strtoul (tmp, NULL, 10)
			: CONFIG_BAUDRATE;

	return (0);
}
```
(1) getenv_r(,,)用于从环境变量读取需要读取的变量，第一个参数是需要读取的参数，第二个参数是都出的数，第三个是数的长度大小  
读出的 `baudrate`的值（都出的数为字符串类型），使用 simple_strtoul(,,)转换成 int 类型。若读出的数大于0，则将读出的数保存到 gd 和 bd 关于波特率的变量中，否则保存为 include\configs\itop_4412_ubuntu.h 中设置的 CONFIG_BAUDRATE(#define CONFIG_BAUDRATE 115200)  
- `serial_init`，目录 cpu\arm_cortexa9\s5pc210\serial.c ，实际本函数并没有什么实质性的内容，只是进行了for(;;)空循环而已，由于在汇编部分已经对串口进行了初始化  
- `console_init_f`在 common\console.c 中，本函数是控制台的第一阶段初始化，_f表示是第一阶段初始化，_r表示第二阶段初始化。有时候初始化函数不能一次完成，在 start_armboot 函数中进行了第二次初始化 console_init_r()。本函数只是对 gd->have_console 设置为1  
控制台就是一个用软件虚拟出来的设备，这个设备有一套专用的通信函数（发送、接收···），控制台的通信函数最终会映射到硬件的通信函数中来实现。 uboot 中实际上控制台的通信函数是直接映射到硬件串口的通信函数中的，也就是说 uboot 中用没用控制器其实并没有本质差别  
控制台的通信函数映射到硬件通信函数时可以用软件来做一些中间优化，譬如说缓冲机制。（操作系统中的控制台都使用了缓冲机制，所以有时候我们 printf 了内容但是屏幕上并没有看到输出信息，就是因为被缓冲了。我们输出的信息只是到了 console 的 buffer 中， buffer 还没有被刷新到硬件输出设备上，尤其是在输出设备是 LCD 屏幕时  

#### printf() → puts() → serial_puts() → serial_putc() → 硬件寄存器输出 ####
- `display_banner`，即在控制台打印 logo  
- `print_cpuinfo`，目录 cpu\arm_cortexa9\s5pc210\cpu_info.c ，根据 include\configs\itop_4412_ubuntu.h 定义的宏打印相关信息，比较容易分析，略过
- `checkboard`，用于检查当前开发板是哪个开发板，并且打印开发板的名字  
`printf("Board:	%s%s\n", CONFIG_DEVICE_STRING,CORE_NUM_STR);`，`CONFIG_DEVICE_STRING`在 include\configs\itop_4412_ubuntu.h 中定义  
- `dram_init`，对于 DDR 已经在 uboot 第一部分汇编初始化过了，这里再次初始化是对 gd->bd->bi_dram 全局结构体变量进行初始化，是软件方面的初始化。而 gd->bd 记录的是当前开发板 DDR 的配置信息， uboot 是使用 gd->bd 来使用内存的  

```C
nr_dram_banks = 8;
gd->bd->bi_dram[0].start = PHYS_SDRAM_1;
gd->bd->bi_dram[0].size = PHYS_SDRAM_1_SIZE;
gd->bd->bi_dram[1].start = PHYS_SDRAM_2;
gd->bd->bi_dram[1].size = PHYS_SDRAM_2_SIZE;
gd->bd->bi_dram[2].start = PHYS_SDRAM_3;
gd->bd->bi_dram[2].size = PHYS_SDRAM_3_SIZE;
gd->bd->bi_dram[3].start = PHYS_SDRAM_4;
gd->bd->bi_dram[3].size = PHYS_SDRAM_4_SIZE;
gd->bd->bi_dram[4].start = PHYS_SDRAM_5;
gd->bd->bi_dram[4].size = PHYS_SDRAM_5_SIZE;
gd->bd->bi_dram[5].start = PHYS_SDRAM_6;
gd->bd->bi_dram[5].size = PHYS_SDRAM_6_SIZE;
gd->bd->bi_dram[6].start = PHYS_SDRAM_7;
gd->bd->bi_dram[6].size = PHYS_SDRAM_7_SIZE;
gd->bd->bi_dram[7].start = PHYS_SDRAM_8;
gd->bd->bi_dram[7].size = PHYS_SDRAM_8_SIZE;
```
根据源代码可知，实际运行的就是类似上面的这段代码，其实就是将 include\configs\itop_4412_ubuntu.h 中定义的 DDR 内存的宏对 gd->bd->bi_dram 进行初始化，通过这样， uboot 就可以读取 gd->bd->bi_dram 得知开发板焊接的是多大的 DDR 了  

- `display_dram_config`，用于计算打印当前开发板内存容量信息  

```C
ulong size = 0;

for (i=0; i<CONFIG_NR_DRAM_BANKS; i++) {
	size += gd->bd->bi_dram[i].size;
}

puts("DRAM:	");
print_size(size, "\n");

return (0);
```
根据分析，这个函数实际运行的就是上面的这段代码。其实这段代码就是根据上面`dram_init`函数保存的`gd->bd->bi_dram`进行累加操作计算出 DDR 总容量  

---
---

```C
/* armboot_start is defined in the board-specific linker script */
mem_malloc_init (_armboot_start - CONFIG_SYS_MALLOC_LEN,
		CONFIG_SYS_MALLOC_LEN);
```
```
void mem_malloc_init(ulong start, ulong size)
{
	mem_malloc_start = start;
	mem_malloc_end = start + size;
	mem_malloc_brk = start;

	memset((void *)mem_malloc_start, 0, size);
}
```
- 用来初始化 uboot 堆管理器，经过初始化后，后面就可以使用 malloc、free 进行内存申请和释放。具体在内存中堆的地址分布请参见![内存分布分析][内存分布分析]

`display_flash_config (flash_init ());`  
- `display_flash_config()`负责打印 norflash 容量  
- flash_init ()，用于对 norflash 进行初始化，实际开发板上面并没有 norflash ，使用的是 inand，但是此处初始化并不会影响其它部分的运行，去掉也可以  

注：  
CONFIG_VFD 和 CONFIG_LCD 是显示相关的，这个是 uboot 中自带的 LCD 显示的软件架构。但是实际上我们用 LCD 而没有使用 uboot 中设置的这套软件架构，我们自己在后面自己添加了一个 LCD 显示的部分。实际 CONFIG_VFD 和 CONFIG_LCD 在  include\configs\itop_4412_ubuntu.h 中也并没有定义  。

##### NAND 初始化
```C
#ifdef CONFIG_GENERIC_MMC
	puts ("MMC:   ");
	mmc_exist = mmc_initialize (gd->bd);
	if (mmc_exist != 0)
	{
		puts ("0 MB\n");
	}
#endif
```
- 在 include\configs\itop_4412_ubuntu.h 中定义了 CONFIG_GENERIC_MMC 。 `mmc_initialize()`在目录 drivers\mmc\mmc.c 中定义  
```C
int mmc_initialize(bd_t *bis)
{
	struct mmc *mmc;
	int err,dev;
	
	INIT_LIST_HEAD (&mmc_devices);
	cur_dev_num = 0;

	if (board_mmc_init(bis) < 0)
		cpu_mmc_init(bis);

#if defined(DEBUG_S3C_HSMMC)
	print_mmc_devices(',');
#endif
	for (dev = 0; dev < cur_dev_num; dev++) 
	{
		mmc = find_mmc_device(dev);

		if (mmc) 
		{
			err = mmc_init(mmc);
...
...
...
```
- `INIT_LIST_HEAD`初始化一个链表，用于链接多个 mmc
- `cur_dev_num = 0`代表第0个 mmc 设备
- `board_mmc_init(bis)`实际定义是空函数，并没有执行什么具体代码，直接返回 -1
- `cpu_mmc_init(bis)`，在目录 cpu\arm_cortexa9\cpu.c 
```C
/*
 * Initializes on-chip MMC controllers.
 * to override, implement board_mmc_init()
 */
int cpu_mmc_init(bd_t *bis)
{
	int ret;
#if defined(CONFIG_S3C_HSMMC) || defined(CONFIG_S5P_MSHC)
	setup_hsmmc_clock();
	setup_hsmmc_cfg_gpio();
#endif
#ifdef USE_MMC4
	ret = smdk_s5p_mshc_init();
#endif
#ifdef USE_MMC2
	ret = smdk_s3c_hsmmc_init();
#endif
	return ret;
}
```
- 上面代码中提到的宏均在 include\configs\itop_4412_ubuntu.h 中有定义
- 用于对 cpu 内部寄存器关于 mmc 进行初始化，追踪的话，可以发现，这个函数下调用的函数都是底层配置代码，直接操作 cpu 寄存器的

---

#### 环境变量重定位

1、环境变量的概念

     可以理解为用户对软件的全局配置信息，这部分信息应该可以从永久性存储器上读取，能被查询，能被修改。

　 启动过程中，应该首先把环境变量读取到合适的内存区域，然后利用环境变量初始化硬件、启动操作系统等等。

2、启动过程中环境变量初始化过程涉及的问题

       这里涉及到两个问题：

　　　环境变量在哪个地方存着（从哪个地方取）

　　　将环境变量存储到哪里（放到哪）  

---

```C
/* initialize environment */
env_relocate ();
```

```C
void env_relocate (void)
{
#ifndef CONFIG_RELOC_FIXUP_WORKS
	DEBUGF ("%s[%d] offset = 0x%lx\n", __FUNCTION__,__LINE__,
		gd->reloc_off);
#endif

#ifdef CONFIG_AMIGAONEG3SE
	enable_nvram();
#endif

#ifdef ENV_IS_EMBEDDED
	/*
	 * The environment buffer is embedded with the text segment,
	 * just relocate the environment pointer
	 */
#ifndef CONFIG_RELOC_FIXUP_WORKS
	env_ptr = (env_t *)((ulong)env_ptr + gd->reloc_off);
#endif
	DEBUGF ("%s[%d] embedded ENV at %p\n", __FUNCTION__,__LINE__,env_ptr);
#else
	/*
	 * We must allocate a buffer for the environment
	 */
	env_ptr = (env_t *)malloc (CONFIG_ENV_SIZE);
	DEBUGF ("%s[%d] malloced ENV at %p\n", __FUNCTION__,__LINE__,env_ptr);
#endif

	if (gd->env_valid == 0) {
#if defined(CONFIG_GTH)	|| defined(CONFIG_ENV_IS_NOWHERE)	/* Environment not changable */
		puts ("Using default environment\n\n");
#else
		puts ("*** Warning - bad CRC, using default environment\n\n");
		show_boot_progress (-60);
#endif
		set_default_env();
	}
	else {
		env_relocate_spec ();
	}
	gd->env_addr = (ulong)&(env_ptr->data);

#ifdef CONFIG_AMIGAONEG3SE
	disable_nvram();
#endif
}
```
- 未找到 `ENV_IS_EMBEDDED`有定义，则执行下面代码：  
```C
/*
 * We must allocate a buffer for the environment
 */
env_ptr = (env_t *)malloc (CONFIG_ENV_SIZE);
DEBUGF ("%s[%d] malloced ENV at %p\n", __FUNCTION__,__LINE__,env_ptr);
```
申请并获取环境变量的内存地址  

##### 环境变量从哪里来？  
SD卡中有一些（8个）独立的扇区作为环境变量存储区域的。但是我们烧录/部署系统时，我们只是烧录了uboot分区、kernel分区和rootfs分区，根本不曾烧录env分区。所以当我们烧录完系统第一次启动时ENV分区是空的，本次启动uboot尝试去SD卡的ENV分区读取环境变量时失败（读取回来后进行CRC校验时失败），我们uboot选择从uboot内部代码中设置的一套默认的环境变量出发来使用（这就是默认环境变量）；这套默认的环境变量在本次运行时会被读取到DDR中的环境变量中，然后被写入（也可能是你saveenv时写入，也可能是uboot设计了第一次读取默认环境变量后就写入）SD卡的ENV分区。然后下次再次开机时uboot就会从SD卡的ENV分区读取环境变量到DDR中，这次读取就不会失败了。

```C
	if (gd->env_valid == 0) {
#if defined(CONFIG_GTH)	|| defined(CONFIG_ENV_IS_NOWHERE)	/* Environment not changable */
		puts ("Using default environment\n\n");
#else
		puts ("*** Warning - bad CRC, using default environment\n\n");
		show_boot_progress (-60);
#endif
		set_default_env();
	}
```
- 由于第一次开机 `gd->env_valid`并未设置，则会执行上面的代码 。`show_boot_progress()` 的作用是便于调试，告诉当前进行到什么地方了
- `set_default_env()`是对获取到的 env 内存区域进行初步配置：清0、复制系统预先定制的默认环境变量到新申请的内存地址处、CRC校验更新  

若非第一次开机，则会执行 `env_relocate_spec ();`，真正的从SD卡到DDR中重定位 ENV 的代码是在 env_relocate_spec 内部的 movi_read_env 完成的。

`gd->bd->bi_ip_addr = getenv_IPaddr ("ipaddr");`  
开发板的 IP 地址是在 gd->bd 中维护的，来源于环境变量 ipaddr 。 getenv 函数用来获取字符串格式的 IP 地址，然后用 string_to_ip 将字符串格式的IP地址转成字符串格式的点分十进制格式。 IP 地址由4个0-255之间的数字组成，因此一个IP地址在程序中最简单的存储方法就是一个 unsigend int 。但是人类容易看懂的并不是这种类型，而是点分十进制类型（192.168.1.2）。这两种类型可以互相转换

`stdio_init ();	/* get the devices list going. */`  
放在这里初始化的设备都是驱动设备，这个函数本来就是从驱动框架中衍生出来的。 uboot 中很多设备的驱动是直接移植linux内核的（譬如网卡、SD 卡）， linux 内核中的驱动都有相应的设备初始化函数。linux内核在启动过程中就有一个 devices_init (名字不一定完全对，但是差不多)，作用就是集中执行各种硬件驱动的 init 函数。 uboot 的这个函数其实就是从 linux 内核中移植过来的，它的作用也是去执行所有的从 linux 内核中继承来的那些硬件驱动的初始化函数。  
实际这个函数中很多宏是未定义的

`jumptable_init ();`：实际未发现有什么用  

`console_init_r ();	/* fully init console as a device */`:是对软件架构方面的初始化，即对相关的结构体进行初始化。因为在前面已经对串口做过初始化，并且已经能够使用 printf() 函数直接通过硬件输出打印，实际也是通过这样的方式实现的，这里做这样的初始化其实没什么意义，到头来还是调用底层进行输出打印  

`enable_interrupts ();`：实际并未使用中断，其实在 uboot 中并不常用中断，追踪代码的话，这个函数内容定义是空的，返回为 0  

```C
if ((s = getenv ("loadaddr")) != NULL) {
	load_addr = simple_strtoul (s, NULL, 16);
}
```
从已初始化的环境变量中获取加载地址并更新到 load_addr 变量。`simple_strtoul`功能是将一个字符串转换成 unsigend long 型数据

```C
#ifdef BOARD_LATE_INIT
	board_late_init ();
#endif
```
`BOARD_LATE_INIT`在 include\configs\itop_4412_ubuntu.h 中定义了。`board_late_init()`定义在 board\samsung\smdkc210\smdkc210.c 中。  
在这个里面是对剩余的一些板级初始化。在 iTop4412 提供的 uboot 中，在这个函数中对 bootcmd 环境变量进行了修改，搞不懂为啥要写到这个地方，在 common\env_common.c 目录文件中的 default_environment[]中是有对这个变量的初始化，按道理应该是在 include\configs\itop_4412_ubuntu.h 中对 `CONFIG_BOOTCOMMAND`宏进行配置即可  

```C
#ifdef CONFIG_RECOVERY //mj for factory reset
	recovery_preboot();
#endif
```
`CONFIG_RECOVERY`在 include\configs\itop_4412_ubuntu.h 中定义了  
```C
/* add by cym 20141211 GPX1_1 */
int value = 0;

char run_cmd[50];

value = __REG(GPX1DAT);

if(0x2 == (value & 0x2))//not press
{
	printf("SYSTEM ENTER NORMAL BOOT MODE\n");
}
else	//press
{
	printf("SYSTEM ENTER Updating MODE\n");

	sprintf(run_cmd, "sdfuse flashall");
	run_command(run_cmd, 0);
}
/* end add */

return 0;
```
在`recovery_preboot()`函数中实际执行的应该是上面的这段代码，通过读取 GPIO 判断是否需要更新固件，我们可以将要升级的镜像放到 SD 卡的固定目录中，然后开机时在 uboot 启动的最后阶段检查升级标志（是一个按键，这个按键如果按下则表示 update mode，如果启动时未按下则表示 boot mode ）。如果进入 update mode 则 uboot 会自动从 SD 卡中读取镜像文件然后烧录到 iNand 中；如果进入 boot mode 则 uboot 不执行 update ，直接启动正常运行。这种机制能够帮助我们快速烧录系统，常用于量产时用 SD 卡进行系统烧录部署  








[内存分布分析]: 