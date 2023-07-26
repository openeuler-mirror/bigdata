# redis 调优指南
## 1-调优概述
### 1.1 环境介绍
  |软件        	|版本       |
  |-------------|-----------|                       
  |OS        	|CentOS 7.6、openEuler 22.03|
  |JDK       	|OpenJDK version 1.8.0_222|
  |Redis     	|3.x/4.x/5.x|              
  |Redis_tool	|自研工具|                     

### 1.2 调优原则
性能调优就是对计算机硬件、操作系统和应用有相当深入的了解，调节三者之间的关系，实现整个系统（包括硬件、操作系统、应用）的性能最大化，并能不断的满足现有的业务需求。在性能优化时，我们必须遵循一定的原则，否则，有可能得不到正确的调优结果。主要有以下几个方面：

- 对性能进行分析时，要多方面分析系统的资源瓶颈所在，因为系统某一方面性能低，也许并不是它自己造成的，而是其他方面造成的。如CPU利用率是100%时，很可能是内存容量太小，因为CPU忙于处理内存调度。
- 一次只对影响性能的某方面的一个参数进行调整，多个参数同时调整的话，很难界定性能的影响是由哪个参数造成的。
- 在进行系统性能分析时，性能分析工具本身会占用一定的系统资源，如CPU资源、内存资源等等。我们必须注意到这点，即分析工具本身运行可能会导致系统某方面的资源瓶颈情况更加严重。

### 1.3 调优思路
- 1. 对于服务端的问题，需要重点定位的硬件指标包括CPU、内存、硬盘、BIOS配置，其中CPU是需要重点关注的：服务端CPU未压满或存在明显热点函数直接影响最终测试结果。需要重点关注的软件指标包括应用软件、OS层的优化，该部分的调优对于性能的提升是很可观的。
- 2. 对于网络问题，需要重点定位的包括网络带宽和网络中断，处理网络中断的核是否压满是需要重点关注的。
- 3. 对于客户端问题：客户端的性能是否满足测试需要。
- 4. 识别网卡物理插槽所在NUMA node的CPU，将网卡中断平均绑定至该片CPU的两个NUMA node上，每个中断绑定一个核。
- 5. Redis集群master节点绑定至网卡中断所在CPU上，同时保证与网卡中断使用不同的核，slave节点绑定至另一片CPU上。
- 6. 对于规模较小的集群（单个物理节点上master和slave的总数小于等于网卡中断所在CPU核数减去中断使用的核数），可以把slave节点同时绑定至5中master使用的核上。
- 7. 对于规模极小的集群（单个NUMA node就可以包含所有的中断和进程，此时由于集群规模小，网卡可以多个中断绑定在一个核上），可以将网卡中断和进程绑定在单个NUMA node上，即网卡物理插槽所在NUMA node。

## 2-硬件调优
### 2.1 配置BIOS

#### 1. 关闭SMMU。
   1. 重启服务器，按Esc键进入BIOS设置界面。
   2. 依次进入
      “Advanced > MISC Config > > Support Smmu”。
      **图1** BIOS设置界面
      ![点击放大](https://www.hikunpeng.com/doc_center/source/zh/kunpengbds/ecosystemEnable/HBase/zh-cn_image_0000001154826923.png)

   3. 将“Support Smmu”设置为“Disabled”，按“F10”保存退出（永久有效）。
      **图2** 关闭SMMU BIOS界面
      ![点击放大](https://www.hikunpeng.com/doc_center/source/zh/kunpengbds/ecosystemEnable/HBase/zh-cn_image_0000001108147314.png)

#### 2. 关闭预取。
   1. 进入BIOS设置界面。
   2. 依次进入
      “Advanced > MISC Config > CPU Prefetching Configuration”。
      **图3** BIOS设置界面
      ![点击放大](https://www.hikunpeng.com/doc_center/source/zh/kunpengbds/ecosystemEnable/HBase/zh-cn_image_0000001154986843.png)

   3. 将“CPU Prefetching Configuration”设置为 “Disabled” ，按“F10”保存退出（永久有效）。
     **图4** 关闭Prefetching Configuration界面
      ![点击放大](https://www.hikunpeng.com/doc_center/source/zh/kunpengbds/ecosystemEnable/HBase/zh-cn_image_0000001154706973.png)

### 2.2 调整rx_buff
```
目的 ：1822网卡默认的rx_buff配置为2KB，在聚合64KB报文的时候需要多片不连续的内存，使用率较低；该参数可以配置为2/4/8/16KB，修改后可以减少不连续的内存，提高内存使用率。

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
### 2.3 调整Ring Buffer
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

### 2.4 开启LRO
```
目的：1822网卡支持LRO（Large Receive Offload），可以打开该选项，并合理配置1822 LRO参数。
方法
- 查看LRO参数是否开启，默认为off，假设当前网卡名为enp131s0。
ethtool -k enp131s0
- 打开LRO。
ethtool -K enp131s0 lro on
- 查看LRO参数是否开启。
ethtool -k enp131s0
```

### 2.5 配置MTU
```
目的：当MTU与业务收发包接近时，可以达到最优的吞吐量，由于当前测试数据包大小为1000，因此采用默认的1500可以达到最优的性能。当数据包大于MTU时，在发包过程中数据将被切割后发送，导致网络利用率下降，因此可以适当修改该值使之与业务收发包大小匹配。

方法
- 配置MTU为1500，假设当前网卡名为enp136s0。
ifconfig enp136s0 mtu 1500
```

### 2.6 配置网卡中断合并
```
目的：开启中断合并自适应，当网络上小包占比大时，会自动进行合并中断，有助于提升系统性能，当前测试模型小包占比大，因此需要开启中断合并。

方法
- 开启中断合并，假设当前网卡名为enp136s0。
ethtool -C enp136s0 adaptive-rx on
```
### 2.7 配置网卡中断绑核
```
目的：相比使用内核的irqbalance使网卡中断在所有核上进行调度，使用手动绑核将中断固定住能有效提高业务网络收发包的能力。

方法
- 关闭irqbalance。
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
- 查看NUMA node对应的core的区间，例如此处就可以绑到48~63。
lscpu
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
利用脚本查看是否绑核成功。
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

### 2.8 组件Raid 0
```
目的：磁盘创建RAID可以提高磁盘的整体存取性能，环境上若使用的是LSI 3108/3408/3508系列RAID卡，推荐创建单盘RAID 0/RAID 5，充分利用RAID卡的Cache（LSI 3508卡为2GB），提高读写速率。

方法
- 使用storcli64_arm文件检查RAID组创建情况。
./storcli64_arm /c0 show

说明
storcli64_arm文件放在任意目录下执行皆可。

- 创建RAID 0，命令表示为第2块1.2T硬盘创建RAID 0，其中，c0表示RAID卡所在ID、r0表示组RAID 0。依次类推，对除了系统盘之外的所有盘做此操作。
./storcli64_arm /c0 add vd r0 drives=65:1

```
### 2.9 开启Raid卡Cache
```
目的：使用RAID卡的RAWBC（无超级电容）/RWBC（有超级电容）性能更佳。

- 使用storcli工具修改RAID组的Cache设置，Read Policy设置为Read ahead，使用RAID卡的Cache做预读；Write Policy设置为Write back，使用RAID卡的Cache做回写，而不是Write through透写；IO policy设置为Cached IO，使用RAID卡的Cache缓存IO。
方法
- 配置RAID卡的Cache。
./storcli64_arm /c0/vx set rdcache=RA
./storcli64_arm /c0/vx set wrcache=WB/AWB
./storcli64_arm /c0/vx set iopolicy=Cached

说明
./storcli64_arm /c0/vx set wrcache=WB/AWB #命令中的WB/AWB取决于RAID卡是否有超级电容，无超级电容使用AWB。

- 检查配置是否生效。
./storcli64_arm /c0 show
```
## 3-操作系统调优
### 3.1 配置内核参数
```
目的：配置内核参数，包括TCP/IP协议栈、透明大页等相关参数，提升性能。

方法
- 配置OS内核参数。
sysctl -w net.core.netdev_budget = 600
sysctl -w net.core.rmem_max = 16777216
sysctl -w net.core.somaxconn = 2048
sysctl -w net.core.optmem_max = 40960
sysctl -w net.core.rmem_default = 65535
sysctl -w net.core.wmem_default = 65535
sysctl -w net.core.wmem_max = 8388608
sysctl -w net.ipv4.tcp_rmem = 16384 349520 16777216
sysctl -w net.ipv4.tcp_wmem = 16384 349520 16777216
sysctl -w net.ipv4.tcp_mem = 8388608 8388608 8388608
sysctl -w vm.overcommit_memory= 1
echo never > /sys/kernel/mm/transparent_hugepage/enabled
```
### 3.2 配置进程NUMA绑核
```
目的
根据Redis node在集群中的角色可以将具体的实例分为Master和Slave两类，其中，Master节点负责request的处理，写入、读取、更新数据，定时同步数据到备用Redis node，数据持久化。Slave节点负责当主用Redis node故障时，取代主用Redis node对外提供服务。由于角色不同，在实际业务中，Master实例需要更多的CPU资源与网络资源，因此在Redis集群部署完成后，根据Redis节点在集群中负责的角色，将相应的Redis实例进行NUMA绑核操作。当集群规模不同时，需要调整该绑核策略以发挥不同规模集群的最优性能。

方法
- 获取当前集群的节点（实例）信息，首先使用如下命令通过集群中任一节点登录集群。
redis-cli -h ${ip_server} -p ${port_server}
- 查询当前集群各节点的角色信息，示例如下（简化版），可以看到通过该方式可以获得当前集群中各个实例的具体角色，将该信息保存至临时文件tmp_nodes_info.log中。
cluster nodes
- 根据实例的角色进行绑核，因为Redis集群是部署在多个物理服务器上的分布式数据库，因此需要分别获取每个服务器上Redis实例的角色信息。使用如下命令从tmp_nodes_info.log中批量获取属于某个服务IP地址的实例角色信息。
local_master_ports=($(cat tmp_nodes_info.log |grep "master"|awk '{print $2}'|awk -F@ '{print $1}' | grep ${ip} | awk -F: '{print $2}'))
local_slave_ports=($(cat tmp_nodes_info.log |grep "slave"|awk '{print $2}'|awk -F@ '{print $1}'|grep ${ip} | awk -F: '{print $2}'))
- 根据local_master_ports和local_slave_ports中的端口信息获取相应的进程pid，再根据pid对相应的Redis实例进行绑核操作。
pid=$(ps -ef | grep "redis" | grep ${port} | awk '{print $2}')
taskset -cp ${destCores} ${pid}
```

## 4-resdis服务端调优
### 4.1 目的

服务端参数保持默认。

### 4.2 方法

每个Redis实例的参数在redis.conf文件配置。

| 参数                          | 含义                                                         | 默认值                                                       |
| ----------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| appendfsync                   | fsync（）策略：no：不配置fsync，OS在需要的时候将数据刷入磁盘。always：每次写盘都调fsync，速度慢，但是最安全。everysec：每秒fsync，折中。 | 【默认值】everysec【取值范围】no，always，everysec           |
| appendonly                    | 如果启用了AOF，Redis启动时会加载AOF。                        | 【默认值】no【取值范围】yes，no                              |
| cluster-require-full-coverage | 默认情况下如果一个槽位没有被分配（该槽位没有分配可用的实例）那么Redis集群会停止接受查询请求，在这种情况下，如果集群部分实例故障（例如一部分槽位没有被覆盖），那整个集群也将不可用。一旦所有槽位被分配，集群会自动变为可用。 | 【默认值】yes【取值范围】yes，no                             |
| maxmemory                     | Redis能使用的最大内存（单位：MB）。                          | 【默认值】1024【取值范围】1～1048576                         |
| maxmemory-policy              | 最大内存策略：当达到最大内存时，Redis选择移除的数据。可在以下八个行为中选择：volatile-lru -> 使用LRU算法淘汰有一个过期配置的key。allkeys-lru -> 根据LRU算法淘汰所有key。volatile-random -> 在设置了过期时间的key中随机选择一个进行淘汰。allkeys-random -> 在所有的key中随机选择一个进行淘汰。volatile-ttl -> 淘汰最快过期的key。allkeys-smart -> 使用冷却算法在所有的key中选择一个进行淘汰。volatile-smart -> 使用冷却算法在设置了过期时间的key中选择一个进行淘汰。noeviction -> 不会淘汰任何key，执行写操作时返回错误。 | 【默认值】noeviction【取值范围】volatile-lru，allkeys-lru，volatile-random，allkeys-random，volatile-ttl，allkeys-smart，volatile-smart，noeviction |
| rdbchecksum                   | 使用RDB校验。                                                | 【默认值】yes【取值范围】yes，no                             |
| rdbcompression                | 使用RDB压缩。                                                | 【默认值】yes【取值范围】yes，no                             |
| stop-writes-on-bgsave-error   | 如果启用了RDB快照并且最近的后台保存失败，Redis会停止接受写入请求。 | 【默认值】yes【取值范围】yes，no                             |
| activerehashing               | 将在每100毫秒时使用1毫秒的CPU时间来对redis的hash表进行重新hash，可以降低内存的使用。 | 【默认值】no【取值范围】yes，no                              |
| activedefrag                  | 自动内存碎片整理功能。                                       | 【默认值】no【取值范围】yes，no                              |