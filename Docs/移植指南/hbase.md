###  **Hbase-2.5.0 移植指南** 

编写日期：2023年03月13日

####  **1. 软硬件环境**
 
##### **1.1 硬件** 	
CPU	鲲鹏920
网络	Ethernet-10GE
存储	SATA 1T
内存	Hynix 512G 2400MHz
##### **1.2 OS** 	
EulerOS	22.03
Kernel	5.10.0-60.18.0.50.oe2203.aarch64
OpenJDK	1.8.0_342
Maven	3.8.6
Hadoop	3.3.4
####  **2.编译环境准备** 
##### **2.1 编译工具安装** 
`yum –y install wget openssl-devel zlib-devel automake libtool make libstdc++-static glibc-static git snappy snappy-devel fuse fuse-devel gcc.aarch64 gcc-c++.aarch64 gcc-gfortran.aarch64 libgcc.aarch64 cmake patch protobuf`
#####  **2.2 安装openjdk** 
安装指南参考链接：https://gitee.com/openeuler/bishengjdk-8/wikis/%E6%AF%95%E6%98%87JDK%208%20%E5%AE%89%E8%A3%85%E6%8C%87%E5%8D%97?sort_id=2891179
#####  **2.3 安装maven**
###### 2.3.1 下载并安装目录

```
下载： wget https://archive.apache.org/dist/maven/maven-3/3.8.6/binaries/apache-maven-3.8.6-bin.tar.gz --no-check-certificate
安装： tar -zxf apache-maven-3.8.6-bin.tar.gz -C /home/installed/
修改maven环境变量，在/etc/profile文件末尾增加下面代码：
export JAVA_HOME=/home/installed/jdk1.8.0_342   // 注意jdk等安装路径
export MAVEN_HOME=/home/installed/apache-maven-3.8.6
export PATH=$MAVEN_HOME/bin:$JAVA_HOME/bin:$PATH
source /etc/profile
```
###### 2.3.2 配置本地仓、远程仓等
```
<!--默认在“~/.m2/”目录下，修改成自定义的目录-->
<localRepository>/path/to/local/repo</localRepository>

<!--修改成自己搭建的maven仓库，ARM使能后的jar包替换到该仓库-->
<mirror>
  <id>huaweimaven</id>
  <name>huawei maven</name>
  <url> http://cmc-cd-mirror.rnd.huawei.com/maven/</url>
  <mirrorOf>central</mirrorOf>
</mirror>
```
#### 3 移植分析
| 原始jar                           | so文件                                      |
|---------------------------------|-------------------------------------------|
| commons-crypto-1.1.0.jar               | libcommons-crypto.so                            |
| jine-2.14.6.jar      | libjansi.so |
| leveldbjni-all-1.8.jar          | libleveldbjni.so                          |
#### 4.编译依赖包
##### 4.1编译commons-crypto-1.1.0.jar 

```
下载源码：wget https://github.com/apache/commons-crypto/archive/refs/tags/rel/commons-crypto-1.1.0.tar.gz --no-check-certificate
解压：tar -zxf commons-crypto-1.1.0.tar.gz
进入目录：cd commons-crypto-rel-commons-crypto-1.1.0
执行编译命令：mvn clean install -DskipTests
```
##### 4.2 编译leveldbjni-all-1.8.jar
1）master分支支持aarch64，编译前需要修改pom.xml文件里面的版本号，也可在windows下载master分支的代码压缩包，再上传到服务器进行编译。
```
vim /etc/profile
在最后一行加入
export GIT_SSL_NO_VERIFY=1
保存并退出

git config --global url."https://".insteadOf git://

git clone git://github.com/fusesource/leveldbjni.git
```
2）进入当前目录
```
cd  leveldbjni
export LEVELDBJNI_HOME=`pwd`
```
3）修改pom.xml文件
注释无关操作系统类型（freebsd64、sunos64-amd64、sunos64-sparcv9、linux64-ppc64le），避免引起编译不通过。
如下：
```
245     <profile>
246       <id>full</id>
247       <modules>
248         <module>leveldbjni-osx</module>
249         <module>leveldbjni-linux32</module>
250         <module>leveldbjni-linux64</module>
251         <!-- module>leveldbjni-freebsd64</module -->
252         <module>leveldbjni-win32</module>
253         <module>leveldbjni-win64</module>
254         <!-- module>leveldbjni-sunos64-amd64</module -->
255         <!-- module>leveldbjni-sunos64-sparcv9</module -->
256         <!-- module>leveldbjni-linux64-ppc64le</module -->
257         <module>leveldbjni-linux64-aarch64</module>
258         <module>leveldbjni-all</module>
259       </modules>
260     </profile>
```
并注释 
```
288
289     <!-- profile>
290       <id>freebsd64</id>
291       <modules>
292         <module>leveldbjni-freebsd64</module>
293       </modules>
294     </profile -->
295
...
315
316     <!-- profile>
317       <id>sunos64-amd64</id>
318       <modules>
319         <module>leveldbjni-sunos64-amd64</module>
320       </modules>
321     </profile>
322
323     <profile>
324       <id>sunos64-sparcv9</id>
325       <modules>
326         <module>leveldbjni-sunos64-sparcv9</module>
327       </modules>
328     </profile>
329
330     <profile>
331       <id>linux64-ppc64le</id>
332       <modules>
333         <module>leveldbjni-linux64-ppc64le</module>
334       </modules>
335     </profile -->
```
第60、61行，将编译的目标module加入leveldbjni-all和linux64模块。
```
 57
 58   <modules>
 59     <module>leveldbjni</module>
 60     <module>leveldbjni-linux64</module>
 61     <module>leveldbjni-all</module>
 62   </modules>
 63
```
4）修改leveldbjni-linux64-aarch64/pom.xml文件，第77行，修改后文件对应内容如下：
```
 76         <configuration>
 77           <platform>aarch64</platform>
 78           <name>leveldbjni</name>
 79           <classified>false</classified>
```
5）修改leveldbjni-all/pom.xml文件
注释无关操作系统类型（freebsd64、sunos64-amd64、sunos64-sparcv9、linux64-ppc64le），避免引起编译不通过。将dependency相关的依赖注释掉，注释内容如下：
```
 75     <!-- dependency>
 76       <groupId>org.fusesource.leveldbjni</groupId>
 77       <artifactId>leveldbjni-freebsd64</artifactId>
 78       <version>1.8</version>
 79       <scope>provided</scope>
 80     </dependency -->
 ...
 93     <!-- dependency>
 94       <groupId>org.fusesource.leveldbjni</groupId>
 95       <artifactId>leveldbjni-sunos64-amd64</artifactId>
 96       <version>1.8</version>
 97       <scope>provided</scope>
 98     </dependency>
 99     <dependency>
100       <groupId>org.fusesource.leveldbjni</groupId>
101       <artifactId>leveldbjni-sunos64-sparcv9</artifactId>
102       <version>1.8</version>
103       <scope>provided</scope>
104     </dependency>
105     <dependency>
106       <groupId>org.fusesource.leveldbjni</groupId>
107       <artifactId>leveldbjni-linux64-ppc64le</artifactId>
108       <version>1.8</version>
109       <scope>provided</scope>
110     </dependency -->
```
将build的NativeCode部分注释掉，并修改部分内容，表明生成的leveldbjni-all-1.8.jar里面对应linux64目录下的so也为aarch64，后续在ambari的nodemanager启动过程中，默认使用的是该目录的so：修改后内容如下：
```
147             <Bundle-NativeCode>
148               META-INF/native/windows32/leveldbjni.dll;osname=Win32;processor=x86,
149               META-INF/native/windows64/leveldbjni.dll;osname=Win32;processor=x86-64,
150               META-INF/native/osx/libleveldbjni.jnilib;osname=macosx;processor=x86,
151               META-INF/native/osx/libleveldbjni.jnilib;osname=macosx;processor=x86-64,
152               META-INF/native/linux32/libleveldbjni.so;osname=Linux;processor=x86,
153               META-INF/native/linux64/libleveldbjni.so;osname=Linux;processor=aarch64,
154               META-INF/native/aarch64/libleveldbjni.so;osname=Linux;processor=aarch64
155             </Bundle-NativeCode>
156 <!--
157                META-INF/native/sunos64/amd64/libleveldbjni.so;osname=SunOS;processor=x86-64,
158                META-INF/native/sunos64/sparcv9/libleveldbjni.so;osname=SunOS;processor=sparcv9,
159                META-INF/native/freebsd64/libleveldbjni.so;osname=FreeBSD;processor=x86-64,
160                META-INF/native/linux64/ppc64le/libleveldbjni.so;osname=Linux;processor=ppc64le,
161 -->
```
6）修改所有pom.xml文件中的版本号，将所有pom.xml文件中的版本“99-master-SNAPSHOT”改为 “1.8”：

find . -name pom.xml | xargs sed -i 's/99-master-SNAPSHOT/1.8/g'  

7）编译
mvn clean package -P download -P linux64-aarch64 -DskipTests

##### 4.3编译jine-2.14.6.jar
依赖关系：jline-2.14.6--> jansi-1.12 --> jansi-native-1.7
###### 4.3.1 编译jansi-native-1.7

```
下载源码:wget https://github.com/fusesource/jansi-native/archive/jansi-native-1.7.tar.gz --no-check-certificate
解压并进入目录：tar zxf jansi-native-1.7.tar.gz
cd jansi-native-jansi-native-1.7
```
修改pom.xml的hawtjni-version：改为1.16与openEuler yum源下载版本对应：vim pom.xml +39
```
39     <hawtjni-version>1.16</hawtjni-version>
```
修改280行 maven-hawtjni-plugin 为 hawtjni-maven-plugin
```
280             <artifactId>hawtjni-maven-plugin</artifactId>
```
执行编译：
```
mvn install -Dplatform=linux64
```
结果在“./target”目录下
###### 4.3.2 编译jansi-1.12
下载Jansi 1.12版本源码，并进入源码目录
```
wget https://github.com/fusesource/jansi/archive/jansi-project-1.12.tar.gz
tar -zxvf jansi-project-1.12.tar.gz
cd jansi-jansi-project-1.12/
```
修改pom.xml里jansi-native的版本为1.7: vim pom.xml +40
```
40     <jansi-native-version>1.7</jansi-native-version>
```
执行编译：
```
mvn install -Dmaven.javadoc.skip=true
```
###### 4.3.3 编译jline-2.14.6
下载jline-2.14.6源码，并解压进入目录
```
wget https://github.com/jline/jline2/archive/refs/tags/jline-2.14.6.tar.gz --no-check-certificate
tar -zxvf jline-2.14.6.tar.gz
cd jline2-jline-2.14.6
```
执行编译：
```
mvn install -Dmaven.javadoc.skip=true -DskipTests
```
#### 5编译hbase-2.5.0
编译命令：
```
mvn clean package -DskipTests assembly:single
```
