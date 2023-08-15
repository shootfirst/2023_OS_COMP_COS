## 前言

enclave, scheduler, agent, channel, message，statusword, statuswordtable, runrequest, task，ghost，cpu。这最核心的概念在ghost用户态调度框架里的关系是？



## Agent

#### Agent

explain:论文中的agent，和cpu一一对应，属于enclave，和schduler一一对应，也就是说每一个agent都有自己的调度算法

```
StartBegin()：agent线程开始执行threadbody，对于localagent是将当前agent迁移到其管理的cpu上
StartComplete()：等待enclave ready
TerminateBegin()：通知条件变量
TerminateComplete()：摧毁线程资源
ThreadBody()：agent线程所执行的函数
AgentThread()：在agent相关准备就绪后，由ThreadBody调度。由具体的调度算法的agent去实现
Ping()：让agent线程回到它所管理的cpu上执行
SignalReady() ：在agent初始化结束后唤醒start_complete()，让其可以接下来调用enclave的ready方法
WaitForEnclaveReady() ：等待所在enclave的ready
AgentScheduler()：返回agent的调度类，返回空值，被其他继承的调度类重写，其他继承的调度算法会有自己的Scheduler调度类

Enclave* enclave_:agent所属enclave
Gtid gtid_：agent的ghost线程号
Cpu cpu_：agent所管理的cpu
Notification ready_, finished_, enclave_ready_, do_exit_：相关用于同步的条件变量
std::thread thread_：运行agent的线程
```

#### LocalAgent

explain：继承agent，供其他调度算法继承，如FifoAgent。重写了ThreadBody（就是上面的），多了statusword字段

```
LocalStatusWord status_word_：通过此内核共享内存获取相关信息，如cpu空闲，Aseq等
```

#### FullAgent

explain:将单个enclave下的agent，task，scheduler汇集起来，别的调度算法会新建一个个性化类去继承这个类，如FullFifoAgent，和enclave一一对应

```
StartAgentTasks()：创建当前enclave下的所有cpu的agent，和它们对应的cpu绑定起来，并且一次调用它们的StartBegin方法迁移到对应cpu上运行
TerminateAgentTasks()：被派生类的析构方法调用

LocalEnclave enclave_：一一对应的enclave
std::vector<std::unique_ptr<Agent>> agents_：其中所包含的agent
```


#### FullFifoAgent(p)

explain：继承FullAgent

```
FullFifoAgent(AgentConfig config)：整个调度算法初始化的开始

std::unique_ptr<FifoScheduler> scheduler_:按道理来说，每个agent有一个调度类，也就是说这里应该是一个调度类的list，但是这里只有一个，我觉得应该是因为调度算法是FIFO，所以所有agent应该统一成一个调度类
```

#### FullFifoAgent(c)

explain：继承FullAgent

```
FullFifoAgent(FifoConfig config)：整个调度算法初始化的开始

std::unique_ptr<FifoScheduler> scheduler_:按道理来说，每个agent有一个调度类，也就是说这里应该是一个调度类的list，但是这里只有一个，我觉得应该是因为调度算法是FIFO，所以所有agent应该统一成一个调度类
```

#### AgentProcess
explain:一个地址空间一模一样的父子进程，负责运行FullAgent

```
AgentProcess(AgentConfig config)：构造出父子两进程，父进程负责幕后，子主线程负责创建agent线程并且等待退出，config将传给FullAgent的构造方法

std::unique_ptr<ForkedProcess> agent_proc_：fork系统调用的封装
std::unique_ptr<FullAgent> full_agent_：被运行的FullAgent
std::unique_ptr<SharedBlob> sb_：父子进程共享此内存进行通信
```


## Task

#### Task

explain：代表一次要被调度上cpu的任务，被调度算法继承，如FifoTask

```
Gtid gtid：被调度task所代表线程的gtid
LocalStatusWord status_word：和内核共享的seq等信息
Seqnum seqnum：seq
```

#### TaskAllocator

explain：存储task

#### SingleThreadMallocTaskAllocator

explain：继承TaskAllocator

```

```





## Sheduler

#### Sheduler

explain:做调度决策的类，和agent应该是M对N？

```
Scheduler(Enclave* enclave, CpuList cpus)：将当前调度类加入enclave中
EnclaveReady()：TODO
DiscoverTasks()：TODO
GetDefaultChannel()
GetAgentChannel(const Cpu& cpu)

Enclave* const enclave_
CpuList cpus_;
```

#### BasicDispatchScheduler

explain:继承Sheduler，一种调度器实现，能够解码原始消息（来自channel），将它们与任务派生类型相关联，并调度到适当的调度类方法。其他的调度算法会继承这个类，如FifoScheduler

```
BasicDispatchScheduler(Enclave* enclave, CpuList cpus, std::shared_ptr<TaskAllocator<TaskType>> allocator)
void DispatchMessage(const Message& msg) ：将消息根据类型进行相应处理

相应处理方法：交给对应调度类去实现
CpuTick(const Message& msg)
CpuNotIdle(const Message& msg) 
CpuTimerExpired(const Message& msg) 
CpuAvailable(const Message& msg) 
CpuBusy(const Message& msg) 
AgentBlocked(const Message& msg) 
AgentWakeup(const Message& msg) 

std::shared_ptr<TaskAllocator<TaskType>> const allocator_
```










## Enclave

#### Enclave

explain:论文中的enclave，上面包含运行的agent和scheduler，cpu拓扑

```
Enclave(const AgentConfig config)：通过agentconfig去构造enclave
GetRunRequest(const Cpu& cpu)获取指定cpu上的runrequest
CommitRunRequest(RunRequest* req)：commit此runrequest，底层调用ghost的接口
SubmitRunRequest(RunRequest* req)：submit此runrequest，底层调用ghost的接口
CompleteRunRequest(RunRequest* req)：complete此runrequest，底层调用ghost的接口，和上面那个接口配合使用估计
LocalYieldRunRequest(const RunRequest* req, BarrierToken agent_barrier, int flags)：agent结束在当前cpu上的调度
Ready()：必须在当前enclave上的所有agent和所有scheduler被构造后才能调用
WaitForOldAgent()：如果有一个老agent还在此enclave上，等待直到它退出
AttachAgent(const Cpu& cpu, Agent* agent)
void DetachAgent(Agent* agent)
AttachScheduler(Scheduler* scheduler)
DetachScheduler(Scheduler* scheduler)

const AgentConfig config_：代表本enclave相关参数
Topology* topology_：机器的cpu拓扑
CpuList enclave_cpus_：本enclave包含的cpu！！！
std::list<Scheduler*> schedulers_：在enclave上运行的schedulers
std::list<Agent*> agents_：在enclave上运行的agent
```

#### LocalEnclave

explain：继承enclave，不能再被继承

```
MakeChannel(int elems, int node, const CpuList& cpulist)
struct CpuRep {
   Agent* agent;
   LocalRunRequest req;
}
CpuRep cpus_[MAX_CPUS]：cpu与其一一对应的agent，runrequest
ghost_cpu_data* data_region_：内核共享通信区域
size_t data_region_size_
int dir_fd_ = -1：enclave相当于目录
int ctl_fd_ = -1：控制enclave的fd
```

## RunRequest


#### RunRequestOptions 
explain: commit一个txn的参数

```
Gtid target = Gtid(0) //the task to run next
BarrierToken target_barrier //Tseq
BarrierToken agent_barrier = StatusWord::NullBarrierToken() // Aseq
int commit_flags = 0 // controls how a transaction is committed
int run_flags = 0  // control a variety of side-effects when the task either gets oncpu or offcpu
```


#### RunRequest

explain：代表一次commit的请求容器，容器中装的是task，和一个cpu一一对应，底层将调用GhostHelper()->Run()提交本次请求

```
Init(Enclave* enclave, const Cpu& cpu)：初始化runrequest，其所在的enclave和一一对应的cpu
Open(const RunRequestOptions& options)：开启一个即将要提交的事务，相关参数位于options
void OpenUnschedule()   
void LocalYield(const BarrierToken agent_barrier, const int flags) :Agent must call LocalYield when it has nothing to do
bool Ping() ：懂得都懂
bool Commit() ：对ghost commit的封装
bool Submit() ：对ghost submit的封装


Enclave* enclave_
Cpu cpu_;
```

#### LocalRunRequest

explain：继承runrequest

```
Init(Enclave* enclave, const Cpu& cpu, ghost_txn* txn)：初始化，但是多了代表事务相关信息的txn

ghost_txn* txn_：代表事务相关信息的txn
```









## Cpu

#### Cpu

explain：一个cpu的相关信息，如L3Cache，NUMA等

```
struct CpuRep {
    int cpu;
    int core;
    int smt_idx;
    std::unique_ptr<CpuList> siblings;
    std::unique_ptr<CpuList> l3_siblings;
    int numa_node;
}
const CpuRep* rep_：cpu相关信息
```

#### CpuMap
explain：TODO一个代表指导cpu是否被设置的位图

#### CpuList
explain：继承CpuMap


#### Topology.h
explain：代表机器的cpu拓扑信息（我觉得应该是代表整个机器的，而不是一个enclave的，也就是说整个机器的cpu信息都在这里面）

```
const uint32_t num_cpus_：cpu个数
CpuList all_cpus_：cpu位图
std::vector<Cpu::CpuRep> cpus_：所有cpu信息
int highest_node_idx_：numa节点个数
std::vector<CpuList> cpus_on_node_：各个numa节点的cpu
```








## Message

#### Message

explain：消息队列中的消息，被存放在消息队列中等待agent或者kernel去消费

```
struct ghost_msg {
	uint16_t type;		/* message type */
	uint16_t length;	/* length of this message including payload */
	uint32_t seqnum;	/* sequence number for this msg source */
	uint32_t payload[0];	/* variable length payload */
};

ghost_msg* msg_
```


## Channel

#### Channel

explain：消息队列，存放消息，基于共享内存，和Cpu关系 TODO

```
Peek()：获取队首
Consume(const Message& msg)：弹出队首
max_elements()：环形队列大小
AssociateTask(Gtid gtid, int barrier, int* status)TODO底层调用ghost的api
SetEnclaveDefault()：将当前channe设置为enclave的默认channelTODO
```

#### LocalChannel

explain：继承Channel

```
LocalChannel(int elems, int node, CpuList cpulist)：底层调用ghost的api
struct ghost_queue_header {
	uint32_t version;	/* ABI version */
	uint32_t start;		/* offset from the header to start of ring */
	uint32_t nelems;	/* power-of-2 size of ghost_ring.msgs[] */
} 
int fd_：消息队列的fd
ghost_queue_header* header_：队头
```


## StatusWord

#### StatusWord

explain：和内核通信的共享内存，存储Tseq，Aseq等信息

```
typedef uint32_t BarrierToken

struct ghost_sw_info {
	uint32_t id;		/* status_word region id */
	uint32_t index;		/* index into the status_word array */
};

struct ghost_status_word {
	uint32_t barrier;
	uint32_t flags;
	uint64_t gtid;
	int64_t switch_time;	/* time at which task was context-switched onto CPU */
	uint64_t runtime;	/* total time spent on the CPU in nsecs */
}

ghost_sw_info sw_info_：sw的id和index
ghost_status_word* sw_：sw信息
```

#### LocalStatusWord

explain：上面的继承

#### StatusWordTable

explain：存储statusword的一块内存区域(ps：这里我觉得有必要将statuswordtable和channel来一个对比，它们都是和内核共享数据结构)

```
size_t map_size_ = 0;
ghost_sw_region_header* header_ = nullptr;
ghost_status_word* table_ = nullptr;
```

#### LocalStatusWordTable







## Ghost

#### Ghost

explain：ghost内核相关接口的封装

```
原系统调用：
Run(const Gtid& gtid, BarrierToken agent_barrier, BarrierToken task_barrier, const Cpu& cpu, int flags)：LocalYieldRunRequest和PingRunRequest，这两者是干啥呢
SyncCommit(cpu_set_t& cpuset)：SubmitSyncRequests
Commit(cpu_set_t& cpuset)：SubmitRunRequests
CreateQueue(int elems, int node, int flags, uint64_t& mapsize)：LocalChannel的构造方法
ConfigQueueWakeup(int queue_fd, const CpuList& cpulist, int flags)：LocalChannel的构造方法
AssociateQueue(int queue_fd, ghost_type type, uint64_t arg, BarrierToken barrier, int flags)：LocalChannel的AssociateTask
SetDefaultQueue(int queue_fd)：SetEnclaveDefault
GetStatusWordInfo(ghost_type type, uint64_t arg, ghost_sw_info& info)：LocalStatusWord(StatusWord::AgentSW)

SchedGetAffinity(const Gtid& gtid, CpuList& cpulist)：cfs
SchedSetAffinity(const Gtid& gtid, const CpuList& cpulist)：rocksdb？
SchedTaskEnterGhost(int64_t pid, int dir_fd)
SchedAgentEnterGhost(int ctl_fd, const Cpu& cpu, int queue_fd)  ：Makes calling thread the ghost agent on `cpu`.
```

#### GhostSignals 

explain：ghost线程相关信号处理（不怎么涉及？先不管）

#### GhostThread

explain：原生线程的封装，可以决定被cfs还是ghost调度。和enclave的关系犹如目录与文件

```
int tid_;
Gtid gtid_;
KernelScheduler ksched_：ghost还是cfs
Notification started_：线程开始运行，则这个将被唤醒
std::thread thread_：线程
```





