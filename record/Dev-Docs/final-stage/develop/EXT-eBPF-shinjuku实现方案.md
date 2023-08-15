## 三端思路

### main进程

1. 装载bpf程序

2. 创建两个管道

   1. 一个只读

      读取rocksdb的写入数据

   2. 另一个只写

      发送通知给rocksdb

3. 新建线程

   线程体内容：读取管道1、更新bpf程序、写入管道2

4. join

### rocksdb

Worker线程初始化流程：

1. 创建两个管道

   1. 一个只读

      读取main进程的通知

   2. 另一个只写

      给main进程写入数据

2. 创建CosThread

3. 写入管道2

4. 阻塞读管道1

5. 设置调度类为EXT

6. notify work

后续generator和worker通信：

1. worker在hashtable中标记自己为idle，然后发送idle信息到消息队列
2. main进程不断处理
3. generator挑选出那些hashtable为idle，并且已经收到rpc应答的worker线程，然后将rpc置零，将它们在hashtable中标记为runnable，然后发送

### bpf端

#### 流程

1. enqueue

   直接将任务入deque尾

2. dispatch

   1. 弹出deque头peek
   2. 若peek被标记为IDLE，则重新入队peek，回到1
   3. 否则，dispatch任务到SCX_DSQ_GLOBAL

3. preemption

   for cpu : 所有cpu

    1. 获取cpu当前正在润的task_struct curr

    2. 若调度类非EXT， continue；

    3. 决策是否需要抢占，不需要则continue

       1. 若curr被标记为IDLE，should_preempt = true；

       2. 获取curr的qos。如果比它qos更大的runqueue的元素数不为0，则 should_preempt = true

       3. else 若与它qos相等的runqueue的元素数不为0

          ​	若curr的时间片用完，should_preempt = true；

    4. if(should_preempt), scx_bpf_kick(cpu)

#### 数据结构

1. 共享内存/线程信息

   ```c
   // 外界通过访问该bpf map更新qos和runnable
   struct task_ctx {
       uint64_t last_ran;
       bool should_update_runtime;
       
   	int qos;
   	bool runnable;
   };
   
   // 同时也可以用于标识该task是否为ext调度类
   struct {
   	__uint(type, BPF_MAP_TYPE_HASH);
   	__uint(max_entries, 10000);
   	__type(key, int); // pid
   	__type(value, struct task_ctx);
   } task_sw SEC(".maps");
   ```

2. 调度队列

   ```c
   // 1. 一级队列
   // 通过两个单端队列实现双端队列
   struct runqueue{
   	int front_fd;
   	int back_fd;
   
   	int peek();
   	void pop_front();
   	void push_front(int pid);
   	void push_back(int pid);
   };
   uint64_t nr_queued = 0; // 记录runqueue中的成员总数
   
   // 2. qos多级队列
   // 限定一共 QOS_NUMS 个qos优先级。
   struct runqueue shinjuku_rq[QOS_NUMS];
   uint64_t qos_nr_queued[QOS_NUMS];
   ```

目前是这样的：

bpf map只能静态申请，所以会很丑陋

dsq无法访问peek



## 开发流程

1. 写出shinjuku一级队列，通过simple test
2. 完成preemption机制，通过看log/写测试验证
3. 改造rocksdb、main；增加qos

## 暂留问题

1. 增加对“被内核线程抢占”的处理

   在ghOSt的实现中，好像只有在被其他调度类线程抢占时，才会设置prio boost为true，并且入队到双端队列头。要插入队列头的原因如下：

   ```c
   (1) 该任务在被抢占时可能持有重要的锁，所以通过尽快重新调度任务，我们可以提高性能；
   (2) 因为Shinjuku算法假设任务不会被其他调度类抢占，所以尽快将任务重新调度到CPU上对于忠实地实现该算法非常重要。
   ```

   我觉得它说得很有道理。我们可以修改EXT内核，**新增一个bpf事件**，抢占时触发，具体可以模仿ghost的思路。







1. 消息队列

   解决下锁的问题

2. 消息队列+共享内存

3. 共享内存
