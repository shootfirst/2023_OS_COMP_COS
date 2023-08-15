cfs调度算法的任务队列的数据结构为红黑树，值为每个待调度进程，进程按照虚拟运行时间进行排序，虚拟运行时间越小的进程被调度的优先级越高，即每次调度都从红黑树的最左节点开始选取进程。

- CfsRq：cfs调度场景下的运行队列
    - 代码实现：
        
        ```cpp
        class CfsRq {
         public:
          CfsRq() : cur_(nullptr), min_vruntime_(0), rq_size_(0) {}
          ~CfsRq() = default;
        
          /* cfs调度算法里最重要的函数.
          选择下一个要跑的cfs_task, 同时更新最小的虚拟运行时间.
        	TODO 如有必要将正在执行的任务重新入队
          */
          struct enqueued_task *pick_next_cfs_task();
        
          /*将一个cfs_task加入到cfs运行队列中*/
          void enqueue_cfs_task(struct enqueued_task *task);
        
          /*在cfs运行队列里删除指定的任务*/
          bool dequeue_cfs_task(struct enqueued_task *task);
        
          /*初始化任务的nice值, 权值, 权值的倒数，和任务的
           *状态.*/
          void task_new(struct enqueued_task *task, uint64_t nice);
        
          /* 更新运行队列里的最小运行时间, 保证每个被选择调度的任务可以相对公
           * 平地接受调度决策*/
          void update_min_vruntime();
        
          /*检查运行队列是否为空*/
          bool is_empty() { return rq_size_ == 0; }
        
          /*返回cfs运行队列里最左边的任务*/
          struct enqueued_task *left_most_rq_task() {
            return is_empty() ? nullptr : *cfs_rq_.begin();
          }
        
          struct enqueued_task *cur_;
        
         private:
          std::set<struct enqueued_task *, Compare> cfs_rq_;
          static const int kmin_nice_ = -20;
          static const int kmax_nice_ = 19;
          Duration min_vruntime_;
          uint32_t rq_size_;
        };
        ```
        
    - nice值、weight值和inverse_weight值三者之间的关系以及计算公式：
        
        nice值在内核里被划分为了40个档次，分别是-20到19依次递增1。依据nice值，内核定义了权值到nice值的映射，用于表示每个进程的权重，代码定义如下：
        
        ```cpp
        static constexpr uint32_t kNiceToWeight[40] = {
            88761, 71755, 56483, 46273, 36291,  // -20 .. -16
            29154, 23254, 18705, 14949, 11916,  // -15 .. -11
            9548,  7620,  6100,  4904,  3906,   // -10 .. -6
            3121,  2501,  1991,  1586,  1277,   // -5 .. -1
            1024,  820,   655,   526,   423,    // 0 .. 4
            335,   272,   215,   172,   137,    // 5 .. 9
            110,   87,    70,    56,    45,     // 10 .. 14
            36,    29,    23,    18,    15      // 15 .. 19
        };
        ```
        
        这个权值的定义是以0这个nice值为中心进行扩展的，nice值0对应权重1024，即2^10。
        
        cfs的虚拟运行时间的计算公式如下：
        
        $vruntime = physical_runtime * inverse\_weight >> 22$
        
        inverse_weight是根据2^32 / weight计算出来的，inverse_weight的定义如下：
        
        ```cpp
        static constexpr uint32_t kNiceToInverseWeight[40] = {
            48388,     59856,     76040,     92818,     118348,    // -20 .. -16
            147320,    184698,    229616,    287308,    360437,    // -15 .. -11
            449829,    563644,    704093,    875809,    1099582,   // -10 .. -6
            1376151,   1717300,   2157191,   2708050,   3363326,   // -5 .. -1
            4194304,   5237765,   6557202,   8165337,   10153587,  // 0 .. 4
            12820798,  15790321,  19976592,  24970740,  31350126,  // 5 .. 9
            39045157,  49367440,  61356676,  76695844,  95443717,  // 10 .. 14
            119304647, 148102320, 186737708, 238609294, 286331153  // 15 .. 19
        };
        ```
        
    - 重要api：`pick_next_cfs_task()`
        
        ```cpp
        struct enqueued_task *CfsRq::pick_next_cfs_task() {
          if (is_empty()) {
        		// 更新运行队列的最小虚拟运行时间
            update_min_vruntime();
            return nullptr;
          }
        
        	// 从红黑树的最左边选取任务
          struct enqueued_task *next = left_most_rq_task();
          if (!dequeue_cfs_task(next)) {
        		// 删除选出的任务
            printf("error : delete cfs task\n");
          }
        	// 设置任务状态
          next->set_state(enqueued_task::State::kRunning);
        	// 再次最小虚拟运行时间
          update_min_vruntime();
          return next;
        }
        ```
        
- cfs调度算法的实现：
    1. 在vruntime_enqueue函数里，在任务即将加入运行队列时先调用`task_new`函数来设置任务的nice值和权值的倒数。然后调用`update_enqueued`函数来更新即将加入运行队列的任务的虚拟运行时间，最后调用红黑树的插入元素api来讲任务加入运行队列。
    2. 在`dispatch_batch`函数中，通过`pick_next_cfs_task()`来获取下一个需要运行的任务，选出来的任务自然就被送回内核去执行了。