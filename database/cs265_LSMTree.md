<font face="Monaco">

# CS265-LSMTree

CS265中关于LSMTree的Lab，关于LSMTree的经典完整实现可以看Google的Leveldb。

此项目中的代码仅仅只针对int32_t->int32_t的kv Store，其记录的定义为：

```C++
typedef int32_t KEY_t;
typedef int32_t VAL_t;
struct entry {
    KEY_t key;
    VAL_t val;
};
typedef entry entry_t;
```

## 0x00 设计

这个项目中包含的封装并不多，更多的是关于LSMTree的简单实现，其内部比较重要的封装有如下几个：

> 1. LSMTree: 主角
> 2. Run: 磁盘读写的抽象
> 3. Level: 层次的抽象

那就先从磁盘的方面讲起吧。

### Run

这是一个关于磁盘读写的抽象，有一些属性：

```C++
class Run {
// ...
private:
    string file_;
    long cur_size_;
    long max_size_;

    vector<KEY_t> fence_pointers_; // for fence pointer
    KEY_t max_key_;

    int mapping_fd_; // mmap
    entry_t* mapping_; // the address of mmap return value
    size_t mapping_length_;
};
```

> 1. file_、cur_size_、max_size_

file_是一个磁盘文件的路径名，cur_size_则是当前文件大小，max_size_为文件最大容量。

> 2. mapping_fd_、mapping_、mapping_length_、

既然Run是一个关于磁盘文件管理的类，不免需要和磁盘IO打交道，这里用了比较简单的实现方式，直接采用mmap(更多的是考虑LSMTree实现逻辑，所以对减少磁盘IO次数这里暂不提)， __mapping_fd_就是打开的文件fd，而mapping_即是关于磁盘文件到内存的映射首地址，其是以entry_t为元素的数组，数组大小为mapping_length___ 。

__Run中存储entry_t是有序的，呈递增顺序，在Merge时会进行排序，然后存储到磁盘，并且，每个Run中的entry_t的KEY_t是不会重复的__。

接下来是一些开放的接口：

```C++
class Run {
public:
    entry_t* map_read();
    entry_t* map_write();
    void unmmap();
    VAL_t* get(KEY_t);
    void put(entry_t);
};
```

> map_read() & map_write()

map_read是一个关于文件读的函数，通过打开file_文件，然后将其映射到内存中，返回entry_t\*的数组。map_write则是文件写函数，同样返回entry_t\*数组，但其返回的内容是可写权限的。

> get(KEY_t) & put(entry_t)

说明的很明显了，首先是get，获取磁盘上一个关于KEY_t的值，put也就将entry_t放入到磁盘中，这两个接口并不建议用户直接使用，直接读取磁盘的效率是非常低下的，get和put一般都是LSMTree内部其他实现类来进行调用，put也仅仅支持追加的形式，更多的是在Merge的时候来使用。


### Level

LSMTree的结构是一层层的，一般而言，每层的容量都是不一样的，它们会层层递增，并且层度也会有限制，比如Leveldb中限制为7层。

项目中Level的封装就是整个层的抽象，具体一些属性：

```C++
class Level
private:
    float test_filter_;
    int max_runs_;
    long max_run_size_; //??
    std::deque<Run> runs_; // ?
};
```

> runs_ & max_runs

如果每个Level仅仅只存储关于一个磁盘读写的抽象，每次Merge需要将整个Level文件读出，那么随着层数的增高，整个操作将是非常耗时的，所以一层将存储多个小的Run，并且固定其大小，其中max_runs就是每层Level中所固定的大小，当容量满时，就需要进行Run的Merge操作了。

__这里的Run排序也是有讲究的，是按照时间排序，最新的Run将被排到数组头部，并且前面说过，每个Run中不会出现重复的entry_t，但多个Run其实还是有可能会出现重复的entry_t__ 。

Level也开放了一些接口，比如：

```C++
class Level {
public:
    bool remainning() const;
    void newRunAndWrite();
};
```

> remainning()

一个关于容量的接口

> newRunAndWrite()

在runs_头部添加进一个新的Run，并且打开Run其中磁盘文件的读写，这里一般会被用在Merge上。

### LSMTree

前面的小弟介绍完了，后面就是主角了，即LSMTree，直接来看一些属性吧：

```C++
class LSMTree {
private:
    Buffer buffer_; // 内存中的kv
    std::vector<Level> levels_;
};
```

> levels_

这是一个关于Level的vector，其数组的索引代表着层数的高低，index越低表示层数越低，其中可被容纳的Run也就越少。


## 0x01 LSMTree的接口以及Merge操作

```C++
class LSMTree{
public:
    void put(KEY_t, VAL_t);
    VAL_t get(KEY_t);
    void del(KEY_t);
private:
    void merge_down(vector<Level>::iterator);
};
```

正常情况下，数据库都是有增删改查的，但LSM结构不同，其删除也是append操作，改也是append操作，为的就是最大限度的提高写吞吐量。

### read

read即get操作，get操作的解析很简单，首先直接在内存Buffer中进行寻找，也就是所谓的Level0，内存的查找操作是非常迅速的，如果当前key不在内存中，那么根据层层Level向下进行寻找，每层Level中又有不同的Run， __根据前面设计的性质，每个Level中，总是最新(latest)的Run排在前面，即我们仅仅需要找到最新的并且其中含有key的Run即可，按照这个思路不难写出代码__ ：

```C++
VAL_t* LSMTree::get(KEY_t key) {
    // 直接在内存中找到
    VAL_t* value = buffer_.get(key);
    if ( value != nullptr )  {
        if ( *value == VAL_TOMBSTONE ) // 无效数据，被删除了
            return nullptr;
        return value;
    }

    Run* run = nullptr;
    VAL_t* val = nullptr;
    int run_index = 0;
    while ( true ) {
        if ((run = get_run(run_index)) == nullptr) {
            return nullptr;
        }
        if ( run->get(key) == nullptr ) {
            run_index ++;
            continue;
        }
        if ( *val == VAL_TOMBSTONE )
            return nullptr;
        return val;
    }
}

Run* LSMTree::get_run(int index) {
    for (const auto& level: levels_) {
        if ( index < level.runs.size() ) {
            return (Run*)&level.runs[index];
        }else {
            index -= level.runs.size();
        }
    }
    return nullptr;
}
```

从查找思路中就可以明确的看出，read在LSMTree中是非常耗时的，特别是当需要的key不存在于数据库中，那么我们必须从头到位的扫描一遍，当然我们可以通过一些比如布隆过滤器来过滤掉一些肯定不存在的read(key)操作，但即便如此，LSM在read方面也确实不如其他数据库。

### write

这就是LSM的强项，write操作有3个，增、删、改在LSMTree中都是以append来实现的，首先我们讲put操作，也就是增。

#### put(entry_t)

同样的，如果Level0的数据并没有达到极限，那么显而易见就是直接丢到存于内存中的Level0即可立即返回了，复杂的情况发生在Buffer满了，并且有可能的是Level1或者其余层数上的Run也达到了最大值，这时候我们不得不进行Merge操作。

> case one:

仅仅是Level0满，而Level1中的runs_并未达到最大值，那么非常简单，直接生成一个新的Run，插到Level1头部即可，然后将Level0，也就是Buffer的数据丢给新创建的Run，由这个Run将数据排序并且Flush到磁盘中即可，然后我们请空Buffer，执行put操作即可。

> case two:

Level1中的runs_也满，那么我们不得不的进行Merge了，我们将Level1中的所有Run都取出，然后进行合并，合并的优先级是越往前越新，这是我们规定的，相同的key，越往后的Run中的key将被靠前的Run中的key所覆盖，这样能确保我们获取到的数据是最新数据(up to date)。

> case three:

执行Level2或者Level3中的数据都满了，那么我们只能递归的进行这项操作了，具体操作如同case two，不过这个层数并不是越来越高的，我们可以将其限制在某个层数，比如像Leveldb那样，在Level7上，不限制Run的大小。

__无论我们如何进行Merge操作，最终Level1上总是有一个可用的空间可以用来存放新的Run__ 。

基于上述，不难写出代码：

```C++
void LSMTree::put(KEY_t key, VAL_t val) {
    if ( buffer_.put(key, val) ) { // Level0中即可返回
        return; // success
    }
    // 失败了，那么就需要进行向下合并操作
    merge_down(levels_.begin());

    // 开辟新的Run空间
    levels_.front().newRunAndWrite();

    // 持久化: 写入到run中mmap得到的内存地址 -> 磁盘
    for (const auto& i: buffer_.entries_) {
        entry_t target;
        target.key = i.first;
        target.val = i.second;
        levels_.front().runs.front().put(target);
    }

    levels_.front().runs.front().unmap(); // 释放磁盘资源

    buffer_.empty(); // 清空Level0
    assert( buffer_.put(key, val)); // 这个操作总是成功的
}
```

#### update & delete

说完了增，还剩下改和删，这两个操作在LSMTree中直接由增代替， __如果需要修改一个存在的entry，那么我们直接增加一个新的entry即可，查询时LSMTree会自己判别哪个才是最新的数据__ (Run上查找的规律)。

删除就更简单了，同样由增实现，直接增加一个相同的key，但其映射到一个无效的value上，我们在查询的时候就可以判别出来。

## 0x02 测试

// TODO: wait for update

## 0x03 参考及改进

目前项目还未做到多线程的查询操作，如果发生Merge操作，那么整个数据库都将被阻塞，以及缺少布隆过滤器，磁盘快速索引等等，后续如果有时间将实现多线程的、包含布隆过滤器的、有磁盘索引的版本，在此留坑。

课程地址：[这里](http://daslab.seas.harvard.edu/classes/cs265/project.html)
参考项目：[这里](https://github.com/jackdent/cs265-lsm-tree)

</font>