# RANCID 入门

标签（空格分隔）： RANCID router 网络设备自动化配置 配置管理

---
![此处输入图片的描述][1]RANCID 是一个配置文件对比工具，它本身像它听起来一样无聊；你一到达办公司，就打开一个维护窗口，然后输入用户名，并且抱怨已经停止工作的系统。晚上工作的技术人员正在睡觉，并且他们似乎忘记了把他们所做的事情记录进文档（坑队友）。幸好，RANCID 能帮助我们，RANCID 除了可以向你展现昨晚的变更，还可以展现自从它正式使用后的所有变更。因此如果你已经使用 RANCID 三年以上，它可以向你展现从那时候起到现在的所有硬件的所有变更。把你所有的配置文件存储进 RANCID 服务器作为一个备份。它不但可以帮助你很好的收集硬件的信息，你还可以通过发送一个命令到几个节点来使用 RANCID 从你的硬件上获取指定的信息，比如“show ip route”” 或者 “show crypto pki certificates”。更进一步，你可以使用它变更你的配置信息，因此如果你需要改变所有防火墙或是路由器上的访问列表，你可以使用 RANCID 来做到这些事情。

虽然 RANCID 不是很完美，但是真的是非常容易使用。如果你没有其他类似的工具，RANCID 将是一款非常适合你的工具。

## 安装

这里有针对不同发行版的安装包。
你可以在  CentOS 6.4 系统上运行如下命令安装 RANCID：

```
sudo rpm -Uvh --replacepkgs http://dl.fedoraproject.org/pub/epel/6/x86_64/epel-release-6-8.noarch.rpm
sudo yum install rancid cvs
```

在 Ubuntu 12.04 LTS 或 Debian 7.4 上你可以使用 apt 命令来安装：

```
sudo apt-get install rancid
```

你会注意到，如果你安装 Debian 或 Ubuntu 包，你将被展现一个 文本信息，告诉你这是个早期版本包（开发阶段），

![此处输入图片的描述][2]

通常，当你使用一个发行版的包的时候，你不会访问最新的版本。如果你需要或者想要安装最新版本，你可以通过源码安装的方式代替。如果你选择 route，你依然可以使用本教程来帮助你设置 RANCID。

## Windows 用户怎么办？

RANCID 不能原生的运行在 Windows 上。但是你或许可以通过 Cygwin 来运行它。我依然建议你安装在一台 Linux 服务器上。尽管如此，如果你是一个 Windows 忠实用户（个人爱好，或是公司环境），我建议你使用 [CatTools][3] 代替。CatTools 是一个和 RANCID 在许多地方都类似的工具，你大部分想做的事情在两个产品上都有。
## 我们的测试网络

为了测试的目的，假设你已经在一个拥有一个总部和四个分支机构的网络中安装了 RANCID。在总部，有一个路由器，一个 汇聚交换机，三个接入交换机和6个独立的无线接入节点。在每个分支机构，一个路由器，一个接入交换机和两个无线接入节点。你想使用 RANCID 处理所有的硬件。

![此处输入图片的描述][4]


因此在我们的网络中，我们有以下硬件设备：

Stockholm：sth-rtr-1， sth-dsw-1，sth-asw-1， sth-asw-2， sth-asw-3， sth-ap-1， sth-ap-2， sth-ap-3， sth-ap-4， sth-ap-5 和 sth-ap-6
Boston：bos-rtr-1，bos-asw-1， bos-ap-1 和 bos-ap-2
Amsterdam： ams-rtr-1， ams-asw-1， ams-ap-1 和 ams-ap-2
Paris：par-rtr-1， par-asw-1， par-ap-1 和 par-ap-2
London：lon-rtr-1， lon-asw-1， lon-ap-1 和 lon-ap-2

## 建立分组

安装完成 RANCID 的第一件事就是你需要配置你将使用到的分组。如果你以前没有使用过 RANCID，怎样设置你的分组可能会比较困难，或是为什么要这样做。最早的方式是仅仅创建一个叫做 “all” 的分组，然后把所有的硬件设备存放在该分组里面。


因此，什么是分组？在这个上下文中，分组就是一组硬件设备的集合，当一个硬件发生了改变，组管理员将收到一封通知邮件。在这个测试网络中，Boston 的技术人员可能并不关心 London 的交换机做的变更。在这样的情况下，你需要为每个分支机构创建独立的分组。另外一个情况就是你可能有一个无线管理员处理所有的无线节点（在这个情况下，还没有说服管理层投资一个无线局域网控制器）。在这里你可能想为每种类型的硬件创建一个分组，比如，一组用于路由器，一组用于交换机，一组用于无线访问点。

另外一个使用分组的原因是除了分离变更通知以外，RANCID 还需要在单独的目录为每一个分组存储该组的硬件配置。因此如果你想使用 Unix 命令或是类似的命令来在你的配置文件中搜索，考虑如何设置你的分组会非常有用。如果你不确定，你可以在最后改变。

分组设置在 ` /etc/rancid/rancid.conf` 文件中，我们根据硬件类型设置分组的示例如下：

```
# list of rancid groups
LIST_OF_GROUPS="routers switches access-points"
```

## 设置邮件别名

RANCID 发送邮件给 `rancid-[groupname]` 和 `rancid-admin-[groupname]` 的别名。因此编辑你的 `/etc/aliases` 文件。

```
rancid-routers:             router-admins@yourcompany.com
rancid-admin-routers:       router-admins@yourcompany.com
rancid-switches:            switches-admins@yourcompany.com
rancid-admin-switches:      switches-admins@yourcompany.com
rancid-access-points:       ap-admins@yourcompany.com
rancid-admin-access-points: ap-admins@yourcompany.com
```

## 设置 CVS （吐槽下，这个确实太古老了）

RANCID 默认使用 CVS 做版本管理追踪你硬件的变更。你也可以使用 Subversion。为了准备 CVS 数据库，你需要以 rancid 用户（在安装的时候创建的）运行 rancid-cvs 命令。这个需要在你每次创建一个新组的时候做一次。

```
sudo -i -u rancid
/var/lib/rancid/bin/rancid-cvs (on Debian and Ubuntu)
/usr/bin/rancid-cvs (on CentOS)
```

这会在 /var/lib/rancid (Debian and Ubuntu) 和 /var/rancid (CentOS) 下面为每个组创建一个目录。因此路由器目录位于 `/var/lib/rancid/routers`（Debian 或 Ubuntu），如果是使用的 CentOS 那目录就位于 `/var/rancid/routers`。


## 添加 hosts 到 router.db

router.db 是为每个组指定的，位于分组目录，像下面这样把硬件添加进组。

Routers (/var/lib/rancid/routers/router.db or /var/rancid/routers/router.db)：

```
sth-rtr-1:cisco:up
bos-rtr-1:cisco:up
ams-rtr-1:cisco:up
par-rtr-1:cisco:up
lon-rtr-1:cisco:up
```

Switches /var/lib/rancid/switches/router.db or /var/rancid/switches/router.db：

```
sth-dsw-1:cisco:up
sth-asw-1:cisco:up
sth-asw-2:cisco:up
sth-asw-3:cisco:up
bos-asw-1:cisco:up
ams-asw-1:cisco:up
par-asw-1:cisco:up
lon-asw-1:cisco:up
```

Access Points：

```
sth-ap-1:cisco:up
sth-ap-2:cisco:up
sth-ap-3:cisco:up
sth-ap-4:cisco:up
sth-ap-5:cisco:up
sth-ap-6:cisco:up
bos-ap-1:cisco:up
bos-ap-2:cisco:up
ams-ap-1:cisco:up
ams-ap-2:cisco:up
par-ap-1:cisco:up
par-ap-2:cisco:up
lon-ap-1:cisco:up
lon-ap-2:cisco:up
```

## 配置访问硬件

访问硬件的资格证书是存储在 ` .cloginrc` 文件中，在 rancid 用户 home 目录 /var/lib/rancid (Debian and Ubuntu) 和 /var/rancid (CentOS) 创建该文件。编辑该文件并添加以下行：

```
add autoenable * 1
add method * ssh
add user * rancid
add password * s3ctetp@ssw0rd
```

以上配置就是告诉 RANCID 使用 SSH 登陆所有的硬件，所有硬件的登陆用户名是 rancid，密码是 s3ctetp@ssw0rd。autoenable 行告诉 RANCID， 硬件是被配置为登陆后，指定的 rancid 用户直接在启用模式。
配置 .cloginrc 文件以便 RANCID 发出可用的用户名和密码。或者是根据不同的硬件使用不同的用户名和密码：

```
add user *-asw-* rancid-switch-user
add password *-asw-* $witchpassw0rd
```

因为 RANCID 将直接登陆硬件设备，通过使用 Tacacs 命令授权控制 rancid 用户能做什么是一个非常好的主意。因此 RANCID 仅仅被允许运行特定的显示命令。如果你没有一个 Tacacs 服务器，你也可以用 RADIUS 使用 cli-views。

为了能锁定它，你必须改变文件的访问权限：

```
chmod 600 .cloginrc
```

## 测试 Rancid

一旦授权完成，你可以测试它怎样工作的，使用 rancid 用户（sudo -i -u rancid），你可以尝试登陆其中一台硬件，通过使用位于 /var/lib/rancid/bin (Debian and Ubuntu) 和 /usr/libexec/rancid (CentOS) 下的 `clogin`  命令登陆进思科的硬件设备。你或许想把它们添加进你的 path。当在那个目录的时候，你将注意到其他的 *login 文件。例如 `hlogin` 用于惠普设备，flogin 用于 Foundry 设备。

```
clogin sth-asw-2
```

![此处输入图片的描述][5]

你现在应该登陆进了交换机，然后键入 exit 返回服务器。

## 调度 RANCID

为了让 RANCID 保持追踪变更，你需要以 rancid  用户启动一个定时的调度任务：

```
crontab -e

0 * * * * * /usr/bin/rancid-run
```

这将每小时运行一次调度任务，并且在每次硬件设备有变更的时候通知每组的责任成员。你也可以设置成你喜欢的运行频率，在一些环境中，每晚一次足够了。另外一个选择就是每组有一个独立的调度任务。如果防火墙的规则经常被添加或者删除，但是你的交换机很少变更，你或许想为每个组设置一个不同的调度任务。


## 查看 RANCID 变更历史

一旦你执行了 rancid-run 或者 被定时任务执行，你的硬件配置将被存储进每个组的配置文件目录。每次 RANCID 发现一个变更，它都将创建一个版本，并且存储自从上次运行后到现在的所有变更。

```
[rancid@tsrv-rancid-cen configs]$ cvs log sth-asw-2

RCS file: /var/rancid/CVS/switches/configs/sth-asw-2,v
Working file: sth-asw-2
head: 1.3
branch:
locks: strict
access list:
symbolic names:
keyword substitution: o
total revisions: 3;     selected revisions: 3
description:
----------------------------
revision 1.3
date: 2014/04/04 20:01:11;  author: rancid;  state: Exp;  lines: +5 -4
updates
----------------------------
revision 1.2
date: 2014/04/04 19:01:06;  author: rancid;  state: Exp;  lines: +446 -0
updates
----------------------------
revision 1.1
date: 2014/04/04 19:00:09;  author: rancid;  state: Exp;
new router
=============================================================================
[rancid@tsrv-rancid-cen configs]$
```

检查你能看到的输出，目前为止，sth-asw-2 有 3 个版本。当文件被创建的时候，版本是 1.1。当 rancid-run 第一次收集配置文件信息的时候，增加了 446 行，版本是 1.2。最新的版本 1.3 增加了 5 行，删除了 4 行。

你可以使用 `cvs dif` 命令来看两个版本之间的变化。

![此处输入图片的描述][6]

以上图片给我们展示的信息比我们想要的还多。首先我们在文件系统看到了交换机信息的变更，随着配置文件大小的增长，或者是减少，当我们变更配置的时候。在底部我们可以看到实际变更了什么。在这个示例中，自从上次 rancid-run 运行，薪增了行 “snmp-server location Stockholm” 。如果你对文件系统中的某些变更不感兴趣，你可以排除 RANCID 的部分输出。

如果你已经使用 RANCID 很长一段时间，你可以走到你同事的身边，然后在他耳边低语：“我们知道你去年夏季做了什么。”（如果你想成为那样的人的话）

## 收集特定信息

尽管有能力看对你的网络做的所有变更是一件非常棒的事情，但你也可以使用 clogin 命令来获取更多的特定信息。使用 `clogin -c` 你可以告诉 RANCID 登陆一个指定的硬件然后运行一个命令，如：

```
clogin -c "show processes cpu" bos-rtr-1
```

将登陆 Boston 的路由器，然后执行显示 CPU processes，然后退出路由器。这个可能本身不会帮你节省多少时间，但是或许你想在一定的时间内运行一个命令几次。这时，你可以像这样调度一个命令来收集信息。或者是你可以在你所有的路由器上运行这个命令。

```
rancid@tsrv-rancid-ubu:~/routers/configs$ pwd
/var/lib/rancid/routers/configs
rancid@tsrv-rancid-ubu:~/routers/configs$ clogin -c "show processes cpu" `ls *rtr*`
```

这个将列出在配置文件目录下的所有名为 `*rtr*` 的文件，然后在所有匹配的文件上运行命令 `show processes cpu`，比如你的路由器。你也可以使用分号分离命令来在每台设备商运行几个命令。

```
clogin -c "show processes cpu;show memory free" ams-rtr-1
```

## 使用 RANCID 变更配置

你可以使用的另外一个参数是 `-x`。它像 ` -c` 一样工作，但是不同的是，你要键入你想执行的命令，然后把它们存储进一个独立的文件。

```
clogin -x commands.txt lon-ap-2
```

以上命令将登陆在 London 办公室的无线接入点，然后执行你输入 `commands.txt` 文件的所有命令。该文件包含了你键入的任何命令，如果你自己使用 ssh 登陆硬件设备。

假设你正在推出你单位的 dot1x，并想拉出一个认证 ACL 到你的所有硬件，你可以创建一个名为 access-list.txt 的文件，包含以下命令：

```
conf term
ip access-list extended PREAUTH
 permit udp any any eq bootps
 permit udp any any eq domain
 permit icmp any any
end
write mem
```

然后创建一个名为 switches.txt 的文件，包含了你想做变更的交换机的名字。


```
clogin -x access-list.txt `cat switches.txt`
```

如果你有上白台交换机，或许仅仅 10 台 RANCID 就能帮你节约很多时间。一个额外的好处是相同的配置被添加进每一台设备。如果你使用 Tacacs 锁定了 rancid  的账号，你可能不能做变更配置。即使你没有用这种方式限制 rancid 账号变更配置，因为 rancid 做了变更，让它们匿名。你可以在你的 `home` 目录设置一个 `.cloginrc` 文件，然后以你自己的用户运行 `clogin` 命令。注意，你的密码是以明文记录在 `.cloginrc` 文件中的，一个解决方案是 encrypt 你的 home 目录，或者是在你做完变更后删除你的密码。

## 使用脚本做更多的事情

使用 RANCID 在你的所有硬件上做相同的变更是非常直接的。如果你部署了不同的配置到你的硬件。比如 access-lists 对于每个路由器都不同。如果你学习更多的基础脚本，以及使用 clogin 组合它们的技术，你可以为你的硬件收集到更多的特定信息，或是在几分钟内实现每个路由器的特定变更，而不是手动的几个小时。



## 定制 RANCID

在这个教程中，我向你展示了 RANCID 的基础知识。但是有非常多的方式可以让你定制 RANCID。一旦你开始了，你可能想深入配置或是 rancid 的程序，为了增加 rancid-run 使用的命令或是删除其他。如果你以前没有使用过 RANCID，在你感到需要做一些改变的时候，那将花费你一些时间。我希望带给你的是，改变默认值可能是非常容易的。

## 总结

现在你有一个收集你硬件设备信息的系统，它将作为一个备份的角色，并且你也能看到变更。你也应该对 RANCID 有一个基本的了解，知道它能干什么，以及使用它怎样节省你的时间。


> 注：貌似命令不支持华为设备？需要深入了解下。





  [1]: http://networklore.com/wp-content/uploads/2014/03/rancid-101-300x225.png
  [2]: http://networklore.com/wp-content/uploads/2014/04/rancid-debian-package.png
  [3]: http://www.solarwinds.com/kiwi-cattools.aspx
  [4]: http://networklore.com/wp-content/uploads/2014/04/rancid-network-overview-1024x724.png
  [5]: http://networklore.com/wp-content/uploads/2014/04/clogin-example.png
  [6]: http://networklore.com/wp-content/uploads/2014/04/rancid-cvs-dif.png