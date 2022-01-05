# MESI

## 0x00 基础

MESI分别表示了4种缓存状态，为`M(Modify exclusive)`、`E(Exclusive)`、`S(Shared)`、`I(invalid)`。

具体含义为：

### M(Modify exclusive)

即修改的，当前Cache Line数据和对应的内存数据不同，只有当前core在使用此内存数据(独享)。


### E(Exclusive)

即独享的，当前Cache Line数据和对应的内存数据相同，只有当前core在使用此内存数据(独享)。

### S(Shared)

即共享的，当前Cache Line数据和对应的内存数据相同，多个core都在使用此内存数据，也就是多个core中的Cache Line都有此内存数据的高缓(共享)。

### I(Invalid)

该Cache Line没有被使用(无效)。


## 0x01 状态改变

![](./MESI_pic/MESI_state.png)
图片引用自：https://blog.csdn.net/xiaowenmu1/article/details/89705740



## 0x02 代码模拟

为了加深理解，试着用代码来“模拟”了一下MESI的运行流程：

```C++
/* MESI.h */
#ifndef MESI_MESI_H
#define MESI_MESI_H

#define CORE_NUM 2048

typedef enum {
  Modify,
  Exclusive,
  Shared,
  Invalid
}State;

class CacheLine {
public:

  CacheLine();

  ~CacheLine()=default;

  bool read(int coreIndex, int* ret);
  bool write(int coreIndex, int value);
  void check();
  void stateChange(State src, State to);

private:

  State state_[CORE_NUM];
  int data_[CORE_NUM];

  // in mem
  int mem_data_;

  // for check
  bool debug_;
  int m_count_;
  int e_count_;
  int s_count_;
  int i_count_;
  
};
#endif //MESI_MESI_H
```

```C++
/* MESI.cc */
#include "MESI.h"
#include <cassert>
#include <iostream>

using std::cout;
using std::endl;

CacheLine::CacheLine()
: m_count_(0)
, e_count_(0)
, s_count_(0)
, i_count_(CORE_NUM)
, debug_(true)
, mem_data_(0)
{
  for (int i=0; i<CORE_NUM; ++i) {
    state_[i] = Invalid;
  }
}

void CacheLine::stateChange(State src, State to) {
  switch (src) {
    case Modify: {
      m_count_ -= 1;
      break;
    }
    case Exclusive: {
      e_count_ -= 1;
      break;
    }
    case Shared: {
      s_count_ -= 1;
      break;
    }
    case Invalid: {
      i_count_ -= 1;
      break;
    }
    default: {
      cout << "Error at change state" << endl;
      exit(0);
    }
  }
  switch (to) {
    case Modify: {
      m_count_ += 1;
      break;
    }
    case Exclusive: {
      e_count_ += 1;
      break;
    }
    case Shared: {
      s_count_ += 1;
      break;
    }
    case Invalid: {
      i_count_ += 1;
      break;
    }
    default: {
      cout << "Error at change state" << endl;
      exit(0);
    }
  }
}

bool CacheLine::read(int coreIndex, int *ret) {
  assert(coreIndex >= 0);
  assert(coreIndex < CORE_NUM);

  // Cache hit
  if ( state_[coreIndex] == Modify ) {
    *ret = data_[coreIndex];
    return true;
  }

  if ( state_[coreIndex] == Exclusive ) {
    *ret = data_[coreIndex];
    return true;
  }

  if ( state_[coreIndex] == Shared ) {
    *ret = data_[coreIndex];
    return true;
  }

  // Cache Miss at current core Cache Line
  // state == Invalid
  assert ( state_[coreIndex] == Invalid );
  // broadcast on bus
  for (int otherCoreIndex = 0; otherCoreIndex < CORE_NUM; ++ otherCoreIndex) {
    if ( otherCoreIndex == coreIndex ){
      continue;
    }
    // Modify case:
    // in other core state change: M -> write back -> S
    // in current core state : I -> write allocate -> S
    if ( state_[otherCoreIndex] == Modify ) {
      mem_data_ = data_[otherCoreIndex]; // write back
      state_[otherCoreIndex] = Shared;  // M -> S
      stateChange(Modify, Shared);

      data_[coreIndex] = mem_data_; // write allocate
      state_[coreIndex] = Shared; // I -> S
      stateChange(Invalid, Shared);

      *ret = data_[coreIndex];
      check();
      return true;
    }

    // Exclusive case:
    // in other core state change: E -> S
    // in current core state change: I -> S
    if ( state_[otherCoreIndex] == Exclusive ) {
      state_[otherCoreIndex] = Shared; // E -> S
      stateChange(Exclusive, Shared);

      // current core
      state_[coreIndex] = Shared; // I -> S
      data_[coreIndex] = data_[otherCoreIndex]; // get data by bus
      stateChange(Invalid, Shared);

      *ret = data_[coreIndex];
      check();
      return true;
    }

    // Shared case:
    // current core state change: I -> S
    if ( state_[otherCoreIndex] == Shared ) {
      state_[coreIndex] = Shared; // I -> S
      data_[coreIndex] = data_[otherCoreIndex]; // get data by bus
      stateChange(Invalid, Shared);

      *ret = data_[coreIndex];
      check();
      return true;
    }
  }

  // Cache Miss: all the core are Invalid
  // current core state change: I -> E
  data_[coreIndex] = mem_data_; // write allocate
  state_[coreIndex] = Exclusive;
  stateChange(Invalid, Exclusive);
  *ret = data_[coreIndex];
  check();
  return false;
}

bool CacheLine::write(int coreIndex, int value) {
  assert( coreIndex >= 0 );
  assert( coreIndex < CORE_NUM );

  // Cache hit
  if ( state_[coreIndex] == Modify ) {
    // Modify didn't change state
    data_[coreIndex] = value;

    check();
    return true;
  }

  if ( state_[coreIndex] == Exclusive ) {
    data_[coreIndex] = value;
    state_[coreIndex] = Modify; // E -> M
    stateChange(Exclusive, Modify);
    check();
    return true;
  }


  if ( state_[coreIndex] == Shared ) {
    // broadcast other core state to Invalid
    for (int otherCoreIndex = 0; otherCoreIndex < CORE_NUM; ++ otherCoreIndex) {
      if ( otherCoreIndex == coreIndex ) {
        continue;
      }
      if ( state_[otherCoreIndex] == Shared ){
        state_[otherCoreIndex] = Invalid; // S -> I
        stateChange(Shared, Invalid);
      }
    }

    // S -> M
    state_[coreIndex] = Modify;
    data_[coreIndex] = value;
    stateChange(Shared, Modify);
    check();
    return true;
  }

  // Cache Miss at current core
  if ( state_[coreIndex] == Invalid ) {
    // broadcast to get the other core state
    for (int otherCoreIndex = 0; otherCoreIndex < CORE_NUM; ++ otherCoreIndex) {
      if ( otherCoreIndex == coreIndex ) {
        continue;
      }

      // case Modify:
      // in other core state change: M -> write back -> I
      // in current core state change: I -> write allocate -> E -> M
      // TODO: MESI protocol may has some diff?
      if ( state_[otherCoreIndex] == Modify ) {
        mem_data_ = data_[otherCoreIndex]; // write back
        state_[otherCoreIndex] = Invalid; // M -> I
        stateChange(Modify, Invalid);

        data_[coreIndex] = mem_data_; // write allocate
        state_[coreIndex] = Exclusive; // I -> E
        stateChange(Invalid, Exclusive);

        data_[coreIndex] = value; // write to cache
        state_[coreIndex] = Modify; // E -> M
        stateChange(Exclusive, Modify);

        check();
        return true;
      }

      // case Exclusive:
      // in other core state change: E -> I
      // in current core state change: I -> E -> M
      // TODO: MESI protocol may has some diff ?
      if ( state_[otherCoreIndex] == Exclusive ) {
        state_[otherCoreIndex] = Invalid; // E -> I
        stateChange(Exclusive, Invalid);

        data_[coreIndex] = mem_data_; // write allocate
        state_[coreIndex] = Exclusive; // I -> E
        stateChange(Invalid, Exclusive);
        data_[coreIndex] = value;
        state_[coreIndex] = Modify; // E -> M
        stateChange(Exclusive, Modify);

        check();
        return true;
      }

      // case Shared:
      // need to change all others core state: S -> I
      if ( state_[otherCoreIndex] == Shared ) {
        state_[otherCoreIndex] = Invalid;
        stateChange(Shared, Invalid);
      }
    }
    // case Shared and Invalid:
    // in others case state change: S -> I
    // in current case state change: I -> M
    state_[coreIndex] = Modify;
    data_[coreIndex] = value;
    stateChange(Invalid, Modify);

    check();
    return true;
  }
  // Error!
  cout << "Error at write end!" << endl;
  exit(0);
  return false;
}

void CacheLine::check() {
  if ( (m_count_ == 1 && i_count_ == CORE_NUM - 1)
    || (e_count_ == 1 && i_count_ == CORE_NUM - 1)
    || (s_count_ >= 2 && i_count_ == CORE_NUM - s_count_)
    || (i_count_ == 4) ){

    if (debug_) {
      cout << "m_count: " << m_count_ << " ";
      cout << "e_count: " << e_count_ << " ";
      cout << "s_count: " << s_count_ << " ";
      cout << "i_count: " << i_count_ << endl;
    }
  }else {
    cout << "Last Error state: " << endl;
    if (debug_) {
      cout << "m_count: " << m_count_ << " ";
      cout << "e_count: " << e_count_ << " ";
      cout << "s_count: " << s_count_ << " ";
      cout << "i_count: " << i_count_ << endl;
    }
    exit(0);
  }
}
```

测试代码：

```C++
/* main.cc */
#include <iostream>
#include "MESI.h"

using std::cout;
using std::endl;

int main() {
 srand(1111);
 CacheLine cache;
 int lastValue = 0; // memory initialization value is 0
 for (int i=0; i<1000000; i++) {
   int coreIndex = rand() % CORE_NUM;
   int value = rand() % 1000;
   int op = rand() % 2;
   if ( op == 0 ) { // read
      cache.read(coreIndex, &value);
      printf("loop[%d]: read data from core[%d]: %d\n", i, coreIndex, value);
      if (lastValue != value) { // check consistency
        printf("Error at lastValue[%d] -> got value[%d]\n", lastValue, value);
        exit(0);
      }
   }else if ( op == 1 ) { // write
     cache.write(coreIndex, value);
     lastValue = value;
     printf("loop[%d]: write data from core[%d]: %d\n", i, coreIndex, value);
   }
 }
 cout << "ALL TEST OK" << endl;
 cout << "PASS" << endl;
}
```
## 0x03 消息机制

// TODO: update....

MESI中的6种消息

* Read
    读取请求，包括要读取的主存地址

* Read Respond
    读取请求的响应，可能来自主存或者是其他CPU，其中包含了请求的数据，并且收到此请求的CPU会将数据放到Cache Line中。
    
* Invalidate
    其中包含一个失效的主存地址，收到此消息的CPU将在对应的Cache Line中移除这个主存地址的数据，并且回复Invalidate Acknowledge。
    
* Invalidate Acknowledge
    收到Invalidate后移除对应Cache Line后响应。
    
* Read Invalidate
    Read和Invalidate两者结合
    
* WriteBack
    