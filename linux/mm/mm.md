# mm


TODO:
关于current这个结构体，是从文件系统的哪个地方拿到的？

从do page fault入手

代码在arch/x86/mm里头


从arch/x86/setup.c中看到内核内存初始化的流程，从x86_init_ops这个里头，初始化pagetable

```c
setup_arch
    memblock_reserve(_text, __bss_stop - _text) @1
    memblock_reserve(0, PAGE_SIZE) @1
  ->pagetable_init
      ->paging_init
        
  
```

> @1: memblock_reserve是初始化内存之前的一些预留操作，系统在这个时候还没有完善的体系，这些代码为了保护内核代码不被覆盖掉





---

Q1: idt_setup_early_traps()陷入的一些设定，表保存到了IDTR？

内存映射表的一些例子？

Entry	Base Address (Start)	Length	Type
0	0x00000000	0x0009FC00	Usable (1)
1	0x0009FC00	0x00000400	Reserved (2)
2	0x00100000	0x7FEF0000	Usable (1)
3	0x7FF00000	0x00100000	Reserved (2)
4	0x100000000	0x400000000	Usable (1)

---














