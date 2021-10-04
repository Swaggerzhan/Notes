# ELF文件

主流的PC可执行文件格式一般有PE(Windows)和ELF(Linux)。

其中ELF文件包含可执行文件和可重定向文件等，比如Linux下的 __可执行文件以及.o文件和.so(动态链接库)和coredump文件等等。__


### 1. ELF Header 中的一些信息

整个ELF的大致一览图

![](./ELF_pic/2.png)

操作系统如何识别一个ELF文件呢？答案是通过ELF Header来进行识别，Header存在整个ELF文件的开头位置，具体结构由elf.h来进行定义，其中比较重要的结构体`Elf32_Ehdr`用来定义ELF Header。

```C
 typedef struct{
     unsigned char e_ident[EI_NIDENT]; /* Magic number and other info */
     Elf32_Half    e_type;         /* Object file type */
     Elf32_Half    e_machine;      /* Architecture */
     Elf32_Word    e_version;      /* Object file version */
     Elf32_Addr    e_entry;        /* Entry point virtual address */
     Elf32_Off e_phoff;        /* Program header table file offset */
     Elf32_Off e_shoff;        /* Section header table file offset */
     Elf32_Word    e_flags;        /* Processor-specific flags */
     Elf32_Half    e_ehsize;       /* ELF header size in bytes */
     Elf32_Half    e_phentsize;        /* Program header table entry size */
     Elf32_Half    e_phnum;        /* Program header table entry count */
     Elf32_Half    e_shentsize;        /* Section header table entry size */
     Elf32_Half    e_shnum;        /* Section header table entry count */
     Elf32_Half    e_shstrndx;     /* Section header string table index */
   } Elf32_Ehdr;
```

其中第一个字段e_ident我们称之为魔数，接下来解析比较重要的字段。

`e_shoff`全名`Section Header Table Offset`即节头表偏移，这个字段的意思是表明 __节头表在整个文件中的偏移量__ 。

`e_shnum`全名`Section Header Table number`即节头表的数量，表明整个ELF文件中， __关于节头表中的成员一共有多少个__ 。

`e_shentsize`全名`Section Header Entry Table Size`即节头表入口长度，表示的是每个 __节头表中的成员长度__ 。

有了以上的这些字段，我们就可以知道一个节头表的开始位置在哪，每个节头表中的成员大小多少，数量多少。

### 2. Section Header Table

同样的，Section Header Table它由结构体`Elf32_Shdr`来定义的。具体结构如下

```C
typedef struct {
    Elf32_Word sh_name;
    Elf32_Word sh_type;
    Elf32_Word sh_flags;
    Elf32_Addr sh_addr;
    Elf32_Off sh_offset;
    Elf32_Word sh_size;
    Elf32_Word sh_link;
    Elf32_Word sh_info;
    Elf32_Word sh_addralign;
    Elf32_Word sh_entsize;
} Elf32_Shdr;
```

我们只讲其中比较重要的部分。

`sh_name`故名思义，为这个节的名字，它是一个偏移量，并不是真正的一个字符串，需要配合另外的一个字段确定名字，具体后续解析。

`sh_type`此字段可以标明整个节的属性，比如`.data`，`.text`等等。

`sh_flags`这是一个位集，用来确定这个节的可读可写等等的属性。

`sh_offset`这是一个偏移量，记住，当前它只是一个 __节头表__ 而已，真正的节位置，是由这个sh_offset来定义的。

`sh_size`配合sh_offset，我们就能知道节的起始和大小了。

`sh_entsize`就比较有意思了，全名`Entry Table Size`即入口表长度，这是由于节存在着不同的类别，比如说，如果这个节是关于`.symtab`类型的，也就是说它是一个 __符号表的节__ ，那么表中的每个成员就是一个符号，多少个成员由这个字段定义，如果 __此节不是一个关于表的类型，那么这个字段直接为0即可__ 。

使用命令 `readelf`就可以查看到关于`Section Header Table`的信息了。

```shell
readelf -S ./a.out # -S 表示查看Section Header Table
readelf -s ./a.out # 则表示查看符号解析表，即.symtab类似节的具体信息
```

### 各个段中的一些具体细节

##### .symtab

这是一个关于符号表的一个段，具体这个表长度多大由节头表中的`sh_entsize`来确定。

.symtab中的每个节点由`Elf32_Sym`来进行定义

```C++
typedef struct
{
    Elf32_Word    st_name;        /* Symbol name (string tbl index) */
    Elf32_Addr    st_value;       /* Symbol value */
    Elf32_Word    st_size;        /* Symbol size */
    unsigned char st_info;        /* Symbol type and binding */
    unsigned char st_other;       /* Symbol visibility */
    Elf32_Section st_shndx;       /* Section index */
 } Elf32_Sym;
```

解析一下其中比较重要的信息

`st_value`即表示一个偏移量，具体偏移量就是关于某个节起始点的偏移量了，需要配合st_shndx来定位了，也有可能就直接是一个符号的地址。

`st_size`则用于表示这个符号的大小。

`st_shndx` 是一个索引，为节头表中的索引。

`st_info` __就比较重要了，这是一个8bit的段，但表示了2种东西，高4位表示Bind，低4位表示Type，即Bind和Type都有16种可能，但是我们只挑出比较重要的几个来进行讨论__ 。

```C++
/* Legal values for ST_BIND subfield of st_info (symbol binding).  */
#define STB_LOCAL   0       /* Local symbol */
#define STB_GLOBAL  1       /* Global symbol */
#define STB_WEAK    2       /* Weak symbol */
#define STB_NUM     3       /* Number of defined types.  */
#define STB_LOOS    10      /* Start of OS-specific */
#define STB_GNU_UNIQUE  10      /* Unique symbol.  */
#define STB_HIOS    12      /* End of OS-specific */
#define STB_LOPROC  13      /* Start of processor-specific */
#define STB_HIPROC  15      /* End of processor-specific */
/* Legal values for ST_TYPE subfield of st_info (symbol type).  */
#define STT_NOTYPE  0       /* Symbol type is unspecified */
#define STT_OBJECT  1       /* Symbol is a data object */
#define STT_FUNC    2       /* Symbol is a code object */
#define STT_SECTION 3       /* Symbol associated with a section */
#define STT_FILE    4       /* Symbol's name is file name */
#define STT_COMMON  5       /* Symbol is a common data object */
#define STT_TLS     6       /* Symbol is thread-local data object*/
#define STT_NUM     7       /* Number of defined types.  */
#define STT_LOOS    10      /* Start of OS-specific */
#define STT_GNU_IFUNC   10      /* Symbol is indirect code object */
#define STT_HIOS    12      /* End of OS-specific */
#define STT_LOPROC  13      /* Start of processor-specific */
#define STT_HIPROC  15      /* End of processor-specific */
```

比如STB中的`STB_LOCAL`和`STB_GLOBAL`以及`STB_WEAK`，都是用来表示一个符号的可见程度的， __正常的变量，为STB_GLOBAL形式，使用static来修饰变量将使得一个变量变成STB_LOCAL的形式，而STB_WEAK则需要使用对应的`__atrribute(())__` 来修饰才行__ 。

而STT中的数据其实一个索引，也就是`st_shndx`，正常情况下， __节头表中的第0个位置一般都是全为0的一个节点，也就是`STT_NOTYPE`，表示这个符号的类型并不知道，需要更多的信息才可以确定，剩下的就比较简单了`STT_OBJECT`表示一个变量，`STT_FUNC`则表示一个函数__ 。

我们可以使用一段代码来稍微确定一下，由此也更能深入理解static和extern这些关键字的实现手段。

```C
int data_1;
int data_2 = 1;
extern int data_3;
static int data_4;

void func1();
void func2(){}
extern void func3();
static void func4() {}
```

使用readelf来读取并且分析一下

```shell
Symbol table '.symtab' contains 16 entries:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT  UND 
     1: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS main.c
     2: 0000000000000000     0 SECTION LOCAL  DEFAULT    1 
     3: 0000000000000000     0 SECTION LOCAL  DEFAULT    3 
     4: 0000000000000000     0 SECTION LOCAL  DEFAULT    4 
     5: 0000000000000000     4 OBJECT  LOCAL  DEFAULT    4 data_4
     6: 0000000000000007     7 FUNC    LOCAL  DEFAULT    1 func4
     7: 0000000000000000     0 SECTION LOCAL  DEFAULT    6 
     8: 0000000000000000     0 SECTION LOCAL  DEFAULT    7 
     9: 0000000000000000     0 SECTION LOCAL  DEFAULT    5 
    10: 0000000000000004     4 OBJECT  GLOBAL DEFAULT  COM data_1
    11: 0000000000000000     4 OBJECT  GLOBAL DEFAULT    3 data_2
    12: 0000000000000000     7 FUNC    GLOBAL DEFAULT    1 func2
    13: 000000000000000e    51 FUNC    GLOBAL DEFAULT    1 main
    14: 0000000000000000     0 NOTYPE  GLOBAL DEFAULT  UND func1
    15: 0000000000000000     0 NOTYPE  GLOBAL DEFAULT  UND func3
```

我们可以清楚的看到，第5和第6第data_4和func4由于加上了static字段，都成为LOCAL形式的，并且通过Type我们可以明确的区分一个符号到底是函数，还是变量。

只不过这里需要注意的关于10行和14行以及15行，我们一个个解释。

对于第10行，OBJECT毫无争议，这是一个变量，GLOBAL也是如此，一个全局变量，只是这个Ndx居然是一个COM，表明是一个`.common`段的，这是由于编译器并不知道要将data这个变量放到哪个段中去，如果后续链接的时候，发现其他文件中也有关于data的定义，并且其是有值的，那么data将被放到`.data`段中去，而如果链接器没有找到其他关于data的定义，那么data将被放到`.bss`段中去。

那么对于14和15行，就比较好理解了，由于他们都没有定义一个函数，只是声明了而已，所以都被定义为`NOTYPE`，因为变量可以放入`.bss`段，而函数不行，如果链接器没有在其他文件中找到对应的符号，那么链接的时候将报错。

没有初始化的变量`st_shndx`都为4，其实使用一下`readelf -S`来查看每个节头表就会发现，4为`.bss`段。

我们这里以可运行文件进行分析，一个可运行文件要可以运行，肯定需要保存有一系列代码以及数据等等，为了正确的保存这些代码以及数据，ELF文件中有着一个个的节(Section)又或者称之为段(Segment)的分区，程序编译完的汇编数据一般放在代码段(常见的代码段有.code或者.text，不同的编译器有不同的编译方式)，而数据则一般放到数据段(常见的有.data)。

![](./ELF_pic/1.png)

ELF中只有这些段当然不够，操作系统需要知道当一个可运行文件加载到内存中的时候对应的段在哪些地址上，要做到这些，就需要用到段表(Section header table)。

段表中保存着所有段的信息，包括段名，段长度，段在文件中的偏移，段的读取权限。(段在加载后在内存中的位置-->不确定)。

使用以下命令可以看到整个ELF文件的段分布图。

```shell
# 使用objdump -h 命令可以取到整个文件的段信息
# 其中包括段的开始地址，长度，在文件中的偏移量等等
objdump -h target.elf 
# 或者可以使用readelf -S target.elf进行查看


# 使用objdump -s 命令可以得到整个ELF文件的hex值
objdump -s target.elf

# 使用objdump -d 命令可以打印ELF文件中的汇编代码(反汇编)
# 加上-M intel可以使用intel方式打印汇编代码
objdump -d target.elf
```

TODO: 关于其他段的具体用途

只有以上的段表还不足以让操作系统完全了解整个ELF文件，只有将所有所需信息都塞进一个叫做 __File Header__ 中才能做到，整个ELF文件结构:

![](./ELF_pic/2.png)

文件头保存着最基本的ELF文件信息，比如 <font color=F0000> ELF魔数，文件机器字节长度，数据的存储方式，版本，运行平台，ABI版本，ELF重定位类型，硬件平台，还有最重要的入口地址，程序头入口和长度，以及我们上面提到的段表的位置和长度及段的数量等。 </font>

段表实际上存储的方式一个数组，数组中每个元素表示一个段，其中有段的信息，也就是一个结构体:

![](./ELF_pic/3.png)

其中有些信息比较重要，比如sh_type表示这个段是属于什么类型的段，sh_addr对于一个可以被加载到内存的段则表示其被加载的虚拟地址，sh_offset表示它在文件中的偏移量，还有sh_link表示段链接信息等等。

TODO: ELF链接关键，符号

# ELF文件的静态链接

对于两个代码:

```C
// a.c
extern int shared;
int main(){
    int a = 100;
    swap( &a, &shared );
}
```

```C
// b.c
int shared = 1;
void swap( int* a, int* b){
    *a ^= *b ^= *a ^= *b;
}
```

其中分别编译出a.o和b.o，a中使用b中的shared变量和swap函数，但是对a.o使用nm命令查看信息可以明确的看到shared和swap地址并不知道，当使用 __ld a.o b.o -e main -o ab__ 命令将两个可重定向文件链接到一起时就可以查看到shared变量地址和swap函数地址。

这种静态链接，本质就像是将两个可重定向文件直接`合成`到一起，现代连接器一般采用两部，先分配空间和地址，之后对符号进行解析，并且重新写入对应符号在链接后的真正地址。

![](./ELF_pic/4.png)






