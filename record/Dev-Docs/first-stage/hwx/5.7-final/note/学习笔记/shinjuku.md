```c++
/*
我暂时忽略的点：
2. work class
我们必须在client端创建线程的时候，一定得要填写其flags为runnable状态

相比于原shinjuku的改动点：
1. 它是每次schedule前都copy一份priotable，我是直接实时读取
2. shinjuku最主要的有两项任务，一个是根据共享内存更新线程状态，另一个就是其具体的调度算法状态机。
还有一点就是时间操作，这个也很难实现！
前者有一个困难点，就是shinjuku的话，要求根据共享内存信息更新线程状态要做到实时、敏锐。
也就是说，你最好必须共享内存一变化，你就马上感知到，然后更新。
ghost的做法是轮询。
所以在我们这里应该改变做法，不能在每次task enqueue时才更新task的共享内存，而是应该在sched_main_loop
中的drain之前，再加一个函数update
【不过这样我仍然觉得有点问题，因为看bpf代码，这个agent只会在有task enqueue的时候被kick醒
然后执行centrallized的逻辑，这样其实还是不是很及时。
因而，我觉得应该需要修改bpf程序，使得这个agent就专注于agent工作，set下affinity】
3.  * 目前实现的和真shinjuku差距在哪?
 * 我目前只实现了，让该被抢占的cpu reschedule，但是task依然是进入全局队列，会被分配给哪个
 * cpu还是不确定的。这一点不知道有没有关系
*/

					// TODO: 不精确。此处实际为上一次enqueue之前的elapsed_runtime + 上一次被userland dispatch至今的time
					// 而我们要计算的是上一次真正被调度到cpu至今的time
					// 为了实现更精确的计时，也许需要在bpf程序中再开一个map，存放所有task
```





shinjuku最主要的几点：

1. 快速抢占

   1. dune实现

   2. context-switching优化

   3. 抢占时：

      ```c
      这大概也是ghOSt的实现手段。它这个由于每个worker是定死的，所以抢占时，是把request放回请求池，然后让worker执行任务；
      而在ghOSt中，是把worker连带请求一起放回线程池，然后切换为别的worker执行任务。
      
      也就是说，此处的worker相当于ghOSt的CPU。66666
      ```

2. **多优先级** **双端** 调度队列

   1. 每次选择优先级最高的队列的头调度

   2. 每次可以被放入队列头或者队列尾

      > 我们使用的经验法则是，对于多模态或重尾工作负载，请求应该放在队列的尾部，而对于轻尾工作，请求应该放在头部。

   3. 66666，点出了priotable。到时候文档可以着重描绘一下这个单地址操作系统到共享内存的转化







```c++
/* 
看了一遍，对实现per-cpu模型有点思路了。
1. .bpf.c
首先，扩展enqueued和dispatch队列，有多少个cpu就有多少个队列。
然后，在select cpu时获取task的可运行cpu，并且返回。
然后，在enqueue时根据task的可运行cpu（可以存储在task_ctx字段）来分发到特定的enqueued队列
最后在dispatch的时候选择SCX_DSQ_LOCAL即可。
2. .c
首先，bootstrap还是bootstrap，就是enqueued_fd和dispatch_fd从一个变量变成了vector
然后，在main中，需要创建多个线程，都执行sched_main_loop，并且都需要set下affinity
在sched_main_loop中根据自己的enqueued_fd来drain和dispatch就行了。
3. client
需要为每个thread设置affinity。
4. 测试
只需测试线程是否确实在对应cpu运行就行。
 */
#include <string.h>
#include "scx_common.bpf.h"
#include "scx_fifo_percpu_common.h"

char _license[] SEC("license") = "GPL";

const volatile s32 usersched_pid;

/* !0 for veristat, set during init */
// 相当于一个cpu mask？64 = 1000000， 也即使用0-5号共六个cpu
const volatile u32 num_possible_cpus = 64;

/* Stats that are printed by user space. */
u64 nr_failed_enqueues, nr_kernel_enqueues, nr_user_enqueues;

struct user_exit_info uei;

/*
 * Whether the user space scheduler needs to be scheduled due to a task being
 * enqueued in user space.
 */
static bool usersched_needed;

/*
 * The map containing tasks that are enqueued in user space from the kernel.
 *
 * This map is drained by the user space scheduler.
 */
struct {
	__uint(type, BPF_MAP_TYPE_QUEUE);
	__uint(max_entries, USERLAND_MAX_TASKS);
	__type(value, struct scx_userland_enqueued_task);
} enqueued SEC(".maps");

/*
 * The map containing tasks that are dispatched to the kernel from user space.
 *
 * Drained by the kernel in userland_dispatch().
 */
struct {
	__uint(type, BPF_MAP_TYPE_QUEUE);
	__uint(max_entries, USERLAND_MAX_TASKS);
	__type(value, s32);
} dispatched SEC(".maps");

/* Per-task scheduling context */
struct task_ctx {
	bool force_local; /* Dispatch directly to local DSQ */
};

// 这玩意就是存放上面那个task_ctx的map。大胆猜测key可能是pid
/* Map that contains task-local storage. */
struct {
	__uint(type, BPF_MAP_TYPE_TASK_STORAGE);
	__uint(map_flags, BPF_F_NO_PREALLOC);
	__type(key, int);
	__type(value, struct task_ctx);
} task_ctx_stor SEC(".maps");

static bool is_usersched_task(const struct task_struct *p)
{
	return p->pid == usersched_pid;
}

static bool keep_in_kernel(const struct task_struct *p)
{
	/*
	如果进程 p 允许在的 CPU 核心数量小于系统中可用的 CPU 核心数量，则返回 true，
	？？？？？
	*/
	return p->nr_cpus_allowed < num_possible_cpus;
}

static struct task_struct *usersched_task(void)
{
	struct task_struct *p;

	p = bpf_task_from_pid(usersched_pid);
	/*
	 * Should never happen -- the usersched task should always be managed
	 * by sched_ext.
	 */
	if (!p)
		scx_bpf_error("Failed to find usersched task %d", usersched_pid);

	return p;
}

// 看代码的一个点，就是要知道目前的sjf和mfq和cfs都是centrallized模型。
// 它们的centrallized体现在哪里？是只体现在userland程序？还是也体现在了bpf程序？
s32 BPF_STRUCT_OPS(userland_select_cpu, struct task_struct *p,
		   s32 prev_cpu, u64 wake_flags)
{
	// 初步猜测，每次被唤醒首先调用这个select，再调用enqueue
	if (keep_in_kernel(p)) {
		s32 cpu;
		// bool force_local; /* Dispatch directly to local DSQ */
		struct task_ctx *tctx; 

		// void *bpf_task_storage_get(void *map, struct task_struct *task, u64 flags);
		// 作用是从指定的 BPF 映射中获取与当前任务相关联的数据
		tctx = bpf_task_storage_get(&task_ctx_stor, p, 0, 0);
		if (!tctx) {
			scx_bpf_error("Failed to look up task-local storage for %s", p->comm);
			return -ESRCH;
		}

		// 如果该task只能在prev cpu上运行，或者prev cpu有空
		if (p->nr_cpus_allowed == 1 ||
		    scx_bpf_test_and_clear_cpu_idle(prev_cpu)) {
			tctx->force_local = true;
			return prev_cpu;
		}

		// 选择并占用一个空闲的 CPU
		cpu = scx_bpf_pick_idle_cpu(p->cpus_ptr);
		if (cpu >= 0) {
			tctx->force_local = true;
			return cpu;
		}
	}

	return prev_cpu;
}

static void dispatch_user_scheduler(void)
{
	struct task_struct *p;

	usersched_needed = false;
	p = usersched_task();
	if (p) {
		scx_bpf_dispatch(p, SCX_DSQ_GLOBAL, SCX_SLICE_DFL, 0);
		bpf_task_release(p);
	}
}

static void enqueue_task_in_user_space(struct task_struct *p, u64 enq_flags)
{
	
	struct scx_userland_enqueued_task task;
	memset(&task, 0, sizeof(task));
	task.pid = p->pid;
	task.sum_exec_runtime = p->se.sum_exec_runtime;
	task.tgid = p->tgid;
	
	if (bpf_map_push_elem(&enqueued, &task, 0)) {
		/*
		 * If we fail to enqueue the task in user space, put it
		 * directly on the global DSQ.
		 */
		__sync_fetch_and_add(&nr_failed_enqueues, 1);
		scx_bpf_dispatch(p, SCX_DSQ_GLOBAL, SCX_SLICE_DFL, enq_flags);
	} else {
		__sync_fetch_and_add(&nr_user_enqueues, 1);
		usersched_needed = true;
	}
}

void BPF_STRUCT_OPS(userland_enqueue, struct task_struct *p, u64 enq_flags)
{
	// 我觉得我懂了，SCX_DSQ_LOCAL表示一定要运行在select cpu时选中的那个cpu上
	if (keep_in_kernel(p)) {
		u64 dsq_id = SCX_DSQ_GLOBAL;
		struct task_ctx *tctx;

		/*
		// Per-task scheduling context 
		struct task_ctx {
			bool force_local; /* Dispatch directly to local DSQ 
		};
		*/
		tctx = bpf_task_storage_get(&task_ctx_stor, p, 0, 0);
		if (!tctx) {
			scx_bpf_error("Failed to lookup task ctx for %s", p->comm);
			return;
		}

		if (tctx->force_local)
			dsq_id = SCX_DSQ_LOCAL;
		tctx->force_local = false;
		// 20ms抢占时间
		scx_bpf_dispatch(p, dsq_id, SCX_SLICE_DFL, enq_flags);
		__sync_fetch_and_add(&nr_kernel_enqueues, 1);
		return;
	} else if (!is_usersched_task(p)) {
		enqueue_task_in_user_space(p, enq_flags);
	}
}
/*
 * Called when a CPU's local dsq is empty. The operation should dispatch
 * one or more tasks from the BPF scheduler into the DSQs using
 * scx_bpf_dispatch() and/or consume user DSQs into the local DSQ using
 * scx_bpf_consume().
*/
void BPF_STRUCT_OPS(userland_dispatch, s32 cpu, struct task_struct *prev)
{
	/*
	* Whether the user space scheduler needs to be scheduled due to a task being
	* enqueued in user space.
	*/
	if (usersched_needed)
		dispatch_user_scheduler();

	bpf_repeat(4096) {
		s32 pid;
		struct task_struct *p;

		if (bpf_map_pop_elem(&dispatched, &pid))
			break;

		/*
		 * The task could have exited by the time we get around to
		 * dispatching it. Treat this as a normal occurrence, and simply
		 * move onto the next iteration.
		 */
		p = bpf_task_from_pid(pid);
		if (!p)
			continue;

		scx_bpf_dispatch(p, SCX_DSQ_GLOBAL, 200000ULL, 0);
		bpf_task_release(p);
	}
}

s32 BPF_STRUCT_OPS(userland_prep_enable, struct task_struct *p,
		   struct scx_enable_args *args)
{
	if (bpf_task_storage_get(&task_ctx_stor, p, 0,
				 BPF_LOCAL_STORAGE_GET_F_CREATE))
		return 0;
	else
		return -ENOMEM;
}

s32 BPF_STRUCT_OPS(userland_init)
{
	if (num_possible_cpus == 0) {
		scx_bpf_error("User scheduler # CPUs uninitialized (%d)",
			      num_possible_cpus);
		return -EINVAL;
	}

	if (usersched_pid <= 0) {
		scx_bpf_error("User scheduler pid uninitialized (%d)",
			      usersched_pid);
		return -EINVAL;
	}
	return 0;
}

void BPF_STRUCT_OPS(userland_exit, struct scx_exit_info *ei)
{
	uei_record(&uei, ei);
}

SEC(".struct_ops")
struct sched_ext_ops userland_ops = {
	.select_cpu		= (void *)userland_select_cpu,
	.enqueue		= (void *)userland_enqueue,
	.dispatch		= (void *)userland_dispatch,
	.prep_enable		= (void *)userland_prep_enable,
	.init			= (void *)userland_init,
	.exit			= (void *)userland_exit,
	.timeout_ms		= 3000,
	.name			= "userland",
};
```

