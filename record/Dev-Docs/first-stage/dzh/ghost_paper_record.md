# ghost论文阅读笔记


## 概要

+ 现如今争对使用场景对内核调度策略进行修改，在性能方面可以得到很大提升

+ 但是为单一使用场景定制特定调度策略的内核是不切实际的，而且还涉及到重启内核，这会导致性能，可用性大大降低

+ ghost是一个这样的用户态调度框架，通过用户态agent和内核通信，能够实时灵活定制用户想要的复杂调度策略而不需要重启内核，并且适应性广泛，无论是percpu还是centralized

+ 使用ghost能够增大吞吐量，减少延迟，同时为数据中心工作负载启用策略优化、非中断升级和故障隔离。


## 1.介绍

+ 许多特定场景的调度策略：

    - Shinjuku request scheduler
    - Tableau scheduler
    - Caladan scheduler

+ 在大型工程中部署特定调度策略难度极大，很可能造成内核崩溃，就算部署成功，对内核升级也需要停机

+ 以前的用户态调度框架设计有明显缺点：对应用部署需要修改；需要专门的资源时期可以高响应；需要针对应用来特定修改内核

+ 硬件环境变化

+ The goal of ghOSt is to fundamentallychange how scheduling policies are designed, implemented, and deployed. ghOSt provides the agility of userspace development and ease of deployment, while still enabling 𝜇s-scale scheduling

+ agent是一个os进程，通过相关系统调用与内核通信

+ 内核通过异步消息队列告诉agent它管理的线程状态变化

+ agent通过内核传递的消息同步地告诉内核调度策略的转变

+ ghost支持并发执行多个调度策略

+ ghost相关通信和调度时长很可观


### 2.背景与设计目标

##### 背景

+ linux目前采用cfs调度策略，很难针对特定场景进行针对性优化

+ 实现内核调度策略很难

+ 部署内核调度策略更难

+ 用户态线程的调度策略是不够的，归根结底它还是受制于内核调度策略

+ 为特定场景定制内核也是不切实际

+ 通过ebpf去定制调度策略？也不是很适合
    - ebfp受到诸多限制，如栈大小，循环次数，访问内核数据受限
    - ebpf是同步的，在调度前需要阻塞

##### 设计目标

+ 容易实现和测试

+ 效率高，易表达

+ 不局限于per-CPU模型

+ 支持多种并发策略

+ 非中断更新（不需要重启）和错误隔离


### 3.设计与实现

##### 基本理念

+ ghost概述

    - 用户态agent通知内核如何进行调度

    - 内核实现通过用户态信息实现一个类似于cfs的调度类 sheduling class

    - 调度类提供用户态一组接口让用户态去定制调度策略

    - 为了帮助用户态判断，内核将管理线程的状态通过消息和状态码传递给agent

    - 而agent通过系统调用syscall和事务transaction通知内核调度策略

+ percpu和centralized概念

    - percpu：调度只管本cpu的调度，有steal策略

    - centralized：全局调度

+ cpu与线程的概念

    - 线程：内核线程

    - cpu：执行单元

+ enclaves

    - 支持在单机上执行多种调度策略

    - 因地制宜分配cpu（如NUMA架构）

+ ghost使用agent

    - 用户态agent实现方便，调试简单

    - 配置调度策略无需重启系统

    - 对于percpu，都有一个agent对应，可以对每个cpu配置不同调度策略

    - 对于centralized，全局agent对所有cpu调度负责，同时还有其他不活动的agent

    - 所有agent通过内核线程的模式实现，他们同属于一个进程


##### 内核到代理的通信

+ 将所线程状态传递给agent

    - 共享内存？

    - 系统内存文件/proc/pid？

    - API（消息队列）yes

+ message消息

    - THREAD_CREATED
    - THREAD_BLOCKED
    - THREAD_PREEMPTED
    - THREAD_YIELD
    - THREAD_DEAD
    - THREAD_WAKEUP
    - THREAD_AFFINITY（线程绑核）
    - TIMER_TICK（确保agent基于最新状态做决定）

+ mq消息队列

    - 组织方式：共享内存中使用自定义队列

    - percpu每个cpu和agent间有一个初始mq

    - centralized所有cpu和全局agent间有一个初始mq



+ 线程和mq间组织方式 

    - CREATE/DESTROY_QUEUE：创建/摧毁mq

    - ASSOCIATE_QUEUE：修改线程msg和mq之间的发送关系

+ mq和agent间组织方式

    - CONFIG_QUEUE_WAKEUP：自定义msg到来时，对agent的唤醒后的行为（centralized没有配置，因为全局agent不能被阻塞）

+ 在mq/cpu间移动线程

    - ASSOCIATE_QUEUE：修改线程msg和mq之间的发送关系，失败场景：试图移动的线程还有msg在当前mq没有处理

+ 在agents和kernel间同步

    - 在agent做调度策略决定的时候，可能又有新的msg到来（后面讲述如何解决）

    - Aseq和Tseq的递增条件

+ 通过共享内存传递seq修改信息


##### 代理到内核的通信

+ agent通过transaction事务来和内核通信

    - percpu：一个系统调用接口足矣，centralized：核如果很多，那么使用系统调用性能将下降，共享内存更合适，所以，最终采用共享内存方案

    - TXN_CREATE

    - TXNS_COMMIT：对于percpu，发生context swtich，意味着当前agent被替换为要运行的线程

+ Group commits（批量提交）

    - 对于centralized调度，单个提交会导致性能大大下降

+ seq核事务

    - 在agent做调度策略决定的时候，可能又有新的msg到来（后面讲述如何解决），并且该msg可能来自高优先级线程，当前agent处于running，无法唤醒通知

    - 1)Read Aseq

    - 2)读取msq

    - 3)决定调度策略

    - 4)commit，若commit的最新Aseq比内核观测到的最新Aseq小，那么commit失败

+ 通过ebpf加速

    - cpu空闲，但是agent没有调度线程时，ebpf会选择线程运行

##### centralized 调度

+ 避免全局agent线程被抢占

    - 全局agent优先级最高，无论ghost还是非ghost，没有任何线程能抢占

    - 造成负面影响：每个线程存在绑定的工作线程

    - 通过切换到inactive 的agent解决

+ sql和centralized调度

    - 判断Tseq是否为最新


##### 故障隔离与动态升级

+ 和内核其他调度策略的关系

    - 优先级低于内核原生调度类，如cfs

+ 动态更新与回滚

    - 替换agent，保留enclave：新旧agent

    - 摧毁enclave，从头开始：摧毁当前enclave下所有agent，相关线程送回内核默认调度


+ 看门狗

    - 摧毁不进行线程调度的enclave


### 4.评估和对比

三个问题：

+ ghost相比于其他调度器有啥额外开销

+ 和之前的调度器相比

+ ghost是解决大规模低延迟工作负载，比如 Google Snap, Google Search和virtual machines的可行方案吗

##### ghost的开销

+ 代码量：少，而且高级语言通过调库使得代码量更少

+ 消息传递开销

+ 本地调度开销（percpu）

+ 远程调度开销（centralized）
    每秒每个cpu可以25200个线程（100个cpu），线程40us，能让所有cpu繁忙。
    随着agent个数增多，这个数据也是线性增长

+ 全局agent性能分析：全局agent调度其他线程，统计cpu数量和commit个数的关系
    - 第一次drop：工作线程和全局代理竞争
    - 第二次drop：核间通信开销大于核增加开销

##### 和其他定制的centralized调度类比较

+ 配置

    - 初始Shinjuku：调度线程绑物理核，20个工作线程绑逻辑核（别的线程不能在此核上运行）
    - 基于ghost：710行代码，应该全局agent线程，200个工作线程（都不绑核，所以允许空闲cpu调度其他线程）
    - 非抢占基于cfs的Shinjuku

    
+ 单一调度：只调度rocksdb
    - ghost比初始代码量少82%
    - 延迟ghost稍微高
    - 饱和吞吐量相差不到5%
    - 原因：对于每次请求，ghost都得调度一次；初始只要在20个pin住的工作线程上传递调度请求即可

    - cfs由于不能抢占，太费

+ 多种调度：
    - 初始由于不能控制其他cpu线程，当负载低的时候也不能将cpu放出
    - 试Shenango？不适合执行时间时刻变化的场景，导致延迟相比于Shinjuku更久
    - 吞吐量比较和单一调度差不多，但是cpushare，ghost可以将空闲cpu分享给其他线程


个人总结：延迟ghost略逊，但是对于将cpu分享给其他线程这块，ghost做的更好，Shinjuku不会分享空闲cpu

##### Google Snap

+ 实时工作线程如何调度？
    MicroQuanta保证在任何一个时间周期内（例如1ms）将cpu分享给指定工作线程：兼顾实时调度工作线程和不让其他线程挨饿（个人解读：寻找一个恰好合适的周期，能够兼顾二者）
    

+ 测试环境
    - 6对客户-服务器模型线程，1个64b，另外5个64kb
    - 两种实验测试：quiet只有工作线程；loaded还有其他40竞争线程

+ ghost配置
    - ghost的部署：全局agent线程管理工作线程和其他竞争线程，snap工作线程具有高优先级
    - ghost管理的线程会被cfs调度的客户-服务器模型线程和原生线程抢占
    - 竞争线程只会在空闲时被调度

+ 延迟比较：
    - quiet：64b ghost要好，64kb初始要好   64kb拷贝时间过多导致调度事件过少
    - loaded： 同上，但是64kb ghost的差距没64b大

个人总结：ghost能够灵活调度线程去空闲cpu，一个特别简单的ghost配置就能接近大量修改内核的MicroQuanta调度策略

##### Google Search

+ 测试环境

    - cpu和内存 敏感
    - SSD和内存敏感
    - cpu敏感

+ ghost配置

    - 全局agent调度256个cpu，NUMA和CCX情况需要考虑，前者有自己内存，后者有自己L3cache

+ QPS和延迟比较

    - QPS二者差不多，二者都能针对NUMA和CCX情况优化，但是CFS需要重启

    - 在延迟上，ghost明显好于cfs，除了第三种。CFS以毫秒级平衡线程调度，ghost则会在微秒级做出反应。对于第三种，也能通过重新制定调度策略优化

+ 快速进行实验的经验

    - 对内核修改后重新编译花费很长时间，每天只能干5次

    - ghost不需修改重启内核，只需要1分钟

个人总结：ghost低延迟，特定场景对空闲cpu的掌控能力使得其延迟很凶残

##### 保护虚拟机，防止L1TF/MDS攻击

ps：这两个攻击我都不懂，乐，大概是可以从运行在同一个超线程的另一个虚拟机中窃取数据。解决办法是确保每个虚拟机的每个虚拟cpu只运行在特定的物理核上。 Microarchitectural buffers在切换其他虚拟机的时候需要被清空

+ per-core 调度：让一个核上运行相同虚拟机的cpu

+ 性能，差不多，但是后两个要安全

个人总结：针对攻击能够制定特定调度策避免被攻击

### 5.未来工作

+ 使用ebpf加速

+ 关闭时间中断

### 6.相关工作

没啥好看的

### 7.结论

没啥好说的

