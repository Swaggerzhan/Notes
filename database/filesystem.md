<font face="Monaco">

# File System

## 0x00 MetaData

文件中存储了信息，而文件本身也有一些其他需要被存储的信息，比如一个文件的大小，文件的权限，创建修改时间等等各种属性，其存储这些属性的“文件”称为元数据(MetaData)，在Linux中，也就是Inode节点了。

> 磁盘Block

首先是Sector，这是硬件概念，一般现代出厂磁盘都是4K对其的，也就是一个Sector即4K，Block是对应文件系统的概念，表示的是文件的最小单位，一般而言，一个Block对应一个Sector，也就是4K。

Block的大小会影响存储的效率，如果Block太大，那么占不满一个Block的文件也将消耗整个Block的空间，如果Block太小，那么反之Inode中索引就会占用过多。

> Inode

Linux中使用Inode来存储元数据，这里只讨论文件系统的存储方式(先抛开其权限)，Inode中存储着文件的真正所在地，也就Block，通过Inode，我们可以找到真正的文件内容。

而Inode本身也是属于一种文件数据，它也会被存储在磁盘上，大小则看文件系统的实现，有128bytes，也有256bytes。Inode是存储在一个数组中的，这个数组是固定的，当这个Inode数组被用光，那么也就视为磁盘写满了(即使Block还有剩余)。

可以使用df -i查看当前系统挂载的磁盘还有多少可用以及剩余的Inode。

```shell
root@VM-16-10-ubuntu:~# df -i
Filesystem      Inodes  IUsed   IFree IUse% Mounted on
udev            479934    387  479547    1% /dev
tmpfs           484407   2095  482312    1% /run
/dev/vda1      5242880 124725 5118155    3% /
tmpfs           484407     13  484394    1% /dev/shm
tmpfs           484407      4  484403    1% /run/lock
tmpfs           484407     18  484389    1% /sys/fs/cgroup
tmpfs           484407     11  484396    1% /run/user/0
```

> 如何读取文件？

一般的文件系统中都会存储一个Map，其内容是关于文件名映射到Inode索引的Map。

假设有一文件名为"/tmp/tmp.file"，其映射Inode为1001，那么将直接到Inode数组中读取其元数据内容，进而读取到文件的整个内容(Inode中有权限信息，没有权限会被堵在这个位置)。

也可以用ls指定-i来读取Inode。

```shell
root@VM-16-10-ubuntu:~# ls -il
407599 -rw-r--r--  1 root root  571 Jan 10 19:17 test.txt
```

## 0x01 Block使用情况

通过上述方法，我们有了一种存储文件的方式，寻找文件也非常迅速，但现在有另外一个文件，如何知道哪些块是可以用的？如何知道哪些Inode数组索引处是可用的？

> i-bitmap和d-bitmap

通过bitmap来存储这种“哪个Block”是否被使用是合理的，0表示空闲，1表示使用，其中i-bitmap表示的是Inode位图，而d-bitmap则表示的是“data”，即数据块的使用位图，通过这些位图，我们可以明确哪些Block被使用了，哪些Block是空闲的。

> Super

即使有了上述的存储方式，还需要一个统领全部文件系统的信息，也是MetaData的一种，可以理解为整个文件系统的“MetaData”，其用来区分Inode数组、数据块、Inode位图、数据块位图在磁盘中的位置。

## 0x02 跟踪文件的位置

我们知道，Inode的大小是固定的，可能是128bytes，可能是256bytes，但 __文件的大小是不固定的，如果一个文件非常的大，而Inode空间是固定的，那么所能够容纳的BlockID的大小也是有上限的，那如何存储一个巨大的文件呢__ ？

> multi-level index

比较容易想到的是和多级页表类似的实现，也就是套娃。

通过Inode中存储一个单独的Block，在这个特殊的Block中，我们可以都用来存储BlockID，如果不够，可以继续套娃，直到够用为止，这也就是multi-level index的实现思路，通过类似C/C++中的指针方式来实现。

> 文件目录结构

// TODO: wait for update...

</font>