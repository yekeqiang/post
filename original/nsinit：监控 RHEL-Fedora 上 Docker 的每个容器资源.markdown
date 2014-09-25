# nsinit：监控 RHEL/Fedora 上 Docker 的每个容器资源

标签（空格分隔）： Docker 监控 容器 container

---

## 应用资源计数实例

基于 *NIX 系统的管理员习惯于查看系统各处的资源计数器，在某些地方，像  /proc， /sys 以及最近的 /cgroup 或是 /sys/fs/cgroup。RHEL6 版本广泛的采用了 Control Groups (cgroups)，cgroups 已经连续稳定的运行了几年，并且通过审查在 Fedora 版本中也是一样稳定的。

实现 cgroups 不仅仅是让系统管理员在多个逻辑分区中划分一个独立的 OS。它也给它们带来了内核维护的每个 cgroup 的计数器。除了常用的用例，比如服务质量保障或是收费。


## Docker 的独特转折

随着采用 Linux 容器技术的数据提升（Docker 是把一些成熟技术封装进一个令人钦佩的可用性包中）。管理员期望每个容器的资源计数器也在里面。非常幸运的是，因为 Docker 严重依赖 Cgroups，很多系统管理员熟悉的计数器是可以使用的。他们可能受益于一些可用性改进。但是如果你通过  cgroup VFS 来探索，你可以非常容易的挖掘出它们。

> 我应该注意到某些层次结构和命令在 RHEL 和 Fedora 是特定的。因此你或许需要为你的系统定制某些路径或包名。


在 Fedora 的最新版本，工程师已经开始构建并发布一个名为 “[nsinit][1]” 的二进制包，它目前是 [libcontainer][2] 的一部分，它是 Docker 的 [execution driver][3]。 nsinit 是一个非常强大的 debugging  工具，系统管理员不仅仅可以看到每个容器的资源计数器，还可以看到容器的运行期配置和“跳进”容器。

## 怎样使用 nsinit 的功能

首先你应该从 Fedora 获取一个副本，或者构建一个你自己的。自己构建是一项没有必要复杂操作。因此我们非常高兴他们开始为 Fedora  构建它以至于你可以这样做：

```
# yum install --enablerepo=updates-testing golang-github-docker-libcontainer

$ rpm -qf `which nsinit`
golang-github-docker-libcontainer-1.1.0-7.git29363e2.fc20.x86_64

# nsinit
NAME:
 nsinit - A new cli application

USAGE:
 nsinit [global options] command [command options] [arguments...]

VERSION:
 0.1

COMMANDS:
 exec 在一个容器中执行一个新命令
 init 在命名空间运行初始化进程
 stats 显示容器的统计信息
 config 显示容器的配置信息
 nsenter 初始化进程用于进入一个已经存在的命名空间
 pause 暂停容器的进程
 unpause 解除暂停的容器进程
 help, h 显示命令列表或是一个命令的帮助信息
```

我覆盖测试了 nsinit 的大部分有用的功能；config， stats 和 exec。

> 注意：nsinit 当前要求你在容器的状态目录运行，因此到目前为止，我们假设你的所有运行的命令都在那里。

因此，像这样做：

```
# docker ps -q
4caad549289

# CID=`docker ps -q`
# cd /var/lib/docker/execdriver/native/$CID*
# ll
total 8
-rw-r-xr-x. 1 root root 3826 Sep  1 20:11 container.json
-rw-r--r--. 1 root root  114 Sep  1 20:11 state.json
```

这些文件时可读的文本格式，尽管不是人类可读的。nsinit 可以完美的打印这些文件。比如，一个 nsinit 配置文件的删减版本输出（完整的版本在[这里][4]）。注意，你可以通过使用 docker inspect 获取更多的这些信息（但不是全部）。

```
# nsinit config

{
 "mount_config": {
 "mounts": [
 {
 "type": "bind",
 "source": "/var/lib/docker/init/dockerinit-1.1.1",
 "destination": "/.dockerinit",
 "private": true
 },
 {
 "type": "bind",
 "source": "/etc/resolv.conf",
 "destination": "/etc/resolv.conf",
 "private": true
 },
<snip>
 "mount_label": "system_u:object_r:svirt_sandbox_file_t:s0:c631,c744"
 },
 "hostname": "4caad5492898",
 "environment": [
 "HOME=/",
 "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/goroot/bin:/gopath/bin",
 "HOSTNAME=4caad5492898",
 "DEBIAN_FRONTEND=noninteractive",
 "GOROOT=/goroot",
 "GOPATH=/gopath"
 ],
 "namespaces": {
 "NEWIPC": true,
 "NEWNET": true,
 "NEWNS": true,
 "NEWPID": true,
 "NEWUTS": true
 },
 "capabilities": [
 "CHOWN",
 "DAC_OVERRIDE",
 "FOWNER",
 "MKNOD",
 "NET_RAW",
 "SETGID",
 "SETUID",
 "SETFCAP",
 "SETPCAP",
 "NET_BIND_SERVICE",
 "SYS_CHROOT",
 "KILL"
 ],
 "networks": [
 {
 "type": "loopback",
 "address": "127.0.0.1/0",
 "gateway": "localhost",
 "mtu": 1500
 },
 {
 "type": "veth",
 "bridge": "docker0",
 "veth_prefix": "veth",
 "address": "172.17.0.6/16",
 "gateway": "172.17.42.1",
 "mtu": 1500
 }
 ],
 "cgroups": {
 "name": "4caad5492898f1a4230353de15e2acfc05809c69d05ec7289c6a14ef6d57b195",
 "parent": "docker",
 "allowed_devices": [
<snip>
 "process_label": "system_u:system_r:svirt_lxc_net_t:s0:c631,c744",
 "restrict_sys": true
}
```

统计模式更有趣，nsinit 读取 cgroup 中 CPU 和 memory 使用计数器。网络统计信息来自于 `/sys/class/net/<EthInterface>/statistics`。从这里你可以看到你的应用使用了多少内存，图表在增长，观察 CPU 使用率，用其他工具交叉检查数据等等。

```
{
 "network_stats": {
 "rx_bytes": 180568,
 "rx_packets": 89,
 "tx_bytes": 28316,
 "tx_packets": 92
 },
 "cgroup_stats": {
 "cpu_stats": {
 "cpu_usage": {
 "total_usage": 985559718,
 "percpu_usage": [
 43613750,
 79789656,
 132486590,
 78759739,
 49063680,
 60703059,
 36277458,
 35919550,
 36329424,
 20096103,
 8148695,
 25279255,
 0,
 0,
 0,
 6144761,
 14814784,
 2612915,
 95162480,
 33853872,
 114861235,
 71115914,
 6533416,
 33993382
 ],
 "usage_in_kernelmode": 510000000,
 "usage_in_usermode": 440000000
 },
 "throlling_data": {}
 },
 "memory_stats": {
 "usage": 27992064,
 "max_usage": 29020160,
 "stats": {
 "active_anon": 4411392,
 "active_file": 3149824,
 "cache": 22278144,
 "hierarchical_memory_limit": 9223372036854775807,
 "hierarchical_memsw_limit": 9223372036854775807,
 "inactive_anon": 0,
 "inactive_file": 19128320,
 "mapped_file": 3723264,
 "pgfault": 94783,
 "pgmajfault": 25,
 "pgpgin": 19919,
 "pgpgout": 13902,
 "rss": 4460544,
 "rss_huge": 2097152,
 "swap": 0,
 "total_active_anon": 4411392,
 "total_active_file": 3149824,
 "total_cache": 22278144,
 "total_inactive_anon": 0,
 "total_inactive_file": 19128320,
 "total_mapped_file": 3723264,
 "total_pgfault": 94783,
 "total_pgmajfault": 25,
 "total_pgpgin": 19919,
 "total_pgpgout": 13902,
 "total_rss": 4460544,
 "total_rss_huge": 2097152,
 "total_swap": 0,
 "total_unevictable": 0,
 "unevictable": 0
 },
 "failcnt": 0
 },
 "blkio_stats": {}
 }
}
```


nsenter 通常用于在一个已经存在的容器中运行一个命令，像这样：

```
# nsenter -m -u -n -i -p -t 19119 bash
```

19119 是容器中一个进程的 PID。丑陋啊，nsinit 使得这个稍微容易些（恕我直言）。

```
# nsinit exec cat /etc/hostname
4caad549289
# nsinit exec bash
bash-4.2# exit
```

当你在调试每个容器的 QoS 实现时，nsinit 的功能和统计信息报告是非常有用的。实现/校验 资源天花板/保障，对你的容器正在做的事情有一个完全的了解。


这个区域是在快速移动的。。。我想介绍另外两个工具，最后应该比 nsinit 有更广阔的适用性。

Google 已经发布了一个叫做 [cAdvisor][5] 的项目，提供了一个基本的 web 接口，但是更重要的 API 是给上一层（比如 Kubernetes）使用了。


Red Hat 提议容器支持 [Performance Co-Pilot][6]，一个在 RHEL7 中系统级别的性能监控工具。随着这个目标教授了很多其他关于容器的工具。


  [1]: https://github.com/docker/libcontainer/blob/master/README.md#nsinit
  [2]: https://github.com/docker/libcontainer
  [3]: http://blog.docker.com/2014/03/docker-0-9-introducing-execution-drivers-and-libcontainer/
  [4]: https://github.com/jeremyeder/docker-performance/blob/master/logs/nsinit-config.out
  [5]: https://github.com/google/cadvisor
  [6]: http://www.performancecopilot.org/