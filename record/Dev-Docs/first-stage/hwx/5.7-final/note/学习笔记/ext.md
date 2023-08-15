dsq：

```c
/*
 * DSQ (dispatch queue) IDs are 64bit of the format:
 *
 *   Bits: [63] [62 ..  0]
 *         [ B] [   ID   ]
 *
 *    B: 1 for IDs for built-in DSQs, 0 for ops-created user DSQs
 *    B为1代表built-in的dsq，为0代表user创建的dsq
 *   ID: 63 bit ID
 *
 * Built-in IDs:
 *
 *   Bits: [63] [62] [61..32] [31 ..  0]
 *         [ 1] [ L] [   R  ] [    V   ]
 *
 *    1: 1 for built-in DSQs.
 *    L: 1 for LOCAL_ON DSQ IDs, 0 for others
 *    V: For LOCAL_ON DSQ IDs, a CPU number.
 		For others, a pre-defined value.
 */
// 也就是说，dsq可以是built-in的，也可以是user defined的。前者又分为local和global
// 对于built-in的dsq，local的前32位是CPUID！！！！！
// 而global的是预定义的value
// 六
enum scx_dsq_id_flags {
	SCX_DSQ_FLAG_BUILTIN	= 1LLU << 63, // B
	SCX_DSQ_FLAG_LOCAL_ON	= 1LLU << 62, // L

	SCX_DSQ_INVALID		= SCX_DSQ_FLAG_BUILTIN | 0, 
	SCX_DSQ_GLOBAL		= SCX_DSQ_FLAG_BUILTIN | 1,
    // 啥玩意？意思是说Local-on和local不是一个东西？
	SCX_DSQ_LOCAL		= SCX_DSQ_FLAG_BUILTIN | 2,
	SCX_DSQ_LOCAL_ON	= SCX_DSQ_FLAG_BUILTIN | SCX_DSQ_FLAG_LOCAL_ON,
	SCX_DSQ_LOCAL_CPU_MASK	= 0xffffffffLLU,// 前32位为cpuid
};
```



还有一个很重要的概念：dsq buffer slots。

这个函数用于返回当前剩余的slots：

```c
/**
 * scx_bpf_dispatch_nr_slots - Return the number of remaining dispatch slots
 *
 * Can only be called from ops.dispatch().
 */
u32 scx_bpf_dispatch_nr_slots(void)
{
	if (!scx_kf_allowed(SCX_KF_DISPATCH))
		return 0;

	return scx_dsp_max_batch - __this_cpu_read(scx_dsp_ctx.buf_cursor);
}
```

而用于将dsq中任务接到local队列以执行的consume可以重置这个slots计数：

```c
/**
 * scx_bpf_consume - Transfer a task from a DSQ to the current CPU's local DSQ
 * @dsq_id: DSQ to consume
 *
 * Consume a task from the non-local DSQ identified by @dsq_id and transfer it
 * to the current CPU's local DSQ for execution. Can only be called from
 * ops.dispatch().
 *
 * This function flushes the in-flight dispatches from scx_bpf_dispatch() before
 * trying to consume the specified DSQ. It may also grab rq locks and thus can't
 * be called under any BPF locks.
 *
 * Returns %true if a task has been consumed, %false if there isn't any task to
 * consume.
 */
bool scx_bpf_consume(u64 dsq_id)
{
	struct scx_dsp_ctx *dspc = this_cpu_ptr(&scx_dsp_ctx);
	struct scx_dispatch_q *dsq;

	if (!scx_kf_allowed(SCX_KF_DISPATCH))
		return false;

	flush_dispatch_buf(dspc->rq, dspc->rf);

	dsq = find_non_local_dsq(dsq_id);
	if (unlikely(!dsq)) {
		scx_ops_error("invalid DSQ ID 0x%016llx", dsq_id);
		return false;
	}

	if (consume_dispatch_q(dspc->rq, dspc->rf, dsq)) {
		/*
		 * A successfully consumed task can be dequeued before it starts
		 * running while the CPU is trying to migrate other dispatched
		 * tasks. Bump nr_tasks to tell balance_scx() to retry on empty
		 * local DSQ.
		 */
		dspc->nr_tasks++;
		return true;
	} else {
		return false;
	}
}
```

每次耗尽slots就需要重置计数：

```c
central
		/*
		 * Retry if we ran out of dispatch buffer slots as we might have
		 * skipped some CPUs and also need to dispatch for self. The ext
		 * core automatically retries if the local dsq is empty but we
		 * can't rely on that as we're dispatching for other CPUs too.
		 * Kick self explicitly to retry.
		 */
		if (!scx_bpf_dispatch_nr_slots()) {
            // 如果slots耗尽
			__sync_fetch_and_add(&nr_retries, 1);// 增加retry次数
			scx_bpf_kick_cpu(central_cpu, SCX_KICK_PREEMPT);// 呼吁当前CPU赶紧干活
			return;
		}

		/* look for a task to run on the central CPU */
		if (scx_bpf_consume(FALLBACK_DSQ_ID))
			return;
		dispatch_to_cpu(central_cpu);
```





整理下这些API接口：

##### select_cpu

```c
/*
prev_cpu：p在沉睡以前所处在的cpu
为正在被唤醒的task选择一个新cpu
	 * 【这里应该是在内核中实现的。也就是说，内核会根据我们返回的CPU id来做事】
	 * If an idle CPU is returned, the CPU is kicked and will try to
	 * dispatch. 如果被分配到了一个空闲cpu，那么该cpu就会被踢醒然后try dispatch
	 * While an explicit custom mechanism can be added,
	 * select_cpu() serves as the default way to wake up idle CPUs.
	 *  select_cpu() 是唤醒空闲 CPU 的默认方式
*/
// 一般在里面会调用
s32 (*select_cpu)(struct task_struct *p, s32 prev_cpu, u64 wake_flags);
```

它里面实现可以摆烂，或者找一下空闲的：

```c
s32 BPF_STRUCT_OPS(central_select_cpu, struct task_struct *p,
		   s32 prev_cpu, u64 wake_flags)
{
    // 直接摆烂return central
	return central_cpu;
}
```

```c
s32 BPF_STRUCT_OPS(userland_select_cpu, struct task_struct *p,
		   s32 prev_cpu, u64 wake_flags)
{
	if (keep_in_kernel(p)) {
		s32 cpu;
		struct task_ctx *tctx;

		tctx = bpf_task_storage_get(&task_ctx_stor, p, 0, 0);
		if (!tctx) {
			scx_bpf_error("Failed to look up task-local storage for %s", p->comm);
			return -ESRCH;
		}

		if (p->nr_cpus_allowed == 1 ||
		    scx_bpf_test_and_clear_cpu_idle(prev_cpu)) {
			tctx->force_local = true;
			return prev_cpu;
		}
		// 找空闲的，找到了就返回，没找到就随便返回一个perv的
		cpu = scx_bpf_pick_idle_cpu(p->cpus_ptr);
		if (cpu >= 0) {
			tctx->force_local = true;
			return cpu;
		}
	}

	return prev_cpu;
}
```

##### enqueue

```c
	/**
	 * 将task enqueue到bpf scheduler
	 * 确实，比如说在central实现中，就执行了bpf_map_push_elem(&central_q, &pid, 0)将task入队
	 *
	 * @p is ready to run. Dispatch directly by calling scx_bpf_dispatch()
	 * or enqueue on the BPF scheduler. 
	 * If not directly dispatched, the bpf
	 * scheduler owns @p and if it fails to dispatch @p, the task will
	 * stall.
	 */
	void (*enqueue)(struct task_struct *p, u64 enq_flags);
```

这个是因策略而异的

##### dispatch

```c
/**
 * dispatch - Dispatch tasks from the BPF scheduler and/or consume DSQs
 
 * 
 * @cpu: CPU to dispatch tasks for
 * @prev: previous task being switched out
 *
 * Called when a CPU's local dsq is empty. The operation should dispatch
 * one or more tasks from the BPF scheduler into the DSQs using
 * scx_bpf_dispatch() and/or consume user DSQs into the local DSQ using
 * scx_bpf_consume().
 *
 * The maximum number of times scx_bpf_dispatch() can be called without
 * an intervening scx_bpf_consume() is specified by
 * ops.dispatch_max_batch. See the comments on top of the two functions
 * for more details.
 *
 * When not %NULL, @prev is an SCX task with its slice depleted. If
 * @prev is still runnable as indicated by set %SCX_TASK_QUEUED in
 * @prev->scx.flags, it is not enqueued yet and will be enqueued after
 * ops.dispatch() returns. To keep executing @prev, return without
 * dispatching or consuming any tasks. Also see %SCX_OPS_ENQ_LAST.
 */
void (*dispatch)(s32 cpu, struct task_struct *prev);
```









userland scheduler和测试程序之间建立共享内存。

1. 共享内存

   map\<pid，ddl\>

2. 测试程序

   由两部分组成。

   1. main线程

      创建2，创建线程但不让他跑，并根据其pid向共享内存写入ddl，然后通过pid设置线程的调度类为EXT，然后运行线程

      【但是发现好像，创建线程了不能先让它不跑，它马上就会开跑。所以说，目前解决思路有两个，一个用提供了该功能的boost库（详情见下面的chatgpt），另一个是自己在这里以及下面userland scheduler获取相关信息之前使用条件变量，且条件变量也存储在共享内存中】

   2. 其他一坨需要被userland scheduler调度的线程

3. userland scheduler

   从enqueued中取出task, 从共享内存中获取相关信息，填入自己的scx_userland_enqueued_task结构体中。然后把scx_userland_enqueued_task task放进什么红黑树里面排序。











那我可不可以在c++中像java那样，创建一个线程但不立刻执行，直到显示调用start方法才开始执行？

ChatGPT



在C++标准库中，并没有提供像Java中的Thread类中的start方法那样的功能，也就是说，当你使用C++标准库创建一个线程时，它会立即开始执行。

但是，你可以使用一些技巧来达到类似于Java中的start方法的效果，即创建一个线程但不立即开始执行。比如可以在线程函数中使用条件变量等同步机制来实现等待一个信号，当需要开始执行线程时，发送这个信号，线程就会开始执行。这种方式需要自己手动实现线程的调度和同步，需要一定的编程技巧和经验。

此外，也可以使用第三方库或框架来实现这个功能。例如，Boost库提供了类似于Java中的线程控制类，可以控制线程的启动和停止，以及线程的优先级等。C++11标准引入了std::thread类，也可以使用它来创建线程，并使用std::condition_variable等同步机制来实现类似于Java中的start方法的效果。



Boost库是什么东西



ChatGPT

2 / 2

Boost库是一个流行的、开源的C++程序库集合，提供了许多高质量、可重用的组件和工具，用于开发C++应用程序。Boost库最初由David Abrahams和其他C++社区成员开发，目的是为了扩展C++标准库，并为C++开发人员提供一些现代的、高效的C++库。

Boost库包含了许多不同的组件，例如：智能指针、字符串算法、线程、多线程、正则表达式、文件系统、数学算法等等。这些组件都经过广泛的测试和优化，可以帮助开发人员提高编码效率和程序性能。

Boost库是跨平台的，可以在多个操作系统和编译器上使用，包括Windows、Linux、Mac OS X等，并且被广泛应用于各种领域，如游戏开发、科学计算、图形图像处理等。

