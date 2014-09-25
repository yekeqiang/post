# 基于云使用 salt 和 docker 自动化部署应用 

标签（空格分隔）： docker salt automating

---

![enter image description here][1] 如果你还没有机会使用 [Salt][2]的话，那我简单介绍下，Salt 是一个非常强大的配置管理系统，它是一个非常容易运行，并且能支持分布式命令执行和复杂的配置文件管理，具有高可扩展性，能同时支持上千台服务器运行。

> 注： SaltStack 的进一步学习和了解可以加入 [SaltStack 中国用户组][3]，SaltStack 确实是一个非常强大的工具，用 Python 编写的，可以通过 API 进行二次开发

最近，我在写一篇关于一个事实上的集装箱化标准怎么成为[新一代的管理工具][4]的文章。回到一月，[SaltStack][5] 宣布了 ```Salt 2014.1.0``` 版本[一些非常好的新特性][6]，包括支持 [Docker 容器生命周期的管理][7]。SaltStack 早期就得到了 Docker 的支持，Docker 自己认为还没有准备好（tell that to Yelp and Spotify），但是这些工具为固定不变的基础设施提供了一个开箱即用的解决方案。

在一年前，通过使用多个公共云横跨多个虚拟机来部署和管理一个应用，会被认为是非常难的使用案例。像 Google , FaceBook 和 Ning 这些公司为了解决扩展性难题，划分了他们很多年来发展内部的 orchestration 技术。今天，使用 Docker 容器结合 [Salt-Cloud][8]，还有一些 Salt 状态， 我们只需要花费几十分钟就能处理好。并且，因为我们使用了 SaltStack 的声明式配置管理，我们能在实际生产环境中按照以下模式操作。

![enter image description here][9]

这个案例的核心是在一个或多个公有云供应商上，把一个或多个应用部署在一个或者多个虚拟主机上。

为了这个简单的目的，我们将对这个案例做如下限制：

* 假设熟悉 Salt
* 假设熟悉 Docker
* 假设你已经安装了一个 SaltSack Master
* 假设你已经在一个公有云上做了这些事情，使用 Digital Ocean （因为添加一个新的云到 Salt-Cloud 是非常简单的）
* 用一个虚拟的 apache 服务模仿真实世界的应用


![enter image description here][10]

为了模仿一个真实的应用，需要创建一个 Apache 的 Docker 容器。从概念上讲，这个容器可能一个前端代理，一个 Web 中间件服务，一个数据库，或者是我们需要部署在生产上的其他类型的服务。为了这个，我们在一个目录下创建一个 DockerFile，构建这个容器，并且把这个容器 PUSH 到 Docker 仓库。

**Step 1: Create a Dockerfile**
```
##### An apache Dockerfile
root@salt:/some/dir/apache# cat Dockerfile
# A basic apache server. To use either add or bind mount content under /var/www
FROM ubuntu:12.04
 
MAINTAINER Kimbro Staken version: 0.1
 
RUN apt-get update && apt-get install -y apache2 && apt-get clean && rm -rf /var/lib/apt/lists/*
 
ENV APACHE_RUN_USER www-data
ENV APACHE_RUN_GROUP www-data
ENV APACHE_LOG_DIR /var/log/apache2
 
EXPOSE 80
 
CMD ["/usr/sbin/apache2", "-D", "FOREGROUND"]
```
**Step 2: Build the container**

```
##### Building the container
root@salt:/some/dir/apache# docker build -t jthomason/apache .
Uploading context  2.56 kB
Uploading context
Step 0 : FROM ubuntu:12.04
 ---> 1edb91fcb5b5
Step 1 : MAINTAINER Kimbro Staken version: 0.1
 ---> Using cache
 ---> 534b8974c22c
Step 2 : RUN apt-get update && apt-get install -y apache2 && apt-get clean && rm -rf /var/lib/apt/lists/*
 ---> Using cache
 ---> 7d24f67a5573
...
<A WHOLE LOT OF STUFF HAPPENS>
...
Successfully built 527ad6962e09
```

**Step 3: Push the container**

```
##### Pushing the container
root@salt:/tmp/apache# docker push jthomason/apache
The push refers to a repository [jthomason/apache] (len: 1)
...
<A WHOLE LOT OF STUFF HAPPENS>
...
Pushing tag for rev [527ad6962e09] on {https://registry-1.docker.io/v1/repositories/jthomason/apache/tags/latest}
```

当我们的应用 Demo 成功 PUSH 到了 Docker 的仓库后，我们准备开始协调它的部署。需要提前注意的是，我们假设你已经安装了一个 SaltStack ，如果没有，你可以根据这个[文档][11] 来获取 SaltStack Master 安装。下一步，就是在你选择的公有云供应商上配置 Salt-Cloud 。 配置 Salt-Cloud 是非常简单的，我们需要创建一个 SSH Key，Salt-Cloud 用它来在新创建的 VMs 上安装 Salt Minion，把这个 keypair 添加进公有云提供商，使用认证的 API 为我们的公有云创建一个 Salt-Cloud 配置文件。

**Step 4: Create an SSH Key Pair**

```
##### Generate ssh keys for salt-cloud
root@salt:/etc/salt/keys# ssh-keygen -t dsa
Generating public/private dsa key pair.
Enter file in which to save the key (/root/.ssh/id_dsa): digital-ocean-salt-cloud
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in id_dsa.
Your public key has been saved in digital-ocean-salt-cloud.pub.
The key fingerprint is:
06:8f:6f:e1:97:5a:5a:48:ce:09:f3:b6:33:42:48:9a root@salt.garply.org
The key's randomart image is:
+--[ DSA 1024]----+
|                 |
|                 |
|      .          |
|    .  +         |
|   + .+ S        |
|  E . .@ + .     |
|     .  @ =      |
|      .ooB       |
|       .+o       |
+-----------------+
root@salt:/etc/salt/keys#
```

**Step 5: Upload SSH Key Pair**

![enter image description here][12]

使用 Digital Ocean 让我们的 SSH Keys 生效，下一步的准备时间是使用 Digital Ocean’s API key 为我们的账号配置 Salt-Cloud，为 Digital Ocean 虚拟主机的大小，地理位置，镜像 定义配置文件，Salt-Cloud 的认证文件保存在 Salt-Master 的 ```/etc/salt/cloud.providers.d/``` 路径下，查看 Salt-Cloud 的[文档][13]了解更加多的配置文件细节

**Step 6: Configure Salt-Cloud**

```
##### /etc/salt/cloud.providers.d/digital_ocean.conf
do:
  provider: digital_ocean
  # Digital Ocean account keys
  client_key: <YOUR KEY HERE>
  api_key: <YOUR API KEY HERE>
  ssh_key_name: digital-ocean-salt-cloud.pub
  # Directory & file name on your Salt master
  ssh_key_file: /etc/salt/keys/digital-ocean-salt-cloud
```

```
##### /etc/salt/cloud.profiles.d/digital_ocean.conf
# Official distro images available for Arch, CentOS, Debian, Fedora, Ubuntu
 
ubuntu_512MB_sf1:
  provider: do
  image: Ubuntu 12.04.4 x64
  size: 512MB
#  script: Optional Deploy Script Argument
  location: San Francisco 1
  script: curl-bootstrap-git
  private_networking: True
 
ubuntu_1GB_ny2:
  provider: do
  image: Ubuntu 12.04.4 x64
  size: 1GB
#  script: Optional Deploy Script Argument
  location: New York 2
  script: curl-bootstrap-git
  private_networking: True
 
ubuntu_2GB_ny2:
  provider: do
  image: Ubuntu 12.04.4 x64
  size: 2GB
#  script: Optional Deploy Script Argument
  location: New York 2
  script: curl-bootstrap-git
  private_networking: True
```

现在是时候为我们的应用容器配置 Salt。要做到这一点，我们需要创建两个 Salt [States][14] ,一个为在一个新创建的 VMs 上的 Docker 准备，另外一个为应用容器准备。Salt States 是 Salt 的声明式状态配置，它在目标主机上被 salt-minion 执行，States 有非常丰富的功能，几乎很少有像它这样在教程中覆盖了如此之多的各等级的细节，这个不是最聪明或是最优的 States 使用示例，但是它足够简单。你需要仔细研读 Salt States 来发展你自己的环境的最佳实践。

**Step 7: Create a Salt state for Docker**

```
##### /srv/salt/docker/init.sls
docker-python-apt:
  pkg.installed:
    - name: python-apt
 
docker-python-pip:
  pkg.installed:
    - name: python-pip
 
docker-python-dockerpy:
  pip.installed:
    - name: docker-py
    - repo: git+https://github.com/dotcloud/docker-py.git
    - require:
      - pkg: docker-python-pip
 
docker-dependencies:
   pkg.installed:
    - pkgs:
      - iptables
      - ca-certificates
      - lxc
 
docker_repo:
    pkgrepo.managed:
      - repo: 'deb http://get.docker.io/ubuntu docker main'
      - file: '/etc/apt/sources.list.d/docker.list'
      - key_url: salt://docker/docker.pgp
      - require_in:
          - pkg: lxc-docker
      - require:
        - pkg: docker-python-apt
      - require:
        - pkg: docker-python-pip
 
lxc-docker:
  pkg.latest:
    - require:
      - pkg: docker-dependencies
 
docker:
  service.running
```

第一个 salt state 定义了依赖和安装 Docker 的配置

**Step 8: Create Salt state for the application container**

```
##### /srv/salt/apache/init.sls
apache-image:
   docker.pulled:
     - name: jthomason/apache
     - require_in: apache-container
 
apache-container:
   docker.installed:
     - name: apache
     - hostname: apache
     - image: jthomason/apache
     - require_in: apache
 
apache:
   docker.running:
     - container: apache
     - port_bindings:
            "80/tcp":
                HostIp: ""
                HostPort: "80"
```

现在终于配置完成，我们将准备 1-n 个虚拟机，每个运行一个应用容器实例。在我们做这个之前，让我们校验下 Salt Master 是否正常工作，这时候，我们知道至少有一个客户端应该与 Salt Master 通信，这个客户端与 Salt Master 运行在同一台服务器上。

**Step 9: Verify that Salt is working**

```
##### Pinging the minions
root@salt:~# salt '*' test.ping
salt.garply.org:
True
root@salt:~#
```

非常满意，随着 Salt 的安装，一切工作有序。现在我们可以用 Salt-Cloud 在我们的第一个虚拟机上准备我们的一个容器实例。

**Step 10: Provision a VM with an instance of the container**

```
##### Provisioning a VM with an instance of a container
root@salt:# salt-cloud --profile ubuntu_512MB_sf1 one.garply.org
[INFO    ] salt-cloud starting
[INFO    ] Creating Cloud VM one.garply.org
[INFO    ] Rendering deploy script: /usr/lib/python2.7/dist-packages/salt/cloud/deploy/curl-bootstrap-git.sh
...
<A WHOLE LOT OF STUFF HAPPENS>
...
one.garply.org:
    ----------
    backups_active:
        False
    created_at:
        2014-04-23T19:23:12Z
    droplet:
        ----------
        event_id:
            22373933
        id:
            1521385
        image_id:
            3101045
        name:
            one.garply.org
        size_id:
            66
    id:
        1521385
    image_id:
        3101045
    ip_address:
        107.170.230.112
    locked:
        True
    name:
        one.garply.org
    private_ip_address:
        None
    region_id:
        3
    size_id:
        66
    status:
        new
```

当运行完成 Salt-Cloud，它发出一个包含新近创建的 VM 实例的信息的 YAML blob，让我们使用这个实例的 IP 来看下我们的应用是否在运行。

**Step 11: Verify application is running **

![enter image description here][15] 

我们已经为我们的基础设施建立了基本的设置和管理模式，增加额外的公有云是非常简单的，非常感谢 Salt-Cloud 为我们的应用基础设施提供了一个控制接口。在整个持续集成和部署过程，该何去何从？一个起始点是考虑 salt states 怎样被用来管理 VM 和 控制容器的生命周期。我计划在未来的文章中分享我的一些明确的想法。非常明显的，从你最后决定你的应用部署和操作的架构设计，这里会有非常多的想法适用于你的明确目标。不管怎样，Salt 是一个非常强大的工具，当与 Docker 组合的时候，在不可变的基础架构上，提供了一个声明式的框架来管理应用生命周期的开箱即用的范例。That versatility puts a whole lot of miles behind you，允许你专注于应用程序部署与操作的其他核心挑战。


> 注： 该文章由 JAMES THOMASON 编写， 本文的[原文][16]请看这里，如需转载，请注明出处，谢谢！
> 注： 该文由 [Docker 中文社区][17]收集，并且发布翻译任务，各位如果对 Docker 感兴趣，可以加入 Docker 中文社区。


  [1]: http://thomason.io/wp-content/uploads/2014/04/saltstack-logo.png
  [2]: https://github.com/saltstack/salt
  [3]: http://www.saltstack.cn/
  [4]: http://thomason.io/why-containerization-is-a-key-enabling-technology-for-paas/
  [5]: http://www.saltstack.com/
  [6]: http://docs.saltstack.com/en/latest/topics/releases/2014.1.0.html
  [7]: http://docs.saltstack.com/en/latest/ref/modules/all/salt.modules.dockerio.html
  [8]: http://salt-cloud.readthedocs.org/en/latest/
  [9]: http://thomason.io/wp-content/uploads/2014/04/salt-docker-use-cases.png
  [10]: http://thomason.io/wp-content/uploads/2014/04/docker-salt-digitalocean.png
  [11]: http://docs.saltstack.com/en/latest/topics/installation/index.html
  [12]: http://thomason.io/wp-content/uploads/2014/04/digital-ocean-add-ssh-key.png
  [13]: http://salt-cloud.readthedocs.org/en/latest/topics/config.html
  [14]: http://docs.saltstack.com/en/latest/topics/tutorials/starting_states.html
  [15]: http://thomason.io/wp-content/uploads/2014/04/it-works.png
  [16]: http://thomason.io/automating-application-deployments-across-clouds-with-salt-and-docker/
  [17]: http://www.dockboard.org/