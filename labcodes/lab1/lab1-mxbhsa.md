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
	
	```
	```

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
	
	```

	
	0x7ccc:	repnz 	insl (%dx), %es:(%edi)
	0x7cce:	pop		%edi


## 练习3 分析bootloader进入保护模式的过程
###设计实现过程
```
#在实模式中，计算机从0x7c00处开始执行命令，也即汇编代码的start处
start:
.code16                                   # 以16位模式编译
# -> 栈段寄存器# 初始化A20，将键盘控制器上所有地址线置高，32位地址线都可用
.seta20.1:    
inb $0x64, %al                             #等待8042控制器空闲.    testb $0x2, %al    
jnz seta20.1    
movb $0xd1, %al             # 参数0xd1表示写数据至8042的端口
seta20.2:    
inb $0x64, %al                 #等待8042控制器空闲.
jnz 
seta20.2    
movb $0xdf, %al                      # 参数0xdf表示    
outb %al, $0x60                      # 0xdf = 11011111# 从实模式转换为保护模式, 使用GDT和段寄存器实现虚地址至实地址的转换，同时在转换中当前的地址仍然有效    
lgdt gdtdesc                         #装载gdt表头至寄存器
movl %eax, %cr0    
#通过长跳转更新cs的基地址    
ljmp $PROT_MODE_CSEG, $protcseg
.code32         # 表明以下代码以32位编译
movl $start, %esp    
call bootmain
```
## 练习4 分析bootloader加载ELF格式的OS的过程

bootloader使用两层封装的读取硬盘的代码将ELF格式的OS加载到内存中。


```
	outb(0x1F2, 1);    //设置读取扇区的数目为1，即512KB的bootblock	outb(0x1F3, secno & 0xFF); 
```


	{        
		goto bad;    
	}
	struct proghdr *ph, *eph;
	eph = ph + ELFHDR->e_phnum;    
	for (; ph < eph; ph ++) {        
		readseg(ph->p_va & 0xFFFFFF, ph->p_memsz, ph->p_offset);    
	}//读取至指定的内存处
	outw(0x8A00, 0x8E00);    
	/* do nothing */    
	while (1);
}

实现之后的输出如下：

```
Kernel executable memory footprint: 64KB

代码如下：
```
答案没有对中断门的类型进行区分，对所有的中断都视为陷阱门，但是仍然能智能工程运行。
## 练习7 实验中重要的知识点
1.	理解了makefile的复杂应用