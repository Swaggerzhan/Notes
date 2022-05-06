<font face="Monaco">

# xv6-riscv threads switch 笔记

## 0x00 xv6-riscv中的进程

首先是位于kernel/proc.h下的一个重要结构体：

```c
// kernel/proc.h
enum procstate { UNUSED, SLEEPING, RUNNABLE, RUNNING, ZOMBIE };

// Per-process state
struct proc {
  struct spinlock lock;

  // p->lock must be held when using these:
  enum procstate state;        // Process state
  struct proc *parent;         // Parent process
  void *chan;                  // If non-zero, sleeping on chan
  int killed;                  // If non-zero, have been killed
  int xstate;                  // Exit status to be returned to parent's wait
  int pid;                     // Process ID

  // these are private to the process, so p->lock need not be held.
  uint64 kstack;               // Virtual address of kernel stack
  uint64 sz;                   // Size of process memory (bytes)
  pagetable_t pagetable;       // User page table
  struct trapframe *trapframe; // data page for trampoline.S
  struct context context;      // swtch() here to run process
  struct file *ofile[NOFILE];  // Open files
  struct inode *cwd;           // Current directory
  char name[16];               // Process name (debugging)

  struct usyscall *syscall_page;
};
```

其中关于trapframe在之前的syscall中有提及到，是用于保存用户上下文的，随后才会进入到内核态中，这里比较关注，其实是context的内容：

```c
// kernel/proc.h
// Saved registers for kernel context switches.
struct context {
  uint64 ra;
  uint64 sp;

  // callee-saved
  uint64 s0;
  uint64 s1;
  uint64 s2;
  uint64 s3;
  uint64 s4;
  uint64 s5;
  uint64 s6;
  uint64 s7;
  uint64 s8;
  uint64 s9;
  uint64 s10;
  uint64 s11;
};
```

其context所保存的，是当前进程所在的“内核线程”中的寄存器，同时，对于swtich的函数位于kernel/switch.S：

```assembly
# kernel/switch.S
# Context switch
#
#   void swtch(struct context *old, struct context *new);
# 
# Save current registers in old. Load from new.	


.globl swtch
swtch:
        sd ra, 0(a0)
        sd sp, 8(a0)
        sd s0, 16(a0)
        sd s1, 24(a0)
        sd s2, 32(a0)
        sd s3, 40(a0)
        sd s4, 48(a0)
        sd s5, 56(a0)
        sd s6, 64(a0)
        sd s7, 72(a0)
        sd s8, 80(a0)
        sd s9, 88(a0)
        sd s10, 96(a0)
        sd s11, 104(a0)

        ld ra, 0(a1)
        ld sp, 8(a1)
        ld s0, 16(a1)
        ld s1, 24(a1)
        ld s2, 32(a1)
        ld s3, 40(a1)
        ld s4, 48(a1)
        ld s5, 56(a1)
        ld s6, 64(a1)
        ld s7, 72(a1)
        ld s8, 80(a1)
        ld s9, 88(a1)
        ld s10, 96(a1)
        ld s11, 104(a1)
        
        ret
```

switch的功能就是将一个上下文保存至context中，然后恢复另外一个context的内容到寄存器中(完成跳转)。

接下来就是从fork系统调用入手了，在kernel/proc.c的fork中：

```c
// Create a new process, copying the parent.
// Sets up child kernel stack to return as if from fork() system call.
int
fork(void)
{
  int i, pid;
  struct proc *np;
  struct proc *p = myproc();

  // Allocate process.
  if((np = allocproc()) == 0){
    return -1;
  }

  // Copy user memory from parent to child.
  if(uvmcopy(p->pagetable, np->pagetable, p->sz) < 0){
    freeproc(np);
    release(&np->lock);
    return -1;
  }
  np->sz = p->sz;

  np->parent = p;

  // copy saved user registers.
  *(np->trapframe) = *(p->trapframe);

  // Cause fork to return 0 in the child.
  np->trapframe->a0 = 0;

  // increment reference counts on open file descriptors.
  for(i = 0; i < NOFILE; i++)
    if(p->ofile[i])
      np->ofile[i] = filedup(p->ofile[i]);
  np->cwd = idup(p->cwd);
  safestrcpy(np->name, p->name, sizeof(p->name));
  pid = np->pid;
  np->state = RUNNABLE;
  release(&np->lock);
  return pid;
}
```

这里也许还看不出什么端倪，运行在当前的stack，是调用fork父进程所在的内核stack，return pid的操作最后将由trapframe->a0来去接收，这就是为什么fork在父进程中返回的是子进程的pid了。同时，可以注意到，fork函数调用了allocproc函数来“创建”一个新的proc结构体，这就是子进程的结构体了：

```c
// kernel/proc.c
// Look in the process table for an UNUSED proc.
// If found, initialize state required to run in the kernel,
// and return with p->lock held.
// If there are no free procs, or a memory allocation fails, return 0.
static struct proc*
allocproc(void)
{
  struct proc *p;

  for(p = proc; p < &proc[NPROC]; p++) {
    acquire(&p->lock);
    if(p->state == UNUSED) {
      goto found;
    } else {
      release(&p->lock);
    }
  }
  return 0;

found:
  p->pid = allocpid();

  // Allocate a trapframe page.
  if((p->trapframe = (struct trapframe *)kalloc()) == 0){
    release(&p->lock);
    return 0;
  }

  // An empty user page table.
  p->pagetable = proc_pagetable(p);
  if(p->pagetable == 0){
    freeproc(p);
    release(&p->lock);
    return 0;
  }

  // Set up new context to start executing at forkret,
  // which returns to user space.
  memset(&p->context, 0, sizeof(p->context));
  p->context.ra = (uint64)forkret;
  p->context.sp = p->kstack + PGSIZE;

  return p;
}
```

代码只是简单的获取一个子进程该有的proc，然后设定一些子进程的pagetable，在线程切换中，我们关注的是底下的一个p->context设置的操作：

```c
// Set up new context to start executing at forkret,
// which returns to user space.
memset(&p->context, 0, sizeof(p->context));
p->context.ra = (uint64)forkret;
p->context.sp = p->kstack + PGSIZE;
```

它将新创建的进程的p->context.ra设定为forkret函数，然后将p->context.sp设置为当前新进程的内核栈起始位置(栈是倒着的)，__ra寄存器，将在ret的时候生效，将ra中的数值放到pc寄存器中__。

之后的一切就如同fork函数中的一样，正常返回，__而新创建的子进程，直到内核调用它之前，都不会返回__。

## 0x01 boot启动的角度查看proc的初始化

首先在kernel/main.c中main函数做了一系列初始化操作，其中调用了一个procinit函数：

```c
// kernel/proc.c
// initialize the proc table at boot time.
void
procinit(void)
{
  struct proc *p;
  
  initlock(&pid_lock, "nextpid");
  for(p = proc; p < &proc[NPROC]; p++) {
      initlock(&p->lock, "proc");

      // Allocate a page for the process's kernel stack.
      // Map it high in memory, followed by an invalid
      // guard page.
      char *pa = kalloc();
      if(pa == 0)
        panic("kalloc");
      uint64 va = KSTACK((int) (p - proc));
      kvmmap(va, (uint64)pa, PGSIZE, PTE_R | PTE_W);
      p->kstack = va;
  }
  kvminithart();
}
```

其作用是为每个proc结构体生成内核stack，vx6-riscv支持最大64个进程，每个proc的内核stack都通过invalid的page来进行保护。

随后，调用了一个scheduler函数，它同样位于kernel/proc.c中：

```c
// kernel/proc.c
// Per-CPU process scheduler.
// Each CPU calls scheduler() after setting itself up.
// Scheduler never returns.  It loops, doing:
//  - choose a process to run.
//  - swtch to start running that process.
//  - eventually that process transfers control
//    via swtch back to the scheduler.
void
scheduler(void)
{
  struct proc *p;
  struct cpu *c = mycpu();
  
  c->proc = 0;
  for(;;){
    // Avoid deadlock by ensuring that devices can interrupt.
    intr_on();
    
    int found = 0;
    for(p = proc; p < &proc[NPROC]; p++) {
      acquire(&p->lock);
      if(p->state == RUNNABLE) {
        // Switch to chosen process.  It is the process's job
        // to release its lock and then reacquire it
        // before jumping back to us.
        p->state = RUNNING;
        c->proc = p;
        swtch(&c->context, &p->context);

        // Process is done running for now.
        // It should have changed its p->state before coming back.
        c->proc = 0;

        found = 1;
      }
      release(&p->lock);
    }
#if !defined (LAB_FS)
    if(found == 0) {
      intr_on();
      asm volatile("wfi");
    }
#else
    ;
#endif
  }
}
```

scheduler是一个死循环，持续的从proc数组中获取进程结构体，然后上锁(后面会解锁)，检查是否为RUNNABLE，如果进程可以运行，那就使用swtch来去恢复这个进程中的p->context，回忆一下一个allocproc中有一个p->context.ra和sp的设置操作，当一个进程是第一次被运行的时候，swtch总是会跳到forkret中，同时，当前执行的上下文(我们比较关注sa和sp)会被保存至c->context中。

```c
// kernel/proc.c
// A fork child's very first scheduling by scheduler()
// will swtch to forkret.
void
forkret(void)
{
  static int first = 1;

  // Still holding p->lock from scheduler.
  release(&myproc()->lock);

  if (first) {
    // File system initialization must be run in the context of a
    // regular process (e.g., because it calls sleep), and thus cannot
    // be run from main().
    first = 0;
    fsinit(ROOTDEV);
  }

  usertrapret();
}
```

可以看到，forkret在一开始就解锁了之前的p->lock，然后进入usertrapret返回，有趣的是，由于之前在fork中子进程拷贝了父进程的一切页表，所以trapframe和父进程一摸一样(除了trapframe->a0)，自然，当usertrapret的时候，子进程返回用户态的时候会和父进程在一摸一样的位置(相同的pc)。

## 0x02 中断导致的进程切换

同样的，硬件timer的触发会导致程序陷入，然后内核开始保存用户态的数据到trapframe中，之后才开始着手处理是什么原因导致的：

```c
// kernel/trap.c  usertrap()
// give up the CPU if this is a timer interrupt.
if(which_dev == 2)
    yield();
```

如果中断是timer导致的，那就yield：

```c
// kernel/proc.c
// Give up the CPU for one scheduling round.
void
yield(void)
{
  struct proc *p = myproc();
  acquire(&p->lock);
  p->state = RUNNABLE;
  sched();
  release(&p->lock);
}
```

它获取当前的proc的lock，然后进到了sched()：

```c
// kernel/proc.c
// Switch to scheduler.  Must hold only p->lock
// and have changed proc->state. Saves and restores
// intena because intena is a property of this
// kernel thread, not this CPU. It should
// be proc->intena and proc->noff, but that would
// break in the few places where a lock is held but
// there's no process.
void
sched(void)
{
  int intena;
  struct proc *p = myproc();

  if(!holding(&p->lock))
    panic("sched p->lock");
  if(mycpu()->noff != 1)
    panic("sched locks");
  if(p->state == RUNNING)
    panic("sched running");
  if(intr_get())
    panic("sched interruptible");

  intena = mycpu()->intena;
  swtch(&p->context, &mycpu()->context);
  mycpu()->intena = intena;
}
```

可以清楚的看到，其同样使用了swtch跳到了mycpu()->context中，这个context，就是scheduler中保存的那个c->context，随后当前进程的内核栈跳到了另外的一个内核栈中，也就是scheduler里，而scheduler会从swtch函数后开始运行，它总是先解锁当前proc的lock！然后才去寻找下一个。



</font>