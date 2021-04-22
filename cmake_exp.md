# CMAKE EXP

### 1.初始化

设置一个cmake所需要的最低版本，FATAL_ERROR表示小于此版本为致命错误。
`cmake_minimun_required(VERSION 3.5 FATAL_ERROR)`

声明项目名字为test1，语言使用C++(CXX)
`project(test1 LANGUAGES CXX)`

将编译出一个名为hello-world的可执行文件，它由hello-world.cpp生成
__add_executable ( hello-world hello-world.cpp )__


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
