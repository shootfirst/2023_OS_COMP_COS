# task delegation时延测试

## COS

首先确保在COS内核中：

```bash
uname -r # 6.4.0+
```

打开两个终端，在其中一个运行FIFO Scheduler：

```bash
pwd # 确保在cos_userspace目录下
sudo build/fifo_scheduler 2> agent_time
```

另一个执行python脚本：

```bash
pwd # 确保在cos_userspace目录下
sudo python3 task_delegation_latency.py
```

等待测试完成即可。最终原始数据保留在项目根目录的`task_delegation_latency_result`文件中，平均值、最大值以及最小值在python脚本对应终端中打印输出。



## EXT

首先确保在EXT内核中：

```bash
uname -r # 6.4.0-rc3+
```

打开两个终端，在其中一个运行Shinjuku Scheduler：

```bash
pwd # 确保在ext_kernel/tools/sched_ext/目录下
sudo sched/scx_shinjuku_bpf -p 2> agent_time
```

另一个执行python脚本：

```bash
pwd # 确保在ext_kernel/tools/sched_ext/目录下
sudo python3 latency_test.py
```

等待测试完成即可。最终原始数据保留在项目根目录的`latency_result`文件中，平均值、最大值以及最小值在python脚本对应终端中打印输出。



## ghOSt

首先确保在ghOSt内核中：

```bash
uname -r # 5.11.0+
```

打开两个终端，在其中一个运行Shinjuku Scheduler：

```bash
pwd # 确保在ghost_userspace目录下
bazel build -c opt fifo_centralized_agent && sudo bazel-bin/fifo_centralized_agent 2> agent_time
```

另一个执行python脚本：

```bash
pwd # 确保在ghost_userspace目录下
bazel build -c opt task_delegation_latency_load && sudo python3 task_delegation_latency_test.py
```

等待测试完成即可。最终原始数据保留在项目根目录的`latency_result`文件中，平均值、最大值以及最小值在python脚本对应终端中打印输出。



# Cgroup测试

## COS

打开两个终端，在其中一个运行Shinjuku Scheduler：

```bash
pwd # 确保在cos_userspace目录下
sudo build/shinjuku_scheduler
```

另一个执行python脚本：

```bash
pwd # 确保在cos_userspace目录下
sudo python3 cgroup_latency.py
```

等待测试完成即可。最终原始数据保留在项目根目录的`cgroup_time`文件中，平均值在python脚本对应终端中打印输出。



## ghOSt

打开两个终端，在其中一个运行Shinjuku Scheduler：

```bash
pwd # 确保在ghost_userspace目录下
bazel build -c opt agent_shinjuku && sudo bazel-bin/agent_shinjuku
```

另一个执行python脚本：

```bash
pwd # 确保在ghost_userspace目录下
bazel build -c opt cgroup_latency_load&& sudo python3 cgroup_test.py
```

等待测试完成即可。最终原始数据保留在项目根目录的`cgroup_time`文件中，平均值在python脚本对应终端中打印输出。



# RocksDB实验

## COS

建立RocksDB实验所需临时文件夹：

```bash
sudo mkdir /etc/cos
```

然后打开两个终端，在其中一个运行Shinjuku Scheduler：

```bash
pwd # 确保在cos_userspace目录下
sudo build/shinjuku_scheduler
```

另一个运行RocksDB实验：

```bash
pwd # 确保在cos_userspace目录下
sudo build/rocksdb
```

等待实验完成即可。如需修改吞吐量，只需修改类型为`struct option`的全局变量`options`的参数再重新编译即可。



## ghOSt

建立RocksDB实验所需临时文件夹：

```bash
sudo mkdir /etc/cos
```

确保`experiments/rocksdb/main.cc`文件中前1223行取消注释，1223行后内容注释。

然后打开两个终端，在其中一个运行Shinjuku Scheduler：

```bash
pwd # 确保在ghost_userspace目录下
bazel build -c opt agent_shinjuku && sudo bazel-bin/agent_shinjuku --ghost_cpus "1-11" --preemption_time_slice 500us
```

另一个运行RocksDB实验：

```bash
pwd # 确保在ghost_userspace目录下
bazel build -c opt rocksdb_ghost && sudo bazel-bin/rocksdb_ghost
```

等待实验完成即可。如需修改吞吐量，只需修改类型为`struct option`的全局变量`options`的参数再重新编译即可。



## CFS

确保`experiments/rocksdb/main.cc`文件中前1223行内容注释，1223行后取消注释。

打开一个终端，运行：

```bash
pwd # 确保在ghost_userspace目录下
bazel build -c opt rocksdb_cfs && sudo bazel-bin/rocksdb_cfs --load_generator_cpus 1 --cfs_dispatcher_cpus 2 --num_workers  9 --worker_cpus "3-11" --rocksdb_db_path "/etc/cos/db"  --scheduler cfs --experiment_duration 30s --throughput 10000.0 --batch 12 --get_duration 10us --discard_duration 1s --print_range false --print_get false --print_ns true --print_last false
```

等待实验完成即可。如需修改吞吐量，只需修改命令行参数即可。



## EXT

建立RocksDB实验所需临时文件夹：

```bash
sudo mkdir /etc/cos
```

然后打开两个终端，在其中一个运行Shinjuku Scheduler：

```bash
pwd # 确保在ext-kernel/tools/sched_ext目录下
cd sched/
make scx_shinjuku_bpf
sudo ./scx_shinjuku_bpf -p
```

另一个运行RocksDB实验：

```bash
pwd # 确保在ext-kernel/tools/sched_ext目录下
cd client/
make rocksdb
sudo ./rocksdb
```

等待实验完成即可。如需修改吞吐量，只需修改类型为`struct option`的全局变量`options`的参数再重新编译即可。
