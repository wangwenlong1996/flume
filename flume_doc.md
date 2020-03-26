[TOC]
# Flume 用户指导
## 介绍
### 概况
Apache Flume是一个分布式的、可靠的、可用的系统，它可以有效地收集、聚合和移动大量的日志数据，这些数据可以从许多不同的来源转移到一个集中的数据存储中。

Apache Flume的使用不仅限于日志数据聚合。由于数据源是可定制的，所以Flume可以用来传输大量的事件数据，包括但不限于网络流量数据、社交媒体生成的数据、电子邮件消息以及几乎所有可能的数据源。

Apache Flume是Apache软件基金会的一个顶级项目。

### 系统要求
1. java运行环境- Java 1.8或更高版本
2. 内存——源、通道或接收器使用的配置所需的足够内存
3. 磁盘空间——足够的磁盘空间用于通道或接收器使用的配置
4. 目录权限——代理使用的目录的读/写权限

### 架构
#### 数据流模型
Flume 事件被定义为一个数据流单元，它具有一个字节负载和一组可选的字符串属性。Flume代理是一个(JVM)进程，通过组件，事件从外部源流向下一个目标(hop)。
![Spark Streaming Data Flow](./assets/UserGuide_image00.png)

Flume源消费由外部源(如web服务器)传递给它的事件。外部源以目标Flume源可识别的格式向Flume发送事件。例如，Avro Flume源可用于从Avro客户端接收Avro事件，或从数据流中的其他Flume代理的Avro sink发送事件。类似的流可以定义为使用Thrift flume源来接收来自Thrift sink或Thrift Rpc客户端或Thrift客户端的事件，这些客户端使用由Thrift协议生成的任何语言编写。当一个Flume源接收到一个事件时，它将其存储到一个或多个通道中。通道是一个被动存储，它保存事件，直到它被水槽吸收为止。文件通道就是一个例子——它由本地文件系统支持。发送器从通道中移除事件，并将其放入诸如HDFS(通过Flume HDFS接收)之类的外部存储库中，或将其转发到流中的下一个Flume代理(下一跳)的Flume源。给定代理中的源和接收器与通道中暂存的事件异步运行。
#### 复杂流
Flume允许用户构建多跃点流，其中事件在到达最终目的地之前通过多个代理。它还允许扇入和扇出流、上下文路由和失败跳转的备份路由(故障转移)。
#### 可靠性
事件在每个代理上的一个通道中转移。然后将事件传递到流中的下一个代理或终端存储库(如HDFS)。事件只有在存储到下一个代理的通道或终端存储库中之后，才会从通道中删除。这就是Flume中的单跳消息传递语义如何提供流的端到端可靠性的原因。

Flume使用事务性方法来保证事件的可靠交付。sources和sink封装在事务中，存储/检索由通道提供的事务放置或提供的事件。这可以确保事件集在流中可靠地从一个代理传递到另一个代理。在多跳流的情况下，来自前一跳的接收器和来自下一跳的源都运行它们的事务，以确保数据安全地存储在下一跳的通道中。
#### 可恢复性
事件在通道中暂存，通道负责管理故障恢复。Flume支持由本地文件系统支持的持久文件通道。还有一个内存通道，它只是将事件存储在内存队列中，这是更快的，但任何事件仍然留在内存通道时，代理进程故障不能恢复。

## 安装
### 安装代理
Flume代理配置存储在一个本地配置文件中。这是一个遵循Java属性文件格式的文本文件。可以在同一个配置文件中指定一个或多个代理的配置。配置文件包括代理中的每个源、接收器和通道的属性，以及如何将它们连接在一起以形成数据流。
#### 配置单个组件
流中的每个组件(源、接收或通道)都有特定于类型和实例化的名称、类型和属性集。例如，Avro源需要一个主机名(或IP地址)和一个端口号来接收数据。内存通道可以有最大的队列大小(“容量”)，而HDFS接收器需要知道文件系统URI、创建文件的路径、文件旋转的频率(“HDFS . rollinterval”)等。组件的所有这些属性都需要在宿主Flume代理的属性文件中设置。
#### 连接聚集
为了构成流，代理需要知道要加载哪些组件以及它们是如何连接的。这是通过列出代理中每个源、接收器和通道的名称来完成的，然后为每个接收器和源指定连接通道。例如，代理通过名为file-channel的文件通道将事件从称为avroWeb的Avro源流到HDFS接收器HDFS -cluster1。配置文件将包含这些组件的名称和文件通道，作为avro Web源和hdfs-cluster1接收器的共享通道。

#### 开始一个代理
使用一个名为flume-ng的shell脚本启动一个代理，该脚本位于Flume发行版的bin目录中。您需要在命令行上指定代理名称、配置目录和配置文件:
`bin/flume-ng agent -n $agent_name -c conf -f conf/flume-conf.properties.template`
现在，代理将开始运行在给定属性文件中配置的源和接收器。
#### 一个简单例子
在这里，我们给出一个示例配置文件，它描述了一个单节点Flume部署。此配置允许用户生成事件，并随后将其记录到控制台。
```properties
# example.conf: A single-node Flume configuration

# Name the components on this agent
a1.sources = r1
a1.sinks = k1
a1.channels = c1

# Describe/configure the source
a1.sources.r1.type = netcat
a1.sources.r1.bind = localhost
a1.sources.r1.port = 44444

# Describe the sink
a1.sinks.k1.type = logger

# Use a channel which buffers events in memory
a1.channels.c1.type = memory
a1.channels.c1.capacity = 1000
a1.channels.c1.transactionCapacity = 100

# Bind the source and sink to the channel
a1.sources.r1.channels = c1
a1.sinks.k1.channel = c1
```

这个配置定义了一个名为a1的代理。a1有一个监听端口44444上的数据的源、一个缓冲内存中事件数据的通道和一个将事件数据记录到控制台的接收器。配置文件命名各个组件，然后描述它们的类型和配置参数。一个给定的配置文件可以定义几个指定的代理;当启动给定的Flume进程时，将传递一个标志，告诉它要显示哪个指定的代理。
有了这个配置文件，我们可以按如下方式启动Flume:
`bin/flume-ng agent --conf conf --conf-file example.conf --name a1 -Dflume.root.logger=INFO,console`
注意，在完整的部署中，我们通常会包含一个额外的选项:``--conf=\<conf-dir>`。`\<conf-dir>``目录将包括一个shell脚本flume-env.sh和一个log4j属性文件。在本例中，我们传递了一个Java选项来强制Flume登录到控制台，并且我们没有使用自定义环境脚本。

从一个单独的终端，我们可以telnet端口44444和发送Flume一个事件:
```shell
$ telnet localhost 44444
Trying 127.0.0.1...
Connected to localhost.localdomain (127.0.0.1).
Escape character is '^]'.
Hello world! <ENTER>
OK
```

原始的Flume终端将在日志消息中输出事件。
```shell
12/06/19 15:32:19 INFO source.NetcatSource: Source starting
12/06/19 15:32:19 INFO source.NetcatSource: Created serverSocket:sun.nio.ch.ServerSocketChannelImpl[/127.0.0.1:44444]
12/06/19 15:32:34 INFO sink.LoggerSink: Event: { headers:{} body: 48 65 6C 6C 6F 20 77 6F 72 6C 64 21 0D          Hello world!. }
```
祝贺您——您已经成功地配置和部署了一个Flume代理!后续部分将更详细地介绍代理配置。

#### 配置文件配置环境变量
Flume能够在配置中替换环境变量。例如:
```properties
a1.sources = r1
a1.sources.r1.type = netcat
a1.sources.r1.bind = 0.0.0.0
a1.sources.r1.port = ${NC_PORT}
a1.sources.r1.channels = c1
```
注意:它目前只适用于值，不适用于键。(即。仅在配置行=标记的“右侧”。)

可以通过设置propertiesImplementation = org.apache.flume.node.EnvVarResolverProperties来启用代理调用上的Java系统属性。
例如:
`$ NC_PORT=44444 bin/flume-ng agent –conf conf –conf-file example.conf –name a1 -Dflume.root.logger=INFO,console -DpropertiesImplementation=org.apache.flume.node.EnvVarResolverProperties`

注意以上只是一个例子，环境变量可以通过其他方式配置，包括在conf/flume-env.sh中设置。

#### 记录原始数据
在许多生产环境中，记录通过ingest管道的原始数据流不是理想的行为，因为这可能导致敏感数据或安全相关的配置(例如密钥)泄漏到Flume日志文件。默认情况下，Flume不会记录这些信息。另一方面，如果数据管道被破坏，Flume将尝试为调试问题提供线索。

调试事件管道问题的一种方法是设置一个连接到日志记录器接收器的额外内存通道，它将把所有事件数据输出到Flume日志。然而，在某些情况下，这种方法是不够的。

为了能够记录与事件和配置相关的数据，除了log4j属性外，还必须设置一些Java系统属性。

要启用与配置相关的日志记录，请设置Java系统属性-Dorg.apache.flume.log.printconfig=true。它可以通过命令行传递，也可以通过在flume-env.sh中的JAVA_OPTS变量中设置它来传递。

要启用数据日志记录，请设置Java系统属性-Dorg.apache.flume.log.rawdata=true，与上面描述的相同。对于大多数组件，还必须将log4j日志级别设置为DEBUG或TRACE，以使特定于事件的日志出现在Flume日志中。

下面是一个启用配置日志记录和原始数据日志记录的示例，同时还将Log4j日志级别设置为调试控制台输出:
`$ bin/flume-ng agent --conf conf --conf-file example.conf --name a1 -Dflume.root.logger=DEBUG,console -Dorg.apache.flume.log.printconfig=true -Dorg.apache.flume.log.rawdata=true`

#### 基于Zookeeper的配置
Flume通过Zookeeper支持代理配置。这是一个实验性的特性。配置文件需要在Zookeeper中以可配置的前缀上传。配置文件存储在Zookeeper节点数据中。下面是代理a1和a2的Zookeeper节点树。

```
- /flume
 |- /a1 [Agent config file]
 |- /a2 [Agent config file]
```
上传配置文件后，使用以下选项启动代理。
```shell
$ bin/flume-ng agent –conf conf -z zkhost:2181,zkhost1:2181 -p /flume –name a1 -Dflume.root.logger=INFO,console
```
Argument Name |Default | Description
-|-|-
z|-|zookeeper连接字符串。用逗号分隔的主机名列表:端口
p|/flume | Zookeeper中存储代理配置的基本路径

#### 安装第三方插件
Flume有一个完全基于插件的架构。虽然Flume附带了许多开箱即用的源、通道、接收器、序列化器等，但是也有许多实现是与Flume分开发布的。

虽然可以通过将其jar添加到Flume -env.sh文件中的FLUME_CLASSPATH变量中来包含定制的Flume组件，但是Flume现在支持一个名为plugins的特殊目录。它会自动获取以特定格式打包的插件。这样可以更容易地管理插件打包问题，也可以更简单地调试和排除几类问题，特别是库依赖冲突。

##### plugins.d目录
`plugins.d`目录位于`$FLUME_HOME/plugins.d`。在启动时，flume-ng启动脚本在`plugins.d`目录中查找插件，符合以下格式，并包括在正确的路径时，启动java。

##### 用于插件的目录布局
插件中的(子目录)plugins.d可以有多达三个子目录:
1. lib -插件的jar文件
2. libext -插件的依赖jar(s)
3. 本机-任何必需的本机库，如.so文件

插件中的两个插件的plugins.d目录:
```
plugins.d/
plugins.d/custom-source-1/
plugins.d/custom-source-1/lib/my-source.jar
plugins.d/custom-source-1/libext/spring-core-2.5.6.jar
plugins.d/custom-source-2/
plugins.d/custom-source-2/lib/custom.jar
plugins.d/custom-source-2/native/gettext.so
```
