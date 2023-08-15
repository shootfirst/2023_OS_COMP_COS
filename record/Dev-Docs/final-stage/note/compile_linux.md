# 编译安装linux详解

## 配置编译选项

linux中有许多编译选项，如kvm，bpf，以及海量的驱动程序。你可以根据自己的需求去选择编译一些，来达到缩减内核编译时间和体积大小的目的。

所有在.config文件中被选中的编译选项会以宏的形式保存在Linux内核根目录下的`include/generated/autoconf.h`文件下，这个宏用于控制编译时的条件编译选项。因而，开启对应的编译选项就会开启内核中的条件编译。

常见的配置选项：

- ！！使用最适配本机的配置

  会执行 `lsmod`命令查看当前系统中加载了哪些模块 (Modules)， 并最后将原来的 .config 中不需要的模块去掉，仅保留前面 lsmod 出来的这些模块，从而简化了内核的配置过程。 这种直接一步到位，我在vmware上就是采用这种。

  ```
  make localmodconfig
  ```

- 使用默认配置

  实测在vmware上启动会卡在根文件系统挂载，因为此时的/dev下根本没有sda

  ```
  make defconfig
  ```

- 使用已有配置文件

  ```
  make oldconfig
  ```

- 交互式配置

  打开图形化界面，让用户手动去配置相关的编译选项。
  读取`arch/$ARCH/`目录下的Kconfig文件生成整个配置界面选项，该Kconfig文件会包含其他子目录下的Kconfig文件。
  记录选还是不选则是在.config文件中。

  如果一开始没有.config文件，则默认会读取/boot下的对应版本内核配置文件。

  ```
  make menuconfig
  ```

* 全部config选项都回答no

  ```
  make allnoconfig
  ```


* 最小配置

  ```
  make tinyconfig
  ```

实际中，我们可以使用直接对`.config`文件进行编辑，和使用menuconfig图形化界面，这两种方法，来对内核进行配置。

下面记录几个常见/我认为比较有意思的内核配置选项。

* 开启BPF

  General setup->BPF subsystem

* 调度相关的选项整理

  开启这些选项就会开启内核中的条件编译，就会为`task_struct`结构体或者别的什么地方增加一些调度信息，对之后的写代码应该挺有帮助的。因而在此做一些简单的整理。

  应该大部分重要的都在General setup里，其他未涉及的自己去.config搜下了解下就行，不多说了。

  * General setup
    * Core Scheduling for SMT是一种处理器调度策略，用于管理同时执行的多个线程（或任务）在超线程技术（Hyper-Threading）中的调度方式。
    * Extensible Scheduling Class
    * CPU/Task time and stats accounting  记录每个任务使用CPU的情况，包括运行时间、等待时间、执行次数等统计信息
    * RCU
      * CONFIG_PREEMPT_RCU 支持在RCU中的抢占机制
      * CONFIG_TASK_RCU  支持RCU对任务的生命周期和调度的管理
    * Scheduling features
      * Enable utilization clamping for RT/FAIR tasks  用于启用或禁用对实时 (`RT`) 和公平 (`FAIR`) 任务的资源利用率限制
    * Memory placement aware NUMA scheduler  启用或禁用针对非一致性内存访问（NUMA）架构的内存调度器
    * Automatic process group scheduling 用于启用或禁用自动进程组调度功能

* Kernel hacking

  记录了内核调试相关的一些支持。有需要可以重点关注下。



## 编译内核


在配置完编译选项后，就可以开始编译内核了。编译器会根据上面指定的编译选项去编译。在linux每个子目录下都有自己的makefile，根据最底层的makefile生成.o文件

脚本scripts/link-vmlinux.sh把不同的编译好的子模块链接到一起形成了vmlinux。

生成的vmlinux在arch/$ARCH/boot/目录下

生成的system-map在当前目录下

生成的为usr/initramfs_data.cpio.d


## 内核模块编译安装

make modules_install 

编译内核模块（选项位M的），将其放入/lib/modules/$kernel-version/

## 安装内核

make install 

在/boot目录下存放一些文件。vmlinuz、initrd、System-map、config。grub.cfg会自动更新，默认启动新内核


## 卸载内核

make clean：删除编译产物

make mrproper：删除编译产物和.config



## 问题记录

1. ALERT! root=UUIDxxx does not exist. Dropping to a shell!

   因缺少磁盘驱动挂载文件系统失败

2. init进程启动失败，error code为-8

   ```c
   Starting init: /sbin/init exists but couldn't execute it (error -8)
   然后init进程被杀死了，然后就进入BusyBox Shell
   ```

   最后发现是不知道为什么它将启动参数中的text误认为init进程的启动参数了，最后的解决方案是删掉启动参数里的text（具体方法可以看GRUB介绍部分，看完大概就懂了）

3. cannot set terminal proccess group(-1): Inappropriate ioctl for device.

   不知道为啥重启下就好了。偶尔出现偶尔不出现，很奇怪

4. initrd.img Unable to mount root fs on Unknown-block(0,0)

   最后发现好像是这个initrd.img损坏了，最后执行`mkinitramfs -o /boot/initrd.img-xxx`重新做了个initrd.img解决问题。

5. Kernel panic - not syncing: No working init found. Try passing init= option to kernel.

   按它说的那样，在GRUB参数加个比方说`init=/bin/bash`就行了

   

