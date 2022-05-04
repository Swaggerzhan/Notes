<font face="Monaco">

# syscall lab

[lab](https://pdos.csail.mit.edu/6.S081/2021/labs/syscall.html)

## 0x00 trace

实现一个追踪系统调用的系统调用，要求给定第一个参数是一个int，这个参数是系统调用号的(1 << sys_call_number)的关系，后面跟一系列命令。

其用户层的系统调用接口(user/trace.c)已经实现了：

```c
#include "kernel/param.h"
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

int
main(int argc, char *argv[])
{
  int i;
  char *nargv[MAXARG];

  if(argc < 3 || (argv[1][0] < '0' || argv[1][0] > '9')){
    fprintf(2, "Usage: %s mask command\n", argv[0]);
    exit(1);
  }

  if (trace(atoi(argv[1])) < 0) {
    fprintf(2, "%s: trace failed\n", argv[0]);
    exit(1);
  }
  
  for(i = 2; i < argc && i < MAXARG; i++){
    nargv[i-2] = argv[i];
  }
  exec(nargv[0], nargv);
  exit(0);
}
```

通过用户层的trace实现，可以意识到通过对proc结构体添加一个mask位来实现(mask = 1 << sys_call_number)。

在进行trace调用时，将mask设定至proc中，再之后由本进程所fork并exec的所有进程，都将继承这个mask，然后在执行系统调用返回的时候，在内核态进行比对(kernel/syscall.c)：

```c
// kernel/syscall.h
// 其他系统调用号略
#define SYS_trace  23
```

```c
// kernel/syscall.c
// 新增系统调用
extern uint64 sys_trace(void);

// 添加识别名
char* syscalls_name[23] = {"", "fork", "exit", "wait", "pipe", "read", "kill", "exec",
                      "fstat", "chdir", "dup", "getpid", "sbrk", "sleep", "uptime",
                      "open", "write", "mknod", "unlink", "link", "mkdir", "close", "trace"};


void
syscall(void)
{
  int num;
  struct proc *p = myproc();

  num = p->trapframe->a7;
  if(num > 0 && num < NELEM(syscalls) && syscalls[num]) {
    p->trapframe->a0 = syscalls[num]();
    if (p->mask & (1 << num)) { // 如果设定了mask，系统调用号匹配，那么就打印出来
      printf("%d: syscall %s -> %d\n",
        p->pid, syscalls_name[num], p->trapframe->a0
      );
    }
  } else {
    printf("%d %s: unknown sys call %d\n",
            p->pid, p->name, num);
    p->trapframe->a0 = -1;
  }
}
```

实现真正的sys_trace系统调用：

```c
// kernel/sysproc.c
// trace实现
uint64 sys_trace(void) {
  int n;
  if (argint(0, &n) < 0 ) {
    return -1;
  }
  myproc()->mask = n;
  return 0;
}
```

sys_trace只是仅仅做一个记录mask的操作而已，同时需要增加`struct proc{}`一个mask新的字段，还需要在user/user.h系统调用中，增加一个系统调用头文件供用户进程使用，以及在user/usys.pl中新增一个汇编entry以供进入trampoline：

```c
// int arg for mask
// 0 for success, -1 on fail
int trace(int);
```

并且，一个重要的点是，需要在fork系统调用中，实现继承至父进程的mask，否则trace就不起作用了：

```c
// kernel/proc.c
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
    // ...
    // 略...
    // ...
    np->mask = p->mask; // 继承mask
    release(&np->lock);
    return pid;
}
```

测试通过：

```shell
root@VM-4-15-ubuntu:~/xv6-labs-2020# ./grade-lab-syscall trace
== Test trace 32 grep == trace 32 grep: OK (1.1s) 
== Test trace all grep == trace all grep: OK (1.0s) 
== Test trace nothing == trace nothing: OK (1.0s) 
== Test trace children == trace children: OK (11.0s) 
```

## 0x01 sysinfo

给了一个结构体(kernel/sysinfo.h)，需要内核去填满：

```c
// kernel/sysinfo.h
struct sysinfo {
  uint64 freemem;   // amount of free memory (bytes)
  uint64 nproc;     // number of process that state no unused
};
```

freemem字段是当前系统空闲的内存大小，nproc是当前系统中非UNUSED进程数量，系统调用sysinfo接受一个参数，一个指向sysinfo结构体的指针，那么其调用接口大致就是：

```c
int sysinfo(struct sysinfo* info);
```

如同trace一样，需要在user/user.h、user/usys.pl、kernel/syscall.h、kernel/syscall.c中分别添加相应的代码入口。

其实现为：

```c
// kernel/sysproc.c
extern int free_memory();
extern int get_nproc();

uint64 sys_sysinfo(void) {
  uint64 n; // user space sysinfo address
  // 0 for arg 0
  if ( argaddr(0, &n) < 0 ) {
    return -1;
  }
  struct sysinfo kernel_sysinfo;
  // cal result
  kernel_sysinfo.freemem = free_memory();
  kernel_sysinfo.nproc = get_nproc();
  // get user proc 
  struct proc* p = myproc();

  if ( copyout(p->pagetable, n, (char*)&kernel_sysinfo, sizeof(kernel_sysinfo))){
    return -1;
  }
  return 0; 
}
```

其中关于free_memory的实现为计算空闲物理内存，在xv6内核初始化的时候，将从end到PHYSTOP之间的所有地址都列进了kmem中的freelist，由此，我我们可以直接在初始化的时候进行计算即可，每次kalloc都减少一个空闲页，每次kfree都增加一空闲页。

```c
// kernel/kalloc.c
// for free memory record
static int free_chunk = 0;

struct {
  struct spinlock lock;
  struct run *freelist;
} kmem;

void *
kalloc(void)
{
  struct run *r;

  acquire(&kmem.lock);
  r = kmem.freelist;
  if(r){
    kmem.freelist = r->next;
    free_chunk -= 1; // 每次内存被申请后，空闲物理页 - 1
  }
  release(&kmem.lock);

  if(r)
    memset((char*)r, 5, PGSIZE); // fill with junk
  return (void*)r;
}

void
kfree(void *pa)
{
  struct run *r;

  if(((uint64)pa % PGSIZE) != 0 || (char*)pa < end || (uint64)pa >= PHYSTOP)
    panic("kfree");

  // Fill with junk to catch dangling refs.
  memset(pa, 1, PGSIZE);

  r = (struct run*)pa;

  acquire(&kmem.lock);
  r->next = kmem.freelist;
  kmem.freelist = r;
  free_chunk += 1; // 每次kfree后，空闲的物理页 + 1
  release(&kmem.lock);
}

// count the free memory 
int free_memory() {
  int count = 0;
  acquire(&kmem.lock);
  count = 4096 * free_chunk; // 计算就很简单了，直接返回页大小 * 页数量即可
  release(&kmem.lock);
  return count;
}
```

get_nproc的实现在kernel/proc.c中，xv6中，支持的最大进程数量是64，我是直接在proc中进行便利，然后计算非UNUSED的数量，然后返回：

```c
// kernel/proc.c
// add for nproc
int get_nproc() {
  uint64 n = 0;
  for (int i=0; i<NPROC; ++i) {
    acquire(&proc[i].lock);
    if ( proc[i].state != UNUSED ) n++;
    release(&proc[i].lock);
  }
  return n;
}
```

在sysinfo返回的时候，用到了kernel/vm.c::copyout函数，其作用是将内核态的数据拷贝至用户态传递过来的虚拟地址指针上，原理应该会在后面的lab中做到，其使用接口：

```c
// Copy from kernel to user.
// Copy len bytes from src to virtual address dstva in a given page table.
// Return 0 on success, -1 on error.
int
copyout(pagetable_t pagetable, uint64 dstva, char *src, uint64 len);
```

跑一下测试：

```shell
root@VM-4-15-ubuntu:~/xv6-labs-2020# ./grade-lab-syscall 
== Test trace 32 grep == trace 32 grep: OK (1.4s) 
== Test trace all grep == trace all grep: OK (1.0s) 
== Test trace nothing == trace nothing: OK (1.0s) 
== Test trace children == trace children: OK (11.8s) 
== Test sysinfotest == sysinfotest: OK (1.9s) 
Score: 35/35
```






</font>