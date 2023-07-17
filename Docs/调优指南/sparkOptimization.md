# spark调优指南
## 1-组件架构
### 1.1 物理组网
物理环境HDP采用控制节点*1+数据节点*3的部署方案，在网络层面，管理与业务分离部署，管理网络使用GE电口来进行网络间的通信，业务网络使用10GE光口来进行网络间的通信。
### 1.2 软件版本
| 软件  | 版本  |
|---|---|
| OS  | openEuler 22.03 |
| kernel  | 4.19.90-2005.1.0.0038.oe1.aarch64  |
| JDK  | Openjdk version “1.8.0_272”  |
| Zookeeper |3.4.9 |
| Hadoop | 3.1.1 |
| Hive | 3.0.0 |
| Spark | 2.3.0 |
| Tez | 0.9.0 |
### 1.3 节点部署规划
| 服务器  | 组件部署   |
|---|---|
| server1 | NameNode、ResourceManager、Timeline Service V2.0 Reader、YARN Registry DNS、Timeline Service V1.5、SNameNode 、History Server、Metricks Collector、Grafana、Activity Analyzer、Activity Explorer、HST Server、Spark2 History Server、Spark2 Thrift Server |
| agent1 | Zookeeper Server 、DataNode |
| agent2 | Zookeeper Server、DataNode |
| agent3 | Zookeeper Server、DataNode |
### 1.4 调优原则

![输入图片说明](https://foruda.gitee.com/images/1679283857872121110/890f3193_10227919.png "屏幕截图")

在性能优化时，我们必须遵循一定的原则，否则有可能得不到正确的调优结果。
主要包括以下几个方面：
- 对性能进行分析时，要多方面分析系统资源瓶颈所在，学会利用web端获取到的日志进行分析。
- 每一次调整参数时，采用控制变量法，只对影响性能的某方面的一个参数进行调整，避免调整多个参数带来的问题分析定位不准确的问题

### 1.5 调优思路
- 对sql性能进行分析时，在客户端及Web按照基础固定值配置的基础上，按照理论计算公式得到较合理的一组Executor执行参数，使能在鲲鹏上效果明显的亲和性配置调优，进行SQL测试能跑出较优的一组性能结果。
- 在Hibench场景下，根据集群总核数，采用3-5倍总核数作为数据分片的Partitions和Parallelism进行数据分片，减小单Task文件大小，对性能有正面提升。
- 调优时尽量在任务运行时CPU与内存尽量压满，当然若不能满足同时压满，则优先压满CPU，然后根据实际GC的log判断是否需要增加内存；在内存需求较高的场景，最终逐步调整可能会趋向内存压满，CPU留有一定余量的情况。

## 2-硬件优化
### 2.1 配置BIOS
 **目的：** 对于不同的硬件设备，通过在BIOS中设置一些高级选项，可以有效提升服务器性能。
- 服务器上的SMMU一般用来完成设备的地址转换，并且可以实现设备隔离，在虚拟化中很实用，但是在物理机测试场景下，SMMU可能会导致性能下降，尤其对于小包网络场景，因此建议关闭该功能提升服务器性能。在虚拟机场景需要打开此配置来使用PCI直通功能。
- 在本测试场景中，预取会导致cache污染，cache miss增加，因此建议关闭预取功能。
```
方法 
关闭SMMU。
说明
- 此优化项只在非虚拟化场景使用，在虚拟化场景，则开启SMMU。
- 重启服务器过程中，单击Delete键进入BIOS，选择“Advanced > MISC Config”，单击Enter键进入。
- 将“Support Smmu”设置为“Disable” 。
- 关闭预取。
在BIOS中，选择“Advanced>MISC Config”，单击Enter键进入。
- 将“CPU Prefetching Configuration”设置为“Disabled”。
```
### 2.2 创建RAID 0
```
目的
磁盘创建RAID可以提高磁盘的整体存取性能，环境上若使用的是LSI 3108/3408/3508系列RAID卡，推荐创建单盘RAID 0/RAID 5，充分利用RAID卡的Cache（LSI 3508卡为2GB），提高读写速率。
方法
- 使用storcli64_arm文件检查RAID组创建情况。
./storcli64_arm /c0 show
- 创建RAID 0，命令表示为第2块1.2TB硬盘创建RAID 0，其中，c0表示RAID卡所在ID、r0表示组RAID 0。依次类推，对除了系统盘之外的所有盘做此操作。
./storcli64_arm /c0 add vd r0 drives=65:1
```
![输入图片说明](https://foruda.gitee.com/images/1679292452664538651/90ba8b2b_10227919.png "屏幕截图")
### 2.3 开启RAID卡Cache
```
目的
使用RAID卡的RWTD性能更佳。

使用storcli工具修改RAID组的Cache设置，Read Policy设置为Read ahead，使用RAID卡的Cache做预读；Write Policy设置为Write Through；IO policy设置为Direct IO。

方法
- 配置RAID卡的cache，其中vx是卷组号（v0/v1/v2....），根据环境上的实际情况进行设置。
./storcli64_arm /c0/vx set rdcache=RA
./storcli64_arm /c0/vx set wrcache=WT
./storcli64_arm /c0/vx set iopolicy=Direct
- 检查配置是否生效。
./storcli64_arm /c0 show
```
![输入图片说明](https://foruda.gitee.com/images/1679292548783445749/728f38d6_10227919.png "屏幕截图")

### 2.4 调整rx_buff
```
目的
1822网卡默认的rx_buff配置为2KB，在聚合64KB报文的时候需要多片不连续的内存，使用率较低；该参数可以配置为2/4/8/16KB，修改后可以减少不连续的内存，提高内存使用率。
方法
- 查看rx_buff参数值，默认为2。
cat /sys/bus/pci/drivers/hinic/module/parameters/rx_buff
- 在目录“/etc/modprobe.d/”中增加文件hinic.conf，修改rx_buff值为8。
options hinic rx_buff=8
- 重新挂载hinic驱动，使得新参数生效。
rmmod hinic
modprobe hinic
- 查看rx_buff参数是否更新成功。
cat /sys/bus/pci/drivers/hinic/module/parameters/rx_buff
```
### 2.5 调整队列深度和队列数
```
目的
1822网卡队列深度最大支持4096，默认配置为1024，可以增大buffer大小用于提高网卡Ring大小。
方法
- 查看队列深度默认大小，假设当前网卡名为enp131s0。
ethtool -g enp131s0
- 修改队列深度值为4096。
ethtool -G enp131s0 rx 4096 tx 4096
- 查看队列深度值是否更新成功。
ethtool -g enp131s0
- 减少队列数。
ethtool -L enp131s0 combined 4
ethtool -l enp131s0
```
### 2.6 开启GRO、TSO、GSO
```
1822网卡支持LRO（Large Receive Offload），可以打开该选项，并合理配置1822 LRO参数。

方法
- 查看LRO参数是否开启，默认为off，假设当前网卡名为enp131s0。
ethtool -k enp131s0
- 打开LRO。
ethtool -K enp131s0  lro on
ethtool -K enp131s0 gro on
ethtool -K enp131s0 tso on
ethtool -K enp131s0 gso on
ifconfig enp131s0 mtu 9000 up
- 查看LRO参数是否开启。
ethtool -k enp131s0
```
### 2.7 配置网卡中断绑核
```
目的
相比使用内核的irqbalance使网卡中断在所有核上进行调度，使用手动绑核将中断固定住能有效提高业务网络收发包的能力。

方法
- 关闭irqbalance。
若要对网卡进行绑核操作，则需要关闭irqbalance。
- 停止irqbalance服务，重启后会失效。
systemctl stop irqbalance.service
- 关闭irqbalance服务，永久有效。
systemctl disable irqbalance.service
- 查看irqbalance服务状态是否已关闭。
systemctl status irqbalance.service
- 查看网卡pci设备号，假设当前网卡名为enp131s0。
ethtool -i enp131s0
- 查看pcie网卡所属NUMA node。
lspci -vvvs <bus-info>
- 查看NUMA对应哪些CORE。
numactl -H
新建一个shell脚本，在脚本内复制以下内容来进行绑核。其中$1是网卡名称，即上面的enp131s0。
# /bin/bash
t=60
for i in `cat /proc/interrupts |grep -e $1 |awk -F ':' '{print $1}'`
do
cat /proc/irq/$i/smp_affinity_list
echo $t > /proc/irq/$i/smp_affinity_list
cat /proc/irq/$i/smp_affinity_list
t=`expr $t + 1`
done
```
## 3-通用优化项
### 3.1 内存参数
#### 3.1.1 队列调整
```
目的
如果没有特殊说明，全部使用单队列（软队列）模式，单队列模式在spark测试时会有更佳的性能。
方法
- 在/etc/grub2-efi.cfg +142 的内核启动命令行中添加scsi_mod.use_blk_mq=0，重启生效。
linux   /vmlinuz-5.10.0-60.63.0.89.oe2203.aarch64 root=UUID=2d3d625b-c1c9-48cd-ab63-fe5f76f195a2 ro video=VGA-1:640x480-32@60me rhgb quiet console=tty0 crashkernel=1024M,high smmu.bypassdev=0x1000:0x17 smmu.bypassdev=0x1000:0x15 video=efifb:off scsi_mod.use_blk_mq=0
```
#### 3.1.2 其他io相关的配置
```
目的
对调度参数、IO参数进行调整，能有一定的性能提升。

方法
新建一个shell脚本，在脚本内复制以下内容并运行：

#! /bin/bash

echo 3000 > /proc/sys/vm/dirty_expire_centisecs
echo 500 > /proc/sys/vm/dirty_writeback_centisecs

echo 15000000 > /proc/sys/kernel/sched_wakeup_granularity_ns
echo 10000000 > /proc/sys/kernel/sched_min_granularity_ns

systemctl start tuned
sysctl -w kernel.sched_autogroup_enabled=0
sysctl -w kernel.numa_balancing=0

echo 11264 > /proc/sys/vm/min_free_kbytes
echo 60 > /proc/sys/vm/dirty_ratio
echo 5 > /proc/sys/vm/dirty_background_ratio

list="b c d e f g h i j k l m"
for i in $list
do
  echo 1024 > /sys/block/sd$i/queue/max_sectors_kb
  echo 32 > /sys/block/sd$i/device/queue_depth
  echo 256 > /sys/block/sd$i/queue/nr_requests
  echo deadline > /sys/block/sd$i/queue/scheduler
  echo 2048 > /sys/block/sd$i/queue/read_ahead_kb
  echo 2 > /sys/block/sd$i/queue/rq_affinity
  echo 0 > /sys/block/sd$i/queue/nomerges
done

注意
在wordcount等IO密集型的场景，建议使用多队列mq-deadline调度，效果稍好。该场景会在下面单独配置和说明。
```
### 3.2 内存参数
```
目的 ：关闭内存大页，防止内存泄漏，减少卡顿。
方法 ：
- 透明大页对性能影响甚大，关闭：
echo never > /sys/kernel/mm/transparent_hugepage/enabled
echo never > /sys/kernel/mm/transparent_hugepage/defrag
```
### 3.3 JVM参数和版本适配
```
目的 ：最新版本的JDK对Spark性能进行了优化。

- JDK参数更新
可在spark-defaults.conf文件中添加以下配置来使用新的JDK。

spark.executorEnv.JAVA_HOME /usr/local/jdk8u222-b10
spark.yarn.appMasterEnv.JAVA_HOME /usr/local/jdk8u222-b10
spark.executor.extraJavaOptions -XX:+UseNUMA -XX:BoxTypeCachedMax=100000 -XX:ParScavengePerStrideChunk=8192
spark.yarn.am.extraJavaOptions -XX:+UseNUMA -XX:BoxTypeCachedMax=100000 -XX:ParScavengePerStrideChunk=8192
```
### 3.4 spark应用参数
```
目的 ：在Spark基础配置值的基础上，按照理论公式得到一组较合理的Executor执行参数，使能在鲲鹏上会带来明显的性能提升。

方法
- 如果用Spark-Test-Tool工具测试sql1~sql10场景，打开工具目录下的“script/spark-default.conf”文件，添加以下配置项：
yarn.executor.num 15
yarn.executor.cores 19
spark.executor.memory 44G
spark.driver.memory 36G
- 如果使用HiBench工具测试wordcount、terasort、bayesian、kmeans场景，打开工具目录下的“conf/spark.conf”文件，可以根据实际环境对运行核数、内存大小做调整：
yarn.executor.num 15
yarn.executor.cores 19
spark.executor.memory 44G
spark.driver.memory 36G
```
### 3.5 专用场景优化项--SQL场景
#### 3.5.1 sql1-IO密集型SQL
```
目的 ：sql1是IO密集型场景，可以优化IO参数来带来最佳性能。
方法
- 对以下IO相关参数进行设置，其中sd$i指所有参与测试的磁盘名：
echo 128 > /sys/block/sd$i/queue/nr_requests
echo 512 > /sys/block/sd$i/queue/read_ahead_kb
- 对内存脏页参数进行设置：
/proc/sys/vm/vm.dirty_expire_centisecs  500
/proc/sys/vm/vm.dirty_writeback_centisecs   100
该场景其余参数都使用通用优化项的通用优化值。
```
#### 3.5.2 sql2 & sql7 - CPU密集型SQL
```
目的 ：sql2和sql7是CPU密集型场景，可以优化spark执行参数来带来最佳性能。

方法
- Spark-Test-Tool在配置文件（script/spark-default.conf）中指定的运行核数、内存大小可以根据实际环境来做调整，来达到最优性能。比如对于鲲鹏920 5220处理器，sql2和sql7场景建议以下Executor参数。

yarn.executor.num 42
yarn.executor.cores 6
spark.executor.memory 15G
spark.driver.memory 36G
```
#### 3.5.3 sql3-IO + CPU
```
目的 ：sql3是IO+CPU密集型场景，可以优化spark执行参数、调整IO参数来带来最佳性能。

方法
- Spark-Test-Tool在配置文件（script/spark-default.conf）中指定的运行核数、内存大小可以根据实际环境来做调整，来达到最优性能。比如对于鲲鹏920 5220处理器，sql3场景建议以下Executor参数。
yarn.executor.num 30
yarn.executor.cores 6
spark.executor.memory 24G
spark.driver.memory 36G
- 调整io预取值，其中sd$i表示所有参与spark的磁盘名：
echo 4096 > /sys/block/sd$i/queue/read_ahead_kb
其余参数都使用硬件优化的默认参数。
```
#### 3.5.4 sql4 - CPU密集
```
目的 ：sql4是CPU密集型场景，可以优化spark执行参数、调整IO参数来带来最佳性能。

方法
Spark-Test-Tool在配置文件中指定的运行核数、内存大小可以根据实际环境来做调整，来达到最优性能。比如对于鲲鹏920 5220处理器，sql4场景建议以下Executor参数。

- 打开工具目录下的script/spark-default.conf文件，添加以下配置项：
yarn.executor.num 42
yarn.executor.cores 6
spark.executor.memory 15G
spark.driver.memory 36G
- 同时调整io预取值，其中sd$i表示所有参与spark的磁盘名：
echo 4096 > /sys/block/sd$i/queue/read_ahead_kb
```
#### 3.5.5 sql5/6//8/9/10
都使用通用优化项的默认参数。
### 3.6 专用场景优化项--HiBench场景
#### 3.6.1 Wordcount – IO + CPU密集型
```
目的 ：Wordcount是IO+CPU密集型场景，二者均衡，采用单队列的deadline调度算法反而不好，采用多队列的mq-deadline算法并调整相关io参数，能得到较好的性能结果。

方法
- 对以下配置进行修改，其中sd$i指所有参与测试的磁盘名称：
echo mq-deadline > /sys/block/sd$i/queue/scheduler
echo 512 > /sys/block/sd$i/queue/nr_requests
echo 8192 > /sys/block/sd$i/queue/read_ahead_kb
echo 500 > /proc/sys/vm/dirty_expire_centisecs
echo 100 > /proc/sys/vm/dirty_writeback_centisecs
echo 5 > /proc/sys/vm/dirty_background_ratio
- 该场景下采用3-5倍总核数作为数据分片的Partitions和 Parallelism进行数据分片，减小单Task文件大小，对性能有正面提升。可以使用以下分片设置：
spark.sql.shuffle.partitions 300
spark.default.parallelism 600
- HiBench在配置文件中指定的运行核数、内存大小可以根据实际环境来做调整，来达到最优性能。比如对于鲲鹏920 5220处理器，Wordcount场景建议以下Executor参数：
yarn.executor.num 51
yarn.executor.cores 6
spark.executor.memory 13G
spark.driver.memory 36G
- 调整Jdk参数，对性能有正面影响（3%性能提升），以下配置添加到spark.conf文件中：
spark.executor.extraJavaOptions -XX:+UseNUMA -XX:BoxTypeCachedMax=100000 -XX:ParScavengePerStrideChunk=8192
spark.yarn.am.extraJavaOptions -XX:+UseNUMA -XX:BoxTypeCachedMax=100000 -XX:ParScavengePerStrideChunk=8192"
```
#### 3.6.2 Terasort – IO + CPU密集型
网卡绑核调整
```
目的 ：该场景下可以通过减少网卡队列数并绑固定核来优化。

方法
- 请新建一个shell脚本，将以下代码复制到其中并执行。
#! /bin/bash
ethtool -K enp134s0 gro on
ethtool -K enp134s0 tso on
ethtool -K enp134s0 gso on
ifconfig enp134s0 mtu 9000 up
ethtool -G enp134s0 rx 4096 tx 4096
ethtool -G enp134s0 rx 4096 tx 4096
ethtool -L enp134s0 combined 4
- 请新建一个shell脚本，将以下代码复制到其中并执行，其中$1是网卡名称。执行后可以实现绑核：
# /bin/bash
t=32
for i in `cat /proc/interrupts |grep -e $1 |awk -F ':' '{print $1}'`
do
cat /proc/irq/$i/smp_affinity_list
echo $t > /proc/irq/$i/smp_affinity_list
cat /proc/irq/$i/smp_affinity_list
t=`expr $t + 1`
done
```
其他调优
```
echo cfq > /sys/block/sd$i/queue/scheduler
echo 512 > /sys/block/sd$i/queue/nr_requests
echo 8192 > /sys/block/sd$i/queue/read_ahead_kb
echo 4 > /sys/block/sd$i/queue/iosched/slice_idle
echo 0 > /sys/module/scsi_mod/parameters/use_blk_mq
echo 500 > /proc/sys/vm/dirty_expire_centisecs
echo 100 > /proc/sys/vm/dirty_writeback_centisecs
- 该场景下采用3-5倍总核数作为数据分片的Partitions和 Parallelism进行数据分片，减小单Task文件大小，对性能有正面提升。可以使用以下分片设置：
spark.sql.shuffle.partitions 1000
spark.default.parallelism 2000
- HiBench在配置文件中指定的运行核数、内存大小可以根据实际环境来做调整，来达到最优性能。比如对于鲲鹏920 5220处理器，Terasort场景建议以下Executor参数：
yarn.executor.num 27
yarn.executor.cores 7
spark.executor.memory 25G
spark.driver.memory 36G
```
#### 3.6.3 Bayesian – CPU密集型
```
目的 ：Bayesian是CPU密集型场景，可以对IO参数和spark执行参数进行调整。

方法
- 该场景可以使用以下分片设置：
spark.sql.shuffle.partitions 1000
spark.default.parallelism 2500
- 打开HiBench工具的“conf/spark.conf”文件，增加以下Executor参数：
yarn.executor.num 9
yarn.executor.cores 25
spark.executor.memory 73G
spark.driver.memory 36G
- 该场景使用以下内核参数：
echo mq-deadline > /sys/block/sd$i/queue/scheduler
echo 0 > /sys/module/scsi_mod/parameters/use_blk_mq
echo 50 > /proc/sys/vm/dirty_background_ratio
echo 80 > /proc/sys/vm/dirty_ratio
echo 500 > /proc/sys/vm/dirty_expire_centisecs
echo 100 > /proc/sys/vm/dirty_writeback_centisecs
- 调整Jdk参数，以下配置添加到spark.conf文件中：
spark.executor.extraJavaOptions -XX:+UseNUMA -Xms60g -Xmn25g -XX:+UseParallelOldGC -XX:ParallelGCThreads=24 -XX:+AlwaysPreTouch -XX:-UseAdaptiveSizePolicy
```
#### 3.6.4 Kmeans – 纯计算密集型
```
目的
Kmeans是CPU密集型场景，可以对IO参数和spark执行参数进行调整。

方法
- 主要是调整Spark Executor参数适配到一个较优值，该场景可以使用以下分片设置：
spark.sql.shuffle.partitions 1000
spark.default.parallelism 2500
- 使用以下内核参数：
echo 4096 > /sys/block/sd$i/queue/read_ahead_kb
- HiBench在配置文件中指定的运行核数、内存大小可以根据实际环境来做调整，来达到最优性能。比如对于鲲鹏920 5220处理器，K-means场景建议以下Executor参数：
yarn.executor.num 42
yarn.executor.cores 6
spark.executor.memory 15G
spark.driver.memory 36G
spark.locality.wait 10s
- 调整Jdk参数，以下配置添加到spark-default.conf文件中：
spark.executor.extraJavaOptions -XX:+UseNUMA -XX:BoxTypeCachedMax=100000 -XX:ParScavengePerStrideChunk=8192
spark.yarn.am.extraJavaOptions -XX:+UseNUMA -XX:BoxTypeCachedMax=100000 -XX:ParScavengePerStrideChunk=8192
- 毕昇JDK对Kmeans有特定的优化，可以使用毕昇JDK进行加速。

- 下载毕昇JDK地址
下载地址：https://mirrors.huaweicloud.com/kunpeng/archive/compiler/bisheng_jdk/bisheng-jdk-8u262-linux-aarch64.tar.gz
- 更换毕昇JDK
- 停止集群
- 将BiSheng JDK解压并移动到/usr/local/下
tar –zxvf bisheng-jdk-8u262-linux-aarch64.tar.gz

mv bisheng-jdk1.8.0_262 /usr/local/

- 将原来已有的jdk文件重命名并将BiSheng JDK重命名为命名为原来的jdk的文件名并更改权限
mv jdk8u222-b10/ jdk8u222-b10-openjdk/

mv bisheng-jdk1.8.0_262/ jdk8u222-b10/

chmod -R 755 jdk8u222-b10/

重启集群
- 参数优化
/opt/HiBench-HiBench-7.0/conf/spark.conf 中修改spark.executor.extraJavaOptions的值
- 设置为spark.executor.extraJavaOptions -XX:+UnlockExperimentalVMOptions
-XX:+EnableIntrinsicExternal -XX:+UseF2jBLASIntrinsics -Xms43g

-XX:ParallelGCThreads=8

其中-xms（n）g 取决于/opt/HiBench-HiBench-7.0/conf/spark.conf中spark.executor.memory值得大小n的值建议设置为spark.executor.memory的值减1。
```
