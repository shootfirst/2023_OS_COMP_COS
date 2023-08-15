### 如果要实现agent，哪些部分代码是必要的

从`scheduler`目录下可以看出，实现agent需要有三个文件：`XXX_agent.cc`、`XXX_scheduler.h`、`XXX_scheduler.cc`。下面分别提取下这三个文件中哪些部分是共有的、必须实现的。

#### XXX_agent.cc

##### ParseAgentConfig

在`agen.cc`中，我们一般需要实现一个提取命令参数、初始化config的函数`ParseAgentConfig`。其一般形式为：

```c++
namespace ghost {
    
ABSL_FLAG(std::string, ghost_cpus, "1-5", "cpulist");
ABSL_FLAG(std::string, enclave, "", "Connect to preexisting enclave directory");

static void ParseAgentConfig(AgentConfig* config) {
  // 从命令中获取使用的cpu范围
  CpuList ghost_cpus =
      MachineTopology()->ParseCpuStr(absl::GetFlag(FLAGS_ghost_cpus));
  CHECK(!ghost_cpus.Empty());
  config->cpus_ = ghost_cpus;

  // 获取机器的拓扑参数
  Topology* topology = MachineTopology();
  config->topology_ = topology;

  // 获取enclave。这段是必要的
  std::string enclave = absl::GetFlag(FLAGS_enclave);
  if (!enclave.empty()) {
    int fd = open(enclave.c_str(), O_PATH);
    CHECK_GE(fd, 0);
    config->enclave_fd_ = fd;
  }
}

}  // namespace ghost
```

可以看到，它大概是用了absl的一个提取工具。命令参数一般至少有cpu list（如命令`bazel-bin/fifo_per_cpu_agent --ghost_cpus 0-1`），如果自定义了其他命令参数，照着它加上即可。如：

```c++
// in cfs_agent.cc
// Scheduling tuneables
ABSL_FLAG(
    absl::Duration, min_granularity, absl::Milliseconds(1),
    "The minimum time a task will run before being preempted by another task");
static void ParseAgentConfig(CfsConfig* config) {
  // ...
  config->min_granularity_ = absl::GetFlag(FLAGS_min_granularity);
}
```

##### main

差不多每个agent都是这种写法，修改`AgentProcess`的参数即可。

```c++
int main(int argc, char* argv[]) {
  absl::InitializeSymbolizer(argv[0]);
  absl::ParseCommandLine(argc, argv);

  ghost::AgentConfig config;
  ghost::ParseAgentConfig(&config);

  printf("Initializing...\n");

  // Using new so we can destruct the object before printing Done
  auto uap = new ghost::AgentProcess<ghost::FullFifoAgent<ghost::LocalEnclave>,
                                     ghost::AgentConfig>(config);

  ghost::GhostHelper()->InitCore();
  printf("Initialization complete, ghOSt active.\n");
  fflush(stdout);

  // 这个SIGINT是Linux自带的，表示^C信号。这里是在说收到^C就立即停止
  ghost::Notification exit;
  ghost::GhostSignals::AddHandler(SIGINT, [&exit](int) {
    static bool first = true;  // We only modify the first SIGINT.

    if (first) {
      exit.Notify();
      first = false;
      return false;  // We'll exit on subsequent SIGTERMs.
    }
    return true;
  });

  ghost::GhostSignals::AddHandler(SIGUSR1, [uap](int) {
    uap->Rpc(ghost::FifoScheduler::kDebugRunqueue);
    return false;
  });

  exit.WaitForNotification();

  delete uap;

  printf("\nDone!\n");

  return 0;
}
```

#### XXX_scheduler.h

需要定义以下几个类：

##### XXXTask

task一般至少会记录两个参数：`run_state`、`cpu`，其他的都由调度算法自己决定，比如cfs还会记录nice和weight。

###### run_state

其一般有一个对应的枚举类，该枚举类可以选择定义在`XXXTask`类外部，或者内部。

在task中，一般会为其提供类似这样的getter/setter方法：

```c++
  bool blocked() const { return run_state == RunState::kBlocked; }
  bool queued() const { return run_state == RunState::kQueued; }
  bool runnable() const { return run_state == RunState::kRunnable; }
  bool oncpu() const { return run_state == RunState::kOnCpu; }
  bool yielding() const { return run_state == RunState::kYielding; }
  void SetState(State state) { ... }
```

##### XXXScheduler

###### 消息处理函数

```c++
  // Callbacks to IPC messages delivered by the kernel against `task`.
  // Implementations typically will advance the task's state machine and adjust
  // the runqueue(s) as needed.
  virtual void TaskNew(TaskType* task, const Message& msg) = 0;
  virtual void TaskRunnable(TaskType* task, const Message& msg) = 0;
  virtual void TaskDead(TaskType* task, const Message& msg) = 0;
  virtual void TaskDeparted(TaskType* task, const Message& msg) = 0;
  virtual void TaskYield(TaskType* task, const Message& msg) = 0;
  virtual void TaskBlocked(TaskType* task, const Message& msg) = 0;
  virtual void TaskPreempted(TaskType* task, const Message& msg) = 0;
  virtual void TaskSwitchto(TaskType* task, const Message& msg) {}
  virtual void TaskAffinityChanged(TaskType* task, const Message& msg) {}
  virtual void TaskPriorityChanged(TaskType* task, const Message& msg) {}
  virtual void TaskOnCpu(TaskType* task, const Message& msg) {}

  virtual void TaskDiscovered(TaskType* task) {}
  virtual void DiscoveryStart() {}
  virtual void DiscoveryComplete() {}

  virtual void CpuTick(const Message& msg) {}
  virtual void CpuNotIdle(const Message& msg) {}
  virtual void CpuTimerExpired(const Message& msg) {}
  virtual void CpuAvailable(const Message& msg) {}
  virtual void CpuBusy(const Message& msg) {}
  virtual void AgentBlocked(const Message& msg) {}
  virtual void AgentWakeup(const Message& msg) {}
```

###### 主要逻辑

```c++
  // 调度算法实现主要逻辑
  void Schedule(const Cpu& cpu, const StatusWord& sw);
  // 在enclave->Ready()中被调用，用于给每个cpu绑定对应的agent
  // 可能有一个疑问就是，enclave->Ready()在`StartAgentTasks()`之后被调用
  // 之前在`StartAgentTasks()`中不是已经绑过一次了吗？
  // 答案是这个绑定需要是双向的，StartAgentTasks只绑了agent->cpu，这里要绑的是agent<-cpu
  void EnclaveReady() {
      for (const Cpu& cpu : cpus()) {
        CpuState* cs = cpu_state(cpu);
        Agent* agent = enclave()->GetAgent(cpu);

        // 如果是centralized模型就不用这一步
        while (!cs->channel->AssociateTask(agent->gtid(), agent->barrier(),
                                           /*status=*/nullptr)) {
          CHECK_EQ(errno, ESTALE);
        }
      }
  }
  // 这个应该是消息队列
  Channel& GetDefaultChannel() final { return unused_channel_; };
  Channel* default_channel_ = nullptr;
  // 如果当前scheduler中没有待调度任务，则返回true
  bool Empty(const Cpu& cpu) { ... }
```

###### 管理CPU state

```c++
  struct CpuState { ... }
  inline CpuState* cpu_state(const Cpu& cpu) { return &cpu_states_[cpu.id()]; }
  inline CpuState* cpu_state_of(const FifoTask* task) { 
    CHECK_GE(task->cpu, 0);
    CHECK_LT(task->cpu, MAX_CPUS);
    return &cpu_states_[task->cpu];
  }

  CpuState cpu_states_[MAX_CPUS];
```

###### debug

```c++
  void DumpState(const Cpu& cpu, int flags) final;
  std::atomic<bool> debug_runqueue_ = false;
  static constexpr int kDebugRunqueue = 1;

  int CountAllTasks() {
    int num_tasks = 0;
    allocator()->ForEachTask([&num_tasks](Gtid gtid, const CfsTask* task) {
      ++num_tasks;
      return true;
    });
    return num_tasks;
  }
  static constexpr int kCountAllTasks = 2;
```

##### XXXAgent

格式非常统一。最主要要实现`AgentThread`

```c++
class XXXAgent : public LocalAgent {
 public:
  XXXAgent(Enclave* enclave, Cpu cpu, XXXScheduler* scheduler)
      : LocalAgent(enclave, cpu), scheduler_(scheduler) {}

  void AgentThread() override;
  Scheduler* AgentScheduler() const override { return scheduler_; }

 private:
  XXXScheduler* scheduler_;
};
```

##### FullXXXAgent

格式非常统一

```c++
template <class EnclaveType>
class FullXXXAgent : public FullAgent<EnclaveType> {
 public:
  explicit FullXXXAgent(AgentConfig config) : FullAgent<EnclaveType>(config) {
    scheduler_ =
        MultiThreadedXXXScheduler(&this->enclave_, *this->enclave_.cpus()/* ... */);
    this->StartAgentTasks();
    this->enclave_.Ready();
  }

  ~FullXXXAgent() override {
    this->TerminateAgentTasks();
  }

  std::unique_ptr<Agent> MakeAgent(const Cpu& cpu) override {
    return std::make_unique<XXXAgent>(&this->enclave_, cpu, scheduler_.get());
  }

  // 应该是用于debug
  void RpcHandler(int64_t req, const AgentRpcArgs& args,
                  AgentRpcResponse& response) override {
    switch (req) {
      case XXXScheduler::kDebugRunqueue:
        scheduler_->debug_runqueue_ = true;
        response.response_code = 0;
        return;
      case XXXScheduler::kCountAllTasks:
        response.response_code = scheduler_->CountAllTasks();
        return;
      default:
        response.response_code = -1;
        return;
    }
  }

 private:
  std::unique_ptr<FifoScheduler> scheduler_;
};
```

##### XXXConfig

按需要重写`AgentConfig`即可。

#### XXX_scheduler.cc

##### 调度

###### per-CPU

```c++
// 忽略了debug用代码
void CfsAgent::AgentThread() {
  SignalReady();
  WaitForEnclaveReady();

  while (!Finished() || !scheduler_->Empty(cpu())) {
    scheduler_->Schedule(cpu(), status_word());
  }
}
```

```c++
// 这个实现似乎类似集中型的agent
void FifoScheduler::Schedule(const Cpu& cpu, const StatusWord& agent_sw) {
  BarrierToken agent_barrier = agent_sw.barrier();
  CpuState* cs = cpu_state(cpu);

  Message msg;
  while (!(msg = Peek(cs->channel.get())).empty()) {
    DispatchMessage(msg);
    Consume(cs->channel.get(), msg);
  }

  // 转交到具体的调度策略实现
  FifoSchedule(cpu, agent_barrier, agent_sw.boosted_priority());
}
```

```c++
void CfsScheduler::CfsSchedule(const Cpu& cpu, BarrierToken agent_barrier,
                               bool prio_boost) {
  RunRequest* req = enclave()->GetRunRequest(cpu);
  CpuState* cs = cpu_state(cpu);

  // ... 总之通过一系列办法，获取到了下一个要执行的task  next
  cs->current = next;

  if (next) {
	// 事务提交
    req->Open({
        .target = next->gtid,
        .target_barrier = next->seqnum,
        .agent_barrier = agent_barrier,
        .commit_flags = COMMIT_AT_TXN_COMMIT,
    });
    while (next->status_word.on_cpu()) {
      Pause();
    }
    if (req->Commit()) {
      // TODO 处理事务提交成功情况
    } else {
      // TODO 处理事务提交失败情况
    }
  } else {
    req->LocalYield(agent_barrier, 0);
  }
}
```

###### centralized

参考`centralized/fifo_scheduler.cc`实现，不多赘述

##### 消息处理函数

挨个实现就行