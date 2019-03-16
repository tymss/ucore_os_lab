# OS2019 lab1实验报告
## 练习1：理解通过make生成执行文件的过程
### 1.列出本实验各练习中对应的OS原理的知识点，并说明本实验中的实现部分如何对应和体现了原理中的基本概念和关键知识点。

* 练习一对应的知识点是编译运行ucore OS的过程。
* 练习二对应的知识点是qemu的使用、gdb调试工具的使用以及反汇编。
* 练习三对应的知识点是实模式和保护模式间的切换。
* 练习四对应的知识点是bootloader读取磁盘扇区加载操作系统。
* 练习五对应的知识点是函数的堆栈。
* 练习六对应的知识点是系统中断处理。

### 2.操作系统镜像文件ucore.img是如何一步一步生成的？(需要比较详细地解释Makefile中每一条相关命令和命令参数的含义，以及说明命令导致的结果)

1. makefile中，产生ucore.img的代码为

	```makefile
	UCOREIMG	:= $(call totarget,ucore.img)

	$(UCOREIMG): $(kernel) $(bootblock)
		$(V)dd if=/dev/zero of=$@ count=10000
		$(V)dd if=$(bootblock) of=$@ conv=notrunc
		$(V)dd if=$(kernel) of=$@ seek=1 conv=notrunc

	$(call create_target,ucore.img)
	```

	可以看到，要生成ucore.img，先分配了一块大小为10000的空间，然后将bootblock和kernel写入。因此，需要先生成bootblock和kernel。

2. kernel的生成

	makefile中生成kernel的代码为
	```
	kernel = $(call totarget,kernel)

	$(kernel): tools/kernel.ld

	$(kernel): $(KOBJS)
		@echo + ld $@
		$(V)$(LD) $(LDFLAGS) -T tools/kernel.ld -o $@ $(KOBJS)
		@$(OBJDUMP) -S $@ > $(call asmfile,kernel)
		@$(OBJDUMP) -t $@ | $(SED) '1,/SYMBOL TABLE/d; s/ .* / /; /^$$/d' > $(call symfile,kernel)

	$(call create_target,kernel)
	```
	通过make V=命令可以看到，首先通过gcc将kern目录下的.c和.S文件全部编译生成.o文件。其中，gcc使用的参数为
	```
	-fno-builtin 不使用C语言的内建函数
	-fno-PIC 不使用PIC
	-Wall 显示所有警告
	-ggdb 生成gdb所需要的调试信息
	-m32 生成32位机器码
	-gstabs 以stabs格式生成调试信息 
	-nostdinc 不在标准系统目录中搜索头文件
	-fno-stack-protector 启用堆栈保护
	```
	接着使用一条ld命令将生成的.o文件连接为bin/kernel
	```
	ld -m    elf_i386 -nostdlib -T tools/kernel.ld -o bin/kernel  obj/kern/init/init.o obj/kern/libs/stdio.o obj/kern/libs/readline.o obj/kern/debug/panic.o obj/kern/debug/kdebug.o obj/kern/debug/kmonitor.o obj/kern/driver/clock.o obj/kern/driver/console.o obj/kern/driver/picirq.o obj/kern/driver/intr.o obj/kern/trap/trap.o obj/kern/trap/vectors.o obj/kern/trap/trapentry.o obj/kern/mm/pmm.o  obj/libs/string.o obj/libs/printfmt.o
	```
	连接器为elf_i386，nostdlib选项使得不连接系统标准库文件。	
	
3. bootblock的生成

	

### 3.一个被系统认为是符合规范的硬盘主引导扇区的特征是什么？