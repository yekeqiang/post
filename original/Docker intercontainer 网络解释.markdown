# Docker intercontainer 网络解释

标签（空格分隔）： Docker intercontainer networking 

---

> 注：该文作者是  Attila Kanto，原文地址是 [Docker intercontainer networking explained][1]

这篇文章的目的是覆盖关于 Docker 网络的高级主题和解释当容器运行在不同的主机上时，Docker 容器相互之间连接的不同概念。我们使用 Vagrant 运行 VirtualBox 上的 VMS  来演示。如无特殊说明，关于网络概念的解释全部是基于  Amazon EC2 (with VPC) 和 Azure。

为了设置测试环境，克隆 [SequenceIQ’s samples repository][2] 并使用以下指令：

```
git clone git@github.com:sequenceiq/sequenceiq-samples.git
cd sequenceiq-samples/docker-networking
vagrant up
```

`vagrant up` 命令启动测试设置，其中包含两个有以下网络配置的 Ubuntu 14.04 VMs ：

 - [NAT][3]
 - [Private networking][4]

NAT（涉及 VMS 上的 eth0 接口）仅用于从 VMS 访问外部网络等等。从 debian 仓库下载文件，但它不是用于 inter-container 通信。Vagrant 在 VirtualBox 中设置一个正确的仅仅只有网络的配置主机。因此 VMs 可以使用定义好的 IP 地址相互通信：

 - vm1: 192.168.40.11
 - vm2: 192.168.40.12
 
让我们看下 Docker 容器是怎样运行在这些 VMs 上，并彼此发送 IP 包的。


## 设置 bridge0

Docker 通过 docker0 实现的虚拟子网来连接所有的容器，这意味着默认在两台虚拟机上，Docker 容器都将使用IP地址范围为 172.17.42.1/24 d的 IP 启动。这是一个问题的一些解决方案，解释如下，因为如果容器在不同的主机上有相同的 IP 地址，那我们就不能在它们之间正确路由 IP 包。因此在每台虚拟机上，使用以下子网创建一个网络桥：

  - vm1: 172.17.51.1/24
  - vm2: 172.17.52.1/24

这意味着在 vm1 上启动的每个容器将获取的 IP 地址范围是从 172.17.51.2 – 172.17.51.255。在 vm2 上启动的每个容器获取到的 IP 地址范围是从  172.17.52.2 – 172.17.52.255。

```
# do not execute, it was already executed on vm1 as root during provision from Vagrant
brctl addbr bridge0
sudo ifconfig bridge0 172.17.51.1 netmask 255.255.255.0
sudo bash -c 'echo DOCKER_OPTS=\"-b=bridge0\" >> /etc/default/docker'
sudo service docker restart

# do not execute, it was already executed on vm1 as root during provision from Vagrant
sudo brctl addbr bridge0
sudo ifconfig bridge0 172.17.52.1 netmask 255.255.255.0
sudo bash -c 'echo DOCKER_OPTS=\"-b=bridge0\" >> /etc/default/docker'
sudo service docker restart
```


请注意以上配置中的注释，这些动作在配置虚拟机的时候已经执行了，复制在这里只是为了清晰性和完整性。

## 暴露容器的端口给主机

可能解决 inter-container 通信的最简单的方式就是暴露容器的端口给主机。这个可以使用 `-p` 来完成。比如，暴露端口 3333 是一样简单：

```
# execute on vm1
sudo docker run -it --rm --name cont1 -p 3333:3333 ubuntu /bin/bash -c "nc -l 3333"

# execute on vm2
sudo docker run -it --rm --name cont2 ubuntu /bin/bash -c "nc -w 1 -v 192.168.40.11 3333"
#Result: Connection to 192.168.40.11 3333 port [tcp/*] succeeded!
```

当通信端口被定义好（比如，MySQL 将运行在 3306 端口），这可能是一个很合适的示例，但是当应用程序使用动态端口通信的时候（像  Hadoop does with IPC ports），它将不能正常工作。

## 主机网络

如果容器是以 `--net=host` 启动的，它就避免了把容器放在一个单独网络栈里面，但是在 Docker 的文档中说这个选项“告诉 Docker 不要集容器化容器的网络”。`cont1` 容器可以直接绑定到主机的网络接口，因此 `nc` 在 192.168.40.11 上是可用的。

```
# execute on vm1
sudo docker run -it --rm --name cont1 --net=host ubuntu /bin/bash -c "nc -l 3333"

# execute on vm2
sudo docker run -it --rm --name cont2 ubuntu /bin/bash -c "nc -w 1 -v 192.168.40.11 3333"
#Result: Connection to 192.168.40.11 3333 port [tcp/*] succeeded!
```

当然，如果你想从 `cont1` 访问 `cont2`，你也需要以 `--net=host` 选项开始，主机网络对于  inter-container 通信是非常有用的解决方案，但是它也有自己的缺点，因为被容器使用的端口可能与主机使用的端口或是其他容器使用 `--net=host` 冲突。因为它们所有的都是共享同样的网络栈。


## 直接路由

到目前为止，我们已经看到的方法是容器使用主机的 IP 地址彼此通信。但是也有使用它们自己的 IP 来做容器互连的解决方案。如果我们使用容器自己的 IP 路由，那能基于 IP 区分哪个容器是运行在 vm1 和 哪个容器是运行在 vm2就相当重要。这就是如在“设置 bridge0”一节中解释那样，  bridge0  接口被创建的原因。为了使得事情更容易理解一点，我已经创建了一个在我们当前测试设置中的简单的网络接口图表。

![此处输入图片的描述][5]

为了达到这个目的，我们需要在主机上配置路由表，被转发到 vm1 的每个包的目的地都是 172.17.51.0/24，被转发到 vm2 的每个包的目的地都是 172.17.52.0/24。运行在 vm1 上的容器被放置在子网  172.17.51.0/24，运行在 vm2 的容器被放置在子网  172.17.52.0/24。

```
# execute on vm1
sudo route add -net 172.17.52.0 netmask 255.255.255.0 gw 192.168.40.12
sudo iptables -t nat -F POSTROUTING
sudo iptables -t nat -A POSTROUTING -s 172.17.51.0/24 ! -d 172.17.0.0/16 -j MASQUERADE
sudo docker run -it --rm --name cont1  ubuntu /bin/bash
#Inside the container (cont1)
nc -l 3333

# execute on vm2
sudo route add -net 172.17.51.0  netmask 255.255.255.0  gw 192.168.40.11
sudo iptables -t nat -F POSTROUTING
sudo iptables -t nat -A POSTROUTING -s 172.17.52.0/24 ! -d 172.17.0.0/16 -j MASQUERADE
sudo docker run -it --rm --name cont2  ubuntu /bin/bash
#Inside the container (cont2)
nc -w 1 -v 172.17.51.2 3333
#Result: Connection to 172.17.51.2 3333 port [tcp/*] succeeded!
```

`route add` 命令添加所需的路由到路由表中，但你可能想知道为什么 iptables 配置是必需的。原因就是 Docker 默认给 nat 表设置了一条规则伪装所有的 IP 包离开机器。在我们的情况中，我们明确不需要这个，因此我们使用 `-F` 选项删除了所有的 MASQUERADE 规则，在这一点上，我们已经可以从这个容器连接到其他容器，反之亦然。但是容器不能与外部世界通信，因此一条 iptables 规则需要被添加用来伪装数据包是从 `172.17.0.0/16` 出去的。我需要提到另外一种方法，使用 daemon 的 [—iptables=false][6]  选项来避免在 iptables 中的任何篡改，你可以手动的做所有的配置。

像这样的直接路由，从一台 vm 到另外一台 vm 是非常简单并且容易设置的，但是不适用于主机不在同一个子网的情况。如果主机位于不同的子网，tunneling 或许是一个选择，你将在下一章节看到。


> 注意：如果 [Source/Destionation Check][7] 禁用时，该解决方案仅仅运行在  Amazon EC2 实例上。

> 注意：由于 Azure 的包过滤策略，该方法是失效的。


## Generic Routing Encapsulation (GRE) tunnel


GRE 是一个隧道协议，它可以封装多种网络层协议进虚拟点对点的连接。大意是在虚拟机直接创建一个  GRE 隧道，并且通过它发送所有的流量。

![此处输入图片的描述][8]


为了创建一个隧道，你需要指定一个名字，一个类型（在我们的案例中是 gre ），本机和远程的 IP 地址等等。因此，tun2 名字用于 vm1 上的隧道，因为从 vm1 的视角它是通向 vm2 的隧道终点。每个发往 tun2 的包最终在 vm2 出现。

```
#GRE tunnel config execute on vm1
sudo iptunnel add tun2 mode gre local 192.168.40.11 remote 192.168.40.12
sudo ifconfig tun2 10.0.201.1
sudo ifconfig tun2 up
sudo route add -net 172.17.52.0 netmask 255.255.255.0 dev tun2
sudo iptables -t nat -F POSTROUTING
sudo iptables -t nat -A POSTROUTING -s 172.17.51.0/24 ! -d 172.17.0.0/16 -j MASQUERADE
sudo docker run -it --rm --name cont1  ubuntu /bin/bash
#Inside the container (cont1)
nc -l 3333

#GRE tunnel config execute on vm2
sudo iptunnel add tun1 mode gre local 192.168.40.12 remote 192.168.40.11
sudo ifconfig tun1 10.0.202.1
sudo ifconfig tun1 up
sudo route add -net 172.17.51.0 netmask 255.255.255.0 dev tun1
sudo iptables -t nat -F POSTROUTING
sudo iptables -t nat -A POSTROUTING -s 172.17.52.0/24 ! -d 172.17.0.0/16 -j MASQUERADE
sudo docker run -it --rm --name cont2  ubuntu /bin/bash
#Inside the container (cont2)
nc -w 1 -v 172.17.51.2 3333
#Result: Connection to 172.17.51.2 3333 port [tcp/*] succeeded!
```

tunnel 设置后，剩下的命令激活和在“直接路由”章节的命令非常类似。主要的不同是我们没有直接路由流量到其他 VM，但是我们把它路由到 `dev tun1` 和 `dev tun2` 来代替。


GRE 隧道是设置一个两台主机直接点到点的连接，这意味着，如果在你的网络中有多个主机，并且想互连它们，这时候  n-1 tunnel endpoint 需要在每台主机上创建，如果你有一个非常大的集群，维护这些是一个非常大的挑战。

## Virtual Private Network (VPN)


如果在容器直接要求安全连接，这时， VPNs 可以被用在 VMs 上。额外的安全可能显著的提高处理开销。这个负载高度依赖你使用的 VPN 解决方案。在这个演示中，我们使用 SSH 的 VPN 功能，这不适用于生产环境。为了启用 SSH 的 VPN 功能，在 sshd_config 文件中，PermitTunnel 参数需要被切换。如果你是使用的本教程提供的 Vagranfile ，那什么事情也不需要做，因为该参数已经在配置  bootstrap.sh 脚本的时候设置了。

```
#execute on vm1
sudo ssh -f -N -w 2:1 root@192.168.40.12
sudo ifconfig tun2 up
sudo route add -net 172.17.52.0 netmask 255.255.255.0 dev tun2
sudo iptables -t nat -F POSTROUTING
sudo iptables -t nat -A POSTROUTING -s 172.17.51.0/24 ! -d 172.17.0.0/16 -j MASQUERADE
sudo docker run -it --rm --name cont1  ubuntu /bin/bash
#Inside the container (cont1)
nc -l 3333

#execute on vm2
sudo ifconfig tun1 up
sudo route add -net 172.17.51.0 netmask 255.255.255.0 dev tun1
sudo iptables -t nat -F POSTROUTING
sudo iptables -t nat -A POSTROUTING -s 172.17.52.0/24 ! -d 172.17.0.0/16 -j MASQUERADE
sudo docker run -it --rm --name cont2  ubuntu /bin/bash
#Inside the container (cont2)
nc -w 1 -v 172.17.51.2 3333
#Result: Connection to 172.17.51.2 3333 port [tcp/*] succeeded!
```

ssh 使用 `-w` 选项启动，tun devices 指定的数值 ID。执行这个命令之后，tunnel 接口在两台 VMs 上都被创建了。该接口需要使用 ` ifconfig up` 命令激活，然后我们需要设置支持直接流量到 172.17.51.0/24 和  172.17.52.0/24 通过  tun2 和 tun1。 

如上所述，SSH 的 VPN 功能不建议在生产环境使用，但是另外的解决方案像  [OpenVPN][9] 值得尝试一下来加密你的主机之间的通信（容器之间也一样）。

## 总结

以上所有示例都是用于演示的目的，但是也有很好的工具像 [Pipework][10] 能使得你的生活更简单，并且是给你的一个大礼物。

如果你想校验这些方法在生产环境怎样工作，你只需几次点击，因为这些方法是负责解决我们的 cloud agnostic Hadoop 的 inter-container 通信的，作为一个服务 API 调用 Cloudbreak。


  [1]: http://blog.sequenceiq.com/blog/2014/08/12/docker-networking/
  [2]: https://github.com/sequenceiq/sequenceiq-samples
  [3]: https://www.virtualbox.org/manual/ch06.html#network_nat
  [4]: https://docs.vagrantup.com/v2/networking/private_network.html
  [5]: https://raw.githubusercontent.com/sequenceiq/sequenceiq-samples/master/docker-networking/img/routing.png
  [6]: https://docs.docker.com/articles/networking/#between-containers
  [7]: http://docs.aws.amazon.com/AmazonVPC/latest/UserGuide/VPC_NAT_Instance.html#EIP_Disable_SrcDestCheck
  [8]: https://raw.githubusercontent.com/sequenceiq/sequenceiq-samples/master/docker-networking/img/gre.png
  [9]: https://openvpn.net/index.php/open-source.html
  [10]: https://github.com/jpetazzo/pipework