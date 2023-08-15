## 4.3 启动流程

> EXT下所有的调度算法共有一个框架—基于ebpf的bootstrap为主导的用户态框架。

接下来就详细地介绍bootstrap在用户态的运行。

### 4.3.1 bootstrap概述

在ext的框架下，有三个主要步骤：

1. bootstrap
2. sched_main_loop
3. destroy

为了统一叫法，这个流程被称为bootstrap大众化流程。

在这三个流程中，bootstrap主要负责main函数的传参以及bpf程序的搭建，是一个相对固定的流程；sched_main_loop主要负责具体task的调度，其中包括了具体调度算法的实现以及最终被调度的task的分发；destroy主要负责bpf程序的销毁，任务量最轻。

接下来将就这三个流程做详细说明。

### 4.3.2 bootstrap

> bootstrap(int argc, char ** argv)，作为main函数第一个调用的函数，接受argc和argv两个参数，起了初始化eBPF程序配置，处理参数和将eBPF程序打开并加载入内核使其发挥连接用户态和内核态的作用。
> 

bootstrap的主体流程：  

1. 初始化eBPF程序配置
2. 处理参数
3. 打开并加载eBPF程序


#### 初始化eBPF程序配置

在`bootstrap()`函数的开始，可能会初始化eBPF程序的配置。

* 重要常量的赋值：

    ```cpp
    struct sched_param sched_param = {
    .sched_priority = sched_get_priority_max(SCHED_EXT),
    };
    bool switch_partial = false;
    ```

    sched_priority字段顾名思义，可以指定调度算法的优先级。而switch_partial是一个可选参数，在处理参数的环节中，如果输入的参数有 **-p** （不同调度算法参数可能不一致），其被赋值为true，并且当前调度类会接管当前用户态的所有任务的调度工作。

* 信号量的响应函数以及bpf严格模式的开启：

```cpp
signal(SIGINT, sigint_handler);
signal(SIGTERM, sigint_handler);
libbpf_set_strict_mode(LIBBPF_STRICT_ALL);
```

ext自定义了进程终止函数`sigint_handler`，这个函数将预先设置为0的变量exit_req的重新赋值为1，从而退出某些条件为 `!exit_req` 的循环。

bpf严格模式是一种安全机制，旨在增加对 BPF 程序的验证和限制，以提高安全性和防止滥用。通过开启严格模式，libbpf 库会强制执行一系列规则和限制，以确保 BPF 程序的安全性和正确性。

最后，将当前进程设置为新的调度器来作为启用ext框架的第一步：

`err = syscall(__NR_sched_setscheduler, getpid(), SCHED_EXT, &sched_param);`  

这一行代码调用了系统调用sched_setscheduler(getpidi(), SCHED_EXT, &schd_param)，告知内核当前进程需要使用SCHED_EXT调度类去调用，为接下来在用户态实现具体调度类做了铺垫。  

#### 处理参数

bootstrap接受main函数的两个参数：int argc, char ** argv。这是典型的处理参数的模型。

以cfs调度类处理参数的方法为例：

```cpp
while ((opt = getopt(argc, argv, "b:ph")) != -1) {
  switch (opt) {
    case 'b':
      batch_size = strtoul(optarg, NULL, 0);
      break;
    case 'p':
      switch_partial = true;
      break;
    default:
      fprintf(stderr, help_fmt, basename(argv[0]));
      exit(opt != 'h');
  }
}
```

1. 在while循环中不断接受参数
2. 使用getopt系统调用来获取参数，赋值到opt
3. 根据opt的值进入不同的case

从实现来看，ext框架下的调度类接受的参数为一个小写字母，而每个小写字母都有其独特的作用。  

#### 打开并加载eBPF程序

eBPF程序使得用户无需修改内核模块，可以直接将编写好的函数加载进入内核模块中，使得事件发生时内核调用的不是原来的函数而是用户在eBPF程序定义好的钩子函数。通过这种方式，用户态的调度类从内核获取需要被调度的任务，并且再拿到了内核任务之后复制一个几乎一模一样的副本，结合在用户态实现的数据结构和相关调度算法，将需要调度的任务按预想的逻辑进行排序，从而决策出下一次需要被运转到内核中执行的任务。  

在介绍如何打开并加载eBPF程序之前，先介绍相关背景知识。  


#### eBPF程序背景知识

- eBPF程序注入内核大体过程：eBPF 程序本身使用 C 语言编写的 “内核代码”，注入到内核之前需要用 LLVM 编译器编译得到 BPF 字节码，然后加载程序将字节码载入内核。当然，为了防止注入的 eBPF 程序导致内核崩溃，内核中有专门的验证器保证 eBPF 程序的安全性，如 eBPF程序 不能随意调用内核参数，只能受限于 BPF helpers 函数（BPF helpers函数来自于libbpf库，需要手动下载）；除此之外，eBPF 程序不能包含不能到达的逻辑，循环必须在有限时间内完成且次数有限制等等。  

![bpf](/record/xaf/image/ebpf-instruct.png)  

参考：https://morven.life/posts/knowledge-about-ebpf/  

- bpf_map：常驻于内存，用户用户态与eBPF程序之间，以及eBPF程序之间和内核之间的通信，bpf_map是实现用户和内核通信的重要角色。bpf_map在eBPF程序中定义，需要声明其类型、条目限制数，值的类型，并且根据需要进行命名。在用户态需要打开bpf_map的文件描述符以获取对bpf_map的控制权，进而用户可以对bpf_map进行插入、查询、删除等操作。  
- struct_ops：在修改了内核，内核态里增加了用户定义好的结构体的情况下，struct_ops可以定义内核结构体里成员到eBPF程序编写好的函数的映射，在用户态里可以通过libbpf库里的相关api获取这一类结构体变量指针，并将结构体里的成员和在eBPF程序里编写好的函数进行一一映射，而这些结构体里的成员，实际上是对应了独立的内核事件，当事件被触发时，相应的eBPF钩子函数就会被调用。  



#### eBPF程序构建整体流程

在了解了eBPF程序背景知识后，再来看看在ext框架里eBPF程序是如何运行的。

整体流程可以分为四步：

1. 打开eBPF程序
2. bpf字节码加载进入内核
3. 获取bpf_map文件描述符
4. 将bpf钩子函数和内核事件绑定

接下来以cfs调度算法为例详细介绍这四个步骤。

- 打开eBPF程序：
  
    源代码：
    
    ```cpp
    skel = scx_cfs__open(); // scx_cfs是bpf程序名称
    
    // skel的定义如下：
    static struct scx_cfs *skel;
    ```
    
    skel(skeleton)，为eBPF程序的骨架，在eBPF开发中，使用骨架来加载、验证和管理eBPF程序。
    
    <aside>
    当scx_cfs.bpf.h程序被编译时，首先会生成程序骨架，骨架是一个用于加载、验证和管理eBPF程序的框架，它提供了与eBPF程序相关的各种功能和操作的接口。然后生成eBPF字节码，字节码可以在linux内核中执行。在生成eBPF字节码的过程中，会生成skel.h文件，这个文件包含了与eBPF程序骨架相关的定义和数据结构，可以在用户空间代码中使用，以与eBPF程序进行交互。
    
    </aside>
    
    skel成功生成后，可以通过其中的两个字段：bss和rodata来分别获取在eBPF程序中定义的变量和常量。这些变量和常量可以扮演用户态程序和内核通信之间的重要角色，比如，在cfs调度算法中，可以在eBPF程序中定义nice值变量，当触发任务入队的钩子函数时将该变量赋值，同时，在用户态也可以看到这个值的变化。还可以通过skel的maps字段来实现bpf_map在用户态的构造，来实现用户态和内核之间信息的传递。
    
- bpf字节码加载进入内核：
  
    源代码：
    
    ```cpp
    err = scx_cfs__load(skel);
    ```
    
    将骨架skel加载进入内核后，内核中即可以执行与该骨架相关联的eBPF程序。
    
- 打开bpf_map文件描述符：
  
    源代码：
    
    ```cpp
    // 包含了从内核中获取的需要被调度的任务
    enqueued_fd = bpf_map__fd(skel->maps.enqueued);
    // 包含了从用户态决定的需要传到内核中被调度的任务
    dispatched_fd = bpf_map__fd(skel->maps.dispatched);
    
    // enqueued-map的定义
    struct {
    __uint(type, BPF_MAP_TYPE_QUEUE);
    __uint(max_entries, USERLAND_MAX_TASKS);
    __type(value, struct scx_userland_enqueued_task);
    } enqueued SEC(".maps");
    
    // dispatched-map的定义
    struct {
    __uint(type, BPF_MAP_TYPE_QUEUE);
    __uint(max_entries, USERLAND_MAX_TASKS);
    __type(value, s32);
    } dispatched SEC(".maps");
    ```
    
    `skel->maps.enqueued`和`skel->maps.dispatched`都是在eBPF程序里定义的一种映射关系，前者是入队列，后者是出队列，分别负责任务从内核到用户态的入队和用户态到内核的出队。内核当中一旦有任务需要被调度，触发了相关的钩子函数之后，这些需要被调度的任务就会加入到enqueued_map中，用户态相应地会感知到新的需要被调度的出现，于是开始基于特定调度算法对任务进行调度。当用户态决策出哪些任务会被传回内核中被执行后，这些任务就会加入到dispatched_map中，由eBPF程序将这些任务传回内核。  
    
- 将bpf钩子函数和内核事件绑定：
  
    源代码：
    
    ```cpp
    ops_link = bpf_map__attach_struct_ops(skel->maps.userland_ops);
    
    // ops_link的定义
    static struct bpf_link *ops_link;
    
    // userland_ops的定义
    SEC(".struct_ops")
    struct sched_ext_ops userland_ops = {
        .select_cpu     = (void *)userland_select_cpu,
        .enqueue        = (void *)userland_enqueue,
        .dispatch       = (void *)userland_dispatch,
        .prep_enable    = (void *)userland_prep_enable,
        .init           = (void *)userland_init,
        .exit           = (void *)userland_exit,
        .timeout_ms     = 3000,
        .name           = "userland",
    };
    ```
    
    `skel->maps.userland_ops`是在eBPF程序定义的内核结构体schd_ext_ops的实现，sched_ext_ops是在内核中定义的有关ext调度类事件信息的结构体，其中的成员代表着一个调度事件或者一些参数的设置，例如`init`代表着调度类的初始化事件。在eBPF程序中，userland_ops并没有被直接指明为maps字段，而是被声明为结构体的操作字段，通过`bpf_map__attach_struct_ops`这个函数，可以将userland_ops作为一个映射并且使其和一个操作该映射的结构体绑定起来，并将关联后的结构体指针存储在ops_link中，便可以使用该结构体指针去执行map的各种操作，如查找、插入、更新和删除。
    
    

至此，整个bootstrap流程的介绍就结束了，bootstrap结束之后，用户态程序拥有了能够与内核进行直接信息交换的各种映射，以及可以获取在eBPF程序中定义的各种变量，这些映射和变量都为调度算法在用户态的实现提供了很大的帮助。

### 4.3.3 sched_main_loop流程

> sched_main_loop流程本质上就是调度算法的运行流程。在ext框架的用户态，每个调度算法首先要被实现，能支持pick_next_task()这样的获取下一个需要被调度的任务的api等，然后调度算法被搬上sched_main_loop流程，按部就班的执行从内核中获取需要执行的任务、调度任务、任务放弃cpu的流程。当然，在某些特定的调度算法中这个流程可能会稍微变化。
> 

sched_main_loop的三个流程：

1. 内核中需要被调度的任务入队
2. 将选择好的任务发送至内核使其被调度
3. 放弃cpu

源代码如下：

```cpp
static void sched_main_loop(void) {
  while (!exit_req) {
    drain_enqueued_map();
    dispatch_batch();
    sched_yield();
  }
}
```

#### drain_enqueued_map

- 作用：在eBPF定义的enqueued_map映射里不断取值，即需要被调度的任务，并且对待调度的任务按需执行响应操作。
- 步骤：（在无限循环里进行）
    1. `bpf_map_lookup_and_delete_elem(enqueued_fd, NULL, &task)`
       
        该行代码的作用是：根据enqueued_fd文件描述符从enqueued_map这个映射中查找指定的值，将这个值的指针赋值到task变量的地址，并且将这个值从enqueued_map中删除。由于这个被指定的值设置成了NULL，即说明可以从enqueued_map中任意取出一个元素。如果取出失败了，即enqueued_map空了或者发生了错误，那么就直接返回。这与该函数的意义--“榨干enqueud_map”是完全符合的，不断地从enqueued_map中不管顺序地取出元素，直到enqueued_map被“榨干”为止。
        
    2. 建立task的共享内存
       
        在生成测试线程时，每个测试线程会根据线程组号在/etc/cos/shm目录下生成相应的文件，这个文件被视作该线程组内所有线程和用户态进行通信的区域，即共享内存区域。通过这段共享内存区域，用户可以自定义对于task注入运行截止时间（ddl）的设置，而这样的定义会在这段共享内存中被获取，依照这样的设置可以选用合适的调度算法，比如fifo进行对任务的调度。
        
        源代码：
        
        ```cpp
        if (tgid != getpid()) {
            // 由于本pid下的线程不需要被调度，因此需要避开
            printf("%dagent dispatch thread %d parent proccess%d\n", getpid(),
                task.pid, tgid);
        
            if (tgid2hashtable.count(tgid) == 0) {
                char buf[128];
                snprintf(buf, sizeof(buf), "/etc/cos/shm/shm_%d", tgid);
                int shm_fd = open(buf, O_RDWR);
                void *shm =
                    mmap(NULL, SHM_SIZE, PROT_READ | PROT_WRITE, MAP_SHARED, shm_fd, 0);
                if (shm == MAP_FAILED) {  // the client doesn't use share memory
                    goto enqueue;
                }
                tgid2hashtable[tgid] = LFHashTable<struct entry>(shm, SHM_SIZE, 0);
            }
        
            struct entry tmp = tgid2hashtable[tgid].Get(task.pid);
            memcpy(&(task.data), &tmp, sizeof(struct entry));
        }
        ```
        
        由代码可以看出，这里利用了mmap系统调用和哈希表，首先将对应的文件打开，获取文件描述符后建立共享内存并且获得共享内存指针，接着将需要被调度的线程的线程组号和对应的文件的共享内存指针映射起来，再通过线程号从中获取条目，将刚刚拿到的待调度的任务中的数据（`task.data`）复制到条目中，`task.data`包含着注入任务累计运行时间等重要信息，这些信息在相应的调度算法中会起作用。
        
    3. enqueue
        这一步被封装为了函数`vruntime_enqueue(&task)`。  
        
        每个从enqueued_map里获得的任务（`task`），都需要在用户态定义的与自身调度算法相符合的数据结构中入队（`enqueue`），即往自身定义的队列中加入新的元素。
        
        例如，在cfs调度算法场景中，自定义了一个运行队列数据结构`cfs_rq`，这一步就可以归结为一行代码：
        
        ```cpp
        cfs_rq.enqueue_cfs_task(curr); // 将当前任务curr加入运行队列中
        ```
        
        当然，不同的调度算法中在enqueue时可能需要进行一些特定的操作，但大体的入队流程是：
        
        1. get_enqueued_task(bpf_task→pid)：  

            这里的bpf_task实际上就是在`drain_enqueue_map()`函数里传入`vruntime_enqueue`函数的`task`，都是来自于内核的任务。这一步需要注意的是，在ext的调度框架下，每个调度场景都预先定义好了`USERLAND_MAX_TASKS`个用户态任务类型（`enqueued_task`）的任务：  

            ```cpp
            struct enqueued_task tasks[USERLAND_MAX_TASKS];
            ```

            用户态任务类型`enqueued_task`是对于来自内核的任务的一个简要的副本，不同的调度算法会给出不同的定义，主要区别取决于调度算法决策下一个调度任务的关键因素是什么，`enqueued_task`是根据这些关键因素来定义的。比如，cfs场景下的调度关键是虚拟运行时间的计算，而要计算虚拟运行时间，就应该获取任务的nice值，这个nice值又只能通过内核来获取，因此在构造cfs场景下用户态的任务副本是，nice值就是必须要从内核任务`bpf_task`获取的一个关键因素，它是`enqueued_task`的一个成员变量。   

            cfs场景下`enqueued_task`的定义：  
            ```cpp
            struct enqueued_task {
                public:
                enum class State : uint32_t {
                    kBlocked = 0,  // Task阻塞
                    kRunnable,     // Task可以运行
                    kRunning,      // Task正在运行
                    kDone,         // Task被终止或者运行结束
                    kNumStates,
                };
                __u64 sum_exec_runtime_;
                enqueued_task() : sum_exec_runtime_(0), vruntime_(0) {
                    set_state(State::kRunnable);
                }

                Duration vruntime_;

                bool operator<(const struct enqueued_task &s) const {
                    if (vruntime_ == s.vruntime_) {
                    return (uintptr_t)this < (uintptr_t)(&s);
                    }
                    return vruntime_ < s.vruntime_;
                }

                void set_state(const State &state);
                void set_nice(uint64_t nice) { nice_ = nice; }
                uint64_t get_nice() { return nice_; }
                void set_weight(uint32_t weight) { weight_ = weight; }
                void set_inverse_weight(uint32_t inverse_weight) {
                    inverse_weight_ = inverse_weight;
                }
                uint32_t get_inverse_weight() { return inverse_weight_; }
                bool is_runnable() { return task_state_ == State::kRunnable; }
                bool is_running() { return task_state_ == State::kRunning; }

                private:

                uint64_t nice_;
                uint32_t weight_;
                uint32_t inverse_weight_;

                State task_state_;
            };
            ```

            通过`bpf_task->pid`，直接通过tasks数组随机访问一个已经静态初始化好的`enqueued_task`，基于这个访问到的`enqueued_task`需要从`bpf_task`里获取nice值以执行cfs场景下的调度算法。  

            于从tasks数组通过pid获取`enqueued_task`相对应的是，通过pid从tasks数组里获取pid：  

            ```cpp
            static __u32 task_pid(const struct enqueued_task *task) {
                return ((uintptr_t)task - (uintptr_t)tasks) / sizeof(*task);
            }
            ```

            在每个调度算法里，都会有结构体`scx_userland_enqueued_task`的定义，这个定义在对应的common.h头文件中，这个结构体里的成员都是这个调度算法所必须的一些成员变量，这个结构体被作为`enqueued_map`中的值。  

            cfs场景下`scx_userland_enqueued_task`的定义：  
            ```cpp
            struct scx_userland_enqueued_task {
                __s32 pid;
                u64 sum_exec_runtime;
                u64 weight;

                __s32 tgid;
                uint64_t nice; // cfs场景下的关键变量
                struct entry data;
            };
            ```

        2. `update_enqueued(curr, bpf_task)`：  

            根据内核的`bpf_task`与自身调度算法对当前需要加入运行队列的任务`curr`进行一些关键值的计算等操作，例如cfs场景下每个元素在入队前都需要计算虚拟运行时间，而计算虚拟运行时间需要来自内核的`bpf_task`的nice值和实际累计运行时间，eBPF程序使得用户与内核通讯的高效性在此得见。  

        3. enqueue_task：将当前更新获得到的任务`curr`加入运行队列中，以供后续的调度操作。通常情况下，这个入队操作实际上就是排序的过程，这取决于当前用户态是何调度算法，采用的是什么数据结构。  

#### dispatch_batch

在这一步，需要完成任务的分发。每次从`enqueued_map`获取来自内核需要调度的任务后，就需要通过自身的调度算法计算需要被传到内核去执行的任务是哪些。在ext框架下，每次分发默认的任务数上限为8（定义好的`batch_size`），即每次传到内核去执行的任务的个数最多为8个，这取决于用户态当前的运行队列的可以被调度的任务总数。

步骤：

1. `pick_next_task()`：根据自身调度算法选取下一个需要被执行的任务
2. `dispatch_task(pid)` ：执行`err = bpf_map_update_elem(dispatched_fd, NULL, &pid, 0);`来更新dispatched_map中的元素，再由相应的事件触发后通过dispatched_map来进行任务到内核的分发。

源代码如下：  
```cpp
static void dispatch_batch(void) {

  for (i = 0; i < batch_size; i++) {

    struct enqueued_task *task;
    __s32 pid;

    task = cfs_rq.pick_next_cfs_task();

    pid = task_pid(task);

    dispatch_task(pid); // bpf_map_update_elem(dispatched_fd, NULL, &pid, 0)，即更新dispatched_map
}
```

#### sched_yield

在这一步，会调用sched_yield()系统调用来放弃当前线程对于cpu的占用，让调度器去调度其它线程。不让用户态调度线程一直开启是有原因的，这一步的意义对于性能非常重要。

其实，ext的整体实现思想是：只有在有需要的时候才开启唤醒调度线程去调度来自内核的任务，在没有需要的时候尽可能的让不必要的调度线程处于可以运行但不占用cpu的状态即可。

这种思想的实现方法体现在eBPF程序和诸多钩子事件中。

在内核态里，当触发了enqueue事件时，会调用eBPF程序里的`enqueue_task_in_user_space`函数，来将这个在内核里需要被调度的任务进行初始化、加入`enqueued_map`的映射中，并且将值原本是false的`usersched_needed`变量设置为true。在触发了dispath事件后，如果usersched_needed为true，则会唤醒用户态的调度线程并开启调度操作。

综上可知，在必要的时候放弃cpu可以使得ext框架的性能得到进一步提高，因此这一步是至关重要的。

### 4.3.4 析构

在触发了终止信号后，exit_req变量由0被变为1。sched_main_loop循环退出。

之后分别执行：

1. bpf_link__destroy(ops_link)
2. scx_cfs__destroy(skel)

即将控制钩子函数的结构体的链接和eBPF程序的骨架给销毁。