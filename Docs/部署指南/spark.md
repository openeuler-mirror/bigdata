### spark-3.3.0部署指南
编写日期 2023.3.14
#### 1- 环境要求
##### 1.1 建议版本
| 软件 | 说明 | 获取方法 |
| ---------------- |---------------- |---------------- |
|OpenJDK|1.8.0_342|yum安装或者官网获取|
|hadoop|3.3.4|官网获取，aarch64版本需移植，参考[hadoop移植指南](https://gitee.com/openeuler/bigdata/blob/master/Docs/%E7%A7%BB%E6%A4%8D%E6%8C%87%E5%8D%97/hadoop.md)|
|spark|3.3.0|官网获取，aarch64版本需移植，参考[spark移植指南](https://gitee.com/openeuler/bigdata/blob/master/Docs/%E7%A7%BB%E6%A4%8D%E6%8C%87%E5%8D%97/spark.md)|
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
|server1|Namenode、ResourceManager、RunJar、RunJar|
|agent1|DataNode、NodeManager、JournalNode|
|agent2|DataNode、NodeManager、JournalNode|
|agent3|DataNode、NodeManager、JournalNode|
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
#### 3- 部署Spark
在部署Hive之前请先安装部署好Hadoop,参考[hadoop部署指南](https://gitee.com/openeuler/bigdata/blob/master/Docs/%E9%83%A8%E7%BD%B2%E6%8C%87%E5%8D%97/hadoop.md)
##### 3.1 安装Scala
```
wget https://downloads.lightbend.com/scala/2.12.15/scala-2.12.15.tgz
tar zxf scala-2.12.15.tgz -C /usr/local/
ln -s /usr/local/scala /usr/local/scala-2.12.15
```
##### 3.2 配置Spark环境变量（下面Spark的路径已经提前做了软连接的操作）
```
vim /etc/profile
#增加以下内容：
# SPARK_HOME
export SPARK_HOME=/usr/local/spark
export PATH=$PATH:$SPARK_HOME/bin
# SCALA_HOME
export SCALA_HOME=/usr/local/scala
export PATH=${SCALA_HOME}/bin:${PATH}

#保存退出 source 使其生效
source /etc/profile
```
##### 3.3 配置Spark文件
cd $SPARK_HOME/conf
###### 3.3.1 配置 spark-env.sh
```
cp spark-env.sh.template spark-env.sh

vim spark-env.sh

export JAVA_HOME=/usr/local/jdk1.8.0_342
export SCALA_HOME=/usr/local/scala
export HADOOP_HOME=/usr/local/hadoop
export HADOOP_CONF_DIR=/usr/local/hadoop/etc/hadoop
```
###### 3.3.2 配置 spark-defaults.conf
```
cp spark-defaults.conf.template spark-defaults.conf 

vim spark-defaults.conf

park.eventLog.dir                hdfs://server1:9000/spark_history
spark.eventLog.compress           true
spark.eventLog.enabled            true
spark.serializer                   org.apache.spark.serializer.KryoSerializer
spark.history.fs.logDirectory        hdfs:///server1:9000/spark_history
spark.yarn.historyServer.address      server1:18080
spark.history.ui.port               18080
spark.jobhistory.address         http://server1:19888/history/logs
```
###### 3.3.3 配置 log4j2.properties
```
cp log4j2.properties.template log4j2.properties
```
###### 3.3.4 配置 slaves
```
vim slaves
server1
agent1
agent2
agent3
```
###### 3.3.5 将hadoop配置文件core-site.xml，hdfs-site.xml拷贝倒spark/conf目录下
```
cp $HADOOP_HOME/etc/hadoop/core-site.xml $HADOOP_HOME/etc/hadoop/hdfs-site.xml $SPARK_HOME/conf
```
###### 3.3.6 将hive配置文件hive-site.xml拷贝倒spark/conf目录下（配置hive on spark模式才进行）
```
cp $HIVE_HOME/conf/hive-site.xml $SPARK_HOME/conf
```
##### 3.4 启动Spark
```
sh $SPARK_HOME/sbin/start-all.sh
sh $SPARK_HOME/sbin/start-history-server.sh
# sh $SPARK_HOME/sbin/start-thriftserver.sh --driver-class-path $SPARK_HOME/jars/${jdbc驱动名称}

```
