### hadoop-3.3.4部署指南
编写日期 2023.3.13
#### 1- 环境要求
##### 1.1 建议版本
| 软件 | 说明 | 获取方法 |
| ---------------- |---------------- |---------------- |
|OpenJDK|1.8.0_342|yum安装或者官网获取|
|hadoop|3.3.4|官网获取，aarch64版本需移植，参考[hadoop移植指南](https://gitee.com/openeuler/bigdata/blob/master/Docs/%E7%A7%BB%E6%A4%8D%E6%8C%87%E5%8D%97/hadoop.md)|
|zookeeper|3.8.1|官网获取，aarch64版本需移植，参考[zookeeper移植指南](https://gitee.com/openeuler/bigdata/blob/master/Docs/%E7%A7%BB%E6%A4%8D%E6%8C%87%E5%8D%97/zookeeper.md)|
##### 1.2硬件要求
```
最低配置：任意CPU、一根内存（大小不限）、一块硬盘（大小不限）。
具体配置视实际应用场景而定。
操作系统要求： 适用于CentOS 7.4~7.6、openeuler-20.03、openEuler-22.03操作系统。
说明
本文以openeuler 22.03为例，介绍Hadoop（1+3）集群部署。
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
|server1|Namenode、ResourceManager|
|agent1|QuorumPeerMain、DataNode、NodeManager、JournalNode|
|agent2|QuorumPeerMain、DataNode、NodeManager、JournalNode|
|agent3|QuorumPeerMain、DataNode、NodeManager、JournalNode|
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
#### 3-部署 ZooKeeper
##### 3.1 编译并解压 ZooKeeper
步骤1 参考ZooKeeper移植指南“[zookeeper移植指南](https://gitee.com/openeuler/bigdata/blob/master/Docs/%E7%A7%BB%E6%A4%8D%E6%8C%87%E5%8D%97/zookeeper.md)”编译出“zookeeper-3.8.1.tar.gz”部署包。
```
步骤2 将zookeeper-3.8.1.tar.gz放置于agent1节点的“/usr/local”目录下，并解压。
mv zookeeper-3.8.1.tar.gz /usr/local
cd /usr/local
tar -zxvf zookeeper-3.8.1.tar.gz
步骤3 建立软链接，便于后期版本更换。
ln -s zookeeper-3.8.1 zookeeper
```
##### 3.2 添加 ZooKeeper 到环境变量
```
步骤1 打开配置文件。
vim /etc/profile
步骤2 添加ZooKeeper到环境变量。
export ZOOKEEPER_HOME=/usr/local/zookeeper
export PATH=$ZOOKEEPER_HOME/bin:$PATH
步骤3 使环境变量生效。
source /etc/profile
```
##### 3.3 修改 ZooKeeper 配置文件
```
步骤1 进入ZooKeeper所在目录。
cd /usr/local/zookeeper/conf
步骤2 拷贝配置文件。
cp zoo_sample.cfg zoo.cfg
步骤3 修改配置文件。
vim zoo.cfg
1. 修改数据目录。
dataDir=/usr/local/zookeeper/tmp
2. 在最后添加如下代码，其中server.1-3是部署ZooKeeper的节点。
server.1=agent1:2888:3888
server.2=agent2:2888:3888
server.3=agent3:2888:3888
4 创建tmp目录作数据目录。
mkdir /usr/local/zookeeper/tmp
步骤5 在tmp目录中创建一个空文件，并向该文件写入ID。
touch /usr/local/zookeeper/tmp/myid
echo 1 > /usr/local/zookeeper/tmp/myid
```
##### 3.4 同步配置到其它节点
```
步骤1 将配置好的ZooKeeper拷贝到其它节点。
scp -r /usr/local/zookeeper-3.4.6 root@agent2:/usr/local
scp -r /usr/local/zookeeper-3.4.6 root@agent3:/usr/local
步骤2 登录agent2、agent3，创建软链接并修改myid内容。
● agent2：
cd /usr/local
ln -s zookeeper-3.8.1 zookeeper
echo 2 > /usr/local/zookeeper/tmp/myid
● agent3：
cd /usr/local
ln -s zookeeper-3.8.1 zookeeper
echo 3 > /usr/local/zookeeper/tmp/myid
```
##### 3.5 运行验证
```
步骤1 分别在agent1，agent2，agent3上启动ZooKeeper。
cd /usr/local/zookeeper/bin
./zkServer.sh start
说明
您可以分别在agent1，agent2，agent3上停止ZooKeeper。
cd /usr/local/zookeeper/bin
./zkServer.sh stop
步骤2 查看ZooKeeper状态。
./zkServer.sh status
```
#### 4-部署Hadoop
##### 4.1 编译并解压 Hadoop
```
步骤1 参考Hadoop 3.3.4 移植指南 编译出Hadoop软件部署包 “hadoop-3.3.4.tar.gz”。
步骤2 将“hadoop-3.3.4.tar.gz”放置于server1节点的“/usr/local”目录，并解压。
mv hadoop-3.3.4.tar.gz /usr/local
cd /usr/local
tar -zxvf hadoop-3.3.4.tar.gz
步骤3 建立软链接，便于后期版本更换。
ln -s hadoop-3.3.4 hadoop
```
##### 4.2 添加 Hadoop 到环境变量
```
步骤1 编辑/etc/profile文件。
vim /etc/profile
步骤2 在文件底部添加环境变量，如下所示。
export HADOOP_HOME=/usr/local/hadoop
export PATH=$HADOOP_HOME/bin:$HADOOP_HOME/sbin:$PATH
步骤3 使环境变量生效。
```
#####  4.3 修改 Hadoop 配置文件
```
说明
Hadoop所有的配置文件都在“$HADOOP_HOME/etc/hadoop”目录下，修改以下配置文件
前，需要切换到“HADOOP_HOME/etc/hadoop”目录。
cd $HADOOP_HOME/etc/hadoop
```
修改 hadoop-env.sh
```
修改环境变量JAVA_HOME为绝对路径，并将用户指定为root。（为jdk安装的路径）
echo "export JAVA_HOME=/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.352.b08-3.oe2203sp1.aarch64/jre/" >> hadoop-env.sh
echo "export HDFS_NAMENODE_USER=root" >> hadoop-env.sh
echo "export HDFS_SECONDARYNAMENODE_USER=root" >> hadoop-env.sh
echo "export HDFS_DATANODE_USER=root" >> hadoop-env.sh
```
修改 yarn-env.sh
```
修改用户为root。
echo "export YARN_REGISTRYDNS_SECURE_USER=root" >> yarn-env.sh
echo "export YARN_RESOURCEMANAGER_USER=root" >> yarn-env.sh
echo "export YARN_NODEMANAGER_USER=root" >> yarn-env.sh
```
修改 core-site.xml
```
步骤1 编辑core-site.xml文件。
vim core-site.xml
步骤2 添加或修改ÑnĒªñàì²Ñn标签范围内的参数。
<property>
 <name>fs.defaultFS</name>
 <value>hdfs://server1:9000</value>
</property>
<property>
 <name>hadoop.tmp.dir</name>
 <value>/home/hadoop_tmp_dir</value>
</property>
<property>
 <name>ipc.client.connect.max.retries</name>
 <value>100</value>
</property>
<property>
 <name>ipc.client.connect.retry.interval</name>
 <value>10000</value>
</property>
<property>
 <name>hadoop.proxyuser.root.hosts</name>
 <value>*</value>
</property>
<property>
 <name>hadoop.proxyuser.root.groups</name>
 <value>*</value>
</property>
```
在节点server1上创建目录。
mkdir /home/hadoop_tmp_dir
修改 hdfs-site.xml
```
步骤1 修改hdfs-site.xml文件。
vim hdfs-site.xml
步骤2 添加或修改ÑnĒªñàì²Ñn标签范围内的参数。
<property>
 <name>dfs.replication</name>
 <value>1</value>
</property>
<property>
 <name>dfs.namenode.name.dir</name>
 <value>/data/data1/hadoop/nn</value>
</property>
<property>
 <name>dfs.datanode.data.dir</name>
 <value>/data/data1/hadoop/dn,/data/data2/hadoop/dn,/data/data3/hadoop/dn,/data/data4/hadoop/dn,/
data/data5/hadoop/dn,/data/data6/hadoop/dn,/data/data7/hadoop/dn,/data/data8/hadoop/dn,/data/data9/
hadoop/dn,/data/data10/hadoop/dn,/data/data11/hadoop/dn,/data/data12/hadoop/dn</value>
</property>
<property>
 <name>dfs.http.address</name>
 <value>server1:50070</value>
</property>
<property>
 <name>dfs.namenode.http-bind-host</name>
 <value>0.0.0.0</value>
</property>
<property>
 <name>dfs.datanode.handler.count</name>
 <value>600</value>
</property>
<property>
 <name>dfs.namenode.handler.count</name>
 <value>600</value>
</property>
<property>
 <name>dfs.namenode.service.handler.count</name>
 <value>600</value>
</property>
<property>
 <name>ipc.server.handler.queue.size</name>
 <value>300</value>
</property>
<property>
 <name>dfs.webhdfs.enabled</name>
 <value>true</value>
</property>
```
```
节点agent1、agent2、agent3分别创建dfs.datanode.data.dir对应目录。
举例：mkdir -p /data/data{1,2,3,4,5,6,7,8,9,10,11,12}/hadoop
```
修改 mapred-site.xml
```
步骤1 编辑mapred-site.xml文件。
vim mapred-site.xml
步骤2 添加或修改ÑnĒªñàì²Ñn标签范围内的参数。
<property>
 <name>mapreduce.framework.name</name>
 <value>yarn</value>
 <final>true</final>
 <description>The runtime framework for executing MapReduce jobs</description>
</property>
<property>
 <name>mapreduce.job.reduce.slowstart.completedmaps</name>
 <value>0.88</value>
</property>
<property>
 <name>mapreduce.application.classpath</name>
 <value>
 /usr/local/hadoop/etc/hadoop,
 /usr/local/hadoop/share/hadoop/common/*,
 /usr/local/hadoop/share/hadoop/common/lib/*,
 /usr/local/hadoop/share/hadoop/hdfs/*,
 /usr/local/hadoop/share/hadoop/hdfs/lib/*,
 /usr/local/hadoop/share/hadoop/mapreduce/*,
 /usr/local/hadoop/share/hadoop/mapreduce/lib/*,
 /usr/local/hadoop/share/hadoop/yarn/*,
 /usr/local/hadoop/share/hadoop/yarn/lib/*
 </value>
</property>
<property>
 <name>mapreduce.map.memory.mb</name>
 <value>6144</value>
</property>
<property>
 <name>mapreduce.reduce.memory.mb</name>
 <value>6144</value>
 </property>
 <property>
 <name>mapreduce.map.java.opts</name>
 <value>-Xmx5530m</value>
</property>
<property>
 <name>mapreduce.reduce.java.opts</name>
 <value>-Xmx2765m</value>
</property>
<property>
 <name>mapred.child.java.opts</name>
 <value>-Xmx2048m -Xms2048m</value>
</property>
<property>
 <name>mapred.reduce.parallel.copies</name>
 <value>20</value>
</property>
<property>
 <name>yarn.app.mapreduce.am.env</name>
 <value>HADOOP_MAPRED_HOME=/usr/local/hadoop</value>
</property>
<property>
 <name>mapreduce.map.env</name>
 <value>HADOOP_MAPRED_HOME=/usr/local/hadoop</value>
</property>
<property>
 <name>mapreduce.reduce.env</name>
 <value>HADOOP_MAPRED_HOME=/usr/local/hadoop</value>
</property>
```
修改 yarn-site.xml
```
步骤1 编辑yarn-site.xml文件。
vim yarn-site.xml
步骤2 添加或修改ÑnĒªñàì²Ñn标签范围内的参数。
<property>
 <name>yarn.nodemanager.aux-services</name>
 <value>mapreduce_shuffle</vaule>
 <final>true</final>
</property>
<property>
 <name>yarn.nodemanager.aux-services</name>
 <value>mapreduce_shuffle</vaule>
</property>
<property>
 <name>yarn.resourcemanager.hostname</name>
 <value>server1</value>
</property>
<property>
 <name>yarn.resourcemanager.bind-host</name>
 <value>0.0.0.0</value>
</property>
<property>
 <name>yarn.scheduler.maximum-allocation-mb</name>
 <value>65536</value>
</property>
<property>
 <name>yarn.nodemanager.resource.memory-mb</name>
 <value>102400</value>
</property>
<property>
 <name>yarn.nodemanager.resource.cpu-vcores</name>
 <value>48</value>
</property>
<property>
 <name>yarn.log-aggregation-enable</name>
 <value>true</value>
</property>
<property>
 <name>yarn.client.nodemanager-connect.max-wait-ms</name>
 <value>300000</value>
</property>
<property>
 <name>yarn.nodemanager.vmem-pmem-ratio</name>
 <value>7.1</value>
</property>
<property>
 <name>yarn.nodemanager.vmem-check-enabled</name>
 <value>false</value>
</property>
<property>
 <name>yarn.nodemanager.pmem-check-enabled</name>
 <value>false</value>
</property>
<property>
 <name>yarn.scheduler.minimum-allocation-mb</name>
 <value>3072</value>
</property>
<property>
 <name>yarn.app.mapreduce.am.resource.mb</name>
 <value>3072</value>
</property>
<property>
 <name>yarn.scheduler.maximum-allocation-vcores</name>
 <value>48</value>
</property>
<property>
 <name>yarn.application.classpath</name>
 <value>
 /usr/local/hadoop/etc/hadoop,
 /usr/local/hadoop/share/hadoop/common/*,
 /usr/local/hadoop/share/hadoop/common/lib/*,
 /usr/local/hadoop/share/hadoop/hdfs/*,
 /usr/local/hadoop/share/hadoop/hdfs/lib/*,
 /usr/local/hadoop/share/hadoop/mapreduce/*,
 /usr/local/hadoop/share/hadoop/mapreduce/lib/*,
 /usr/local/hadoop/share/hadoop/yarn/*,
 /usr/local/hadoop/share/hadoop/yarn/lib/*
 </value>
</property>
<property>
 <name>yarn.nodemanager.local-dirs</name> <value>/data/data1/hadoop/yarn/local,/data/data2/
hadoop/yarn/local,/data/data3/hadoop/yarn/local,/data/data4/hadoop/yarn/local,/data/data5/hadoop/yarn/
local,/data/data6/hadoop/yarn/local,/data/data7/hadoop/yarn/local,/data/data8/hadoop/yarn/local,/data/
data9/hadoop/yarn/local,/data/data10/hadoop/yarn/local,/data/data11/hadoop/yarn/local,/data/data12/
hadoop/yarn/local</value>
 </property>
 <property>
 <name>yarn.nodemanager.log-dirs</name> <value>/data/data1/hadoop/yarn/log,/data/data2/
hadoop/yarn/log,/data/data3/hadoop/yarn/log,/data/data4/hadoop/yarn/log,/data/data5/hadoop/yarn/log,/
data/data6/hadoop/yarn/log,/data/data7/hadoop/yarn/log,/data/data8/hadoop/yarn/log,/data/data9/
hadoop/yarn/log,/data/data10/hadoop/yarn/log,/data/data11/hadoop/yarn/log,/data/data12/hadoop/yarn/
log</value>
 </property>
```
```
节点agent1、agent2、agent3分别创建yarn.nodemanager.local-dirs对应目录。
举例：mkdir -p /data/data{1,2,3,4,5,6,7,8,9,10,11,12}/hadoop/yarn
```
修改 slaves 或 workers
```
步骤1 确认Hadoop版本，3.x以下的版本编辑slaves文件，3.x及以上的编辑workers文件。
步骤2 编辑文件（本文版本3.1.1）。
vim workers
步骤3 修改workers文件，只保存所有agent节点的IP地址（可用主机名代替），其余内容均
删除。
agent1
agent2
agent3
```
##### 4.4 同步配置到其它节点
```
步骤1 所有节点依次创建journaldata目录。
mkdir -p /usr/local/hadoop-3.3.4/journaldata
步骤2 拷贝hadoop-3.1.1到agent1、agent2、agent3节点的“/usr/local”目录。
scp -r /usr/local/hadoop-3.3.4 root@agent1:/usr/local
scp -r /usr/local/hadoop-3.3.4 root@agent2:/usr/local
scp -r /usr/local/hadoop-3.3.4 root@agent3:/usr/local
步骤3 分别登录到agent1、agent2、agent3节点，为hadoop-3.3.4建立软链接。
cd /usr/local
ln -s hadoop-3.3.4 hadoop
```
##### 4.5启动 Hadoop 集群
```
步骤1 启动ZooKeeper集群。
分别在agent1，agent2，agent3节点上启动ZooKeeper。
cd /usr/local/zookeeper/bin
./zkServer.sh start
步骤2 启动JournalNode。
分别在agent1，agent2，agent3节点上启动JournalNode。
说明
只在第一次进行格式化操作时，需要执行步骤2-步骤4，完成格式化后，下次启动集群，只需要
执行步骤1、步骤5、步骤6。
cd /usr/local/hadoop/sbin
./hadoop-daemon.sh start journalnode
步骤3 格式化HDFS。
1. 在server1节点上格式化HDFS。
hdfs namenode -format
2. 格式化后集群会根据core-site.xml配置的hadoop.tmp.dir参数生成目录。
本文档配置目录为“/home/hadoop_tmp”。
步骤4 格式化ZKFC。
在server1节点上格式化ZKFC。
hdfs zkfc -formatZK
步骤5 启动HDFS。
在server1节点上启动HDFS。
cd /usr/local/hadoop/sbin
./start-dfs.sh
步骤6 启动Yarn。
在server1节点上启动Yarn。
cd /usr/local/hadoop/sbin
./start-yarn.sh
步骤7 观察进程是否都正常启动。
使用jps命令查看进程
```
##### 4.6 验证hadoop
在浏览器中输入URL地址，访问Hadoop Web页面，URL格式为“http://
server1:50070”。
