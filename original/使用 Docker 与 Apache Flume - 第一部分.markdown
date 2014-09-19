# 使用 Docker 与 Apache Flume - 第一部分

标签（空格分隔）： Docker Apache Flume RUN CMD  ENTRYPOINT flume-ng

---

在 Unruly，我们使用 [Apache Flume][1] 处理我们 event-streaming 架构的一部分，因为它很容易设置和减少自定义 sources 和 sinks，在我的创新时间，我尝试设置一些 Flume 拓扑来学习 Docker 和 集装箱化。

## 设置基础镜像

Docker 有镜像的概念，从镜像中我们可以启动一个容器，因此，第一步就是创建一个 Flume pre-installed 的镜像。Flume 仅仅依赖 JAVA（它是一个 JAVA 工程），我从 Ubuntu 基础镜像中创建它，将执行以下步骤：

 - 安装 java 和 wget 
 - 下载和解压 flume 工程到 `/opt/flume`
 - 设置 JAVA_HOME 并且把 flume-ng 加入 PATH

下面是我们需要做的

```
FROM ubuntu

# install wget + java
RUN apt-get update -q
RUN DEBIAN_FRONTEND=noninteractive apt-get install \
  -qy --no-install-recommends \
  wget openjdk-7-jre

# download and unzip Flume
RUN mkdir /opt/flume
RUN wget -qO- \
  https://archive.apache.org/dist/flume/stable/apache-flume-1.4.0-bin.tar.gz \
  | tar zxvf - -C /opt/flume --strip 1

# set environment variables
ENV JAVA_HOME /usr/lib/jvm/java-7-openjdk-amd64
ENV PATH /opt/flume/bin:$PATH
```

从这个 Dockerfile 中构建一个镜像（使用 `docker build -t flume .`），可行的版本在 [Docker index][2] 中。

## 一个基础的 Flume 拓扑

一个 Flume 拓扑由 agents 组成，它由三个核心概念：sources, channels, 和 sinks。

我们从 sources 接收数据，把它传递进一个或多个 channels，然后被 sinks 读取和处理。大部分基础的拓扑包含一个单独的节点，下面我构建一个叫做 Docker 的 agent，有以下结构：

 - **一个 NetcatSource**，从一个端口读取数据，并且传递进 events。
 - **一个 MemoryChannel**，在内存中 buffering events 。
 - **一个 LoggerSink**，仅仅记录它接收到的 events。

这个拓扑的配置文件如下，我们会以 flume-example.conf  作为参考，看起来像这样：

```
docker.sinks = logSink
docker.sources = netcatSource
docker.channels = inMemoryChannel

docker.sources.netcatSource.type = netcat
docker.sources.netcatSource.bind = 0.0.0.0
docker.sources.netcatSource.port = 44444
docker.sources.netcatSource.channels = inMemoryChannel

docker.channels.inMemoryChannel.type = memory
docker.channels.inMemoryChannel.capacity = 1000
docker.channels.inMemoryChannel.transactionCapacity = 100

docker.sinks.logSink.type = logger
docker.sinks.logSink.channel = inMemoryChannel
```

从这里，我们可以通过这个配置文件创建一个新的容器，并且启动这个 docker agent。

```
FROM probablyfine/flume

ADD flume-example.conf /var/tmp/flume-example.conf

EXPOSE 44444

ENTRYPOINT [ "flume-ng", "agent",
  "-c", "/opt/flume/conf", "-f", "/var/tmp/flume-example.conf", "-n", "docker",
  "-Dflume.root.logger=INFO,console" ]
```

在 ENTRYPOINT 点的 flume-ng 命令将在容器启动的时候被运行（需要配置目录，配置文件，和 agent 的名字），[EXPOSE][3] 指令使得端口在运行时可用，这个端口 NetcatSource 将监听。

> 注：关于 CMD、RUN、ENTRYPOING 的区别请看这里 http://segmentfault.com/q/1010000000417103

一旦我们构建了这个新的镜像（我们称作 flume-example），我们使用 `ocker run -p 444:44444 -t flume-example` 命令启动这个容器，

`-p 444:44444` 参数将容器中的 44444 端口映射到本地主机上的 444 端口。现在我们可以给它写消息，使用 `echo foo bar baz | nc localhost 444` 然后看事件被记录。

```
...
2014-05-05 19:26:13,218 (SinkRunner-PollingRunner-DefaultSinkProcessor)
  [INFO - org.apache.flume.sink.LoggerSink.process(LoggerSink.java:70)]
  Event: { headers:{} body: 66 6F 6F 20 62 61 72 20 62 61 7A foo bar baz }
...
```

酷！现在我们有一个工作的 Flume agent 可以提取和处理数据了。

下一篇文章将展示一些有趣的 Flume 拓扑，以及使得我们怎样能更加容易的把 Docker 的功能（比如 共享卷以及只读 mounting）整合进一个 Flume 中设置。

 


  [1]: https://flume.apache.org/
  [2]: https://index.docker.io/u/probablyfine/flume/
  [3]: http://docs.docker.io/reference/builder/#expose