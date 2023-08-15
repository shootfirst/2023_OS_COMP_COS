我们在开发过程中所用的具体硬件配置如下：

> CPU: Intel(R) Core(TM) i7-8700 CPU @ 3 with hyper-threading
>
> Memory: 14GB
>
> OS: Ubuntu 22.04.3 LTS 
>
> ​	with Linux kernel 6.4.0+(COS) && 6.4.0-rc3+(EXT) && 5.11.0+(ghOSt)



# COS环境搭建



COS可以部署在Linux物理机和虚拟机上，但建议部署在Linux物理机以获取更好的性能效果。下文将详细介绍环境搭建的详细步骤。



## COS内核

### 内核编译

安装内核编译所需包：

```bash
sudo apt-get update && sudo apt-get install build-essential gcc g++ make libncurses5-dev libssl-dev bison flex bc libelf-dev
```

克隆COS内核：

```bash
git clone https://github.com/shootfirst/cos_kernel.git
cd cos_kernel/
```

生成内核编译配置文件：

```bash
make localmodconfig
```

修改`.config`文件：

```bash
vim .config
```

1. 删除PSI监测

   查找`CONFIG_PSI`，将其对应行修改为：

   ```bash
   CONFIG_PSI=n
   ```

3. 删除系统吊销密钥

   查找`CONFIG_PSI`，将其对应行修改为：
   
   ```bash
   CONFIG_SYSTEM_REVOCATION_KEYS=""
   ```

然后就可以进行内核编译：

```bash
make -j12 && sudo make modules_install && sudo make install
```



### 修改grub

打开grub配置文件：

```bash
sudo vim /etc/default/grub
```

进行以下修改：

1. 注释`GRUB_TIMEOUT_STYLE=hidden`
2. 将`GRUB_CMDLINE_LINUX_DEFAULT`设置为”text”
3. 将`GRUB_TIMEOUT`修改成30

然后保存退出，更新grub：

```bash
sudo update-grub
```



### 进入COS内核

完成上述步骤后，重启虚拟机：

```bash
sudo reboot
```

在进入GRUB界面时选择`Advanced Ubuntu`，然后选择内核版本6.4.0+即可。



## COS用户态

在完成COS内核编译，并进入COS内核之后，就完成了COS的基本环境搭建。接下来，将介绍如何搭建COS用户态环境，从而运行Shinjuku Scheduler和RocksDB实验。

首先，确保所处内核正确：

```bash
$ uname -r
6.4.0+
```

### 依赖安装

#### apt包

安装编译用户态需要的包：

```
sudo apt-get install cmake python2 python3 libtbb-dev libsnappy-dev zlib1g-dev libgflags-dev libbz2-dev liblz4-dev libzstd-dev
```



#### RocksDB

获取RocksDB 6.15.5版本release：

```
wget https://github.com/facebook/rocksdb/archive/refs/tags/v6.15.5.tar.gz
tar -xvf v6.15.5.tar.gz
cd rocksdb-6.15.5/
```

> 注：测试得发现本RocksDB负载同最新版本RocksDB不兼容，故而建议使用上述命令对应版本，也即v6.15.5。

修改CMakeLists：

```
vim CMakeLists.txt
```

1. 搜索`WITH_TBB`，将这一项改为ON:

   ```bash
   option(WITH_TBB "build with Threading Building Blocks (TBB)" ON)
   ```

2. 搜索`ROCKSDB_LITE `，**确保这一项为OFF**

   ```bash
   option(ROCKSDB_LITE "Build RocksDBLite version" OFF)
   ```

保存退出后进行编译安装：

```bash
mkdir build && cd build && cmake .. && make -j12 && sudo make install
```



### 运行COS

克隆COS用户态代码：

```bash
git clone https://gitlab.eduxiji.net/202318123111334/cos_userspace.git
cd cos_userspace
```

编译COS用户态：

```bash
mkdir build && cd build && cmake .. && make -j($nproc)
```

然后就可以开始运行COS用户态了。在此以Fifo Scheduler为例。

打开两个终端，在其中一个运行Fifo Scheduler：

```bash
pwd # 确保在cos_userspace/build目录下
sudo ./fifo_scheduler 
```

另一个运行GTest测试：

```bash
pwd # 确保在cos_userspace/build目录下
sudo ./simple_exp
```

等待测试完成即可。

若要运行展示中所提到的测试，可详细见[性能测试教程](./run_test.md)。



# EXT环境搭建

## EXT内核

### 依赖安装

#### llvm

SCHED-EXT内核由于用到新eBPF特性，故而编译需要用到还未发行到apt包管理器的clang最新版本，因此需要手动拉取并且编译一些依赖包。

```bash
# 克隆llvm仓库
git clone --depth=1 https://github.com/llvm/llvm-project.git

# 编译llvm项目
cd llvm-project
mkdir build
cd build
cmake -DLLVM_ENABLE_PROJECTS=clang -DCMAKE_BUILD_TYPE=Release -G "Unix Makefiles" ../llvm
make -j($nproc)

# 在~/.bashrc文件添加
export PATH=$PATH:/yourpath/llvm-project/build/bin

# 确认
echo $PATH
clang --version # >= 17.0.0
llvm-config --version # >= 17.0.0
```



#### pahole

```bash
# 克隆pahole项目
git clone git://git.kernel.org/pub/scm/devel/pahole/pahole.git

# 编译pahole项目
cd pahole/
mkdir build
cd build
cmake -D__LIB=lib -DBUILD_SHARED_LIBS=OFF ..
make
sudo make install

# 检查
$ pahole --version # >= v1.25
```



#### rust-nightly

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
rustup toolchain install nightly
rustup default nightly
# 查看rust版本
rustc --version
```



### 内核编译

安装内核编译所需包：

```bash
sudo apt-get update && sudo apt-get install build-essential gcc g++ make libncurses5-dev libssl-dev bison flex bc libelf-dev
```

克隆COS内核：

```bash
git clone https://gitlab.eduxiji.net/202318123111334/ext-kernel.git
cd ext-kernel/
```

生成内核编译配置文件：

```bash
make localmodconfig
```

对生成的.config配置文件做如下修改：

1. 将`CONFIG_SYSTEM_TRUSTED_KEYS="debian/canonical-certs.pem"`修改为`CONFIG_SYSTEM_TRUSTED_KEYS=""` 
2. 添加`CONFIG_SCHED_CLASS_EXT=y`
3. 确保`CONFIG_DEBUG_INFO`以及`CONFIG_DEBUG_INFO_BTF`为开启状态

编译安装内核：

```bash
make -j12 && sudo make modules_install && sudo make install
```

### 修改grub

详见COS环境搭建的对应部分。在此不做赘述。



## EXT用户态

```bash
pwd # 确保在ext-kernel/项目根目录下
cd ext-kernel/tools
```

随后，我们需要替换原有EXT使用示例为我们搭建的EXT用户态框架。

```bash
rm -rf sched_ext/
git clone https://gitlab.eduxiji.net/202318123111334/proj134-cfs-based-userspace-scheduler.git sched_ext
```

然后就可以运行EXT用户态框架了。



若要运行展示中所提到的测试，可详细见[性能测试教程](./run_test.md)。


# ghOSt环境搭建



由于测试中需要以ghOSt作为比较对象，故在运行测试之前，必须搭建ghOSt环境。



## ghOSt内核

### 内核编译

安装内核编译所需包：

```bash
sudo apt-get update && sudo apt-get install build-essential gcc g++ make libncurses5-dev libssl-dev bison flex bc libelf-dev
```

克隆ghOSt内核：

```bash
git clone https://github.com/google/ghost-kernel
cd ghost-kernel/
```

生成内核编译配置文件：

```bash
make localmodconfig
```

对生成的.config配置文件做如下修改：

1. 将`CONFIG_SYSTEM_TRUSTED_KEYS="debian/canonical-certs.pem"`修改为`CONFIG_SYSTEM_TRUSTED_KEYS=""` 
2. 添加`CONFIG_SCHED_CLASS_GHOST=y`

编译安装内核：

```bash
make -j12 && sudo make modules_install && sudo make install
```



### 修改grub

详见COS环境搭建的对应部分。在此不做赘述。



## ghOSt用户态

### 依赖安装

#### bazel

> 详细安装可参照官方文档[Installing Bazel on Ubuntu](https://docs.bazel.build/versions/4.0.0/install-ubuntu.html#ubuntu)，在此仅给出其中第一种方法。

将Bazel分发URL添加为软件包来源

```bash
# 在~目录下
sudo apt install apt-transport-https curl gnupg -y
curl -fsSL https://bazel.build/bazel-release.pub.gpg | gpg --dearmor >bazel-archive-keyring.gpg
sudo mv bazel-archive-keyring.gpg /usr/share/keyrings
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/bazel-archive-keyring.gpg] https://storage.googleapis.com/bazel-apt stable jdk1.8" | sudo tee /etc/apt/sources.list.d/bazel.list
```

安装和更新Bazel

```bash
sudo apt update && sudo apt install bazel
# 将bazel升级到最新版本
sudo apt update && sudo apt full-upgrade
```



#### apt包

```bash
sudo apt update
sudo apt install libnuma-dev libcap-dev libelf-dev libbfd-dev gcc clang-12 llvm zlib1g-dev python-is-python3 libabsl-dev
```



### 运行ghOSt

克隆ghOSt用户态：

```bash
git clone https://gitlab.eduxiji.net/202318123111334/ghost_userspace.git
cd ghost-userspace
```


