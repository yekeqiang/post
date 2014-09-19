# 使用 syslog 处理多个 Docker 容器的日志

标签（空格分隔）： Docker syslog

---

> 注：该文作者 [jpetazzo][1]，该文章的原文为 [Multiple Docker containers logging to a single syslog][2]

这里有一个简单方法展示了怎样在一个容器中运行 syslog ，然后发送多台其他容器的 syslog 消息到另外一台。

可运行的 `Dockerfile` 和基础的指令已经在一个小型的 github 仓库里：https://github.com/jpetazzo/syslogdocker。

这个构思非常的简单。

首先，我们使用以下的规格参数表构建容器：

 - 已经安装了 `rsyslogd`，并且是作为默认的命令
 - `/dev` 被定义成一个卷
 - `/var/log` 被定义成一个卷

这里有这样的容器的 [Dockerfile][3]

```
FROM ubuntu:14.04
RUN apt-get update -q
RUN apt-get install rsyslog
CMD rsyslogd -n
VOLUME /dev
VOLUME /var/log
```

然后，我们启动容器；但是我们使用了一个显式的主机 bind-mount，例如：

```
docker run --name syslog -d -v /tmp/syslogdev:/dev syslog
```

为什么要使用显式的主机 bind-mount？因为当 syslog 启动的时候， 容器将创建 `/dev/log`。然后我们想得到那个 socket 并且在我们未来的容器中  bind-mount ，不用 bind-mount 整个 `/dev`。如果我们仅仅使用 `--volumes-from`，我们将得到整个 `/dev` 。它现在暂时不会产生重大的影响，但是当我们后续做一些设想的工作（像增加一个普通的 devices ），可能会把事情搞砸，因此让我们细粒度一些。

Docker 的后续版本可能允许细粒度的 `--volumes-from`。

然后我们可以启动任何容器，bind-mounting `/dev/log` 到容器里面去：

```
docker run -v /tmp/syslogdev/log:/dev/log myimage somecommand
```

对于一个教育示例，你可以这样做：

```
docker run -v /tmp/syslogdev/log:/dev/log ubuntu logger hello
```

容器将把日志消息发送到 `/dev/log`，其实际是通过 syslog 创建 socket 。

你可以通过使用 `--volumes-from syslog` 运行另外一个容器来查看日志，以及检查在 `/var/log` 里面的文件。



 


  [1]: http://jpetazzo.github.io/
  [2]: http://jpetazzo.github.io/2014/08/24/syslog-docker/
  [3]: https://github.com/jpetazzo/syslogdocker/blob/master/Dockerfile