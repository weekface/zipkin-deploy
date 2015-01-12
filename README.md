## Zipkin生产环境部署步骤

此安装过程中需要用到的安装包文件：

* jdk-7u71-linux-x64.rpm

请下载好。

### 安装scala

* 安装oracle jdk7

```
$ rpm -ivh ./jdk-7u71-linux-x64.rpm
```

* 安装scala

```
$ wget http://downloads.typesafe.com/scala/2.11.4/scala-2.11.4.tgz
$ tar zxvf scala-2.11.4.tgz
$ mv scala-2.11.4 /usr/lib
```

* 配置scala路径

```
$ echo "
export SCALA_HOME=/usr/lib/scala-2.11.4
export PATH=$PATH:$SCALA_HOME/bin
" >> /etc/profile

$ source /etc/profile
```

* 输入scala命令，输入下面表示安装成功：

```
$ scala
Welcome to Scala version 2.11.4 (Java HotSpot(TM) 64-Bit Server VM, Java 1.7.0_71).
Type in expressions to have them evaluated.
Type :help for more information.
scala> 
```

### 安装Cassandra

注意：cassandra需要部署在所有机器上

* 下载并安装cassandra

```
$ wget http://ftp.riken.jp/net/apache/cassandra/2.1.2/apache-cassandra-2.1.2-bin.tar.gz
$ tar zxvf apache-cassandra-2.1.2-bin.tar.gz 
$ mv apache-cassandra-2.1.2 cassandra
$ mv cassandra /opt/
```

* 优化系统参数，使cassandra达到最优运行效率

```
$ echo 0 > /proc/sys/vm/zone_reclaim_mode
$ vim /etc/security/limits.conf # 加入以下设置
* - memlock unlimited
* - nofile 100000
* - nproc 32768
* - as unlimited
$ vim /etc/security/limits.d/90-nproc.conf # 加入以下设置
* - nproc 32768
$ vim /etc/sysctl.conf # 加入以下设置
vm.max_map_count = 131072
$ sysctl -p
$ swapoff --all
$ # 保持所有服务器时间一致
```

* 配置cassandra集群

```
$ vim /opt/cassandra/conf/cassandra.yaml # 修改如下配置
cluster_name: 'ZipkinCassandraCluster'
num_tokens: 256
listen_address: 192.168.xxx.xxx # 这里需要修改为本地的静态IP地址，目的是其他节点可以远程连接
rpc_address: 192.168.xxx.xxx # 这里需要修改为本地的静态IP地址，目的是其他节点可以远程连接
seeds: "ip1,ip2,ip3" # 这里的ip1,ip2,ip3为cassandra集群所有节点的ip地址，逗号分割

commitlog_directory /data/cassandra_commitlog # 此目录需要配置到一块单独硬盘上
concurrent_reads: 64 # 具体数值为硬盘个数x16
concurrent_writes: 192 # 具体数值为cpu核数x8
```

* 清除数据

```
$ rm -rf /opt/cassandra/data/data/system/*
$ rm -rf /opt/cassandra/logs/system.log
```

* 启动第一个节点

```
$ /opt/cassandra/bin/cassandra # 请使用Supervisor启动cassandra
```

* 创建Zipkin数据库

```
$ wget https://raw.githubusercontent.com/twitter/zipkin/master/zipkin-cassandra/src/schema/cassandra-schema.txt
$ vim cassandra-schema.txt
# 修改里面的地址：connect 127.0.0.1/9160; 为本机静态ip地址
$ /opt/cassandra/bin/cassandra-cli -host xxx.xxx.xxx.xxx -port 9160 -f cassandra-schema.txt
```

* 挨个启动其他节点

```
$ /opt/cassandra/bin/cassandra # 请使用Supervisor启动cassandra
```

### 安装Zipkin

```
$ git clone https://github.com/twitter/zipkin.git /opt/zipkin
```

* 配置zipkin的JVM启动参数

```
$ vim bin/sbt
```

将启动参数修改为：

```
java -ea                          \
  $SBT_OPTS                       \
  $JAVA_OPTS                      \
  -XX:+CMSClassUnloadingEnabled   \
  -XX:+UseThreadPriorities        \
  -XX:ThreadPriorityPolicy=42     \
  -Xms8192M                       \
  -Xmx8192M                       \
  -Xmn2048M                       \
  -XX:+HeapDumpOnOutOfMemoryError \
  -Xss256k                        \
  -XX:StringTableSize=1000003     \
  -XX:+UseParNewGC                \
  -XX:+UseConcMarkSweepGC         \
  -XX:+CMSParallelRemarkEnabled   \
  -XX:SurvivorRatio=8             \
  -XX:MaxTenuringThreshold=1      \
  -XX:CMSInitiatingOccupancyFraction=75 \
  -XX:+UseCMSInitiatingOccupancyOnly \
  -XX:+UseTLAB                    \
  -server                         \
  -jar $sbtjar "$@"
```

* 编译zipkin

```
$ cd /opt/zipkin; ./bin/sbt compile
```

* 配置并启动zipkin-collector

注意：zipkin-collector需要部署在所有机器上

```
$ vim /opt/zipkin/zipkin-collector-service/config/collector-cassandra.scala
val keyspaceBuilder = cassandra.Keyspace.static(nodes = Set("xxx.xxx.xxx.xxx"))
# 修改cassandra服务连接地址

.writeTo(cassandraBuilder).queueNumWorkers(50) # 修改zipkin的worker数为50

$ vim project/Project.scala
fork := false, # 将fork修改为false

$ ./bin/collector-cassandra # 请使用Supervisor启动zipkin collector
```

* 配置并启动zipkin-query

注意：zipkin-query只需要部署在一台机器上

```
$ vim zipkin-query-service/config/query-cassandra.scala
val keyspaceBuilder = cassandra.Keyspace.static(nodes = Set("xxx.xxx.xxx.xxx.")) # 修改为cassandra服务地址
$ ./bin/query cassandra # 请使用Supervisor启动zipkin query
```

* 配置并启动zipkin-web

注意：zipkin-web只需要部署在一台机器上

```
$ echo "#!/usr/bin/env sh
bin/sbt 'project zipkin-web' 'run -zipkin.web.query.dest=xxx.xxx.xxx.xxx:9411 -zipkin.web.rootUrl=http://localhost:8080/'
# 请将xxx.xxx.xxx.xxx替换为zipkin-query启动的机器ip
" > bin/zipkin-web

$ chmod +x ./bin/collector-web
$ ./bin/collector-web # 请使用Supervisor启动zipkin web
```

Finish!

## 参考资料：

* http://blog.chinaunix.net/uid-15463753-id-4252690.html
* http://blog.csdn.net/chenxingzhen001/article/details/11072765
* https://github.com/weekface/docker-zipkin
* http://www.scala-lang.org/download/
* https://github.com/twitter/zipkin/blob/master/doc/ubuntu-quickstart.txt
* http://www.csdn123.com/html/exception/274/274381_274380_274372.htm
* http://www.datastax.com/documentation/cassandra/2.0/cassandra/install/installRecommendSettings.html
