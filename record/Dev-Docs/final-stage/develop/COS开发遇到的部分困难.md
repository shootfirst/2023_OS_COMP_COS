本文档记录部分在开发COS过程中较难解决的问题及对应解决方法。

### 线程安全

shoot的时候，需要判断该线程是否还在运行，可能在抢占和dequeue的时候暂时不能运行，但是又马上可以运行了，此时，在被抢占和dequeue的时候，建议将nexttosched设置为null

同时，对于每一个cpu上的cosrq，会存在线程安全问题，cpu会操控自己的rq，同时，lord cpu也会修改，例如，shoot的时候



### COS线程相互抢占问题

线程安全问题

shoot     dequeue

设置        清空

​			    获取消息队列锁

打印        打印

dequeue清空——dequeue获取锁——shoot设置——shoot打印——dequeue打印

```c
[  435.701742] kernel produce msg: type: 2, pid 3095
[  435.701751] sched_getscheduler start! 3096
[  435.701792] kernel produce msg: type: 2, pid 3096

1 3095
[  435.701792] shoot thread 3095 to cpu 1
[  435.701902] select_task_rq_cos, lord_cpu 4
[  435.701940] select_task_rq_cos, lord_cpu 4
[  435.701968] kernel produce msg: type: 1, pid 3095
[  435.701970] check_preempt_curr_cos
[  435.701971] kernel produce msg: type: 1, pid 3096
[  435.701972] check_preempt_curr_cos

[  435.702115] shoot thread 3095 to cpu 0
[  435.702117] kernel produce msg: type: 7, pid 3095

[  435.702118] shoot thread 3096 to cpu 1

[  435.702189] sched_getscheduler start! 3095
```



### exit系统调用被打断导致的segmentation fault

```c
[  488.147158] exit start 3374

[  488.147170] kernel produce msg: type: 2, pid 3374

[  488.147172] kernel produce msg: type: 1, pid 3374

【do exit之后，两次dequeue一次enqueue被shoot打断，导致segmentation fault】

[  488.147178] kernel produce msg: type: 7, pid 3374

[  488.147180] shoot thread 3384 to cpu 6【shoot打断】

[  488.147181] shoot pid 3374 is not exist

[  488.147182] kernel produce msg: type: 2, pid 3374

[  488.147185] kernel produce msg: type: 4, pid 3374
```

解决方法：维护exit标志位，在do exit开始拉高；shoot时检查被抢占线程的该标志位，如果为高则shoot失败



### 漏增加exit标志位

```c
[ 3817.022287] exit start 6776

[ 3817.022299] kernel produce msg: type: 2, pid 6776

[ 3817.022300] kernel produce msg: type: 1, pid 6776

【在两此dequeue一次enqueue中被shoot了，也不会触发seq，泪目】

[ 3817.022305] shoot thread 6776 to cpu 3

[ 3817.022311] kernel produce msg: type: 2, pid 6776

[ 3817.022316] kernel produce msg: type: 4, pid 6776

【然后就在dead之后被抢占了，触发了用户态panic】

[ 3817.022429] kernel produce msg: type: 7, pid 6776
```

除了exit之后防止被别的cos线程抢占，也得防止自己被shoot。。。



### exit过程被cfs抢占

exit后拉高电平还是会被cfs抢占，解决方法：为电平读取和修改加锁即可



### 线程在两个CPU上运行

解决方法：修改抢占消息发送时间

```c
    6768    在CPU 3 运行
    6768    被cfs抢占, 入用户态队列

初态：
    6768    在用户态队列
    6779    在CPU 5 运行

    6768    shoot到 CPU 5
[  151.979058] kernel produce msg: type: 7, pid 6779
[  151.979059] shoot thread 6768 to cpu 5
    6779    被6768抢占,入用户态队列
    6779    shoot到 CPU 8
[  151.979076] shoot thread 6779 to cpu 8

[  151.979078] preempt by 6768 sched_class 8
[  151.979081] preempt by 6779 sched_class 8


    最终: 6779在CPU 5运行, 6768在CPU 8运行
[  151.979087] ------------[ cut here ]------------
[  151.979088] shoot thread 6760 to cpu 3
[  151.979088] WARNING: CPU: 8 PID: 6768 at kernel/rseq.c:95 __rseq_handle_notify_resume+0x403/0x500
[  151.979092] BUG: kernel NULL pointer dereference, address: 000000000000001f
[  151.979094] #PF: supervisor read access in kernel mode

[  151.979199] CPU: 5 PID: 6779 Comm: rocksdb Kdump: loaded Tainted: G        W          6.4.0+ #7

trace:
[  151.979312]  ? finish_task_switch.isra.0+0x4b/0x260
[  151.979316]  __schedule+0x3d2/0x1510
```


### dequeue之后不再enqueue

解决方法：未知，据调查是因为返回用户态的时候获取锁而blocked；

最后好像发现是vmware虚拟机问题，改换物理机之后解决



### 0和50延迟高

RocksDB实验的0%和50%延迟过高，可能是有时候比不过ghost的重要原因，解决方法：全链路追踪探究

最后发现是rocksdb实验参数和用户态写得有问题，已解决。



### 禁用CPU

qps过高会出现vmvare死机，显示禁用cpu，解决方法：vmvare对cpu的限制，最后改换物理机解决


### CGroup无法及时下CPU

当线程对应的CGroup时间片耗尽时，该线程应该立刻下CPU，这个过程由每次时钟中断都被调用一次的task_tick()调度类回调函数监测并执行强制线程下CPU的逻辑。

在实际开发中，我们发现当COS线程数较少时，task_tick()事件中会关闭sched_core，也即rq->core_sched == false，从而导致无法通过设置标志位的方法使对应CPU重新调度，而使线程立刻下CPU；但在COS线程数较多时没有问题。

由于CGroup适用场景为限制CPU使用率，当系统线程数较少时无需发挥过大作用，因而我们选择暂时忽视此bug。

