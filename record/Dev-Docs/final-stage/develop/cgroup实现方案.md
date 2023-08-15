## 初步方案

### 数据结构

```c
struct cos_cgroup {
    int rate; // CPU占用比率
    int level; // 剩余运行时间
    struct task_struct *s_list;
}
```

在cos.c中做一个**以name为key的cgroup哈希表**

### 系统调用

在cos.c文件中实现

1. 创建

   ```c
   int cos_create_cgroup(char* name)
   ```

2. 添加/删除

   ```c
   int cos_ctl_task_cgroup(char* name, int pid, bool add)
   ```

3. 修改CPU利用率

   ```c
   int cos_ctl_rate_cgroup(char* name, int level)
   ```

4. 销毁

   ```c
   int cos_destroy_cgroup(char* name)
   ```

### 定时器

在struct task_struct的**sched_cos_entity**中增加一个对task所属的cgroup的引用。

注册定时器，周期为1ms，我们在定时器函数中应该轮询所有的cgroup，然后根据rate更新cgroup的level。

每次对应task从cpu上下来的时候，就获取task的cgroup然后修改其level。

> 注：可以参考下原生cgroup的更新时机。[这篇文章](https://zhuanlan.zhihu.com/p/620713467)写得不错
>
> ```c
> pick_next_task_fair
>     |->put_prev_entity
>         |->update_curr
>             |->cgroup_account_cputime
> ```

pick的时候也获取下其cgroup对应cpu时间，如果已经用完了那就跳过该任务。





在内核编程中使用内核定时器，你可以按照以下步骤进行：

1. 包含头文件：首先，在你的内核模块中包含必要的头文件，其中包括 `<linux/module.h>` 用于模块编程，`<linux/init.h>` 用于初始化函数，`<linux/timer.h>` 用于内核定时器等。
2. 定义定时器：定义一个 `struct timer_list` 类型的定时器变量。

```c
#include <linux/module.h>
#include <linux/init.h>
#include <linux/timer.h>

static struct timer_list my_timer;
```

1. 编写定时器回调函数：创建一个函数，它将作为定时器到期时执行的回调函数。

```c
void my_timer_callback(unsigned long data)
{
    // 在这里编写定时器到期时要执行的任务代码
}
```

1. 初始化定时器：在模块的初始化函数中初始化定时器，并设置回调函数和到期时间。

```c
static int __init my_module_init(void)
{
    // 初始化定时器
    setup_timer(&my_timer, my_timer_callback, 0);
    
    // 设置定时器到期时间，这里设置为 1 秒后执行
    mod_timer(&my_timer, jiffies + msecs_to_jiffies(1000));
    
    return 0;
}
```

1. 注销定时器：在模块的退出函数中注销定时器，确保在模块卸载时定时器不再运行。

```c
static void __exit my_module_exit(void)
{
    // 注销定时器
    del_timer(&my_timer);
}

module_init(my_module_init);
module_exit(my_module_exit);
MODULE_LICENSE("GPL");
```

注意事项：

- 使用`msecs_to_jiffies()`函数将毫秒转换为`jiffies`，以便设置定时器的到期时间。
- 如果在定时器回调函数中需要对内核数据结构进行操作，务必遵循内核编程规范和安全性要求。
- 编译模块并加载到内核中。可以使用`insmod`命令加载模块，使用`rmmod`命令卸载模块。

以上是一个简单的示例，展示了如何在内核编程中使用定时器。在实际应用中，你可能需要更复杂的定时器任务和更复杂的内核模块逻辑。确保在内核编程中遵循最佳实践和安全性原则，以确保系统的稳定性和安全性。

## 最终方案

### 核心数据结构

```c++
struct cos_cgroup {

  double rate; // cgroup在一个周期得到的运行时间，80%为所有核心上一共0.8ms，最大值为n * 100%，n为核心数

  int level_time;  // cgroup在当前周期（1ms内）剩余运行时间，每个周期开始由定时器补满为 rate * 周期；当前cgroup线程运行结束或触发时间中断时，减去本次运行时间，若level_time小于0，则当前cgroup中线程不能运行，直到下一次周期开始由定时器补满

  struct task_struct structs;  // 该cgroup上的所有线程

  struct spin_lock lock; // 保护上面两个字段

}
```

### 系统调用

```
cos_create_cgroup 
```

作用：创建一个cgroup，返回其描述符

底层：在内核创建一个cos_cgroup结构体

```
cos_ctl_cgroup
```

作用：往指定cgroup中添加，删除和修改线程

底层：往对应cos_cgroup结构体中加入、删除和修改线程

```
cos_rate_cgroup
```

作用：调整设置对应cgroup的时间分配

底层：修改对应cos_cgroup结构体的rate

```
cos_delete_cgroup
```

作用：删除对应的cgroup

底层：删除对应的cos_cgroup结构体

### 流程

1.首先启动一个内核定时器，使用高精度定时器hrtimer，每一个周期（1ms）触发一次

2.定时器回调函数为遍历所有cos_cgroup结构体，将其level_time补满

3.每个cos线程运行结束，或者触发时间中断，检测自己是否在cgroup中，若是，则将cgroup中的level_time减去本次运行时间，若小于0，则不能运行

4.在pick_next_task和shoot中，若被选中的线程在cgroup中且 其level_time小于0，则不能得到运行，pick_next_task返回NULL，shoot则提示失败
