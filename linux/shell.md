<font face="Monaco">

# SHELL 语法

## 0x00 interpreter

首先是shell的运行方式，如果直接运行shell脚本，那么一般而言，操作系统会为当前的shell脚本起一个新的进程来运行(起新进程)，那么其中的环境变量将无法继承，如果使用source命令来启动shell脚本，那么环境变量可以继承(直接运行)。

### example

```shell
# 设置当前一个测试的环境变量test
test=10
echo "this is test value: ${test}"
# 下面是输出：
this is test value: 10
```

如果直接当成脚本运行or直接sh命令运行，那么环境变量不继承，也就是test值是空的。

```shell
echo 'echo "this is test value: ${test}"' > script.sh
sh ./script.sh
# 
chmod +x ./script.sh
./script.sh
```

这两个输出都将得到空的值。

如果使用source命令，那么环境变量可以看到(直接在当前进程中运行)。

```shell
source ./script.sh
this is test value: 10
```

又或者，可以提前使用export命令将变量引入环境变量，之后再另起的脚本进程(子进程)，将可以得到来自父进程的export变量。

即，正常的shell脚本可以在当前的环境变量中进行寻找“需要的变量”，而如果没有，则可以向上寻找(父进程)，而进程中的变量需要被标记为export才能被其他进程所发现。如果父进程中的变量不是由export引入的，那么将无法被子进程所识别。

## 0x01 变量

在shell中，变量的声明比较多种多样，并且比较简单，其中需要特别注意的是关于=号两边不能存在空格。

一些变量的声明和使用例子：

```shell
# ********************** 声明 **********************
test=abcd

test1=$test

# 将out.txt中的数据作为test2变量的内容
test2=`cat ./out.txt`

# 或者也可以这样写
test2=$(cat ./out.txt)

# ********************** 使用 **********************

# 通过$来使用变量
echo "this is value of test1=$test1"

# 通过${}来使用变量
echo "this is value of test1=${test1}"
```

其中，如果是双引号，那么shell可以解析其中字符串的特殊含义，如果是单引号，那么shell解释器将对其中的字符串不做特殊解释，直接作为字符串。

set可以获取当前环境的所有变量，unset可以删除某个特定的变量。

### 一些特殊变量

C/C++的程序运行时可以通过args等传入一些启动时的参数，shell也是如此，shell设定$N为某个输入的参数，其中$0为当前shell脚本的文件名字，例如：

```shell
#!/bin/bash
# script.sh
echo "current shell name: $0"
echo "args1: $1"
echo "args2: $2"
```

运行：

```shell
root@Swagger:~/shell# sh script.sh args1 args2
current shell name: script.sh
args1: args1
args2: args2
```

#### $*

当前程序所有参数，也就是从$1 - $N的所有参数。可以理解成C/C++中的args，不同的是少了一个index=0的参数而已。

#### $# 

当前程序参数的个数，仅计算从$1开始到$N的个数($0不算在内)。可以理解成C/C++中的argc，少了一个index=0的个数。

#### $$

当前程序的进程id，即PID。

#### $!

执行上一个后台程序的PID。

#### $?

上一个程序的返回值。

### 交互

read命令类似C++中的cin操作，默认可以从标准输入中获取数据到指定的变量中，比如：

```shell
#!/bin/bash
read var1 var2 var3
echo "${var1} ${var2} ${var3}"
```

运行

```shell
root@Swagger:~/shell# sh script.sh 
10 20 30
10 20 30
```

### expr做运算

expr可以做一些简单的运算，比如：

```shell
#!/bin/bash
ans=`expr 10 + 10`
ans=`expr 10 - 10`
ans=`expr 10 / 10`
ans=`expr 10 \* 10` # 乘法符需要转义
```

### 变量比对

// TODO:


## 0x02 重定向

### > 和 >>

\>是重定向符号，会进行覆盖操作，而\>\>也是重定向，但执行追加操作，在默认的shell环境下，0为标准输入，1为标准输出，2为标准错误输出。

例如：

```shell
echo "test" > log.txt
# 其真正的应该是：
echo "test" 1> log.txt
```

其中1是可以简写省略，但整体的意思其实是将fd=1的输出重定向到文件中(原来是屏幕)， __如果没有文件名，我们也可以指定为某个“文件描述符”__ 。比如：

```shell
echo "test" > log.txt 2>&1
# 简写：
echo "test" &> log.txt # ☑️
echo "test" >& log.txt
```

代表将标准错误输出2输出至fd=1的文件描述符号，也就是log.txt。

## 0x03 循环和条件控制

### for循环语法

```shell
num=10
for((i=0; i<${num}; ++i)); do
    echo "this is loop"
done
```

## some script

记录一些脚本TODO：

</font>