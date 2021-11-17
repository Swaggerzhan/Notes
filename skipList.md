## SkipList 跳表

跳表的查找操作可以达到logn，对比红黑树，跳表的实现简单了很多，`leveldb `中采用的就是跳表。



#### 一些从leveldb跳表实现学到的



__placement new__ 一般而言，在C++中，我们直接使用`new`关键字来申请内存，在申请内存的同时，如果申请的数据类型是非POD的，还会调用构造函数。

不过有时候，我们需要在一块已经申请的内存上去进行调用构造函数的操作等，就可以使用到`placement new`。

```c++
class Object;
void* mem = malloc(sizeof(Object));
new (mem) Object; // 在内存 mem上构造对象Object
```

在高并发中，如果有提升性能的要求，我们会在用户空间直接创建属于自己的内存池，而不会直接调用到底层的malloc等申请内存的系统调用以提高内存的申请速度，这时`placement new`就可以派上用场了。

在leveldb的skiplist中，对于`placement new`有一用法，为:

```c++
template <typename Key, class Comparator>
typename SkipList<Key, Comparator>::Node* SkipList<Key, Comparator>::NewNode(
    const Key& key, int height) {
  char* const node_memory = arena_->AllocateAligned(
      sizeof(Node) + sizeof(std::atomic<Node*>) * (height - 1));
  return new (node_memory) Node(key);
}
```

其中，通过申请的内存块`node_memory`，之后直接构建`Node`节点，但这里只构建了首个`Node`，我们可以看到申请的其实是一整块连续的内存，大小为`sizeof(Node) + sizeof(Node*)*(height-1)`，也就是一个节点加若干个指针的大小，这种申请方式取决于`Node`中的定义。



### SkipList节点的设计节点设计以及最重要的函数FindGreaterOrEqual

#### 节点Node设计

leveldb中的节点设计和申请内存的方式挺巧妙的，我这边以(int -> int)来演示，具体简略为代码:

```c++
struct Node{
	int key;
  int value;
  Node next[1];
};
```

这个`next`为何不使用`malloc`？主要后续申请内存时，我们可以申请同一块内存来进行存储，少了许多不必要的麻烦，就是之前提到的`NewNode`函数。

#### FindGreaterOrEqual

我们知道，对于单向链表，插入操作需要找到插入位置的前驱才能完成，而这个函数做的就是找到需要插入的位置的前驱。

具体的步骤

```
1. 在最高的层上进行寻找，直到next为nullptr了，又或者next已经大于等于我们需要寻找的key了，那也就是等同于找到
	了当前层数中的前驱cur。
	
2. 以此类推，知道找完所有的层数，如果是对于insert操作：找到insert位置的前驱，那么这些任务已经够了。

//----- 寻找操作中的FindGreaterOrEqual ----- // 

1. 和上步骤1相同
2. 当我们找到最后一层，也就是第0层时，cur是key的前驱，而next必定时大于等于key的，我们通过返回next来进行比对
	是否next->key == key，就可得知SkipList中是否存在当前的节点了。
```

```c++
Node* FindGreaterOrEqual(int key, Node** prev){
	Node* cur = head;
  int level = maxHeight; // 当前最大层数，我们从最高层开始寻找
  while ( true ){
    Node* next = cur->next(level);
    if (next != nullptr && key < next->key){
      cur = next; // 继续在当前层进行遍历查找操作
    }else { // 如果发现了next的key已经大于等于key了，那cur必定是其前驱
      if (prev != nullptr ) prev[level] = cur;
      if (level == 0)
        return next;
      else
        level --;
    }
  }
}
```

有了这个最核心的查找前驱的函数，接下来编写插入，寻找函数就变得十分容易了。

### 1. 插入步骤

```
1. 找到需要插入的位置，通过Node* prev[kMaxHeight]来存储前驱节点，即我们需要插入的节点的前驱
2. 摇筛子来确定新节点的层数，例如我们摇到了r,即表示区间[0, r]
3. 将 (新节点的next[0]到next[r]) 指向 (prev[0] 到 prev[r] 的next)
4. 将 (prev[0]到prev[r]的next) 指向新节点
```

插入的程序实现

```c++
int kMaxHeight; // 最大层限度
int MaxHeight;  // 当前最大层数
void SkipList::insert(int key, int value){
  Node* prev[kMaxHeight];
  // 找出所有前驱prev
  Node* x = FindGreaterOrEqual(key, prev);
  
  int height = rand() % kMaxHeight;
  // 如果超过了最高层，那么其前驱必定是头节点
  if ( height > MaxHeight){
    for (int i=maxHeight; i<height; i++) {
      prev[i] = head;
    }
    maxHeight = height;
  }
  x = newNode(key, value, height);
  for (int i=0; i<height; i++) {
    x->setNext(i, prev[i]->next(i));
    prev[i]->setNext(i, x);
  }
}
```

### 2. 查找操作

查找操作非常的简单，直接通过一开始的FindGreaterOrEqual找到大于等于key的节点即可。

```c++
bool SkipList::get(int key, int* value){
	Node* next = FindGreaterOrEqual(key, nullptr); // 找到key节点
  if (next != nullptr && next->key == key){
    *value = next->value;
    return true;
  }
  return false;
}
```























