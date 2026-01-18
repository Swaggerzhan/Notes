# 1、theory

todo

# 2、Practice

当DB使用Percolator算法作为事务的实现模型时，底层的存储层一般是RocksDB，比如TiKV，下面是对其工程实践的简单理解。

## 2.1 Data Structure

TiKV使用了RocksDB中的3个CF，分别是下面的键值对格式：

* DEFAULT: `key + start_ts` => `value`
* LOCK: `key` => `lock_info`
* WRITE: `key + commit_ts` => `write_info`

Percolator是快照隔离级别，每个`key`同一时刻只能持有一把锁，锁的归属通过`lock_info.start_ts`来标识，故不需要替他维护多种版本，所以`LOCK`的`key`字段没有`ts`的，下面是一些名词解释需要知道的：

* key: 用户需要写入的key
* value: 用户需要写入的value
* start_ts: 事务开始时间戳
* commit_ts: 事务结束时间戳
* lock_info: 锁的一些信息
* write_info: 写的一些信息，或者说是commit情况

`start_ts/commit_ts`实际上可以理解为是一个时间线，类似raft中的index，在TiKV中，由PD来生成，在事务开始前和事务提交时都会获取一次，`start_ts`用来获取事务开始时数据库的snapshot，`commit_ts`则是用来原子的提交事务(什么时候提交)。

lock_info/write_info 结构可以简单的看下下面的代码例子：

```cpp
enum LockType {
	LOCK_TYPE_PUT = 0,
	LOCK_TYPE_DELETE = 1,
	LOCK_TYPE_LOCK = 2,
};

struct LockInfo {
	std::string primary_key;
	LockType lock_type;
	uint64_t start_ts;
	uint64_t ttl;
};

enum WriteType {
	WRITE_TYPE_PUT = 0,
	WRITE_TYPE_DELETE,
	WRITE_TYPE_ROLLBACK,
	WRITE_TYPE_LOCK,
};

struct WriteInfo {
	WriteType write_type;
	uint64_t start_ts;
};
```

## 2.2 Process

> BigTable能够做到单行事务，但实际工程实现，比如TiKV使用RocksDB的方式，不会将所有数据都塞到一个value中的(性能问题)，而如果将Value分开存储，就需要保证一次每次操作能够做到原子，这点多数数据库采用的是RocksDB的Snapshot，以及内存中的Latch锁。

### 2.2.1 Read

> 为了保证视图一致，三个step都会在一个相同的RocksDB Snapshot内操作。

TiKV中，事务的读流程为：
##### step1:
读取`LOCK`中对应的`key`，判断是否存在对应的`lock_info`：
* 不存在：执行step2
* 存在：检测lock_info内容，根据`lock_info.start_ts`进行判断：
	* `lock_info.start_ts` > 当前事务的`start_ts`： 此锁是在本事务之后被加上的，可以执行step2
	* `lock_info.start_ts` < 当前事务的`start_ts`：有事务在修改该`key`，根据`ttl`判断这个锁是否有效：
		* 有效：[1] 回滚当前事务，存在冲突。
		* 无效：可能需要执行cleanup流程(后续说明)

##### step2:
读取`WRITE`中的`key`，在各种key的版本中，找到`commit_ts` < `start_ts`中最大的那个：
* 不存在：返回key不存在
* 存在：查看`write_info`中的内容，由于有很多种type，所以需要一个个判断：
	* `PUT`：获取`start_ts`，进入step3
	* `DELETE`：返回`key`不存在
	* `LOCK/Rollback`：继续沿着version往下寻找

##### step3:
读取`DEFAULT`中的`key`(step2中我们已经拿到对应的start_ts了)，并且返回值即可。

[1]: 理论上可以有优化，可以在不改变start_ts的情况下，进行退避重试，重试时需要重新获取RocksDB的Snapshot，重试step1 - step3

### 2.2.2 Write

> PreWrite/Commit流程需要看做是一次原子的操作，由于需要写入，所以这里用的是内存中的Latch，对相同Key的Read-Check-Modify需要原子。

TiKV中，事务的写流程为：
##### step1 (preWrite):
读`WRITE`中对应的`key`，找到最新的`commit_ts`和当前的事务的`start_ts`判断：
* `commit_ts` >= `start_ts`：[1] 事务终止，写写冲突，已有更新的事务修改了该`key`，并且已经commit了
* `commit_ts` < `start_ts`：事务可以继续
读LOCK中的`key`，根据是否存在判断：
* 不存在：继续下一步
* 存在：事务中断，已有事务对此`key`上锁了，处理中，事务中断
写`LOCK/DEFAULT`中对应的`key`，其中，`DEFAULT`对应的`value`就是用户需要写入的值，而`LOCK`中的值需要指向主键[2]

##### step2 (commit):
读`LOCK`中的`primary key`，判断先前写入的内容是否还存在：
* 不存在：可能是`ttl`原因被删，事务终止
* 存在：需要继续判断，判断其中的`start_ts`是否是当前的事务`start_ts`
	* 是：事务继续
	* 否：事务终止
* 写入`WRITE`内容，事务到此如果成功，就已经commit了
* 删除`LOCK`中`primary key`对应的内容

在完成`primary key`的commit操作后，返回成功了，异步的按照写`WRITE`、删`LOCK`的逻辑清除后续的`secondary key`即可。


[1]: 写写冲突主要看commit_ts，当用户使用SELECT FOR UPDATE时，也会在这里写入一条write_info.type = lock的记录，从而创造写写冲突，故这里应该判断lock/put/delete类型，如果非这些类型，那么不视为冲突。
[2]: LOCK中的Key，需要都指向主键。


### 2.2.3 Cleanup(CheckTxnStatus)

> Cleanup的整个请求是需要在Latch(Primary Key)下完成的，因为需要进行修改，需要原子，读则也是需要在RocksDB的Snapshot下完成

Cleanup可能触发Rollback或者Roll-forward，在读过程中，如果一个读发现了Lock超时了，那么可能触发清理，我们假设start_ts和commit_ts对应需要被清理的事务：

##### step1:
读`LOCK`中`primary key`对应的`lock_info`，根据`lock_info.start_ts`是否和事务的`start_ts`一致来判断存在与否：
* 存在：检测`ttl`值，判断是否过期：
	* 过期：返回，并尝试执行rollback，删除`LOCK`表中的`lock_info`，并给`WRITE`写入`primary key + start_ts`[1]的Rollback记录，并返回回滚成功
	* 不过期：返回，不进行其他操作
* 不存在：`primary`锁已经被清理了，此时无法确认事务回滚了还是被提交了，需要进一步检测，进入step2

##### step2:
读`WRITE`中的`primary key`，这里使用的是遍历，由于我们只知道`start_ts`，所以只能遍历找到`write_info.start_ts == start_ts`：
* 找到了，记录是`Put/Delete`：说明commit已经提交，返回[2]
* 找到了，但是记录是Rollback：说明事务已回滚，返回
* 没找到：有可能是PreWrite丢了(没有LOCK)，那么我们尝试回滚，写入`WRITE`的`primary key + start_ts`[1]的Rollback记录，并返回回滚成功

[1]: 对WRITE写回滚记录的时候，其实使用的是start_ts，并不是commit_ts，因为事务没有提交，并没有commit_ts，但由于寻找是否commit不止是找Rollback记录，还有可能是Put/Delete等，所以还是需要进行遍历的
[2]: 这里是会返回事务的真实commit_ts的，后续ResolveLock会用到


### 2.2.4 Cleanup(ResolveLock)

> 过程仍然是需要在Latch(Secondary Key)以及RocksDB的Snapshot加持下进行的

通过主键可以判断事务是被committed还是被rollback的，之后就可以清理Secondary Key了。

##### step1(roll-forward):
拿`LOCK`中对应`secondary key`的锁，判断是否存在(存在的话还需进一步判断`start_ts`是否一致)：
* 不存在：事务可能已被清理了，结束
* 存在：锁还在，获取`lock_info`内容，根据其中的lock类型，删除`LOCK`中的这条记录，并写入`WRITE`的对应信息(commit_ts)，完成secondary的commit

##### step2(rollback):
拿`LOCK`中对应的`secondary key`的锁，判断是否存在(存在的话还需进一步判断`start_ts`是否一致)：
* 不存在：事务可能已经被清理了，结束
* 存在：锁还在，获取lock_info内容，删除`LOCK`记录，并在`WRITE`写入对应信息(start_ts)，完成secondary的Rollback

## 2.3 TSO (Timestamp Oracle)

TiKV中的时间戳貌似是以高bit使用物理时钟填充，低bit使用逻辑时间填充；一个更很简单的想法就是使用一个raft group，并在状态机上维护一个counter，每一次请求就递增一下即可，亦或者直接提交一个空的raft log即可，使用raft apply log index作为逻辑时间即可，或者还有一个办法，每一次apply的时候，申请的多一点：

例如当前时间戳已用`[0, current)` ，那么Leader直接申请 `[current, upper_bound)`，即提交upper_bound到状态机，之后`[current, upper_bound)`都可以由Leader自主分配，不用走raft log，而当新Leader起来后，只需要重新发一次`[upper_bound, new_upper_bound)`的申请即可，保证时间戳仍然是递增，只是浪费了一些没有使用到的而已。



