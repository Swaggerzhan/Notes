<font face="Monaco">

# Trap

## 0x00 Register

在RISC-V中，有几个特权寄存器尤为重要：

#### satp(supervisor address translation and pretection register)

> 用于保存一个物理地址，此物理地址为进程的顶级页表所在。

#### stvec(supervisor trap vector base address register)

> 用于保存一个虚拟地址，其中保存着进入trap之前所需要处理的代码，其在内核页表和用户页表所在的虚拟地址一致，且映射到相同物理地址。

#### sepc(supervisor exception program counter register)
> 用于保存一个虚拟地址，此地址为指令所在地，在trap中，用作保存原先用户态的指令。

#### sscratch(supervisor scratch register)

> sscratch所保存的是一个虚拟地址，在trap中，其指向的是trapframe所在地。

#### sstatus(supervisor status register)

> 状态寄存器，在节课中，我们只对SSP位和SIE位比较感兴趣，其中SIE表示中断是否开启，0位关闭中断，1则允许中断。SSP位为0时，sret指令会使得进入用户态，如果为1，那么sret后进入内核模式，然后重新将SSP置为0。

### 页表相关

其中stvec和sscratch都指向了一个固定的地址，一个是trampoline的地址，一个是trapframe地址，并且都是固定的，并且其所在的页表PTE_U都没有被置位(用户态不可访问)。


## 0x01 write系统调用的陷入过程

以shell代码为例，在`user/sh.c:134`的getcmd函数中：

```c
int
getcmd(char *buf, int nbuf)
{
  fprintf(2, "$ ");
  memset(buf, 0, nbuf);
  gets(buf, nbuf);
  if(buf[0] == 0) // EOF
    return -1;
  return 0;
}
```

其调用了fprintf，最终将调用write系统调用进行写入，而且系统调用write将由汇编代码进行处理，其代码所在地在`user/usys.S`：

```assembly
 .global write
 write:
  li a7, SYS_write # SYS_write为系统调用号
  ecall
  ret
```

其中ecall指令会做以下这些事情：

> 1. ecall将用户态转为内核态(可以执行特权指令)。

> 2. ecall将用户态下的程序计数器保存至sepc中，也就是ecall的下一行代码ret的地址。

> 3. 随后，ecall将当前的pc设定为stvec寄存器中的地址，这里的stvec寄存器中保存的，其实就是trampoline.S所在的地址。 而stvec寄存器，在一个进程启动并且进入用户态前，将由内核进行设定，并且这个trampoline.S代码所在的PTE中，没有PTE_U标记位，即用户态无法读写这个页表项。

ecall之后，代码跳转至trampoline.S中：

```assembly
	.section trampsec
.globl trampoline
trampoline:
.align 4
.globl uservec
uservec:    
	#
        # trap.c sets stvec to point here, so
        # traps from user space start here,
        # in supervisor mode, but with a
        # user page table.
        #
        # sscratch points to where the process's p->trapframe is
        # mapped into user space, at TRAPFRAME.
        #
        
	# swap a0 and sscratch
        # so that a0 is TRAPFRAME
        csrrw a0, sscratch, a0

        # save the user registers in TRAPFRAME
        sd ra, 40(a0)
        sd sp, 48(a0)
        sd gp, 56(a0)
        sd tp, 64(a0)
        sd t0, 72(a0)
        sd t1, 80(a0)
        sd t2, 88(a0)
        sd s0, 96(a0)
        sd s1, 104(a0)
        sd a1, 120(a0)
        sd a2, 128(a0)
        sd a3, 136(a0)
        sd a4, 144(a0)
        sd a5, 152(a0)
        sd a6, 160(a0)
        sd a7, 168(a0)
        sd s2, 176(a0)
        sd s3, 184(a0)
        sd s4, 192(a0)
        sd s5, 200(a0)
        sd s6, 208(a0)
        sd s7, 216(a0)
        sd s8, 224(a0)
        sd s9, 232(a0)
        sd s10, 240(a0)
        sd s11, 248(a0)
        sd t3, 256(a0)
        sd t4, 264(a0)
        sd t5, 272(a0)
        sd t6, 280(a0)

	# save the user a0 in p->trapframe->a0
        csrr t0, sscratch
        sd t0, 112(a0)

        # restore kernel stack pointer from p->trapframe->kernel_sp
        ld sp, 8(a0)

        # make tp hold the current hartid, from p->trapframe->kernel_hartid
        ld tp, 32(a0)

        # load the address of usertrap(), p->trapframe->kernel_trap
        ld t0, 16(a0)

        # restore kernel page table from p->trapframe->kernel_satp
        ld t1, 0(a0)
        csrw satp, t1
        sfence.vma zero, zero

        # a0 is no longer valid, since the kernel page
        # table does not specially map p->tf.

        # jump to usertrap(), which does not return
        jr t0
```

也就是运行uservec处的代码，其中，sscratch寄存器和stvec一样，在进程进入用户态时由内核进行设定，并且指向一个没有PTE_U标记的页表(用户态不可读写)，其地址所保存的，即为trapframe(kernel/proc.h)：

```c
struct trapframe {
  /*   0 */ uint64 kernel_satp;   // kernel page table
  /*   8 */ uint64 kernel_sp;     // top of process's kernel stack
  /*  16 */ uint64 kernel_trap;   // usertrap()
  /*  24 */ uint64 epc;           // saved user program counter
  /*  32 */ uint64 kernel_hartid; // saved kernel tp
  /*  40 */ uint64 ra;
  /*  48 */ uint64 sp;
  /*  56 */ uint64 gp;
  /*  64 */ uint64 tp;
  /*  72 */ uint64 t0;
  /*  80 */ uint64 t1;
  /*  88 */ uint64 t2;
  /*  96 */ uint64 s0;
  /* 104 */ uint64 s1;
  /* 112 */ uint64 a0;
  /* 120 */ uint64 a1;
  /* 128 */ uint64 a2;
  /* 136 */ uint64 a3;
  /* 144 */ uint64 a4;
  /* 152 */ uint64 a5;
  /* 160 */ uint64 a6;
  /* 168 */ uint64 a7;
  /* 176 */ uint64 s2;
  /* 184 */ uint64 s3;
  /* 192 */ uint64 s4;
  /* 200 */ uint64 s5;
  /* 208 */ uint64 s6;
  /* 216 */ uint64 s7;
  /* 224 */ uint64 s8;
  /* 232 */ uint64 s9;
  /* 240 */ uint64 s10;
  /* 248 */ uint64 s11;
  /* 256 */ uint64 t3;
  /* 264 */ uint64 t4;
  /* 272 */ uint64 t5;
  /* 280 */ uint64 t6;
};
```

可以看到，指令csrrw将a0和sscratch进行交换：

```assemlby
csrrw a0, sscratch, a0
```

a0之前所保存的是用户态的寄存器，内核对此并不感兴趣，而是将其保存至sscratch中，原先sscratch中保存的是trapframe的地址将被放到a0中，随后，执行sd指令，将用户态的所有寄存器都保存到trapframe中，包括之前被替换至sscratch中的a0：

```assembly
# save the user a0 in p->trapframe->a0
csrr t0, sscratch
sd t0, 112(a0)
```

操作后，除了ecall所做的用户态pc还保存在sepc中以外，其余寄存器均已经保存完整。

在保存完毕后，从trapframe中得到之前内核所保存的kernel_sp、kernel_hartid、kernel_trap、以及kernel_satp，随后通过指令：

```assembly
csrw satp, t1
```

将satp设定为内核PTE页，至此，进程获得了内核页表。

kernel_hartid表示的是当前进程所运行在的CPU核心index。

__注：trampoline是由内核设置的，并且由stvec寄存器来保存，stvec所指向的页表在所有进程中都是一样的(和内核也一样)，即映射到相同位置的trampoline代码上(trampoline只有一份代码)，所以在进行页表切换后，CPU对虚拟地址的读写并不会导致错误发生__。

之后，跳转到了t0所在的地址：

```assembly
# load the address of usertrap(), p->trapframe->kernel_trap
ld t0, 16(a0)
# ...
jr t0
```

即usertrap(kernel/trap.c)，这里的代码是进行陷入判定的，判断是因为何种原因陷入，然后进行处理：

```c
//
// handle an interrupt, exception, or system call from user space.
// called from trampoline.S
//
void
usertrap(void)
{
  int which_dev = 0;

  if((r_sstatus() & SSTATUS_SPP) != 0)
    panic("usertrap: not from user mode");

  // send interrupts and exceptions to kerneltrap(),
  // since we're now in the kernel.
  w_stvec((uint64)kernelvec);

  struct proc *p = myproc();
  
  // save user program counter.
  p->trapframe->epc = r_sepc();
  
  if(r_scause() == 8){
    // system call

    if(p->killed)
      exit(-1);

    // sepc points to the ecall instruction,
    // but we want to return to the next instruction.
    p->trapframe->epc += 4;

    // an interrupt will change sstatus &c registers,
    // so don't enable until done with those registers.
    intr_on();

    syscall();
  } else if((which_dev = devintr()) != 0){
    // ok
  } else {
    printf("usertrap(): unexpected scause %p pid=%d\n", r_scause(), p->pid);
    printf("            sepc=%p stval=%p\n", r_sepc(), r_stval());
    p->killed = 1;
  }

  if(p->killed)
    exit(-1);

  // give up the CPU if this is a timer interrupt.
  if(which_dev == 2)
    yield();

  usertrapret();
}
```

其中，代码：

```c
p->trapframe->epc = r_sepc();
```

在ecall进到trap后，中断被暂时关闭，而在执行系统调用之前，trap代码手动开启了中断，这就需要提前将之前未保存的寄存器继续保存至trapframe中，也就是sepc。

随后，检测陷入的原因是否是由于系统调用：

```c
if (r_scause() == 8 ){
    intr_on(); // 开中断
    syscall(); // 处理系统调用
}
```

syscall(kernel/syscall.c)代码为：

```c
static uint64 (*syscalls[])(void) = {
[SYS_fork]    sys_fork,
[SYS_exit]    sys_exit,
[SYS_wait]    sys_wait,
[SYS_pipe]    sys_pipe,
[SYS_read]    sys_read,
[SYS_kill]    sys_kill,
[SYS_exec]    sys_exec,
[SYS_fstat]   sys_fstat,
[SYS_chdir]   sys_chdir,
[SYS_dup]     sys_dup,
[SYS_getpid]  sys_getpid,
[SYS_sbrk]    sys_sbrk,
[SYS_sleep]   sys_sleep,
[SYS_uptime]  sys_uptime,
[SYS_open]    sys_open,
[SYS_write]   sys_write,
[SYS_mknod]   sys_mknod,
[SYS_unlink]  sys_unlink,
[SYS_link]    sys_link,
[SYS_mkdir]   sys_mkdir,
[SYS_close]   sys_close,
};

void
syscall(void)
{
  int num;
  struct proc *p = myproc();

  num = p->trapframe->a7;
  if(num > 0 && num < NELEM(syscalls) && syscalls[num]) {
    p->trapframe->a0 = syscalls[num]();
  } else {
    printf("%d %s: unknown sys call %d\n",
            p->pid, p->name, num);
    p->trapframe->a0 = -1;
  }
}
```

在trapframe->a7中保存系统调用号，trapframe->a0则保存系统调用的返回值，通过查表，进入系统调用函数。之后再跳转到sys_write函数(kernel/sysfile.c)中：

```c
uint64
sys_write(void)
{
  struct file *f;
  int n;
  uint64 p;

  if(argfd(0, 0, &f) < 0 || argint(2, &n) < 0 || argaddr(1, &p) < 0)
    return -1;

  return filewrite(f, p, n);
}
```

而且系统调用sys_write将同样从trapframe中得到函数的参数，在这个例子中，sys_write的三个参数分别存储在trapframe->a0，trapframe->a1和trapframe->a2中。

## 0x02 write系统调用的返回过程

在syscall中，trapframe->a0保存着sys_write系统调用的返回值，随后sys_write结束，函数回到usertrap(kernel/trap.c)中，其剩下的代码为：

```c
if(p->killed)
    exit(-1);

// give up the CPU if this is a timer interrupt.
if(which_dev == 2)
    yield();

usertrapret();
```

其中比较重要的是usertrapret()函数，它将做内核态返回用户态前的操作：

```c
//
// return to user space
//
void
usertrapret(void)
{
  struct proc *p = myproc();

  // we're about to switch the destination of traps from
  // kerneltrap() to usertrap(), so turn off interrupts until
  // we're back in user space, where usertrap() is correct.
  intr_off();

  // send syscalls, interrupts, and exceptions to trampoline.S
  w_stvec(TRAMPOLINE + (uservec - trampoline));

  // set up trapframe values that uservec will need when
  // the process next re-enters the kernel.
  p->trapframe->kernel_satp = r_satp();         // kernel page table
  p->trapframe->kernel_sp = p->kstack + PGSIZE; // process's kernel stack
  p->trapframe->kernel_trap = (uint64)usertrap;
  p->trapframe->kernel_hartid = r_tp();         // hartid for cpuid()

  // set up the registers that trampoline.S's sret will use
  // to get to user space.
  
  // set S Previous Privilege mode to User.
  unsigned long x = r_sstatus();
  x &= ~SSTATUS_SPP; // clear SPP to 0 for user mode
  x |= SSTATUS_SPIE; // enable interrupts in user mode
  w_sstatus(x);

  // set S Exception Program Counter to the saved user pc.
  w_sepc(p->trapframe->epc);

  // tell trampoline.S the user page table to switch to.
  uint64 satp = MAKE_SATP(p->pagetable);

  // jump to trampoline.S at the top of memory, which 
  // switches to the user page table, restores user registers,
  // and switches to user mode with sret.
  uint64 fn = TRAMPOLINE + (userret - trampoline);
  ((void (*)(uint64,uint64))fn)(TRAPFRAME, satp);
}
```

可以看到，usertrapret先关闭了中断，随后就开始设定stvec的地址，保存kernel下的pagetable、stack、usertrap指针、hartid至trapframe中，之后，将设定sstatus位的SSP和SPIE，其将在sret指令执行时生效，SSP表示执行sret后将内核态转为用户态，SPIE则表示sret后开启中断。

最后恢复之前的trapframe中保存的pc至sepc，重新获取之前用户态下的页表地址，作为第二参数(寄存器a1)，第一参数a0为trapframe地址，然后跳转至trampoline.S中的userret：

```assembly
globl userret
userret:
        # userret(TRAPFRAME, pagetable)
        # switch from kernel to user.
        # usertrapret() calls here.
        # a0: TRAPFRAME, in user page table.
        # a1: user page table, for satp.

        # switch to the user page table.
        csrw satp, a1
        sfence.vma zero, zero

        # put the saved user a0 in sscratch, so we
        # can swap it with our a0 (TRAPFRAME) in the last step.
        ld t0, 112(a0)
        csrw sscratch, t0

        # restore all but a0 from TRAPFRAME
        ld ra, 40(a0)
        ld sp, 48(a0)
        ld gp, 56(a0)
        ld tp, 64(a0)
        ld t0, 72(a0)
        ld t1, 80(a0)
        ld t2, 88(a0)
        ld s0, 96(a0)
        ld s1, 104(a0)
        ld a1, 120(a0)
        ld a2, 128(a0)
        ld a3, 136(a0)
        ld a4, 144(a0)
        ld a5, 152(a0)
        ld a6, 160(a0)
        ld a7, 168(a0)
        ld s2, 176(a0)
        ld s3, 184(a0)
        ld s4, 192(a0)
        ld s5, 200(a0)
        ld s6, 208(a0)
        ld s7, 216(a0)
        ld s8, 224(a0)
        ld s9, 232(a0)
        ld s10, 240(a0)
        ld s11, 248(a0)
        ld t3, 256(a0)
        ld t4, 264(a0)
        ld t5, 272(a0)
        ld t6, 280(a0)

	# restore user a0, and save TRAPFRAME in sscratch
        csrrw a0, sscratch, a0
        
        # return to user mode and user pc.
        # usertrapret() set up sstatus and sepc.
        sret
```

可以看到，起始时，恢复了用户态下的页表：

```assembly
csrw satp, a1
```

并且，a0作为第一参数，保存trapframe地址(还需要用)，所以，先取出原来trapframe中的a0，保存至sscratch中。

```assembly
ld t0, 112(a0) # 先从trapframe得到a0的数据，保存至t0中
csrw sscratch, t0 # 然后，将t0中的数据，也就是a0(用户态)保存到sscratch中
```

目前为止，sscratch和a0还未回到用户态的情况，sscratch中保存的是之前系统调用所返回的数值，a0还是trapframe，之后，从trapframe恢复所有的寄存器，然后再交换sscratch和a0：

```assembly
csrrw a0, sscratch, a0
```

之后，所有的寄存器就回到了进入内核态前的状态，sret后，模式转为用户态模式，a0为系统调用返回值。


以上，为xv6-riscv的write系统调用过程。

## ref

[课程地址](https://www.bilibili.com/video/BV19k4y1C7kA?p=5)




</font>