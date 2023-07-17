### 1.调优概述
#### 1.1 Hive介绍
 **Hive工作流** 
    Hive是基于Hadoop的一个数据仓库工具，可以将结构化的数据文件映射为一张表，并提供类SQL查询功能。
![hive执行任务流程图](https://foruda.gitee.com/images/1679293870604619016/63e29923_12582270.png "hive执行任务流程图")
 **流程大致步骤为：** 
1）用户提交查询等任务给Driver。
2）编译器获得该用户的任务Plan。
3）编译器Compiler根据用户任务去MetaStore中获取需要的Hive的元数据信息。
4）编译器Compiler得到元数据信息，对任务进行编译，先将HiveQL转换为抽象语法树，然后将抽象语法树转换成查询块，将查询块转化为逻辑的查询计划，重写逻辑查询计划，将逻辑计划转化为物理的计划（TEZ）, 最后选择最佳的策略。
5）将最终的计划提交给Driver。
6）Driver将计划Plan转交给ExecutionEngine去执行，获取元数据信息，提交给JobTracker或者SourceManager执行该任务，任务会直接读取HDFS中文件进行相应的操作。
7）获取执行的结果。
8）取得并返回执行结果。

从上面的流程可以看出，影响Hive执行的主要有两部分：
1）编译器任务编译。这部分会直接影响查询计划，不同的查询计划影响实际的物理计划（TEZ）。
2）TEZ引擎。这个是Hive任务的主体。

 **Tez引擎** 
    Tez是基于Hadoop YARN之上的DAG（有向无环图，Directed Acyclic Graph）计算框架。Tez直接源于MapReduce框架，核心思想是将Map和Reduce两个操作进一步拆分，即Map被拆分成Input、Processor、Sort、Merge和Output， Reduce被拆分成Input、Shuffle、Sort、Merge、Processor和Output等，被分解后的元操作可灵活组合，产生新的操作，这些操作经一些控制程序组装后，可形成一个大的DAG作业。采用Tez计算框架，生成一个简洁的DAG作业，算子跑完不退出，下轮继续使用上一轮的算子，这样大大减少磁盘IO操作，从而计算速度更快。
![tez引擎](https://foruda.gitee.com/images/1679294129068361556/cf8c79c8_12582270.png "tez引擎")
总结起来，Tez有以下特点：

1）运行在YARN之上。
2）与MapReduce兼容，继承了MapReduce的各种优点（比如良好的扩展性与容错性）。
3）适用于DAG应用场景。

Tez在底层提供了DAG编程接口，用户利用这些接口进行程序编写，其主要由两部分组成：数据处理引擎和DAGAppMaster，其中数据处理引擎为其提供了一套编程接口和数据计算操作符，而DAGAppMaster则是一个YARN ApplicationMaster，它使得Tez应用程序可运行在YARN上。

简单举例演示，例如以下Hive SQL会翻译成四个MR作业，而采用Tez则生成一个DAG作业，可大大减少磁盘IO。
![tez引擎执行任务图](https://foruda.gitee.com/images/1679294292394163183/f5929a81_12582270.png "tez引擎执行任务图")

#### 1.2 环境介绍

物理环境采用控制节点*1（server1）+数据节点*3（agent1、agent2、agent3）的部署方案，在网络层面，管理与业务分离部署，管理网络使用GE电口来进行网络间的通信，业务网络使用10GE光口来进行网络间的通信，组网如下图所示。
![组网](https://foruda.gitee.com/images/1679294533567297550/34c614ad_12582270.png "组网")

 **软件版本** 
| 软件        | 版本                        |
|-----------|---------------------------|
| OS        | OpenEuler22.03            |
| JDK       | OpenJDK version 1.8.0_272 |
| Zookeeper | 3.4.9                     |
| Hive      | 3.0.0                     |
| Hadoop    | 3.1.1                     |
| Spark2x   | 2.3.0                     |
| Tez       | 0.9.0                     |

 **节点部署规划** 
| 机器名称 | 服务名称 |
| ---------------- |---------------- |
|server1|Namenode、ResourceManager、RunJar、RunJar|
|agent1|QuorumPeerMain、DataNode、NodeManager、JournalNode|
|agent2|QuorumPeerMain、DataNode、NodeManager、JournalNode|
|agent3|QuorumPeerMain、DataNode、NodeManager、JournalNode|

#### 1.3 调优规则

性能调优就是对计算机硬件、操作系统和应用有相当深入的了解，调节三者之间的关系，实现整个系统（包括硬件、操作系统、应用）的性能最大化，并能不断的满足现有的业务需求。在性能优化时，我们必须遵循一定的原则，否则，有可能得不到正确的调优结果。主要有以下几个方面：

1）对性能进行分析时，利用nmon等性能分析工具，多方面分析系统的资源瓶颈所在，因为系统某一方面性能低，也许并不是它自己造成的，而是其他方面造成的。
2）一次只对影响性能的某方面的一个参数进行调整，例如在Hive调优过程中，影响参数有内存大小、cbo查询计划等，每次选取某一个参数进行调优，否则很难界定性能的影响是由哪个参数造成的。
3）由于在进行系统性能分析时，性能分析工具本身会占用一定的系统资源，如CPU资源、内存资源等等。我们必须注意到这点，即分析工具本身运行可能会导致系统某方面的资源瓶颈情况更加严重。
#### 1.4 调优思路
性能调优首先要发现问题，找到性能瓶颈点，然后根据瓶颈所处层级选择优化的方法。

调优分析思路如下：

1）对于服务端的问题，需要重点定位的硬件指标包括CPU、内存、硬盘、BIOS配置，利用监测得到的数据进行性能分析，查看性能瓶颈点。
2）对于网络问题，网卡中断绑核对于性能提升增益可观。
3）具体而言，在Hive优化时，主要是两方面影响了性能，一个是查询计划，一个是Tez的任务执行。
Hive调优需要看HiveServer的运行日志及GC日志。
HiveServer日志路径为：HiveServer节点的“/var/log/hive”。

主要分为以下过程：

a.分析SQL待处理的表及文件。统计待处理的表的文件数、数据量、条数、文件格式、压缩格式，分析是否有更适合的文件存储格式、压缩格式，是否能够使用分区或分桶，是否有大量小文件需要map前合并。如果有优化空间，则执行优化。
1.如何判断是否有更合适的文件存储格式
当前TPC-DS测试，使用的文件存储格式为orc，当前测试性能最优，推荐使用该存储格式。

2.如何判断是否有更合适的压缩格式
当前TPC-DS测试，在map/reduce阶段，使用的数据压缩格式为SNAPPY，当前测试性能最优，推荐使用该压缩格式。

3.如何判断是否能够使用分区或分桶
庞大的数据集可能需要耗费大量的时间去处理。在许多场景下，可以通过分区或切片的方法减少每一次扫描总数据量，这种做法可以显著地改善性能。Hive的分区使用HDFS的子目录功能实现。每一个子目录包含了分区对应的列名和每一列的值。但是由于HDFS并不支持大量的子目录，这也给分区的使用带来了限制。我们有必要对表中的分区数量进行预估，从而避免因为分区数量过大带来一系列问题。Hive查询通常使用分区的列作为查询条件。这样的做法可以指定MapReduce任务在HDFS中指定的子目录下完成扫描的工作。HDFS的文件目录结构可以像索引一样高效利用。

如：
```
CREATE TABLE logs(
timestamp BIGINT,
line STRING
)
PARTITIONED BY (date STRING,country STRING);
```
PARTITONED BY子句中定义的列是表中正式的列（分区列），但是数据文件内并不包含这些列。

Hive还可以把表或分区，组织成桶。将表或分区组织成桶有以下几个目的：
第一个目的是为看取样更高效，因为在处理大规模的数据集时，在开发、测试阶段将所有的数据全部处理一遍可能不太现实，这时取样就必不可少。
第二个目的是为了获得更好的查询处理效率。

桶为表提供了额外的结构，Hive在处理某些查询时利用这个结构，能够有效地提高查询效率。
桶是通过对指定列进行哈希计算来实现的，通过哈希值将一个列名下的数据切分为一组桶，并使每个桶对应于该列名下的一个存储文件。
在建立桶之前，需要设置hive.enforce.bucketing属性为true，使得Hive能识别桶。

以下为创建带有桶的表的语句：
```
CREATE TABLE bucketed_user(
id INT,
name String
)
CLUSTERED BY (id) INTO 4 BUCKETS;
```
分区中的数据可以被进一步拆分成桶，bucket，不同于分区对列直接进行拆分，桶往往使用列的哈希值进行数据采样。

在分区数量过于庞大以至于可能导致文件系统崩溃时，建议使用桶。
4.如何判断是否有大量小文件需要map前合并。
查看每张表的下文件的大小，如果小文件（文件大小小于blocksize）很多，那么建议在map前将小文件进行合并，合并语句如下。
```
ALTER TABLE $table_name CONCATENATE.
```
b.根据nmon抓取的数据分析性能瓶颈，根据CPU以及内存的使用情况，修改hive.tez.container.size等参数。

### 2.硬件调优
#### 2.1 配置BIOS
 **目的** 
对于不同的硬件设备，通过在BIOS中设置一些高级选项，可以有效提升服务器性能。

1）服务器上的SMMU一般用来完成设备的地址转换，并且可以实现设备隔离，在虚拟化中很实用，但是在物理机测试场景下，SMMU可能会导致性能下降，尤其对于小包网络场景，因此建议关闭该功能提升服务器性能。在虚拟机场景需要打开此配置来使用PCI直通功能。
2）在本测试场景中，预取会导致cache污染，cache miss增加，因此建议关闭预取功能。

 **方法** 
1.关闭SMMU。
1）重启服务器，按Esc键进入BIOS设置界面。
2）依次进入“Advanced > MISC Config > Support Stmmu”。
3）将“Support Smmu”设置为“Disabled”，按“F10”保存退出（永久有效）。

2.关闭预取。
1）进入BIOS设置界面。
2）依次进入“Advanced > MISC Config > CPU Prefetching Configuration”。
3）将“CPU Prefetching Configuration”设置为“Disabled”，按“F10”保存退出（永久有效）。
#### 2.2 创建RAID0
 **目的** 
磁盘创建RAID可以提高磁盘的整体存取性能，环境上若使用的是LSI 3108/3408/3508系列RAID卡，推荐创建单盘RAID0/RAID5，充分利用RAID卡的Cache（LSI 3508卡为2GB），提高读写速率。
 **方法** 
1.使用storcli64_arm文件检查RAID组创建情况。
```
./storcli64_arm /c0 show
```
2.创建RAID 0，命令表示为第2块1.2T硬盘创建RAID 0，其中，c0表示RAID卡所在ID、r0表示组RAID 0。依次类推，对除了系统盘之外的所有盘做此操作。
```
./storcli64_arm /c0 add vd r0 drives=65:1
```
![raid0硬盘信息](https://foruda.gitee.com/images/1679296692757699634/3edc3db6_12582270.png "raid0硬盘信息")
#### 2.3 开启RAID卡Cache
 **目的** 
使用RAID卡的RAWBC（无超级电容）/RWBC（有超级电容）性能更佳。

使用storcli工具修改RAID组的Cache设置，Read Policy设置为Read ahead，使用RAID卡的Cache做预读；Write Policy设置为Write back，使用RAID卡的Cache做回写，而不是Write through透写；IO policy设置为Cached IO，使用RAID卡的Cache缓存IO。
 **方法** 
1.配置RAID卡的cache。
```
./storcli64_arm /c0/vx set rdcache=RA
./storcli64_arm /c0/vx set wrcache=WB/AWB
./storcli64_arm /c0/vx set iopolicy=Cached
```
2.检查配置是否生效。
```
./storcli64_arm /c0 show
```
#### 2.4 调整rx_buff
 **目的** 
1822网卡默认的rx_buff配置为2KB，在聚合64KB报文的时候需要多片不连续的内存，使用率较低；该参数可以配置为2/4/8/16KB，修改后可以减少不连续的内存，提高内存使用率。
 **方法** 
1.查看rx_buff参数值，默认为2。
```
cat /sys/bus/pci/drivers/hinic/module/parameters/rx_buff
```
2.在目录“/etc/modprobe.d/”中增加文件hinic.conf，修改rx_buff值为8。
```
options hinic rx_buff=8
```
3.重新挂载hinic驱动，使得新参数生效。
```
rmmod hinic
modprobe hinic
```
4.查看rx_buff参数是否更新成功。
```
cat /sys/bus/pci/drivers/hinic/module/parameters/rx_buff
```
#### 2.5 调整Ring Buffer
 **目的** 
1822网卡Ring Buffer最大支持4096，默认配置为1024，可以增大buffer大小用于提高网卡Ring大小。
 **方法** 
1.查看Ring Buffer默认大小，假设当前网卡名为enp131s0。
```
ethtool -g enp131s0
```
![Ring Buffer](https://foruda.gitee.com/images/1679297097168932888/086eddb2_12582270.png "Ring Buffer信息")
2.修改Ring Buffer值为4096。
```
ethtool -G enp131s0 rx 4096 tx 4096
```
3.查看Ring Buffer值是否更新成功。
```
ethtool -g enp131s0
```
!Ring Buffer值](https://foruda.gitee.com/images/1679297151878395704/1b4e76b5_12582270.png "Ring Buffer值信息")
#### 2.6 开启LRO
 **目的** 
1822网卡支持LRO（Large Receive Offload），可以打开该选项，并合理配置1822 LRO参数。
 **方法** 
1.查看LRO参数是否开启，默认为off，假设当前网卡名为enp131s0。
```
ethtool -k enp131s0
```
![LRO参数信息](https://foruda.gitee.com/images/1679297290754554842/dea3f5bb_12582270.png "LRO参数信息")
2.打开LRO。
```
ethtool -K enp131s0 lro on
```
3.查看LRO参数是否开启。
```
ethtool -k enp131s0
```
![LRO参数信息](https://foruda.gitee.com/images/1679297341729933809/2f81569d_12582270.png "LRO参数信息")
#### 2.7 配置网卡中断绑核
 **目的** 
相比使用内核的irqbalance使网卡中断在所有核上进行调度，使用手动绑核将中断固定住能有效提高业务网络收发包的能力。
 **方法** 
1.关闭irqbalance。
若要对网卡进行绑核操作，则需要关闭irqbalance。
a.停止irqbalance服务，重启失效。
```
systemctl stop irqbalance.service
```
b.关闭irqbalance服务，永久有效。
```
systemctl disable irqbalance.service
```
c.查看irqbalance服务状态是否已关闭。
```
systemctl status irqbalance.service
```
2.查看网卡pci设备号，假设当前网卡名为enp131s0。
```
ethtool -i enp131s0
```
![网卡pci设备号](https://foruda.gitee.com/images/1679297453531256848/9c085b58_12582270.png "网卡pci设备号")
3.查看pcie网卡所属NUMA node。
```
lspci -vvvs <bus-info>
```
![pcie网卡所属NUMA node](https://foruda.gitee.com/images/1679297480360574839/d79861bd_12582270.png "pcie网卡所属NUMA node")
4.查看NUMA node对应的core的区间，例如此处就可以绑到48~63。
```
lscpu
```
![cpu](https://foruda.gitee.com/images/1679297507730165344/9160f1e3_12582270.png "cpu")
5.进行中断绑核，1822网卡共有16个队列，将这些中断逐个绑至所在NumaNode的16个Core上（例如此处就是绑到NUMA node1对应的48-63上面）。
```
bash smartIrq.sh

脚本内容如下：
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
6.利用脚本查看是否绑核成功。
```
sh irqCheck.sh enp131s0

脚本内容如下：
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
![绑核信息](https://foruda.gitee.com/images/1679297565768443806/03c2f828_12582270.png "绑核信息")
#### 2.8 磁盘配置
 **磁盘规划** 
在Hadoop类环境中，基本为单盘做数据盘，因此存在两种选项——JBOD和单盘RAID 0。

环境上若使用的是LSI 3108/3408/3508系列RAID卡，推荐组建单盘RAID 0，充分利用RAID卡的Cache（LSI 3508卡为2GB），提高读写速率。
 **Cache策略** 
使用storcli工具修改RAID组的Cache设置，Read Policy设置为Read ahead，使用RAID卡的Cache做预读；Write Policy设置为Write back，使用RAID卡的Cache做回写，而不是Write through透写；IO policy设置为Cached IO，使用RAID卡的Cache缓存IO。

在实测中，RAWBD与RAWBC之间会有3~5%的端到端性能差异，使用RAWBC（无超级电容）/RWBC（有超级电容）性能更佳。

配置命令如下：
```
./storcli /c0/vx set rdcache=RA
./storcli /c0/vx set wrcache=WB/AWB
./storcli /c0/vx set iopolicy=Cached
```
 **块设备配置** 
主要涉及scheduler、sector_size、read_ahead_kb配置。

根据实测，scheduler应使用deadline性能最优。
```
cat /sys/block/sdx/queue/scheduler
```
![scheduler信息](https://foruda.gitee.com/images/1679297834015460513/293de6c1_12582270.png "scheduler信息")

块设备的sector_size应与物理盘的扇区大小进行匹配。可通过hw_sector_size、max_hw_sectors_kb、max_sectors_kb三者进行匹配，前两者是从硬件中读取出来的值，第三者是内核块设备的聚合最大块大小，推荐与硬件保持一致，即后两者参数保证一致。
a.查看三者参数。
```
cat /sys/block/sdx/queue/hw_sector_size
cat /sys/block/sdx/queue/max_hw_sectors_kb
cat /sys/block/sdx/queue/max_sectors_kb
```
![输入图片说明](https://foruda.gitee.com/images/1679297894129006604/c5acf731_12582270.png "屏幕截图")
b.修改参数并查看是否修改成功。
![输入图片说明](https://foruda.gitee.com/images/1679297915770767981/2f1a6577_12582270.png "屏幕截图")

块设备的预读推荐设置为4M，读性能更佳，默认值一般为128KB。
a.查看预读设置值
```
/sbin/blockdev --getra /dev/sdb
```
![输入图片说明](https://foruda.gitee.com/images/1679297949082595011/86046816_12582270.png "屏幕截图")
b.设置并检验。
```
/sbin/blockdev --setra 4096 /dev/sdb
```
![输入图片说明](https://foruda.gitee.com/images/1679297976912084956/b4f830e3_12582270.png "屏幕截图")

### 3.操作系统优化
#### 3.1 内存参数配置
 **目的** 
通过关闭内存大页、配置脏页参数等方法提升整体性能。
 **方法** 
1.关闭内存大页。
```
echo never > /sys/kernel/mm/transparent_hugepage/enabled
echo never > /sys/kernel/mm/transparent_hugepage/defrag
```
2.配置脏页参数。
```
echo 500 > /proc/sys/vm/dirty_expire_centisecs
echo 100 > /proc/sys/vm/dirty_writeback_centisecs
```
3.其他
```
echo 1 > /proc/sys/kernel/numa_balancing
echo 0 > /proc/sys/kernel/sched_autogroup_enabled
```
#### 3.2 调整IO调度策略
 **目的** 
对agent节点，将IO调度策略调整成mq-deadline，能有更高的IO效率。
 **方法** 
在agent节点上，把所有的HDD数据盘的IO策略都调整成mq-deadline。
```
list="a b c d e f g h i j k l"   # 按需修改
for i in $list
do
echo mq-deadline > /sys/block/sd$i/queue/scheduler
done
```
#### 3.3 关闭swap
 **目的** 
Linux的虚拟内存会根据系统负载自动调整，内存页（page）swap到磁盘会影响测试性能。
 **方法** 
```
swapoff -a
```
修改前后对比如下：
![输入图片说明](https://foruda.gitee.com/images/1679298358283896974/7a1db435_12582270.png "屏幕截图")
### 4.Hive调优
#### 4.1 组件参数配置
具体的各个组件的参数设置如表1所示。

表1 参数设置
| 组件                                   | 参数名                                        | 推荐值                                      | 修改原因                                                     |
| -------------------------------------- | --------------------------------------------- | ------------------------------------------- | ------------------------------------------------------------ |
| Yarn->NodeManagerYarn->ResourceManager | ResourceManager Java heap size                | 1024                                        | 修改JVM内存大小，保证内存水平较高，减少GC的频率。**说明：**非固定值，需要根据GC的释放情况来调大或调小Xms及Xmx的值。 |
| NodeManager Java heap size             | 1024                                          |                                             |                                                              |
| Yarn->NodeManager                      | yarn.nodemanager.resource.cpu-vcores          | 鲲鹏计算平台48核环境推荐值48。              | 可分配给Container的CPU核数。                                 |
| Yarn->NodeManager                      | yarn.nodemanager.resource.memory-mb           | 与实际数据节点物理内存总量相等。            | 可分配给Container的内存。                                    |
| Yarn->NodeManager                      | yarn.nodemanager.numa-awareness.enabled       | True                                        | NodeManager启动Container时的Numa感知。                       |
| Yarn->NodeManager                      | yarn.nodemanager.numa-awareness.read-topology | True                                        | NodeManager的Numa拓扑自动感知。                              |
| MapReduce2                             | mapreduce.map.memory.mb                       | 7168                                        | 一个 Map Task可使用的内存上限。                              |
| MapReduce2                             | mapreduce.reduce.memory.mb                    | 14336                                       | 一个 Reduce Task可使用的资源上限。                           |
| MapReduce2                             | mapreduce.job.reduce.slowstart.completedmaps  | 0.35                                        | 当Map完成的比例达到该值后才会为Reduce申请资源。              |
| HDFS->NameNode                         | NameNode Java heap size                       | 3072                                        | 修改JVM内存大小，保证内存水平较高，减少GC的频率。**说明：**非固定值，需要根据GC的释放情况来调大或调小Xms及Xmx的值。 |
| NameNode new generation size           | 384                                           |                                             |                                                              |
| NameNode maximum new generation size   | 384                                           |                                             |                                                              |
| HDFS->DataNode                         | dfs.datanode.handler.count                    | 512                                         | DataNode服务线程数，可适量增加。                             |
| HDFS->NameNode                         | dfs.namenode.service.handler.count            | 32                                          | NameNode RPC服务端监测DataNode和其他请求的线程数，可适量增加。 |
| HDFS->NameNode                         | dfs.namenode.handler.count                    | 1200                                        | NameNode RPC服务端监测客户端请求的线程数，可适量增加。       |
| TEZ                                    | tez.am.resource.memory.mb                     | 7168                                        | 等同于yarn.scheduler.minimum-allocation-mb，默认7168。       |
| TEZ                                    | tez.runtime.io.sort.mb                        | SQL1: 32SQL2: 256SQL3: 256SQL4: 128SQL5: 64 | 根据不同的场景进行调整。                                     |
| TEZ                                    | tez.am.container.reuse.enabled                | true                                        | Container重用开关。                                          |
| TEZ                                    | tez.runtime.unordered.output.buffer.size-mb   | 537                                         | 10%* hive.tez.container.size。                               |
| TEZ                                    | tez.am.resource.cpu.vcores                    | 10                                          | 使用的虚拟CPU数量，默认1，需要手动添加。                     |
| TEZ                                    | tez.container.max.java.heap.fraction          | 0.85                                        | 基于Yarn提供的内存，分配给java进程的百分比，默认是0.8，需要手动添加。 |

 **客户端配置**
表2 SQL1
| 参数名                                   | 参数配置 |
| ---------------------------------------- | -------- |
| hive.map.aggr                            | TRUE     |
| hive.vectorized.execution.enabled        | TRUE     |
| hive.auto.convert.join                   | TRUE     |
| hive.auto.convert.join.noconditionaltask | TRUE     |
| hive.limit.optimize.enable               | TRUE     |
| hive.exec.parallel                       | TRUE     |
| hive.cbo.enable                          | TRUE     |
| hive.exec.compress.intermediate          | TRUE     |
| hive.tez.container.size                  | 7941     |
| tez.runtime.io.sort.mb                   | 32       |

表3 SQL2
| 参数名                                   | 参数配置 |
| ---------------------------------------- | -------- |
| hive.map.aggr                            | TRUE     |
| hive.vectorized.execution.enabled        | TRUE     |
| hive.auto.convert.join                   | TRUE     |
| hive.auto.convert.join.noconditionaltask | TRUE     |
| hive.exec.parallel                       | TRUE     |
| hive.cbo.enable                          | TRUE     |
| hive.exec.compress.intermediate          | TRUE     |
| hive.exec.reducers.max                   | 576      |
| hive.tez.container.size                  | 5120     |
| tez.runtime.io.sort.mb                   | 256      |


表4 SQL3
| 参数名                                   | 参数配置 |
| ---------------------------------------- | -------- |
| hive.map.aggr                            | TRUE     |
| hive.vectorized.execution.enabled        | TRUE     |
| hive.auto.convert.join                   | TRUE     |
| hive.auto.convert.join.noconditionaltask | TRUE     |
| hive.limit.optimize.enable               | TRUE     |
| hive.exec.parallel                       | TRUE     |
| hive.exec.reducers.max                   | 376      |
| hive.tez.container.size                  | 9216     |
| tez.runtime.io.sort.mb                   | 256      |


表5 SQL4
| 参数名                                   | 参数配置 |
| ---------------------------------------- | -------- |
| hive.map.aggr                            | TRUE     |
| hive.vectorized.execution.enabled        | TRUE     |
| hive.auto.convert.join                   | TRUE     |
| hive.auto.convert.join.noconditionaltask | TRUE     |
| hive.limit.optimize.enable               | TRUE     |
| hive.exec.parallel                       | TRUE     |
| hive.cbo.enable                          | TRUE     |
| hive.tez.container.size                  | 11264    |
| tez.runtime.io.sort.mb                   | 128      |

表6 SQL5
| 参数名                                   | 参数配置 |
| ---------------------------------------- | -------- |
| hive.map.aggr                            | TRUE     |
| hive.vectorized.execution.enabled        | TRUE     |
| hive.auto.convert.join                   | TRUE     |
| hive.auto.convert.join.noconditionaltask | TRUE     |
| hive.limit.optimize.enable               | TRUE     |
| hive.exec.parallel                       | TRUE     |
| hive.cbo.enable                          | TRUE     |
| hive.tez.container.size                  | 15121    |
| tez.runtime.io.sort.mb                   | 64       |

#### 4.2 numa特性开启
Yarn组件在3.1.0版本合入的新特性支持，支持Yarn组件在启动Container时使能NUMA感知功能，原理是读取系统物理节点上每个NUMA节点的CPU核、内存容量，使用Numactl命令指定启动container的CPU范围和membind范围，减少跨片访问。

1.安装numactl（在server和agent上都安装）。
```
yum install numactl.aarch64 -y
```
2.开启NUMA感知（在组件参数配置中已经做过的可以不做）。
增加Yarn->NodeManager：
```
yarn.nodemanager.numa-awareness.enabled              true
yarn.nodemanager.numa-awareness.read-topology    true
```








