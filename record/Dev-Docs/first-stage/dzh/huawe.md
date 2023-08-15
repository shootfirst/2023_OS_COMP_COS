你好Roman和列表成员，

我们希望实现一个可编程的调度器，以满足不同工作负载的调度需求。

使用BPF，我们可以轻松地为特定工作负载部署调度策略，快速验证，无需修改内核代码。这大大降低了在生产环境中部署新调度策略的成本。

因此，我们希望在您的补丁的基础上继续开发。我们计划将其合并到openeuler开源社区中，并利用社区不断演进和维护它。
（链接：https://www.openeuler.org/en/）

我们对您的补丁进行了一些更改：
1.适应openeuler-OLK-5.10分支，该分支大多基于长期支持的Linux分支5.10。
2.引入Kconfig CONFIG_BPF_SCHED以在编译时隔离相关代码。
3.修改了helpers bpf_sched_entity_to_cgrpid()和bpf_sched_entity_belongs_to_cgrp()，通过se->my_q->tg->css.cgroup获取调度实体所属的任务组。

我们有一些关于下一次Scheduler BPF迭代的想法，想与您分享：
1.在struct task_struct和struct task_group中添加tag字段。用户可以使用文件系统接口为特定工作负载标记不同的标签。bpf prog获取标签以检测不同的工作负载。
2.添加BPF hook和helper来调度进程，如select_task_rq和pick_next_task，以实现可扩展性。

这是一个新的尝试，后面肯定会有很多问题，但让调度器可编程是令人兴奋的。

祝好，
任志杰
