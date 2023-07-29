### Flink-1.13.0部署指南
编写日期 2023.3.13
#### 1- 环境要求
##### 1.1 建议版本
| 软件 | 说明 | 获取方法 |
| ---------------- |---------------- |---------------- |
|OpenJDK|1.8.0_342|yum安装或者官网获取|
|flink|1.13.0|官网获取，aarch64版本需移植，参考[flink移植指南](https://gitee.com/openeuler/bigdata/blob/master/Docs/%E7%A7%BB%E6%A4%8D%E6%8C%87%E5%8D%97/flink.md)
|hadoop|3.3.4|官网获取，aarch64版本需移植，参考[hadoop移植指南](https://gitee.com/openeuler/bigdata/blob/master/Docs/%E7%A7%BB%E6%A4%8D%E6%8C%87%E5%8D%97/hadoop.md)|
|zookeeper|3.8.1|官网获取，aarch64版本需移植，参考[zookeeper移植指南](https://gitee.com/openeuler/bigdata/blob/master/Docs/%E7%A7%BB%E6%A4%8D%E6%8C%87%E5%8D%97/zookeeper.md)|
##### 1.2硬件要求
```
最低配置：任意CPU、一根内存（大小不限）、一块硬盘（大小不限）。
具体配置视实际应用场景而定。
操作系统要求： 适用于CentOS 7.4~7.6、openeuler-20.03、openEuler-22.03操作系统。
说明
本文以openeuler 22.03为例，介绍flink部署。
```
##### 1.3 集群环境规划
本章节规划以四台机器分别作为集群的节点1、节点2、节点3、节点4。各个节点数据
| 机器名称 | IP地址 | 硬盘数 | OS & JDK|
| ---------------- |---------------- |---------------- |---------------- |
|server1|IPaddress1|系统盘：1 * 4TB 数据盘：12 * 4TB HDD|openeuler-22.03 & OpenJDK-1.8.0_342|
|agent1|IPaddress2|系统盘：1 * 4TB 数据盘：12 * 4TB HDD|openeuler-22.03 & OpenJDK-1.8.0_342|
|agent2|IPaddress3|系统盘：1 * 4TB 数据盘：12 * 4TB HDD|openeuler-22.03 & OpenJDK-1.8.0_342|
|agent3|IPaddress4|系统盘：1 * 4TB 数据盘：12 * 4TB HDD|openeuler-22.03 & OpenJDK-1.8.0_342|
##### 1.4 软件规划
| 机器名称 | 服务名称 |
| ---------------- |---------------- |
|server1|StandaloneSessionClusterEntrypoint、(Namenode、ResourceManager)|
|agent1|TaskManagerRunner、(DataNode、NodeManager、JournalNode)|
|agent2|TaskManagerRunner、(DataNode、NodeManager、JournalNode)|
|agent3|TaskManagerRunner、(DataNode、NodeManager、JournalNode)|
#### 2- 配置部署环境
```
步骤1 依次登录节点1-4，将节点的主机名分别修改为server1、agent1、agent2、agent3。
hostnamectl set-hostname 主机名 --static
步骤2 登录所有节点，修改“/etc/hosts”文件。
在hosts文件中添加集群所有节点的“地址-主机名”映射关系。
IPaddress1 server1
IPaddress2 agent1
IPaddress3 agent2
IPaddress4 agent3
步骤3 登录所有节点，关闭防火墙。
systemctl stop firewalld.service
systemctl disable firewalld.service
systemctl status firewalld.service
步骤4 登录所有节点，配置SSH免密登录。
1. 生成密钥，遇到提示时，按回车。
ssh-keygen -t rsa
2. 在每台机器上配置SSH免密登录（包括配置自身节点的免密）。
ssh-copy-id -i ~/.ssh/id_rsa.pub root@节点IP
步骤5 登录所有节点，安装OpenJDK，可使用指定版本jdk。
yum install -y java-1.8.0
java -version
```
#### 3-flink部署
Flink的部署有3种模式，分别是local模式、Standalone模式、yarn模式。其中local就是单机模式，一般来说用于本地开发测试；Standalone跟yarn模式都可以支撑集群部署、实现HA，但是两者在任务分配机制、内存管理等内容上有比较大的差异。一般在处理计算数据量级非常大的生产环境，使用flink on yarn的模式更多一些。

下载flink。 官网获取 或者 aarch64版本需移植，参考[flink移植指南](https://gitee.com/openeuler/bigdata/blob/master/Docs/%E7%A7%BB%E6%A4%8D%E6%8C%87%E5%8D%97/flink.md)。
##### 3.1 本地模式
解压并直接执行 ./bin/start-cluster.sh  即可启动flink服务。
##### 3.2 Standalone模式
```
步骤1 将flink-1.13.0-bin.tar.gz放置于server1节点的“/usr/local”目录，并解压。
mv flink-1.13.0-bin.tar.gz /usr/local
tar -zxvf flink-1.13.0-bin.tar.gz
步骤2 建立软链接，便于后期版本更换。
ln -s flink-1.13.0 flink
```
###### 3.2.1 添加 flink 到环境变量
```
步骤1 编辑“/etc/profile”文件。
vim /etc/profile
步骤2 在文件底部添加环境变量，如下所示。
export FLINK=HOME=/usr/local/flink
export PATH=$FLINK_HOME/bin:$PATH
步骤3 使环境变量生效。
source /etc/profile
说明
flink所有的配置文件都在“FLINK_HOME/conf”目录下，修改以下配置文件前，切换到
“FLINK_HOME/conf”目录。
cd $FLINK_HOME/conf
```
###### 3.2.2配置flink conf文件
配置flink-conf.yaml
```
vim flink-conf.yaml
jobmanager.rpc.address: server1
注：
如需配置flink的历史任务地址则需要配置hadoop，配以下参数：
# 指定由JobManager归档的作业信息所存放的目录，这里使用的是HDFS
jobmanager.archive.fs.dir: hdfs://server1:9000/completed-jobs/
# 指定History Server扫描哪些归档目录，多个目录使用逗号分隔
historyserver.archive.fs.dir: hdfs://server1:9000/completed-jobs/
# 指定History Server间隔多少毫秒扫描一次归档目录
historyserver.archive.fs.refresh-interval: 10000
# History Server所绑定的ip，0.0.0.0代表允许所有ip访问
historyserver.web.address: 0.0.0.0
# 指定History Server所监听的端口号
historyserver.web.port: 8082
内存配置用户自定义，略
```
配置 master
```
vim masters
修改为server1的地址
```
配置 work
```
vim workers
agent1
agent2
agent3
```
##### 3.2.3 同步配置到其它节点
```
步骤1 拷贝hbase-2.5.0到agent1、agent2、agent3节点的“/usr/local”目录。
scp -r /usr/local/flink-1.13.0 root@agent1:/usr/local
scp -r /usr/local/flink-1.13.0  root@agent2:/usr/local
scp -r /usr/local/flink-1.13.0  root@agent3:/usr/local
步骤2 分别登录到agent1、agent2、agent3节点，为flink-1.13.0 建立软链接。
cd /usr/local
ln -s flink-1.13.0  flink
```
###### 3.2.4 启动flink 
./start-cluster.sh
###### 3.2.5 查看flink
可通过master所在机器地址查看运行状态：server1:8081
##### 3.3 flink on yarn
配置hadoop。再参考flink Standalone

##### 3.4 flink 绑核
```
vim bin/taskmanager.sh
注释如下信息：
60 #if [[ $STARTSTOP == "start-foreground" ]]; then
 61 #    exec "${FLINK_BIN_DIR}"/flink-console.sh $ENTRYPOINT "${ARGS[@]}"
 62 #else
 63 #    if [[ $FLINK_TM_COMPUTE_NUMA == "false" ]]; then
 64 #        # Start a single TaskManager
 65 #        "${FLINK_BIN_DIR}"/flink-daemon.sh $STARTSTOP $ENTRYPOINT "${ARGS[@]}"
 66 #    else
 67         # Example output from `numactl --show` on an AWS c4.8xlarge:
 68         # policy: default
 69         # preferred node: current
 70         # physcpubind: 0 1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19 20 21 22 23 24 25 26 27 28 2    9 30 31 32 33 34 35
 71         # cpubind: 0 1
 72         # nodebind: 0 1
 73         # membind: 0 1
 74         read -ra NODE_LIST <<< $(numactl --show | grep "^nodebind: ")
 75         for NODE_ID in "${NODE_LIST[@]:1}"; do
 76             # Start a TaskManager for each NUMA node
 77             numactl --membind=$NODE_ID --cpunodebind=$NODE_ID -- "${FLINK_BIN_DIR}"/flink-daemon.sh $    STARTSTOP $ENTRYPOINT "${ARGS[@]}"
 78         done
 79 #    fi
 80 #fi

查看是否生效
jps  
taskset –pc 进程号
可查看到TaskManagerRunner 在哪几个cpu上运行。
```
