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

其中，通过申请的内存块`node_memory`，之后直接构建`Node`节点，但这里只构建了首个`Node`，我们可以看到申请的其实是一整块连续的`Node`内存，除非你知道你在做什么，否则少了构造函数的类型可能会出现问题。



### 1. 插入步骤

```
1. 找到需要插入的位置，通过Node* prev[kMaxHeight]来存储前驱节点，即我们需要插入的节点的前驱
2. 摇塞子来确定新节点的层数，例如我们摇到了r,即表示区间[0, r]
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
  Node* x = findGreaterOrEqual(key, prev);
  
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

























