# CMAKE

### 1.基本语法

在`CMakeLists.txt`中，我们可以定义一个最低的版本要求，生成的二进制名，也就是整个项目的名字，比如:

```cmake
cmake_minimum_required(VERSION 3.10) # 最低要求3.10版本的cmake
project(MyProject) # 项目名，也就是最终生成二进制时的名字
```

通过`set`指令，我们可以设定一些必要的参数，具体一些示例:

```cmake
set(CMAKE_CXX_STANDARD 11) # CXX表示C++ 版本11
set(CMAKE_BUILD_TYPE DEBUG | RELEASE) # 构建DEBUG或者是RELEASE版本
```

 添加源文件到项目中时，我们可以使用`add_executable`。例如:

```cmake
add_executable(
	MyProject
  A.h	A.cpp
  B.h B.cpp
) # 表示将其中的文件都添加到二进制MyProject中去
```

##### 使用一些库文件

一些指令是用于添加库文件的，有对于文件引用，又或者是库引用的。

```cmake
include_directories(/tmp/include) 		# 引用/tmp/include下的文件
link_directories(/tmp/lib)						# 引用/tmp/lib下的库，需要配合target_link_libraries
target_link_libraries(MyProject test)	# 将libtest.so库添加到MyProject中
																			# /tmp/lib/libtest.so
```


### 2. 生成静态链接库

```shell
add_library(
    message # 库名
    STATIC  # 生成库的方式，这里是静态链接库
    Target.hpp # 头文件
    Target.cpp # 
)
```
以上代码将生成静态链接库`libmessage.a`，在使用可以直接链接名字`message`即可。
其中的 __STATIC__ 字段也可以改为 __SHARED__ 表明生成动态链接库
最后在编写cmake的时候先由源代码生成所谓的`目标文件`，之后再进行链接静态链接库
`target_link_libraries(hello-world message)`。这里链接器会将其链接成一个可执行文件。

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

### 5. Cmake中一些变量

__CMAKE_BUILD_TYPE__ cmake编译生成的可执行文件的类型，`Release`使用优化，且不含调试符号。`Debug`不使用优化，包含调试符号，RelWithDebInfo使用少量优化，含调试符号。 

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



