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

	```
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

	makefile中生成bootblock的代码为
	```
	bootfiles = $(call listf_cc,boot)
	$(foreach f,$(bootfiles),$(call cc_compile,$(f),$(CC),$(CFLAGS) -Os -nostdinc))

	bootblock = $(call totarget,bootblock)

	$(bootblock): $(call toobj,$(bootfiles)) | $(call totarget,sign)
	@echo + ld $@
	$(V)$(LD) $(LDFLAGS) -N -e start -Ttext 0x7C00 $^ -o $(call toobj,bootblock)
	@$(OBJDUMP) -S $(call objfile,bootblock) > $(call asmfile,bootblock)
	@$(OBJCOPY) -S -O binary $(call objfile,bootblock) $(call outfile,bootblock)
	@$(call totarget,sign) $(call outfile,bootblock) $(bootblock)

	$(call create_target,bootblock)
	```
	通过make V=命令看到，首先使用gcc命令将bootasm.S和bootmain.c编译为对应的.o文件，这里的选项和编译kernel文件时一样。接着使用gcc编译sign
	```
	gcc -Itools/ -g -Wall -O2 -c tools/sign.c -o obj/sign/tools/sign.o
	gcc -g -Wall -O2 obj/sign/tools/sign.o -o bin/sign
	```
	之后使用ld命令将bootasm.o和bootmain.o连接起来生成bin/bootblock
	```
	ld -m    elf_i386 -nostdlib -N -e start -Ttext 0x7C00 obj/boot/bootasm.o obj/boot/bootmain.o -o obj/bootblock.o
	```
	指定start为程序入口，text段地址是0x7C00
	
4. 完成了以上操作后，创建大小为10000的0空间给bin/ucore.img，然后将bootblock和kernel依次拷贝进去，即完成了ucore.img的生成。
	
### 3.一个被系统认为是符合规范的硬盘主引导扇区的特征是什么？

大小为512字节，其中，最后两个字节为结束标志字0x55AA
	
## 练习2：使用qemu执行并调试lab1中的软件。

1. 从CPU加电后执行的第一条指令开始，单步跟踪BIOS的执行。
	
	首先修改lab1/tools/gdbinit，将内容修改为
	```
	set architecture i8086
	target remote :1234
	```
	然后在lab1目录下执行make debug，即打开qemu和gdb开始调试。
	此时gdb停在BIOS第一条指令处。
	执行x /i 0xffff0可以看到显示：
	```
	0xffff0:     ljmp   $0xf000,$0xe05b
	```
	这是BIOS的第一条指令。
	运行si单步运行，跳转到0xfe05b。
	执行x /10i 0xfe05b可以看到继续的BIOS的代码：
	```
	0xfe05b:     cmpl   $0x0,%cs:0x6c48
	0xfe062:     jne    0xfd2e1
	0xfe066:     xor    %dx,%dx
	0xfe068:     mov    %dx,%ss
	0xfe06a:     mov    $0x7000,%esp
	0xfe070:     mov    $0xf3691,%edx
	0xfe076:     jmp    0xfd165
	0xfe079:     push   %ebp
	0xfe07b:     push   %edi
	0xfe07d:     push   %esi
	```
	之后不断用si依次进行单步调试。
	
2. 在初始化位置0x7c00设置实地址断点,测试断点正常。

	在gdb中输入b *0x7c00设置其为断电，之后输入continue，看到gdb正常在执行到0x7c00的断点处停下来。
	
3. 从0x7c00开始跟踪代码运行,将单步跟踪反汇编得到的代码与bootasm.S和 bootblock.asm进行比较。

	在gdb执行到0x7c00停下来之后，输入x /10i $pc显示接下来10条指令：
	```
	0x7c00:      cli
	0x7c01:      cld
	0x7c02:      xor    %ax,%ax
	0x7c04:      mov    %ax,%ds
	0x7c06:      mov    %ax,%es
	0x7c08:      mov    %ax,%ss
	0x7c0a:      in     $0x64,%al
	0x7c0c:      test   $0x2,%al
	0x7c0e:      jne    0x7c0a
	0x7c10:      mov    $0xd1,%al
	```
	经过比对，与bootasm.S文件中16行开始的代码内容一致。
	接下来使用si命令让程序单步执行，然后查看指令，经比对，与bootasm.S中一致。
	完成了bootasm中的一系列操作后，跳转到bootmain函数中执行。
	
4. 自己找一个bootloader或内核中的代码位置，设置断点并进行测试。

	还原lab1/tools/gdbinit，将内容修改为：
	```
	file bin/kernel
	target remote :1234
	break kern_init
	continue
	```
	则执行make debug后运行到kern/init/init.c文件中的kern_init函数时停止，然后运行next进行调试，可以看到程序一次执行。
	
## 练习3：分析bootloader进入保护模式的过程。

## 练习4：分析bootloader加载ELF格式的OS的过程。

## 练习5：实现函数调用堆栈跟踪函数

## 练习6：完善中断初始化和处理	