<font face="Monaco">

# 6.S081 实验环境安装

OS: ubuntu20.04 LTS

## 0x00 安装qemu之前的环境检查

是否有libgcc.a库

```shell
gcc -m32 -print-libgcc-file-name
# 如果没有，则安装：
sudo apt install build-essential
```

如果是64bit操作系统，那就安装32bit库支持：

```shell
sudo apt install gcc-multilib
```

## 0x01 qemu 安装和依赖

拉取qemu：

```shell
git clone https://github.com/mit-pdos/6.828-qemu.git qemu
```

或者也可以直接到[官网:https://www.qemu.org/download/](https://www.qemu.org/download/)进行下载，这里最好下5.1版本和之前的，不然可能会出现make qemu后卡住的问题。

如果出现：

```shell
Disabling libtool due to broken toolchain support
# 则
sudo apt install libtool-bin
```

```shell
ERROR: glib-2.12 gthread-2.0 is required to compile QEMU
# 则
sudo apt install libglib2.0-dev
```

关于pixman-1的依赖安装：

```shell
sudo apt install libpixman-1-dev libcairo2-dev libpango1.0-dev libjpeg8-dev libgif-dev
```

关于ninja依赖的安装

```shell
sudo apt install ninja-build
```

configure

```shell
./configure --disable-kvm --disable-werror --target-list="i386-softmmu x86_64-softmmu,riscv64-softmmu,riscv64-linux-user"
```

试试是否成功安装了：

```shell
qemu-system-riscv64 --version
```

## 0x02 RISC-V toolchain 安装和依赖

xv6属于是精简指令集操作系统，所以RISC-V toolchain也是运行xv6依赖的一环。

```shell
git clone --recursive https://github.com/riscv/riscv-gnu-toolchain
./configure --prefix=/usr/local/opt/riscv-gnu-toolchain
make
```

其中整个项目是非常大的，需要很长的时间，并且需要拉取其子模块，所以 `--recursive` 是必要的。

如果出现依赖错误：

makeinfo：

```shell
sudo apt install texinfo
```

bison：

```shell
sudo apt install bison
```

flex

```shell
sudo apt install flex
```

除此之外，还有以下依赖：

```shell
sudo apt install qemu-system-misc gcc-riscv64-linux-gnu binutils-riscv64-linux-gnu
```

最后添加其至环境变量：

```shell
# 写入到~/.bashrc中
export PAHT="$PATH:/usr/local/opt/riscv-gnu-toolchain/bin"
source ~/.bashrc
# 查看是否成功安装
riscv64-unknown-elf-gcc -v
```

## 0x03 运行xv6和调试

安装环境检测：

```shell
qemu-system-riscv64 --version
riscv64-linux-gnu-gcc --version
riscv64-unknown-elf-gcc --version
riscv64-unknown-linux-gnu-gcc --version
```

```shell
git clone git://g.csail.mit.edu/xv6-labs-2021
cd xv6-riscv
make
make qemu
```

调试需先运行gdbserver

```shell
make qemu-gdb
```

然后再去链接：

```shell
riscv64-unknown-elf-gdb kernel/kernel
target remote IP:port
```


## Ref

[https://github.com/mit-pdos/xv6-riscv/issues/7](https://github.com/mit-pdos/xv6-riscv/issues/7)

[https://zhayujie.com/mit6828-env.html](https://zhayujie.com/mit6828-env.html)

[https://blog.csdn.net/aiyimo_/article/details/78655372](https://blog.csdn.net/aiyimo_/article/details/78655372)

[https://zhuanlan.zhihu.com/p/58143429](https://zhuanlan.zhihu.com/p/58143429)

[pixman-1依赖: https://github.com/Automattic/node-canvas/issues/1065](https://github.com/Automattic/node-canvas/issues/1065)

</font>