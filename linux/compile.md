# linux 2.6.34 内核编译 & qemu调试(x86)

## 0x00 qemu安装

直接下载源码编译即可，可以选择4.0版本的，比较新的版本可能会需要python环境高于3.8

```bash
wget https://download.qemu.org/qemu-4.2.0.tar.xz 
tar xvJf qemu-4.2.0.tar.xz
cd qemu-4.2.0
./configure --target-list=x86_64-softmmu,i386-softmmu
make -j8
make install
```

## 0x01 内核编译

### gcc
2.6.34内核比较老了，最好选个老一点的gcc来搞，比如gcc4，ubuntu/debian可以直接这样下载和切换：

```bash
sudo apt-get install gcc-4.8 g++-4.8
sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-4.8 100
sudo update-alternatives --config gcc
```

### 内核准备
直接用清华大学源下更快，然后切到2.6.34即可：

```bash
# https://mirrors.tuna.tsinghua.edu.cn/help/linux.git/
git clone https://mirrors.tuna.tsinghua.edu.cn/git/linux.git
git checkout v2.6.34
```

### 编译

直接menuconfig开始编译，退出后直接make即可

```bash
make menuconfig
make -j8
```

可能会有一些错误，比如：
##### gcc: error: elf_x86_64: No such file or directory

```
vim ./arch/x86/vdso/Makefile
```

把其中的-m elf-x86_64改成-m64，-m elf_i386改成-m32即可。

##### defined(@arg)
直接去掉中间的参数即可

##### drivers/net/igbvf/igbvf.h:129:15: error: duplicate member 'page'

```bash
vim ./drivers/net/igbvf/igbvf.h
```
把后面的129行的page注释掉即可

## 0x02 qemu启动内核

### busybox

```bash
# https://www.busybox.net/downloads/
wget https://www.busybox.net/downloads/busybox-1.36.1.tar.bz2
tar -xvjf ./busybox-1.36.1.tar.bz2
make menuconfig

# 把这个选中
# --- Build Options                                                                           
#   [ ] Build static binary (no shared libs) (NEW)
  
make
```

### 制作rootfs

```bash
dd if=/dev/zero of=rootfs.img bs=1M count=10
mkfs.ext4 rootfs.img
```
随便创个fs，然后将img挂上去，进到busybox编译目录，把内容编译到这个rootfs中：
```bash
mkdir fs
sudo mount -t ext4 -o loop rootfs.img ./fs
sudo make install CONFIG_PREFIX=./fs
```

在到这个rootfs中创建一些必须的目录、拷贝一下busybox中例子的etc：
```bash
sudo mkdir proc dev etc home mnt
sudo cp -r ../examples/bootfloppy/etc/* etc/
# cd .. && sudo chmod -R 777 fs/ 
```
最后卸载了：
```bash
sudo umount fs
```

##### ext4错误问题

如果遇到ext4的挂载问题，可以尝试改成ext2即可

##### kernel too old
这是个ABI兼容问题，busybox编译使用的glibc版本太高，其中又调了系统调用就有可能引发这个问题，可以直接下个现成的，替换掉rootfs中，bin/busybox即可，实测下面这个可以在2.6.34 kernel中使用

```bash
wget https://busybox.net/downloads/binaries/1.21.1/busybox-x86_64
```

### qemu启动

```bash
qemu-system-x86_64 -kernel ./arch/x86_64/boot/bzImage -hda ./rootfs.img -append "root=/dev/sda console=ttyS0" -nographic
```

如果遇到需要退出的，可以使用：
```
ctrl+a, c
```

## 0x03 qemu 调试内核相关

## 0x04 网络相关





## end
参考：
https://www.cnblogs.com/QiQi-Robotics/p/15229668.html
https://stackoverflow.com/questions/10772319/how-to-solve-drivers-net-igbvf-igbvf-h12915-error-duplicate-member-page
https://kerneltravel.net/blog/2021/debug_kernel_szp/
