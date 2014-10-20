# 使用 Fluentd 管理 Docker 日志

标签（空格分隔）： Fluentd Docker 日志管理

---

> 本文作者是 [jwilder][1]，本文原文地址是 [Docker Log Management Using Fluentd][2]


当前 docker 版本的一个问题就是日志管理。每个容器运行一个单独的进程，并且进程的输出被 docker 保存在主机上的一个位置。

在当前有一些操作问题：

- 日志无限制的增长。Docker 以 JSON 消息记录每一行日志，这能引起文件增长过快以及超过主机上的磁盘空间，因为它不会自动轮转。
- `docker logs` 命令返回在每次它运行的时候返回所有的日志记录。任何长时间运行的进程产生的日志都是冗长的，这会导致仔细检查非常困难。
- 日志位于容器 `/var/log` 下或者是其他不容易可视化或访问的位置。


## Docker 日志选项

虽然日志在 Docker 中正在演变，有几种方法可以处理当前的 Docker 的日志：

- **在容器内收集** - 除了正在运行的应用程序之外，每个容器设置一个日志收集进程。[baseimage-docker][3] 使用 [runit][4] 连同 syslog 作为一个示例。
- **在容器外收集** - 一个单独的收集 agent 运行在主机上，容器有一个从该主机挂载的卷，它们把日志记录在那里。
- **在单独的容器中收集** - 这是一个在主机上运行收集 agent 的细微变化。该收集 agent 也是运行在一个容器中并且该容器的卷是使用 docker run 的 `volumes-from` 选项被绑定给任何应用程序容器。这篇 [Docker 和 logstash][5] 有一个这种方法的示例。

这些方法可以工作，但是也有一些缺点。如果收集被执行在容器里面，这时每个容器运行多个进程会导致资源浪费（在一个容器中[运行多个进程][6] 看起来似乎是一个争议性的话题，甚至 docker 官方文档使用 [supervisor][7] 作为一个示例）。


如果收集使用 volumes 运行在容器外面，你依然需要确保你的应用程序日志记录进这些 volumes 而不是 stdout/stderr。对于所有应用来说，这或许不可能。最终，容器运行依然有容器 JSON 日志文件，它也将无限增长。 


## Docker 使用 Fluentd

容器外收集的另外一个变化就是通过一个中央化的 agent 来处理，不用绑定 volumes 到容器。这个方法是直接供工作于容器在主机上的 JSON 日志文件。

当你运行一个容器，容器的状态存活在 `/var/lib/docker/containers/<id>` 下面：

```
root@precise64:/var/lib/docker/containers/fe38c4124f36d0a5b2a38ea7dd58fe88ac92980286f1f6a7b7ed3ced7c994374# ls -la
total 44
drwx------  3 root root  4096 Mar 14 19:56 .
drwx------ 83 root root 12288 Mar 14 21:53 ..
-rw-r--r--  1 root root   106 Mar 14 19:56 config.env
-rw-r--r--  1 root root  1522 Mar 14 19:56 config.json
-rw-------  1 root root   241 Mar 14 19:56 fe38c4124f36d0a5b2a38ea7dd58fe88ac92980286f1f6a7b7ed3ced7c994374-json.log
-rw-r--r--  1 root root   126 Mar 14 19:56 hostconfig.json
-rw-r--r--  1 root root    13 Mar 14 19:56 hostname
-rw-r--r--  1 root root   181 Mar 14 19:56 hosts
drwxr-xr-x  2 root root  4096 Mar 14 19:56 root
```

文件 `e38c4124f36d0a5b2a38ea7dd58fe88ac92980286f1f6a7b7ed3ced7c994374-json.log` 是容器的日志文件。每一行是一个 JSON 对象并且对于容器输入和输出的每一行这有一个界限。

```
{"log":"root@c835298de6dd:/# ls\r\n","stream":"stdout","time":"2014-03-14T22:15:15.155863426Z"}
{"log":"bin  boot  dev\u0009etc  home  lib\u0009lib64  media  mnt  opt\u0009proc  root  run  sbin  selinux\u0009srv  sys  tmp  usr  var\r\n","stream":"stdout","time":"2014-03-14T22:15:15.194869963Z"
```

[fluentd][8] 是一个开源的数据收集器，它原生就支持 JSON 格式，因此你可以在主机上运行一个单独的 fluentd 实例并配置它来 tail 每个容器的 JSON 文件。

如果你需要 tail 一个在容器文件系统某处的日志文件，你也可以使用 `root` 子目录。所有 tailed 文件这时能被转发到一个[中央日志系统][9]。

这有一个 fluentd.conf 配置文件的例子，tails 每个容器的日志并发送它们到 stdout。

```
## File input
## read docker logs with tag=docker.container

<source>
  type tail
  format json
  time_key time
  path /var/lib/docker/containers/c835298de6dde500c78a2444036101bf368908b428ae099ede17cf4855247898/c835298de6dde500c78a2444036101bf368908b428ae099ede17cf4855247898-json.log
  pos_file /var/lib/docker/containers/c835298de6dde500c78a2444036101bf368908b428ae099ede17cf4855247898/c835298de6dde500c78a2444036101bf368908b428ae099ede17cf4855247898-json.log.pos
  tag docker.container.c835298de6dd
  rotate_wait 5
</source>

<source>
  type tail
  format json
  time_key time
  path /var/lib/docker/containers/965c22a2ad1e935cb1476772ebe1ebef0050559b4cbcc7775b936348e7822347/965c22a2ad1e935cb1476772ebe1ebef0050559b4cbcc7775b936348e7822347-json.log
  pos_file /var/lib/docker/containers/965c22a2ad1e935cb1476772ebe1ebef0050559b4cbcc7775b936348e7822347/965c22a2ad1e935cb1476772ebe1ebef0050559b4cbcc7775b936348e7822347-json.log.pos
  tag docker.container.965c22a2ad1e
  rotate_wait 5
</source>

<source>
  type tail
  format json
  time_key time
  path /var/lib/docker/containers/889fe291f590c2c2aa2852856687efbb3e6fdd2faeca84da4bc0be2263f37953/889fe291f590c2c2aa2852856687efbb3e6fdd2faeca84da4bc0be2263f37953-json.log
  pos_file /var/lib/docker/containers/889fe291f590c2c2aa2852856687efbb3e6fdd2faeca84da4bc0be2263f37953/889fe291f590c2c2aa2852856687efbb3e6fdd2faeca84da4bc0be2263f37953-json.log.pos
  tag docker.container.889fe291f590
  rotate_wait 5
</source>

<match docker.**>
  type stdout
</match>
```

在一个在线环境，实际的日志内容可以被发送到一个 [elasticsearch][10] 集群，并使用 [kibana][11] 或 [graylog2][12] 查看。二选一，这也有使用 JSON 工作的主机服务。

因为容器  ID 在随着工作变化的，我创建了一个简单的称为 [docker-gen][13] 的 golang 工程，它可以使用模板从运行中的 docker 容器数据中生成任意文件。在这个工程中的 [fluentd 模板][14]示例被用于生成以上的样本。

尽管没有展示，[docker-gen][15] 也能生成 logrotate 配置文件来轮转容器的 JSON 文件来避免在主机上没有磁盘空间运行。希望，docker 项目能在未来的发行版中添加这个。

## 总结

这个方法提供了一下好处：

- 主机可以使用一个单独的收集 agent 转发任何容器的日志到一个中央日志服务器.
- 不需要要求应用程序使用 syslog 或是写日志到特定的卷。
- 主机可以访问容器日志以及任何在容器文件系统上的日志文件。
- 主机可以为容器轮转日志。

这个方法的一个缺点就是它直接访问 docker 文件系统而没有使用 API，这意味着它在未来可能出现问题，因为如果未来 docker 版本改变了它怎样在主机文件系统存储容器日志的话。



  [1]: https://github.com/jwilder
  [2]: http://jasonwilder.com/blog/2014/03/17/docker-log-management-using-fluentd/
  [3]: https://github.com/phusion/baseimage-docker
  [4]: http://smarden.org/runit/
  [5]: http://denibertovic.com/post/docker-and-logstash-smarter-log-management-for-your-containers/
  [6]: http://phusion.github.io/baseimage-docker/
  [7]: http://docs.docker.io/en/latest/examples/using_supervisord/
  [8]: http://fluentd.org/
  [9]: http://jasonwilder.com/blog/2012/01/03/centralized-logging/
  [10]: http://elasticsearch.org/
  [11]: http://www.elasticsearch.org/overview/kibana/
  [12]: http://graylog2.org/
  [13]: https://github.com/jwilder/docker-gen
  [14]: https://github.com/jwilder/docker-gen/blob/master/templates/fluentd.conf.tmpl
  [15]: https://github.com/jwilder/docker-gen