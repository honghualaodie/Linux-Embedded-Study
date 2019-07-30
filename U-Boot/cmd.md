# 命令体系分析

---

### 命令体系构成  
在源码目录的 common 文件夹下有关于命令的所有文件。当中 command.c 是旧的命令，之前命令均在此文件中包含。后面加入的命令为单独建立的文件，如 cmd_help.c 为帮助命令的代码文件。  


(1) 填充1个结构体实例构成一个命令  
(2) 给命令结构体实例附加特定段属性（用户自定义段），链接时将带有该段属性的内容链接在一起排列（挨着的，不会夹杂其他东西，也不会丢掉一个带有这种段属性的，但是顺序是乱序的）。  
(3) uboot重定位时将该段整体加载到DDR中。加载到 DDR 中的 uboot 镜像中带有特定段属性的这一段其实就是命令结构体的集合，有点像一个命令结构体数组。  
(4) 段起始地址和结束地址（链接地址、定义在 u-boot.lds 中）决定了这些命令集的开始和结束地址。  


命令结构体
```C
struct cmd_tbl_s {
	char		*name;		/* Command Name			*/
	int		maxargs;	/* maximum number of arguments	*/
	int		repeatable;	/* autorepeat allowed?		*/
					/* Implementation function	*/
	int		(*cmd)(struct cmd_tbl_s *, int, int, char *[]);
	char		*usage;		/* Usage message	(short)	*/
#ifdef	CONFIG_SYS_LONGHELP
	char		*help;		/* Help  message	(long)	*/
#endif
#ifdef CONFIG_AUTO_COMPLETE
	/* do auto completion on the arguments */
	int		(*complete)(int argc, char *argv[], char last_char, int maxv, char *cmdv[]);
#endif
};
typedef struct cmd_tbl_s cmd_tbl_t;

```
解析：  
(1) name：命令名称，字符串格式。  
(2) maxargs：命令最多可以接收多少个参数  
(3) repeatable：指示这个命令是否可重复执行。重复执行是uboot命令行的一种工作机制，就是直接按回车则执行上一条执行的命令。  
(4) cmd：函数指针，命令对应的函数的函数指针，将来执行这个命令的函数时使用这个函数指针来调用。  
(5) usage：命令的短帮助信息。对命令的简单描述。  
(6) help：命令的长帮助信息。细节的帮助信息。  
(7) complete：函数指针，指向这个命令的自动补全的函数。  

总结：  
`uboot` 的命令体系在工作时，一个命令对应一个  `cmd_tbl_t` 结构体的一个实例，然后  `uboot` 支持多少个命令，就需要多少个结构体实例。`uboot` 的命令体系把这些结构体实例管理起来，当用户输入了一个命令时，`uboot` 会去这些结构体实例中查找（查找方法和存储管理的方法有关）。如果找到则执行命令，如果未找到则提示命令未知。  

以 help 命令为例： 

在 cmd_help.c 文件中，包含有下面的代码：  
```C
int do_help(cmd_tbl_t * cmdtp, int flag, int argc, char *argv[])
{
	return _do_help(&__u_boot_cmd_start,
			((cmd_tbl_t *)(&__u_boot_cmd_end) - (cmd_tbl_t *)(&__u_boot_cmd_start))/sizeof(cmd_tbl_t *),
			cmdtp, flag, argc, argv);
}

U_BOOT_CMD(
	help,	CONFIG_SYS_MAXARGS,	1,	do_help,
	"print command description/usage",
	"\n"
	"	- print brief description of all commands\n"
	"help command ...\n"
	"	- print detailed usage of 'command'"
);
```
解析：  
1， `do_help()` 为命令执行函数  
2， `U_BOOT_CMD()` 为命令注册的部分。  
在 `command.h` 中有对此部分的宏定义：  
```C
#define Struct_Section  __attribute__ ((unused,section (".u_boot_cmd")))

#define U_BOOT_CMD(name,maxargs,rep,cmd,usage,help) \
cmd_tbl_t __u_boot_cmd_##name Struct_Section = {#name, maxargs, rep, cmd, usage, help}
```
根据上面的宏定义，我们可以将 help 定义部分填充预编译为如下：  
`cmd_tbl_t __u_boot_cmd_help __attribute__ ((unused,section (".u_boot_cmd"))) = {help, CONFIG_SYS_MAXARGS, 1, do_help, "print command description/usage\n", "	- print brief description of all commands\nhelp command ...\n	- print detailed usage of 'command'"}`  
  
注：  
1, `##name` 代表需要使用前面的  `name` 代替  
2, `Struct_Section` 意思是声明名称为 `.u_boot_cmd` 的段属性，即上面填充预编译的代码意义为：在 `cmd_help.c` 中，使用  `U_BOOT_CMD()` 填充的命令的相关信息成功建立名为`__u_boot_cmd_help` 的结构体，并且使用 `.u_boot_cmd` 标记段属性（实际所有的命令均标记名为`.u_boot_cmd`的段属性）。在 `u-boot.lds` 链接文件中定义有如下代码：  
```
	__u_boot_cmd_start = .;
	.u_boot_cmd : { *(.u_boot_cmd) }
	__u_boot_cmd_end = .;
```
当程序编译链接时，标记有 `.u_boot_cmd` 段属性的结构体会自动编译到一个区域中。系统运行时，也会运行到同一片连续的内存区域，这点类似直接申请一个结构体变量数组或 `malloc`，但这样的方式更灵活，如此就可以灵活添加删除并使用了。  

### 自建(添加)命令

#### 方式一  
在 `commond.c` 或其它命令文件中使用  `U_BOOT_CMD()` 新建命令结构体，并且新建命令执行函数：如下：  
```C
int do_test(cmd_tbl_t * cmdtp, int flag, int argc, char *argv[])
{
	return _do_help(&__u_boot_cmd_start,
			((cmd_tbl_t *)(&__u_boot_cmd_end) - (cmd_tbl_t *)(&__u_boot_cmd_start))/sizeof(cmd_tbl_t *),
			cmdtp, flag, argc, argv);
}

U_BOOT_CMD(
	test,	CONFIG_SYS_MAXARGS,	1,	do_test,
	"print command description/usage",
	"\n"
	"long help message test command ...\n""
);
```

#### 方式二  
1， 在 common 目录下新建命令的 `.c` 文件，如  `cmd_test.c`  
2， 在 `.c` 文件中包含头文件 `#include <common.h> #include <command.h>`  
3， 加入 方式一中建立的 `U_BOOT_CMD` 和 `do_test`  
4， 在 common 目录下的 Makefile 文件中添加：`COBJS-y += cmd_test.o`  

### 指令的执行  

当 uboot 执行到 `main_loop()`后，在这个函数中开始执行  `run_command()`函数对指令进行检索遍历，解析并执行相关的动作函数主干函数如下：  

|main_loop()  
|--run_command()    
|----parse_line()  
|----find_cmd()  

`find_cmd()` 函数
```C
cmd_tbl_t *find_cmd (const char *cmd)
{
	int len = &__u_boot_cmd_end - &__u_boot_cmd_start;
	return find_cmd_tbl(cmd, &__u_boot_cmd_start, len);
}
```
解析：  
1，`__u_boot_cmd_end` 和 `__u_boot_cmd_start` 在 `u-boot.lds` 文件中有定义
2， 通过上面的程序可以知道，这里遍历指定地址数据达到遍历所有指令的目的。（每个命令注册时标记的段属性会在链接时链接到指定的运行内存中）  
注意：遍历时所有的指令虽然都标记段属性并在链接时链接到一起，但并不是按照顺序排列的