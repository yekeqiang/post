# 一种 Dockerize 应用的简单方式

标签（空格分隔）： Docker dockerize

---


> 作者是 [jasonwilder][1]。原文地址是 [A Simple Way to Dockerize Applications][2]

Dockerizing 一个应用是转化一个应用运行在 Docker 容器中的过程。虽然 dockering 大部分应用是简单的，但是这里每次都有一些问题围绕着工作。每次工作的时候有几个问题都需要待解决。

在 dockerization 时两个常见的问题是：

 1. 当它依赖于配置文件时，使得应用使用环境变量
 2. 发送应用日志到 STDOUT/STDERR，当它默认记录在 Docker 的文件系统

这篇文章介绍一个新工具：`dockerize` ，它简化了这两个常见的问题。

## 问题

### 配置

许多应用使用配置文件来控制它们怎么工作，不同的运行环境有不同的值。比如，对于一个开发环境的数据库连接细节将与生产环境的不同。类似的，API keys 和其他的敏感细节在不同环境将不同。


使用 docker 容器有几个方法可以处理这些环境变量的问题：

 1. 在镜像中嵌入所有的环境变量细节和使用一个控制环境变量变量来指出在运行时使用哪个文件。（比如：APP_CONFIG=/etc/dev.config）
 2. 在运行时，使用卷来挂载绑定配置文件的数据
 3. 使用封装脚本，使用工具像 `sed` 那些环境变量来修改配置数据

嵌入所有的环境变量细节是不理想的，因为环境变量的改变应该不需要重新构建一个镜像。它也缺少安全，因为敏感数据 API keys， login 证书等等,作为环境变量被存储在镜像中。私发一个开发环境可能会泄露生产环境细节。有些类型的细节在任何镜像中都应该避免的。
使用 volumes 保持这些细节在镜像外面，但会使得部署更复杂，因为你不仅部署镜像。你必须使配置文件的变更和镜像协调。

注入环境变量到普通文件中也不是重要的。你可能有时会制作一个 `sed` 命令或写一些普通的脚本给它，但这是重复性的工作。这确实产生了一个镜像，但在 Docker 生态系统中工作的很好。
 
 
### Logging
 
Docker 容器日志记录到 STDOUT 和 STDERR 更容易故障排解，监控和融入一个[中央日志系统][3]。日志可以通过 docker logs 命令和 Docker 日志 API 调用来直接访问。这也有许多工具可以自动拉取 docker 日志和运送它们如果日志记录进 STDOUT 和 STDERR。

不幸地是，默认，许多应用日志记录一个或多个文件到文件系统上。虽然这通常可以[围绕工作][4]，计算出每个应用的日志配置的细微差别是乏味的。


## 使用 Dockerize

[dockerize][5] 是一个小型的 Golang 应用，可以通过以下简化 dockerization 过程：

 1. 在启动时使用模板生成配置文件和容器环境变量
 2. tail 任意的日志文件到 STDOUT 和 STDERR
 3. 启动一个进程，运行在容器里面

###一个示例

为了证明它怎样工作，我们将详细讲述使用 dockerize 来 dockerizing 一个一般的 nginx 的过程。

```
FROM ubuntu:14.04

# Install Nginx.
RUN echo "deb http://ppa.launchpad.net/nginx/stable/ubuntu trusty main" > /etc/apt/sources.list.d/nginx-stable-trusty.list
RUN echo "deb-src http://ppa.launchpad.net/nginx/stable/ubuntu trusty main" >> /etc/apt/sources.list.d/nginx-stable-trusty.list
RUN apt-key adv --keyserver keyserver.ubuntu.com --recv-keys C300EE8C
RUN apt-get update
RUN apt-get install -y nginx

RUN echo "daemon off;" >> /etc/nginx/nginx.conf

EXPOSE 80

CMD nginx

```

下一步，我们将安装 `dockerize` 和通过它运行 `nginx`：

```
FROM ubuntu:14.04

# Install Nginx.
RUN echo "deb http://ppa.launchpad.net/nginx/stable/ubuntu trusty main" > /etc/apt/sources.list.d/nginx-stable-trusty.list
RUN echo "deb-src http://ppa.launchpad.net/nginx/stable/ubuntu trusty main" >> /etc/apt/sources.list.d/nginx-stable-trusty.list
RUN apt-key adv --keyserver keyserver.ubuntu.com --recv-keys C300EE8C
RUN apt-get update
RUN apt-get install -y wget nginx

RUN echo "daemon off;" >> /etc/nginx/nginx.conf

RUN wget https://github.com/jwilder/dockerize/releases/download/v0.0.1/dockerize-linux-amd64-v0.0.1.tar.gz
RUN tar -C /usr/local/bin -xvzf dockerize-linux-amd64-v0.0.1.tar.gz

ADD dockerize /usr/local/bin/dockerize

EXPOSE 80

CMD dockerize nginx
```


默认 Nginx 在 `/var/log/nginx ` 目录下记录两个不同的文件。如果你交互式的运行这个容器，这将有 nginx 的 access and error 日志流到控制台，或者是你运行 `docker logs nginx`，因此你可以看到发生了什么。

我们可以通过传递 `-stdout <file>` 和 `-stderr <file>` 命令行选项来解决它。如果你有几个文件需要 tail ，这里可以传递多次。

```
CMD dockerize -stdout /var/log/nginx/access.log -stderr /var/log/nginx/error.log nginx
```

现在当你运行容器，nginx 日志通过 docker logs nginx 是可用的。

为了证明模板，我们可以使用环境变量配置让这个变成一个更通用的代理服务器。我们将定义环境变量 `PROXY_URL` 作为一个站点的代理 URL。

```
PROXY_URL="http://jasonwilder.com"
```

当这个容器使用这个环境变量启动，`dockerize` 将使用它来生成一个 nginx 的location 路径。

这是模板：

```
server {
    listen 80 default_server;
    listen [::]:80 default_server ipv6only=on;

    root /usr/share/nginx/html;
    index index.html index.htm;

    # Make site accessible from http://localhost/
    server_name localhost;

    location / {
      access_log off;
      proxy_pass {{ .Env.PROXY_URL }};
      proxy_set_header X-Real-IP $remote_addr;
      proxy_set_header Host $host;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

这时我们最后的 Dockerfile 将看起来这样：

```
FROM ubuntu:14.04

# Install Nginx.
RUN echo "deb http://ppa.launchpad.net/nginx/stable/ubuntu trusty main" > /etc/apt/sources.list.d/nginx-stable-trusty.list
RUN echo "deb-src http://ppa.launchpad.net/nginx/stable/ubuntu trusty main" >> /etc/apt/sources.list.d/nginx-stable-trusty.list
RUN apt-key adv --keyserver keyserver.ubuntu.com --recv-keys C300EE8C
RUN apt-get update
RUN apt-get install -y wget nginx

RUN echo "daemon off;" >> /etc/nginx/nginx.conf

RUN wget https://github.com/jwilder/dockerize/releases/download/v0.0.1/dockerize-linux-amd64-v0.0.1.tar.gz
RUN tar -C /usr/local/bin -xvzf dockerize-linux-amd64-v0.0.1.tar.gz

ADD default.tmpl /etc/nginx/sites-available/default.tmpl

EXPOSE 80

CMD dockerize -template /etc/nginx/sites-available/default.tmpl:/etc/nginx/sites-available/default -stdout /var/log/nginx/access.log -stderr /var/log/nginx/error.log nginx

```


`-template <src>:<dest>` 选项指明 template 在 `/etc/nginx/sites-available/default.tmpl ` 应该被生成并写入 `/etc/nginx/sites-available/default`。多个模板也可以被指定。

使用下面命令运行容器：

```
$ docker run -p 80:80 -e PROXY_URL="http://jasonwilder.com" --name nginx -d nginx
```
然后你可以通过 http://localhost 访问，它将代理到这个站点。

这是一个简化的例子，但是使用嵌入的 `split` 函数和 `range` 声明使它可以很容易的被扩展来处理多个代理值和其他选项。这里有一些其他的可用[模板函数][6]示例。


## 总结

虽然这个例子有点简单，许多应用需要一些 shims 来使得在 Docker 中运行的更好。`dockerize` 是一个通用的工具来帮助你处理这个过程。

你可以在  [jwilder/dockerize][7] 找到代码。
 


  [1]: https://github.com/jwilder
  [2]: http://jasonwilder.com/blog/2014/10/13/a-simple-way-to-dockerize-applications/
  [3]: http://jasonwilder.com/blog/2012/01/03/centralized-logging/
  [4]: http://jasonwilder.com/blog/2014/03/17/docker-log-management-using-fluentd/
  [5]: http://github.com/jwilder/dockerize
  [6]: https://github.com/jwilder/dockerize#using-templates
  [7]: http://github.com/jwilder/dockerize