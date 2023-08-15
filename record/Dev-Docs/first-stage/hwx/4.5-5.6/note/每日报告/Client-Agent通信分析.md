写着写着我自己都麻了。这是不是client和agent的通信方式？还是说它只是or什么什么玩意内部通信方式？我发现压根没有agent会用到futex啊。麻了。真麻了。

### 前言

之前说到可以分为Agent端和Client端，Agent端跑agent，Client端跑如simple_text、experiment之类的application。

ghost的kernel和userspace间会通信，同样的，Agent端和Client端也需要进行通信。

Client端可以通过通信，向Agent端传递消息，来：

1. 控制调度决策。

   比如说，它可以设置一个线程状态从running变成idle；（EDF算法）它可以设置调整线程的ddl；它还可以改变QoS，从而影响调度决策。

2. 获取线程信息。

   比如说，Client可以等待直到线程Runnable，然后再开始做事。

ghost为了支持这种通信，提出了两种解决方案：

1. Futex（详见`ThreadWait`）

   其实跟notifcation差不多。Client通过`MarkIdle`、`WaitUntilRunnable`等等等方法

2. 共享内存（详见`PrioTable`）