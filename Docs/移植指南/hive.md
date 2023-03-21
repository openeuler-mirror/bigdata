###  **hive-3.1.3 移植指南** 

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
| commons-crypto-1.1.0.jar                   | libcommons-crypto.so                            |
| jline-2.12.jar      | libjansi.so |
| netty-all-4.1.17.Final.jar           | libnetty_transport_native_epoll_x86_64.so                            |

#### 4 编译依赖包
##### 4.1 编译commons-crypto-1.1.0.jar
```
下载源码：wget https://github.com/apache/commons-crypto/archive/refs/tags/rel/commons-crypto-1.1.0.tar.gz --no-check-certificate
解压：tar -zxf commons-crypto-1.1.0.tar.gz
进入目录：cd commons-crypto-rel-commons-crypto-1.1.0
执行编译命令：mvn clean install -DskipTests
```
##### 4.2 编译jline-2.12.jar
依赖关系：jline-2.12 --> jansi-1.12 --> jansi-native-1.7
###### 4.2.1 编译jansi-native-1.7

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

###### 4.2.2 编译jansi-1.12
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
###### 4.3.3 编译jline-2.12
下载jline-2.12源码，并解压进入目录
```
wget https://github.com/jline/jline2/archive/refs/tags/jline-2.12.tar.gz --no-check-certificate
tar -zxvf jline-2.12.tar.gz
cd jline2-jline-2.12
```
修改pom.xml文件第120行jansi.version的版本1.11为1.12
```
120         <jansi.version>1.12</jansi.version>
```

执行编译：
```
mvn install -Dmaven.javadoc.skip=true -DskipTests
```

##### 4.3 netty-all-4.1.17.Final.jar 移植到本地仓对应目录io/netty/netty-all/4.1.17.Final/下
可直接从鲲鹏镜像仓拉取：
```
wget https://repo.huaweicloud.com/kunpeng/maven/io/netty/netty-all/4.0.17.Final/netty-all-4.0.17.Final.jar https://repo.huaweicloud.com/kunpeng/maven/io/netty/netty-all/4.0.17.Final/netty-all-4.0.17.Final.jar.sha1 https://repo.huaweicloud.com/kunpeng/maven/io/netty/netty-all/4.0.17.Final/netty-all-4.0.17.Final.pom https://repo.huaweicloud.com/kunpeng/maven/io/netty/netty-all/4.0.17.Final/netty-all-4.0.17.Final.pom.sha1 --no-check-certificate
```


#### 5 编译Hive-3.1.3
##### 5.1 提升hive的guave版本
原因：集群中所安装的Hadoop-3.3.4中和Hive-3.1.3中包含guava的依赖，Hadoop-3.3.4中的版本为guava-27.0-jre，而Hive-3.1.3中的版本为guava-19.0。由于Hive运行时会加载Hadoop依赖，故会出现依赖冲突的问题。

修改pom.xml文件

将pom.xml文件中147行的  <guava.version>19.0</guava.version>修改为<guava.version>27.0-jre</guava.version>
##### 5.2 无法推断类型变量V问题解决
Error:Failed to execute goal org.apache.maven.plugins:maven-compiler-plugin:3.6.1:compile (default-compile) on project hive-llap-common: Compilation failure: Compilation failure

原因：无法推断类型变量V (实际参数列表和形式参数列表长度不同)

解决：修改下列源码
1) vim druid-handler/src/java/org/apache/hadoop/hive/druid/serde/DruidScanQueryRecordReader.java 
```
28   import com.google.common.collect.Iterators;
29   
30   import java.io.IOException;
31 + import java.util.Collections;
32   import java.util.Iterator;
33   import java.util.List;
```

```
44
45   private ScanResultValue current;
46
47 - private Iterator<List<Object>> compactedValues = Iterators.emptyIterator();
48 + private Iterator<List<Object>> compactedValues = Collections.emptyIterator();
49
50   @Override
51   protected JavaType getResultTypeDef() {
```
2) vim llap-server/src/java/org/apache/hadoop/hive/llap/daemon/impl/AMReporter.java


```
174          Thread.getDefaultUncaughtExceptionHandler().uncaughtException(Thread.currentThread(), t);
175        }
176      }
177 -  });
177 +  },MoreExecutors.directExecutor());
178    // TODO: why is this needed? we could just save the host and port?
179    nodeId = LlapNodeId.getInstance(localAddress.get().getHostName(), localAddress.get().getPort());
180    LOG.info("AMReporter running with DaemonId: {}, NodeId: {}", daemonId, nodeId);

...

274         LOG.warn("Failed to send taskKilled for {}. The attempt will likely time out.",
275             taskAttemptId);
276       }
277 -   });
277 +   },MoreExecutors.directExecutor());
278   }
279
280   public void queryComplete(QueryIdentifier queryIdentifier) {

...

342                     amNodeInfo.amNodeId, currentQueryIdentifier, t);
343                   queryFailedHandler.queryFailed(currentQueryIdentifier);
344                }
345  -           });
345  +           },MoreExecutors.directExecutor());
346            }
347          }
348        } catch (InterruptedException e) {

```
3) vim llap-server/src/java/org/apache/hadoop/hive/llap/daemon/impl/LlapTaskReporter.java
```
128         sendCounterInterval, maxEventsToGet, requestCounter, containerIdStr, initialEvent,
129         fragmentRequestId, wmCounters);
130     ListenableFuture<Boolean> future = heartbeatExecutor.submit(currentCallable);
131 -   Futures.addCallback(future, new HeartbeatCallback(errorReporter));
131 +   Futures.addCallback(future, new HeartbeatCallback(errorReporter),MoreExecutors.directExecutor());
132   }
```
4) vim llap-server/src/java/org/apache/hadoop/hive/llap/daemon/impl/TaskExecutorService.java
```
175     executionCompletionExecutorService = MoreExecutors.listeningDecorator(
176        executionCompletionExecutorServiceRaw);
177     ListenableFuture<?> future = waitQueueExecutorService.submit(new WaitQueueWorker());
178 -   Futures.addCallback(future, new WaitQueueWorkerCallback());
178 +   Futures.addCallback(future, new WaitQueueWorkerCallback(),MoreExecutors.directExecutor());
179   }
180 
181   private LlapQueueComparatorBase createComparator(
```
5) vim ql/src/test/org/apache/hadoop/hive/ql/exec/tez/SampleTezSessionState.java
```
19 package org.apache.hadoop.hive.ql.exec.tez;
20 
21 - import com.google.common.util.concurrent.Futures;
   - import com.google.common.util.concurrent.FutureCallback;
   - import com.google.common.util.concurrent.ListenableFuture;
   - import com.google.common.util.concurrent.SettableFuture;
22 + import com.google.common.util.concurrent.*;
23
24   import java.io.IOException;
25   import java.util.concurrent.ScheduledExecutorService;
26   import javax.security.auth.login.LoginException;

...

126      public void onFailure(Throwable t) {
127        future.setException(t);
128      }
    -  });
129 +  }, MoreExecutors.directExecutor());
130    return future;
131  }
132
```
6) vim ql/src/java/org/apache/hadoop/hive/ql/exec/tez/WorkloadManager.java 
```
18   package org.apache.hadoop.hive.ql.exec.tez;
19
20 + import com.google.common.util.concurrent.*;
21   import org.apache.hadoop.hive.metastore.api.WMPoolSchedulingPolicy;
22   import org.apache.hadoop.hive.metastore.utils.MetaStoreUtils;
23
24   import com.google.common.annotations.VisibleForTesting;
25   import com.google.common.collect.Lists;
26   import com.google.common.collect.Sets;
27   import com.google.common.math.DoubleMath;
   - import com.google.common.util.concurrent.FutureCallback;
   - import com.google.common.util.concurrent.Futures;
   - import com.google.common.util.concurrent.ListenableFuture;
   - import com.google.common.util.concurrent.SettableFuture;
   - import com.google.common.util.concurrent.ThreadFactoryBuilder;
28
29   import java.util.ArrayList;
30   import java.util.Collection;

...

1088   }
1089 
1090   private void failOnFutureFailure(ListenableFuture<?> future) {
     -     Futures.addCallback(future, FATAL_ERROR_CALLBACK);
1091 +     Futures.addCallback(future, FATAL_ERROR_CALLBACK, MoreExecutors.directExecutor());
1092   }
1093   
1094   private void queueGetRequestOnMasterThread(

...

1921 
1922     public void start() throws Exception {
1923       ListenableFuture<WmTezSession> getFuture = tezAmPool.getSessionAsync();
     -     Futures.addCallback(getFuture, this);
1924 +     Futures.addCallback(getFuture, this,MoreExecutors.directExecutor());
1925     }
1926
1927     @Override

...

1975       case GETTING: {
1976         ListenableFuture<WmTezSession> waitFuture = session.waitForAmRegistryAsync(
1977             amRegistryTimeoutMs, timeoutPool);
      -      Futures.addCallback(waitFuture, this);
1978  +      Futures.addCallback(waitFuture, this,MoreExecutors.directExecutor());
1979         break;
1980       }
1981       case WAITING_FOR_REGISTRY: {
```
7) vim llap-tez/src/java/org/apache/hadoop/hive/llap/tezplugins/LlapTaskSchedulerService.java
```
744       }, 10000L, TimeUnit.MILLISECONDS);
745 
746       nodeEnablerFuture = nodeEnabledExecutor.submit(nodeEnablerCallable);
    -     Futures.addCallback(nodeEnablerFuture, new LoggingFutureCallback("NodeEnablerThread", LOG));
747 +     Futures.addCallback(nodeEnablerFuture, new LoggingFutureCallback("NodeEnablerThread", LOG),MoreExecutors.directExecutor());
748 
749       delayedTaskSchedulerFuture =
750           delayedTaskSchedulerExecutor.submit(delayedTaskSchedulerCallable);
751       Futures.addCallback(delayedTaskSchedulerFuture,
    -         new LoggingFutureCallback("DelayedTaskSchedulerThread", LOG));
752 +         new LoggingFutureCallback("DelayedTaskSchedulerThread", LOG),MoreExecutors.directExecutor());
753 
754       schedulerFuture = schedulerExecutor.submit(schedulerCallable);
    -     Futures.addCallback(schedulerFuture, new LoggingFutureCallback("SchedulerThread", LOG));
755 +     Futures.addCallback(schedulerFuture, new LoggingFutureCallback("SchedulerThread", LOG),MoreExecutors.directExecutor());
756
757       registry.start();
758       registry.registerStateChangeListener(new NodeStateChangeListener());

```
8) vim llap-common/src/java/org/apache/hadoop/hive/llap/AsyncPbRpcProxy.java
```
171           CallableRequest<T, U> request, LlapNodeId nodeId) {
172         ListenableFuture<U> future = executor.submit(request);
173         Futures.addCallback(future, new ResponseCallback<U>(
    -           request.getCallback(), nodeId, this));
174 +           request.getCallback(), nodeId, this),MoreExecutors.directExecutor());
175       }
176 
177       @VisibleForTesting

...

283           LOG.warn("RequestManager shutdown with error", t);
284         }
285       }
    -   });
286 +   },MoreExecutors.directExecutor());
287   }
288 
289   @Override
```
##### 5.3 整合spark版本，以下以整合spark3.3.0为例
1）修改hive源码目录下pom.xml文件第201开始：
```
<spark.version>3.3.0</spark.version>
<scala.binary.version>2.12</scala.binary.version>
<scala.version>2.12.13</scala.version>
```
2）vim ql/src/test/org/apache/hadoop/hive/ql/stats/TestStatsUtils.java
```
31   import org.apache.hadoop.hive.ql.plan.ColStatistics.Range;
32   import org.apache.hadoop.hive.serde.serdeConstants;
33   import org.junit.Test;
   - import org.spark_project.guava.collect.Sets;
34 + import org.sparkproject.guava.collect.Sets;
35  
36  public class TestStatsUtils {
```
3) vim spark-client/src/main/java/org/apache/hive/spark/client/metrics/ShuffleWriteMetrics.java
```
47     }
48   
49     public ShuffleWriteMetrics(TaskMetrics metrics) {
   -     this(metrics.shuffleWriteMetrics().shuffleBytesWritten(),
   -       metrics.shuffleWriteMetrics().shuffleWriteTime());
50 +     this(metrics.shuffleWriteMetrics().bytesWritten(),
51 +       metrics.shuffleWriteMetrics().bytesWritten());
52     }
53   
54   }

```
4) vim spark-client/src/main/java/org/apache/hive/spark/counter/SparkCounter.java
```
19   
20    import java.io.Serializable;
21   
   -  import org.apache.spark.Accumulator;
   -  import org.apache.spark.AccumulatorParam;
22    import org.apache.spark.api.java.JavaSparkContext;
23 +  import org.apache.spark.util.LongAccumulator;
24    
25    public class SparkCounter implements Serializable {
26    
27      private String name;
28      private String displayName;
   -    private Accumulator<Long> accumulator;
29 +    private LongAccumulator accumulator;
30    
31      // Values of accumulators can only be read on the SparkContext side. This field is used when
32      // creating a snapshot to be sent to the RSC client.

...

54    
55        this.name = name;
56        this.displayName = displayName;
   -      LongAccumulatorParam longParam = new LongAccumulatorParam();
57 +      String accumulatorName = groupName + "_" + name; 
   -      this.accumulator = sparkContext.accumulator(initValue, accumulatorName, longParam);
58 +      this.accumulator = sparkContext.sc().longAccumulator(accumulatorName);
59      }
60    
61      public long getValue() {

...
86    
87        return new SparkCounter(name, displayName, accumulator.value());
88      }
   -  
   -    class LongAccumulatorParam implements AccumulatorParam<Long> {
   -  
   -       @Override
   -       public Long addAccumulator(Long t1, Long t2) {
   -         return t1 + t2;
   -       }
   -
   -       @Override
   -       public Long addInPlace(Long r1, Long r2) {
   -         return r1 + r2;
   -       }
   -
   -       @Override
   -       public Long zero(Long initialValue) {
   -         return 0L;
   -       }
   -    }
89    
90    }
```
##### 5.4 修复插入数据的bug
源码按照链接修复16个类：https://github.com/gitlbo/hive/commit/c073e71ef43699b7aa68cad7c69a2e8f487089fd

#### 6 编译Hive-3.1.3
```
编译命令：mvn clean package -Pdist -DskipTests -Dmaven.javadoc.skip=true
```
##### 6.1 问题：protoc-2.5.0linux-aarch_64.exe本地仓缺失
1) 下载并解压源码
```
wget https://github.com/protocolbuffers/protobuf/releases/download/v2.5.0/protobuf-2.5.0.tar.gz

tar -zxf protobuf-2.5.0.tar.gz

mv protobuf-2.5.0 /opt/tools/installed/

cd /opt/tools/installed/protobuf-2.5.0
```
2) 安装依赖库
```
yum -y install patch automake libtool
```
3) 上传protoc.patch到服务器，打补丁，其中protoc.patch的路径视实际情况而定。
```
wget https://obs-mirror-ftp4.obs.cn-north-4.myhuaweicloud.com/tools/protoc.patch

cp protoc.patch ./src/google/protobuf/stubs/

cd ./src/google/protobuf/stubs/

patch -p1 < protoc.patch

cd -
```
4) 编译并安装到系统默认目录
```
./autogen.sh && ./configure CFLAGS='-fsigned-char' && make –j128 && make install
```
5) Protoc部署在本地Maven仓库中
```
mvn install:install-file -DgroupId=com.google.protobuf -DartifactId=protoc -Dversion=2.5.0 -Dclassifier=linux-aarch_64 -Dpackaging=exe -Dfile=/usr/local/bin/protoc
```
