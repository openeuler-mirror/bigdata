### accumulo 2.1.0 部署指南
编写日期 2026.1.26
#### 1- 环境要求
##### 1.1 建议版本
| 软件 | 说明 | 获取方法 |
| ---------------- |---------------- |---------------- |
|OpenJDK|1.8.0_342|yum安装或者官网获取|
|hadoop|3.3.4|官网获取|
|zookeeper|3.8.1|官网获取|
|accumulo|2.1.0|官网获取|
#### 2- 部署

accumulo运行依赖hadoop、zookeeper和JDK。

hadoop，zookeeper部署参考[hadoop部署指南](https://atomgit.com/xiexing01/bigdata/edit/master/Docs/%E9%83%A8%E7%BD%B2%E6%8C%87%E5%8D%97/hadoop.md)

需在conf/accumulo-env.sh中配置：

```
##### 示例配置
export HADOOP_HOME=/usr/local/hadoop
export ZOOKEEPER_HOME=/usr/local/zookeeper
export JAVA_HOME=/usr/lib/jvm/java-8-openjdk
```
accumulo配置环境变量：

```
# 临时设置（当前会话有效）
export ACCUMULO_HOME=/opt/accumulo-2.1.0

# 永久设置（推荐）
echo 'export ACCUMULO_HOME=/opt/accumulo-2.1.0' >> ~/.bashrc
echo 'export PATH=$PATH:$ACCUMULO_HOME/bin' >> ~/.bashrc
source ~/.bashrc
```
#### 3-服务启动流程
| 步骤 | 命令 | 说明 |
| ---------------- |---------------- |---------------- |
|1. 初始化系统|$ACCUMULO_HOME/bin/accumulo init|创建系统表和元数据|
|2. 启动服务|$ACCUMULO_HOME/bin/start-all.sh|启动Master、TServer等组件|
|3. 验证状态|$ACCUMULO_HOME/bin/accumulo admin metrics|检查集群健康状态|

#### 4-常见问题处理

##### 4.1 依赖缺失
解决方案：

1.检查Hadoop JAR包是否在Accumulo classpath中

2.手动添加依赖路径：
```
export CLASSPATH=$CLASSPATH:$HADOOP_HOME/share/hadoop/common/*.jar
```
##### 4.2 权限问题
现象：Permission denied错误
解决方案：
```
# 修正Accumulo目录权限
chmod -R 755 $ACCUMULO_HOME

# 检查运行用户是否属于hadoop组
groups $(whoami)
```

