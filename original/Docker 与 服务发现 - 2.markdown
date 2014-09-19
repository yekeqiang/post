# Docker 与 服务发现 - 2

标签（空格分隔）： Docker etcd 

---

> 注：该文由 [adetante][1] 编写，该文的原文地址为 [Service discovery with Docker - 2][2]

该文紧接着上篇文章 [Docker 与服务发现 - 1][3]

在上一篇文章中，我们看到了一个简单的方法，通过使用 Synapse 来做基于同一台 Docker 主机上的多个容器的服务发现。

现在，我们想在多台 Docker 主机上部署相同的应用，来扩展不同的服务以及确保容错性。

这次，在架构中我们需要一个新的组件： **etcd** 。

##etcd

etcd 是一个键值存储，用于共享配置以及服务发现。它使用 GO 编写，并且是 CoreOS 发行版的一部分。etcd 集群提供了高可用的机制：基于 Raft，它允许一组 etcd 实例组成一个集群。

etcd 提供了 REST API，允许客户端创建，更新和删除键。客户端还可以监听发生在特定键上的变化（客户端将被通知在该键上或者是键的目录的每次变化）。当创建一个键，客户端可以定义一个 TTL （存活时间），当客户端不再更新这个键的时候，它将会被自动清除，这个对于服务注册是非常有用的。

#概述

![etcd 的架构][4]

原理就是 Docker 容器注册进 etcd 集群：当一个应用的实例作为一个 Docker 容器启动的时候，该容器自己注册进 etcd 集群 ，当一个容器停止或者是应用挂掉的时候，对应的键将被移除出 etcd 集群。

Haproxy 只需要简单的查找 etcd 来获取可用的后端来提供 HTTP 服务，Haproxy 监听 etcd 的变化并且相应的更新配置。

在这篇文章中，我将描述服务注册部分的解决方案，关于 Haproxy 的发现部分将在下一篇文章中描述。


# Docker 的动态端口映射

Docker 是一个非常伟大的工具，它提供了很多的功能来简化应用程序的部署。但它有一个新的方式管理以及部署这些应用。

其中的一个方式就是对外公开的端口关联应用的能力。

当你启动一个 Docker 容器，你可以使用以下命令：

```
docker run -p 8000 myApplication
```

`-p 8000` 参数意味着你可以暴露 8000 端口运行这个容器，并且在 Docker 主机上是一个随机的端口。我们的应用在 Docker 容器中是监听的 8000 端口，但是这个端口将被映射到 Docker 主机上的一个动态端口。为了查看映射的端口，当容器正在运行的时候，你可以使用 `docker ps` 命令：

```
ONTAINER ID        IMAGE                COMMAND              CREATED             STATUS              PORTS                                              NAMES    
6275ea4e2ebd        coreos/etcd:latest   /opt/etcd/bin/etcd   2 weeks ago         Up 1 seconds        0.0.0.0:49153->4001/tcp, 0.0.0.0:49154->7001/tcp   pensive_leakey
```

在以上的输出中，你可以看到 `PORTS` 列的在容器中的 4001 端口被映射成了 **在 Docker 主机上的** `O.O.O.O:49153`。


对于我们而言，意味着注册进程必须考虑到端口映射。我们必须注册映射端口，而不是应用监听的端口。同样的，我们必须使用 Docker 主机的 IP 代替容器的 IP 来注册。


# 服务注册


为了把服务注册进 etcd ，我们有以下不同的解决方案：

 - 应用它自己能建立一个连接到 etcd，并且创建一个 key 来通知其他的服务它正在运行。当停止的时候，应用必须移除该 key 。它可能在服务终止的时候会变得更加复杂，但是使用 TTL ，我们可以使损失降到最小（key 不会被立即移除，仅仅在 TTl 后才被移除）。但是这意味着应用必须感知到注册进程：必须知道 etcd 实例的清单列表，必须保持一个循环定期的刷新 etcd 中的 key 。。。我更喜欢应用能与注册进程完全独立。
 - 在容器启动的时候，执行一个启动脚本来创建 etcd 中的 key ，然后在停止的时候移除它。它可能在通知应用停止的时候会变得更加复杂。而且，这还有一个风险，就是在应用完全启动之前， key 已经在 etcd 中被创建了，然后 Haproxy 可能转发 HTTP 请求到容器中，尽管应用还未启动。
 - 一个 ‘sidekick’ 进程，运行在同样的容器中，能处理应用的注册和健康检查。这个方法被 [CoreOS 用于管理服务发现][5]。这就是我将在示例中使用的解决方案。


# Sidekick 进程用于注册


负责注册的进程执行以下任务：

 - 调用 Docker API 来检索应用在容器中的对外i端口映射到本地主机的端口
 - 执行应用的健康检查：当应用可用，使用 Docker 主机的 IP 和端口 创建或是更新 etcd 中的 key
 - 当应用没有正确响应，或者容器停止了：从 etcd 中移除 key

![服务注册流程][6]


为了执行这些任务，我使用 GO 写了一些程序，可用的版本在我的 GitHub 仓库中：[github.com/adetante/dockreg][7]。
 

这个程序接收以下参数：

**--etcd**：必须的，用于注册的 etcd 的服务列表，使用以下格式：
```
--etcd http://host1:port1,http://host2:port2
```

**--key**：可选的，程序在 etcd 中为 keys 创建的父目录的名字，默认的值是 `service` 。

**--port**：必须的，应用用于注册的本地端口
**--docker**：可选的，用于访问 Docker API 的 Docker UNIX socket，默认的值是 `/var/run/docker.sock`。（看 [Docker Introspection][8]）
**--ip**：可选的，放进 etcd 的 Docker 主机的 IP 地址。如果未指定，IP 将被从 Docker API 的端口映射找到（意味着，你必须在 docker 运行的命令中指定它，比如 `docker run -p 8000::192.168.1.54`）。


这个进程将每 5 秒请求应用监听的 `--port`，如果它在 3 秒内没有得到响应，key 将被从 etcd 中移除，否则，一个 key 将被创建在 etcd 服务器的以下路径：`/keys/{service}/{ip}:{mapped_port}`。

# Docker 回顾

正如你以上所见，dockreg 进程通过一个 Unix Domain socket（/var/run/docker.sock）访问 Docker API。这是必须的，因为 Docker 没有提供其他的方式来获取容器的信息。这个需求在 Docker repository 中被讨论（see i[ssues 7421][9] and [3778][10] for example），这个在未来的版本可能会实现。目前，唯一的方法就是通过共享容器的 `docker.sock` 来实现这个需求。它并不理想，因为安全原因（使用这个接口，在安全认证之外，容器能访问和修改很多信息）。

为了分享 Docker socket，意味着容器必须以 `-v` 选项启动：

```
docker run -p 8000 -v /var/run/docker.sock:/var/run/docker.sock myApplication
```

如果一个端口映射已经在容器中定义，JSON 响应内容包含的一些东西如下：

```
"NetworkSettings": {
  "Bridge": "docker0",
  "Gateway": "172.17.42.1",
  "IPAddress": "172.17.0.4",
  "IPPrefixLen": 16,
  "PortMapping": null,
  "Ports": {
    "8000/tcp": [
      {
        "HostIp": "0.0.0.0",
        "HostPort": "49155"
      }
    ]
  }
}
```

dockreg 将使用响应中的 `ports` 数据来找回对外服务的端口（和 IP 地址）。


# 现在开始

作为一个概念验证，我在基于 Vagrant 构建的 2 个主机上创建了一个 demo，首先，克隆 vagrant 仓库：

```
git clone git@github.com:adetante/dockreg-vagrant.git
```

这个仓库包含：

 - 用于创建 2 个主机的 Vagrantfile。基础的 Vagrant box 是一个 Ubuntu 14.04 以及 放进 VM 的 2 个 Docker 镜像：ubuntu 和 etcd，这个仅仅是用于完成主机加速。
 - NodeJS 示例程序，我们想部署进 Docker 容器中的。
 - 一个 dockreg 的构建工具
 - 一个 Dockerfile 用于构建一个镜像，包含  NodeJS app 和 `dockreg` sidekick 进程。

当开始的时候，vagrant 脚本（`build.sh`） 将启动一个 etcd 容器，作为一个 host-1 和 host-2 的集群配置。下一步，它将构建一个包含 NodeJS app 和 `dockreg` sidekick 进程的镜像，Dockerfile 如下：

```
FROM ubuntu:trusty
 
RUN apt-get update && \
	apt-get install -y python python-setuptools nodejs && \
	easy_install supervisor && \
	mkdir /var/log/dockreg
 
EXPOSE 8000
 
CMD []
ENTRYPOINT ["/usr/local/bin/supervisord","-c","/etc/supervisord.conf"]
 
ADD supervisord.conf /etc/supervisord.conf
ADD dockreg /usr/bin/dockreg
 
ADD node-app/server.js /root/server.js
 
RUN chmod a+x /usr/bin/dockreg
```

在这个构建镜像中，一个 supervisord 进程将启动 NodeJS 应用和 dockreg 进程。

Supervisord 配置如下：

```
[supervisord]
nodaemon=true
logfile=/var/log/dockreg/supervisord.log
logfile_maxbytes=50MB
logfile_backups=4
loglevel=info
pidfile=/var/run/supervisord.pid
 
[program:nodejs-server]
command=nodejs /root/server.js
autorestart=unexpected
stdout_logfile=/var/log/dockreg/http.stdout
stdout_logfile_maxbytes=1MB
stdout_logfile_backups=10
stderr_logfile=/var/log/dockreg/http.stderr
stderr_logfile_maxbytes=1MB
stderr_logfile_backups=10
 
[program:dockreg]
command=/usr/bin/dockreg --port 8000 --etcd http://%(ENV_ETCD_PORT_4001_TCP_ADDR)s:%(ENV_ETCD_PORT_4001_TCP_PORT)s --ip %(ENV_IP)s
autorestart=unexpected
stdout_logfile=/var/log/dockreg/dockreg.stdout
stdout_logfile_maxbytes=1MB
stdout_logfile_backups=10
stderr_logfile=/var/log/dockreg/dockreg.stderr
stderr_logfile_maxbytes=1MB
stderr_logfile_backups=10
```

是时间开始了，启动一个主机：

```
vagrant up host-1
```

在 boot 日志中，你将看到 Docker 容器的构建进程。在最后：

```
Successfully built a98722a9f44b
```

下一步，登陆进 VM：

```
vagrant ssh host-1
```

检查 etcd 容器是否在运行：

```
docker ps
```

你可以使用以下的地址访问 etcd ：http://10.1.0.101:4001/v2/machines
 
下一步，启动一个新的容器，运行 NodeJS app：

```
docker run -d -p 8000 -v /var/run/docker.sock:/var/run/docker.sock -e IP=10.1.0.101 --link etcd:etcd local/dockreg
```

当容器启动的时候，可以在 etcd 中看到注册：

```
curl http://10.1.0.101:4001/v2/keys/service

{
  "action": "get",
  "node": {
    "key": "/service",
    "dir": true,
    "nodes": [
      {
        "key": "/service/10.1.0.101:49155",
        "value": "running",
        "expiration": "2014-08-26T21:17:28.431730841Z",
        "ttl": 19,
        "modifiedIndex": 23,
        "createdIndex": 23
      }
    ],
    "modifiedIndex": 3,
    "createdIndex": 3
  }
}
```

在这个示例中，暴露的端口是 49155。

访问应用：

```
curl http://10.1.0.101:49155

Hello from 2d151d56f838
```

成功，下一步，启动第二个主机：

```
vagrant up host-2
```

一旦运行，检查已经加入到集群中的新的 etcd 的实例：

```
curl http://10.1.0.102:4001/v2/machines

http://10.1.0.101:4001, http://10.1.0.102:4001
```

检查已经复制的插入进  host-1 的 key：

```
curl http://10.1.0.102:4001/v2/keys/service

{
  "action": "get",
  "node": {
    "key": "/service",
    "dir": true,
    "nodes": [
      {
        "key": "/service/10.1.0.101:49155",
        "value": "running",
        "expiration": "2014-08-26T21:39:07.415916256Z",
        "ttl": 17,
        "modifiedIndex": 73,
        "createdIndex": 73
      }
    ],
    "modifiedIndex": 3,
    "createdIndex": 3
  }
```

现在，登陆 host-2 （vagrant ssh host-2），然后启动一个新的应用容器：

```
docker run -d -p 8000 -v /var/run/docker.sock:/var/run/docker.sock -e IP=10.1.0.102 --link etcd:etcd local/dockreg
```

http://10.1.0.101:4001/v2/keys/service 和 http://10.1.0.102:4001/v2/keys/service 现在可以显示运行在 host-2 上的新的实例（或许需要花一点时间用于应用启动）。

还需要一个新的？仅仅需要再一次运行前面的 `docker run` 命令。现在 etcd 包含 3 个实例。

为了测试取消登记，停止一个容器：

```
docker stop fdd27afacbbb
```

key 将立即从 etcd 中移除。

# 总结

在这个例子中，我们看到了一种把 Docker 容器注册进一个外部配置存储的方法。通过使用一个 sidekick 进程，服务注册完全独立于应用，并且它提供了更精确的监控。

达到这个目标的其他可选方案，The CoreOS project 提议了一份工具列表来简化注册和服务发现。

在下一篇文章中，我将描述基于 HAProxy 和 etcd 的服务发现。


  [1]: http://adetante.github.io/
  [2]: http://adetante.github.io/articles/service-discovery-with-docker-2/
  [3]: http://adetante.github.io/articles/service-discovery-with-docker-1/
  [4]: http://adetante.github.io/images/docker/overview2.png
  [5]: https://coreos.com/docs/launching-containers/launching/launching-containers-fleet/#toc_3
  [6]: http://adetante.github.io/images/docker/case2.png
  [7]: https://github.com/adetante/dockreg
  [8]: http://adetante.github.io/articles/service-discovery-with-docker-2/#docker-introspection
  [9]: https://github.com/docker/docker/issues/7421
  [10]: https://github.com/docker/docker/issues/3778