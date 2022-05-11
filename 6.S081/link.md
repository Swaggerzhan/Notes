<font face="Monaco">

# C Link

## 0x00 Static Link

在编译过程中，如果编译器遇到不懂的变量，那么编译器就会留空，生成可重定位的符号表以供后续链接器进行重定位，其结构定义于elf/elf.h(glibc下)

```c
/* Relocation table entry without addend (in section of type SHT_REL).  */

typedef struct
{
  Elf32_Addr	r_offset;		/* Address */
  Elf32_Word	r_info;			/* Relocation type and symbol index */
} Elf32_Rel;

/* I have seen two different definitions of the Elf64_Rel and
   Elf64_Rela structures, so we'll leave them out until Novell (or
   whoever) gets their act together.  */
/* The following, at least, is used on Sparc v9, MIPS, and Alpha.  */

typedef struct
{
  Elf64_Addr	r_offset;		/* Address */
  Elf64_Xword	r_info;			/* Relocation type and symbol index */
} Elf64_Rel;
```

对于符号，每个编译器都会在编译的时候形成，不管是找得到的、还是找不到的(那就留空等链接器来，需要生成重定向表)，所有的符号，都需要记录在符号表中，同样定义在elf.h下：

```c
/* Symbol table entry.  */

typedef struct
{
  Elf32_Word	st_name;		/* Symbol name (string tbl index) */
  Elf32_Addr	st_value;		/* Symbol value */
  Elf32_Word	st_size;		/* Symbol size */
  unsigned char	st_info;		/* Symbol type and binding */
  unsigned char	st_other;		/* Symbol visibility */
  Elf32_Section	st_shndx;		/* Section index */
} Elf32_Sym;

typedef struct
{
  Elf64_Word	st_name;		/* Symbol name (string tbl index) */
  unsigned char	st_info;		/* Symbol type and binding */
  unsigned char st_other;		/* Symbol visibility */
  Elf64_Section	st_shndx;		/* Section index */
  Elf64_Addr	st_value;		/* Symbol value */
  Elf64_Xword	st_size;		/* Symbol size */
} Elf64_Sym;
```

### 1. 链接例子


```c
extern int sum(int x, int y);
extern int y;
int x = 10;
int main() {
    int z = sum(x, y);
}
```

可以看看编译器为他生成的符号表：

```shell
root@paperSwagger:# readelf -s main.o 

Symbol table '.symtab' contains 14 entries:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT  UND 
     1: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS main.c
     2: 0000000000000000     0 SECTION LOCAL  DEFAULT    1 
     3: 0000000000000000     0 SECTION LOCAL  DEFAULT    3 
     4: 0000000000000000     0 SECTION LOCAL  DEFAULT    4 
     5: 0000000000000000     0 SECTION LOCAL  DEFAULT    6 
     6: 0000000000000000     0 SECTION LOCAL  DEFAULT    7 
     7: 0000000000000000     0 SECTION LOCAL  DEFAULT    8 
     8: 0000000000000000     0 SECTION LOCAL  DEFAULT    5 
     9: 0000000000000000     4 OBJECT  GLOBAL DEFAULT    3 x
    10: 0000000000000000    43 FUNC    GLOBAL DEFAULT    1 main
    11: 0000000000000000     0 NOTYPE  GLOBAL DEFAULT  UND y
    12: 0000000000000000     0 NOTYPE  GLOBAL DEFAULT  UND _GLOBAL_OFFSET_TABLE_
    13: 0000000000000000     0 NOTYPE  GLOBAL DEFAULT  UND sum
```

其中我们所关注的，便是那些我们定义的符号，__比如x符号和main符号，编译器找到了，然后将其定义为对应的属性，OBJECT对应变量，FUNC对应函数。而y和sum，编译器没有找到，所以将其定义为NOTYPE，并且期待链接器来去补全它__。

或者也可以直接用-r来查看重定向的表：

```shell
root@paperSwagger:# readelf -r main.o 

Relocation section '.rela.text' at offset 0x278 contains 3 entries:
  Offset          Info           Type           Sym. Value    Sym. Name + Addend
00000000000e  000b00000002 R_X86_64_PC32     0000000000000000 y - 4
000000000014  000900000002 R_X86_64_PC32     0000000000000000 x - 4
00000000001d  000d00000004 R_X86_64_PLT32    0000000000000000 sum - 4

Relocation section '.rela.eh_frame' at offset 0x2c0 contains 1 entry:
  Offset          Info           Type           Sym. Value    Sym. Name + Addend
000000000020  000200000002 R_X86_64_PC32     0000000000000000 .text + 0
```

如果在直接链接而不给定真正的sum和y地址情况下，那么ld必定会报错，并显示无法找到y和sum的符号(或者你多给了一个强x符号，这个其实已经定义在当前.o文件中，多个相同的强符号同样会报错)。

## 0x01 Dynamic Link

之前的例子代码中，缺乏了y和sum符号，现在，通过以下代码来生成一个libsum.so的动态链接库：

```c
// sum.c
int y = 20;

int sum(int x, int y) {
	return x + y;
}
```

生成，并且将其链接到之前的main.o上，再来看看有何不同

```shell
gcc -shared -fPIC sum.c -o libsum.so
gcc main.o ./libsum.so
```

首先是多了很多section，其中多了一个符号表，它的名字是.dynsym，动态链接符号表，可以看到，y和sum在这里面被解析出来了，为OBJECT和FUNC，显然是对的。

```shell
root@paperSwagger:# readelf --dyn-syms sum

Symbol table '.dynsym' contains 8 entries:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT  UND 
     1: 0000000000000000     0 NOTYPE  WEAK   DEFAULT  UND _ITM_deregisterTMCloneTab
     2: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND __libc_start_main@GLIBC_2.2.5 (2)
     3: 0000000000000000     0 NOTYPE  WEAK   DEFAULT  UND __gmon_start__
     4: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND sum
     5: 0000000000000000     0 NOTYPE  WEAK   DEFAULT  UND _ITM_registerTMCloneTable
     6: 0000000000004018     4 OBJECT  GLOBAL DEFAULT   26 y
     7: 0000000000000000     0 FUNC    WEAK   DEFAULT  UND __cxa_finalize@GLIBC_2.2.5 (2)
```

最早的时候，.rel重定位表是编译器留给链接器的，现在链接器早已运行过，之后ELF文件一旦运行，将直接被加载至内存，这样会有问题？可以发现还是有重定位表的：

```shell
root@paperSwagger:# readelf -r sum

Relocation section '.rela.dyn' at offset 0x548 contains 9 entries:
  Offset          Info           Type           Sym. Value    Sym. Name + Addend
000000003da8  000000000008 R_X86_64_RELATIVE                    1140
000000003db0  000000000008 R_X86_64_RELATIVE                    1100
000000004008  000000000008 R_X86_64_RELATIVE                    4008
000000003fd8  000100000006 R_X86_64_GLOB_DAT 0000000000000000 _ITM_deregisterTMClone + 0
000000003fe0  000200000006 R_X86_64_GLOB_DAT 0000000000000000 __libc_start_main@GLIBC_2.2.5 + 0
000000003fe8  000300000006 R_X86_64_GLOB_DAT 0000000000000000 __gmon_start__ + 0
000000003ff0  000500000006 R_X86_64_GLOB_DAT 0000000000000000 _ITM_registerTMCloneTa + 0
000000003ff8  000700000006 R_X86_64_GLOB_DAT 0000000000000000 __cxa_finalize@GLIBC_2.2.5 + 0
000000004018  000600000005 R_X86_64_COPY     0000000000004018 y + 0

Relocation section '.rela.plt' at offset 0x620 contains 1 entry:
  Offset          Info           Type           Sym. Value    Sym. Name + Addend
000000003fd0  000400000007 R_X86_64_JUMP_SLO 0000000000000000 sum + 0
```

这些重定位符号将由动态链接器来去填补，在加载过程中，Linux内核会启动动态链接器，随后将elf入口设定为动态链接器的入口。

## 0x02 Linux kernel load ELF

以下以Linux kernel 2.6为例。

在ELF加载的代码中，我们比较关注的是位于fs/exec.c的加载器，代码为：

```c
/*
 * sys_execve() executes a new program.
 */
/**
 * 
 */
int do_execve(char * filename,
	char __user *__user *argv,
	char __user *__user *envp,
	struct pt_regs * regs)
{
	struct linux_binprm *bprm;
	struct file *file;
	int retval;
	int i;

	retval = -ENOMEM;
	/**
	 * 动态分配linux_binprm结构，并用新的可执行文件的数据填充这个结构。
	 */
	bprm = kmalloc(sizeof(*bprm), GFP_KERNEL);
	if (!bprm)
		goto out_ret;
	memset(bprm, 0, sizeof(*bprm));

	/**
	 * open_exec调用path_lookup,dentry_open,path_release以获得与可执行文件
	 * 相关的目录项对象、文件对象和索引结点对象。
	 * file也可能是错误码，所以调用IS_ERR(file)进行判断
	 * 注：open_exec也会检查执行权限。
	 */
	file = open_exec(filename);
	retval = PTR_ERR(file);
	if (IS_ERR(file))
		goto out_kfree;

	/**
	 * 在多处理器中，调用sched_exec以确定最小负载CPU以执行新程序。
	 * 并把当前进程转移过去。
	 */
	sched_exec();

	bprm->p = PAGE_SIZE*MAX_ARG_PAGES-sizeof(void *);

	bprm->file = file;
	bprm->filename = filename;
	bprm->interp = filename;
	bprm->mm = mm_alloc();
	retval = -ENOMEM;
	if (!bprm->mm)
		goto out_file;

	/**
	 * init_new_context检查当前进程是否使用自定义局部描述符表。
	 * 如果是，函数为新程序分配和准备一个新的LDT
	 */
	retval = init_new_context(current, bprm->mm);
	if (retval < 0)
		goto out_mm;

	bprm->argc = count(argv, bprm->p / sizeof(void *));
	if ((retval = bprm->argc) < 0)
		goto out_mm;

	bprm->envc = count(envp, bprm->p / sizeof(void *));
	if ((retval = bprm->envc) < 0)
		goto out_mm;

	retval = security_bprm_alloc(bprm);
	if (retval)
		goto out;

	/**
	 * prepare_binprm填充linux_binprm数据结构。
	 */
	retval = prepare_binprm(bprm);
	if (retval < 0)
		goto out;

	/**
	 * 把路径名，命令行参数，环境串复制到一个或者多个新分配的页框中，
	 * 最终，它们会被分配给用户态地址空间
	 */
	retval = copy_strings_kernel(1, &bprm->filename, bprm);
	if (retval < 0)
		goto out;

	bprm->exec = bprm->p;
	retval = copy_strings(bprm->envc, envp, bprm);
	if (retval < 0)
		goto out;

	retval = copy_strings(bprm->argc, argv, bprm);
	if (retval < 0)
		goto out;

	/**
	 * 调用search_binary_handler函数对formats链表进行扫描，并尽力应用每个元素的load_binary方法，把
	 * linxu_binprm数据结构传递给这个函数。只要load_binary方法成功应答了文件的可执行格式。对formats的扫描就终止。
	 */
	retval = search_binary_handler(bprm,regs);
	if (retval >= 0) {
		/**
		 * 在formats中找到了，就释放linux_binprm数据结构，并返回load_binary所获得的代码。
		 */
		free_arg_pages(bprm);

		/* execve success */
		security_bprm_free(bprm);
		acct_update_integrals();
		update_mem_hiwater();
		kfree(bprm);
		return retval;
	}

out:
	/* Something went wrong, return the inode and free the argument pages*/
	/**
	 * 可执行文件格式不在formats链表中，就释放所分配的所有页框并返回错误码-ENOEXEC
	 */
	for (i = 0 ; i < MAX_ARG_PAGES ; i++) {
		struct page * page = bprm->page[i];
		if (page)
			__free_page(page);
	}

	if (bprm->security)
		security_bprm_free(bprm);

out_mm:
	if (bprm->mm)
		mmdrop(bprm->mm);

out_file:
	if (bprm->file) {
		allow_write_access(bprm->file);
		fput(bprm->file);
	}

out_kfree:
	kfree(bprm);

out_ret:
	return retval;
}
```

这里的struct linux_binprm中，我们比较关注ELF的结构体，它位于include/linux/binfmts.h中：

```c
/*
  * This structure defines the functions that are used to load the binary formats that
  * linux accepts.
  */
struct linux_binfmt {
    struct list_head lh;
    struct module *module;
    int (*load_binary)(struct linux_binprm *);
    int (*load_shlib)(struct file *);
    int (*core_dump)(struct coredump_params *cprm);
    unsigned long min_coredump;     /* minimal dump size */
 };
```

寻找一个可以处理ELF文件的handler，在do_exec中的代码：

```c
retval = search_binary_handler(bprm,regs);
```

这个search_binary_handler寻找的其实是一个可以处理二进制文件的句柄，这些句柄以链表形式在内核中存储，其代码位于fs/binfmt_elf.c下，代码非常的长，这里仅截取一些比较核心的：

```c
// 一些填充的操作
for (i = 0; i < loc->elf_ex.e_phnum; i++) {
    /*  3.1  检查是否有需要加载的解释器  */
    if (elf_ppnt->p_type == PT_INTERP) {
        /* This is the program interpreter used for
         * shared libraries - for now assume that this
         * is an a.out format binary
        */

        /*  3.2 根据其位置的p_offset和大小p_filesz把整个"解释器"段的内容读入缓冲区  */
        retval = kernel_read(bprm->file, elf_ppnt->p_offset,
             elf_interpreter,
             elf_ppnt->p_filesz);

        if (elf_interpreter[elf_ppnt->p_filesz - 1] != '\0')
            goto out_free_interp;
        /*  3.3 通过open_exec()打开解释器文件 */
        interpreter = open_exec(elf_interpreter);



        /* Get the exec headers 
        3.4  通过kernel_read()读入解释器的前128个字节，即解释器映像的头部。*/
        retval = kernel_read(interpreter, 0,
             (void *)&loc->interp_elf_ex,
             sizeof(loc->interp_elf_ex));


        break;
    }
    elf_ppnt++;
}
```

在检测ELF文件需要动态链接器后，Linux内核会读入这些所需的动态链接器，并且先启动它们，但这之前，还需要读入ELF文件到对应的内存中：

```c
for(i = 0, elf_ppnt = elf_phdata; i < loc->elf_ex.e_phnum; i++, elf_ppnt++)
    // 读取各个需要加载的程序段到内存...
    if (elf_ppnt->p_type != PT_LOAD)
        continue;  
```

随后就是判断动态解析器是否需要，如果需要，那入口就是动态解析器入口，如果不需要(静态链接)，那程序入口直接就是ELF文件的entry了：

```c
if (elf_interpreter) {
    if (interpreter_type == INTERPRETER_AOUT)
        elf_entry = load_aout_interp(&loc->interp_ex, interpreter);
    else
        elf_entry = load_elf_interp(&loc->interp_elf_ex, interpreter, &interp_load_addr);
    // 略...
} else {
    elf_entry = loc->elf_ex.e_entry;
}
```

到这里，关于ELF的加载工作就结束了，之后的start_thread()就会修改EIP和ESP从入口处启动程序，如果需要动态链接，那么之后的工作就在动态链接器上了。


</font>