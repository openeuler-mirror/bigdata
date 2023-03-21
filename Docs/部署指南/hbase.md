### Hbase-2.5.0 移植指南
编写日期 2023.3.13
#### 1- 环境要求
##### 1.1 建议版本
| 软件 | 说明 | 获取方法 |
| ---------------- |---------------- |---------------- |
|OpenJDK|1.8.0_342|yum安装或者官网获取|
|hadoop|3.3.4|官网获取，aarch64版本需移植，参考[hadoop移植指南](https://gitee.com/macchen1/bigdata/blob/change-bigdat/Docs/%E7%A7%BB%E6%A4%8D%E6%8C%87%E5%8D%97/hadoop.md)|
|zookeeper|3.8.1|官网获取，aarch64版本需移植，参考[zookeeper移植指南](https://gitee.com/macchen1/bigdata/blob/change-bigdat/Docs/%E7%A7%BB%E6%A4%8D%E6%8C%87%E5%8D%97/zookeeper.md)|
|hbase|2.5.0|官网获取，aarch64版本需移植，参考[hbase移植指南](https://gitee.com/macchen1/bigdata/blob/change-bigdat/Docs/%E7%A7%BB%E6%A4%8D%E6%8C%87%E5%8D%97/hbase.md)|
##### 1.2硬件要求
```
最低配置：任意CPU、一根内存（大小不限）、一块硬盘（大小不限）。
具体配置视实际应用场景而定。
操作系统要求： 适用于CentOS 7.4~7.6、openeuler-20.03、openEuler-22.03操作系统。
说明
本文以openeuler 22.03为例，介绍flink（1+3）集群部署。
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
|server1|Namenode、ResourceManager、Hmaster|
|agent1|QuorumPeerMain、DataNode、NodeManager、JournalNode、HRegionServer|
|agent2|QuorumPeerMain、DataNode、NodeManager、JournalNode、HRegionServer|
|agent3|QuorumPeerMain、DataNode、NodeManager、JournalNode、HRegionServer|
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
#### 3-部署zookeeper
参考 [hadoop部署指南](https://gitee.com/macchen1/bigdata/blob/change-bigdat/Docs/%E9%83%A8%E7%BD%B2%E6%8C%87%E5%8D%97/hadoop.md)
#### 4-部署hadoop
参考 [hadoop部署指南](https://gitee.com/macchen1/bigdata/blob/change-bigdat/Docs/%E9%83%A8%E7%BD%B2%E6%8C%87%E5%8D%97/hadoop.md)
#### 5-部署hbase
##### 5.1 下载并解压 HBase
步骤1 下载HBase。
官网获取 或者 aarch64版本需移植，参考[hbase移植指南](https://gitee.com/macchen1/bigdata/blob/change-bigdat/Docs/%E7%A7%BB%E6%A4%8D%E6%8C%87%E5%8D%97/hbase.md)|
```
步骤2 将hbase-2.5.0-bin.tar.gz放置于server1节点的“/usr/local”目录，并解压。
mv hbase-2.5.0-bin.tar.gz /usr/local
tar -zxvf hbase-2.5.0-bin.tar.gz
步骤3 建立软链接，便于后期版本更换。
ln -s hbase-2.5.0 hbase
```
##### 5.2 添加 HBase 到环境变量
```
步骤1 编辑“/etc/profile”文件。
vim /etc/profile
步骤2 在文件底部添加环境变量，如下所示。
export HBASE_HOME=/usr/local/hbase
export PATH=$HBASE_HOME/bin:$HBASE_HOME/sbin:$PATH
步骤3 使环境变量生效。
source /etc/profile
```
##### 5.3 修改 HBase 配置文件
```
说明
HBase所有的配置文件都在“HBASE_HOME/conf”目录下，修改以下配置文件前，切换到
“HBASE_HOME/conf”目录。
cd $HBASE_HOME/conf
```
修改 hbase-env.sh
```
步骤1 编辑hbase-env.sh文件。
vim hbase-env.sh
步骤2 修改环境变量JAVA_HOME为绝对路径，HBASE_MANAGES_ZK设为false。
export JAVA_HOME=/usr/local/jdk8u252-b09
export HBASE_MANAGES_ZK=false
export HBASE_LIBRARY_PATH=/usr/local/hadoop/lib/native
```
修改 hbase-site.xml
```
步骤1 修改hbase-site.xml文件。
vim hbase-site.xml
步骤2 添加或修改ÑnĒªñàì²Ñn标签范围内的部分参数。
<configuration>
 <property>
 <name>hbase.rootdir</name>
 <value>hdfs://server1:9000/HBase</value>
 </property>
 <property>
 <name>hbase.tmp.dir</name>
 <value>/usr/local/hbase/tmp</value>
 </property>
 <property>
 <name>hbase.cluster.distributed</name>
 <value>true</value>
 </property>
 <property>
 <name>hbase.unsafe.stream.capability.enforce</name>
 <value>false</value>
 </property>
 <property>
 <name>hbase.zookeeper.quorum</name>
 <value>agent1:2181,agent2:2181,agent3:2181</value>
 </property>
 <property>
 <name>hbase.unsafe.stream.capability.enforce</name>
 <value>false</value>
 </property>
</configuration>
```
修改 regionservers
```
步骤1 编辑regionservers文件。
vim regionservers
步骤2 将regionservers文件内容替换为agent节点IP（可用主机名代替）。
agent1
agent2
agent3
```
拷贝 hdfs-site.xml
```
拷贝hadoop目录下的的的hdfs-site.xml文件到“hbase/conf/”目录，可选择软链接或
拷贝。
cp /usr/local/hadoop/etc/hadoop/hdfs-site.xml /usr/local/hbase/conf/hdfs-site.xml
```
##### 5.4 同步配置到其它节点
```
步骤1 拷贝hbase-2.5.0到agent1、agent2、agent3节点的“/usr/local”目录。
scp -r /usr/local/hbase-2.5.0 root@agent1:/usr/local
scp -r /usr/local/hbase-2.5.0 root@agent2:/usr/local
scp -r /usr/local/hbase-2.5.0 root@agent3:/usr/local
步骤2 分别登录到agent1、agent2、agent3节点，为hbase-2.5.0建立软链接。
cd /usr/local
ln -s hbase-2.5.0 hbase
```
##### 5.5 启动 HBase 集群
```
步骤1 依次启动ZooKeeper和Hadoop。
步骤2 在server1节点上启动HBase集群。
/usr/local/hbase/bin/start-hbase.sh
步骤3 观察进程是否都正常启动。
jps
server1：
ResourceManager
NameNode
HMaster
agent1：
NodeManager
DataNode
HRegionServer
JournalNode
QuorumPeerMain
```
##### 5.6 停止 HBase 集群(可选)
在server1节点上停止HBase集群。
/usr/local/hbase/bin/stop-hbase.sh
##### 5.7 验证 HBase
打开浏览器，可通过URL地址，访问HBase Web页面，URL格式为“http://
server1:16010”。其中，“server1”填写HMaster进程所在节点的IP地址，
“16010”是HBase 1.0以后版本的默认端口，可通过修改“hbase-site.xml”文件的
“hbase.master.info.port”参数进行设置。
通过观察Region Servers数量是否与agent数目相等（本文是3个agent）判断集群是否
正常启动。
