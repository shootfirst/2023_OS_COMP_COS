

毫无疑问，有人会尝试将BPF引入内核的CPU调度器，这只是时间问题。在1月底，Tejun Heo与David Vernet、Josh Don和Barret Rhoden合作发布了30个补丁系列的第二版，旨在实现这一目标。将调度决
策延迟到BPF程序中可能会有一些有趣的事情，但要让整个开发社区接受这个想法可能需要一些工作。

BPF的核心思想是允许程序在运行时从用户空间加载到内核中；使用BPF进行调度具有潜力使得调度行为与目前在Linux系统中看到的有很大不同。“可插拔”的调度器概念并不是新鲜的；例如，在2004年的一
次讨论中，Con Kolivas提出了一系列注定失败的补丁，其中涉及到可插拔的调度器。当时，这个可插拔调度器的想法受到了强烈的反对；因为只有将精力集中在单个调度器上，开发社区才能找到一种方
式，满足所有工作负载，而不会将内核填满各种特殊目的的调度器的混乱。

当然，内核只有一个CPU调度器的想法并不完全准确；实际上，还有几个调度器可供应用程序选择，包括实时调度器和截止时间调度器。但是，在Linux系统上几乎所有的工作都在默认的“完全公平调度器”
下运行，它确实在各种从嵌入式系统到超级计算机的工作负载管理方面都做得很好。人们总是渴望更好的性能，但多年来几乎没有要求提供可插拔调度器机制的请求。

那么，为什么现在提出BPF机制呢？为了避免长时间的讨论，这个补丁系列的说明信详细描述了这项工作的动机。简而言之，这个论点是，使用BPF编写调度策略极大地降低了尝试新的调度方法的难度。自完
全公平调度器问世以来，我们的工作负载和运行它们的系统变得更加复杂；需要进行实验来开发适合当前系统的调度算法。BPF调度类可以以安全的方式进行实验，甚至无需重新启动测试机器。使用BPF编写
的调度器还可以提高针对某些特定工作负载的性能，这些工作负载可能不值得在主线内核中支持，并且部署到大型系统集群中也更加容易。

## Scheduling with BPF

这个补丁集添加了一个名为SCHED_EXT的新调度类，可以通过类似于大多数其他调用sched_setscheduler()的调用来选择它（选择SCHED_DEADLINE有点更加复杂）。它是一个非特权类，这意味着任何进程
都可以将自己置于SCHED_EXT中。SCHED_EXT被放置在优先级堆栈中的空闲类（SCHED_IDLE）和完全公平调度器（SCHED_NORMAL）之间。因此，没有SCHED_EXT调度器可以以一种阻止例如以SCHED_NORMAL
运行的普通shell会话运行的方式接管系统。它还建议，在使用SCHED_EXT的系统上，期望大部分工作负载将在该类中运行。

BPF编写的调度程序对整个系统是全局的；没有为不同的进程组加载自己的调度程序的规定。如果没有加载BPF调度程序，则放置在SCHED_EXT类中的任何进程将像在SCHED_NORMAL中一样运行。然而，一旦
加载了BPF调度程序，它将接管所有SCHED_EXT任务的责任。还有一个神奇的函数，BPF调度程序可以调用（scx_bpf_switch_all()），它将所有运行在实时优先级以下的进程移动到SCHED_EXT中。

实现调度程序的BPF程序通常会管理一组调度队列，每个队列都可能包含等待在CPU上执行的可运行任务。默认情况下，系统中每个CPU都有一个调度队列和一个全局队列。当CPU准备好运行新任务时，调度
程序将从相应的调度队列中取出一个任务并将其分配给CPU。调度程序的BPF部分大多实现为一组通过操作结构调用的回调函数，每个回调函数通知BPF代码需要进行的事件或决策。该列表很长，完整的列表
可以在SCHED_EXT存储库分支的include/sched/ext.h中找到。该列表包括：

        当一个新的任务进入SCHED_EXT时，prep_enable()和enable()这两个回调函数将通知调度程序。prep_enable()可以用于为该任务设置任何相关数据，它可以阻塞并用于内存分配。enable()则
        无法阻塞，它实际上启用了新任务的调度。

        select_cpu()回调函数用于为刚刚唤醒的任务选择一个CPU，并返回要将任务放置在的CPU编号。这个决策可以在任务实际运行之前重新审视，但它可能被调度程序用于唤醒选择的CPU（如果它当
        前处于空闲状态）。

        enqueue()回调函数将一个任务加入调度程序以进行运行。通常，该回调将调用scx_bpf_dispatch()将任务放置到选择的调度队列中，该队列最终将在任务运行时为其提供时间片长度。如果将片
        长指定为SCX_SLICE_INF，则在此任务运行时，CPU将进入无节拍模式。

        值得注意的是，enqueue()不必将任务放入任何调度队列；如果任务不应立即运行，它可以将任务暂时放在某个地方。但内核会跟踪这些任务，以确保没有任务被遗忘；如果任务滞留时间过长（默
        认为30秒，但超时时间可以缩短），BPF调度程序最终将被卸载。

        当一个CPU的调度队列为空时，调用dispatch()回调函数将任务分派到该队列中以保持CPU忙碌。如果调度队列仍然为空，调度程序将尝试从全局队列中获取任务。

        update_idle()回调函数将通知调度程序一个CPU何时进入或离开空闲状态。

        runnable()、running()、stopping()和quiescent()回调函数分别通知调度程序任务的状态更改。它们分别在任务变为可运行、在CPU上开始运行、从CPU上被取下或变为不可运行时调用。

        cpu_acquire()和cpu_release()回调函数通知调度程序系统中CPU的状态。当一个CPU对BPF调度程序可用时，回调函数cpu_acquire()将通知它这个事实。当一个CPU不可用时（例如，一个实时
        调度类可能已经使用它），将通过调用cpu_release()来通知它。


还有许多其他的回调函数用于控制组的管理、CPU亲和性、核心调度等。此外，还有一组函数可供调度程序调用以影响调度决策；例如，scx_bpf_kick_cpu() 可用于抢占正在给定CPU上运行的任务，并回
调调度程序以选择在该CPU上运行的新任务。

## Examples

最终的结果是一个框架，允许在 BPF 代码中实现各种调度策略。为了证明这一点，这个补丁系列包含了许多示例调度器。其中一部分是一个最小的“虚拟”调度器，它使用默认的回调函数；另一个则是一个
基本调度器，实现了五个优先级级别，并展示了如何将任务存储到 BPF 映射中。“虽然不是很实用，但它作为一个简单的示例很有用，并将用于演示不同的功能”。

此外，还有一个“中央”调度程序，它将一个CPU专用于调度决策，使得所有其他CPU都可以自由运行工作负载。后续的补丁为该调度程序添加了tickless支持，并总结道：

        尽管 scx_example_central本身不足以用作生产调度程序，但可以使用相同的方法构建更具特色的中央调度程序。Google 的经验表明，这种方法对某些应用程序（如 VM 托管）具有重要的好处

此外，scx_example_pair 采用控制组实现了一种核心调度形式。scx_example_userland 调度程序“在用户空间实现了一个相当不成熟的排序列表 vruntime 调度程序，以演示大多数调度决策如何委托给
用户空间”。该系列最后介绍了 Atropos 调度程序，它具有用 Rust 编写的重要的用户空间组件。信件封面还介绍了另一个调度程序 scx_example_cgfifo，因为它依赖于仍未合并到主线的 BPF rbtree 
补丁而未被包含在该系列中。它“为各个工作负载提供 FIFO 策略，并提供扁平化分层 vtree 用于控制组”，显然在 Apache Web 服务基准测试中比 SCHED_NORMAL 提供更好的性能。

## Prospects

这个补丁集目前已经发布了第二个版本，并且迄今为止还没有引起很多评论，也许太大了，无法进行辩论。然而，调度器维护者Peter Zijlstra在第一个版本中回应说：“我讨厌所有这些。Linus在过去多
次否决了可加载的调度器，这只是又一次——加上了整个BPF问题的额外缺陷。”然而，他继续审查了许多组成补丁，这表明他可能不打算完全拒绝这项工作。

BPF调度器类显然是核心内核社区难以接受的重要改动。它增加了超过10,000行的核心代码，并公开了许多迄今为止被深深隐藏在内核中的调度细节。这将承认一个通用调度器无法最优地服务于所有工作负
载。一些人可能担心这将标志着完全公平调度器的工作结束，并增加Linux系统的碎片化。BPF调度器的开发人员则持相反的观点，认为能够自由实验调度模型，将加速完全公平调度器的改进。

这个子系统的最终结果如何还很难预测，但可以指出的是，迄今为止，BPF巨头已经成功地克服了几乎所有遇到的反对意见。在内核中锁定核心功能的时代似乎正在结束。看到这个子系统将会开启哪些新的
调度方法将会是很有趣的。
