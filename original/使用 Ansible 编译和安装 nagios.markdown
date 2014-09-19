# 使用 Ansible 编译和安装 nagios

标签（空格分隔）： Ansible nagios 自动化管理 自动化配置

---

> 注：该文作者是 [Patrick Ogenstad][1]，原文地址是 [Compile and Install Nagios with Ansible][2]

![此处输入图片的描述][3] 如果你决定尝试 [Nagios][4] 并且通过你的发行版软件管理系统来安装它，你或许注意到它的版本太老了。你想要的一些特性或是扩展在早期版本不支持。因此你决定下载 Nagios 源码包并用传统的方式安装。尽管这不像键入 `yum install nagios` 或 `apt-get install nagios3` 那样简单，但你真的感觉你已经做了什么。这个方法的一个问题就是，如果你以后要重复安装，你或许已经忘记了你使用的一些配置选项或是丢失了必须的插件清单。在完美的世界中，你可能一直把你的步骤记录到文档中。不幸的是，很多人不是生活在完美的世界中。然而，好的一面是这里有很多工具比如 [Ansible][5]，将帮助你自动化一些任务。这个方案的另外一个好处就是 Ansible Playbooks 可以作为系统文档服务。

我已经创建了一个幂等的 Ansible Playbook，从源码安装并且在 Ubuntu 14.04 LTS， Debian 7.5 和 CentOS 6.5 平台测试了，这个 playbook 与 [Nelmon][6] 捆绑在一起。


## playbook 做了什么

一旦 playbook 执行，Ansible 将：

  - 安装 Apache 和 PHP
  - 安装 Nagios 和 Plugins 的先决条件
  - 安装 Centos 的 EPEL Repo （python-passlib 需要）
  - 创建一个 nagios 用户和组
  - 下载 Nagios 和 Plugins 的 tar 包
  - 解压，配置和编译安装这些包
  - 为 nagiosadmin 用户改变 htpasswd
  - 如果 SELinux 是在 enforcing 模式，改变访问  /usr/local/nagios 的权限

这些给你一个基础的 nagios 设置。它不是一个完全的解决方案。目的是向你展示你可以使用 Ansible 来做哪些标准化的设置。你当然也可以把配置添加进这个 Playbook 以至于你可以完全重新安装和配置你的 Nagios 设置。

## 首先要做什么

你要做的第一件事情就是设置你的 Ansible，这个在这篇指南里面没有包括进来，然后下载[这些文件][7]到你的服务器上，然后进入 `ansible/nagios-src` 目录，在  group_vars 目录有叫做 ‘all’ 的文件包含了这些变量。在这里你应该改变的变量是 nagiosadminpass 变量，它控制着分配给 nagiosadmin 用户的密码。然后如果 Nagios 发行了一个新版本，我也不需要改变这个 Playbook，你可以改变其他的变量。


## 使用 Ansible 安装 Nagios 

`site.yml` 文件（在 nagios-src 根目录）是主要的 playbook 文件，依赖于你的设置，如果你想改变这个文件中的一些参数。默认它会在你的所有主机上运行，你可以通过把 `hosts` 变量设置到一个 Ansible 组里面来改变这个。

![此处输入图片的描述][8]

另外一个选项仅仅是在命令行上定义你的服务器。这个将在 srv-nagios-1 和 srv-nagios-2 运行这个 Playbook。

```
ansible-playbook site.yml -l srv-nagios-1,srv-nagios-2
```

以上命令将使用  ssh keys 登陆你的服务器并且尝试使用你当前登陆的用户。在我的环境变量中我创建了一个 `deploy` 用户，然后像这样代替运行 playbook：

```
ansible-playbook site.yml -l srv-nagios-1,srv-nagios-2 -u deploy -s
```

`-u` 选项是指用户，`-s` 选项是指使用 sudo。如果你已经改变了 site.yml  文件中的 `hosts` 变量，你也可以不使用 ` -l (limit)` 选项运行 playbook。

```
ansible-playbook site.yml
```

![此处输入图片的描述][9]

## Nagios 启动和运行

现在你应该使用 Ansible 安装完成了 Nagios。通过 http://[ip-address]/nagios 并且使用 nagiosadmin 用户登陆你的站点。下一步是配置 Nagios。




  [1]: http://networklore.com/author/patrick-ogenstad/
  [2]: http://networklore.com/ansible-install-nagios/
  [3]: http://networklore.com/wp-content/uploads/2014/05/ansible-nagios-300x168.png
  [4]: http://networklore.com/nagios/
  [5]: http://networklore.com/ansible/
  [6]: http://networklore.com/nelmon/
  [7]: http://networklore.com/go/download-nelmon/
  [8]: http://networklore.com/wp-content/uploads/2014/05/nagios-src-ansible-playbook.png
  [9]: http://networklore.com/wp-content/uploads/2014/05/run-ansible-install-nagios.png