# 环境变量分析

---

### 环境变量的载入
1， 在`start_armboot()` 中执行调用了 `env_init` 函数， 由于未定义宏 `ENV_IS_EMBEDDED` ，则 `env_init` 函数执行的程序如下：  
```C
int env_init(void)
{
	gd->env_addr  = (ulong)&default_environment[0];
	gd->env_valid = 1;
	
	return (0);
}
```
即初始都会将默认环境变量包含到 `gd` 中来，并且会置位环境变量标识。

2， 同样在 `start_armboot()` 中，执行了 `env_relocate ()`，即环境变量重定位，忽略未定义的预编译部分后代码如下：  
```C
void env_relocate (void)
{
	/*
	 * We must allocate a buffer for the environment
	 */
	env_ptr = (env_t *)malloc (CONFIG_ENV_SIZE);
	DEBUGF ("%s[%d] malloced ENV at %p\n", __FUNCTION__,__LINE__,env_ptr);

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
}
```
由上面的代码可知，先为环境变量动态申请一段内存，由于在`env_init` 中已经将 `gd->env_valid` 置位，则直接执行 `env_relocate_spec ()`，代码如下：  

```C
void env_relocate_spec(void)
{
#if defined(CONFIG_SMDKC210)|| defined(CONFIG_SMDKV310)
	if (INF_REG3_REG == 1)
		env_relocate_spec_onenand();
	else if (INF_REG3_REG == 2)
		env_relocate_spec_nand();
	else if (INF_REG3_REG == 3)
		env_relocate_spec_movinand();
	else
		use_default();
#endif
}
```
解析：  
(1)，`INF_REG3_REG`，查询手册可知此寄存器为用户自定义寄存器。其实本寄存器在 `Start.S` 中读取判断 OMR 引脚启动方式确定为 `BOOT_MMCSD` ，即 3 ，并将此值写入 `INF_REG3_REG` 。  
识别出启动方式部分代码如下：    
```ASM
/* Read booting information */
ldr	r0, =POWER_BASE
ldr	r1, [r0,#OMR_OFFSET]
bic	r2, r1, #0xffffffc1

...

/* SD/MMC BOOT */
cmp     r2, #0x4
moveq   r3, #BOOT_MMCSD	

...

ldr	r0, =INF_REG_BASE
str	r3, [r0, #INF_REG3_OFFSET]  
```
(2)，由 (1) 中分析则判定此函数直接执行 `env_relocate_spec_movinand()` 。去除宏定义后代码如下：  

```C
void env_relocate_spec_movinand(void)
{
	uint *magic = (uint*)(PHYS_SDRAM_1);

	if ((0x24564236 != magic[0]) || (0x20764316 != magic[1])) {

		mmc_init(find_mmc_device(0));
		movi_read_env(virt_to_phys((ulong)env_ptr));
	}
	
	if (crc32(0, env_ptr->data, ENV_SIZE) != env_ptr->crc)
		return use_default();
}
```
解析：  
①，第一个if语句判断 `PHYS_SDRAM_1` 内存地址处是否有魔数 0x24564236 或者 PHYS_SDRAM_1 + 4 地址处是否有魔数 0x20764316 ，我们之前并没有填充过这两个区域，所以执行`movi_read_env` 函数。若没有在 inand 中保存过环境变量，则 crc 校验是不通过的，则会直接执行 `use_default()` 。显然是继续使用默认的环境变量，且对前面申请的 `env_ptr`指针赋值。  
```C
static void use_default()
{
	puts("*** Warning - using default environment\n\n");

	if (default_environment_size > CONFIG_ENV_SIZE) {
		puts("*** Error - default environment is too large\n\n");
		return;
	}

	memset (env_ptr, 0, sizeof(env_t));
	memcpy (env_ptr->data,
			default_environment,
			default_environment_size);
	env_ptr->crc = crc32(0, env_ptr->data, ENV_SIZE);
	gd->env_valid = 1;

}
```
②，在执行 `movi_read_env` 读取环境变量时，会通过 `raw_area_control` 进行操作。`raw_area_control` 为系统分区表，在 `start_armboot()` 中调用 `mmc_initialize()` 实现的初始化，相关分区表的内容参看专门的总结。  
疑问：在分区表中`raw_area_control.image[2]` 是标识为 `bootloder` 分区，env 分区则为 `raw_area_control.image[3]`，为何此处传入的是`raw_area_control.image[2]`？？？待验证。。。
```C
void movi_read_env(ulong addr)
{
	movi_read(raw_area_control.image[2].start_blk,
		raw_area_control.image[2].used_blk, addr);
}
```

### 环境变量相关命令  

相关命令在 `common/cmd_nvedit.c` 中。命令体系相关的分析已在 `cmd.md` 文件中，并且命令操作相关的可以直接参看 `cmd_nvedit.c`，下面只是记录相关的注意事项：  
1，printenv  
2，setenv  
3，saveenv  
4，getenv  
实现方式就是去遍历`default_environment` 数组，挨个拿出所有的环境变量比对 `name`，找到相等的直接返回这个环境变量的首地址即可。`getenv` 函数是直接返回这个找到的环境变量在 `DDR` 中环境变量处的地址，`getenv` 中返回的地址只能读不能随便乱写  
5，getenv_r  
可重入版本，`getenv_r` 函数的做法是找到了 `DDR` 中环境变量地址后，将这个环境变量复制一份到提供的 `buf` 中，如此操作是可以随便改写加工的   
