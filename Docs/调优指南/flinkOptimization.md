# 1 调优概述
## 1.1 flink介绍
Flink是一个批处理和流处理结合的统一计算框架，其核心是一个提供了数据分发以及并行化计算的流数据处理引擎。它的最大亮点是流处理，是业界最顶级的开源流处理引擎。
## 1.2 环境介绍
| 软件                            | 版本                        |
|-------------------------------|---------------------------|
| OS                            | openEuler 22.03 |
| JDK                           | OpenJDK 1.8.0_262         |
| Kafka                         | 2.11-1.1.0                |
| Flink                         | 1.7.2                     |
| Redis                         | 3.0.7                     |
| yahoo-streaming-benchmark测试工具 | 0.2.0（基于开源适配修改）           |
## 1.3 调优原则
1. Flink的yahoo-streaming-benchmark测试涉及Kafka、Flink、Redis3个组件，Flink测试的时候同时需要调整Kafka的参数，但测试主体是Flink，Kafka可以不调整到Kafka最优的情况，把CPU尽量分给Flink使用。
2. Flink在网络、磁盘等各项资源均未达到瓶颈的情况下，吞吐量也会达到瓶颈，且吞吐量和测试数据量之间存在正比关系。可以先测试最大数据量，初步估计当前环境的吞吐量瓶颈，再改数据量上下调整，获得较好的性能结果。
3. 调优主要参数有：
```
a. Kafka、Redis、Zookeeper所在节点IP、端口。
b. Topic：即Kafka TOPIC名称，每次运行修改名称即可。
c. PARTITIONS：即Kafka分区数，比较关键的参数，分区数与Flink的并发度必须保持一致，所以调试Flink性能主要需要修改该参数。
d. LOAD：数据量，该值直接决定了吞吐量大小，调整该值可以测出一定时延下最大可达到的吞吐量值。
e. TEST_TIME：测试时间，一般典型值为240s。
f. -p：Flink run时附带的参数，即并发度，必须小于等于PARTITIONS，一般两者取值相等。
这些参数在配置完成后，大部分会写入到local_conf.yaml配置文件中，供Flink执行时读取。
```
## 1.4 调优思路
Flink作为一个流处理框架，主要作用是处理数据，将数据通过一定的方法计算、统计、分析，最终输出。典型的数据处理场景，即为Kafka、Flink、Redis三个组件共同使用，数据从Kafka获取，Flink进行计算分析，最终将结果写入Redis。

流处理框架，有一定的编程规范，主要思路如图所示。
![图1 流处理框架](https://www.hikunpeng.com/doc_center/source/zh/kunpengbds/ecosystemEnable/Flink/zh-cn_image_0000001089149901.png)

每个Flink程序都可以以类似的方式编写，主要思路即为先定义Flink流处理过程，每个步骤输入输出对齐，最终把数据输出。整个工作流定义完成后，就可以开始执行了，整个执行过程是无限流，永不停止，直到Flink任务被外部kill为止。

yahoo-streaming-benchmark工具主要用途是测试集群Flink处理性能，其测试过程从数据生成->数据写入Kafka->数据从Kafka读出->预处理数据->窗口处理数据->结果写入Redis，端到端的测试Flink流处理过程。主要性能指标为数据吞吐量和处理时延。
![输入图片说明](https://www.hikunpeng.com/doc_center/source/zh/kunpengbds/ecosystemEnable/Flink/zh-cn_image_0000001089006855.png)

数据从Flink其中一个进程生成，不断写入Kafka，由另一个进程读出数据进行处理，最终写入Redis，在Flink job执行完成后，Redis进行结果统计分析。
# 2 硬件调优
## 2.1 配置BIOS
### 目的
对于不同的硬件设备，通过在BIOS中设置一些高级选项，可以有效提升服务器性能。

- 服务器上的SMMU一般用来完成设备的地址转换，并且可以实现设备隔离，在虚拟化中很实用，但是在物理机测试场景下，SMMU可能会导致性能下降，尤其对于小包网络场景，因此建议关闭该功能提升服务器性能。在虚拟机场景需要打开此配置来使用PCI直通功能。
- 在本测试场景中，预取会导致Cache污染，Cache miss增加，因此建议关闭预取功能。
### 方法
#### 1) 关闭SMMU。
- a.重启服务器，进入BIOS设置界面。
具体操作请参见《TaiShan 服务器 BIOS 参数参考（鲲鹏920处理器）》中“进入BIOS界面”的相关内容。
- b.依次进入“Advanced > MISC Config > Support Stmmu”。
![BIOS设置界面](https://www.hikunpeng.com/doc_center/source/zh/kunpengbds/ecosystemEnable/Flink/zh-cn_image_0226825857.png)
- c.将“Support Smmu”设置为“Disabled”，按“F10”保存退出（永久有效）。
![关闭SMMU BIOS界面](https://www.hikunpeng.com/doc_center/source/zh/kunpengbds/ecosystemEnable/Flink/zh-cn_image_0226825858.png)
#### 2) 关闭预取。
- a.进入BIOS设置界面。
- b.依次进入“Advanced > MISC Config > CPU Prefetching Configuration”。
![BIOS设置界面](https://www.hikunpeng.com/doc_center/source/zh/kunpengbds/ecosystemEnable/Flink/zh-cn_image_0226825862.png)
- c.将“CPU Prefetching Configuration”设置为“Disabled”，按“F10”保存退出（永久有效）。
![关闭Prefetching Configuration界面](https://www.hikunpeng.com/doc_center/source/zh/kunpengbds/ecosystemEnable/Flink/zh-cn_image_0226825863.png)
## 2.2 调整rx_buff
### 目的
1822网卡默认的rx_buff配置为2KB，在聚合64KB报文的时候需要多片不连续的内存，使用率较低；该参数可以配置为2/4/8/16KB，修改后可以减少不连续的内存，提高内存使用率。
### 方法
- 1.查看rx_buff参数值，默认为2。

```
cat /sys/bus/pci/drivers/hinic/module/parameters/rx_buff
```
- 2.在目录“/etc/modprobe.d/”中增加文件hinic.conf，修改rx_buff值为8。

```
options hinic rx_buff=8
```
- 3.重新挂载hinic驱动，使得新参数生效。

```
rmmod hinic
modprobe hinic
```
- 4.查看rx_buff参数是否更新成功。

```
cat /sys/bus/pci/drivers/hinic/module/parameters/rx_buff
```
## 2.3 调整Ring Buffer
### 目的
1822网卡Ring Buffer最大支持4096，默认配置为1024，可以增大buffer大小用于提高网卡Ring大小。
### 方法
- 1.查看Ring Buffer默认大小，假设当前网卡名为enp131s0。

```
ethtool -g enp131s0
```
![输入图片说明](https://www.hikunpeng.com/doc_center/source/zh/kunpengbds/ecosystemEnable/Flink/zh-cn_image_0231041158.png)

- 2.修改Ring Buffer值为4096。

```
ethtool -G enp131s0 rx 4096 tx 4096
```
- 3.查看Ring Buffer值是否更新成功。

```
ethtool -g enp131s0
```
![输入图片说明](https://www.hikunpeng.com/doc_center/source/zh/kunpengbds/ecosystemEnable/Flink/zh-cn_image_0231041266.png)
## 2.4 开启LRO
### 目的
1822网卡支持LRO（Large Receive Offload），可以打开该选项，并合理配置1822 LRO参数。
### 方法
- 1.查看LRO参数是否开启，默认为off，假设当前网卡名为enp131s0。

```
ethtool -k enp131s0
```
![输入图片说明](https://www.hikunpeng.com/doc_center/source/zh/kunpengbds/ecosystemEnable/Flink/zh-cn_image_0231038281.png)
- 2.打开LRO。

```
ethtool -K enp131s0 lro on
```
- 3.查看LRO参数是否开启。

```
ethtool -k enp131s0
```
![输入图片说明](https://www.hikunpeng.com/doc_center/source/zh/kunpengbds/ecosystemEnable/Flink/zh-cn_image_0231039073.png)
## 2.5 配置网卡中断绑核
### 目的
相比使用内核的irqbalance使网卡中断在所有核上进行调度，使用手动绑核将中断固定住能有效提高业务网络收发包的能力。

### 方法
- 1.关闭irqbalance。
```
- a. 若要对网卡进行绑核操作，则需要关闭irqbalance。
systemctl stop irqbalance.service
- b. 关闭irqbalance服务，永久有效。
systemctl disable irqbalance.service
- c. 查看irqbalance服务状态是否已关闭。
systemctl status irqbalance.service
```
- 2.查看网卡PCI设备号，假设当前网卡名为enp131s0。

```
ethtool -i enp131s0
```
![输入图片说明](https://www.hikunpeng.com/doc_center/source/zh/kunpengbds/ecosystemEnable/Flink/zh-cn_image_0231023410.png)
- 3.查看PCIe网卡所属NUMA node。
```
lspci -vvvs <bus-info>
```
![输入图片说明](https://www.hikunpeng.com/doc_center/source/zh/kunpengbds/ecosystemEnable/Flink/zh-cn_image_0231022972.png)
- 4.查看NUMA node对应的core的区间，例如此处就可以绑到48~63。

```
lscpu
```
![输入图片说明](https://www.hikunpeng.com/doc_center/source/zh/kunpengbds/ecosystemEnable/Flink/zh-cn_image_0231023247.png)
- 5.进行中断绑核，1822网卡共有16个队列，将这些中断逐个绑至所在NumaNode的16个Core上（例如此处就是绑到NUMA node1对应的48-63上面）。

```
bash smartIrq.sh
```
脚本内容如下：
```
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
- 6.利用脚本查看是否绑核成功。

```
sh irqCheck.sh enp131s0
```
![输入图片说明](https://www.hikunpeng.com/doc_center/source/zh/kunpengbds/ecosystemEnable/Flink/zh-cn_image_0303198443.png)

脚本内容如下：
```
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
## 2.6 创建RAID0
### 目的
磁盘创建RAID可以提高磁盘的整体存取性能，环境上若使用的是LSI 3108/3408/3508系列RAID卡，推荐创建单盘RAID 0/RAID 5，充分利用RAID卡的Cache（LSI 3508卡为2GB），提高读写速率。
### 方法
- 1.使用storcli64_arm文件检查RAID组创建情况。

```
./storcli64_arm /c0 show
```
- 2.创建RAID 0，命令表示为第2块1.2T硬盘创建RAID 0，其中，c0表示RAID卡所在ID、r0表示组RAID 0。依次类推，对除了系统盘之外的所有盘做此操作。

```
./storcli64_arm /c0 add vd r0 drives=65:1
```
![输入图片说明](https://www.hikunpeng.com/doc_center/source/zh/kunpengbds/ecosystemEnable/Flink/zh-cn_image_0226841019.png)
 
## 2.7 开启RAID卡Cache
### 目的
使用RAID卡的RAWBC（无超级电容）/RWBC（有超级电容）性能更佳。

使用storcli工具修改RAID组的Cache设置，Read Policy设置为Read ahead，使用RAID卡的Cache做预读；Write Policy设置为Write back，使用RAID卡的Cache做回写，而不是Write through透写；IO policy设置为Cached IO，使用RAID卡的Cache缓存IO。
### 方法
- 1.配置RAID卡的Cache。

```
./storcli64_arm /c0/vx set rdcache=RA
./storcli64_arm /c0/vx set wrcache=WB/AWB
./storcli64_arm /c0/vx set iopolicy=Cached
```
- 2.检查配置是否生效。

```
./storcli64_arm /c0 show
```
# 3 flink 调优 
## 3.1 Flink参数修改
### 目的
修改Flink参数保证资源可最大限度供Flink使用。

### 方法
修改Flink根目录下的conf目录中的flink-conf.yaml文件。
| 参数                       | 建议值     | 描述                                                                                                                                                                                                                                                   |
|--------------------------|---------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| vcores                   | 根据实际调整。 | YARN container vcores数量，其中一个taskmanager即为一个container，该参数即指定了一个taskmanager可以使用的vcores数量。而jobmanager也是一个container，但是jobmanager固定分配1个vcore，并不受该参数影响。该参数对性能影响非常大，可以尽可能的调大，其中jobmanager必定会占用1个vcore，除去之后，剩下每个节点都可以使用该节点的核，所以可以根据taskmanager数目计算出vcores数目。 |
| network.numberofBuffers  | 根据实际调整。 | TaskManager网络传输缓冲栈数量 ，如果作业运行中出错提示系统中可用缓冲不足，可以增加这个配置项的值。                                                                                                                                                                                              |
| taskmanager.compute.numa | 详见参数配置。 | 该参数在默认Flink配置文件里没有，需要手动加入该参数。该参数的用途是开启taskmanager的numa绑核，必须与yarn的相关参数配合使用，具体使用方法请参见                                                                                                                                                                  |

## 3.2 Yarn/Kafka参数修改
### 目的
配置Yarn和Kafka相关参数，保证Kafka不会成为Flink调优瓶颈。
### 方法
在Flink服务的Web界面搜索以下参数，并根据实际情况进行修改。
| 参数                                            | 建议值             | 描述                                                                                                                                                                                                                         |
|-----------------------------------------------|-----------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| yarn.nodemanager.resource.cpu-vcores          | 每个Kafka服务节点的核数。 | 该参数在Web界面“yarn > CONFIGS > SETTINGS > Number of virtual cores”。一个nodemanager可以使用的cpu-vcores数目，直接配置成1个节点的核数即可。                                                                                                              |
| yarn.nodemanager.numa-awareness.enabled       | true            | 该参数在yarn配置中默认没有，需要手动添加，将其添加到Web界面“yarn > CONFIGS > ADVANCED > Nodemanager > Custom yarn > site > Add Property” 。该参数为是否使能container的numa感知功能，默认为false，如果需要开启，需要设置为true。                                                      |
| yarn.nodemanager.numa-awareness.read-topology | true            | 该参数在yarn配置中默认没有，需要手动添加，将其添加到Web界面“yarn > CONFIGS > ADVANCED > Nodemanager > Custom yarn > site > Add Property” 。该参数为是否从系统或配置中读取numa的topo结构。如果设置为true，那么会直接通过numactl –hardware命令从系统中读取系统的numa 结构。Flink测试需要开启numa绑核则设置为true。 |
| kafka.num.network.threads                     | 8               | 该参数在Web界面“Kafka > CONFIGS > Advanced kafka > broker > num.network.threads”。Kafka网络线程数，影响Kafka读写效率。但Flink处理资源有限，不建议将该值设置过大，否则Kafka占用资源影响Flink调优性能，该值可根据实际情况调整为默认值的1~2倍。                                                     |
## 3.3 Yarn提交Flink任务参数修改
### 目的
Flink应用运行前，需要先提交Flink任务，向Yarn申请相关内存CPU等资源，提交任务命令为：yarn-session.sh -n 4 -s 64 -jm 5000 -tm 50000 -d；修改提交任务参数，调整并发及分配资源参数。
### 方法
提交Yarn任务时，根据实际情况对以下参数进行调整：
| 参数                      | 建议值        | 描述                                                                                                                                                                                                                                                                                    |
|-------------------------|------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| -n (taskmanager)        | 节点数*（4-8）。 | 该参数为Flink的taskmanager数目，Flink引擎运行需要由一个jobmanager以及若干个taskmanager构成。每个taskmanager都是独立的一部分，当有Flink应用需要运行时，会被随机分配到一个taskmanager运行，所以taskmanager的数目增多，可以互不干扰的并行运行更多的Flink应用。但是很显然，集群中的资源时有限的，taskmanager越多，单个taskmanager能够分配的资源则越少，也会导致Flink性能下降。所以，根据经验值，taskmanager数目可以取集群节点数*4 ~节点数*8。 |
| -s (slot)               | 根据实际场景配置。  | 该参数为单个taskmanager所拥有的槽位数，Flink任务提交后，集群中可以使用的总槽位数即为slot * taskmanager。一个slot可以为一个taskmanager提供1个并发，例如slot=30，即1个taskmanager最多可以跑30并发，当然实际运行的时候，也可以只跑20并发，那么此时剩余10个slot即为空闲。剩余未使用的slot，并不会占用CPU资源，但是会占用相关内存资源。该参数的修改，根据实际需要用到的并发度而动态调节。在内存足够的情况下，可以适当设置较大。                              |
| -tm(taskmanager memory) | 30000      | 该参数为分配给单个taskmanager的内存资源。只要taskmanager内存足够使用，内存资源分配增多对性能也无直接提高。可以在jobmanager内存分配之后，先将所有剩余内存分配给taskmanager。根据经验值，分配30000MB左右即可。                                                                                                                                                       |
| -jm(jobmanager memory)  | 5000       | 该参数为分配给jobmanager的内存资源。该参数对整体性能几乎无影响，不需要分配太大。根据经验值，分配5000MB~15000MB即可。                                                                                                                                                                                                                |
## 3.4 Flink任务参数调整
### 目的
使用yahoo- streaming-benchmark工具测试时，测试脚本中也有相关Flink配置需要修改，这些配置都在stream-bench-hdp.sh里，需要对这些参数根据实际情况调整。
### 方法
在stream-bench-hdp.sh文件中搜索以下参数并修改：
| 参数         | 建议值                   | 描述                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
|------------|-----------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Partitions | Kafka服务节点上磁盘数目的2~3倍。  | 该值为Kafka的分区数目，但在修改该值的同时，必须同步修改Flink任务启动时的两个并发度值。这是因为前面那个Flink任务是Flink随机生成数据，并且写入Kafka中，所以Flink并发度必须<=kafka的partitions数目，不然将会无法写入。而后者是Flink消费Kafka数据，虽然并发度不受partitions数目限制，但尽量将Flink两个任务并发度保持一致进行测试。由于Flink在向Yarn提交任务的时候，已经将slot、内存、taskmanager等分配完毕，所以此时Flink能够利用的资源时有限的。而-p后的并发度参数，就是需要占用taskmanager slot的参数。该用例需要两个进程同时运行，所以partitions参数不能超过总slot的1/2。即partitions <= -n(taskmanager) * -s(slot) / 2, 并发度对Flink性能的影响是比较大的，而且也没有太清晰的规律。一般来说，随partitions增加，时延的变化是先降低，再到一个平台期，最后再上升。 |
| Load       | 根据实际调整。               | 数据量，该值唯一控制了单位时间的吞吐量，而当TEST_TIME固定的时候，设置LOAD即设置了该次用例运行的吞吐量（在Flink引擎足够处理的情况下）。所以Flink测试的性能数据有两种观察方式。  1是固定LOAD看时延大小。 2是固定时延看最大能将LOAD加到多大。 目前比较通用的都是方式2，即调整LOAD。                                                                                                                                                                                                                                                                                                                      |
| TOPIC      | ad-event{$partitions} | 该参数即为Kafka topic名称，不同的partition数目，必须修改不同的topic名称，一般采取ad-event{$partitions}的命名方式以避免重复。                                                                                                                                                                                                                                                                                                                                                                                              |
| TEST_TIME  | 240                   | 测试时间，一般都固定时间为240s。                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
# 调优实例
## 参数配置
### Web配置
| 组件                | 参数名                                           | 推荐值                                                                                                                                         | 修改原因                                                      |
|-------------------|-----------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------|
| Kafka->Broker     | log.dir                                       | /hadoop/data1/kafka-logs,  /hadoop/data2/kafka-logs,  /hadoop/data3/kafka-logs,  ...  /hadoop/data11/kafka-logs,  /hadoop/data12/kafka-logs | 配置Kafka使用盘的数量为12盘，防止硬盘读写成为瓶颈。                             |
| Kafka->Broker     | num.partitions                                | 36                                                                                                                                          | 使用更多的partition数量，配合log.dir参数提升Kafka性能。以及对应Flink数据生成器的并发度。 |
| Kafka->Broker     | num.network.threads                           | 128                                                                                                                                         | 增大Kafka网络线程数。                                             |
| Kafka->Broker     | num.io.threads                                | 8                                                                                                                                           | Flink数据量不大，保持IO线程数较小，减轻CPU压力。                             |
| Yarn->Nodemanager | yarn.nodemanager.resource.memory-mb           | 224256                                                                                                                                      | 增大Yarn每个nodemanager能分配的最大内存数。                             |
| Yarn->Nodemanager | yarn.nodemanager.resource. cpu-vcores         | 48                                                                                                                                          | 增大Yarn每个nodemanager能分配的最大vcores数。                         |
| Yarn->Nodemanager | yarn.nodemanager.numa-awareness.enabled       | true                                                                                                                                        | Yarn NUMA绑核开关开启。                                          |
| Yarn->Nodemanager | yarn.nodemanager.numa-awareness.read-topology | true                                                                                                                                        | 选择由参数进行绑核还是启动命令中进行绑核，Flink选择启动命令中绑核。                      |
| Yarn->Nodemanager | yarn.nodemanager.numa-awareness.numactl.cmd   | /usr/bin/numactl                                                                                                                            | NUMA绑核工具系统路径。                                             |









