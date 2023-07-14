# 1.调优概述
## 1.1 HBase介绍
HBase–Hadoop Database，是一个高可靠性、高性能，面向列、可伸缩的分布式存储系统，利用HBase技术可以在廉价的PC Server上搭建起大规模结构化存储集群。

HBase主要有三个组件，分别是HMaster、HRegionServer和ZooKeeper，三个组件的主要职责如下。

#### HMaster

HMaster是整个HBase组件的控制者，主要有以下职责：

- 负载均衡。
- 权限管理（ACL）。
- HDFS上的垃圾文件回收。
- 管理namespace和table的元数据。
- 表格的创建、删除和更新（列族的更新）。
- Region分配：启动时分配、失效RegionServer上Region的再分配、Region切分时分配。

#### HRegionServer

HRegionServer是HBase实际读写者，主要有以下职责：

- Region切分。
- HDFS交互，管理table数据。
- 响应client的读写请求，进行I/O操作。

#### ZooKeeper

ZooKeeper是HBase的协调者，主要有以下职责：

- 存储HBase中表格的元数据信息。
- 保证集群中有且只有一个HMaster为Active。
- 存储hbase:meta，即所有Region的位置信息。
- 监测RegionServer状态，将RS的上下线情况汇报给HMaster。
- ZooKeeper集群本身使用一致性协议（PAXOS协议）保证每个节点状态的一致性。

## 1.2 环境介绍
#### 物理组网

物理环境HDP采用控制节点*1+数据节点*3的部署方案，在网络层面，管理与业务分离部署，管理网络使用GE电口来进行网络间的通信，业务网络使用10GE光口来进行网络间的通信，组网如[图1](javascript:;)所示。

**图1** 物理环境组网
![点击放大](https://www.hikunpeng.com/doc_center/source/zh/kunpengbds/ecosystemEnable/HBase/zh-cn_image_0000001108626930.png)

#### 软件版本

使用到的相关软件版本如[表1](javascript:;)所示。



| 软件      | 版本                      |
| --------- | ------------------------- |
| OS        | openEuler 20.03 LTS       |
| JDK       | OpenJDK version 1.8.0_272 |
| HDP       | 3.1.0                     |
| Zookeeper | 3.4.9                     |
| Hadoop    | 3.1.1                     |
| HBase     | 2.0.0                     |

#### 节点部署规划

| 服务器  | 组件部署                                                     |
| ------- | ------------------------------------------------------------ |
| server1 | NameNode、ResourceManager、Timeline Service V2.0 Reader、YARN Registry DNS、Timeline Service V1.5、History Server、HBase Master、Zookeeper Server、Metricks Collector、Grafana、Activity Analyzer、Activity Explorer、HST Server |
| agent1  | SNameNode、DataNode、RegionServer、NodeManager               |
| agent2  | Zookeeper Server、DataNode、RegionServer、NodeManager        |
| agent3  | Zookeeper Server、DataNode、RegionServer、NodeManager        |

## 1.3 调优原则
性能调优就是对计算机硬件、操作系统和应用有相当深入的了解，调节三者之间的关系，实现整个系统（包括硬件、操作系统、应用）的性能最大化，并能不断的满足现有的业务需求。在性能优化时，我们必须遵循一定的原则，否则，有可能得不到正确的调优结果。主要有以下几个方面：

- 对性能进行分析时，利用nmon等性能分析工具，多方面分析系统的资源瓶颈所在，因为系统某一方面性能低，也许并不是它自己造成的，而是其他方面造成的。
- 一次只对影响性能的某方面的一个参数进行调整，例如在HBase调优过程中，Bulkload的影响参数有表的region大小、map的个数等，每次选取某一个参数进行调优，否则很难界定性能的影响是由哪个参数造成的。
- 在进行系统性能分析时，性能分析工具本身会占用一定的系统资源，如CPU资源、内存资源等等。我们必须注意到这点，即分析工具本身运行可能会导致系统某方面的资源瓶颈情况更加严重。

## 1.4 调优思路
性能调优首先要发现问题，找到性能瓶颈点，然后根据瓶颈所处层级选择优化的方法。

调优分析思路如下：

1. 对于服务端的问题，需要重点定位的硬件指标包括CPU、内存、硬盘、BIOS配置，对于读写测试用例重点关注磁盘以及网络的IO性能；对于计算密集型例如Bulkload，需要关注CPU瓶颈。
2. 对于网络问题，网卡中断绑核对于性能提升增益可观。

# 2.硬件调优
## 2.1 配置BIOS
#### 目的

对于不同的硬件设备，通过在BIOS中设置一些高级选项，可以有效提升服务器性能。

- 服务器上的SMMU一般用来完成设备的地址转换，并且可以实现设备隔离，在虚拟化中很实用，但是在物理机测试场景下，SMMU可能会导致性能下降，尤其对于小包网络场景，因此建议关闭该功能提升服务器性能。在虚拟机场景需要打开此配置来使用PCI直通功能。
- 在本测试场景中，预取会导致cache污染，cache miss增加，因此建议关闭预取功能。

#### 方法

1. 关闭SMMU。

   

   1. 重启服务器，按Esc键进入BIOS设置界面。

   2. 依次进入

      “Advanced > MISC Config > > Support Smmu”

      。

      **图1** BIOS设置界面
      ![点击放大](https://www.hikunpeng.com/doc_center/source/zh/kunpengbds/ecosystemEnable/HBase/zh-cn_image_0000001154826923.png)

   3. 将

      “Support Smmu”

      设置为

      “Disabled”

      ，按“F10”保存退出（永久有效）。

      **图2** 关闭SMMU BIOS界面
      ![点击放大](https://www.hikunpeng.com/doc_center/source/zh/kunpengbds/ecosystemEnable/HBase/zh-cn_image_0000001108147314.png)

   

2. 关闭预取。

   

   1. 进入BIOS设置界面。

   2. 依次进入

      “Advanced > MISC Config > CPU Prefetching Configuration”

      。

      **图3** BIOS设置界面
      ![点击放大](https://www.hikunpeng.com/doc_center/source/zh/kunpengbds/ecosystemEnable/HBase/zh-cn_image_0000001154986843.png)

   3. 将

      “CPU Prefetching Configuration”

      设置为

      “Disabled”

      ，按“F10”保存退出（永久有效）。

      **图4** 关闭Prefetching Configuration界面
      ![点击放大](https://www.hikunpeng.com/doc_center/source/zh/kunpengbds/ecosystemEnable/HBase/zh-cn_image_0000001154706973.png)
## 2.2 创建RAID 0
#### 目的

磁盘创建RAID可以提高磁盘的整体存取性能，环境上若使用的是LSI 3108/3408/3508系列RAID卡，推荐创建单盘RAID 0/RAID 5，充分利用RAID卡的Cache（LSI 3508卡为2GB），提高读写速率。

#### 方法

1. 使用storcli64_arm文件检查RAID组创建情况。

   

   `./storcli64_arm /c0 show `

   ![img](https://r.huaweistatic.com/s/ascendstatic/lst/as/software/mindx/modelzoo/img_modelzoo_ts1227.svg)说明

   

   storcli64_arm文件放在任意目录下执行皆可。

   文件下载路径：https://docs.broadcom.com/docs/007.1507.0000.0000_Unified_StorCLI.zip

   

2. 创建RAID 0，命令表示为第2块1.2T硬盘创建RAID 0，其中，c0表示RAID卡所在ID、r0表示组RAID 0。依次类推，对除了系统盘之外的所有盘做此操作。

   

   `./storcli64_arm /c0 add vd r0 drives=65:1 `

   ![点击放大](https://www.hikunpeng.com/doc_center/source/zh/kunpengbds/ecosystemEnable/HBase/zh-cn_image_0000001108626926.png)
## 2.3 开启RAID卡Cache
#### 目的

使用RAID卡的RAWBC（无超级电容）/RWBC（有超级电容）性能更佳。

使用storcli工具修改RAID组的Cache设置，Read Policy设置为Read ahead，使用RAID卡的Cache做预读；Write Policy设置为Write back，使用RAID卡的Cache做回写，而不是Write through透写；IO policy设置为Cached IO，使用RAID卡的Cache缓存IO。

#### 方法

1. 配置RAID卡的Cache。

   

   `./storcli64_arm /c0/vx set rdcache=RA ./storcli64_arm /c0/vx set wrcache=WB/AWB ./storcli64_arm /c0/vx set iopolicy=Cached `

   ![img](https://r.huaweistatic.com/s/ascendstatic/lst/as/software/mindx/modelzoo/img_modelzoo_ts1227.svg)说明

   

   ./storcli64_arm /c0/vx set wrcache=WB/AWB #命令中的WB/AWB取决于RAID卡是否有超级电容，无超级电容使用AWB。

   

2. 检查配置是否生效。

   

   `./storcli64_arm /c0 show`
## 2.4 调整rx_buff
#### 目的

1822网卡默认的rx_buff配置为2KB，在聚合64KB报文的时候需要多片不连续的内存，使用率较低；该参数可以配置为2/4/8/16KB，修改后可以减少不连续的内存，提高内存使用率。

#### 方法

1. 查看rx_buff参数值，默认为2。

   

   `cat /sys/bus/pci/drivers/hinic/module/parameters/rx_buff `

   

2. 在目录“/etc/modprobe.d/”中增加文件hinic.conf，修改rx_buff值为8。

   

   ```reasonml
   options hinic rx_buff=8
   ```

   

3. 重新挂载hinic驱动，使得新参数生效。

   

   `rmmod hinic modprobe hinic `

   

4. 查看rx_buff参数是否更新成功。

   

   `cat /sys/bus/pci/drivers/hinic/module/parameters/rx_buff`
## 2.5 调整Ring Buffer
#### 目的

1822网卡Ring Buffer最大支持4096，默认配置为1024，可以增大buffer大小用于提高网卡Ring大小。

#### 方法

1. 查看Ring Buffer默认大小，假设当前网卡名为enp131s0。

   

   `ethtool -g enp131s0 `

   ![img](https://www.hikunpeng.com/doc_center/source/zh/kunpengbds/ecosystemEnable/HBase/zh-cn_image_0000001108307122.png)

   

2. 修改Ring Buffer值为4096。

   

   ```reasonml
   ethtool -G enp131s0 rx 4096 tx 4096
   ```

   

3. 查看Ring Buffer值是否更新成功。

   

   `ethtool -g enp131s0 `

   ![img](https://www.hikunpeng.com/doc_center/source/zh/kunpengbds/ecosystemEnable/HBase/zh-cn_image_0000001108147308.png)
## 2.6 开启LRO
#### 目的

1822网卡支持LRO（Large Receive Offload），可以打开该选项，并合理配置1822 LRO参数。

#### 方法

1. 查看LRO参数是否开启，默认为off，假设当前网卡名为enp131s0。

   

   `ethtool -k enp131s0 `

   ![img](https://www.hikunpeng.com/doc_center/source/zh/kunpengbds/ecosystemEnable/HBase/zh-cn_image_0000001108467080.png)

   

2. 打开LRO。

   

   ```reasonml
   ethtool -K enp131s0 lro on
   ```

   

3. 查看LRO参数是否开启。

   

   `ethtool -k enp131s0 `

   ![img](https://www.hikunpeng.com/doc_center/source/zh/kunpengbds/ecosystemEnable/HBase/zh-cn_image_0000001154986839.png)
## 2.7 配置网卡中断绑核
#### 目的

相比使用内核的irqbalance使网卡中断在所有核上进行调度，使用手动绑核将中断固定住能有效提高业务网络收发包的能力。

#### 方法

1. 关闭irqbalance。

   

   若要对网卡进行绑核操作，则需要关闭irqbalance。

   1. 停止irqbalance服务，重启失效。

      `systemctl stop irqbalance.service `

   2. 关闭irqbalance服务，永久有效。

      `systemctl disable irqbalance.service `

   3. 查看irqbalance服务状态是否已关闭。

      `systemctl status irqbalance.service `

   

2. 查看网卡pci设备号，假设当前网卡名为enp131s0。

   

   `ethtool -i enp131s0 `

   ![img](https://www.hikunpeng.com/doc_center/source/zh/kunpengbds/ecosystemEnable/HBase/zh-cn_image_0000001108467078.png)

   

3. 查看pcie网卡所属NUMA node。

   

   `lspci -vvvs <bus-info> `

   ![点击放大](https://www.hikunpeng.com/doc_center/source/zh/kunpengbds/ecosystemEnable/HBase/zh-cn_image_0000001154986837.png)

   

4. 查看NUMA node对应的core的区间，例如此处就可以绑到48~63。

   

   `lscpu `

   ![img](https://www.hikunpeng.com/doc_center/source/zh/kunpengbds/ecosystemEnable/HBase/zh-cn_image_0000001154826917.png)

   

5. 进行中断绑核，1822网卡共有16个队列，将这些中断逐个绑至所在NumaNode的16个Core上（例如此处就是绑到NUMA node1对应的48-63上面）。

   

   `bash smartIrq.sh `

   脚本内容如下：

   ```SHELL
   #!/bin/bash
   irq_list=(`cat /proc/interrupts | grep enp131s0 | awk -F: '{print $1}'`)
   cpunum=48  # 修改为所在node的第一个Core
   for irq in ${irq_list[@]}
   do
   echo $cpunum > /proc/irq/$irq/smp_affinity_list
   echo `cat /proc/irq/$irq/smp_affinity_list`
   (( cpunum+=1 ))
   done
   ```

   

   

6. 利用脚本查看是否绑核成功。

   

   `sh irqCheck.sh enp131s0 `

   ![img](https://www.hikunpeng.com/doc_center/source/zh/kunpengbds/ecosystemEnable/HBase/zh-cn_image_0000001154706967.png)

   脚本内容如下：

   ```shell
   #!/bin/bash
   # 网卡名
   intf=$1
   log=irqSet-`date "+%Y%m%d-%H%M%S"`.log
   # 可用的CPU数
   cpuNum=$(cat /proc/cpuinfo |grep processor -c)
   # RX TX中断列表
   irqListRx=$(cat /proc/interrupts | grep ${intf} | awk -F':' '{print $1}')
   irqListTx=$(cat /proc/interrupts | grep ${intf} | awk -F':' '{print $1}')
   # 绑定接收中断rx irq
   for irqRX in ${irqListRx[@]}
   do
   cat /proc/irq/${irqRX}/smp_affinity_list
   done
   # 绑定发送中断tx irq
   for irqTX in ${irqListTx[@]}
   do
   cat /proc/irq/${irqTX}/smp_affinity_list
   done
   ```

# 3.操作系统调优
## 3.1 关闭内存大页
#### 目的

关闭内存大页，防止内存泄漏，减少卡顿。

#### 方法

1. 关闭内存大页。

   ```shell
   echo never > /sys/kernel/mm/transparent_hugepage/enabled
   echo never > /sys/kernel/mm/transparent_hugepage/defrag
   ```

   

2. 检查是否关闭。

   

   `cat /sys/kernel/mm/transparent_hugepage/enabled `

   从下图可以看出内存大页已经从always变成never。

   ![点击放大](https://www.hikunpeng.com/doc_center/source/zh/kunpengbds/ecosystemEnable/HBase/zh-cn_image_0000001154867011.png)


## 3.2 关闭swap
#### 目的

Linux的虚拟内存会根据系统负载自动调整，内存页（page）swap到磁盘会影响测试性能。

#### 方法

```
swapoff -a 
```

修改前后对比。

![点击放大](https://www.hikunpeng.com/doc_center/source/zh/kunpengbds/ecosystemEnable/HBase/zh-cn_image_0000001108467084.png)

# 4.HBase调优
## 4.1 组件参数配置 
对于参数的修改，Ambari基本都是在集群的网页上面修改即可，以Yarn组件为例，单击右边需要更改参数的组件Yarn，参数修改都是在对应组件的CONFIGS里面。修改配置的地方对应的就是在SETTING以及ADVANCED中，如下图所示。

![点击放大](https://www.hikunpeng.com/doc_center/source/zh/kunpengbds/ecosystemEnable/HBase/zh-cn_image_0000001154706969.png)

当要修改具体的参数的时候，主要分两种情况，第一种对于参数原本就在网页设置了初始值的，只需要在CONFIGS页面的搜索栏直接搜索、修改即可，例如修改Yarn中的yarn.nodemanager.resource.cpu-vcores，只需要在搜索框搜索“yarn.nodemanager.resource.cpu-vcores”，单击修改按钮，设置对应的参数即可。

![点击放大](https://www.hikunpeng.com/doc_center/source/zh/kunpengbds/ecosystemEnable/HBase/zh-cn_image_0000001154826919.png)

第二种对于原本页面上没有设置初始值的参数，例如yarn.nodemanager.numa-awareness.enabled这一参数，就需要在对应组件的Custom yarn-site中手动添加（其余组件需要手动添加的也是在对应的Custom ***-site文件中）。

![点击放大](https://www.hikunpeng.com/doc_center/source/zh/kunpengbds/ecosystemEnable/HBase/zh-cn_image_0000001108147310.png)

如下参数为本次测试所配置参数，x86计算平台和鲲鹏920计算平台参数仅有Yarn部分参数有差异（差异处表格中有体现），HBase和HDFS采用同一套参数进行测试。

| 组件                                   | 参数名                                        | 推荐值                           | 修改原因                                                     |
| -------------------------------------- | --------------------------------------------- | -------------------------------- | ------------------------------------------------------------ |
| Yarn->NodeManagerYarn->ResourceManager | ResourceManager Java heap size                | 1024                             | 修改JVM内存大小，保证内存水平较高，减少GC的频率。            |
| NodeManager Java heap size             | 1024                                          |                                  |                                                              |
| Yarn->NodeManager                      | yarn.nodemanager.resource.cpu-vcores          | 与实际数据节点物理核相等。       | 可分配给Container的CPU核数。                                 |
| Yarn->NodeManager                      | yarn.nodemanager.resource.memory-mb           | 与实际数据节点物理内存总量相等。 | 可分配给Container的内存。                                    |
| Yarn->NodeManager                      | yarn.nodemanager.numa-awareness.enabled       | true                             | NodeManager启动Container时的Numa感知，需手动添加。           |
| Yarn->NodeManager                      | yarn.nodemanager.numa-awareness.read-topology | true                             | NodeManager的Numa拓扑自动感知，需手动添加。                  |
| MapReduce2                             | mapreduce.map.memory.mb                       | 7168                             | 一个Map Task可使用的内存上限。                               |
| MapReduce2                             | mapreduce.reduce.memory.mb                    | 14336                            | 一个Reduce Task可使用的资源上限。                            |
| MapReduce2                             | mapreduce.job.reduce.slowstart.completedmaps  | 0.35                             | 当Map完成的比例达到该值后才会为Reduce申请资源。              |
| HDFS->NameNode                         | NameNode Java heap size                       | 3072                             | 修改JVM内存大小，保证内存水平较高，减少GC的频率。            |
| NameNode new generation size           | 384                                           |                                  |                                                              |
| NameNode maximum new generation size   | 384                                           |                                  |                                                              |
| HDFS->DataNode                         | dfs.datanode.handler.count                    | 512                              | DataNode服务线程数，可适量增加。                             |
| HDFS->NameNode                         | dfs.namenode.service.handler.count            | 128                              | NameNode RPC服务端监测DataNode和其他请求的线程数，可适量增加。 |
| HDFS->NameNode                         | dfs.namenode.handler.count                    | 1200                             | NameNode RPC服务端监测客户端请求的线程数，可适量增加。       |
| HBase->RegionServer                    | HBase RegionServer Maximum Memory             | 31744                            | 修改JVM内存大小，保证内存水平较高，减少GC的频率。            |
| HBase->RegionServer                    | hbase.regionserver.handler.count              | 150                              | RegionServer上的RPC服务器实例数量。                          |
| HBase->RegionServer                    | hbase.regionserver.metahandler.count          | 150                              | RegionServer中处理优先请求的程序实例的数量。                 |
| HBase->RegionServer                    | hbase.regionserver.global.memstore.size       | 0.4                              | 最大JVM堆大小（Java -Xmx设置）分配给MemStore的比例。         |
| HBase->RegionServer                    | hfile.block.cache.size                        | 0.4                              | 数据缓存所占的RegionServer GC -Xmx百分比。                   |
| HBase->RegionServer                    | hbase.hregion.memstore.flush.size             | 267386880                        | Regionserver memstore大小，增大可以减小阻塞。                |

## 4.2 NUMA特性开启
Yarn组件在3.1.0版本合入的新特性支持，支持Yarn组件在启动Container时使能NUMA感知功能，原理是读取系统物理节点上每个NUMA节点的CPU核、内存容量，使用Numactl命令指定启动container的CPU范围和membind范围，减少跨片访问。

1. 安装numactl。

   

   `yum install numactl.aarch64 -y `

   

2. 开启NUMA感知（在[组件参数配置](https://www.hikunpeng.com/document/detail/zh/kunpengbds/ecosystemEnable/HBase/kunpenghbasehdp_05_0018_0.html)中已经做过的可以不做）。

   

   增加Yarn->NodeManager：

   ```reasonml
   yarn.nodemanager.numa-awareness.enabled              true
   yarn.nodemanager.numa-awareness.read-topology    true
   ```

## 4.3 读写测试用例
#### 用例分析

随机写：随机写CPU平均占用为8%，但是会产生大量的IO。主要是为HBase写hlog、memstore flush和compaction三个流程会产生大量IO。其中写hlog会产生大量的小IO。因此该用例对IO性能比较敏感，同样对CPU性能不敏感，通用由于有较多的网络小包，因此通过使能网卡lro会使写的性能提升。

随机读：随机读用例整体CPU资源利用很低，整体的IO也是在用例初始时进行hfile的读取到blockcache中，之后数据会有大量的cache命中。由于网络中存在的较多小包，因此主要使能了网卡lro功能，对网络小包进行聚合。

顺序扫描：顺序扫描用例CPU资源利用也很低，整体的IO也是在用例初始时进行hfile的读取到blockcache中，之后的数据都是从blockcache中进行读取。该用例和随机读用例的流程是相同的，只是在读取keyvalue时会顺序的读取一段连续的keyvalue区域。因此scan时的网络包聚合效果不明显。

#### Slow sync调优

**问题描述**

HBase在高并发put场景（随机写用例），由于同步写WAL机制导致随机写性能很差，日志中打印sync时延很高。![点击放大](https://www.hikunpeng.com/doc_center/source/zh/kunpengbds/ecosystemEnable/HBase/zh-cn_image_0000001108307124.png)

**问题分析**

由于同步写WAL机制，必须保证数据落盘才能保证可靠性，所以导致在写流程中写WAL时延很高，严重影响性能。

**解决方法**

硬盘组单盘创建RAID 0，然后配置RAID卡Cache策略为RAWBC，使用RAID卡的Cache做回写，而不是Write through透写，可以极大提高随机写性能。

RAID组Cache策略配置：

RWBC -> Read ahead + Write back + Cached IO

#### 网卡参数调优

在HBase读写测试场景，测试的记录条的大小为1KB，因此使能网卡lro功能和调大网卡Ring buffer。将网络中的小包进行聚合，挺高网络性能。

## 4.4 Bulkload测试用例
#### 用例分析

Bulkload用例前半部分鲲鹏计算平台48核CPU占用90%+，x86计算平台CPU占用100%。分析Bulkload用例具体流程如下：

1. map阶段：该阶段是并发生成hfile，根据数据量的大小，map阶段会有上万个并发去加载hdfs中待导入的数据，然后进行格式转换，格式转换过后会对数据进行校验，检测kv是否有效。最后会对生成的hfile进行压缩。这一过程会消耗大量的CPU。现有mapreduce配置为1个map申请一个vcore。
2. reduce阶段：根据region个数将生成的hfile放置到不同的region。reduce阶段的并发数量是根据region个数来决定的。

#### Map优化

Bulkload的ImportTsv默认是以HDFS的blocksize（默认128MB）来切分数据文件，如200G的数据文件大概有1600多个map任务，但是并没有相应的参数设定来修改map数，故通过更改ImportTsv源码，ImportTsv该类具体位于HBase源码的hbase-mapreduce-2.0.2.3.1.0.0-78.jar中，具体的路径如下。

![点击放大](https://www.hikunpeng.com/doc_center/source/zh/kunpengbds/ecosystemEnable/HBase/zh-cn_image_0000001154867013.png)

在ImportTsv.java增加一个配置参数，即增加一个成员变量：

![点击放大](https://www.hikunpeng.com/doc_center/source/zh/kunpengbds/ecosystemEnable/HBase/zh-cn_image_0000001108467082.png)

在createSubmittableJob方法中增加如下代码：

![点击放大](https://www.hikunpeng.com/doc_center/source/zh/kunpengbds/ecosystemEnable/HBase/zh-cn_image_0000001154986841.png)

将该jar包重新编译后，通过find查找到jar包所在位置，替换到对应的HBase源码中。

![点击放大](https://www.hikunpeng.com/doc_center/source/zh/kunpengbds/ecosystemEnable/HBase/zh-cn_image_0000001154706971.png)

替换之后就可以在ImportTsv上配置一个mapreduce.split.minsize参数，参照如下。

```
hbase org.apache.hadoop.hbase.mapreduce.ImportTsv -Dimporttsv.columns=HBASE_ROW_KEY,f1:H_NAME,f1:ADDRESS -Dimporttsv.separator="," -Dimporttsv.skip.bad.lines=true -Dmapreduce.split.minsize=5368709120 -Dimporttsv.bulk.output=/tmp/hbase/hfile ImportTable /tmp/hbase/datadirImport 
```

![img](https://r.huaweistatic.com/s/ascendstatic/lst/as/software/mindx/modelzoo/img_modelzoo_ts1227.svg)说明



-Dmapreduce.split.minsize=5368709120：该值就是设定了数据文件的分割数值，进而可以修改map数值，设置map数时可以将map数与CPU核数设置差不多。

#### Reduce优化

查看任务运行的时间，发现reduce运行时间很不均衡，有的3分钟，有的40秒，相差很大，故尝试增加reduce数，使每个reduce处理数据更均衡。

![点击放大](https://www.hikunpeng.com/doc_center/source/zh/kunpengbds/ecosystemEnable/HBase/zh-cn_image_0000001154826921.png)

通过修改预分区数（region），可以修改reduce数量。（本次测试设定的是800region）

修改对应文件run_create.txt中的split_num即可修改region数量。

![点击放大](https://www.hikunpeng.com/doc_center/source/zh/kunpengbds/ecosystemEnable/HBase/zh-cn_image_0000001108147312.png)

#### 顺序数据测试

本次bulkload测试的数据为每条1K，单条格式如下：

![点击放大](https://www.hikunpeng.com/doc_center/source/zh/kunpengbds/ecosystemEnable/HBase/zh-cn_image_0000001108307126.png)

HBase的Rowkey是按照ASCII字典排序设计的，排序时会先比对两个Rowkey的第一个字节，如果相同，然后会比对第二个字节，依次类推，直到比较到最后一位。为了保证数据能够均衡写入到每个region，利用位补齐的方法将Rowkey的位数长度设置成一样。

![点击放大](https://www.hikunpeng.com/doc_center/source/zh/kunpengbds/ecosystemEnable/HBase/zh-cn_image_0000001108626928.png)



