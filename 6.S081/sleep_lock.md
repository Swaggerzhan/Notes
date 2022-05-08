<font face="Monaco">

# sleep & kill proc 笔记

## 0x00 xv6-riscv 的sleep实现

首先是从uartputc的代码入手：

```c
// kernel/uart.c
void
uartputc(int c)
{
  acquire(&uart_tx_lock);

  if(panicked){
    for(;;)
      ;
  }

  while(1){
    if(((uart_tx_w + 1) % UART_TX_BUF_SIZE) == uart_tx_r){
      // buffer is full.
      // wait for uartstart() to open up space in the buffer.
      sleep(&uart_tx_r, &uart_tx_lock);
    } else {
      uart_tx_buf[uart_tx_w] = c;
      uart_tx_w = (uart_tx_w + 1) % UART_TX_BUF_SIZE;
      uartstart();
      release(&uart_tx_lock);
      return;
    }
  }
}
```

可以看到，sleep函数中，带入了一个锁(uart_tx_lock)，这是因为我们需要访问uart_tx_buf，这有可能其他核心也在访问，所以需要上锁，考虑这样的情况：

```c
release(&uart_tx_lock);
sleep(&uart_tx_r); // 没有锁的sleep实现
acquire(&uart_tx_lock);
```

是否可以替换？答案是否定的，这是因为释放锁和sleep不是同步的，如果发生：

```c
release(&uart_tx_lock);
// <- 中断发生
sleep(&uart_tx_r);
```

那么其他线程很有可能提前调用了wakeup，但是wakeup并没有唤醒任何线程，因为当前线程压根还没开始sleep，这就是wakeup丢失的情况。

具体可以看看wakeup函数，wakeup只是简单的遍历所有的proc，找到SLEEPING的proc，将其设定为RUNNABLE而已：

```c
// kernel/proc.c
// Wake up all processes sleeping on chan.
// Must be called without any p->lock.
void
wakeup(void *chan)
{
  struct proc *p;

  for(p = proc; p < &proc[NPROC]; p++) {
    acquire(&p->lock);
    if(p->state == SLEEPING && p->chan == chan) {
      p->state = RUNNABLE;
    }
    release(&p->lock);
  }
}
```

那么sleep是如何实现原子的释放锁和原子的SLEEP线程？答案是锁接力：

```c
// kernel/proc.c
// Atomically release lock and sleep on chan.
// Reacquires lock when awakened.
void
sleep(void *chan, struct spinlock *lk)
{
  struct proc *p = myproc();
  
  // Must acquire p->lock in order to
  // change p->state and then call sched.
  // Once we hold p->lock, we can be
  // guaranteed that we won't miss any wakeup
  // (wakeup locks p->lock),
  // so it's okay to release lk.
  if(lk != &p->lock){  //DOC: sleeplock0
    acquire(&p->lock);  //DOC: sleeplock1
    release(lk);
  }

  // Go to sleep.
  p->chan = chan;
  p->state = SLEEPING;

  sched();

  // Tidy up.
  p->chan = 0;

  // Reacquire original lock.
  if(lk != &p->lock){
    release(&p->lock);
    acquire(lk);
  }
}
```

在释放提交的锁之前，sleep先锁上当前需要“SLEEP”的进程proc结构体的锁，然后再释放上一个锁，而一旦proc结构体一旦被上锁，即使有wakeup被调用了，wakeup也将阻塞获取锁的地方，直到当前sleep设定完所有操作后释放锁(sched那头会释放当前proc的锁)，这时wakeup线程才能一次性看到一个“SLEEPING”的锁。

注：在内核的自旋锁中会调用push_off关中断防止死锁。

## 0x01 进程的退出

进程通常在调用exit的时候退出，而进程的退出需要内核去回收其各种各样的资源，包括分配的page，打开的文件描述符等os的资源，在exit源码中：

```c
// kernel/proc.c
// Exit the current process.  Does not return.
// An exited process remains in the zombie state
// until its parent calls wait().
void
exit(int status)
{
  struct proc *p = myproc();

  if(p == initproc)
    panic("init exiting");

  // Close all open files.
  for(int fd = 0; fd < NOFILE; fd++){
    if(p->ofile[fd]){
      struct file *f = p->ofile[fd];
      fileclose(f);
      p->ofile[fd] = 0;
    }
  }

  begin_op();
  iput(p->cwd);
  end_op();
  p->cwd = 0;

  // we might re-parent a child to init. we can't be precise about
  // waking up init, since we can't acquire its lock once we've
  // acquired any other proc lock. so wake up init whether that's
  // necessary or not. init may miss this wakeup, but that seems
  // harmless.
  acquire(&initproc->lock);
  wakeup1(initproc);
  release(&initproc->lock);

  // grab a copy of p->parent, to ensure that we unlock the same
  // parent we locked. in case our parent gives us away to init while
  // we're waiting for the parent lock. we may then race with an
  // exiting parent, but the result will be a harmless spurious wakeup
  // to a dead or wrong process; proc structs are never re-allocated
  // as anything else.
  acquire(&p->lock);
  struct proc *original_parent = p->parent;
  release(&p->lock);
  
  // we need the parent's lock in order to wake it up from wait().
  // the parent-then-child rule says we have to lock it first.
  acquire(&original_parent->lock);

  acquire(&p->lock);

  // Give any children to init.
  reparent(p);

  // Parent might be sleeping in wait().
  wakeup1(original_parent);

  p->xstate = status;
  p->state = ZOMBIE;

  release(&original_parent->lock);

  // Jump into the scheduler, never to return.
  sched();
  panic("zombie exit");
}
```

可以看到，exit仅仅清理的当前进程的打开文件，随后就去唤醒父进程以及init进程，如果父进程调用了，wait，那么wakeup1函数就会将其唤醒，同时父进程就回去进行清理，至于为何需要唤醒init进程？这是因为当前父进程可能已经提前子进程退出了，而其子进程将交由init进程收养，届时子进程的退出将由init来做，至于清理进程的东西，在wait里：

```c
// kernel/proc.c
// Wait for a child process to exit and return its pid.
// Return -1 if this process has no children.
int
wait(uint64 addr)
{
  struct proc *np;
  int havekids, pid;
  struct proc *p = myproc();

  // hold p->lock for the whole time to avoid lost
  // wakeups from a child's exit().
  acquire(&p->lock);

  for(;;){
    // Scan through table looking for exited children.
    havekids = 0;
    for(np = proc; np < &proc[NPROC]; np++){
      // this code uses np->parent without holding np->lock.
      // acquiring the lock first would cause a deadlock,
      // since np might be an ancestor, and we already hold p->lock.
      if(np->parent == p){
        // np->parent can't change between the check and the acquire()
        // because only the parent changes it, and we're the parent.
        acquire(&np->lock);
        havekids = 1;
        if(np->state == ZOMBIE){
          // Found one.
          pid = np->pid;
          if(addr != 0 && copyout(p->pagetable, addr, (char *)&np->xstate,
                                  sizeof(np->xstate)) < 0) {
            release(&np->lock);
            release(&p->lock);
            return -1;
          }
          freeproc(np);
          release(&np->lock);
          release(&p->lock);
          return pid;
        }
        release(&np->lock);
      }
    }

    // No point waiting if we don't have any children.
    if(!havekids || p->killed){
      release(&p->lock);
      return -1;
    }
    
    // Wait for a child to exit.
    sleep(p, &p->lock);  //DOC: wait-sleep
  }
}
```

可以看到，wait里找到子进程中处于ZOMBIE的进程，然后将其内存页表回收，同时如果可以，那就将子进程的退出状态拷贝到父进程的用户态中(wait的返回值)，并且，在访问子进程的parent时候，并没有上锁，这是xv6的代码逻辑决定的，即只有父进程可修改子进程的parent字段，这也是为什么可以直接访问了。

如果没有找到对应的ZOMBIE进程，那么父进程就进入睡眠，并等待wakeup1来唤醒它。

## 0x02 kill 实现

kill实现并没有我想象中的那么粗暴，在xv6中，仅仅通过设定proc的killed字段来进行标记，在被杀死的进程进入内核后，总是会检测，而如果被杀死的进程已经拿到了一些lock，代码逻辑中也没有体现出直接杀死的方式，代码中总是在没有lock的时候进行p->killed检测，如果进程被杀，那就进入退出的阶段。

```c
// kernel/proc.c
// Kill the process with the given pid.
// The victim won't exit until it tries to return
// to user space (see usertrap() in trap.c).
int
kill(int pid)
{
  struct proc *p;

  for(p = proc; p < &proc[NPROC]; p++){
    acquire(&p->lock);
    if(p->pid == pid){
      p->killed = 1;
      if(p->state == SLEEPING){
        // Wake process from sleep().
        p->state = RUNNABLE;
      }
      release(&p->lock);
      return 0;
    }
    release(&p->lock);
  }
  return -1;
}
```

这里在exit中还有一个小小的操作：

```c
// kernel/proc.c :: exit()
// we need the parent's lock in order to wake it up from wait().
// the parent-then-child rule says we have to lock it first.
acquire(&original_parent->lock);

acquire(&p->lock);

// Give any children to init.
reparent(p);

// Parent might be sleeping in wait().
wakeup1(original_parent);
```

就是在退出当前进程的时候，调用了reparent函数：

```c
// kernel/proc.c
// Pass p's abandoned children to init.
// Caller must hold p->lock.
void
reparent(struct proc *p)
{
  struct proc *pp;

  for(pp = proc; pp < &proc[NPROC]; pp++){
    // this code uses pp->parent without holding pp->lock.
    // acquiring the lock first could cause a deadlock
    // if pp or a child of pp were also in exit()
    // and about to try to lock p.
    if(pp->parent == p){
      // pp->parent can't change between the check and the acquire()
      // because only the parent changes it, and we're the parent.
      acquire(&pp->lock);
      pp->parent = initproc;
      // we should wake up init here, but that would require
      // initproc->lock, which would be a deadlock, since we hold
      // the lock on one of init's children (pp). this is why
      // exit() always wakes init (before acquiring any locks).
      release(&pp->lock);
    }
  }
}
```

其做法也非常的简单，将名下所有的子进程，都交由init进程收养，然后当前进程才退出，至于当前进程由谁来收尸，就要看当前进程的父进程是否还活着了，如果没有，想必当时父进程退出的时候也将当前进程交由init进程收养了吧，init进程仅仅只是个无情的收尸机器。

</font>