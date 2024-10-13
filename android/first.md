# Mac平台下交叉编译安卓架构的ELF

## 0x00 下载NDK
TODO

## 0x01 设定环境变量

```shell
# .zshrc
export PATH="/Users/swagger/android/AndroidNDK9519653.app/Contents/NDK:$PATH"
export ANDROID_NDK_HOME="/Users/swagger/android/AndroidNDK9519653.app/Contents/NDK"
```

## 0x02 构建项目

直接使用cmake就好，不过记得需要指定toolchain，给一个例子为：

```cpp
#include <iostream>

int main() {
    std::cout << "Hello World" << std::endl;
    return 0;
}
```

cmake:

```cmake
cmake_minimum_required(VERSION 3.1)
set(NAME first)
project(${NAME})

set(CMAKE_SYSTEM_NAME Android)
set(CMAKE_SYSTEM_VERSION 21) # 改成对应的API级别
set(CMAKE_ANDROID_ARCH_ABI armeabi-v7a) # 对应的架构
set(CMAKE_ANDROID_NDK ${ANDROID_NDK_HOME})
set(CMAKE_ANDROID_STL_TYPE c++_static)

add_executable(${NAME} main.cc)
```

使用cmake构建：
```shell
cmake -DCMAKE_TOOLCHAIN_FILE=$ANDROID_NDK_HOME/build/cmake/android.toolchain.cmake ..
make
```