# Learn

## 0x00 cherry-pick

#### 单commit的pick

cherry-pick可以讲某个分支上的commit点提交到指定的分支上，比如这样的工作分支：

```shell
              f  < - dev2
             /
a - b - c - d    < - master
             \
              e  < - dev1
```

可以在本地同时开启2个工作分支，dev1和dev2，但都提交至master分支，切换过程不产生merge操作，比如讲e这个commit点提交至d分支，产生这样的：

```shell
              f     < - dev2
             /
a - b - c - d - e   < - master
             \
              e     < - dev1
```

使用这样的语句：

```shell
git cherry-pick e
```

如果遇到冲突，可以进行修改后合并：

```shell
git cherry-pick --continue
```

#### 多个commit的pick

比如这样的commit样子：

```shell
              f             < - dev2
             /
a - b - c - d               < - master
             \
              e - g - h - i < - dev1
```

现在从dev1挑选几个commit点pick到master上：

```shell
              f               < - dev2
             /
a - b - c - d - g - h - i     < - master
             \
              e - g - h - i   < - dev1
```

这个range是一个前开后闭的，使用这样的提交语句：

```shell
git cherry-pick e..i
```

这样会产生上述的图像，有3个提交点，也可以增加一个-n，直接将这3个提交点改为一个commit点：

```shell
git cherry-pick e..i -n
git add .
git commit -m "merge g,h,i by one commit"
```

如果需要前闭后闭的区间，比如：[e, i]这种，也可以加上`^`这个来表示：

```shell
git cherry-pick e^..i -n
git add .
git commit -m "merge e,g,h,i by one commit"
```

## 0x01 shell中常用工具

#### less

线上环境排查错误不应该直接使用vim，特别是大log情况下，vim很有可能打爆服务器的内存，应使用less，less内部使用和vim很像，比如“/”可以搜索，打开时指定flag来表示方式：

```shell
-N 显示行号
-I 忽略搜索时的大小写
-g 只高亮一个当前搜索结果(默认是全部高亮)
```

#### grep

grep也是经常使用的一个，但这个最好不要在线上环境使用。
这个命令的意思是，从某个“字符串”中抓出关心的几个，但使用上更多的是将某个命令的结果重定向，然后grep到这个输出结果中抓取重要的几个。

比如：

```shell
ps -ef | grep master # 抓取有master关键字的进程信息
```

但会发现，如果只有一个master进程，但总是可以看到2个结果，其中一个就是grep本身自己这个进程，因为我们指定了master，所以“grep进程也会带上master关键字”，进而也被算进去了，这个可以使用-v过滤掉：

```shell
ps -ef | grep master | grep -v grep # 后面这个过滤掉有grep这个关键字的内容
```

即-v的效果是取反。不匹配的全部抓出来。

不过有时候我也用来抓源码中的函数：

```shell
grep -rn "func1(" *
# -r 表示递归
# -n 表示抓取显示行数
# 总体意思：在当前目录下所有文件中，找到符合条件的所有行
```

#### top

top命令的使用也是经常用到，这里只简单的记录一下常用的。

首先是显示开启后的切换显示命令，这是我用的比较多的：

```shell
m # 切换内存显示方式
M # 以内存使用量排序
t # 切换cpu占比显示方式


E # 内存显示单位切换(头部)
e # 一样是切换单位，显示在列表
1 # 显示所有核心的cpu占用情况

c # 显示完整的进程信息
P # 根据CPU占用方式排序
```

然后是启动时指定的参数：

```shell
-p ${pid} # 指定只看某个进程
-H -p ${pid} # 看这个进程内的所有线程
```

#### htop

TODO:

## 0x02 awk

这玩意用法太多了，另起一个吧，首先是他最基本的工作的原理：

awk处理给定的文本，这个文本可以是一个文件、或者是重定向过来的字符串等等，awk以行为单位进行处理，默认以空格进行分割(-F 指定自定义分割方式)，比如你给这样的数据：

```
name description
swagger hello

```

那么awk得到以下数据：

```shell
# 第一行
$0=整行内容
$1=name
$2=description
# 第二行
$1=swagger
$2=hello
```

现在来看一下比较常用的命令形式

```shell
awk '{pattern command}' ${file_in}
awk 'BEGIN{} pattern{} END{}' ${file_in}
```

BEGIN中的内容只会执行一次，而pattern中的内容则是每行都会执行一次，END同理，结束的时候执行一次而已。

比如抓pid：

```shell
ps -ef | awk 'BEGIN{print "pid"} {print $2} END{print "end of pid"}'
```

中间的这个pattern可以随意加，只要不是被tag了BEGIN/END就会每次循环都执行一次。

每个大括号中也可以加入`;`来增加多条指令进去。

#### 条件

你也可以在里面加上条件，比如我只要pid>1000的：

```shell
ps -ef | awk '{if($2>1000){print $2}}'
```

if条件中我们可以取到行的所有数据，只不过以上例子只用到了$2，你也可以直接使用正则：

```shell
ps -ef | awk '{if(/正则/){xxx}}'
```

或者直接简写：

```shell
ps -ef | awk '/正则/'
```

比如抓ifconfig中的地址，由于地址总是以inet开头，就可以直接这样：

```shell
ifconfig | awk '{if(/inet /){print $0}}'
```




## last

[shell](https://github.com/Swaggerzhan/shellLearn)




