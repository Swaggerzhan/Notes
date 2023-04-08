# Ubuntu22.04 environment

## 0x00 pre

编译器和glibc以及动态链接库的问题，确实是我时常遇到的问题，有时候经常把环境搞炸也是这些东西没有配置好，这里记录一下如何配置这些东西。

#### gcc

gcc即编译器，更新gcc版本时，需要确定glibc的版本是否能够支持，否则会出现问题。

#### glibc

glibc Linux下最基础的系统API wrapper，基本所有程序都是基于这个库的，修改libc.so.6需要非常小心，一些操作：

```shell
# 查看版本
ldd --version 
```
#### ld
动态链接加载器实际上应该是Linux自带的，只要ELF文件是需要链接.so库的，那么都需要将这个动态链接加载器“链”进ELF，这个.so即：`ld-linux-x86-64.so.2`，其作用则是帮忙寻找ELF文件的.so依赖，然后加载即可，寻找路径：

```shell
$LD_LIBRARY_PATH
/etc/ld.so.conf
/lib
/usr/lib
```

其中/etc/ld.so.conf中给定的是一些目录，链接器会将这里面的所有.so都加载到缓存中，如果修改了这些配置，一般需要更新缓存：

```shell
ldconfig # update cache
ldconfig -p # peek cache
```

## 0x00 gcc12

在ubuntu12下搭建一个可以使用的C++开发环境，但是又不至于破坏原先的系统环境，首先是gcc12的基础，有了这个，glibc版本对应的也会更高一些。

下载对应的gcc12源码和对应的依赖：
```shell
wget https://ftp.gnu.org/gnu/gcc/gcc-12.2.0/gcc-12.2.0.tar.gz
./contrib/download_prerequisites
```
选择需要安装的位置，这里我习惯和系统的分开，故放到${pwd}/compiler下：
```shell
./configure --prefix=${pwd}/compiler/gcc12 --with-system-zlib --disable-multilib
make -j8
sudo make install
```

## 0x01 tcmalloc

[tcmalloc下载](https://github.com/gperftools/gperftools/releases)
```shell
./autogen.sh
mkdir build
cd build
../configure --enable-cpu-profiler --enable-heap-profiler --enable-heap-checker --enable-debugalloc
make -j8
```
至于执不执行make install就看你需不需了。

## 0x02 protobuf

和tcmalloc类似即可。
TODO:


