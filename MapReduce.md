# 6.824 Lab1 MapReduce 学习笔记

## 0x00 理论

MapReduce分为Map阶段和Reduce阶段。

Map阶段: Map函数将拆分数据，例如将一辆汽车拆分成一个个零件。

Reduce阶段: Reduce函数将合并之前Map拆分的数据称为一组新的数据，例如将汽车拆分的零件合成变形金刚。



MapReduce细分来讲，存在6个阶段，在最开始是`input阶段`，当Map函数获取input中的数据时，称为`split阶段`，以及Map函数执行的`Map阶段`，Reduce获取Map的输出结果称为`shuffle阶段`，Reduce执行的`Reduce阶段`，Reduce执行完毕输出的`Finalize`阶段。

我们以一个最经常使用到的WorldCount示例作图来说明

![](./MapReduce_pic/1.png)

可以看到，每个阶段上的操作，都能找到一个键值对，在`Split`阶段，健值对可以是`行数->行内容`，在`Map阶段`，健值对是`单词->单词个数1`，在`Shuffle阶段`同样如此，在`Reduce阶段`健值对则被更新为了整体`单词->真正的单词个数`。

由此可以大致推断其中的`Map`和`Reduce`函数，伪代码:

```python
# key为行号
# value为行内容
Map(string key, string value):
  for each word in value:
  	output(word, 1)

# key 为 word
# valueList 是 word个数，也就是一个个 “1”
Reduce(string key, list valueList):
  total = 0
  for value in valueList:
    total += value
  outputFinal(key, total)
```

通过查看Reduce函数，也可以加深理解`Shuffle`过程的重要性。

通过以上例子，整理一下，得到整体的架构图:

![](./MapReduce_pic/2.png)

## 0x01 6.824 Lab1实现

实验源码:

```shell
git clone git://g.csail.mit.edu/6.824-golabs-2021 6.824
```

Lab1中为我们定义了3个文件，我们按照其规则补完其剩下的代码即可，分别为`mr/coordinator.go`、`mr/rpc.go`、`mr/worker.go`，代表的即是Master，调用过程中需要用到的RPC参数以及返回内容，需要由我们自定义，以及Worker。

##### 大致逻辑

在Worker中我们这样设计，Worker是一个人死循环，一直通过RPC向Master获取任务，然后执行这个任务，这个任务可能是Map，也有可能是Reduce。

```
1. 通过RPC向Master获取任务
2. 执行任务
3. 提交任务，继续获取新任务

任务分为Map和Reduce，在这里WorldCount例子中，我们这样做:
Map任务中，通过给定文件读取响应数据，调用Map函数，然后将处理完的数据写入到Hash桶中
Reduce任务中，读取关于这个Reduce需要用到的中间结果，然后执行Reduce操作，输出结果到最终文件
```

在Master中我们这样设计，Master是一个分配任务的角色，所以通过输入的文件量，我们分配固定的Task，也就是任务，然后通过RPC开放某些接口以供Worker来调用，主要用于获取任务，提交任务等等操作。

```
1. 生成Master对象，初始化Task，开始对外提供RPC服务
2. 通过Worker调用的RPC分别处理不同的请求

这里Master对外开放2个RPC，分别为请求任务和提交任务
请求任务:
  通过RPC响应将需要的文件名发送到Worker中，标记为当前Worker正在处理，这里需要处理一下Worker超时情况，需要
  重新分配任务给其他Worker。
提交任务:
	通过RPC开放，检测当前记录中的Worker和提交Worker的ID是否相等，如果不等，说明发生了重新分配等情况，需要做其他处理
```



