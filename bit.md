### 一些位运算的使用 C++



#### 获取某个数据类型的最大值

比如int

```c++
int value = 1;						// 0000 0000 0000 0000 0000 0000 0000 0001
(value << 31);						// 1000 0000 0000 0000 0000 0000 0000 0000
// 以上是关于int的最小值，如果我们按位取反，就可以得到最大值
value = ~(value << 31);		// 0111 1111 1111 1111 1111 1111 1111 1111
```

当然你也可以用sizeof来进行其他类型的操作，比如

```c++
long value = 1;
value = ~(value << (sizeof(long) * 8 - 1));
```

又或者，你可以直接从-1进行移位

```c++
int value = (-1 >> 1);
```

如果是unsigned类型的，更简单，直接取反即可

```c++
unsigned int value = 0;
value = ~value;
```



#### 除2

```c++
int value = 20;
value = value >> 1;
```

注：只能用于正整数！



#### Roundup of power of 2

向上取到一个2的指数，这个用在取余时很方便，Linux Kernel中就用这种方法代替取余。

```c++
// 具体思路就是将最高位以及之后都抹成1，然后+1
int roundup(int value){
  value |= value >> 1;
  value |= value >> 2;
  value |= value >> 4;
  value |= value >> 8;
  value |= value >> 16;
  return value + 1;
}
```

之后就能通过`&`来代替`%`了，在2的指数下，两者的操作是等同的

```c++
int value = 20;
value = roundup(value);
int target;
target & (value - 1) == target % value; // true
```



