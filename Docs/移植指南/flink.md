### flink-1.13.0移植指南
编写日期：2023年03月10日
### 1 简介
  Flink是一个分布式、高性能、高可用的开源大数据处理框架，具有基于内存计算，流式处理等特点，用于对无边界和有边界数据流进行有状态的计算。
### 2 环境要求
#### 2.1 硬件要求
| 项目 | 说明 | 
| ---------------- |---------------- |
|服务器|对服务器无要求|
|CPU|aarch64架构|
|磁盘分区|对磁盘分区无要求|
|网络|可访问外网|
#### 2.2 软件要求
| 项目 | 说明 | 
| ---------------- |---------------- |
|版本|openeuler 22.03|
|OS|5.10.0-60.65.0.90.oe2203.aarch64|
|GCC|10.3.1|
|OpenJDK|1.8.0_342|
|Maven|3.6.3|
|Flink|1.13.0|
### 3 安装基础库
#### 3.1 安装gcc maven jdk-1.8.0
yum -y install gcc.aarch64 gcc-c++.aarch64 gcc-gfortran.aarch64 libgcc.aarch64 maven java-1.8.0-openjdk.aarch64 java-1.8.0-openjdk-devel.aarch64 
#### 3.2 安装依赖
yum -y install gcc gcc-c++ gcc-gfortran.aarch64 libgcc.aarch64 make cmake libtool autoconf automake ant wget git vim clang openssl openssl-devel golang 
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
### 4 移植指南分析
#### 4.1 软件移植分析结果
|原始jar包|so文件|
| ---------------- |---------------- |
|scala-compiler-2.11.12.jar | libjansi.so
|flink-shaded-netty-4.1.49.Final-13.0.jar | liborg_apache_flink_shaded_netty4_netty_transport_native_epoll_x86_64.so|
|frocksdbjni-5.17.2-artisans-2.0.jar | librocksdbjni32.so, librocksdbjni64.so, librocksdbjnile.so|
|flink-shaded-netty-tcnative-dynamic-2.0.30.Final-13.0.jar | liborg_apache_flink_shaded_netty4_netty_tcnative_linux_x86_64.so, liborg_apache_flink_shaded_netty4_netty_tcnative_linux_x86_64_fedora.so|
|conscrypt-openjdk-uber-1.3.0.jar | libconscrypt_openjdk_jni-linux-x86_64.so
|beam-vendor-grpc-1_26_0-0.3.jar | liborg_apache_beam_vendor_grpc_v1p26p0_netty_tcnative_linux_x86_64.so|
### 5 依赖库编译
#### 5.1 scala-compiler-2.11.12.jar
scala-compiler-2.11.12.jar aarch64下载地址：https://mirrors.huaweicloud.com/kunpeng/maven/org/scala-lang/scala-compiler/2.11.12/scala-compiler-2.11.12.jar
#### 5.2 frocksdbjni-5.17.2-artisans-2.0.jar
frocksdbjni-5.17.2-artisans-2.0.jar aarch64下载地址：https://mirrors.huaweicloud.com/kunpeng/maven/com/data-artisans/frocksdbjni/5.17.2-artisans-2.0/frocksdbjni-5.17.2-artisans-2.0.jar
#### 5.3 beam-vendor-grpc-1_26_0-0.3.jar
##### 5.3.1 安装gradle-5.4.1
```
wget https://services.gradle.org/distributions/gradle-5.4.1-bin.zip --no-check-certificate
unzip gradle-5.4.1-bin.zip
export PATH=`pwd`/gradle-5.4.1/bin:$PATH
```
##### 5.3.2 移植beam-vendor-grpc
```
wget https://github.com/apache/beam/archive/v2.19.0.tar.gz -O beam-v2.19.0.tar.gz
tar -zxf beam-v2.19.0.tar.gz
```
##### 5.3.3 进入beam目录
```
cd beam-2.19.0/vendor/grpc-1_26_0/
vim build.gradle
version = "0.3"

repositories {
    maven { url "https://mirrors.huaweicloud.com/kunpeng/maven/" }
    mavenLocal()
    maven { url "https://mirrors.huaweicloud.com/repository/maven/"}
}
```
##### 5.3.3执行编译
```
gradle build
编译好的beam-vendor-grpc-1_26_0-0.3.jar在build/libs目录下
请将编译完成的jar包替换到/.../org/apache/beam/beam-vendor-grpc-1_26_0/0.3/自己搭建的Maven仓库中。
```
#### 5.4 conscrypt-openjdk-uber-1.3.0.jar
##### 5.4.1 准备conscrypt编译环境
```
步骤1 安装cmake，需要（3.0及以上），可使用yum安装，cmake –version查看版本。
步骤2 安装ninja。
1.下载ninja，并解压。
wget https://github.com/ninja-build/ninja/archive/v1.10.0.zip
unzip v1.10.0.zip
2.定位到目录ninja-1.10.0。
cd ninja-1.10.0 
3.编译ninja。
./configure.py --bootstrap
4.将里面的ninja 文件复制到/usr/bin目录下。
cp ninja /usr/bin
步骤3 配置go环境
1.下载，并解压。
cd /usr/local/
wget https://golang.org/dl/go1.16.6.linux-arm64.tar.gz
tar –xf go1.16.6.linux-arm64.tar.gz
2.配置环境变量，并生效。
export GOROOT=/usr/local/go
export PATH=$PATH:$GOROOT/bin 
source /etc/profile
步骤4 构建boringssl环境
1.下载boringssl源码，并解压。
wget https://github.com/google/boringssl/archive/master.zip
unzip master.zip
mv boringssl-master boringssl
2.配置环境变量。
vim /etc/profile 
export BORINGSSL_HOME=/home/boringssl
source /etc/profile
3.进入boringssl 目录。
cd boringssl
构建64位版本和32位版本。因为系统架构为aarch64,可以不构建32位版本。
64位版本：  
mkdir build64
cd build64
cmake -DCMAKE_POSITION_INDEPENDENT_CODE=TRUE \
      -DCMAKE_BUILD_TYPE=Release \
      -DCMAKE_ASM_FLAGS=-Wa,--noexecstack \
      -GNinja ..
ninja
注: 
    1. 找不到libwind，yum安装libunwind-dev。
    2. yum golang下载失败，可手动安装go，并配置环境变量。
    3. ninja时，go：golang get 报x509：certificate错误，可以配置。
export GOPROXY=http://mirrors.tools.huawei.com/goproxy/
export GO111MODULE=on
32 位版本：
mkdir build32
cd build32
cmake -DCMAKE_TOOLCHAIN_FILE=../util/32-bit-toolchain.cmake \
      -DCMAKE_POSITION_INDEPENDENT_CODE=TRUE \
      -DCMAKE_BUILD_TYPE=Release \
      -DCMAKE_ASM_FLAGS="-Wa,--noexecstack -m32 -msse2" \
      -GNinja ..
ninja 
```
##### 5.4.2编译gradle源码
```
1．定位到/home 目录。
cd /home
2．查看conscrypt编译对应的gradle版本。
进入conscrypt目录。
vim /home/conscript/gradle/wrapper/gradle-wrapper.properties
3．下载gradle-v4.8.1源码 。
wget https://github.com/gradle/gradle/archive/v4.8.1.zip
解压v4.8.1.zip。
unzip v4.8.1.zip
重命名为gradle，并定位到gradle目录。
mv gradle-4.8.1 gradle && cd gradle
4．编辑NativePlatforms.java 
vim subprojects/platform-native/src/main/java/org/gradle/nativeplatform/platform/internal/NativePlatforms.java
在44行后面增加一行代码。
ArchitectureInternal aarch64 = Architectures.forInput("aarch64");
在73行后面增加一行代码。
 platforms.add(createPlatform(linux, aarch64));
5．gradle构建。
./gradlew
运行完成后/home/gradle/subprojects/platform-native/build/libs 下会生成gradle-platform-native-4.8.1.jar。
注：1. 这里UnknownHostException问题，请看问题1 gradle代理配置解决。
    2. 出现Reason: invalid distance too far back错误，删除/root/.gradle/wrapper/下目录，重新运行./gradlew即可
6．定位到conscrypt目录。
cd /home/conscript
运行构建命令。
./gradlew 
会自动下载gradle-4.8.1-all，gradle的默认的缓存目录：/root/.gradle/wrapper/dists。
7．定位到下载的gradle 中删除gradle-platform-native-4.8.1.jar
cd  /root/.gradle/wrapper/dists/gradle-4.8.1-all/6fmj4nezasjg1b7kkmy10xgo2/gradle-
4.8.1/lib/plugins ; 
rm -rf gradle-platform-native-4.8.1.jar;
将步骤3生成的jar包替换
cp /home/gradle-4.8.1/subprojects/platform-native/build/libs/gradle-platform-native-4.8.1.jar  ./
8．重启电脑
Reboot
```
##### 5.4.4编译conscrypt
```
1.下载conscrypt源码
定位到/home目录。
cd /home
下载源码，并解压
wget https://github.com/google/conscrypt/archive/1.3.0.zip --no-check-certificate
unzip 1.3.0.zip
mv conscrypt-1.3.0 conscrypt && cd /conscript
2.编译conscrypt。
步骤 1	定位到openjdk目录
cd openjdk
步骤 2	修改build.gradle文件 
vim build.gradle
1.10行增加arm64 = 'aarch64' 
2.修改19行nativeClassifier64Bit = classifierFor(osName, arm64)
3.在第280行增加platform配置。
    linux_aarch64 {
             architecture "aarch64"
             operatingSystem "linux"
        }  
4.将构建jni-library 时64位架构改为“linux_aarch64”。
注释第293行，修改第294行arch64Name为"linux_aarch64"
修改315行修改为aarch64。
5.Clang 或gcc编译时args增加指定头文件地址。(10.3.0为gcc版本，根据自己gcc版本配置)
"-I$jniSourceDir/main/include",
"-I$jniSourceDir/unbundled/include",
"-I/usr/include/c++/10.3.0",
"-I/usr/include/c++/10.3.0/aarch64-redhat-linux ",
修改生成库信息代码。
def archName = binary.targetPlatform.architecture.name
def archNameR = archName
if(archName == 'aarch64'){
 archNameR = 'aarch_64'
}
archName = archName.replaceAll('-','_')
def classifier = classifierFor(osName, archName)
def classifierR = classifierFor(osName,archNameR)
def sourceSetName = sourceSetName("$classifier")
def source = binary.sharedLibraryFile
修改第419行和第421行内容为classifierR
完成后单击“Ecs”，输入:wq保存退出。
6.返回上一级目录。修改“conscrypt”根目录下的build.gradle文件
vi build.gradle
修改改第39行，并增加40行-43行，构建aarch通道。
     gcc(Gcc){
                  target("linux_aarch64") {
                  cppCompiler.executable = "/usr/bin/gcc"
                  }
                }
在第172-174行增加代码。
dependencies { 
errorprone("com.google.errorprone:error_prone_core:2.3.3") 
}
完成后单击“Ecs”，输入:wq保存退出。
7.定位到conscrypt目录。
cd /home/conscrypt
8.构建。
./gradlew clean build --stacktrace -x :conscrypt-openjdk:linux_aarch64Test -Dorg.conscrypt.openjdk.buildUberJar=true
注：1.如找不到linux_aarch64Test，则使用./gradlew clean build，跳过测试。
    2.构建找不到conscrypt_openjdk_jni-linux-aarch_64，可在根目录，find一下，将找到的so文件按照要求放入对应目录下，比如/usr/lib。
    3.Tset构建失败，注释build.gradle下Test类。
9.定位到openjdk-uber/buil/libs目录。
cd /openjdk-uber/build/libs
cp /home/conscrypt-1.3.0/openjdk/build/linux_aarch_64/native-resources/META-INF/native/libconscrypt_openjdk_jni-linux-aarch_64.so
10.解压conscrypt-openjdk-uber-1.3.0.jar。
jar xvf conscrypt-openjdk-uber-1.3.0.jar
11.将/home/conscrypt/openjdk/build/linux_aarch_64/native-resources/META-INF/native\ 目录下的libconscrypt_openjdk_jni-linux-aarch_64.so 放入conscrypt-open-uber-1.3.0.jar 中的META-INF/native 下。
cp /home/conscrypt/openjdk/build/linux_aarch_64/native-resources/META-INF/native\ /libconscrypt_openjdk_jni-linux-aarch_64.so META-INF/native
12.jar打包，移植结束。
jar cvf conscrypt-openjdk-uber-1.3.0.jar META-INF/ org 
将编译好的jar包放入/root/.gradle/caches/modules-2/files-2.1/org.conscrypt/conscrypt-openjdk-uber
```
#### 5.5 flink-shaded-netty
##### 5.5.1 编译安装apr-1.6.5
```
1.下载apr-1.6.5源码。
wget https://archive.apache.org/dist/apr/apr-1.6.5.tar.gz
2.解压后编译安装。
tar -zxvf apr-1.6.5.tar.gz
cd apr-1.6.5
./configure
make
make install
注：如./configure 报错：apr-1.7.0 rm: cannot remove 'libtoolT': No such file or directory
则vim configure +30392 ，修改RM='$RM'修改为RM='$RM -f'
30390     PACKAGE='$PACKAGE'
30391     VERSION='$VERSION'
30392     RM='$RM -rf'
30393     ofile='$ofile'

```
##### 5.5.2 编译安装netty-tcnative-parent-2.0.30.Final
```
1.下载netty-tcnative-parent-2.0.30.Final源码。
wget https://codeload.github.com/netty/netty-tcnative/tar.gz/netty-tcnative-parent-2.0.30.Final
mv netty-tcnative-parent-2.0.30.Final netty-tcnative-parent-2.0.30.Final.tar.gz
2.解压源码包。
tar -zxvf netty-tcnative-parent-2.0.30.Final.tar.gz
3.进入解压后目录。
cd netty-tcnative-netty-tcnative-parent-2.0.30.Final
4.修改pom.xml，屏蔽boringssl-static的编译。
vim pom.xml +603
601         <module>openssl-dynamic</module>
602         <module>openssl-static</module>
603         <!--module>boringssl-static</module-->
604         <!--module>libressl-static</module-->
5.编译打包到maven本地仓库。
mvn install
```
##### 5.5.3 编译安装netty-all-4.1.49.Final
```
1.下载netty-4.1.49.Final源码。
wget https://github.com/netty/netty/archive/netty-4.1.49.Final.tar.gz
2.解压源码包。
tar -zxvf netty-4.1.49.Final.tar.gz
3.进入netty解压目录。
cd netty-netty-4.1.49.Final
4.编译打成jar包，netty-all-4.1.49Final.jar放置于“all/target”目录。
mvn install -DskipTests
```
##### 5.5.4 flink-shaded-netty-4.1.49.Final-13.0编译
```
1.下载flink-shaded-release-13.0安装包。
wget https://codeload.github.com/apache/flink-shaded/tar.gz/release-13.0
2.解压安装包。
mv release-13.0 flink-shaded-release-13.0.tar.gz
tar -zxvf flink-shaded-release-13.0.tar.gz
3.进入解压后的目录。
cd flink-shaded-release-13.0
4.修改pom.xml
因为只需单独编译打包flink-shaded-netty-4，注释掉其余不需要的module。
vim pom.xml +58
 58     <modules>
 59         <!--module>flink-shaded-force-shading</module-->
 60         <!--module>flink-shaded-asm-7</module-->
 61         <!--module>flink-shaded-guava-18</module-->
 62         <module>flink-shaded-netty-4</module>
 63         <!--module>flink-shaded-netty-tcnative-dynamic</module-->
 64         <!--module>flink-shaded-jackson-parent</module-->
 65         <!--module>flink-shaded-zookeeper-parent</module-->
 66     </modules>
5.编译打成jar包。
mvn install package –DskipTests
flink-shaded-netty-4.1.49.Final-13.0.jar放置于“flink-shaded-netty-4/target/”目录。
将编译好的jar包替换本地仓库中
```
##### 5.5.5 flink-shaded-netty-tcnative-dynamic-2.0.30.Final-13.0.jar
```
1.进入flink shaded-release-13.0
cd flink-shaded-release-13.0
2.修改“flink-shaded-release-13.0/pom.xml”。因为只需单独编译打包flink-shaded-netty-tcnative-dynamic，注释掉其余不需要的module。
vim pom.xml
    <modules>
        <!--module>flink-shaded-force-shading</module-->
        <!--module>flink-shaded-asm-7</module-->
        <!--module>flink-shaded-guava-18</module-->
        <!--module>flink-shaded-netty-4</module-->
        <module>flink-shaded-netty-tcnative-dynamic</module>
        <!--module>flink-shaded-jackson-parent</module-->
        <!--module>flink-shaded-zookeeper-parent</module-->
    </modules>
3.修改“flink-shaded-release-13.0/flink-shaded-netty-tcnative-dynamic/pom.xml”。
vim flink-shaded-netty-tcnative-dynamic/pom.xml 在第94行下方插入如下代码。
<artifactItem>
    <groupId>io.netty</groupId>
    <artifactId>netty-tcnative</artifactId>
    <version>${netty.tcnative.version}</version>
    <classifier>linux-aarch_64</classifier>
    <type>jar</type>
    <overWrite>false</overWrite>
<outputDirectory>${project.build.directory}/native_libs</outputDirectory>
</artifactItem>

添加完成之后，在第156行下方插入如下代码。
<unzip src="${project.build.directory}/native_libs/netty-tcnative-${netty.tcnative.version}-linux-aarch_64.jar" dest="${project.build.directory}/native_libs/unpacked/" />
<move todir="${project.build.directory}/unpacked/META-INF/native" includeemptydirs="false">
    <fileset dir="${project.build.directory}/native_libs/unpacked/META-INF/native"/>
    <mapper type="glob" from="libnetty_tcnative.so" to="liborg_apache_flink_shaded_netty4_netty_tcnative_linux_aarch_64.so"/>
</move>
<delete dir="${project.build.directory}/native_libs/unpacked/" />
<unzip src="${project.build.directory}/native_libs/netty-tcnative-${netty.tcnative.version}-linux-aarch_64.jar" dest="${project.build.directory}/native_libs/unpacked/" />
<move todir="${project.build.directory}/unpacked/META-INF/native" includeemptydirs="false">
    <fileset dir="${project.build.directory}/native_libs/unpacked/META-INF/native"/>
    <mapper type="glob" from="libnetty_tcnative.so" to="liborg_apache_flink_shaded_netty4_netty_tcnative.so"/>
</move>
<delete dir="${project.build.directory}/native_libs/unpacked/" />
<unzip src="${project.build.directory}/native_libs/netty-tcnative-${netty.tcnative.version}-linux-aarch_64.jar" dest="${project.build.directory}/native_libs/unpacked/" />
<move todir="${project.build.directory}/unpacked/META-INF/native" includeemptydirs="false">
    <fileset dir="${project.build.directory}/native_libs/unpacked/META-INF/native"/>
    <mapper type="glob" from="libnetty_tcnative.so" to="liborg_apache_flink_shaded_netty4_netty_tcnative_linux_aarch_64.so"/>
</move>
<delete dir="${project.build.directory}/native_libs/unpacked/" />
 
注释掉第175-第198行。
<!--
<unzip src="${project.build.directory}/${artifactId}-${version}.jar" dest="${project.build.directory}/unpacked/" />
<echo message="extracting netty_tcnative dynamically linked wrapper libraries" />

<unzip src="${project.build.directory}/native_libs/netty-tcnative-${netty.tcnative.version}-linux-x86_64.jar" dest="${project.build.directory}/native_libs/unpacked/" />
<move todir="${project.build.directory}/unpacked/META-INF/native" includeemptydirs="false">
    <fileset dir="${project.build.directory}/native_libs/unpacked/META-INF/native"/>
    <mapper type="glob" from="libnetty_tcnative.so" to="liborg_apache_flink_shaded_netty4_netty_tcnative_linux_x86_64.so"/>
</move>
<delete dir="${project.build.directory}/native_libs/unpacked/" />
<unzip src="${project.build.directory}/native_libs/netty-tcnative-${netty.tcnative.version}-linux-x86_64-fedora.jar" dest="${project.build.directory}/native_libs/unpacked/" />
<move todir="${project.build.directory}/unpacked/META-INF/native" includeemptydirs="false">
    <fileset dir="${project.build.directory}/native_libs/unpacked/META-INF/native"/>
    <mapper type="glob" from="libnetty_tcnative.so" to="liborg_apache_flink_shaded_netty4_netty_tcnative_linux_x86_64_fedora.so"/>
</move>
<delete dir="${project.build.directory}/native_libs/unpacked/" />
<unzip src="${project.build.directory}/native_libs/netty-tcnative-${netty.tcnative.version}-osx-x86_64.jar" dest="${project.build.directory}/native_libs/unpacked/" />
<move todir="${project.build.directory}/unpacked/META-INF/native" includeemptydirs="false">
    <fileset dir="${project.build.directory}/native_libs/unpacked/META-INF/native"/>
    <mapper type="glob" from="libnetty_tcnative.jnilib" to="liborg_apache_flink_shaded_netty4_netty_tcnative_osx_x86_64.jnilib"/>
</move>
<delete dir="${project.build.directory}/native_libs/unpacked/" />
<unzip src="${project.build.directory}/native_libs/netty-tcnative-${netty.tcnative.version}-windows-x86_64.jar" dest="${project.build.directory}/native_libs/unpacked/" />
<move todir="${project.build.directory}/unpacked/META-INF/native" includeemptydirs="false">
    <fileset dir="${project.build.directory}/native_libs/unpacked/META-INF/native"/>
    <mapper type="glob" from="netty_tcnative.dll" to="org_apache_flink_shaded_netty4_netty_tcnative_windows_x86_64.dll"/>
</move>
<delete dir="${project.build.directory}/native_libs/unpacked/" />
--> 
4.编译打包到maven本地仓库。
mvn install
编译好的jar包在flink-shaded-netty-tcnative-dynamic/target目录下，并将编译完成的jar包替换到本地仓库中。
```
### 6 Flink-1.3.0编译
#### 6.1 从github上下载flink-release-1.13.1源码并解压：
```
wget https://github.com/apache/flink/archive/release-1.3.0.tar.gz
tar -zxf flink-release-1.13.0.tar.gz
cd flink-release-1.13.0
```
#### 6.2 修改vim flink-runtime-web/pom.xml
说明：国内访问外网地址，由于网络限制或者网速问题，会出现下载报错或者长时间卡住，更换为国内地址。
```
@@ -271,6 +271,7 @@ under the License.
    </goals>
    <configuration>
        <arguments>ci --cache-max=0 --no-save</arguments>
+	<npmRegistryURL>https://mirrors.huaweicloud.com/repository/npm</npmRegistryURL>
 	<environmentVariables>
 	    <HUSKY_SKIP_INSTALL>true</HUSKY_SKIP_INSTALL>
 	</environmentVariables>
@@ -283,6 +284,7 @@ under the License.
    </goals>
    <configuration>
        <arguments>run build</arguments>
+	<npmRegistryURL>https://mirrors.huaweicloud.com/repository/npm</npmRegistryURL>
    </configuration>
  </execution>
</executions>
@@ -262,6 +262,7 @@ under the License.
    </goals>
    <configuration>
 	<nodeVersion>v10.9.0</nodeVersion>
+       <downloadRoot>https://mirrors.huaweicloud.com/nodejs/</downloadRoot>
    </configuration>
 </execution>
<execution>
```
#### 6.3 执行编译：
```
mvn clean install -DskipTests 
多线程编译可以如下命令：
mvn clean install -DskipTests -Dfast -T 32 -Dmaven.compile.fork=true
如flink-runtime-web_2.11编译报错，可使用如下命令继续编译，节约时间。
mvn clean install -DskipTests -rf :flink-runtime-web_2.11
```
#### 6.4 编译结果
编译完成后在flink-release-1.13.0/flink-dist/target/flink-1.13.0-bin/生成目标目录flink-1.13.0。
#### 6.5 使用鲲鹏分析扫描工具扫描编译生成的tar包，确保没有包含有x86的so和jar包。
如果有，则参考4，将jar包替换本地仓库，并重新编译flink。	



