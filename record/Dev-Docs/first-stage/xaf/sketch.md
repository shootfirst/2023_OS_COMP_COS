用户态调度框架的实现一直是工业界的一大需求。

对于不同的负载，实际的调度场景会有不同的需求。比如，io密集型和计算密集型两种负载，前者需要及时地响应输入，因此处理输入的设备或者软件进程应该长时间出于可运行的状态，一旦对应的输入到达，处理进程就应该被及时唤醒被处理数据，因此此类调度场景应该使用灵活的调度算法，以方便处理输入数据的进程拥有更高的优先级，方便抢占；后者需要长时间占据cpu来进行计算，此类调度场景则需要相对固定的优先级，并没有太多抢占的发生。

对于底层，用户态的负载是无法被底层感知的。**内核没办法根据用户态的负载的特性来决定使用何种调度算法更加合适**。

因此需要设计一个用户态的调度框架来解决这个问题，而这样的用户态调度框架应该具有以下特性：

1. 能够与内核通信，时刻更新有关待调度进程和调度的决策等信息。
2. 实现了多种调度算法，可以针对不同的负载作出最合适的调度算法的选择。
3. 能够兼容cgroup，对于底层的cpu、io和内存做到很好的管理。

ghOSt就是一个由google实现的比较成熟的用户态调度框架，但是ghOSt虽然可以支持绑核操作（指定一个负责特定调度算法的agent所调度的任务只运行在设置好的cpu上），但无法彻底兼容cgroup。

SCHED-EXT是由google开发的内核补丁，同样也是用户态的调度框架，主要是基于eBPF注入向内核注入策略去实现。通过在内核修改，添加对应的eBPF事件，结合在eBPF程序编写的钩子函数，去在用户态触发相应的事件后实现用户和内核的通信，进而完成用户态的调度算法实现。

基于SCHED-EXT，我们想要实现以上提出的三个特性。我们希望在用户态实现多一些的调度算法，能应用于不同的负载，这对已经成型的SCHED-EXT框架并不是什么难事。**难点在于如何提高性能**。

ghOSt作为一个大量修改内核后的比较成型的调度框架，使用共享内存的方法使得用户与内核之间的通信非常方便，用户的决策时间传达到内核以及内核将待调度任务发送到用户都是微妙级别的。而SCHED-EXT只是作为一个内核补丁，支持了eBPF的事件实现，提供了注入调度算法的途径，在性能上的实现不如ghOSt完善。

但是，SCHED-EXT的优势在于流程简单，可以彻底兼容cgroup。如果对其在内核的实现进行修改并且让它变得完善，那么这个用户态调度框架就可以实现以上的三个特性进而成为一个十分完美的调度框架了。

因此，我们目前基于SCHED-EXT实现了以下的功能：

1. 多种调度算法的实现。
2. 使用共享内存的方法完善了用户与内核之间的通信。
3. 对其性能进行测试。

ghOSt对我们整个项目的构建提供了很多实现思路，因此在我们的文档里首先会提及ghOSt的相关设计思路，如果我们之后还想优化ghOSt，比如使其兼容cgroup或者增添新的调度算法或者优化其现有调度算法，有关ghOSt的知识是有必要掌握的。然后是对于我们改进的用户态调度框架的介绍，我们称之为ext调度框架。在这篇介绍了可以看到ext的设计实现思路以及不同的调度算法。

在文档的最后可以看到我们对于ext调度框架的未来展望。目前我们知识针对这个调度框架进行了些许提升，我们希望ext调度框架可以成为一个功能强大的调度框架，最终能够适用于各种各样的负载，并且表现出良好的特性。