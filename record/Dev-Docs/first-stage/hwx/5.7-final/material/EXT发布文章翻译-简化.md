之前被反对
 At that time, the idea of pluggable schedulers was strongly rejected; only by focusing energy on a single scheduler, it was argued, could the development community find a way to satisfy all workloads without filling the kernel with a confusion of special-purpose schedulers.

> 当时，可插拔调度器的想法遭到强烈反对； 有人认为，只有将精力集中在单个调度程序上，开发社区才能找到满足所有工作负载的方法，而不会使内核充满专用调度程序的混乱。



BPF的优点：
1.难度降低
2.方式安全，无需重启
3.支持小众负载
4.易部署到大型系统



## Scheduling with BPF

补丁集添加了一个新的调度类，称为 SCHED_EXT，可以像大多数其他调用一样通过 sched_setscheduler() 调用来选择它（选择 SCHED_DEADLINE 有点复杂）。 它是一个非特权类，意味着任何进程都可以将自己放入 SCHED_EXT。 SCHED_EXT 位于优先级栈中空闲类（SCHED_IDLE）和完全公平调度器（SCHED_NORMAL）之间。 

因此，任何 SCHED_EXT 调度程序都不会阻碍别的进程run，如以 SCHED_NORMAL 运行的普通 shell 。 

大部分工作负载将在 SCHED_EXT 调度类中运行。



如果没有加载 BPF 调度程序，那么任何已放入 SCHED_EXT 类的进程将像在 SCHED_NORMAL 中一样运行。 但是，一旦加载了 BPF 调度程序，它将接管所有 SCHED_EXT 任务的责任。 

还有一个 BPF 调度程序可以调用的魔法函数 (scx_bpf_switch_all())，它将所有运行在实时优先级以下的进程移动到 SCHED_EXT。



实现调度程序的 BPF 程序通常会管理一组dispatch queues,  每个dispatch queues可能包含等待 CPU 执行的可运行任务。

默认情况下，系统中每个 CPU 都有一个 dispatch queue，还有一个 global queue。

当 CPU 准备好运行新任务时，调度程序将从dispatch queue中拉出一个任务并将其分配给 CPU。 

调度程序的 BPF 端主要实现为一组回调，通过操作结构调用，每个回调通知 BPF 代码一个事件或需要做出的决定。【**也就是说，这些callbacks是使用这个框架的用户需要自己实现的？**】

 完整的集合可以在 SCHED_EXT 存储库分支的 include/sched/ext.h 中找到。 该列表包括：

**【意思是说，以下这些函数，函数体都由scheduler实现，但是都会由ebpf框架调用？】**

prep_enable()     通知调度程序有一个新任务正在进入 SCHED_EXT；允许阻塞并可用于内存分配
enable()    不能阻塞，enables scheduling for the new task

select_cpu()    **Select a CPU** for a task that is just waking up。返回被选中的CPU id

enqueue()    Enqueue a task into the scheduler for running.通过scx_bpf_dispatch()。

> Among other things, that call provides the length of the time slice that should be given to the task once it runs. 并且在enqueue（）中还可以设置任务的时间片。
>
> If the slice is specified as SCX_SLICE_INF, the CPU will go into the tickless mode when this task runs.如果时间片被设置为INF，CPU就会进入tickless模式。
>
> It's worth noting that enqueue() is not required to put the task into any dispatch queue; it could squirrel that task away somewhere for the time being if the task should not run immediately. enqueue不要求将task放入any dispatch queue（？？？要不然放哪啊），它可以在不需要马上run的时候暂存这些task在别的地方。
>
> The kernel keeps track, though, to ensure that no task gets forgotten; if a task languishes for too long (30 seconds by default, though the timeout can be shortened), the BPF scheduler will eventually be unloaded.kernel会追踪并确保这些task没有被忘记。如果一个task被暂存太久了，这个BPF scheduler就会被卸载，滚蛋、

dispatch()    当一个CPU的dispatch queue为空时，dispatch()将任务分派到该队列中以保持CPU忙碌。如果调度队列仍然为空，调度程序将尝试从全局队列中获取任务。

update_idle()    回调函数将通知调度程序一个CPU何时进入或离开空闲状态。

runnable()、running()、stopping()和quiescent()    通知调度程序任务的状态更改。它们分别在任务变为可运行、在CPU上开始运行、从CPU上被取下或变为不可运行时调用

cpu_acquire()和cpu_release()      回调函数通知调度程序系统中CPU的状态。当一个CPU对BPF调度程序可用时，回调函数cpu_acquire()将通知它这个事实。当一个CPU不可用时（例如，一个实时调度类可能已经使用它），将通过调用cpu_release()来通知它。

还有许多其他的回调函数用于控制组的管理、CPU亲和性、核心调度等。

此外，还有一组函数可供调度程序调用以影响调度决策；例如，scx_bpf_kick_cpu() 可用于抢占正在给定CPU上运行的任务，并回调调度程序以选择在该CPU上运行的新任务。



## Examples

我们最终实现了一个eBPF调度框架。我们还提供了很多scheduler example。

[其中一部分](https://lwn.net/ml/linux-kernel/20230128001639.3510083-15-tj@kernel.org/)是一个最小的“虚拟”调度器，它使用默认的回调函数；另一个则是一个基本调度器，实现了五个优先级级别，并展示了如何将任务存储到 BPF 映射中。“虽然不是很实用，但它作为一个简单的示例很有用，并将用于演示不同的功能”。

此外，还有一个“centralized”调度程序，它将一个CPU专用于调度决策，使得所有其他CPU都可以自由运行工作负载。后续的补丁为该调度程序添加了tickless支持，并总结道：

        尽管 scx_example_central本身不足以用作生产调度程序，但可以使用相同的方法构建更具特色的中央调度程序。Google 的经验表明，这种方法对某些应用程序（如 VM 托管）具有重要的好处

此外，scx_example_pair 采用控制组实现了一种核心调度形式。scx_example_userland 调度程序“在用户空间实现了一个相当不成熟的排序列表 vruntime 调度程序，以演示大多数调度决策如何委托给用户空间”。该系列最后介绍了 Atropos 调度程序，它具有用 Rust 编写的重要的用户空间组件。信件封面还介绍了另一个调度程序 scx_example_cgfifo，因为它依赖于仍未合并到主线的 BPF rbtree 
补丁而未被包含在该系列中。它“为各个工作负载提供 FIFO 策略，并提供扁平化分层 vtree 用于控制组”，显然在 Apache Web 服务基准测试中比 SCHED_NORMAL 提供更好的性能。

## Prospects

BPF调度器类显然是核心内核社区难以接受的重要改动。它增加了超过10,000行的核心代码，并公开了许多迄今为止被深深隐藏在内核中的调度细节。这将承认一个通用调度器无法最优地服务于所有工作负载。一些人可能担心这将标志着完全公平调度器的工作结束，并**增加Linux系统的碎片化**。BPF调度器的开发人员则持相反的观点，认为能够自由实验调度模型，将加速完全公平调度器的改进。

这个子系统的最终结果如何还很难预测，但可以指出的是，迄今为止，BPF巨头已经成功地克服了几乎所有遇到的反对意见。在内核中锁定核心功能的时代似乎正在结束。看到这个子系统将会开启哪些新的调度方法将会是很有趣的。



























