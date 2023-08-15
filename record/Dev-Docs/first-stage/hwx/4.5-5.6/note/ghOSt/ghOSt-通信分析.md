### 通信

根据论文，kernel2agent采用的是status words+mq通信，agent2kernel采用的是提交事务。后者没什么好说的，本质上是通过`ioctl`系统调用。下面重点说说前者

#### 消息队列

##### 数据结构

###### 消息

消息的数据结构：

`in kernel/ghost_uapi.h`

* 父结构

  ```c++
  struct ghost_msg { /* 我觉得没什么好说的 */ };
  ```

* 子结构（们）

  ```c++
  struct ghost_msg_payload_task_new { /* 我觉得没什么好说的 */ };
  ```

###### 消息队列

```c++
// in lib/channel.h 
class Channel { /* 我觉得没什么好说的 */ };
```

##### 基本实现

agent开启之后的主要逻辑是执行这个：

```c++
// in per_cpu/fifo_scheduler.cc
void FifoAgent::AgentThread() {
  SignalReady();
  WaitForEnclaveReady();

  // 当scheduler中没有未调度线程时，并且没有finish时退出循环。
  // 也就是说这个差不多就是个死循环啦。
  while (!Finished() || !scheduler_->Empty(cpu())) {
    scheduler_->Schedule(cpu(), status_word());
  }
}
```

负责消息通信的，主要是`FifoScheduler::Schedule`：

```c++
// in per_cpu/fifo_scheduler.cc
void FifoScheduler::Schedule(const Cpu& cpu, const StatusWord& agent_sw) {
  // 获取status words
  BarrierToken agent_barrier = agent_sw.barrier();
  CpuState* cs = cpu_state(cpu);

  // 从消息队列中取消息
  Message msg;
  while (!(msg = Peek(cs->channel.get())).empty()) {
    // 将消息分发给不同的处理函数
    DispatchMessage(msg);
    Consume(cs->channel.get(), msg);
  }

  // 执行真正的决策逻辑
  FifoSchedule(cpu, agent_barrier, agent_sw.boosted_priority());
  // 当agent被唤醒，就会返回到这里，然后之后再进行一轮Schedule的调用
}
```

可以看到，在`FifoScheduler::Schedule`中，我们是通过while循环来不断从消息队列中获取消息的。获取完信息后，我们会通过dispatch，这个应该能把消息发送给不同的function进行处理（类似这种作用）。获取完所有消息之后，我们就调用`FifoScheduler::FifoSchedule`。

可以先来看看`BasicDispatchScheduler<TaskType>::DispatchMessage`。可以看到，`DispatchMessage`实际上就是通过switch case来调用不同类型消息的处理函数。

```c++
template <typename TaskType>
void BasicDispatchScheduler<TaskType>::DispatchMessage(const Message& msg) {
  // CPU messages.
  if (msg.is_cpu_msg()) {
    switch (msg.type()) {
      case MSG_CPU_TICK:
        CpuTick(msg);
        break;
      // ...
    }
    return;
  }

  // Task messages.

  Gtid gtid = msg.gtid();
  TaskType* task;
  // 如果该msg意思是新建任务
  if (ABSL_PREDICT_FALSE(msg.type() == MSG_TASK_NEW)) {
    // 这里的转化颇有协议栈处理数据包的那种意思
    const ghost_msg_payload_task_new* payload =
        static_cast<const ghost_msg_payload_task_new*>(msg.payload());
    bool allocated;
    // 这个tie的用法值得一学，应该是用来实现批量赋值的
    // GetTask：Returns the Task* associated with `gtid` if the `gtid` is already
    // known to the allocator and nullptr otherwise.
    // 也就是说message中是通过gtid来确认task的。所以也许全局需要保存一张tid-task表
    std::tie(task, allocated) = allocator()->GetTask(gtid, payload->sw_info);
    if (ABSL_PREDICT_FALSE(!allocated)) {
      // allocated为true，说明要新建的任务已经存在了，异常
      return;
    }
  } else {
    task = allocator()->GetTask(gtid);
  }

  // 是否应该upadate seq
  bool update_seqnum = true;

  // Above asserts we've missed no messages, state below should be consistent.
  switch (msg.type()) {
    case MSG_NOP:
      GHOST_ERROR("Saw MSG_NOP! e=%d l=%d\n", msg.empty(), msg.length());
      break;
    case MSG_TASK_NEW:
      TaskNew(task, msg);
      update_seqnum = false;  // `TaskNew()` initializes sequence number.
      break;
    // ...
  }

  if (ABSL_PREDICT_TRUE(update_seqnum)) {
    task->Advance(msg.seqnum());
  }
}
```

挑一个处理函数出来看看：

```c++
// in per_cpu/fifo_scheduler.cc
void FifoScheduler::TaskNew(FifoTask* task, const Message& msg) {
  const ghost_msg_payload_task_new* payload =
      static_cast<const ghost_msg_payload_task_new*>(msg.payload());

  // 初始化Tseq
  task->seqnum = msg.seqnum();
  // kBlocked：当前task并未加入runqueue
  task->run_state = FifoTaskState::kBlocked;

  if (payload->runnable) {
    task->run_state = FifoTaskState::kRunnable;
    Cpu cpu = AssignCpu(task);
    // 将task迁移到cpu上，并且通知该cpu上的agent
    Migrate(task, cpu, msg.seqnum());
  } else {
    // 这里没大看懂
    // Wait until task becomes runnable to avoid race between migration
    // and MSG_TASK_WAKEUP showing up on the default channel.
  }
}
```

```c++
// in per_cpu/fifo_scheduler.cc
// 将task迁移到cpu上，并且通知该cpu上的agent
void FifoScheduler::Migrate(FifoTask* task, Cpu cpu, BarrierToken seqnum) {
  CHECK_EQ(task->run_state, FifoTaskState::kRunnable);
  CHECK_EQ(task->cpu, -1);

  CpuState* cs = cpu_state(cpu);
  const Channel* channel = cs->channel.get();
  // 将agent对应的消息队列绑定到task上
  CHECK(channel->AssociateTask(task->gtid, seqnum, /*status=*/nullptr));

  task->cpu = cpu.id();

  cs->run_queue.Enqueue(task);

  // Get the agent's attention so it notices the new task.
  enclave()->GetAgent(cpu)->Ping();
}
```

##### 当agent正在沉睡

论文中提到，per-CPU model中，agent会在做出调度决策后，将CPU让给target task。当target task执行完毕，或者内核向消息队列中塞进新消息的时候，agent就会被唤醒，抢走CPU的所有权。我们可以来看看这个过程是怎么实现的。

agent沉睡时，内核向消息队列塞进新消息时，内核会唤醒agent，使得agent执行到`DispatchMessage()`处。

`DispatchMessage()`将消息分发到正确的处理函数处，在此假设为`TaskNew`。如果逻辑正确的话（也即确实有了新的task），`TaskNew`中就会调用`Migrate`，继而调用`Ping`(内部也是通过`ioctl`实现的)，继而将agent唤醒：

```c++
void FifoScheduler::TaskNew(FifoTask* task, const Message& msg) {
    // ...
    Migrate(task, cpu, msg.seqnum());
}
```

```c++
// 将task迁移到cpu上，并且通知该cpu上的agent
void FifoScheduler::Migrate(FifoTask* task, Cpu cpu, BarrierToken seqnum) {
  // ...
  // Get the agent's attention so it notices the new task.
  enclave()->GetAgent(cpu)->Ping();
}
```

agent被唤醒后，就会来到这里：

```c++
// in fifo_scheduler.cc
void FifoScheduler::Schedule(const Cpu& cpu, const StatusWord& agent_sw) {
  // ...
  FifoSchedule(cpu, agent_barrier, agent_sw.boosted_priority());
  // 当agent被唤醒，就会返回到这里，然后之后再进行一轮Schedule的调用
}
```

之后就再次进入调度循环，perfect。



#### seq num

##### Aseq

论文中介绍了通过seq num来确保同步的方法。回忆一下其步骤：

1)Read Aseq

2)读取msq

3)决定调度策略

4)commit，若commit的最新Aseq比内核观测到的最新Aseq小，那么commit失败

###### 数据结构

首先看下seq num的数据结构：

```c++
class LocalAgent : public Agent {
 public:
  LocalAgent(Enclave* enclave, const Cpu& cpu) : Agent(enclave, cpu) {}
  ~LocalAgent() override { status_word_.Free(); }

  const StatusWord& status_word() const override { return status_word_; }

 private:
  void ThreadBody() override;

  LocalStatusWord status_word_;
};
```

在`Agent`的字段`status_word_`中，存储着seq num（`BarrierToken`类型）：

```c++
// StatusWord represents an accessor for our kernel shared sequence points.  A
// StatusWord is a movable type representing the mapping for a single thread.
// It releases the word in its destructor.
class StatusWord { // 这里并没有放全代码。StatusWord里面还会存放其他很多东西，如进程状态CPU状态等
 public:
  // Returns a 'Null' barrier token, for call-sites where it is not required.
  static BarrierToken NullBarrierToken() { return 0; }

  // All methods below are invalid on an empty status word.
  BarrierToken barrier() const { return sw_barrier(); }

  bool stale(BarrierToken prev) const { return prev == barrier(); }

 protected:
  uint32_t sw_barrier() const {
    std::atomic<uint32_t>* barrier =
        reinterpret_cast<std::atomic<uint32_t>*>(&sw_->barrier);
    return barrier->load(std::memory_order_acquire);
  }

  ghost_status_word* sw_ = nullptr;
};
```

可以从方法`barrier()`中的`sw_barrier`看出，BarrierToken实质上就是uint32_t。它没有加上atomic包装，是因为自从我们从sw中将其读出来开始，它就只作为一个数据快照了，应该不会对它进行修改。

###### 步骤追踪

1. Read Aseq

   ```c++
   void FifoScheduler::Schedule(const Cpu& cpu, const StatusWord& agent_sw) {
     BarrierToken agent_barrier = agent_sw.barrier();
   ```

2. 读取msq

   ```c++
     Message msg;
     while (!(msg = Peek(cs->channel.get())).empty()) {
       DispatchMessage(msg);
       Consume(cs->channel.get(), msg);
     }
   ```

3. 决定调度策略

   ```c++
     FifoSchedule(cpu, agent_barrier, agent_sw.boosted_priority());
   }
   ```

4. commit

   ```c++
       req->Open({
           .target = next->gtid,
           .target_barrier = next->seqnum,
           .agent_barrier = agent_barrier, // 在这里commit了
           .commit_flags = COMMIT_AT_TXN_COMMIT,
       });
   
       if (req->Commit()) {
   ```

步骤非常完美非常标准（）



##### Tseq

论文中介绍了通过seq num来确保同步的方法，但其步骤并不像Aseq那样清晰（虽然也不难就是了）。因而下面，我们将通过对其生命周期的追踪来探究它的具体实现。

###### 数据结构

每一个task都有属于自己的Tseq：

```c++
struct Task {
  // ...
  // 用参数的next_seqnum来更新当前的seqnum
  inline void Advance(uint32_t next_seqnum) {
    CHECK_GT(next_seqnum, seqnum);
    seqnum.store(next_seqnum);
  }

  Seqnum seqnum;
};
```

```c++
class Seqnum {
 public:
  // Loads the sequence number without memory ordering guarantee.
  operator uint32_t() const {
    return seqnum_.load(std::memory_order_relaxed);
  }

  // Stores the sequence number without memory ordering guarantee.
  Seqnum& operator=(uint32_t seqnum) {
    seqnum_.store(seqnum, std::memory_order_relaxed);
    return *this;
  }

  // Loads the sequence number with acquire semantics.
  uint32_t load() {
    return seqnum_.load(std::memory_order_acquire);
  }

  // Stores the sequence number with release semantics.
  void store(uint32_t seqnum) {
    seqnum_.store(seqnum, std::memory_order_release);
  }

 private:
  std::atomic<uint32_t> seqnum_{0};
};
```

其实也就是对`atomic<uint32_t> seqnum_`进行了小包装。在centralized model中，同一时刻可能有多个线程、多个CPU向同一条消息队列发送消息，这时候就需要抢夺增加Aseq，因而需要对其进行并发安全保证。

###### 生命周期

当一个task被new出来时，它的Tseq会被赋初值为msg从内核中携带而来的seqnum值【这里就颇有网络通信的感觉了】：

```c++
  bool update_seqnum = true;
  case MSG_TASK_NEW:
	  TaskNew(task, msg);
      update_seqnum = false;  // `TaskNew()` initializes sequence number.
      break;

  if (ABSL_PREDICT_TRUE(update_seqnum)) {
    task->Advance(msg.seqnum());
  }
```

```c++
void FifoScheduler::TaskNew(FifoTask* task, const Message& msg) {
  // ...
  task->seqnum = msg.seqnum();
  // ...
}
```

随后，每次有该task的消息进入消息队列，都会更新一次Tseq：

```c++
  if (ABSL_PREDICT_TRUE(update_seqnum)) {
    task->Advance(msg.seqnum());
  }
```

```c++
  inline void Advance(uint32_t next_seqnum) {
    // Note: next_seqnum could be greater than seqnum + 1 if BPF has consumed
    // messages. We only assert that events have not been reordered with this
    // check.
    CHECK_GT(next_seqnum, seqnum);
    seqnum.store(next_seqnum);
  }
```

不同于Aseq，Tseq会进行两次校验：【其实第二次会不会校验我还不确定2333但第一次肯定是会的】

1. 分配任务时

   如果agent中的Tseq与内核中读取到的Tseq不一致，说明还有消息没读到，所以该task需要先暂时yield，等待下一次schedule迭代将消息全部读入后再进行调用。

   注意，global agent并没有消息通知机制！所以这里只能是等一轮循环，等下次循环时global agent进行消息的读取

   ```c++
       FifoTask* next = Dequeue();
   
       // If `next->seqnum != next->status_word.barrier()` is true, then there are
       // pending messages for `next` that we have not read yet. Thus, do not
       // schedule `next` since we need to read the messages. We will schedule
       // `next` in a future iteration of the global scheduling loop.
       if (next->seqnum != next->status_word.barrier()) {
         Yield(next);
         continue;
       }
   ```

2. 提交事务时

   会在Schedule中被提交给内核，由内核进行校验：

   ```c++
     while (!available.Empty()) {
       FifoTask* next = Dequeue();
   	// ...
       RunRequest* req = enclave()->GetRunRequest(next_cpu);
       req->Open({.target = next->gtid,
                  .target_barrier = next->seqnum, // 在这里！
                  .commit_flags = COMMIT_AT_TXN_COMMIT});
     }
   ```

