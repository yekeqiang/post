# 有用的 docker bash 函数 和 别名



我每天会键入许多的docker命令。我有一个习惯，就是当我发现有用的命令的时候，会创建一个bash函数，以别名的形式作为结果保存，同时添加进我的 .bash_profile[^1]。


## 获取IP地址

----------

第一个有用的别名， ```dip ```  ，是一个的 ```docker inspect```  快捷命令，该命令允许你检查一个容器。该命令的一个特性是使用 ``` --format ``` 参数来选择检查数据中的子集。这样我会返回该容器的 IP 。

这就是我的别名：

```
alias dip="docker inspect --format '{{ .NetworkSettings.IPAddress }}'"
```

然后你可以键入 ``` dig ``` 命令、容器ID或者是容器名来得到如上命令的结果：

```
$ dip 72bbff4d768c
172.17.0.3
```

这将返回指定容器的 IP 地址。

##移除容器
----------

一旦你创建了一个 docker 容器，你可以使用 ``` docker  rm ``` 命令很容易的移除它。但是有时候你想一次性移除一批容器。下面这个函数提供了批量移除所有不运行容器的快捷方式。该命令依靠 ``` docker ps -q -a ``` 的输出来工作。其返回一列容器 ID 做给结果传给 ```docker rm``` 命令处理。

我写的移除功能函数如下：

```
drm() { docker rm $(docker ps -q -a); }
```
运行如下命令处理：

```
$ drm
b644290130ec
092f4dc1abca
6451afd047d4
Error: Impossible to remove a running container, please stop it first or use -f
```

你能看到3个容器被移除了，但是还有一个容器被跳过了，没有被移除。
> 注：要移除被跳过的容器，必须用 ```-f ``` 参数先把其停止。

##移除镜像
---------

移除镜像的函数和移除容器的函数非常相似。它是通过把 ```docker images -q``` 命令的输出传递给 ```docker rmi``` 命令来进行处理。

该函数如下：

```
dri() { docker rmi $(docker images -q); }
```

运行结果如下：

```
$ dri
Deleted: 78fc424470fd789d8b5d7a0e3c698137000c0819157efbd0f29bcdee0621c567
Deleted: 4040035d043ca0f18807a80873924ea565e3136ccee3e127c22c335ef32b0a9e
Error: Conflict, cannot delete image 1a6d876a1d70 because it is tagged in multiple repositories, use -f to force
```

通过上面的结果我们看到，其移除了一些镜像，但是跳过了一个正在使用的镜像
> 注：正在使用的镜像是不能被移除的，你要移除的话，必须先停止该镜像。

## 运行不同类型的容器

下一步我有两个别名示例，其分别提供了运行交互式和守护容器的通用设置的快捷方式。

```
alias dkd="docker run -d -P"
```

我喜欢这样用：

```
$ dkd jamtur01/ssh
```

这将启动一个守护容器来运行我的 ``` jamtur01/ssh``` 镜像。我还能在该命令的后面增加一个命令来重写它。

我的第二个别名示例与第一个非常像，但是它是运行一个交互式容器来代替它

```
alias dki="docker run -t -i -P"
```

我喜欢这杨用它：

```
$ dki ubuntu /bin/bash
```

该命令会用 TTY 启动一个交互式容器来运行 ```ubuntu``` 镜像，并且执行 ```/bin/bash``` 命令。

##Docker构建函数

最后我有一个函数是与命令 ```docker build``` 互动。该函数允许我跳过 ```-t``` 这个参数为我的新镜像打标签[^2]。

```
db() { docker build -t="$1" .; }
```  

我喜欢这样用：

```
$ db jamtur01/ssh
```

假设一个 ```Dockerfile``` 在我的当前目录下，然后构建这个文件，并且用 ```jamtur01/ssh```打标签 ，随后构建。

我希望这对人们是有用的，并且这注释让你们感到轻松：

[^1]: 1. 这些别名假设你是使用的 Bash，但是很明显地你很容易用 zsh ， fish 等更新。

[^2]: 2. 欢迎任何阅读我给出的 Docker demo 的人，指出我的排版错误