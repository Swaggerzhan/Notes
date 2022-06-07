<font face="Monaco">

# xv6-riscv file system 笔记

## 0x00 磁盘的布局layout

![](./pic/disk_layout.png)

其中boot块是进入系统的一些指令，super块是文件系统的起始位置，其代码：

```c
// kernel/fs.h
#define ROOTINO  1   // root i-number
#define BSIZE 1024  // block size

// mkfs computes the super block and builds an initial file system. The
// super block describes the disk layout:
struct superblock {
  uint magic;        // Must be FSMAGIC
  uint size;         // Size of file system image (blocks)
  uint nblocks;      // Number of data blocks
  uint ninodes;      // Number of inodes.
  uint nlog;         // Number of log blocks
  uint logstart;     // Block number of first log block
  uint inodestart;   // Block number of first inode block
  uint bmapstart;    // Block number of first free map block
};
```

xv6磁盘中，每个块大小为1024bytes，root块的inode值为1，super block中便记录了这些inode的起始位置，当前的inode数量，以及log、bitmap等。

之后便是inode的结构体：

```c
// kernel/fs.h
#define NDIRECT 12
#define NINDIRECT (BSIZE / sizeof(uint))
#define MAXFILE (NDIRECT + NINDIRECT)

// On-disk inode structure
struct dinode {
  short type;           // File type
  short major;          // Major device number (T_DEVICE only)
  short minor;          // Minor device number (T_DEVICE only)
  short nlink;          // Number of links to inode in file system
  uint size;            // Size of file (bytes)
  uint addrs[NDIRECT+1];   // Data block addresses
};
```

dinode可以视作真正文件的meta数据，type表明当前文件类型，nlink表示当前被链接了多少次，size则是文件大小，addrs中前NDIRECT是一个直接的地址，或者说是文件数据所在的block id，最后一个是一个间接的地址，它所指向一个block，block中并不是保存file的数据，而是新的"addrs",或者可以比喻为“新申请一个block，全部用来存放指针，这些指针才指向真正的数据块”。

所以就可以得出当前文件系统支持的最大文件大小：(12 + (1024 / 4)) * 1024 = 268 KB，并且，可以算出dinode结构体所占空间为：(2 * 4) + 4 + (4 * 13) = 64bytes。

磁盘中的dinode和内存中的inode结构体有所不同。

### 目录

在xv6-riscv的文件系统中，目录同样是文件，只不过其内容是一个个entry，每个entry的结构体为：

```c
struct dirent {
  ushort inum;       // 2
  char name[DIRSIZ]; // 14
};
```

每个entry在文件中也仅仅是以类似数组的方式排列罢了，所以在目录中搜索文件的效率可能不会很高，但这是xv6的实现，真实系统可能会更加的复杂，效率也会更高。

## 0x01 icache

接下去就可以看上层一点的icache了，icache作为将磁盘中数据读取到内存中缓存的结构，其定义为：

```c
// kernel/bio.c
struct {
  struct spinlock lock;
  struct buf buf[NBUF]; // 30

  // Linked list of all buffers, through prev/next.
  // Sorted by how recently the buffer was used.
  // head.next is most recent, head.prev is least.
  struct buf head;
} bcache;
```

其中保存的buf结构定义为：

```c
// kernel/buf.h
struct buf {
  int valid;   // has data been read from disk?
  int disk;    // does disk "own" buf?
  uint dev;
  uint blockno;
  struct sleeplock lock;
  uint refcnt;
  struct buf *prev; // LRU cache list
  struct buf *next;
  uchar data[BSIZE]; // 1024
};
```

其初始化的时候，将所有的buf都作为双向链表链接起来，用于LRU使用：

```c
// kernel/bio.c
void
binit(void)
{
  struct buf *b;

  initlock(&bcache.lock, "bcache");

  // Create linked list of buffers
  bcache.head.prev = &bcache.head;
  bcache.head.next = &bcache.head;
  for(b = bcache.buf; b < bcache.buf+NBUF; b++){
    b->next = bcache.head.next;
    b->prev = &bcache.head;
    initsleeplock(&b->lock, "buffer");
    bcache.head.next->prev = b;
    bcache.head.next = b;
  }
}
```

其使用的接口有几个，比如bget，它的源码为：

```c
// kernel/bio.c
// Look through buffer cache for block on device dev.
// If not found, allocate a buffer.
// In either case, return locked buffer.
static struct buf*
bget(uint dev, uint blockno)
{
  struct buf *b;

  acquire(&bcache.lock);

  // Is the block already cached?
  for(b = bcache.head.next; b != &bcache.head; b = b->next){
    if(b->dev == dev && b->blockno == blockno){
      b->refcnt++;
      release(&bcache.lock);
      acquiresleep(&b->lock);
      return b;
    }
  }

  // Not cached.
  // Recycle the least recently used (LRU) unused buffer.
  for(b = bcache.head.prev; b != &bcache.head; b = b->prev){
    if(b->refcnt == 0) {
      b->dev = dev;
      b->blockno = blockno;
      b->valid = 0;
      b->refcnt = 1;
      release(&bcache.lock);
      acquiresleep(&b->lock);
      return b;
    }
  }
  panic("bget: no buffers");
}
```

bget首先查看是否已经缓存在bcahce中了，如果有，只是简单的增加refcnt，然后返回，否则才会去寻找一个空闲的buffer块，然后找到这个空闲的buf块，上锁，返回。

__这里bget仅仅只是找到一个buf，至于这个buf是不是真的缓存了磁盘中的数据，需要由其中的valid字段来去确定，可以查看bread代码，如果这个buf没有缓存数据，那就需要virtio_disk_rw去获取这个缓存__。

```c
// kernel/bio.c
// Return a locked buf with the contents of the indicated block.
struct buf*
bread(uint dev, uint blockno)
{
  struct buf *b;

  b = bget(dev, blockno);
  if(!b->valid) {
    virtio_disk_rw(b, 0);
    b->valid = 1;
  }
  return b;
}
```

和bget相反的是brelse，它释放锁，然后查看是否还有refcnt，如果已经没有refcnt了，那么证明已经没有人再去使用这个东西了，不过由于采用了LRU，即使没有人使用了，还是需要将其放到链表的头部(bget找空闲的缓存块会从尾往头找)。

```c
// kernel/bio.c
// Release a locked buffer.
// Move to the head of the most-recently-used list.
void
brelse(struct buf *b)
{
  if(!holdingsleep(&b->lock))
    panic("brelse");

  releasesleep(&b->lock);

  acquire(&bcache.lock);
  b->refcnt--;
  if (b->refcnt == 0) {
    // no one is waiting for it.
    b->next->prev = b->prev;
    b->prev->next = b->next;
    b->next = bcache.head.next;
    b->prev = &bcache.head;
    bcache.head.next->prev = b;
    bcache.head.next = b;
  }
  
  release(&bcache.lock);
}
```

## 0x02 log layer

在xv6-riscv中log层也是有点意思的，先看在superblock中对log区域的划分：

```c
struct superblock {
  uint nlog;         // Number of log blocks
  uint logstart;     // Block number of first log block
};
```

logstart标明了log在磁盘中的其实block id，nlog则标明了大小，可以看一下关于logheader和log的结构体定义：

```c
// kernel/log.c
// Contents of the header block, used for both the on-disk header block
// and to keep track in memory of logged block# before commit.
struct logheader {
  int n;
  int block[LOGSIZE]; // MAXOPBLOCKS * 3    MAXOPBLOCKS = 10
};

struct log {
  struct spinlock lock;
  int start;
  int size;
  int outstanding; // how many FS sys calls are executing.
  int committing;  // in commit(), please wait.
  int dev;
  struct logheader lh;
};
```

这个logheader中的n表示的是当前log block中有几个被使用了，或者说，当前是否有需要被提交的log，后面跟随的数组大小是30，__而回过头来看xv6的磁盘分配中，你就会发现log有31个block，第一个block所存储的，就是logheader，之后的30个log block都是拷贝，在一个数据block被写入之前，都会写到log block中，然后在去logheader.block中标明这里的“数据”，该被写到哪个地方__。

例如：logheader中n为1，则标明有一个commit的数据，那么我们肯定可以在block[0]处找到一个block id，这个block id所指向的，才是真正数据该放入的地方，不过在将数据直接写入到这个block之前，我们将其放在了log block的0处，这是一块完整的数据，只需要将其复制到目标区域，然后将n改为0即可完成本次写入操作。

在文件系统启动的时候，recover_from_log会被调用，这是一个关于恢复文件系统的函数，其中调用了read_head：

```c
// kernel/log.c
static void
recover_from_log(void)
{
  read_head();
  install_trans(); // if committed, copy from log to disk
  log.lh.n = 0;
  write_head(); // clear the log
}

// Read the log header from disk into the in-memory log header
static void
read_head(void)
{
  struct buf *buf = bread(log.dev, log.start);
  struct logheader *lh = (struct logheader *) (buf->data);
  int i;
  log.lh.n = lh->n;
  for (i = 0; i < log.lh.n; i++) {
    log.lh.block[i] = lh->block[i];
  }
  brelse(buf);
}
```

可以看到，read_head的作用是将log.start开始的这个block从磁盘中读取，然后将其看成logheader，在写入到已经在内存中的log.lh中。

然后install_trans被调用：

```c
// kernel/log.c
// Copy committed blocks from log to their home location
static void
install_trans(void)
{
  int tail;

  for (tail = 0; tail < log.lh.n; tail++) {
    struct buf *lbuf = bread(log.dev, log.start+tail+1); // read log block
    struct buf *dbuf = bread(log.dev, log.lh.block[tail]); // read dst
    memmove(dbuf->data, lbuf->data, BSIZE);  // copy block to dst
    bwrite(dbuf);  // write dst to disk
    bunpin(dbuf);
    brelse(lbuf);
    brelse(dbuf);
  }
}
```

install_trans从log block中进行数据读取(log.start + 1为了跳过第一个log header块)，然后读“需要写入的block id”，随后进行内存拷贝，再调用bwrite将内存刷入到真正的磁盘位置，当前块就完成了。

在完成install_trans安装后，将log.lh.n置为0，代表完成commit，但这里还仅仅是在内存中而已，随后的write_head才是将真正的数据刷入到log磁盘中。

在此过程中，不论在哪个地方发生crash，在下一次的文件系统重启过程中，recover_from_log总会被再次调用，而磁盘中的“事务”还是会重新的在做一次，使得数据总是我们所期望的那样。

### 在真正写入前，先写到磁盘中的log block位置

log在文件系统中的真正使用流程：

> 1. log_write，将需要更新的数据写到log block中，写logheader.block数组表明数据被提交时的位置。

> 2. commit，更新logheader.n。

> 3. install，或者说apply，将log block中的数据复制到真正的磁盘位置。

> 4. clean，清理log区域的块。

如果crash发生在1和2之间，那么数据将丢失，如果crash发生在2和3之间，那么很显然，在下一次文件系统重启过程中，commit会重新提交，在3和4之间也是如此，这些保证，使得文件系统总是可以正常的运行。

现在，回到code来看一下log到底提供了何种API来保证文件系统的正确性的，xv6的安全写入方式是这样的：

> 1. begin_op()

> 2. log_write()

> 3. end_op()

带着这个API，让我们回看log结构，它是重中之重：

```c
struct log {
  struct spinlock lock;
  int start;
  int size;
  int outstanding; // 当前有多少个“事务”
  int committing;  // 是否在提交中
  int dev;
  struct logheader lh;
};
```

每次的磁盘写入操作时，总是需要以begin_op开始，它标记着磁盘事务的开始：

```c
// kernel/log.c
// called at the start of each FS system call.
void
begin_op(void)
{
  acquire(&log.lock);
  while(1){
    if(log.committing){
      sleep(&log, &log.lock);
    } else if(log.lh.n + (log.outstanding+1)*MAXOPBLOCKS > LOGSIZE){
      // this op might exhaust log space; wait for commit.
      sleep(&log, &log.lock);
    } else {
      log.outstanding += 1;
      release(&log.lock);
      break;
    }
  }
}
```

begin_op有着一些硬性的规定，比如说至多只能3个事务同时运行、事务正在提交中只能等待，一旦begin_op开始后，我们就可以通过log_write来写入日志了：

```c
// kernel/log.c
// Caller has modified b->data and is done with the buffer.
// Record the block number and pin in the cache by increasing refcnt.
// commit()/write_log() will do the disk write.
//
// log_write() replaces bwrite(); a typical use is:
//   bp = bread(...)
//   modify bp->data[]
//   log_write(bp)
//   brelse(bp)
void
log_write(struct buf *b)
{
  int i;

  if (log.lh.n >= LOGSIZE || log.lh.n >= log.size - 1)
    panic("too big a transaction");
  if (log.outstanding < 1)
    panic("log_write outside of trans");

  acquire(&log.lock);
  for (i = 0; i < log.lh.n; i++) {
    if (log.lh.block[i] == b->blockno)   // log absorbtion
      break;
  }
  log.lh.block[i] = b->blockno;
  if (i == log.lh.n) {  // Add new block to log?
    bpin(b);
    log.lh.n++;
  }
  release(&log.lock);
}
```

log_write只写了log结构体，这是在哪里？只是在内存中而已，它的作用是标记了某个内存中的cache不可被evict，同时在log结构体中记录了一个“在内存中的缓存”。

而一旦写入完成后，end_op就可以被调用了：

```c
// kernel/log.c
// called at the end of each FS system call.
// commits if this was the last outstanding operation.
void
end_op(void)
{
  int do_commit = 0;

  acquire(&log.lock);
  log.outstanding -= 1;
  if(log.committing)
    panic("log.committing");
  if(log.outstanding == 0){
    do_commit = 1;
    log.committing = 1;
  } else {
    // begin_op() may be waiting for log space,
    // and decrementing log.outstanding has decreased
    // the amount of reserved space.
    wakeup(&log);
  }
  release(&log.lock);

  if(do_commit){
    // call commit w/o holding locks, since not allowed
    // to sleep with locks.
    commit();
    acquire(&log.lock);
    log.committing = 0;
    wakeup(&log);
    release(&log.lock);
  }
}
```

end_op做了什么？他表示一个事务结束了，如果本轮中所有事务都结束了(group commit)，也就是outstanding为0，那么久可以做提交操作了，即，调用commit：

```c
// kernel/log.c
static void
commit()
{
  if (log.lh.n > 0) {
    write_log();     // Write modified blocks from cache to log
    write_head();    // Write header to disk -- the real commit
    install_trans(); // Now install writes to home locations
    log.lh.n = 0;
    write_head();    // Erase the transaction from the log
  }
}
```

write_log()将内存中的数据写入到log block处的磁盘中，write_head才真正更新了logheader，从这里起，才是真正的提交操作，当write_head()运行完毕后，所有的提交操作都将变得不可丢弃了。

但如果没有install_trans，那么其写入操作其实是没有完整的，之后的再次write_head其实就是清空log block而已。

</font>