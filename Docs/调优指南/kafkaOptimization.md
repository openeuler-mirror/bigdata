# kafka 调优指南
## 1-环境介绍
### 1.1 软件版本信息
| 软件  | 版本  |
|---|---|
| OS  | openEuler 22.03 LTS  |
| kernel  | 4.19.90-2005.1.0.0038.oe1.aarch64  |
| JDK  | Openjdk version “1.8.0_272”  |
| Kafka | 2.11-2.0.0|
测试工具使用自带的Kafka的kafka-perf测试。
| 工具名称  | 工具版本  |
|---|---|
|kafka-perf|2.11-2.0.0|

### 1.2 调优原则
- 建议根据盘数设置parition的值，一般partition数量为所有数据节点盘数总量的1~2倍。
- 数据节点和Kafka测试客户端尽量分开，避免客户端生产数据对数据节点的影响。
- Kafka处理网络和IO的线程数可以尽量增大。
- 主机可以起多个线程启动Kafka的测试命令，且消费并发一般为生产的2~3倍左右，可根据实际场景调整；如在生产场景启动10个kafka-perf的线程（命令），消费场景启动30个线程（消费命令）消费数据。

### 1.3 调优思路
- 为了能够让磁盘性能达到最大，partitions的数目总数需要大于磁盘数目，这样可以让每个磁盘至少拥有1个partition，否则该盘可能未被利用。
- Kafka的一个重要特点就是会对收到的消息都保存到磁盘，而不是只存在于内存中。但是磁盘的读写速率较慢会对数据的实时性产生一定影响，所以Kafka一般会先使用内存cache保存数据，然后再使用异步线程进行刷盘操作。这样极大的提高了Kafka持久化数据的速度，保证了实时性。Kafka主要为IO型组件，在不启用压缩算法时，对CPU的消耗非常低；但如果使用压缩算法，特别是在消费场景会占用大量CPU（由于压缩算法特性，往往解压速度较快，所以可以令消费吞吐量发生质的提升）。
- 由于集群部署以及多副本的原因，Kafka的数据要写入到各个节点前，数据都是通过网络来传输的，所以Kafka对网络的要求非常高，从一定程度上来讲，Kafka多少吞吐量，就需要多少网络带宽。虽然Kafka是一个只将数据保存在磁盘中的组件，但事实上，Kafka对硬盘的依赖反而并没有这么高。Kafka的性能指标时延与吞吐量，其中时延是由系统缓存来保证的，kafka在生产数据时，都只将数据写在缓存中，而由异步线程从缓存中同步到硬盘，所以Kafka会要求硬盘读写总带宽有一定的程度（可通过累加磁盘数目解决），但对硬盘的读写速率要求并不高。
- 生产测试时，可以在每台客户端上启动多个kafka-producer-perf-test.sh进行生产，生产的消息为同一个topic的，生产时间180秒，到时间后停止所有生产进程。
- 消费测试时，可以在每台客户端上启动多个kafka-consumer-perf-test.sh对生产测试的topic进行消费，消费时所有进程采用相同的group，将消息消费完所需时间与生产时间基本一致也为180秒。
## 2-硬件调优
### 2.1 配置BIOS
- 目的

对于不同的硬件设备，通过在BIOS中设置一些高级选项，可以有效提升服务器性能。

服务器上的SMMU一般用来完成设备的地址转换，并且可以实现设备隔离，在虚拟化中很实用，但是在物理机测试场景下，SMMU可能会导致性能下降，尤其对于小包网络场景，因此建议关闭该功能提升服务器性能。在虚拟机场景需要打开此配置来使用PCI直通功能。
在本测试场景中，预取会导致cache污染，cache miss增加，因此建议关闭预取功能。

- 方法

关闭SMMU。
重启服务器，按Esc键进入BIOS设置界面。
依次进入“Advanced > MISC Config > Support Stmmu”。
将“Support Smmu”设置为“Disabled”，按“F10”保存退出（永久有效）。
关闭预取。
进入BIOS设置界面。
依次进入“Advanced > MISC Config > CPU Prefetching Configuration”。
将“CPU Prefetching Configuration”设置为“Disabled”，按“F10”保存退出（永久有效）。
### 2.2 创建RAID 0
```
- 目的
磁盘创建RAID可以提高磁盘的整体存取性能，环境上若使用的是LSI 3108/3408/3508系列RAID卡，推荐创建单盘RAID 0/RAID 5，充分利用RAID卡的Cache（LSI 3508卡为2GB），提高读写速率。
- 方法
使用storcli64_arm文件检查RAID组创建情况。
./storcli64_arm /c0 show
创建RAID 0，命令表示为第2块1.2TB硬盘创建RAID 0，其中，c0表示RAID卡所在ID、r0表示组RAID 0。依次类推，对除了系统盘之外的所有盘做此操作。
./storcli64_arm /c0 add vd r0 drives=65:1
```
![输入图片说明](https://foruda.gitee.com/images/1679292452664538651/90ba8b2b_10227919.png "屏幕截图")
### 2.3 开启RAID卡Cache
```
- 目的
使用RAID卡的RWTD性能更佳。

使用storcli工具修改RAID组的Cache设置，Read Policy设置为Read ahead，使用RAID卡的Cache做预读；Write Policy设置为Write Through；IO policy设置为Direct IO。

- 方法
配置RAID卡的cache，其中vx是卷组号（v0/v1/v2....），根据环境上的实际情况进行设置。
./storcli64_arm /c0/vx set rdcache=RA
./storcli64_arm /c0/vx set wrcache=WT
./storcli64_arm /c0/vx set iopolicy=Direct
检查配置是否生效。
./storcli64_arm /c0 show
```
![输入图片说明](https://foruda.gitee.com/images/1679292548783445749/728f38d6_10227919.png "屏幕截图")

### 2.4 调整rx_buff
```
目的
1822网卡默认的rx_buff配置为2KB，在聚合64KB报文的时候需要多片不连续的内存，使用率较低；该参数可以配置为2/4/8/16KB，修改后可以减少不连续的内存，提高内存使用率。
方法
查看rx_buff参数值，默认为2。
cat /sys/bus/pci/drivers/hinic/module/parameters/rx_buff
在目录“/etc/modprobe.d/”中增加文件hinic.conf，修改rx_buff值为8。
options hinic rx_buff=8
重新挂载hinic驱动，使得新参数生效。
rmmod hinic
modprobe hinic
查看rx_buff参数是否更新成功。
cat /sys/bus/pci/drivers/hinic/module/parameters/rx_buff
```
### 2.5 调整队列深度和队列数
```
目的 ：1822网卡队列深度最大支持4096，默认配置为1024，可以增大buffer大小用于提高网卡Ring大小。
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
### 2.6 开启LRO
```
目的
1822网卡支持LRO（Large Receive Offload），可以打开该选项，并合理配置1822 LRO参数。

方法
查看LRO参数是否开启，默认为off，假设当前网卡名为enp131s0。
ethtool -k enp131s0
打开LRO。
ethtool -K enp131s0 lro on
查看LRO参数是否开启。
ethtool -k enp131s0
```
### 2.7 配置网卡中断绑核
```
目的：相比使用内核的irqbalance使网卡中断在所有核上进行调度，使用手动绑核将中断固定住能有效提高业务网络收发包的能力。

方法
关闭irqbalance。
若要对网卡进行绑核操作，则需要关闭irqbalance。

- 停止irqbalance服务，重启失效。
systemctl stop irqbalance.service
- 关闭irqbalance服务，永久有效。
systemctl disable irqbalance.service
- 查看irqbalance服务状态是否已关闭。
systemctl status irqbalance.service
- 查看网卡pci设备号，假设当前网卡名为enp131s0。
ethtool -i enp131s0
- 查看pcie网卡所属NUMA node。
lspci -vvvs <bus-info>
- 进行中断绑核，1822网卡共有16个队列，将这些中断逐个绑至所在NumaNode的16个Core上（例如此处就是绑到NUMA node1对应的48-63上面）。
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
- 利用脚本查看是否绑核成功。
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
## 3-操作系统调优
目的
修改脏页刷新参数，提高脏页后台刷新的频率。

方法
修改dirty_writeback_centisecs。
echo 5 > /proc/sys/vm/dirty_writeback_centisecs
## 4-Kafka参数配置
| 参数  | 建议值   |  描述 |
|---|---|---|
|num.network.threads|128|broker处理消息的最大线程数，主要处理网络IO，读写缓冲区数据。由于当前Kafka主要是网络瓶颈，所以该参数影响特别大。该参数默认值为3，此时对Kafka性能有决定性影响，当不断调大此参数时，性能会有比较大的提升。该参数的建议值是核数+1，但实际由于网卡瓶颈，该参数调到一定程度后对性能几乎没有影响。为了方便调试，我们可以直接将该参数定为核数+1甚至定为最大值（128）。|
|num.io.threads|65|broker处理磁盘IO的线程数，由于目前Kafka是网络瓶颈，所以磁盘IO的线程数影响并不是很大。目前典型场景单broker部署了23个Kafka盘，均为SAS盘，单盘性能较强。实际3~4个盘，就可以将网络IO跑到瓶颈，所以理论上IO线程数的修改对性能影响非常有限。该参数默认值为8，最高可以调到256。|
|compression.type|根据实际场景调整。|Kafka压缩算法选择。可以配置为producer（由client生产者决定）、None（不压缩）、gzip、snappy、lz4。实测几种压缩算法lz4与与snappy表现较好，gzip尽量不选择。但在大多数场景下，还是不压缩更加有优势。压缩算法有时候是双刃剑，并不一定能带来比较好的收益，需要在系统场景中综合考虑使用。|
|partitions|根据磁盘数量调整，一般为磁盘的1~2倍。|Kafka topic的分区数，该参数与topic强相关，即Kafka数据保存的分区数目，一般要比盘总数大，以保证每个盘都有IO读写。但实际情况由于硬盘IO没有瓶颈，所以可以综合情况进行考虑。|
## 5-Client参数配置
目的：修改Client参数增大测试压力，Client参数主要为kafka-perf工具测试生产和消费场景时的设置参数，主要包括副本数、数据块大小、消息数量等。
| 参数  | 建议值   |  描述 |
|---|---|---|
|replication-factor|2|生产数据的副本数，为了保证数据的可靠性，副本数至少为2，该参数是1是很危险的操作。在性能测试的时候，副本数一般就取2。|
|record-size|1024-65535|数据块大小。实际场景的数据块大小是由具体业务所决定的，一般常见业务的数据块大小在1024~65535之间，一般做性能测试会从这个取值范围内选取至少5个数据块大小进行测试。不同数据块大小的生产与消费性能会有所区别（实际和网络大小包性能有关）。|
|num-fetch-threads|默认|拉取数据的线程数量，即为消费者的数量。在网络瓶颈下，调整此参数基本没有效果。|
|threads|默认|消费处理线程数。在网络瓶颈下，调整此参数基本没有效果。|
|messages|800000000|消费消息的条数，由于Kafka在进行消费时，性能会有所起伏与波动，所以在进行性能测试时，需要对不同消息条数进行测试，得到性能最优值。一般该值取8亿左右。|
|request.required.acks|0|发消息后，是否需要broker端返回。默认需要返回，可以尝试设置为0，在数据块大小较小，发送比较频繁的场景下可能有一定效果。|
|linger.ms|0|多条消息聚合的延时，默认为0，一般保持默认值即可。|




