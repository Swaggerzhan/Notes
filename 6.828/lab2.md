# lab2 notes

## 0x00 preview

在上一个lab中，内核做了一个非常简单的映射，映射表存在kernel ELF中，是一个编译时期就构建的页表，它将虚拟地址`0x000000 ~ 0x400000`和`KBASE + 0x400000` 映射到一个相同的物理地址`0x000000 ~ 0x400000`上，之所以这么做是方便页表开启之后，指令从直接物理内存访问跳转到虚拟内存访问不受影响。

当然，4M的空间远远不够，lab2要求重建一个全新的页表，使得os可以管理这些物理内存。

## 0x01 mapping layout

JOS中，KERNBASE起于`0xF0000000`，从3.75G左右开始向高地址位映射，内核的虚拟地址到物理地址是线性映射的，32bit下实际上内核只能管理256MB的内存。




## 0x01 Exercise 1

Exercise 1要求实现下面的3个函数：

```c
boot_alloc()
mem_init()
page_init()
page_alloc()
page_free()
```
下面是这些函数的实现。

### boot_alloc

boot_alloc是介于内核刚刚启动到page_init之前，在系统还没有一个成型的“free list”的时候使用，用于申请一块可用的内存空间，此时可以直接将物理内存看成一块大的数组，从内核ELF载入的尾部，也就是end符号的地方，向上对齐4096即可找到可以用的物理内存，把这里记为nextfree起始即可。

```c
static void *
boot_alloc(uint32_t n)
{
    static char *nextfree;
    char *result;
    if (!nextfree) {
        extern char end[];
        nextfree = ROUNDUP((char *) end, PGSIZE);
        cprintf("boot_alloc(): first avail mem addr at: 0x%x\n", nextfree);
    }

    if (n == 0) {
        return nextfree;
    }
    n = ROUNDUP(n, PGSIZE);
    result = nextfree;
    nextfree = nextfree + n;
    return result;
}
```

### page_init

page_init中是有一些全局变量的，比如：
* page_free_list，这是一个PageInfo结构体的链表，其实就是物理页对应的那个结构体
* pages，这是PageInfo结构体的存储空间，这是一个数组，连续的内存空间

pages的内存地址来自上面boot_alloc获取到的，从这个init之后，就再也不会使用到boot_alloc了，转而使用page_alloc；

page_init直观的对每一个物理页 创建一个对应的PageInfo结构体，包括那些不可用的，如BIOS映射页面的地方，这里需要注意的就这几个：

* 0页帧 不可用
* IO hold预留的 不可用
* 用于加载ELF内核的 不可用
* boot_alloc 申请到用于存储其他东西的，不可用(这里最后其实就是用来保存pages)

除去这些不可用的，剩余的就能使用page_free_list链起来了。

```c
void
page_init(void)
{
    size_t hole_end_index = npages_basemem + (EXTPHYSMEM - IOPHYSMEM) / PGSIZE;
    char* addr_end_of_pages = (char*)pages + pages_size;
    size_t end_pages_index = PADDR(addr_end_of_pages) >> 12;
    cprintf("hole_end_index: %u, end_pages_index: %u\n",
            hole_end_index, end_pages_index);

    page_free_list = NULL;
    size_t i;
    for (i = 0; i < npages; i++) {
        if (i == 0) {
            pages[i].pp_ref = 1;
            pages[i].pp_link = NULL;
            continue;
        }

        // IO hole
        if (i >= npages_basemem && i < hole_end_index) {
            pages[i].pp_ref = 1;
            pages[i].pp_link = NULL;
            continue;
        }

        // maybe there is some data in last pages, so marked as used
        // those are store kernel ELF and pages
        if (i >= hole_end_index && i <= end_pages_index + 1) {
            pages[i].pp_ref = 1;
            pages[i].pp_link = NULL;
            continue;
        }

        pages[i].pp_ref = 0;
        pages[i].pp_link = page_free_list;
        page_free_list = &pages[i];
    }
    return;
}
```

### page_alloc && page_free

这两个函数一起讲吧，一个是申请，从链表中拿掉一个，一个则是放回链表中，都很简单：


### 地址转化

os中关于va和pa的地址转化是至关重要的，从内核管理角度，至少这几个是需要保证的：

* va 到 pa 的映射(索引)
* pa 到 va 的映射(索引)
* PageInfo结构体和pa的索引
* PageInfo结构体和va的索引

xv6中的映射做的比较简单，xv6只有256M的内存，并且内核是从KERNBASE开始载入的，到0xFFFFFFFF刚好有256MB；在预期的xv6方案中，内核会完成虚拟地址`KERNBASE ~ 0xFFFFFFFF`映射到物理地址`0x00000000 ~ 0x10000000`。

基于上面的点，va和pa的相互映射可用快速解决，直接通过KERNBASE就能索引；pa和PageInfo也是比较简单，直接取高22bit即可，也就是PPN即是物理地址，又能通过pages[PPN]拿到结构体；va也是同理，既然拿到了PPN的物理地址，加上KERNBASE的offset，就能拿到这块物理地址在内核页表中的地址了(这个主要是用于内核读写着块地址使用)。



## 0x02 notes

内核不想要“申请”这些内存，它只想拥有这个权限而已，

to be continued...


# ref

https://pdos.csail.mit.edu/6.828/2018/labs/lab2/