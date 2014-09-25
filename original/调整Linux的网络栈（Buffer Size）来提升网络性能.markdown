# 调整Linux的网络栈（Buffer Size）来提升网络性能

标签（空格分隔）： Linux Network Performance 

---

[『Write by NIX CRAFT 』](http://www.cyberciti.biz/tips/about-us)

基于CENTOS 、DEBIAN/UBUNTU 。

> 我有两台位于不同数据中心的服务器，都用来处理很多并行的大文件传输。但是处理大文件，网络性能非常差。并且涉及到一个大文件，会导致性能降级。我怎样通过调整Linux下面的 TCP 来解决这个问题？

默认，Linux的stack是没有为广域网之间的大文件高速传输而配置的，这样做是为了节约内存资源。为了使连接的系统服务之间能有更加高速的网络处理更多的网络包，你可以很容易的通过增加网络 buffer size 来调整 Linux 网络 stack 。

默认的 Linux  buffer size 的最大值是非常小的，tcp 的内存是基于系统的内存自动计算的，你能通过键入以下命令找到实际的值：

```
 $ cat /proc/sys/net/ipv4/tcp_mem
```

默认的和最大的接收数据包内存大小：

```
$ cat /proc/sys/net/core/rmem_default
$ cat /proc/sys/net/core/rmem_max
```

默认的和最大的发送数据包内存的大小：

```
$ cat /proc/sys/net/core/wmem_default
$ cat /proc/sys/net/core/wmem_max
```

最大的内存 buffers 的选项：

```
$ cat /proc/sys/net/core/optmem_max
```

## 调整值

为所有的协议队列设置操作系统层面的最大的发送 buffer size (wmem) 和 接收 buffer size （rmem）为 12 MB。换句话说，设置内存数量，分配给每一个为了传送文件而打开或者是创建的 tcp socket 。

> ![警告](http://figs.cyberciti.biz/warning-40px.png) 警告！在大多数的 Linux 中 ```rmem_max``` 和 ```wmem_max``` 被分配的值为 128 k，在一个低延迟的网络环境中，或者是 apps 比如 DNS、Web Server，这或许是足够的。尽管如此，如果延迟太大，默认的值可能就太小了，所以请记录以下在你的服务器上用来提高内存使用方法的设置。

```
# echo 'net.core.wmem_max=12582912' >> /etc/sysctl.conf
# echo 'net.core.rmem_max=12582912' >> /etc/sysctl.conf
```

你还需要设置 minimum size, initial size, and maximum size in bytes:

```
# echo 'net.ipv4.tcp_rmem= 10240 87380 12582912' >> /etc/sysctl.conf
# echo 'net.ipv4.tcp_wmem= 10240 87380 12582912' >> /etc/sysctl.conf
```

打开  window scaling ，这是一个用来扩展传输窗口的选项：

```
# echo 'net.ipv4.tcp_window_scaling = 1' >> /etc/sysctl.conf
```

确保定义在 RFC1323 中的 ```timestamps``` 打开：

```
# echo 'net.ipv4.tcp_timestamps = 1' >> /etc/sysctl.conf
```

确保 select acknowledgments：
```
# echo 'net.ipv4.tcp_sack = 1' >> /etc/sysctl.conf
```
> 这个 “select acknowledgments” 不知道该如何翻译，翻译为“选择确认？”

当连接关闭的时候，TCP 默认缓存了很多连接指标在 route cache 中，以至于在不久的将来，连接建立的时候，可以用这些值来设置初始化条件。通常，这提升了整体的性能，但是，有时候会引起性能下降， 如果设置的话，TCP 在关闭的时候不缓存这些指标。

```
# echo 'net.ipv4.tcp_no_metrics_save = 1' >> /etc/sysctl.conf
```

当 interface 接收到的数据包数量比内核处理速度的快的时候， 设置 input 队列最大的 packets 数量值。

```
# echo 'net.core.netdev_max_backlog = 5000' >> /etc/sysctl.conf
```

现在重载这些改变，使其生效：

```
# sysctl -p
```

使用 tcpdump 命令查看 通过 eth0 数据包流量的变化：

```
# tcpdump -ni eth0
```

## 推荐阅读：
- 请参考内核文档[/networking/ip-sysctl.txt](http://www.cyberciti.biz/files/linux-kernel/Documentation/networking/ip-sysctl.txt)获取更加多的信息 

- 请查看 `sysctl` 的 `man` 手册  