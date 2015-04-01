#LAB2实验报告
####马晓彬


## 练习1 理解通过make生成执行文件的过程

### 理解makefile生成ucroe.img的过程
Makefile文件整体进行了宏、变量的声明以及命令的执行，整体分为三大步骤，使用make “V=” 命令编译，得到编译器执行指令，以下为三大步骤内容使用gcc将.c和.s文件编译为.o的二进制文件

1.	通过对function.mk文件中Makefile函数的调用，Makefile代码生成了如下图的一系列命令，最终生成了bootblock二进制文件,生成bootblock相关代码为：
	
	```
	# create bootblock
	bootfiles = $(call listf_cc,boot)
	$(foreach f,$(bootfiles),$(call cc_compile,$(f),$(CC),$(CFLAGS) -Os -nostdinc))
	bootblock = $(call totarget,bootblock)
	$(bootblock): $(call toobj,$(bootfiles)) | $(call totarget,sign)
		@echo + ld $@
		$(V)$(LD) $(LDFLAGS) -N -e start -Ttext 0x7C00 $^ -o $(call toobj,bootblock)
		@$(OBJDUMP) -S $(call objfile,bootblock) > $(call asmfile,bootblock)
		@$(OBJDUMP) -t $(call objfile,bootblock) | $(SED) '1,/SYMBOL TABLE/d; s/ .* / /; /^$$/d' > $(call symfile,bootblock)
		@$(OBJCOPY) -S -O binary $(call objfile,bootblock) $(call outfile,bootblock)
		@$(call totarget,sign) $(call outfile,bootblock) $(bootblock)
	$(call create_target,bootblock)
	```
	生成sign工具的makefile代码为
	
	```
	$(call add_files_host,tools/sign.c,sign,sign)
	$(call create_target_host,sign,sign)
	```
	其中包括了对很多相同文件的操作，以生成init.o为例，命令解释如下：
	
	```	-I<dir> 	添加头文件的搜索路径	-fno-builtin	不接受不是两个下划线开头的内建函数	-Wall	打开gcc所有警告	-ggdb	生成可供gdb使用的调试信息	-m32	生成适用于32位环境的代码	-gstabs	生成stabs格式的调试信息	-nostdinc	不在标准系统目录中搜索头文件，只在-I的路径中搜搜索	-fno-stack-protector	不生成用于检测缓冲区溢出的代码	-c	只编译不链接	-o	表示输出文件名
	```
2.	使用ld命令将.o的二进制文件链接生成大的二进制程序3.	最终将二进制程序使用dd命令按顺序拷贝到ucore.img文件，其它区域用零文件填充。对应的Makefile代码如下，基本是直接调用linux命令执行的。
	最终生成ucore.img的相关代码为
	
	```
	# create ucore.img
	UCOREIMG	:= $(call totarget,ucore.img)
	$(UCOREIMG): $(kernel) $(bootblock)
		$(V)dd if=/dev/zero of=$@ count=10000
		$(V)dd if=$(bootblock) of=$@ conv=notrunc
		$(V)dd if=$(kernel) of=$@ seek=1 conv=notrunc
	$(call create_target,ucore.img)
	```

### 一个被系统认为是符合规范的硬盘主引导扇区的特征是什么？
一个系统认为符合规范的磁盘主引导扇区的大小为512字节，并且以0x55AA结束。

## 练习2 使用qemu执行并调试lab1中的软件
### 练习过程
1.	在/tools/gdbinit文件中添加如下命令：
	
	```	set architecture i8086	target remote :1234	break *(0x7c00)	define hook-stop	x/3i $pc	end	```就可以通过stepi命令以单条汇编代码单步调试，可以观察BIOS启动过程而且可以看到当前的和下一条指令。
2.	使用make debug命令开始执行，会看到qemu和gdb的窗口，之所以在调试窗口中显示3条指令，是因为有些等待的代码会下图所示会在一条汇编指令循环上千次，可以直接再添加一个断点使用continue命令绕过
		```
	0x7ccc:	repnz 	insl (%dx), %es:(%edi)
	0x7cce:	pop		%edi	```
3.	通过在调用qemu 时增加-d in_asm -D q.log 参数，将运行的汇编指令保存在q.log中。发现调试中的信息，bootblock.asm，bootasm.s中代码相同，仅是格式有差别。

## 练习3 分析bootloader进入保护模式的过程
###设计实现过程
```
#在实模式中，计算机从0x7c00处开始执行命令，也即汇编代码的start处.globl start
start:
.code16                                   # 以16位模式编译cli                                            # 禁止中断                 cld                                                 # 初始化各个段寄存器(DS, ES, SS).xorw %ax, %ax                       #段寄存器为0    movw %ax, %ds                                #  数据段movw %ax, %es  额外段，和DI搭配movw %ax, %ss                                   
# -> 栈段寄存器# 初始化A20，将键盘控制器上所有地址线置高，32位地址线都可用
.seta20.1:    
inb $0x64, %al                             #等待8042控制器空闲.    testb $0x2, %al    
jnz seta20.1    
movb $0xd1, %al             # 参数0xd1表示写数据至8042的端口outb %al, $0x64             # 写数据0x64
seta20.2:    
inb $0x64, %al                 #等待8042控制器空闲.testb $0x2, %al    
jnz 
seta20.2    
movb $0xdf, %al                      # 参数0xdf表示    
outb %al, $0x60                      # 0xdf = 11011111# 从实模式转换为保护模式, 使用GDT和段寄存器实现虚地址至实地址的转换，同时在转换中当前的地址仍然有效    
lgdt gdtdesc                         #装载gdt表头至寄存器#将cr0寄存器PE位置1开启保护模式movl %cr0, %eax          orl $CR0_PE_ON, %eax    
movl %eax, %cr0    
#通过长跳转更新cs的基地址    
ljmp $PROT_MODE_CSEG, $protcseg
.code32         # 表明以下代码以32位编译protcseg:    # 初始化段寄存器movw $PROT_MODE_DSEG, %ax                       movw %ax, %ds movw %ax, %esmovw %ax, %fs                                  movw %ax, %gs                                  movw %ax, %ss                                       # 设置栈顶指针以及返回地址并跳转至启动处movl $0x0, %ebp    
movl $start, %esp    
call bootmain
```
## 练习4 分析bootloader加载ELF格式的OS的过程

bootloader使用两层封装的读取硬盘的代码将ELF格式的OS加载到内存中。
*	readsect函数的作用是从设备的第secno扇区读取数据到内存的dst处

```static void	readsect(void *dst, uint32_t secno) {	waitdisk();	//outb操作端口，是与外设通信的汇编代码			    
	outb(0x1F2, 1);    //设置读取扇区的数目为1，即512KB的bootblock	outb(0x1F3, secno & 0xFF);     outb(0x1F4, (secno >> 8) & 0xFF);    outb(0x1F5, (secno >> 16) & 0xFF); //以上三行发送1到24位    outb(0x1F6, ((secno >> 24) & 0xF) | 0xE0); //发25到29位并将30-32位置为1	outb(0x1F7, 0x20);                // 0x20命令是读取扇区	waitdisk();	insl(0x1F0, dst, SECTSIZE / 4);      // 读取值内存dst处，单位转换                 	}
```
*	readseg对readsect函数进行了封装，使其能够从内核的offset处读取count个字节的信息至内存（虚地址）va处```static void readseg(uintptr_t va, uint32_t count, uint32_t offset)```。*	Bootmain函数通过调用readseg函数进行了操作系统到内存的读取，并通过调用其中的入口函数将控制权交给操作系统。
```void bootmain(void) {    // 读取硬盘中的第一页（扇区）   readseg((uintptr_t)ELFHDR, SECTSIZE * 8, 0);    // 通过其中的结束段确定其合法性   if (ELFHDR->e_magic != ELF_MAGIC) 
	{        
		goto bad;    
	}
	struct proghdr *ph, *eph;   	// 读取每个程序段到系统内存中 (ignores ph flags)   	// ph为偏移量   	ph = (struct proghdr *)((uintptr_t)ELFHDR + ELFHDR->e_phoff);    
	eph = ph + ELFHDR->e_phnum;    
	for (; ph < eph; ph ++) {        
		readseg(ph->p_va & 0xFFFFFF, ph->p_memsz, ph->p_offset);    
	}//读取至指定的内存处    // 调用系统代码中的入口函数转移设备控制权，且不返回        ((void (*)(void))(ELFHDR->e_entry & 0xFFFFFF))();    bad://遇到错误的处理   	outw(0x8A00, 0x8A00);    
	outw(0x8A00, 0x8E00);    
	/* do nothing */    
	while (1);
}```		
## 练习5 实现函数调用堆栈跟踪函数
实现之后的输出如下：

```
Kernel executable memory footprint: 64KBebp = 0x7b08 eip = 0x1009a6 arg 0:0x7b1a arg 1:0x7b1b arg 2:0x7b1c arg 3:0x7b1d     kern/debug/kdebug.c:294: print_stackframe+21ebp = 0x7b18 eip = 0x100092 arg 0:0x7b3a arg 1:0x7b3b arg 2:0x7b3c arg 3:0x7b3d     kern/init/init.c:48: grade_backtrace2+33ebp = 0x7b38 eip = 0x1000bb arg 0:0x7b5a arg 1:0x7b5b arg 2:0x7b5c arg 3:0x7b5d     kern/init/init.c:53: grade_backtrace1+38ebp = 0x7b58 eip = 0x1000d9 arg 0:0x7b7a arg 1:0x7b7b arg 2:0x7b7c arg 3:0x7b7d     kern/init/init.c:58: grade_backtrace0+23ebp = 0x7b78 eip = 0x1000fe arg 0:0x7b9a arg 1:0x7b9b arg 2:0x7b9c arg 3:0x7b9d     kern/init/init.c:63: grade_backtrace+34ebp = 0x7b98 eip = 0x100055 arg 0:0x7bca arg 1:0x7bcb arg 2:0x7bcc arg 3:0x7bcd     kern/init/init.c:28: kern_init+84ebp = 0x7bc8 eip = 0x7d68 arg 0:0x7bfa arg 1:0x7bfb arg 2:0x7bfc arg 3:0x7bfd     <unknow>: -- 0x00007d67 --ebp = 0x7bf8 eip = 0x7c4f arg 0:0x2 arg 1:0x3 arg 2:0x4 arg 3:0x5     <unknow>: -- 0x00007c4e --ebp = 0x0 eip = 0xf000ff53 arg 0:0xf000ff55 arg 1:0xf000ff56 arg 2:0xf000ff57 arg 3:0xf000ff58     <unknow>: -- 0xf000ff52 --ebp = 0xf000ff53 eip = 0x0 arg 0:0x2 arg 1:0x3 arg 2:0x4 arg 3:0x5     <unknow>: -- 0xffffffff --ebp = 0x0 eip = 0xf000ff53 arg 0:0xf000ff55 arg 1:0xf000ff56 arg 2:0xf000ff57 arg 3:0xf000ff58     <unknow>: -- 0xf000ff52 --ebp = 0xf000ff53 eip = 0x0 arg 0:0x2 arg 1:0x3 arg 2:0x4 arg 3:0x5     <unknow>: -- 0xffffffff --ebp = 0x0 eip = 0xf000ff53 arg 0:0xf000ff55 arg 1:0xf000ff56 arg 2:0xf000ff57 arg 3:0xf000ff58     <unknow>: -- 0xf000ff52 --ebp = 0xf000ff53 eip = 0x0 arg 0:0x2 arg 1:0x3 arg 2:0x4 arg 3:0x5     <unknow>: -- 0xffffffff --ebp = 0x0 eip = 0xf000ff53 arg 0:0xf000ff55 arg 1:0xf000ff56 arg 2:0xf000ff57 arg 3:0xf000ff58     <unknow>: -- 0xf000ff52 --ebp = 0xf000ff53 eip = 0x0 arg 0:0x2 arg 1:0x3 arg 2:0x4 arg 3:0x5     <unknow>: -- 0xffffffff --ebp = 0x0 eip = 0xf000ff53 arg 0:0xf000ff55 arg 1:0xf000ff56 arg 2:0xf000ff57 arg 3:0xf000ff58     <unknow>: -- 0xf000ff52 --ebp = 0xf000ff53 eip = 0x0 arg 0:0x2 arg 1:0x3 arg 2:0x4 arg 3:0x5     <unknow>: -- 0xffffffff --ebp = 0x0 eip = 0xf000ff53 arg 0:0xf000ff55 arg 1:0xf000ff56 arg 2:0xf000ff57 arg 3:0xf000ff58     <unknow>: -- 0xf000ff52 --ebp = 0xf000ff53 eip = 0x0 arg 0:0x2 arg 1:0x3 arg 2:0x4 arg 3:0x5     <unknow>: -- 0xffffffff --```发现答案中停止条件多了ebp！=0，加上后就避免了我ebp=0之后所有的错误输出，make grade的结果也正确了。
## 练习6 完善中断初始化和处理
代码如下：```voididt_init(void) {	extern uintptr_t __vectors[];	int i;	for(i = 0; i < 256; i++)	{		if(i < IRQ_OFFSET)//32个陷阱门处理异常		{			SETGATE(idt[i], 1, GD_KTEXT, __vectors[i],0);		}		else		{			SETGATE(idt[i], 0, GD_KTEXT, __vectors[i], 0);		}	}	SETGATE(idt[T_SWITCH_TOK],1,KERNEL_CS,__vectors[T_SWITCH_TOK],3);	lidt(&idt_pd);
```
答案没有对中断门的类型进行区分，对所有的中断都视为陷阱门，但是仍然能智能工程运行。
## 练习7 实验中重要的知识点
1.	理解了makefile的复杂应用2.	系统的初始化（bootloader）的工作流程3.	操作系统和硬盘外设交互的流程（ELF）和控制权转向操作系统的流程4.	C函数调用栈的结构