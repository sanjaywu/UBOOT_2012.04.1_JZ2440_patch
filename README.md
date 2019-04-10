## UBOOT_2012.04.1_JZ2440_patch

### 前言
>在前面，了解了[Bootloader的作用](https://blog.csdn.net/Sanjay_Wu/article/details/88980806)以及从0写一个Bootloader之后，最近花了差不多一个星期学习韦东山老师的JZ2440移植UBOOT 2012.04.1，看了视频和参考**博客园NQian**的博客进行学习。我制作的最新补丁：https://github.com/sanjaywu/UBOOT_2012.04.1_JZ2440_patch。

<br/>

### 一、JZ2440移植UBOOT 2012.04.1笔记
>以下笔记全部来自[博客园NQian](https://www.cnblogs.com/lifexy/)的笔记，将得非常详细，我也是参照他的博客进行学习的，收益很大，点击跳转可以直接到他的博客看。

###### [1.移植uboot-分析uboot启动流程(详解)](https://www.cnblogs.com/lifexy/p/8136378.html)
###### [2.移植uboot-添加2440单板,并实现NOR、NAND启动](http://www.cnblogs.com/lifexy/p/8185509.html)
###### [3.移植uboot-使板卡支持nor、nand](http://www.cnblogs.com/lifexy/p/8243972.html)
###### [4.移植uboot-使uboot支持DM9000网卡](http://www.cnblogs.com/lifexy/p/8301072.html)
###### [5.移植uboot-设置默认环境变量,裁剪,并分区](http://www.cnblogs.com/lifexy/p/8302690.html)
###### [6.移植uboot-支持yaffs烧写,打补丁](http://www.cnblogs.com/lifexy/p/8316591.html)
<br/>

### 二、和韦东山老师移植存在的区别
>我是参考博客园NQian的笔记进行学习，里面主要的不同是在**调用第2阶段的代码**、**relocate_code**的处理上有些不一样以及重新设置栈。

###### 1、韦东山老师的处理方法
（1）、在board.c中修改函数`board_init_f`，将该函数改为有返回值，修改函数为：`unsigned int board_init_f(ulong bootflag)`，用于返回`id`（存放 gd_t结构体的首地址）。

<br/>

（2）、将`board_init_f`函数后面的`relocate_code(addr_sp, id, addr);`注释去掉。


<br/>

（3）、在include\common.h里面把`void	board_init_f  (ulong) __attribute__ ((noreturn));`改为`unsigned int board_init_f(ulong bootflag)`。

<br/>

（4）、根据前面三个步骤，再来调用第二阶段代码`board_init_r`，代码如下：
```
/* Set stackpointer in internal RAM to call board_init_f */
call_board_init_f:
	ldr	r0,=0x00000000
	bl	board_init_f

	/* unsigned int的值存在r0里, 正好给board_init_r */
	ldr r1, _TEXT_BASE
	ldr sp, base_sp 			/* 重新设置栈 */

	/* 调用第2阶段的代码 */
	bl board_init_r
```

<br/>

（5）、base_sp定义如下：

在
```python
/* IRQ stack memory (calculated at run-time) + 8 bytes */
.globl IRQ_STACK_START_IN
IRQ_STACK_START_IN:
	.word	0x0badc0de
```
后面添加：
```python
.globl base_sp
base_sp:
	.long 0
```

<br/>

（6）把relocate_code全部去掉： 
```python
/*
 * void relocate_code (addr_sp, gd, addr_moni)
 *
 * This "function" does not return, instead it continues in RAM
 * after relocating the monitor code.
 *
 */
	.globl	relocate_code
relocate_code:
	mov	r4, r0	/* save addr_sp */
	mov	r5, r1	/* save addr of gd */
	mov	r6, r2	/* save addr of destination */

```

<br/>

###### 二、我和NQian的处理方法
>不删除addr_sp, id, addr还是通过relocate_code函数来处理、不返回`id`（存放 gd_t结构体的首地址），也不用定义全局变量`base_sp`，总之就是函数`board_init_f`处理不按照上面提到的韦东山老师的处理方法，`start.S`里面，在`call_board_init_f`前面的和韦东山老师一样，后面的处理如下。

（1）、在`start.S`里面，修改看如下代码：
```python
/* Set stackpointer in internal RAM to call board_init_f */
call_board_init_f:
	
	ldr	r0,=0x00000000
	bl	board_init_f


/* void relocate_code (addr_sp, gd, addr_moni)*/
.globl      relocate_code
relocate_code:

       mov r4, r0      	/* 保存 addr_sp到r4 */       
       mov sp, r4	   /* r4赋给sp来达到重新设置栈的目的 */
       mov r0, r1      	/* save addr of gd */
       mov r1, r2      	/* save addr of destination */
       bl  board_init_r /* 进入uboot第二阶段代码 */
```

<br/>

（2）、由后面的`relocate_code(addr_sp, id, addr);`函数可知：

<br/>

` mov r4, r0`：保存 addr_sp到r4 ；
`mov sp, r4`：4赋给sp来达到重新设置栈的目的 ；
` mov r0, r1`：r1是`id`（存放 gd_t结构体的首地址），将它赋给r0来作为`board_init_r`的第一个入口参数；
 `mov r1, r2`：r2是`addr`，将它赋给r1来作为`board_init_r`的第二个入口参数。

<br/>

（3）、`board_init_r`函数为：
```python
void board_init_r(gd_t *id, ulong dest_addr)
```

<br/>

（4）、通过以上就能够实现重新设置栈了，以及各参数的设置。

<br/>

### 三、我移植时遇到的问题和解决方法
<br/>

###### 1、报错`undefined reference to nand_info`
>这是在u-boot之修改代码支持NAND启动时编译出现的错误。

解决方法：

 - 在smdk2440.h里面屏蔽掉：//#define CONFIG_YAFFS2 
 - make distclean   
 - 配置 make smdk2440_config 
 - make

<br/>

###### 2、按照流程来，移植支持NAND启动后无反应
>在支持NAND启动的移植上，修改之后，下载代码，重新上电复位，发送NAND启动的时候没任何信息打印处理等反应。后来发现我是没删除掉`relocate_code`后面的代码。

解决方法是删除如下代码，因为前面我们已经重新设置栈了，也清除bss段了：
```python

	/* Set up the stack						    */
stack_setup:
	mov	sp, r4

	adr	r0, _start
	cmp	r0, r6
	beq	clear_bss		/* skip relocation */
	mov	r1, r6			/* r1 <- scratch for copy_loop */
	ldr	r3, _bss_start_ofs
	add	r2, r0, r3		/* r2 <- source end address	    */

copy_loop:
	ldmia	r0!, {r9-r10}		/* copy from source address [r0]    */
	stmia	r1!, {r9-r10}		/* copy to   target address [r1]    */
	cmp	r0, r2			/* until source end address [r2]    */
	blo	copy_loop

#ifndef CONFIG_SPL_BUILD
	/*
	 * fix .rel.dyn relocations
	 */
	ldr	r0, _TEXT_BASE		/* r0 <- Text base */
	sub	r9, r6, r0		/* r9 <- relocation offset */
	ldr	r10, _dynsym_start_ofs	/* r10 <- sym table ofs */
	add	r10, r10, r0		/* r10 <- sym table in FLASH */
	ldr	r2, _rel_dyn_start_ofs	/* r2 <- rel dyn start ofs */
	add	r2, r2, r0		/* r2 <- rel dyn start in FLASH */
	ldr	r3, _rel_dyn_end_ofs	/* r3 <- rel dyn end ofs */
	add	r3, r3, r0		/* r3 <- rel dyn end in FLASH */
fixloop:
	ldr	r0, [r2]		/* r0 <- location to fix up, IN FLASH! */
	add	r0, r0, r9		/* r0 <- location to fix up in RAM */
	ldr	r1, [r2, #4]
	and	r7, r1, #0xff
	cmp	r7, #23			/* relative fixup? */
	beq	fixrel
	cmp	r7, #2			/* absolute fixup? */
	beq	fixabs
	/* ignore unknown type of fixup */
	b	fixnext
fixabs:
	/* absolute fix: set location to (offset) symbol value */
	mov	r1, r1, LSR #4		/* r1 <- symbol index in .dynsym */
	add	r1, r10, r1		/* r1 <- address of symbol in table */
	ldr	r1, [r1, #4]		/* r1 <- symbol value */
	add	r1, r1, r9		/* r1 <- relocated sym addr */
	b	fixnext
fixrel:
	/* relative fix: increase location by offset */
	ldr	r1, [r0]
	add	r1, r1, r9
fixnext:
	str	r1, [r0]
	add	r2, r2, #8		/* each rel.dyn entry is 8 bytes */
	cmp	r2, r3
	blo	fixloop
#endif

clear_bss:
#ifndef CONFIG_SPL_BUILD
	ldr	r0, _bss_start_ofs
	ldr	r1, _bss_end_ofs
	mov	r4, r6			/* reloc addr */
	add	r0, r0, r4
	add	r1, r1, r4
	mov	r2, #0x00000000		/* clear			    */

clbss_l:str	r2, [r0]		/* clear loop...		    */
	add	r0, r0, #4
	cmp	r0, r1
	bne	clbss_l

	bl coloured_LED_init
	bl red_led_on
#endif
```

###### 3、移植完成，烧写u-boot之后，发现LCD只显示一半且模糊
解决方法：
在`\board\samsung\smdk2440`中的`board_init`函数将`gd->bd->bi_arch_number = MACH_TYPE_SMDK2410;`改为`gd->bd->bi_arch_number = MACH_TYPE_S3C2440;`


