# 类


## Task

#### FiFoTask(p)

```
enum class FifoTaskState {
  kBlocked,   // not on runqueue.
  kRunnable,  // transitory state:
              // 1. kBlocked->kRunnable->kQueued
              // 2. kQueued->kRunnable->kOnCpu
  kQueued,    // on runqueue.
  kOnCpu,     // running on cpu.
};
FifoTaskState run_state = FifoTaskState::kBlocked：可以看成线程状态
int cpu：运行的cpu
bool preempted：上一次执行被抢占了吗
bool prio_boost：TODO
```

#### FifoTask(c)

```
enum class RunState {
  kBlocked,
  kQueued,
  kRunnable,
  kOnCpu,
  kYielding,
};

RunState run_state
Cpu cpu
bool preempted = false;
bool prio_boost = false;
```

#### EdfTask

```
enum class RunState {
  kBlocked = 0,
  kQueued = 1,
  kOnCpu = 2,
  kYielding = 3,
  kPaused = 4,
};

RunState run_state
int cpu
int rq_pos：在runqueue中的下标
bool prio_boost

absl::Duration runtime：累计运行时间
absl::Duration elapsed_runtime：应计运行时间
bool preempted：上次是否被抢占
absl::Duration estimated_runtime：相应计划项目中的估计值，后来则是观察到的加权平均值
absl::Time deadline = absl::InfiniteFuture()：最晚完成的时间
absl::Time sched_deadline = absl::InfiniteFuture()：最晚被调度的时间
const struct SchedParams* sp：
```

#### CfsTask

```
int cpu
int nice
uint32_t weight
uint32_t inverse_weight
CpuList cpu_affinity：这个线程亲和哪些cpu
absl::Duration vruntime
uint64_t runtime_at_first_pick_ns
```


## Sheduler

#### FifoScheduler(p)
explain:继承BasicDispatchScheduler，体现FIFO调度策略,percpu

```
void Schedule(const Cpu& cpu, const StatusWord& agent_sw) // 作为调度算法的核心决策函数
void FifoSchedule(const Cpu& cpu, BarrierToken agent_barrier, bool prio_boost) // 进行fifo调度决策机理

class FifoRq {
  FifoTask* Dequeue()
  void Enqueue(FifoTask* task)
  std::deque<FifoTask*> rq_
};
struct CpuState {
  FifoTask* current
  std::unique_ptr<Channel> channel
  FifoRq run_queue
}
CpuState cpu_states_[MAX_CPUS]：所有cpu的状态
Channel* default_channel_
```

#### FifoScheduler(c)
explain:继承BasicDispatchScheduler，体现FIFO调度策略,centralized

```
GlobalSchedule(const StatusWord& agent_sw, BarrierToken agent_sw_last)：
Enqueue(FifoTask* task)
Dequeue()
GetGlobalCPUId()
PickNextGlobalCPU(BarrierToken agent_barrier, const Cpu& this_cpu)

struct CpuState {
  FifoTask* current
  const Agent* agent
  absl::Time last_commit;
}
CpuState cpu_states_[MAX_CPUS]：所有cpu的状态
int global_cpu_core_：全局agent所在的core
std::atomic<int32_t> global_cpu_：全局agent所在cpu
LocalChannel global_channel_
int num_tasks_
std::deque<FifoTask*> run_queue_：请求运行的线程
std::vector<FifoTask*> yielding_tasks_：yield的线程
```

#### EdfScheduler

```
struct CpuState {
  EdfTask* current = nullptr;
  EdfTask* next = nullptr;
  const Agent* agent = nullptr;
}
CpuState cpu_states_[MAX_CPUS]
std::atomic<int32_t> global_cpu_
LocalChannel global_channel_
int num_tasks_
bool in_discovery_
std::vector<EdfTask*> run_queue_;
std::vector<EdfTask*> yielding_tasks_;
absl::flat_hash_map<pid_t, std::unique_ptr<Orchestrator>> orchs_:TODO
```

#### CfsScheduler
和迁移相关均未写入：TODO

```
static constexpr int kMaxNice = 19
static constexpr int kMinNice = -20
static constexpr uint32_t kNiceToWeight[40] = {
    88761, 71755, 56483, 46273, 36291,  // -20 .. -16
    29154, 23254, 18705, 14949, 11916,  // -15 .. -11
    9548,  7620,  6100,  4904,  3906,   // -10 .. -6
    3121,  2501,  1991,  1586,  1277,   // -5 .. -1
    1024,  820,   655,   526,   423,    // 0 .. 4
    335,   272,   215,   172,   137,    // 5 .. 9
    110,   87,    70,    56,    45,     // 10 .. 14
    36,    29,    23,    18,    15      // 15 .. 19
}：nice值对应weight
static constexpr uint32_t kNiceToInverseWeight[40] = {
    48388,     59856,     76040,     92818,     118348,    // -20 .. -16
    147320,    184698,    229616,    287308,    360437,    // -15 .. -11
    449829,    563644,    704093,    875809,    1099582,   // -10 .. -6
    1376151,   1717300,   2157191,   2708050,   3363326,   // -5 .. -1
    4194304,   5237765,   6557202,   8165337,   10153587,  // 0 .. 4
    12820798,  15790321,  19976592,  24970740,  31350126,  // 5 .. 9
    39045157,  49367440,  61356676,  76695844,  95443717,  // 10 .. 14
    119304647, 148102320, 186737708, 238609294, 286331153  // 15 .. 19
}：2^32/weight
enum class CpuIdleType : uint32_t {
  kCpuIdle = 0,  // The CPU is idle
  kCpuNotIdle,   // The CPU is not idle
  kCpuNewlyIdle, // The CPU is going to be idle
  kNumCpuIdleType,
}

Schedule(const Cpu& cpu, const StatusWord& sw)
EnclaveReady()
GetDefaultChannel()
GetAgentChannel(const Cpu& cpu)
Empty(const Cpu& cpu)

class CfsRq {
  EnqueueTask(CfsTask* task) ABSL_EXCLUSIVE_LOCKS_REQUIRED(mu_)
  void PutPrevTask(CfsTask* task) ABSL_EXCLUSIVE_LOCKS_REQUIRED(mu_)
  void DequeueTask(CfsTask* task) ABSL_EXCLUSIVE_LOCKS_REQUIRED(mu_)
  CfsTask* LeftmostRqTask() const ABSL_EXCLUSIVE_LOCKS_REQUIRED(mu_)

  absl::Duration min_vruntime_
  absl::Duration min_preemption_granularity_
  absl::Duration latency_
  std::set<CfsTask*, decltype(&CfsTask::Less)> rq_：不用说，cfs的核心
  std::atomic<size_t> rq_size_
}

struct CpuState {
  CfsTask* current
  std::unique_ptr<Channel> channel
  CfsRq run_queue;
  CfsMq migration_queue：TODO
  bool preempt_curr：应该继续运行当前线程？
  int id
}

CpuState cpu_states_[MAX_CPUS];
Channel* default_channel_
absl::Duration min_granularity_
absl::Duration latency_
bool idle_load_balancing_;
```

## Agent

#### FifoAgent(p)

```
FifoScheduler* scheduler_
```

#### FifoAgent(c)

```
FifoScheduler* global_scheduler_
```

#### GlobalSatAgent

```
EdfScheduler* global_scheduler_
```

#### CfsAgent

```
CfsScheduler* scheduler_
```


## FullAgent

#### FullFifoAgent(p)

```
FullFifoAgent(AgentConfig config)：整个调度算法初始化的开始

std::unique_ptr<FifoScheduler> scheduler_
```

#### FullFifoAgent(c)

```
FullFifoAgent(FifoConfig config)：整个调度算法初始化的开始

std::unique_ptr<FifoScheduler> scheduler_
```

#### FullEdfAgent

```
GlobalEdfAgent(GlobalConfig config)：整个调度算法初始化的开始

std::unique_ptr<EdfScheduler> global_scheduler_
```

#### FullCfsAgent

```
FullCfsAgent(CfsConfig config)：整个调度算法初始化的开始

std::unique_ptr<CfsScheduler> scheduler_
```




















## 核心重写方法

#### FullAgent：构造方法

##### FullCfsAgent
explain：步骤大家一模一样，都是1：创建对应scheduler；2：StartAgentTasks()；enclave_.Ready()

#### Agent：AgentThread
explain：步骤其实差不多，percpu和centralized有细微差别

##### FifoP

```
···
SignalReady();
WaitForEnclaveReady();
...
while (!Finished() || !scheduler_->Empty(cpu())) {
  scheduler_->Schedule(cpu(), status_word());
  ...
}
```

##### FifoC
```
Channel& global_channel = global_scheduler_->GetDefaultChannel();
...
SignalReady();
WaitForEnclaveReady();
...
while (!Finished() || !global_scheduler_->Empty()) {
  BarrierToken agent_barrier = status_word().barrier();
  if (cpu().id() != global_scheduler_->GetGlobalCPUId()) {
    RunRequest* req = enclave()->GetRunRequest(cpu());
    ...
    req->LocalYield(agent_barrier, /*flags=*/0);
  } else {
    if (boosted_priority() &&
        global_scheduler_->PickNextGlobalCPU(agent_barrier, cpu())) {
      continue;
    }
    Message msg;
    while (!(msg = global_channel.Peek()).empty()) {
      global_scheduler_->DispatchMessage(msg);
      global_channel.Consume(msg);
    }
    global_scheduler_->GlobalSchedule(status_word(), agent_barrier);
    ...
  }
}
```

#### Edf
```
Channel& global_channel = global_scheduler_->GetDefaultChannel();
...
SignalReady();
WaitForEnclaveReady();
...
while (!Finished()) {
  BarrierToken agent_barrier = status_word().barrier();
  if (cpu().id() != global_scheduler_->GetGlobalCPUId()) {
    RunRequest* req = enclave()->GetRunRequest(cpu());
    ...
    req->LocalYield(agent_barrier, /*flags=*/0);
  } else {
    if (boosted_priority()) {
      global_scheduler_->PickNextGlobalCPU();
      continue;
    }
    Message msg;
    while (!(msg = global_channel.Peek()).empty()) {
      global_scheduler_->DispatchMessage(msg);
      global_channel.Consume(msg);
    }
    global_scheduler_->UpdateSchedParams();
    global_scheduler_->GlobalSchedule(status_word(), agent_barrier);
    ...
  }
}
```

##### Cfs

```
...
SignalReady();
WaitForEnclaveReady();
...
while (!Finished() || !scheduler_->Empty(cpu())) {
  scheduler_->Schedule(cpu(), status_word());
  ...
}
```

#### Scheduler：Schedule

##### FifoP

##### FifoC

##### Edf

##### Cfs
