# ghOSt

> ghOSt provides state encapsulation, communication, and action mechanisms that allow complex expression of scheduling policies within a userspace agent, while assisting in synchronization.ghOSt 提供状态封装、通信和操作机制，允许在用户空间代理中复杂地表达调度策略，同时协助同步。
>
> 可以使用任意语言和优化策略，并且不用reboot操作系统
>
> ghOSt 支持广泛的调度模型，从每个 CPU 到集中式，从运行到完成到抢占式【run-to-completion to preemptive】，并且调度操作的开销很低。

## 具体实现

### 前置知识

#### per-CPU model、 centralized model

ghost对两个模型都支持。

1. 逻辑CPU

   一个物理CPU可以由多个CPU Core和一个Uncore部分组成，每个CPU Core内部又可以由两个CPU Thread组成。每个CPU thread都是一个操作系统可见的**逻辑CPU**。

2. per-CPU model

   > per-CPU：
   >
   > Linux操作系统，特别是针对SMP或者NUMA架构的多CPU系统的时候，描述每个CPU的私有
   >
   > 数据的时候，Linux操作系统提供了per_cpu机制。
   >
   > per_cpu机制就是让每个CPU都有自己的私有数据段，便于保护与访问。
   >
   > [都有哪些per-CPU变量](https://blog.csdn.net/weixin_45030965/article/details/126289230)

   一个agent管理一个逻辑CPU

3. centralized model

   一个agent管理多个逻辑CPU

4. per-NUMA-node

   > [NUMA架构详解](https://blog.csdn.net/qq_20817327/article/details/105925071) 
   >
   > NUMA(Non-Uniform Memory Access)是指多处理器系统中，内存的访问时间是依赖于处理器和内存之间的相对位置的。相对近的内存被称作本地内存；相对远的内存被称为非本地内存。
   >
   > 一个NUMA Node内部是由一个物理CPU和它所有的本地内存(Local Memory) 组成的。广义得讲，一个NUMA Node内部还包含本地IO资源

#### 处理器间中断

处理器间中断（Inter-Processor Interrupt，IPI）是一种特殊类型的中断，即在多处理器系统中，如果中断处理器需要来自其它处理器的动作，一个处理器向另一个处理器发出的中断行为。可能要求采取的行动包括：刷新其它处理器的内存管理单元缓存，如转译后备缓冲器，当一个处理器更改内存映射时；停机，当系统被一个处理器关闭时。

#### 调度类

Linux支持通过调度类来实现多个策略。**这样做，将具体的策略和实际调度进行了解耦**。

> Linux 内核 sched_class 调度器有五种类型：
>
> ```c++
> extern const struct sched_class dl_sched_class // 限期调度类
> extern const struct sched_class rt_sched_class // 实时调度类
> extern const struct sched_class stop_sched_class// 停机调度类
> extern const struct sched_class idle_sched_c lass
> extern const struct sched_class fair_sched_class// CFS
> ```

并且linux不同策略类是有优先级的。每个线程调度应该只能依据一个策略。具有高优先级调度策略的线程会比具有低优先级调度策略的进程优先调度。

也就是说，进程调度的时候，影响其优先级的有：它的调度策略的优先级，以及它在调度策略中的优先级。



### 主要逻辑

Figure.1就可以展现出ghost的总体架构了。总的来说可以分为以下几部分：

**【注：下面的内容有很多猜的部分，都还没看代码验证想法。有必要的话再去看看。】**

#### agent

agent是委托给用户态实现的具体调度策略实现。

它会针对从kernel传来的线程信息，来进行调度决策，然后通过系统调用将决策结果提交给kernel，最后由kernel进行上下文切换。

[图片：Fig.2]

**对于per-CPU model**，一个agent对一个逻辑CPU负责，并且**agent也被视为线程**，在其对应逻辑CPU执行，这意味着它会与自己管理的ghost线程进行CPU抢占。

<u>agent差不多是所有线程（ghost+non-ghost）中优先级最高的</u>，这是为了在需要进行调度决策时【指消息队列中传来消息】能够让agent迅速抢占CPU，快速做出决策。

当agent做出调度决策后，它会进行上下文切换，切换到下一个要被执行的线程。



**对于centralized model**，存在一个global agent和多个incative agent。global agent对多个逻辑CPU负责。

global agent**采用不间断轮询调度**。与per-CPU相同，<u>global agent也被置为最高优先级</u>，从而确保它始终独占一个CPU，禁止其他ghost线程对其进行抢占。

它通过处理器间中断+批处理中断，来对其管理的remote CPU进行调度。

> agent线程的优先级被设为最高，使得kernel stability受到影响。因而，当non-ghost进程要用agent的cpu时，agent就马上yield，并且让其中一个占据了别的cpu的idle agent进行工作。



值得注意的是【下面是猜的】，从论文中给的代码可以看出，如果说scheduler是这样的形式：

```c
// for centralized model
while(true) do:
	policy();
// for per-CPU model
while(true) do:
	policy();
	context_switch();
```

那么用户只需实现其中的`policy()`部分，其余应该都在ghost框架中实现了。

#### kernel

ghost对内核进行了修改，来支持userspace的实现。具体应该有以下两点修改：

1. 新增调度类

   【下面都是我纯纯的猜想，没验证过】

   ghost给内核增加了新的调度类，也即ghost调度类。ghost调度类用来调度ghost线程，但注意它管不到agent线程。

   ghost调度类在具体实现中，不是像其他调度类一样实现什么FIFO之类的调度策略，而是执行将线程状态信息塞进消息队列这样的逻辑。这样一来，就将策略逻辑委托给了agent实现。

   值得注意的是，ghost调度类会设置为比默认调度类（一般是CFS）更低一级的优先级。具体为什么详见`安全防范`部分。

2. 新增syscall `TXNS_COMMIT()`

   agent会调用系统调用，将调度决策传递给内核。

   

#### kernel与agent通信

使用消息队列进行通信。

1. kernel会给agent传递时钟中断的信息，以及发送某个进程当前的状态
2. 一个agent管理一条消息队列
3. 对于per-CPU model，当agent管理的消息队列中有消息，agent就会被唤醒；对于centralized model，消息队列一直被不断轮询。
4. agent可以选择将自己旗下的一些thread扔给别人



#### agent与kernel通信

采用事务机制，事务类的具体方法通过syscall实现。

1. 使用事务提交调度策略，允许group commit

   详见论文那段代码

2. 使用seq num保证通信同步

   seq num被kernel（具体逻辑应该由ghost调度类实现）通过shared memory以status words的形式来与agent进行共享。

   分为agent seq num和thread seq num

   1. per-CPU

      采用Aseq同步

      1. 为什么要同步

         agent想要立刻接收到消息队列的信息的话，必须处于sleep状态，这样它才能被信息wakeup。

         如果当agent正在run，它就无法接收到最新消息。只有在当他做出调度决策之后，将当前CPU所有权转让给下一个需要被调度的线程之后，再次进入沉睡，才能再次被唤醒，从而得到最新消息，但这太晚了。To address this，我们引入了seq num。

      2. 做法

         1)Read Aseq

         2)读取msq

         3)决定调度策略

         4)commit，若commit的最新Aseq比内核观测到的最新Aseq小，那么commit失败

         ​	值得注意的是，对seq num是否陈旧的校验在内核完成，我猜应该是在syscall事务提交的时候。

   2. centralized 

      采用Tseq同步

      意思好像是说有一个thread的seq不一致，该次事务的group commit全部无效，这也体现了事务的原子性。

3. 使用ebpf加速

   当CPU变得空闲而代理尚未发出事务时，BPF程序会发出自己的事务，选择一个在该CPU上运行的线程。这是利用了ebpf的即时性。这部分改动其实也不是不能在agent做或者多弄一个线程做，ebpf胜就胜在其方便高效。

   

### 安全防范

1. agent崩溃后旗下线程转CFS调度

   目标：当policy错误，让threads按照默认调度器CFS进行调度

   我们要实现这个目标，可以在内核的调度类层次结构中为ghOSt调度器类分配一个比默认调度器类更低的优先级——通常是CFS。其结果是，**系统中的大多数线程将优先于ghOSt线程**。

   ghOSt线程的抢占导致了THREAD_PREEMPT消息的创建，从而触发相关的agent（agent在不同的高优先级调度类中运行）来做出调度决策。

2. 通过替换agent（优先）和摧毁enclave进行热更新和rollback

3. 杀死指定时间仍然有线程饥饿的agent enclave



## 性能分析

Our evaluation of ghOSt focuses on three questions:

1. ghost特有操作的代价
2. 实现一些schedule policy，然后将其性能与别人实现的用户态框架对比
3. ghost能否适应大规模低延迟实际场景，如Google Snap 、 Google Search以及virtual machines

### 特有操作代价分析

1. 代码行数

2. 各项特殊操作代价

   1. 通信开销

      对于per-CPU：enque、上下文切换到local agent、deque

      对于centralized：enque、deque

   2. local schedule

      对于per-CPU：提交事务、上下文切换到target thread

   3. remote schedule

      对于centralized：发送IPI（处理器间中断）、上下文切换

   4. global agent管理范围

      使用round-robin policy测试，每次提交尽可能多的事务。

      具体分析了曲线升降原因

### 与Shinjuku对比

ghost是个generic scheduling framework。现在要把它跟highly specialized scheduling system Shinjuku对比。

都在RocksDB workload下测试

1. Shinjuku policy具体场景

   It uses 20 **spinning** worker threads pinned to 20 different hyperthreads and a **spinning** dispatcher thread, running on a dedicated physical core. 

   spinning thread**独占**逻辑CPU，也就是说不论是worker线程还是scheduler线程都是独占CPU一动不动的。

   调度程序通过FIFO的方式将到达的request分配给worker threads。每个请求在被抢占并添加到FIFO后面之前运行到一个有限的运行时。

2. 在ghost中实现shinjuku policy。

   ghOSt-shinjuku global agent从它自己的物理核心开始，但可以自由移动。

   共有20个逻辑CPU，200个worker thread，global agent使用FIFO来管理workers。

3. 也实现了一个跑在CFS上的non-preemptive版本

   但是可能由于CFS的通用性，Shinjuku policy的特征（the use of virtualization features and posted interrupts for preemption）没有发挥出来

4. Single Workload Comparison

   workload为对RockDB的GET query。

   ghost的每个worker线程分配30 *𝜇*s的time slice，CFS的运行到完成。

5. Multiple Workloads Comparison

   在生产场景中，当RocksDB负载较低时，使用空闲的计算资源来服务于低优先级的批处理应用程序或运行无服务器函数是很有吸引力的。原来的Shinjuku不能管理任何其他本机线程

   为了解决此问题，我们拓展为ghOSt-Shinjuku-Shenango模型。

### Google Snap

现在用ghost替代soft realtime kernel scheduler MicroQuanta来跑Google Snap。

1. 具体场景

   Snap维护轮询（worker）线程，这些线程负责与网卡硬件的交互，并代表重要服务运行自定义网络和安全协议。snap维护至少一个polling worker。当一坨networking load到达时，snap会通过唤醒来增加worker线程。

   这种针对load不同的高频率沉睡-唤醒机制，需要快速的调度程序干预，以避免增加延迟。这个调度程序就是MicroQuanta。

2. The test workload

   test workload由6个client thread组成，每秒向另一台机器上的6个server thread发送10k条消息，并接收同样多的回复。 其中1个client消息大小64-bytes，另外5个为64-kb。

   在所有的实验中，客户端和服务器线程都由Linux CFS调度。在我们的ghOSt实验中，工作线程是用ghOSt而不是MicroQuanta来安排的。

   我们在两种模式下运行测试。在quiet模式下，只有client/server。在load模式下，当网络流量低时，机器还运行一批work load，使用服务器或Snap线程未使用的空闲CPU资源。

3. ghOSt policy

   我们通过这样的ghOSt policy来取代MicroQuanta：

   这个policy是一个centralized FIFO policy【似乎说过realtime的话最好采用FIFO，正是本例】。它会管理worker线程和其他的work load，并且设置snap worker线程为高优先级，也即优先级：CFS管理线程 > snap worker > client/server、其他work load。这样一来，在load模式下，就会优先运行snap worker，同时又能确保系统的稳定性。

4. Tail latency comparison

###  Google Search

将ghost替代CFS来跑google search query。

1. The test workload

   有三种query：ABC

   1. queryA

      CPU和内存密集型查询，根据需要唤醒worker线程

   2. queryB

      需要很少的计算，需要访问SSD，并且由一组短暂的worker线程提供服务，也根据需要唤醒

   3. queryC

      由长寿的工作线程提供服务的cpu密集型负载

   worker线程主要处理子查询。其中的一些子查询需要利用数据局部性原理，被一些以NUMA node为特点的线程进行执行。

2. ghOSt policy

   使用centralized model，一个global agent，256个逻辑CPU。

   一开始，global agent会先生成系统的拓扑信息，它会依据这个拓扑信息来调度线程。

   > 这里涉及到图论，是因为NUMA node相当于图结点吗？

   global agent还会维护一个最小堆，每次会挑选具有最小运行时（the least elapsed runtime）的线程执行。

   线程运行到完成/直到被CFS抢占。

   由于我们第二点最后所说的那个原因，部分worker会有指定的CPU从而能形成一个NUMA node。因而，当worker线程创建时，会被分配一个cpumask，表示可以执行它的cpu族。每次调度时，如果不存在属于该cpumask族的idle cpu，该线程就会被调度失败。

   每次选择离上次运行CPU最近的CPU。可以按照三级缓存一级级找。

3. CFS vs. ghOSt

4. Tail Latency

5. Our experience with rapid experimentation

### Protecting VMs from L1TF/MDS Attacks

1. 应用背景

   恶意虚拟机利用微架构缺陷从运行在兄弟超线程上的另一个虚拟机窃取数据。通过确保每个物理CPU只运行来自同一个VM的逻辑CPU可以缓和该问题。当新VM调度到一个物理核心上时，将刷新微体系结构缓冲区。

   所以我们这次背景的目的之一，就是要对同一个物理CPU的逻辑CPU进行调度。这对于ghost比较容易实现。ghOSt agent可以将一个物理CPU视为一个逻辑CPU族，通过执行一个同步的**组提交**来轻松地调度整个核心，即，为一个核心的两个cpu发出提交，这些cpu必须全部成功或全部失败。这样一来，就能保障一个VM的所有线程运行在同一个CPU上，从而预防L1TF/MDS Attacks。【Fig.9】

2. Secure VM Core Scheduling Policy

   我们在内核和ghOSt中实现了一个类似于Tableau的vm调度策略。

   我们使用一个分区EDF方案，其中每个物理CPU为每个线程运行提供保证时间，限制尾部延迟。任何多余的时间都可以在可运行的线程之间公平地共享，从而提高了平均延迟。

   runqueues跨越单个NUMA节点；当物理CPU空闲并寻找要运行的新线程时，它偏向于在其numa-本地运行队列中选择一个线程。在高负载情况下，允许选择不符合cpumask的NUMA运行。

3. Evaluation
