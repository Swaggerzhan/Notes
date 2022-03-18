<font face="Monaco">

# brpc和braft编译

brpc: v0.97

protobuf: v3.19.3

braft: v1.1.2

os: ubuntu20.04

g++: 9.4.0

## 0x00 protobuf

直接到github拉下来即可，我这里使用的是v3.19.3的版本。

```shell
./autogen.sh
./configure
# 如果要链zlib
./configure --with-zlib # 不链zlib的话，后面编译braft可能会有问题
```

可能需要一些依赖，比如：

```shell
sudo apt install automake autoconf libtool
```

安装完如果protoc的lib没有被链到环境变量中的话，可以：

```shell
sudo ldconfig
```

## 0x01 brpc

需要先安装一些依赖，在ubuntu下，需要ssl库、gflags库、leveldb等。

```shell
sudo apt install libssl-dev
sudo apt install libgflags-dev
sudo apt install libleveldb-dev
```

然后直接运行官方给的sh脚本

```shell
# sh config_brpc.sh --headers="dir1 dir2 ..." --libs="dir1 dir2 ..."
sh config_brpc.sh --headers="/usr/include /usr/local/include" --libs="/usr/lib /usr/local/lib"
make
```

### error

有时候，编译的时候会出现zlib依赖的错误，这是因为编译protobuf的时候没有选上，在编译protobuf的configure的时候可以加上：

```shell
./configure --with-zlib # for protobuf
```

安装zlib的依赖可以：

```shell
sudo apt install zlib1g-dev # 开发包
sudo apt install zlib1g     # 使用
```

或者直接编译安装都可以。


## 0x02 braft

同样直接到官方git拉下来即可，这里使用的是v1.1.2版本。

直接在目录中创建出build目录，然后用cmake构建即可：

```shell
mkdir build
cd build
cmake ..
make
```

### error

brpc库找不到，如果brpc没有安到公共目录，就需要指定目录了，或者也可以直接修改CMakeLists.txt中指令：

```cmake
find_path(BRPC_INCLUDE_PATH NAMES brpc/server.h)
find_library(BRPC_LIB NAMES libbrpc.a brpc)
```

直接改成我们的brpc编译目录即可：

```cmake
set(BRPC_INCLUDE_PATH /your_brpc_path/output/include)

# 静态库可能还需要手动添加一些依赖的静库
set(BRPC_LIB /your_brpc_path/output/lib/libbrpc.a) 
# 或者链动态库，两者取其一即可
set(BRPC_LIB /your_brpc_path/output/lib/libbrpc.so)
```

## 0x03 ref

[https://github.com/apache/incubator-brpc/blob/master/docs/cn/getting_started.md](https://github.com/apache/incubator-brpc/blob/master/docs/cn/getting_started.md)

[https://github.com/baidu/braft/tree/v1.1.2](https://github.com/baidu/braft/tree/v1.1.2)

</font>