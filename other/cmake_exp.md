# CMAKE

### 1.基本语法

在`CMakeLists.txt`中，我们可以定义一个最低的版本要求，生成的二进制名，也就是整个项目的名字，比如:

```cmake
cmake_minimum_required(VERSION 3.10) # 最低要求3.10版本的cmake
project(MyProject) # 项目名，也就是最终生成二进制时的名字
```

通过`set`指令，我们可以设定一些cmake变量，也可以设定一些cmake的内置变量

```cmake
set(CMAKE_BUILD_TYPE DEBUG or RELEASE) # 构建DEBUG或者是RELEASE版本
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11") # 添加一些flags

set(MY_VALUE xxxx) # 设定变量名
```

 添加源文件到项目中时，我们可以使用`add_executable`。例如:

```cmake
add_executable(MyProject
  A.h A.cpp
  B.h B.cpp
  ${MY_VALUE}
) # 表示将其中的文件都添加到二进制MyProject中去
```

链接库


```cmake

target_link_libraries(MyProject test) # 链接libtest.so
# or 链接静态的
target_link_libraries(MyProject libtest.a) # 即直接指定为静态
# 设定一些寻找这些库文件的路径：
link_directories(/mylib_path/lib)	

# 或者设定一些头文件的路径
include_directories(/tmp/include) 	
```

有时候，库的区域比较疏散，需要进行寻找，也可以使用find_path来寻找，不过即使如此，有时也会出现无法找到的情况(手动编译or头库等文件不在系统默认路径上)，这时再通过命令指定路径让cmake寻找，可以比较方便的编写cmake程序：

```cmake
find_path(<PATH_THAT_CMAKE_FOUND> file_name [path1, path2] )
# 比如

# 将存储leveldb/db.h的路径保存至LEVELDB_INCLUDE_PAHT
find_path(LEVELDB_INCLUDE_PAHT leveldb/db.h)
```



### 2. 生成库文件


```cmake
set(REAUIRED_SOURCE_CODE test.h test.cc ...)
set(LIB_NAME thisislib)

add_library(
    ${LIB_NAME} # 库名
    STATIC | SHARED # 静态 or 动态
    ${REQURIED_SOURCE_CODE}
)
```


### 安装库到指定文件夹中

```cmake
set(INSTALL_HEAD_FILES 需要安装的头文件)
# 库
install(
    TARGETS ${LIB_NAME}
    ARCHIEVE SAHRED DESTINATION ${CMAKE_INSTALL_PREFIX}/lib 
)
# 头文件
install(
    FILES ${INSTALL_HEAD_FILES} DESTINATION ${CMAKE_INSTALL_PREFIX}/include
)
```
如果需要同时生成静态库和动态库，则需要做两个add_library，并且，静态库在进行编译时，如果所依赖的库也是静态库，cmake是不会将其拷贝到一起的，需要使用ar命令将静态库合并。

比如：

项目会生成一个libB.a静态库，而libB.a依赖于libA.a静态库，直接生成libB.a静态库的话，其中并不会包含libA.a的代码，在后续的其他项目如果通过libB.a来使用libA.a中的代码，那么编译就会挂掉。

```shell
ar x libA.a
ar x libB.a
ar cru libA_with_B.a *.o
```

将libA.a和libB.a都拆解成.o，然后再同样由ar合成一个新的静态库即可。

### 3. 条件语句

使用方法如下
```shell
if (变量名字)
    分支1
else()
    分支2
endif()
```
有几个可以使用的函数如set和message，list
`set(变量名 变量中存的)`
如
```shell
set(USE_LIBRARY OFF) # 设置了一个变量USE_LIBRARY 并且将其设置为OFF
message(STATUS "${USE_LIBRARY}") # 将变量USE_LIBRARY打印。
list(APPEND _source Target.h Target.cpp) # 将Target添加到列表_source中
```
set变量名后再include_directories可能会出错。

### 4. 导入包

`find_package(<PackageName> <>)`
1. 命令首先会在模块路径中寻找Find<name>.cmake，它会通过变量`${CMAKE_MODULE_PATH}`中所有目录。如果没有，就查看自己的模块目录/share/cmake-x.y/Modules/，这个目录位于cmake安装目录(${CMAKE_ROOT})，以上为模块模式。
2. 如果还是一样没有找到，那么find_package会在~/.cmake/packages或者/usr/local/share中各个包目录中查找，寻找<库名字大写>Config.cmake或者<库名字小写>-config.cmake(如果Opencv，就会找/usr/local/share/OpenCV/OpenCVConfig.cmake或者opencv-config.cmake)，以上为配置模式。
如 
```shell
find_package(ffmpeg require)
```

### 5. find_xxx系列命令导入包

```cmake
find_path(变量名, NAMES 需要寻找的头文件, PATHS, 指定在哪个目录里寻找)
```

find_path会找出我们需要的头文件名，然后将目录的位置放到指定的变量名中，默认在`CMAKE_INCLUDE_PATH`中寻找，这玩意属于shell环境变量，如果没有，那么基本只能找到库默认的位置了，比如`/usr/local/include`下的头文件，当然我们可以手动指定一个位置让他寻找，可以通过设定`CMAKE_INCLUDE_PATH`来增加寻找的路径，也可以通过PATHS关键字，比如：

```cmake
find_path(BRPC_INCLUDE_DIR NAMES brpc/server.h PATHS /root/Github/incubator-brpc/output/include)
```

cmake会在/root/Github/incubator-brpc/output/include这个目录里找到对应的brpc/server.h，然后将这个路径放到BRPC_INCLUDE_DIR这个变量中。

find_library和find_path用法大致相同，一个寻找leveldb的例子如：

```cmake
find_library(LEVELDB_LIB NAMES leveldb)
```

cmake会根据leveldb这个名字寻找对应的库，然后放到变量LEVELDB_LIB中。


### 6. Cmake中一些变量

__CMAKE_BUILD_TYPE__ cmake编译生成的可执行文件的类型，`Release`使用优化，且不含调试符号。`Debug`不使用优化，包含调试符号，RelWithDebInfo使用少量优化，含调试符号。 


## 0x01 CMAKE NEW

> 1. aux_source_directory(\<PATH> \<VAR>)

扫描PATH路径中的所有文件，将其作为变量存入VAR变量中，这点在一些源文件比较多，又不想一个个添加的时候比较常用。

### 子项目

假设有这么一个文件树：

```shell
├── CMakeLists.txt
├── main.cc
└── opencv
    ├── CMakeLists.txt
    ├── opencv.cc
    └── opencv.h
```

如果仅由一个CMakeLists来构成项目，那么势必需要包括所有的源码，add_subdirectory可以添加下属的CMakeLists来共同构建一个CMake项目，在主项目的CMakeLists中这样写：

```cmake
cmake_minimum_required(VERSION 3.10)
set(NAME demo3)
project(${NAME})
add_subdirectory(opencv)
aux_source_directory(. CUR_FILE)
add_executable(${NAME} ${CUR_FILE})
target_link_libraries(${NAME} opencv)
```

在次要的opencv目录中的CMakeLists这样写：

```cmake
aux_source_directory(. OPENCV_SOURCE_FILES)

add_library(
  opencv
  SHARED
  ${OPENCV_SOURCE_FILES}
)
```

使得opencv文件夹中的CMake创建出一个动态链接库供主项目调用。

### OPTION

option命令一般配合条件语句来使用，用来控制项编译的时候是否启用某种特性，语法为：

```cmake
option(IS_OPEN "is open?" ON) # 设定为默认打开
if (IS_OPEN)
    # 如果打开的改做什么？
    message("IS_OPEN IS TRUE")
endif(IS_OPEN)
```

然后，通过configure_file命令来进行配合：

```cmake
configure_file(
    "${PROJECT_SOURCE_DIR}/config.h.in"
    "${PROJECT_SOURCE_DIR}/config.h"
)
```

在目录中创建名为config.h.in的文件供cmake进行读取，内容为：

```
#cmakedefine IS_OPEN
```

而在源代码中就可以通过这样进行判断：

```C++
void print(){
#ifdef IS_OPEN
    std::cout << "IS_OPEN TRUE" << std::endl;
#else
    std::cout << "IS_OPEN FALSE" << std::endl;
#endif
```

在cmake生成makefile的时候可以通过-D传入参数来声明某些特性是否打开，比如：

```shell
cmake .. -DIS_OPEN=OFF # 关闭
cmake .. -DIS_OPEN=ON  # 打开
```


# MakeFile

makefile的简易规则

```makefile
target ... : prerequisites ...
 command
 ...
 ...
```

target 是一个目标文件，也可以是一个可执行文件，还可以是一个标签。
prerequisites 是依赖条件
command 该target要执行的命令

<font color=F0000>makefile的核心是一个依赖关系，target依赖于prerequisites，而生成的方式则由command定义。 </font>


### 一些小技巧

使用file来包含所有文件

```cmake
file(GLOB variable [RELATIVE path] [globbing expressions]...)
```

 将检索到的文件都加入到variable中去

 例如: 

 ```cmake
 file(GLOB SRC_FILE "*.cc" "path/test_path/*.cc")
 ```

 则将path/test_path/下的所有.cc文件都加入到变量SRC_FILE中去。



