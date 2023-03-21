###  **zookeeper-3.8.1移植指南** 
编写日期：2023年03月10日

### 1. 介绍
Zookeeper是一个分布式的，开放源码的分布式应用程序协调服务，是Google的Chubby一个开源的实现，是Hadoop和HBase的重要组件。它是一个为分布式应用提供一致性服务的软件，提供的功能包括：配置维护、域名服务、分布式同步、组服务等。

关于Zookeeper的更多信息请访问Zookeeper官网。

### 2. 环境要求
#### 2.1 硬件要求
| 硬件 | 说明 | 
| ---------------- |---------------- |
|服务器|对服务器无要求|
|CPU|aarch64架构|
|磁盘分区|对磁盘分区无要求|
|网络|可访问外网|
#### 2.2 软件要求
| 软件 | 说明 | 
| ---------------- |---------------- |
|版本|openeuler 22.03|
|OS|5.10.0-60.65.0.90.oe2203.aarch64|
|GCC|10.3.1|
|OpenJDK|1.8.0_342|
|Maven|3.6.3|
|cmake|3.22.0|
|zookeeper|3.8.1|
### 3. 安装基础库
#### 3.1 安装gcc maven jdk-1.8.0

yum -y install gcc.aarch64 gcc-c++.aarch64 gcc-gfortran.aarch64 libgcc.aarch64 maven java-1.8.0-openjdk.aarch64 java-1.8.0-openjdk-devel.aarch64 

#### 3.2 安装依赖
yum install -y wget vim openssl-devel zlib-devel autoconf automake libtool ant svn make libstdc++-static glibc-static git snappy snappy-devel fuse fuse-devel

#### 3.3 配置maven
修改Maven配置文件中的：本地仓路径、远程仓等。

配置文件路径：“/etc/maven/settings.xml”

说明：

```
<!--默认在“~/.m2/”目录下，修改成自定义的目录-->
<localRepository>/path/to/local/repo</localRepository>

<!--修改成自己搭建的maven仓库，ARM使能后的jar包替换到该仓库-->
<mirror>
  <id>huaweimaven</id>
  <name>huawei maven</name>
  <url>https://repo.huaweicloud.com/repository/maven/</url>
  <mirrorOf>central</mirrorOf>
</mirror>
```
### 4 移植分析
| 原始jar包 |so文件 | 
| ---------------- |---------------- |
|jline-2.14.6.jar| libjansi.so|
```
jline-2.14.6.jar aarch64下载地址：https://repo.huaweicloud.com/kunpeng/maven/jline/jline/2.14.6/jline-2.14.6.jar
将下载的jline-2.14.6.jar包放入“maven本地仓库下jline/jline/2.14.6/”下。
```
### 5 编译Zookeeper
#### 5.1 从官网上下载Zookeeper-release-3.8.1源码并解压。
```
wget https://codeload.github.com/apache/zookeeper/tar.gz/refs/tags/release-3.8.1
tar -zxf release-3.8.1.tar.gz
```
#### 5.2 进入解压后目录并安装编译。
```
cd zookeeper-release-3.8.1
mvn clean package -Dmaven.test.skip=true
```
#### 5.3 zookeeper-3.8.1.tar.gz包路径 
编译成功后将在源码目录下“zookeeper-assembly/target”生成zookeeper-3.8.1.tar.gz包。
#### 5.4 使用鲲鹏分析扫描工具扫描编译生成的tar包，或者使用find.so脚本查找，确保没有包含有x86的so和jar包。
如果有，则参考4，将jar包替换本地仓库，并重新编译zookeeper。
