h## lab1
### exercise1
通过学习和查找：https://www.cnblogs.com/mfryf/p/3305778.html 了解makefile
Q1:     操作系统镜像文件ucore.img是如何一步一步生成的？(需要比较详细地解释Makefile中每一条相关命令和命令参数的含义，以及说明命令导致的结果)
Answer:
makefile文件中第一个目标是kernel
```
119 # -------------------------------------------------------------------
120 # kernel
121
122 KINCLUDE    += kern/debug/ \
123                kern/driver/ \
124                kern/trap/ \
125                kern/mm/
126
127 KSRCDIR     += kern/init \
128                kern/libs \
129                kern/debug \
130                kern/driver \
131                kern/trap \
132                kern/mm
133
134 KCFLAGS     += $(addprefix -I,$(KINCLUDE))
135
136 $(call add_files_cc,$(call listf_cc,$(KSRCDIR)),kernel,$(KCFLAGS))
137
138 KOBJS   = $(call read_packet,kernel libs)
139
140 # create kernel target
141 kernel = $(call totarget,kernel)
142
143 $(kernel): tools/kernel.ld
144
145 $(kernel): $(KOBJS)
146     @echo + ld $@
147     $(V)$(LD) $(LDFLAGS) -T tools/kernel.ld -o $@ $(KOBJS)
148     @$(OBJDUMP) -S $@ > $(call asmfile,kernel)
149     @$(OBJDUMP) -t $@ | $(SED) '1,/SYMBOL TABLE/d; s/ .* / /; /^$$/d' > $(call symfile,kernel)
150
151 $(call create_target,kernel)
152
153 # -------------------------------------------------------------------
```
1.  := 符号表示传递性赋值，同时，赋的值仅能引用之前定义过的变量
2.  call函数的使用来创建新的变量
    1.  totarget在makefile里面并没有定义，是定义在tools/function.mk之中，然后用include tools/function.mk引入的
        1.  totarget = $(addprefix $(BINDIR)$(SLASH),$(1)) 中，使用了addprefix来添加前缀
            1.  $(BINDIR)$(SLASH)这两个变量使用 := 赋值，能够传递到function.mk中调用
3.  最终得到kernel=bin/kernel
4.  首先，依赖文件是kernel.ld这个链接脚本
5.  KOBJS   = $(call read_packet,kernel libs)
    1.  read_packet kernel
        1.  packetname kernel== __objs_kernel
            1.  $($(__objs_kernel))
6.  根据输出可以定位到在此发生的链接操作
    1.  使用i386-elf-ld 命令来链接
        1.  此条链接命令涉及的依赖obj文件需要编译
            1.  依次根据kern/*.c去编译需要的*.o文件
     生成 -o bin/kernel
7.  objdump使用来查看目标文件构成的工具
    使用objdump -S 生成反汇编出来的指令对照格式，并存在obj/kernel.asm
8.  使用objdump -t 来查看文件符号表入口的对应位置，用sed处理后存在obj/kernel.sym
9.  当生成kernel后，发现kernel是生成ucore.img的prerequisites，因此开始更新（生成）ucore.img
    发现此时依赖于bootblock
    
```
# -------------------------------------------------------------------

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

# -------------------------------------------------------------------

# create 'sign' tools
$(call add_files_host,tools/sign.c,sign,sign)
$(call create_target_host,sign,sign)

# -------------------------------------------------------------------
```

10. 生成bootblock依赖于sign，因此先编译生成sign，然后编译生成bootblock

            

```
    178 # create ucore.img
    179 UCOREIMG        := $(call totarget,ucore.img)
    180
    181 $(UCOREIMG): $(kernel) $(bootblock)
    182         $(V)dd if=/dev/zero of=$@ count=10000
    183         $(V)dd if=$(bootblock) of=$@ conv=notrunc
    184         $(V)dd if=$(kernel) of=$@ seek=1 conv=notrunc
    185
    186 $(call create_target,ucore.img)
    187
    188 # >>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>
    189
    190 $(call finish_all)
```
11. 在kernel和bootblok都准备好了之后，开始生成ucore.img
12. 用dd命令将/dev/zero复制10000个block到bin/ucore.img
13. 在依次将bootblock和kernel无损写入


Q2. 一个被系统认为是符合规范的硬盘主引导扇区的特征是什么？
buf[510] = 0x55;
buf[511] = 0xAA;    在512B的最后两个字节是一个标志性结尾，以此标志。

### exercise2
使用gdb调试程序
gdb基本命令：
si stepi往下走一条机器指令
i r查看寄存器的值
layout reg/split 显示寄存器信息和汇编代码
x /2s 0x10 查看0x10内存的值
x /2i $pc  查看eip的相邻两条命令
b \*0x7c00  在0x7c00处打断点，这个内存位置，也是BIOS加电后会跳到的位置
c 

### exercise3

1.  GDT 全局描述符表。当进入保护模式后，对内存的访问，涉及到段的位置，每个段都必须要有记录和描述，就存储在这张表里面。
2.  GDTR 全局描述符表寄存器，将GDT的存储地址放在此寄存器之中，帮助cpu进行处理访问地址信息

```
IN:
     26 0x000fcf45:  fa                       cli
     27 0x000fcf46:  fc                       cld
     28 0x000fcf47:  66 89 c1                 movl     %eax, %ecx
     29 0x000fcf4a:  66 b8 8f 00 00 00        movl     $0x8f, %eax
```
将中断标志置0和段寄存器初始化
```
     30 0x000fcf50:  e6 70                    outb     %al, $0x70
     31 0x000fcf52:  e4 71                    inb      $0x71, %al
     32 0x000fcf54:  e4 92                    inb      $-0x6e, %al
     33 0x000fcf56:  0c 02                    orb      $2, %al
     34 0x000fcf58:  e6 92                    outb     %al, $0x92
     
```
开启a20，使系统能够使用32位的地址
```
     35 0x000fcf5a:  66 89 c8                 movl     %ecx, %eax
     36 0x000fcf5d:  2e 0f 01 1e b8 5e        lidtw    %cs:0x5eb8
     37 0x000fcf63:  2e 0f 01 16 78 5e        lgdtw    %cs:0x5e78
```
加载gdt
```
     38 0x000fcf69:  0f 20 c1                 movl     %cr0, %ecx
     39 0x000fcf6c:  66 81 e1 ff ff ff 1f     andl     $0x1fffffff, %ecx
     40 0x000fcf73:  66 83 c9 01              orl      $1, %ecx
     41 0x000fcf77:  0f 22 c1                 movl     %ecx, %cr0
     
```
将cr0置1，进入保护模式.

机器加电后，BIOS从0x7c00读取第一个sector内容到内存中，执行bootasm汇编代码，打开保护模式，跳转并进而执行c代码
### exercise4
Q1:bootloader如何读取硬盘扇区的？
Answer: 通过内连汇编方式，用out设置好读取信息后，从设备读取一个扇区到内存
Q2:bootloader是如何加载ELF格式的OS？
Answer：通过elf文件的头，对elf格式进行判断。然后在elf的指定位置里获取关于整个elf的信息，从而加载整个elf文件。
在加载后，跳到ucroe去，交给其控制权。

### exercise5
跟踪打印调用的函数的ebp&eip信息
程序在加载进内存后，每个函数有自己的内存的位置。
ss:ebp+4指向caller调用时的eip，ss:ebp+8等是（可能的）参数。

### exercise6
#### file: init.c
pic_init(); 初始化中断控制器，使之接受中断响应
idt_init(); 初始化中断描述符表，建表
idt的每一项包括：中断号，中断服务routine，基于段描述符和offset在gdt里面查询入口
lidt加载idt
idt的每一项中断信息包含8个字节，2、3是段选择子，0、1、6、7是offset，二者合并得到isr的入口地址

有中断请求时，根据中断号 __alltraps  -->  trap  -->  trap_dispatch --> 根据中断类型进行处理 --> 
