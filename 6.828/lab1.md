# 6.828 lab1 笔记

## 0x00 basics

### CPU real mode

计算机刚通电的时候，CPU运行在实模式下，使用16bit的寄存器，但是地址总线确有20bit，其寻址方式是通过2个寄存器CS:IP来组成，通过左移CS寄存器4个bit加上IP寄存器作为偏移地址实现，即：`CS * 16 + IP`。

在实模式下，CPU的可用内存空间非常有限，只有可怜的1MB，也就是0x00000 - 0x10000，其地址大概为：

```
+------------------+  <- 0x00100000 (1MB)
|     BIOS ROM     |
+------------------+  <- 0x000F0000 (960KB)
|  16-bit devices, |
|  expansion ROMs  |
+------------------+  <- 0x000C0000 (768KB)
|   VGA Display    |
+------------------+  <- 0x000A0000 (640KB)
|                  |
|    Low Memory    |
|                  |
+------------------+  <- 0x00000000
```

### BIOS

x86系统在启动通电的时候，CS:IP寄存器会被固定的设置为 0xF000:0xFFF0，通过计算可用得到是`0xFFFF0`，这个地址位于BIOS中的一块区域，BIOS会固定映射到如上图表示的位置处，其中的代码是做一些启动前的检查，如硬件自检等；
BIOS完成自检后，会从各种磁盘设备中读取前面第一个扇区(512bytes)到内存中，内存范围为 `0x7C00 - 0x7DFF`，假如这512bytes中的后两位“魔数”是符合要求的，即`0xAA55`，那么CS:IP被设定为0x0000:0x7C00，即`0x7C00`。

### MBR

TODO：MBR layout


### x86 register

TODO:

### CPU protected mode

保护模式下CPU拥有32bit的寻址空间，即4GB，其寻址方式也由原来的CS:IP计算方式，转化为段选择子 + 查表的方式了。

TODO:

![](./pic/protect_mode.png)

其中每一个描述表项有8bytes，前4bytes构成一个base地址，共计寻址4GB。

lgdt的layout大致为：
![](./pic/lgdt_layout.png)

其中32bit指向内存中的一块物理地址，上面保存了表信息，后16bit则表示这个表的大小，即最大64K左右，按照每个项8 byte计算，可以保存8192个entry。

段选择子的layout则为：
![](./pic/selector_layout.png)

其中13bit作为index，表示lgdt指向地址的offset(非数组index)，TI是表示指向gdt或者ldt，这里先不管它，后面的RPL表示当前指令执行的特权登记，2bit表示4个等级，0为最高级内核态，3为用户态，xv6和linux这种操作系统都只用了0和3。


## 0x01 bootloader

### assembly 部分

xv6启动代码位于boot文件夹下面，由boot.S和main.c组成，其入口处就在boot.S中的start符号，这点可以从boot/Makefrag中看到：

```makefile
28 ▎   $(V)$(LD) $(LDFLAGS) -N -e start -Ttext 0x7C00 -o $@.out $^
```

start符号被设定为地址0x7C00，即MBR加载后，CPU跳过去的位置。

关于boot.S做了啥，首先是一些基础设定：

```assembly
cli # 关中断
cld
# 一些重要的寄存器，ds、es、ss清零
xorw    %ax,%ax
movw    %ax,%ds
movw    %ax,%es
movw    %ax,%ss
```

然后去掉繁琐的一些兼容代码，这里准备开启保护模式了：

```assmebly
8   .set PROT_MODE_CSEG, 0x8
9   .set PROT_MODE_DSEG, 0x10
10  .set CR0_PE_ON,      0x1
...
48 ▎ lgdt    gdtdesc # 加载 全局段表
49 ▎ movl    %cr0, %eax 
50 ▎ orl     $CR0_PE_ON, %eax
51 ▎ movl    %eax, %cr0
...
77  gdt:
78 ▎ SEG_NULL
79 ▎ SEG(STA_X|STA_R, 0x0, 0xffffffff)
80 ▎ SEG(STA_W, 0x0, 0xffffffff)
81
82 gdtdesc:
83 ▎ .word   0x17
84 ▎ .long   gdt
```
49-51做了CR0寄存器设置，开启了保护模式，CPU进入32bit寻址空间，在进入32bit之前，设定了lgdt，载入寄存器的值分别为16bit的0x17数值、32bit的gdt地址(大小端问题)。

关于SEG是一个宏定义，可以在inc/mmu.h中找到：

```c
172 #define SEG(type, base, lim, dpl)
173 { ((lim) >> 12) & 0xffff, (base) & 0xffff, ((base) >> 16) & 0xff,   \
174 ▎   type, 1, dpl, 1, (unsigned) (lim) >> 28, 0, 0, 1, 1,        \
175 ▎   (unsigned) (base) >> 24 }
```

可以注意到在global descriptor table中，仅有的2个内容都是base=0x00，并且他们的权限，分别是0x8处的 可执行、只读，以及0x10处的可写，不难猜到分别对应着代码段和数据段。

紧跟着后面的保护模式下第一条指令，使用段选择子0x8(对应代码段，同时ljmp会修改cs寄存器)跳转到设定选择子的函数中：

```assembly
55 ▎ ljmp    $PROT_MODE_CSEG, $protcseg
56 ▎
57 ▎ .code32
58 protcseg:
60 ▎ movw    $PROT_MODE_DSEG, %ax
61 ▎ movw    %ax, %ds
62 ▎ movw    %ax, %es
63 ▎ movw    %ax, %fs
64 ▎ movw    %ax, %gs
65 ▎ movw    %ax, %ss
...
68 ▎ movl    $start, %esp
69 ▎ call bootmain
```
在protcseg完成了对 数据段选择子的设定，设定完成后，流程跳转到bootmain这个符号地址处；

从bootmain这个地方开始，关于bootloader的部分进入了C语言的范畴，由于C语言生成汇编的特性，可以看到在call bootmain之前做了esp的调整，完成了临时栈的初始化。

由于栈的特性总是往下生长，并且此时内存中我们在地址`0x7C00 - 0x7DFF`的位置，故栈顶esp设定为start处的0x7C00是安全的，不会覆盖bootloader的代码。

### C 部分

代码来到boot/main.c中的bootmain，此时CPU已经在保护模式下，这段C语言代码做了一个比较简单的内容，从磁盘第二个sector开始，读取剩余的部分到内存中，这部分就是内核的ELF文件了，bootmain首先解析了ELF结构，将代码导入到对应pa地址上(物理地址)，这里可以简单的用readelf peek一下这个内核ELF：

```bash
ubuntu:~/github/lab$ readelf -l obj/kern/kernel

Elf file type is EXEC (Executable file)
Entry point 0x10000c
There are 3 program headers, starting at offset 52

Program Headers:
  Type           Offset   VirtAddr   PhysAddr   FileSiz MemSiz  Flg Align
  LOAD           0x001000 0xf0100000 0x00100000 0x0716c 0x0716c R E 0x1000
  LOAD           0x009000 0xf0108000 0x00108000 0x0a948 0x0a948 RW  0x1000
  GNU_STACK      0x000000 0x00000000 0x00000000 0x00000 0x00000 RWE 0x10

 Section to Segment mapping:
  Segment Sections...
   00     .text .rodata .stab .stabstr
   01     .data .bss
   02
```

在结合代码：

```c
 53         for (; ph < eph; ph++)
 54                 // p_pa is the load address of this segment (as well
 55                 // as the physical address)
 56                 readseg(ph->p_pa, ph->p_memsz, ph->p_offset);
 57
 58         // call the entry point from the ELF header
 59         // note: does not return!
 60         ((void (*)(void)) (ELFHDR->e_entry))();
```
实际的操作就是将LOAD的段，加载到对应的物理地址0x00100000和0x00108000位置上，然后跳到这个ELF的开始处执行，从这个地方开始，正式进入内核初始化阶段。

## 0x02 kernel initialization

TODO
