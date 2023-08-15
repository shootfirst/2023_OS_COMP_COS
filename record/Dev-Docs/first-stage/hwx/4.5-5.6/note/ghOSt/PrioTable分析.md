1. sched_item数组长度
2. 

### 前言

之前说到可以分为Agent端和Client端，Agent端跑agent，Client端跑如simple_text、experiment之类的application。

ghost的kernel和userspace间会通信，同样的，Agent端和Client端也需要进行通信。

Client端可以通过通信，向Agent端传递消息，来：

1. 控制调度决策。

   比如说，它可以设置一个线程状态从running变成idle；（EDF算法）它可以设置调整线程的ddl；它还可以改变QoS，从而影响调度决策。

2. 获取线程状态信息。


ghost为了支持这种通信，提出了共享内存的方法。

Agent和Client共享一片内存区域，这片区域会记录当前agent管理的所有ghost线程的状态。

1. Client可以通过内存区域设置线程状态

   比如下面的例子，就将线程设置为idle状态

   ```c++
   mark_sched_item_idle(table_, start_idx + i);
   ```

2. Agent会在`AgentProccess`中首先查看内存区域，更新任务的状态，再进行调度决策

   ```c++
         // 首先从共享内存获取信息更新状态
   	  global_scheduler_->UpdateSchedParams();
   	  // 再进行调度决策
         global_scheduler_->GlobalSchedule(status_word(), agent_barrier);
   ```

这样一来，就可以实现Client对调度策略的控制。



### PrioTable

`PrioTable`正是实现了这片共享内存区域。

#### 结构

它本质上可以看作一片连续的物理内存。其结构如下：

```c++
/* 
PrioTable:
 Header | sched_item数组 | work_class数组 | stream 
↑	    ↑               ↑                ↑
hdr_   hdr_+si_off   hdr_+wc_off      hdr_+st_off
*/
```

可以看到，它大致分为四个区域，其中hdr\_为指向连续物理内存开始的指针，由hdr\_+offset可以计算出各个区域的开始段。

##### header

记录了一些关键信息，比如说第二个区域包含了多少个sched_item，第三个区域包含了多少个work_class等等等，诸如此类。

```c++
struct ghost_shmem_hdr {
  uint16_t version; // 版本号
  uint16_t hdrlen;  // header长度
  uint32_t maplen;
  uint32_t si_num; /*sched item数 */
  uint32_t si_off; /* offset of 'sched_item[0]' from start of hdr */
  uint32_t wc_num; /* number of elements in 'work_class[]' array */
  uint32_t wc_off; /* offset of 'work_class[0]' from start of hdr */
  uint32_t st_cap; /* capacity of the stream */
  uint32_t st_off; /* offset of stream from start of hdr */
} *hdr_;
```

##### sched_item

（连蒙带猜）是thread / task这种东西的抽象。

```c++
struct sched_item {
  uint32_t sid;   /* 每个sched_item都有unique的sid */
  uint32_t wcid;  /* 这个对象属于哪个任务类（work class，下一个小标题说明） */
  uint64_t gpid;  /* google pid */
  uint32_t flags; /* 调度相关的flags，如Runnable，表明thread当前状态 */
  seqcount_t seqcount;/* 信号量锁，保证并发安全 */
  uint64_t deadline; /* 如果线程超过这个时间任务还没执行完，就得被杀死 */
} ABSL_CACHELINE_ALIGNED;

/* sched_item.flags */
#define SCHED_ITEM_RUNNABLE (1U << 0) /* worker thread is runnable */
```

在实际生产中，应该可以根据需要增删flag，从而控制更多状态。

##### work_class

（连蒙带猜）代表task（sched_item）的不同类别。

比如说，有一堆task，那就可以根据task执行的次数分成两个work class：

1. one-shot

   task只会被调度执行一次，那么task的work class就是one-shot

2. repeatable

   task会被调度执行很多次，那么task的work class就是repeatable

```c++
struct work_class { 
  uint32_t id;       /* work class的unique id，一般会叫wid */
  uint32_t flags;    /* 表示work class是什么种类，是one-shot还是repeatable */
  uint32_t qos;      /* quality of service for this work class */
  // 这个qos的概念我不大明白。
    
  uint64_t exectime; /* 用于EDF算法 */
  uint64_t period;   /* period in nsecs for repeating work */
} ABSL_CACHELINE_ALIGNED;
/* work_class.flags */
#define WORK_CLASS_ONESHOT (1U << 0)
#define WORK_CLASS_REPEATING (1U << 1)
```

在生产实践中，应该可以根据需要增删flag，建立自己的分类方法。

##### stream

这玩意我不知道干什么的，好像没看到它被用到过



#### API介绍

了解下以下这几个常用的就行了，不是重点，可以用到了再看看。

```c++
  // 功能：从prio table的第二块区域中，找到第i项sched_item
  // 注：可以看下其代码（很少很简单），这样就能本质上理解prio table的结构
  struct sched_item* sched_item(int i) const;
  // 功能：从prio table的第二块区域中，找到第i项sched_item
  struct work_class* work_class(int i) const;

  inline struct ghost_shmem_hdr* hdr() const { return hdr_; }// 返回header
  inline int NumSchedItems() { return hdr()->si_num; }// 返回存储的sched item数
  inline int NumWorkClasses() { return hdr()->wc_num; }// 返回存储的work class数

  // 说实话没看懂这俩什么意思
  void MarkUpdatedIndex(int idx, int num_retries);
  int NextUpdatedIndex();

  // 返回当前priotable的owner
  // shmem是指一块署名的共享内存区域，也就跟本部linux0.11 os实验实现的那个共享内存是同一个东西
  // shmem只是纯粹的表示一块内存，经PrioTable包装后才能变成进程通信的工具
  pid_t Owner() const { return shmem_ ? shmem_->Owner() : 0; }

  // 相当于将pid为remote的进程绑定到当前ProTable实例上,这个函数的作用就是初始化shmem。
  // 以下是推测，有待解决：
  // 似乎它意思是说，进程跟PrioTable一一对应？？
  // 不过因为实际中好像只能通过Orchestrator这一个进程访问PrioTable，
  // 所以一进程一PrioTable看起来确实挺合理的
  bool Attach(pid_t remote);
```

