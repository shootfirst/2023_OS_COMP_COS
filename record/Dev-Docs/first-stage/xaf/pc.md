> 在ghOSt的用户态调度框架里，每个调度类主要以两大模型来呈现出各自的效果，分别是per-cpu和centralized。per-cpu场景下，每个agent和一个cpu绑定起来，所有由这个agent管辖的任务都将在这个cpu上被调度和执行；在centralized场景下，由一个global-agent去掌管所有agent所掌管的任务，自由且有效地将这些任务调度到合适理想的cpu上（指负载比较小更适合调度当前任务的cpu）。因此，对于整个ghOSt，per-cpu和centralized两个场景的分析是非常重要的。
>

<aside>
💡 per-cpu场景和centralized场景的差别主要体现在整体框架上，因此会详写per-cpu场景来展示ghOSt，对于centralized场景只会添加整体框架和一些必要的补充说明
</aside>


### Per-cpu

- 每个agent对应着一个cpu，在agent的初始化时每个agent都会和一个cpu绑定起来，每个agent对应一个调度类，而由这个调度类所掌管的任务最终都会被调度到这个cpu上被执行

绑定体现：

```cpp
for (const Cpu& cpu : *enclave_.cpus()) {
      agents_.push_back(MakeAgent(cpu));
      agents_.back()->StartBegin();
    }
```

在上面这段代码中，agents_是一个装有agent泛型类的vector容器，对于每个enclave_（在lib篇会有介绍，只要知道enclave是掌管agent和调度类的集群就好）下的所有cpu作循环，于是对于每个cpu都会调用`MakeAgent`方法来构造与该cpu相关联的特定的agent，这就是per-cpu的字面意思，也是该场景的关键所在。

#### 执行流程

> per-cpu场景的体现最好拿一个具体的agent以及调度类来作说明，于此同时可以展示出整个ghOSt是如何以一个具体的调度类运作的，从agent的创立开始，再到agent线程的诞生，这一切的一切最终都是为了给agent和client线程（**创建出需要被调度的任务的线程**）的通信作准备。在这一章节将会以fifo（First-Input-First-Out）这个调度算法（字面意思，先进先出的调度算法）作展示，说明per-cpu场景下的入口以及整个流程的起初流程，在这之后会逐渐揭开ghOSt的大幕。
> 

##### 流程之前

对于每一个调度类，都需要一些固定的参数或者特定的参数的组合，使用google ABSL（在之前的环境配置篇中提到的google的c++库）的ABSL_FLAG方法将参数传入用户态：

```cpp
// 传入cpu位图，即cpulist，调度类能够使用的cpu编号
ABSL_FLAG(std::string, ghost_cpus, "1-5", "cpulist");
```

而这些传入的参数会在每个调度类的main函数下的前四行有所体现：

```cpp
// 初始化命令对象
absl::InitializeSymbolizer(argv[0]);
// 将命令由shell传入
absl::ParseCommandLine(argc, argv);

// 定义config变量，config中包含了该agent所需要的所有参数，如cpulist，enclave_fd(enclave的目录路径，之后会提及)等
ghost::AgentConfig config;
// 解析config参数，即将config中所包含的参数赋值到全局变量中，由全局变量参与之后的流程
ghost::ParseAgentConfig(&config);
```

这就是在整个正式流程开启之前的工作了。

#### 通信进程的建立

> 通信进程的建立过程就是整个agent搭建起来的过程，而这个agent搭建起来的过程被ghOSt封装得非常完善而且抽象意义非常通俗易懂。接下来仍然会以多个步骤的形式来展现agent的进程的建立过程。在agent进程建立起来之后，在整个用户态就存在了一个可以直接调度用户态程序的调度器，因此这对于agent和client之间的通信显得尤为重要。
> 

##### ShareBlob的建立

> 在计算机领域，BLOB是Binary Large Object的缩写，指二进制大型对象。BLOB通常用于数据库管理系统中，用来存储二进制数据，例如图像、声音或其他大型媒体文件。BLOB可以存储多种数据类型，包括文本、图像、音频和视频等。BLOB通常被视为二进制数据类型，它们不会被数据库处理或修改，而是被视为未经处理的数据块。
> 
- 作用：在父进程和子进程之间就地构造的一块共享内存区域
- 注意：
    1. 这个辅助类是用于父进程和 fork 出来的子进程之间同步的一块共享内存。它只能在共享内存区域中就地构造（in-place construction），否则父进程和子进程将拥有不同的 blob 副本。虽然可以使用一个简单的结构体，但这个类将自动就地构造其成员
- 最重要的字段：`blob_`

ghOSt框架自定义的share memory(共享内存)定义的一块共享内存，其共享内存的建立流程也是被ghOSt写得比较复杂，主要干了一件事情是—使用mmap系统调用来获取父进程和子进程之间共享内存的地址，然后计算出了父进程和子进程之间数据（在share memory之间被定义为data_）存放位置，以及client的数据存放位置（在后续的agent和client之间通信的时候用到）。

大致的共享内存建立流程如下：（在ghOSt里，共享内存是非常重要的一环，它贯穿了agent父进程和子进程之间的通信，agent进程和kernel进程之间的通信，因此详细介绍是非常有必要的）

1. 使用原子定义的变量`unique`来作计数（在一个很多agent被同时建立的pc端，多块共享内存被建立的情况下，将它们区分开来是非常有必要的，因此使用计数来加以区分），同时将共享内存的名字作为参数传入`CreatShmem`函数中去实际地建立共享内存
2. 在整个/proc文件夹下寻找对应当前进程的pid文件，接着在这个文件里面寻找已经创建的共享内存文件，这时候因为对应的共享内存文件还没有被建立，因此这时候找到的共享内存文件的文件描述符会是-1，这样的检查是有必要的，防止之后会创建相同的共享内存文件发生数据的重叠或者覆盖
3. 调用系统调用来在当前进程目录下创建特定名字的共享内存文件
4. 将创建的共享内存文件裁剪到预设好的大小，在ghOSt里面这个大小是按照每一页4096个字节去创建的，有较高的自由度，因此不必太过在意
5. 使用mmap系统调用来创建预设大小的共享内存，将其地址赋值给变量`shmem_`
6. 之后就是依照已经mmap出来的共享内存地址按照预设的规则来分配data_的地址，header的地址和client的地址
7. 最后这块共享内存，即shmem会被当做返回值返回，至此，agent父进程和子进程之间的共享内存就成功建立好了

将这块共享内存转换为二进制数据之后，一个命名为sb的变量，其中转载的是二进制数据就成功建立了

- 其它用到的类：
    1. Notification
       
        创建了四个类型为Notification的变量，分别是：
        
        1. agent_ready：标志着子进程，即agent进程准备就绪
        2. kill_agent：标志着父进程要杀掉子进程
        3. rpc_pending：父进程和子进程之间的rpc通信，这里指子进程在等待父进程的的指令
        4. rpc_done_：父进程和子进程之间的rpc通信，这里指父进程结束了对于子进程的命令
    
    既然是agent父进程和子进程之间的通信用的共享内存，这块共享内存还承载了使用条件变量来进程父子进程之间唤醒的特殊操作，Notification作为通信类很好得实现了一些诸如sleep和wake_up的通信操作，这些通信操作会在之后的父子进程的唤醒等动作大放异彩。
    

##### ForkedProcess

在AgentProcess类里面，有一个变量名叫agent_proc_变量，这个变量有着至关重要的作用—统筹了agent父进程和子进程，作为自定义的ForkedProcess类，其中有着能够完美区分父进程和子进程的`IsChild()`方法，在特定的构造函数里面使用了`fork()`系统调用来进行父进程和子进程的分隔。接下来会对ForkedProcess这个类有关fork()系统调用的部分作介绍，在通信进程的建立过程中，只需要了解ForkedProcess的构造函数过程就可以了，因为在ForedProcess这个类的构造函数里就发生了通信进程，即agent进程的父进程和子进程的构建。

- ForkedProcess构造函数的流程：在通信进程的建立过程中调用的ForkedProcess构造函数时使用时以`stderr`为参数的构造函数，这个stderr是取在前期准备过程中的定义的config变量中的stderr变量，是常规的标准错误输出值—2
1. 获取当前进程的pid，赋值到变量ppid上
2. 检测当前是否有多线程到达，如果当前有多线程到达，那么会存在fork的风险（个人认为这里存在着一些疑问，在源代码里比较难体现何时代表着多线程到达以及危险在何处）
3. 调用fork系统调用，其返回值赋值到变量p上，用于区分父进程和子进程
4. 首先是子进程，即当`p == 0`时
    1. 将当前的标准错误输出值dup一次
    2. 有可能父进程是有多个子进程的，而在通信进程的场景下，我们只需要通信进程fork一次，即出现一个父进程和子进程即可，因此调用了absl库里的特定函数去清楚掉了可能存在的父进程的所有子进程
    3. 防止成为孤儿进程：有可能会出现父进程已经被杀死但是子进程仍然存在的情况，因此当调用能够返回父进程的pid的系统调用`getppid()`时，将其与ppid作比较，如果不相等，说明当前父进程已经被杀死，这时直接返回false的error
    4. 最后，使用setpgid(*/*pid=*/*0, */*pgid=*/*0)的系统调用将自身与父进程彻底分隔开，以保证子进程不会接受到父进程所接收到的信号。虽然这样操作很容易出现孤儿进程的情况，但是在ghOSt的场景下，这种情况的诞生是必然的—我们必须要维护一个父进程来调度用户态下特定的任务，而当接收到这些任务时我们只需要将任务丢给子进程去完成具体的调度就好了。只要对于父进程和子进程的消亡控制得当（在ghOSt场景里父进程会接收到sigint信号被kill，被kill时会触发条件变量来把子进程kill掉），就可以避免孤儿进程出现的情况，并且很好地实现目的。
    5. 来说一说setpgid()的作用吧，它是使得子进程脱离父进程管辖的重要功臣。在系统调用的注释里它是这样说的：
    
    ```
    /* Set the process group ID of the process matching PID to PGID.
       If PID is zero, the current process's process group ID is set.
       If PGID is zero, the process ID of the process is used.  */
    ```
    
    因此，同时将pid和pgid设置为zero，会创建一个新的进程组，而当前的子进程的pid就成为了这个进程组的组长，此时子进程和父进程就隔离了开来，同时享受着终端的信号，因此不难得知在子进程里必须做点什么特殊的工作来确保子进程不会被胡乱地杀死来产生一些奇怪的错误。
    
5. 父进程：父进程干的事情就很简单了，只是定义了对于子进程状态可能发生终止或者停止的`SIGCHILD`信号做了一些特殊操作：获取子进程的进程描述符，然后在特殊情况下（如子进程返回值大于0，正常情况下子进程返回值应该是0）作异常处理就好了。之后还用了RALL机制来拿到当前进程返回值后建立子进程对于ForkedProcess类指针的映射。
6. 将child_变量赋值为父进程和子进程的返回值，这对于区别父进程和子进程有着非常重要的作用，在IsChild()方法里child_ == 0就是对于从ForkedProcess类的构造函数里出来的进程究竟是父进程还是子进程做了很好的判断

ForkedProcess类至此就完美地建立起来了，agent_proc_被赋值后干的第一件事情就是判断当前出来的进程是不是父进程，如果是父进程，它将在条件变量里等待着子进程发出的`agent_ready_.Ready()`信号，然后返回到main函数里去等待用户态任务的出现。而在这里出现的agent_ready正是shareblob里面的字段，接下来的小节里将会介绍—优美的条件变量。

##### 优美的条件变量

ghOSt里使用了Notification类去实现了进程之间的通信，最重要的三个方法：

1. Notify()，告知持有该变量的进程所有准备工作都做好了，可以继续往下行进了，于是被告知的进程会继续接下来的工作
2. WaitForNotification()，告知持有该变量的进程等待通知
3. Reset()，将Notification类的变量设置为NoWaiter，即没有需要等待的进程，需要下一次操作如Ready()或者WaitForNotification()等该变量才发挥特定作用

在探究ghOSt中的过程中，Notification的具体实现其实是吸引人的，接下来让我们详细地看看这几个函数。

在看一下这几个函数之前我们要知道一个Notification变量的几种状态：

1. kNoWaiter：没有正在等待的进程
2. kWaiter：有正在等待的进程
3. kNotified：当前进程已经可以苏醒了

而这几种状态都是在Notification类里面一个名为notified_的变量当中去体现的，其定义如下：

```cpp
// The notification state.
  std::atomic<NotifiedState> notified_ = NotifiedState::kNoWaiter;
```

可以看到，起初notified_的状态其实是kNoWaiter，这些状态其实就是struct enum class类当中的状态。

- WaitForNotification()：

在一个while(true)循环里执行以下逻辑

1. 原子地load出Notification的状态notified_
2. 如果已经得到了kNotified状态那就直接返回，因为当前进程已经被注意到了
3. 如果当前变量的状态是kNowaiter，那么会原子地将当前变量状态交换为kWaiter
4. 如果当前变量的状态是kWaiter，那么直接跳出当前循环，会在之后的futex锁里面持续等待
5. 在futex锁里等待

其源码如下：

```cpp
void Notification::WaitForNotification() {
  while (true) {
    NotifiedState v = notified_.load(std::memory_order_acquire);
    if (v == NotifiedState::kNotified) {
      return;
    } else if (v == NotifiedState::kNoWaiter) {
      // We only need relaxed here since we're always going to sync via
      // futex or re-acquire load above.
      if (!notified_.compare_exchange_weak(v, NotifiedState::kWaiter,
                                           std::memory_order_relaxed)) {
        continue;
      }
    } else {
      break;  // We'll wait below.
    }
  }
  Futex::Wait(&notified_, NotifiedState::kWaiter);
}
```

- Notify()：

在一个while(true)循环里执行以下逻辑

1. 原子地load出Notification的状态
2. 将当前变量的状态原子地交换设置为kNotified，然后break
3. 如果有正在等待通知地进程，那么就直接使用futex锁的wake方法，来将当前变量上正在等待的进程给唤醒

源码如下：

```cpp
void Notification::Notify() {
  NotifiedState v;

  while (true) {
    v = notified_.load(std::memory_order_acquire);

    CHECK(!HasBeenNotified());
    if (notified_.compare_exchange_weak(v, NotifiedState::kNotified,
                                        std::memory_order_release)) {
      break;
    }
  }

  if (v == NotifiedState::kWaiter) {
    Futex::Wake(&notified_, std::numeric_limits<int>::max());
  }
}
```

- Reset()：

将kNoWaiter状态存入notified_中，没有等待的进程即为初始状态。

```cpp
// Resets the notification back to the `unnotified` state.
  // Careful using this - you need to communicate to whoever will call Notify
  // *after* you reset, e.g. with another Notification. There is no way for the
  // notifier to 'notice' that this object is ready for another notification.
  void Reset() {
    notified_.store(NotifiedState::kNoWaiter, std::memory_order_relaxed);
  }
```

至此，Notification类的重要方法都介绍完了，其中几个重要的关键点是原子类，即std::atomic<T>和futex锁，前者可以原子地执行load和store操作来供其它变量来获取Notification的状态，后者使用futex系统调用以及一些原子操作来实现等待和唤醒机制，如果了解了这两个机制，应该可以进一步理解Notification类的原理，因此，本章节将结合源码对于这两个机制做详细介绍。在章节之初就已经强调过条件变量是ghOSt进程之间通信的最重要法宝，其地位可以说甚至比shareblob等共享内存还要更高，因为条件变量的实现为进程之间通信提供了一种即为具象化的方式，充分体现了抽象的魅力。

- 在本章节中出现的原子操作：
1. std::memory_order_relaxed：是C++11标准中提供的一种内存模型，它表示不保证任何原子操作之间的顺序、同步或可见性，因此也是内存模型中最轻量的一种。在多线程并发访问一个整型变量时，如果不需要关心变量的准确顺序，可以使用**`std::memory_order_relaxed`**来保证对该变量的原子操作的顺序，这样可以大大提高多线程的执行效率。可以在无需保证数据同步或顺序时，实现更高效的并发操作。
   
    可以看到，在reset方法的实现上使用了这种内存模型，因为对于notified_的写入，不需要保证原子性，只需要实时地更新为最新状态即可。
    
2. std::memory_order_aquied：**`std::memory_order_acquire`**是C++11中的一个枚举类型，它表示一个原子操作需要获取(acquire)的内存顺序(order)，也就是保证本次操作之前，之前所有的存储(memory)操作已经完成，且变化对其它处理器可见。具体来说，如果一个线程执行了一个使用**`std::memory_order_acquire`**的原子操作，那么它之前的读操作都不能被重排序到它之后，同时，这个操作也不能与之前的写操作重排序。这个顺序保证了本次操作所需的内存状态已经就绪，可以安全地进行后续操作。
   
    在Notify()和WaitForNotification()两个方法里，都使用了该内存模型，为的是在load的时候原子地获取notified_状态，在多线程的情况下仍然保证读取操作的有序性。
    
3. compare_exchange_weak：是一个 **`std::atomic`**类型的成员函数，用于实现原子性的比较和交换操作。它的作用是比较 **`std::atomic`**对象的当前值和期望值，如果相等，则将对象的值替换为新值并返回 true，否则不修改值并返回 false。函数执行的是一个原子操作，保证线程安全

```cpp
bool compare_exchange_weak(T& expected, T desired, std::memory_order order = std::memory_order_seq_cst) noexcept;
```

函数的执行过程是：如果当前值等于期望值，则将当前值修改为新值，并返回 true，否则不修改当前值，将当前值存入 **`expected`**中，返回 false

在std::atomic::compare_exchange_weak中，使用"weak"意味着compare_exchange操作失败时，它不会保证线程一定得到正确的值，因此线程需要根据compare_exchange返回的bool值来进行后续操作。如果返回false，线程应该再次尝试执行compare_exchange操作。相反，compare_exchange_strong保证在操作失败时返回正确的值。虽然compare_exchange_strong提供了比weak更强的保证，但它通常需要更多的硬件开销，并且在某些情况下可能会比weak更慢。因此，选择哪种方式取决于具体的场景和需求。

在Notify()和WaitNotification()，都使用了原子的交换操作并且配合循环来实现了对于notified_应有状态的赋值。

- futex锁的wait和wake操作：
1. wait()

```cpp
// Waits on the futex `f` while its value is `val`. Note that the type `T`
  // must be the same size as an int, which is the size that futex supports. We
  // use a template mainly to support enum class types.
  template <class T>
  static int Wait(std::atomic<T>* f, T val) {
    static_assert(sizeof(T) == sizeof(int));
    while (true) {
      int rc = futex(reinterpret_cast<std::atomic<int>*>(f), FUTEX_WAIT,
                     static_cast<int>(val), nullptr);
      if (rc == 0) {
        if (f->load(std::memory_order_acquire) != val) {
          return rc;
        }
        // This was a spurious wakeup, so sleep on the futex again.
      } else {
        if (errno == EAGAIN) {
          // Futex value mismatch.
          return 0;
        }
        CHECK_EQ(errno, EINTR);
      }
    }
  }
```

1. wake()：

```cpp
// Wakes up threads waiting on the futex. Up to `count` threads waiting on `f`
  // are woken up. Note that the type `T` must be the same size as an int, which
  // is the size that futex supports. We use a template mainly to support enum
  // class types.
  template <class T>
  static int Wake(std::atomic<T>* f, int count) {
    static_assert(sizeof(T) == sizeof(int));
    int rc = futex(reinterpret_cast<std::atomic<int>*>(f), FUTEX_WAKE, count,
                   nullptr);
    return rc;
  }
```

**`futex()`** 调用会导致调用线程进入休眠状态，直到 **`f`** 的值不等于 **`val`**。当条件满足时，线程将从 **`futex()`** 调用中返回，继续执行后续的代码。

举个例子，agent_ready_.WaitForNotification()系统调用会调用Futex::Wait(kWaiter)，此时该线程会一直沉睡直到某个线程调用了agent_ready_.Notify()来让agent_ready_类里的notified_变量状态变为kNotified，就可以唤醒之前wait的线程了。

##### 具体agent的建立

> 在所有的agent里，都存在着agent的具体构造这一步。这一步是在agent通信进程建立的子进程建立这个过程中的fork出来的子进程里进行的。自从fork之后，父进程就在条件变量里一直等待，而子进程需要完成后续的工作，然后通知父进程准备就绪，让父进程回答了main进程中成为和终端交互的利器，在用户态成功建立起agent并且调度任务。而具体agent的建立这一步就是agent通信进程子进程的fork返回出来之后的一个最重要的步骤。
> 

在这一章节中，主要是介绍一个agent的建立都需要经历哪些步骤，而这些步骤其实包含了很多具体的实现细节，这些具体的实现细节将会放到之后的章节中再作介绍，这一章节只需要知道具体agent建立的整体过程就好。

整个agent建立的最关键的一行代码：

```cpp
full_agent_ = std::make_unique<FullAgentType>(config);
```

在这一行代码中，重点需要关注的是FullAgentType这个泛型，这是一个作为可以容纳所有agent类型的基类。这一行代码使用了`make_unique`这个api，调用了FullAgentType(config)的构造函数并且返回了指向具体agent类的智能指针，config是之前提到过的在main进程准备工作里由用户传入的一些必要参数的封装。

仍然以fifo场景为例，来看看这个agent的构造函数的过程：

首先让我们来看看在main进程里面的创建用户态agent进程的代码，重点关注的是传入的泛型。

```cpp
// Using new so we can destruct the object before printing Done
  auto uap = new ghost::AgentProcess<ghost::FullFifoAgent<ghost::LocalEnclave>,
                                     ghost::AgentConfig>(config);
```

FullFifoAgent是为了fifo调度算法实现的特有的调度类，继承了基类FullAgent。其接受模板参数LocalEnclave，也是ghOSt中的一个基类，继承了类Enclave，其中包含了一系列关于enclave建立的api，以及agent和scheduler与它的联系。AgentConfig就是ghOSt当中对调度类所需要的参数的整体封装。不同的调度类对于参数也许有着不同的需求，ghOSt很好地把这些参数封装成了配置类Config并且在用户态main进程就实现了参数地传入。

在这行代码里，可以看到主要是调用了AgentProcess的构造函数，这个构造函数的前面几个步骤在之前就已经介绍：shareblob的建立、ForkedProcess。这个章节作为承上启下的一个章节，将会展示出agent进程即FullFifoAgent的构造函数的实现。

接下来看看FullFifoAgent接收config为参数的构造函数：

```cpp
explicit FullFifoAgent(AgentConfig config) : FullAgent<EnclaveType>(config) {
    scheduler_ =
        MultiThreadedFifoScheduler(&this->enclave_, *this->enclave_.cpus());
    this->StartAgentTasks();
    this->enclave_.Ready();
  }
```

其中包含了四个大步骤：

1. enclave的建立
2. 调度器的建立
3. 开启调度任务
4. 准备好enclave集群

对于这四个步骤，接下来的章节会一一仔细介绍其中的细节。需要注意的一点是FullFifoAgent类继承了FullAgent类，而在FullAgent类里其构造函数即是enclave建立的关键，不仅如此，FullAgent类里还包含着StartAgentTasks这样的开启任务的方法。FullAgent类又是继承了Agent类，其中的包含关系已经不言而喻。这样的层层包含使得ghOSt像之前说的那样抽象意义十分明显。

在这一章节里展示了最重要的三个代码段。三个代码段可谓是环环相扣，到了最终的具体FullAgent的建立时，又看到了Agent建立的四大步骤，这四大步骤虽然才是精华中的精华，但是ghOSt这种封装的巧妙值得学习。接下来的章节就来看看这四大步骤。

##### enclave的建立

```cpp
// Using new so we can destruct the object before printing Done
  auto uap = new ghost::AgentProcess<ghost::FullFifoAgent<ghost::LocalEnclave>,
                                     ghost::AgentConfig>(config);
```

再次来看到这一行代码，传入FullFifoAgent的enclave类型为LocalEnclave，而上一小节里，agent的建立的四大步骤的第一大步骤为FullAgent的构造函数，而FullAgent的构造函数里包含了**enclave(config)**的过程，即enclave以config为参数的构造函数。

来看看LocalEnclave的构造函数过程：

1. 将传入的config中所包含的cpulist，即cpu位图、拓扑参数和enclave_fd，即enclave文件目录传入，即初始值赋值。在ghOSt内核里，enclave是有着固定的文件目录的，因此每次建立agent时都不需要刻意传入enclave_fd，除非自己有需求，在AgentConfig这个类里enclave_fd的初始值也为-1。
2. 如果传入的enclave_fd为-1，那么就调用api CreateAndAttachToEnclave()来创建并且绑定enclave，否则调用api AttachToExistingEnclave()来直接绑定当前已经存在的集群。
3. 这里对于CreateAndAttachToEnclave()做详细说明：
   
    在ghOSt的内赫里，ghOSt的文件系统的顶层目录为：/sys/fs/ghost/，其文件结构如下：
    
    ```cpp
    $pwd
    $/sys/fs/ghost
    $tree
    |-enclave_1
    	|-ctl
    	|-cpu_data
    	|-cpumask
    ```
    
    这个函数做的第一件事情就是获取ctl_fd_核dir_fd_分别为文件ctl核enclave_1的路径的文件描述符。
    
    接着获取data_fd_即文件cpu_data的文件描述符。之后使用mmap系统调用在data_fd_建立共享内存，即lib中提到的StatusWordTable，用于内核与用户态之间的通信。
    
    最后获取cpumask_fd_，即cpumask文件的文件描述符，在该文件将用户态传入的cpulist作为字符串传入，用于标识enclave所管辖的cpu范围。
    
4. BuildCpuReps()：在第3步建立了共享内存区域data_region_之后，对于enclave所管辖的每个cpu，进行事务对象的建立与绑定，即每个cpu都单独与一个事务绑定起来，便于之后的调度决策提交。

到这里enclave就成功建立了，其建立过程主要是用户态与内核通信的共享区域以及cpu核事务txn的一一对应绑定过程。

##### 调度类的建立

调度类的建立主要包含两个步骤：分配器的建立和调度器的建立，后者依赖于前者的创建。源代码如下：

```cpp
auto allocator = std::make_shared<ThreadSafeMallocTaskAllocator<FifoTask>>();
auto scheduler = std::make_unique<FifoScheduler>(enclave, std::move(cpulist),
                                                   std::move(allocator));
```

对于分配器，ThreadSafeMallocTakAllocator继承了SingleThreadMallocTaskAllocator，其构造函数是默认构造函数，类内主要的方法为通过gtid(ghost thread id)获取task和freetask（释放task内存）以及foreachtask(func)遍历每个task使用func方法这些操作。ThreadSafeMallocTakAllocator可以进行适当的重写以满足特殊需求。因此allocator主要是对于task的一些封装操作。

对于调度器，FifoScheduler继承了BasicDispatchScheduler<FifoTask>，在类BasicDispatchScheduler里，对于调度器进行了enclave和cpulist的初始化，并且把刚刚创建的分配器赋值到变量`allocator_`里。并且，还是西安了对于task的一系列状态转移操作，如`TaskNew()`等。

调度器建立后，会被加入enclave所管理的调度器数组schedulers_中。

这里不对分配器和调度器做过多的api介绍了，lib里面已经介绍得比较详细了，只需要直到成功创建了和enclave对应得scheduler，并且这个scheduler可以在自定义得cpulist进行调度操作，即成功绑核就好。

##### 任务的开启

关于任务的开启，可以先看源代码：

```cpp
void StartAgentTasks() {
    // We split start into StartBegin and StartComplete to speed up
    // initialization.  We create all agent tasks, and they all migrate to their
    // cpus and wait until the old agent (if any) dies.
    for (const Cpu& cpu : *enclave_.cpus()) {
      agents_.push_back(MakeAgent(cpu));
      agents_.back()->StartBegin();
    }
    for (auto& agent : agents_) {
      agent->StartComplete();
    }
  }
```

这一步是任务的开启，又分为了两个步骤：每个cpu建立对应的agent，并且每个agent开始运作，然后是每个agent被标志为开始。

本章一开始就提到，per-cpu的精髓就是每个agent和一个cpu对应，而在这段源代码的第一段for循环里，可以看到装有所有agent的数组agents_都加入了以cpu为参数而创建的单独agent，之后新加入的agent调用了StartBegin()方法来开启运作。

- StartBegin()：

```cpp
void Agent::StartBegin() { thread_ = std::thread(&Agent::ThreadBody, this); }
```

startbgein做的事情就是开启另外一个线程，而这个线程会执行有关调度的任务。这就体现了在agent进程中子进程是负责调度的观点。

ThreadBody是一个统一的函数，它干的事情有：

1. 获取消息队列的fd
2. 获取当前enclave的statuswordtable
3. 将当前agent加入enclave的agents_，这同样也是装有agent的数组
4. 执行AgentThread()方法，这个方法是每个调度类都需要去重写的符合自身调度算法的，如fifo调度算法就需要体现出先进先出的调度特点。在这个方法里先进行了信号的唤醒—ready_.Notify()，然后等待集群准备好—WaitForEnclaveReady()。之后AgentThread()的流程会做详细介绍。
5. WaitForExitNotification()：等待子进程的结束通知

在任务的开启的源代码的第二段for循环里，对每个agent进行了标志完成的操作，其实就是ready_.WaitForNotification()的条件操作，于AgentThread()里的SignalReady()操作相互对应。

这就是整个任务开启的过程，在ghOSt里虽然每个调度类的调度算法才是关键所在，但是一直到现在调度算法才浮出水面，ghOSt在main进程到调度算法的过程实在是做了很多的铺垫，但是这些封装和抽象也对应了ghOSt的抽象意义非常的高。

##### enclave准备工作结束，开始调度

enclave的准备工作是agent进程的子进程实现的四大步骤的最后一步，主要是对于enclave掌管的调度器和agent进行了通知，标志着enclave当前已经准备就绪，所有正在等待enclave的进程都可以被唤醒了。

具体步骤：

1. 对enclave掌管的所有调度器进行循环，所有调度器会查看enclave目前是否存在着新的任务，如果有就纳入调度器的管理。
2. 对enclave掌管的所有调度器进行循环，再对当前调度器所掌管的所有cpu进行循环，获取每个cpu的agent，将每个agent的消息队列切换到应该与agent→barrier()相关联的状态（这里是第一次提到barrier，这是ghOSt实现用户态和内核之间通信的一个字段，目的是让agent传到内核的调度策略实现同步，再ghOSt论文中有提及，由于本章不是对ghOSt论文的解读，因此这里不作过多描述）
3. 对于enclave下掌管的所有cpu进行循环，再获取每个cpu对应的agent，每个agent都调用EnclaveReady()方法，标志Enclave已经准备就绪
4. 最后将标志变量提起标志着enclave准备工作结束

##### 幕后工作和Debug

> 在full_agent_建立起来之后，还有一些幕后工作和debug手段
> 
1. GhostSignals::IgnoreCommon()

源代码：

```cpp
std::signal(SIGINT, SigIgnore);
std::signal(SIGTERM, SigIgnore);
std::signal(SIGUSR1, SigIgnore);
```

在子进程里忽略了Ctrl-c，进程销毁和自定义的SIGUSR1信号。之前提到过，在ForkedProcess里面子进程进行了setppid操作之后已经脱离了父进程，因此会和父进程共同相应同一个终端的信号，为了让调度进程持续不断的进行，这里进行了对于终止信号的忽略操作。而对于自定义的SIGUSR1信号，这是父进程用于debug的手段吗，因此在子进程也直接忽略。

- 父进程的debug手段-rpc

ghOSt自定义了信号SIGUSR1，虽然令人遗憾的是这个信号目前还处于TODO状态，没有被明确定义，但是其debug手段确实值得一看。

在子进程里，定义了一个rpc_handler线程，其包含了对于rpc的响应和处理操作，而且仍然是使用Notification条件变量的wait和Notify操作去等待和唤醒。在父进程里自定义了对于SIGUSR1的响应函数，而这个响应函数其实是调用了在父进程中返回main进程的AgentProcess类变量uap中的Rpc函数。Rpc函数和rpc_handler_线程使用条件变量相交互，并且在父进程和子进程的共享内存shareblob里可以实现一些自定义的事情。由于ghOSt还未去做这些事情，这些事情只是被当作一个未来可以拓展的功能而已。

- 退出操作：

在rpc_handler_建立之后这个线程就被detatch出去了（分离）。由于父进程除非被Ctrl-c强行终止否则不会被终止，而子进程也和父进程分离了，在delete uap之前子进程也不会消亡，因此这个被分离出去的线程无需担心主进程突然暴毙带来的特殊情况。

在父进程如果接受了Ctrl-c的终止操作，那么会进行delete uap的操作，AgentProcess的析构函数也是自定义的。在子进程分离出rpc_handler_线程之后执行了下面的代码：

```
sb_->agent_ready_.Notify();
sb_->kill_agent_.WaitForNotification();
```

第一行代码，使得在条件变量里等待的父进程退出，回到main函数。回到main函数之后定义了一个Notification类型的变量exit，在Ctrl-c终止信号到来之前，一直进行着exit.Wait的操作。

第二行代码一直在等待，直到delete uap之后调用了AgentProcess的析构函数之后执行了kill_agent_.Notify()结束了子进程。

##### AgentThread()

介绍ghOSt的agent进程如果失去了AgentThread是没有灵活的，在**任务的开启**一节中，谈到了AgentThread()方法，它是每个调度类都需要去重写的一个能够体现自身调度类个性的方法。但是，其实里面执行的代码也是大部分相同的，实际上需要重写的是调度类里面的Schedule方法。

fifo调度类的schedule方法：

```cpp
void FifoScheduler::Schedule(const Cpu& cpu, const StatusWord& agent_sw) {
  BarrierToken agent_barrier = agent_sw.barrier();
  CpuState* cs = cpu_state(cpu);

  GHOST_DPRINT(3, stderr, "Schedule: agent_barrier[%d] = %d\n", cpu.id(),
               agent_barrier);

  Message msg;
  while (!(msg = Peek(cs->channel.get())).empty()) {
    DispatchMessage(msg);
    Consume(cs->channel.get(), msg);
  }

  FifoSchedule(cpu, agent_barrier, agent_sw.boosted_priority());
}
```

这段源码干了三件事情：

1. 获取了agent进程和内核通信的字段agent_barrier
2. 将消息队列里面的消息一一取出并且做分发
3. 使用特殊的调度算法—FifoSchedule(…)

这就是AgentThread里大致的内容，处于一个while循环中，然后一直调用schedule方法。

#### 继承关系

#### 整体框架

### Centralized

per-cpu场景和centralized场景最大的区别其实就是centralized场景设置了global_cpu_。所有的其余流程都一致，而由于global_cpu_的建立，与其对应的agent就变成了全局agent。这个agent将掌管所有task的调度，而其它被建立起来的task将不参与调度。对于代码体现，可以直接看fullagent的构造函数：

```cpp
explicit FullFifoAgent(FifoConfig config) : FullAgent<EnclaveType>(config) {
    global_scheduler_ = SingleThreadFifoScheduler(
        &this->enclave_, *this->enclave_.cpus(), config.global_cpu_.id(),
        config.preemption_time_slice_);
    this->StartAgentTasks();
    this->enclave_.Ready();
  }
```

可以看到，大体上和per-cpu场景其实是一模一样的，但是调度器换成了全局调度器，也有多线程换成了单线程。调度器内的默认消息队列也变成了global_channel_，还增加了SetGlobalCpu()和PickNextGlobalCpu()等选择全局cpu的方法。