# Docker 资源管理

标签（空格分隔）： Docker 资源管理 内存 CPU 磁盘 I/O

---

> 注：该文作者是 [Marek Goldmann][1]，原文是 [Resource management in Docker][2]。

在这篇博客文章中，我想谈谈 Docker 容器资源管理的话题，我们往往不清楚它是怎样工作的以及我们能做什么不能做什么。我希望你读完这篇关于资源管理的博客文章之后，可以帮助你更容易理解这些。

> 注意：我们假设你在一个 systemd 可用的系统上运行了 Docker。如果你是使用 RHEL/CentOS 7+ 或 Fedora 19+，`systemd` 肯定可用，但是请注意在不同的 systemd 版本之间可能配置选项会有改变。有疑问时，使用你所工作的系统的 systemd 的 man pages。

[TOC]

**目录预览**

##1. 基础概念

Docker 使用 [cgroups][3] 归类运行在容器中的进程。这使你可以管理一组进程的资源，可想而知，这是非常宝贵的。

如果你运行一个操作系统，其使用 [systemd][4] 管理服务。每个进程（不仅仅是容器中的进程）都将被放入一个 cgroups 树中。如果你运行 `systemd-cgls` 命令，你自己可以看到这个结构：

```
$ systemd-cgls
├─1 /usr/lib/systemd/systemd --switched-root --system --deserialize 22
├─machine.slice
│ └─machine-qemu\x2drhel7.scope
│   └─29898 /usr/bin/qemu-system-x86_64 -machine accel=kvm -name rhel7 -S -machine pc-i440fx-1.6,accel=kvm,usb=off -cpu SandyBridge -m 2048
├─system.slice
│ ├─avahi-daemon.service
│ │ ├─ 905 avahi-daemon: running [mistress.local
│ │ └─1055 avahi-daemon: chroot helpe
│ ├─dbus.service
│ │ └─890 /bin/dbus-daemon --system --address=systemd: --nofork --nopidfile --systemd-activation
│ ├─firewalld.service
│ │ └─887 /usr/bin/python -Es /usr/sbin/firewalld --nofork --nopid
│ ├─lvm2-lvmetad.service
│ │ └─512 /usr/sbin/lvmetad -f
│ ├─abrtd.service
│ │ └─909 /usr/sbin/abrtd -d -s
│ ├─wpa_supplicant.service
│ │ └─1289 /usr/sbin/wpa_supplicant -u -f /var/log/wpa_supplicant.log -c /etc/wpa_supplicant/wpa_supplicant.conf -u -f /var/log/wpa_supplica
│ ├─systemd-machined.service
│ │ └─29899 /usr/lib/systemd/systemd-machined

[SNIP]
```

当我们想管理资源的时候，这个方法提供了很大的灵活性，因为我们可以分别的管理每个组。尽管这篇博客文章着重与容器，同样的原则也适用于其他的进程。

> 注意：如果你想知道更多关于 systemd 的知识，我强烈推荐 RHEL 7 的 [Resource Management and Linux Containers Guide][5] 。


###1.1 测试说明

在我的例子中，我将使用 `stress` 工具来帮助我生成容器的一些负载，因此我可以实际看到资源的申请限制。我使用这个 `Dockerfile` 创建了一个名为 `stress` 的定制的 Docker 镜像：

```
FROM fedora:latest
RUN yum -y install stress && yum clean all
ENTRYPOINT ["stress"]
```

###1.2 关于资源报告工具的说明

你使用这个工具来报告如 `top`， `/proc/meminfo` 等等 cgroups 不知道的使用情况。这意味着你将报告关于这台主机的信息，尽管我们在容器中运行着它们。我发现了一篇很好来自于 Fabio Kung 的关于这个主题的[文章][6]。读一读它吧。

因此，我们能做什么？

如果你想快速发现在该主机上哪个容器（或是最近的任何 systemd 服务）使用最多资源，我推荐 `systemd-cgtop` 命令：

```
$ systemd-cgtop
Path                                    Tasks   %CPU   Memory  Input/s Output/s

/                                         226   13.0     6.7G        -        -
/system.slice                              47    2.2    16.0M        -        -
/system.slice/gdm.service                   2    2.1        -        -        -
/system.slice/rngd.service                  1    0.0        -        -        -
/system.slice/NetworkManager.service        2      -        -        -        -

[SNIP]
```

这个工具能立刻给你一个快速预览什么运行在系统上。但是如果你想得到关系使用情况的更详细信息（比如，你需要创建一个好看的图表），你将去分析 `/sys/fs/cgroup/…​` 目录。我将向你展示去哪里能找到我们将讨论的每个资源的有用文件（看下面的 CGroups fs 段落）。

##2. CPU

Docker 能够指定（通过运行[命令的 `-c` 开关][7]）给一个容器的可用的 CPU 分配值。这是一个相对权重，与实际的处理速度无关。事实上，没有办法说一个容器应该只获得 1Ghz CPU。记住。

默认，每个新的容器将有 `1024` shares 的 CPU，当我们单独讲它的时候，这个值并不意味着什么。但是如果我们启动两个容器并且两个都将使用 100% CPU，CPU 时间将在这两个容器之间平均分割，因为它们两个都有同样的 CPU shares（为了简单起见，我假设它们没有任何进程在运行）。

如果我们设置容器的 CPU shares 是 `512`，相对于另外一个容器，它将使用一半的 CPU 时间。但这不意味着它仅仅能使用一半的 CPU 时间。如果另外一个容器（1024 shares 的）是空闲的 - 我们的容器将被允许使用 100% CPU。这是需要注意的另外一件事。

限制是仅仅当它们应该被执行的时候才会强制执行。CGroups 不限制进程预先使用（比如，不允许它们允许的更快，即使它们有空余资源）。相反它提供了它尽可能提供的，以及它仅仅在必需的时候限制（比如，当太多的进程同时大量使用 CPU）。

当然，这很难说清楚（我想说的是这不可能说清楚的）多少资源应该被分配给你的进程。这实际取决于其他进程的行为和多少 shares 被分配给它们。


###2.1 示例：管理一个容器的 CPU 分配

正如我在前面提到的，你可以使用 `-c` 开关来分配给运行在容器中的所有进程的 shares 值。

因为在我的机器上我有 4 核，我将使用 4 压测：

```
$ docker run -it --rm stress --cpu 4
stress: info: [1] dispatching hogs: 4 cpu, 0 io, 0 vm, 0 hdd
```

如果你想以相同的方式启动两个容器，两个都将使用 50% 左右的 CPU。但是当我们修改其中一个容器的 shares 时，将发生什么？

```
$ docker run -it --rm -c 512 stress --cpu 4
stress: info: [1] dispatching hogs: 4 cpu, 0 io, 0 vm, 0 hdd
```
![此处输入图片的描述][8]

正如你所看到的，CPU 在两个容器之间是以这样的方式分割了，第一个容器使用了 60% 的 CPU，另外一个使用了 30% 左右。这似乎是预期的结果。

>注：丢失的 10% CPU 被 GNOME，Chrome 和我的音乐播放器使用了。


###2.2 Attaching containers to cores

除了限制 CPU 的 shares（股份，相当于配额的意思），我们可以做更多的事情，我们可以把容器的进程固定到特定的处理器（core）。为了做到这个，我们使用 `docker run` 命令的 `--cpuset` 开关。

为了允许仅在第一个核上执行：

```
docker run -it --rm --cpuset=0 stress --cpu 1
```

为了允许仅在第一个和第二个核上执行：

```
docker run -it --rm --cpuset=0,1 stress --cpu 2
```

你当然可以混合使用选项 `--cpuset` 和 `-c`。

> 注意：Share enforcement 仅仅发生在当进程运行在相同的核上的时候。这意味着如果你把一个容器固定在第一个核，而把另外一个容器固定在另外一个核，两个都将使用各自核的 100%，即使它们有不同的 CPU shares 设置（再一次，我假设仅仅有两个容器运行在主机上）。


###2.3 变更一个正在运行的容器的分配值

有可能改变一个正在运行的容器的 shares（或是任何进程）。你可以直接与 cgroups 文件系统交互，但是因为我们有 systemds，我们可以通过它来为我们管理。

为了这个目的，我们将使用 `systemctl` 命令和 `set-property` 参数。使用 `docker run` 命令新的容器将有一个 systemd scope，自动分配到其内的所有进程都将被执行。为了改变容器中所有进程的 CPU share，我们仅仅需要在 scope 内改变它，像这样：

```
$ sudo systemctl set-property docker-4be96b853089bc6044b29cb873cac460b429cfcbdd0e877c0868eb2a901dbf80.scope CPUShares=512
```

> 注意：添加 `--runtime` 暂时的改变设置。否则，当主机重起的时候，这个设置会被记住。


把默认值从 `1024` 变更到 `512`。你可以在下面看到结果。这一变化发生在记录的中间。请注意 CPU 使用率。在 systemd-cgtop 中 100% 意味着满额使用了一核，并且这是正确的，因为我绑定了两个容器在相同的核上。

> 注意：为了显示所有的属性，你可以使用 `systemctl show docker-4be96b853089bc6044b29cb873cac460b429cfcbdd0e877c0868eb2a901dbf80.scope` 命令。为了列出所有可用的属性，请看下 `man systemd.resource-control`。


###2.4 CGroups fs

你可以在 `/sys/fs/cgroup/cpu/system.slice/docker-$FULL_CONTAINER_ID.scope/` 下面发现指定容器的关于 CPU 的所有信息，例如：

```
$ ls /sys/fs/cgroup/cpu/system.slice/docker-6935854d444d78abe52d629cb9d680334751a0cda82e11d2610e041d77a62b3f.scope/
cgroup.clone_children  cpuacct.usage_percpu  cpu.rt_runtime_us  tasks
cgroup.procs           cpu.cfs_period_us     cpu.shares
cpuacct.stat           cpu.cfs_quota_us      cpu.stat
cpuacct.usage          cpu.rt_period_us      notify_on_release
```

> 注意：关于这些文件的更多信息，请移步 [RHEL Resource Management Guide][9] 查看。

###2.5 概要重述

需要记住的一些事项：

 - CPU share 仅仅是一个数字 - 与 CPU 速度无关
 - 新容器默认有 `1024` shares
 - 在一台空闲主机上，低 shares 的容器仍可以使用 100% 的 CPU
 - 如果你想，你可以把容器固定到一个指定核

 
##3. 内存

现在让我看下限制内存。

第一件事需要注意的是，默认一个容器可以使用主机上的所有内存。

如果你想为容器中的所有进程限制内存，使用 `docker run` 命令的 `-m` 开关即可。你可以使用 bytes 定义它的值或是（k， m 或 g）。

###3.1 示例：管理一个容器的内存分配

你可以像这样使用 `-m` 开关：

```
$ docker run -it --rm -m 128m fedora bash
```
为了显示限制的实际情况，我将再次使用我的 `stress` 镜像。考虑一下的运行：

```
$ docker run -it --rm -m 128m stress --vm 1 --vm-bytes 128M --vm-hang 0
stress: info: [1] dispatching hogs: 0 cpu, 0 io, 1 vm, 0 hdd
```

`stress` 工具将创建一个进程，并尝试分配 128MB 内存给它。它工作的很好，但是如果我们使用的比实际分配给容器的更多，将发生什么？

```
$ docker run -it --rm -m 128m stress --vm 1 --vm-bytes 200M --vm-hang 0
stress: info: [1] dispatching hogs: 0 cpu, 0 io, 1 vm, 0 hdd
```

它照样正常工作，是不是很奇怪？是的，我同意。

我们将在 [libcontainer 源码][10] 找到解释（cgroups 的 Docker 接口）。我们可以看到源码中默认的 `memory.memsw.limit_in_bytes` 值是被设置成我们指定的内存参数的两倍，当我们启动一个容器的时候。`memory.memsw.limit_in_bytes` 参数表达了什么？它是 [memory 和 swap][11] 的总和。这意味着 Docker 将分配给容器 `-m` 内存值以及 `-m` swap 值。

当前的 Docker 接口不允许我们指定（或者是完全禁用它）多少 swap 应该被使用，所以我们现在需要一起使用它。

有了以上信息，我们可以再次运行我们的示例。这次我们尝试分配超过我们分配的两倍内存。它将使用所有的内存和所有的 swap，然后玩完了。

```
$ docker run -it --rm -m 128m stress --vm 1 --vm-bytes 260M --vm-hang 0
stress: info: [1] dispatching hogs: 0 cpu, 0 io, 1 vm, 0 hdd
stress: FAIL: [1] (415) <-- worker 6 got signal 9
stress: WARN: [1] (417) now reaping child worker processes
stress: FAIL: [1] (421) kill error: No such process
stress: FAIL: [1] (451) failed run completed in 5s
```

如果你尝试再次分配比如 250MB（--vm-bytes 250M），它将工作的很好。

> 警告：如果你不通过 `-m` 开关限制内存，swap 也被不会被限制[^footnote1]。

不限制内存将导致将导致一个容器可以很容易使得整个系统不可用的问题。因此请记住要一直使用 `-m` 参数[^footnote2]。

###3.2 CGroups fs

你可以在 `/sys/fs/cgroup/memory/system.slice/docker-$FULL_CONTAINER_ID.scope/` 下面发现关于内存的所有信息，例如：

```
$ ls /sys/fs/cgroup/memory/system.slice/docker-48db72d492307799d8b3e37a48627af464d19895601f18a82702116b097e8396.scope/
cgroup.clone_children               memory.memsw.failcnt
cgroup.event_control                memory.memsw.limit_in_bytes
cgroup.procs                        memory.memsw.max_usage_in_bytes
memory.failcnt                      memory.memsw.usage_in_bytes
memory.force_empty                  memory.move_charge_at_immigrate
memory.kmem.failcnt                 memory.numa_stat
memory.kmem.limit_in_bytes          memory.oom_control
memory.kmem.max_usage_in_bytes      memory.pressure_level
memory.kmem.slabinfo                memory.soft_limit_in_bytes
memory.kmem.tcp.failcnt             memory.stat
memory.kmem.tcp.limit_in_bytes      memory.swappiness
memory.kmem.tcp.max_usage_in_bytes  memory.usage_in_bytes
memory.kmem.tcp.usage_in_bytes      memory.use_hierarchy
memory.kmem.usage_in_bytes          notify_on_release
memory.limit_in_bytes               tasks
memory.max_usage_in_bytes
```

> 注意：想了解关于这些文件的更多信息，请移步到 [RHEL Resource Management Guide, memory section。][12]

##4. 块设备（磁盘）

对于 块设备，我们可以考虑两种不同类型的限制：

 - 读写速率
 - 可写的空间 (定额)

第一个是非常容易实施的，但是第二个仍未解决。

> 注意：我假设你正在使用 [devicemapper storage][13] 作为 Docker 的后端。使用其他后端，任何事情都将不是确定的。

###4.1 限制读写速率

Docker 没有提供任何的开关来定义我们可以多快的读或是写数据到块设备中。但是 CGroups 内建了。它甚至通过 `BlockIO*` 属性暴露给了 systemd。

为了限制读写速率我们可以分别使用 `BlockIOReadBandwidth` 和 ` BlockIOWriteBandwidth` 属性。

默认 bandwith 是没有被限制的。这意味着一个容器可以使得硬盘”热“，特别是它开始使用 swap 的时候。

###4.2 示例：限制写速率

让我测试没有执行限制的速率：

```
$ docker run -it --rm --name block-device-test fedora bash
bash-4.2# time $(dd if=/dev/zero of=testfile0 bs=1000 count=100000 && sync)
100000+0 records in
100000+0 records out
100000000 bytes (100 MB) copied, 0.202718 s, 493 MB/s

real  0m3.838s
user  0m0.018s
sys   0m0.213s
```

花费了 3.8秒来写入 100MB 数据，大概是 26MB/s。让我们尝试限制一点磁盘的速率。

为了能调整容器可用的 bandwitch，我们需要明确的知道容器挂载的文件系统在哪里。当你在容器里面执行 `mount` 命令的时候，你可以发现它，发现设备挂载在 root 文件系统：

```
$ mount
/dev/mapper/docker-253:0-3408580-d2115072c442b0453b3df3b16e8366ac9fd3defd4cecd182317a6f195dab3b88 on / type ext4 (rw,relatime,context="system_u:object_r:svirt_sandbox_file_t:s0:c447,c990",discard,stripe=16,data=ordered)
proc on /proc type proc (rw,nosuid,nodev,noexec,relatime)
tmpfs on /dev type tmpfs (rw,nosuid,context="system_u:object_r:svirt_sandbox_file_t:s0:c447,c990",mode=755)

[SNIP]
```

在我们的示例中是 `/dev/mapper/docker-253:0-3408580-d2115072c442b0453b3df3b16e8366ac9fd3defd4cecd182317a6f195dab3b88`。

你也可以使用 `nsenter` 得到这个值，像这样：

```
$ sudo /usr/bin/nsenter --target $(docker inspect -f '{{ .State.Pid }}' $CONTAINER_ID) --mount --uts --ipc --net --pid mount | head -1 | awk '{ print $1 }'
/dev/mapper/docker-253:0-3408580-d2115072c442b0453b3df3b16e8366ac9fd3defd4cecd182317a6f195dab3b88
```

现在我们可以改变 `BlockIOWriteBandwidth` 属性的值，像这样：

```
$ sudo systemctl set-property --runtime docker-d2115072c442b0453b3df3b16e8366ac9fd3defd4cecd182317a6f195dab3b88.scope "BlockIOWriteBandwidth=/dev/mapper/docker-253:0-3408580-d2115072c442b0453b3df3b16e8366ac9fd3defd4cecd182317a6f195dab3b88 10M"
```
这应该把磁盘的速率限制在 10MB/s，让我们再次运行 `dd`：

```
bash-4.2# time $(dd if=/dev/zero of=testfile0 bs=1000 count=100000 && sync)
100000+0 records in
100000+0 records out
100000000 bytes (100 MB) copied, 0.229776 s, 435 MB/s

real  0m10.428s
user  0m0.012s
sys   0m0.276s
```

可以看到，它花费了 10s 来把 100MB 数据写入磁盘，因此这速率是 10MB/s。

>注意：你可以使用 `BlockIOReadBandwidth` 属性同样的限制你的读速率

###4.3 限制磁盘空间

正如我前面提到的，这是艰难的话题，默认你每个容器有 10GB 的空间，有时候它太大了，有时候不能满足我们所有的数据放在这里。不幸的是，我们不能为此做点什么。

我们能做的唯一的事情就是新容器的默认值，如果你认为一些其他的值（比如 5GB）更适合你的情况，你可以通过指定 Docker daemon 的 `--storage-opt`来实现，像这样：

```
docker -d --storage-opt dm.basesize=5G
```

你可以[调整一些其他的东西][14]，但是请记住，这需要在后面重起你的 Docker daemon，想了解更多的信息，请看[这里][15]。

###4.4 CGroups fs

你可以在 `/sys/fs/cgroup/blkio/system.slice/docker-$FULL_CONTAINER_ID.scope/` 目录下发现关于块设备的所有信息，例如：

```
$ ls /sys/fs/cgroup/blkio/system.slice/docker-48db72d492307799d8b3e37a48627af464d19895601f18a82702116b097e8396.scope/
blkio.io_merged                   blkio.sectors_recursive
blkio.io_merged_recursive         blkio.throttle.io_service_bytes
blkio.io_queued                   blkio.throttle.io_serviced
blkio.io_queued_recursive         blkio.throttle.read_bps_device
blkio.io_service_bytes            blkio.throttle.read_iops_device
blkio.io_service_bytes_recursive  blkio.throttle.write_bps_device
blkio.io_serviced                 blkio.throttle.write_iops_device
blkio.io_serviced_recursive       blkio.time
blkio.io_service_time             blkio.time_recursive
blkio.io_service_time_recursive   blkio.weight
blkio.io_wait_time                blkio.weight_device
blkio.io_wait_time_recursive      cgroup.clone_children
blkio.leaf_weight                 cgroup.procs
blkio.leaf_weight_device          notify_on_release
blkio.reset_stats                 tasks
blkio.sectors
```

> 注意：想了解关于这些文件的更多信息，请移步到 [RHEL Resource Management Guide, blkio section][16]。


##总结

正如你所看到的，Docker 容器的资源管理是可行的。甚至非常简单。唯一的事情就是我们不能为磁盘使用设置一个定额，这有一个上游[问题列表][17] -- 跟踪它并且评论。

希望你发现我的文章对你有用。


  [1]: https://goldmann.pl/socially/
  [2]: https://goldmann.pl/blog/2014/09/11/resource-management-in-docker/
  [3]: https://www.kernel.org/doc/Documentation/cgroups/cgroups.txt
  [4]: http://www.freedesktop.org/wiki/Software/systemd/
  [5]: https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/7/html/Resource_Management_and_Linux_Containers_Guide/index.html
  [6]: http://fabiokung.com/2014/03/13/memory-inside-linux-containers/
  [7]: http://docs.docker.com/reference/run/#runtime-constraints-on-cpu-and-memory
  [8]: https://goldmann.pl/images/docker-resources/stress-half.png
  [9]: https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/6/html/Resource_Management_Guide/index.html
  [10]: https://github.com/docker/libcontainer/blob/v1.2.0/cgroups/fs/memory.go#L39
  [11]: https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/6/html/Resource_Management_Guide/sec-memory.html#important-Order-of-setting-memory.limit_in_bytes-and-memory.memsw.limit_in_bytes
  [12]: https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/6/html/Resource_Management_Guide/sec-memory.html
  [13]: https://github.com/docker/docker/tree/v1.2.0/daemon/graphdriver/devmapper
  [14]: https://github.com/docker/docker/blob/v1.2.0/daemon/graphdriver/devmapper/README.md
  [15]: https://github.com/docker/docker/blob/v1.2.0/daemon/graphdriver/devmapper/README.md
  [16]: https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/6/html/Resource_Management_Guide/ch-Subsystems_and_Tunable_Parameters.html#sec-blkio
  [17]: https://github.com/docker/docker/issues/3804
  
  [^footnote1]: 这在技术上是不正确的； 这有限度, 但是它设置的值在我们当前运行的系统是不可达的。 例如在我的笔记本上 16GB 的内存值是 18446744073709551615，这是 ~18.5 exabytes…
  
  [^footnote2]: 或者是使用 `MemoryLimit` 属性。