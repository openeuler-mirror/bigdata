### YCSB-HBASE 部署指南
编写日期 2026.1.26
#### 1- 环境要求
##### 1.1 建议版本
| 软件 | 说明 | 获取方法 |
| ---------------- |---------------- |---------------- |
|OpenJDK|1.8.0_342|yum安装或者官网获取|
|hadoop|3.3.4|官网获取，aarch64版本需移植，参考[hadoop移植指南](https://atomgit.com/openeuler/bigdata/blob/master/Docs/%E7%A7%BB%E6%A4%8D%E6%8C%87%E5%8D%97/hadoop.md)|
|zookeeper|3.8.1|官网获取，aarch64版本需移植，参考[zookeeper移植指南](https://atomgit.com/openeuler/bigdata/blob/master/Docs/%E7%A7%BB%E6%A4%8D%E6%8C%87%E5%8D%97/zookeeper.md)|
|hbase|2.5.0|官网获取，aarch64版本需移植，参考[hbase移植指南](https://atomgit.com/openeuler/bigdata/blob/master/Docs/%E7%A7%BB%E6%A4%8D%E6%8C%87%E5%8D%97/hbase.md)|
|ycsb|0.16.0|官网获取，https://github.com/brianfrankcooper/YCSB/archive/master.zip|
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
参考 [hadoop部署指南](https://atomgit.com/openeuler/bigdata/blob/master/Docs/%E9%83%A8%E7%BD%B2%E6%8C%87%E5%8D%97/hadoop.md)
#### 4-部署hadoop
参考 [hadoop部署指南](https://atomgit.com/openeuler/bigdata/blob/master/Docs/%E9%83%A8%E7%BD%B2%E6%8C%87%E5%8D%97/hadoop.md)
#### 5-部署hbase
参考 [hbase部署指南](https://atomgit.com/aa941101/bigdata/blob/master/Docs/%E9%83%A8%E7%BD%B2%E6%8C%87%E5%8D%97/hbase.md)
##### 5.1 下载并解压 YCSB
```
wget  https://github.com/brianfrankcooper/YCSB/archive/master.zip
unzip https://github.com/brianfrankcooper/YCSB/archive/master.zip
```
##### 5.2 编译YCSB
```
cd YCSB-master
mvn -pl com.yahoo.ycsb:hbase20-binding -am clean package
编译完成后，生成ycsb-hbase20-binding-0.16.0-SNAPSHOT.tar.gz
tar xf ycsb-hbase20-binding-0.16.0-SNAPSHOT.tar.gz
```
##### 5.3 使用说明
```
执行bin/ycsb -h 可查看具体的ycsb使用说明
这里以hbase2进行测试说明，在hbase里建好一张表usertable1
create "usertable1", "family"
查看workloads/workloada 文件
修改readproportion=0.5 updateproportion=0.5
读50%数据更新50%
bin/ycsb load hbase20 -P workloads/workloada -cp /home/hbasetest/installed/hbase-2.1.0/conf/ -p table=usertable1 -p columnfamily=fa        mily -s -threads 64 -p recordcount=2000000
运行完之后可以查看测试的数据库Throughput(ops/sec)
```
