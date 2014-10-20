# 自动化的 Nginx 反向代理 Docker

标签（空格分隔）： Docker Nginx Automated

---

> 本文作者是 [jwilder][1]，原文地址是 [Automated Nginx Reverse Proxy for Docker][2]


## 为什么 Docker 要使用反向代理

Docker 容器被分配随机 IP 和端口，这使得从客户端角度来寻址它们是非常复杂的。默认，IP 和端口是专用于主机的，并且不能被外部访问除非它们被绑定到主机上。

绑定容器到主机端口可以防止多个容器在同一个主机上运行。比如，在同一时间仅仅只有一个容器可以被绑定到 80 端口。这也使得新版本的容器无停机的推出复杂化了，因为老版本必须在新版本启动之前先停止。


一个反向代理可以帮助处理这些问题，同时通过减轻零停机部署的困难来提升可用性。

## 生成反向代理配置文件

当一个容器被启动和停止的时候，设置一个反向代理配置可能是复杂的。通常的配置需要手动更新，这容易出错并且费时。

幸运的是，Docker 提供了一个远程 API 来 [inspect containers][3] 和访问他们的 IP，端口和其他配置元数据。另外，它也提供一个[实时事件 API][4]可以被用于通知什么时候容器被启动和停止。这些 API 可以被用于自动地生成一个反向代理配置。

[docker-gen][5] 是一个使用这些 API 的小工具，并导出容器的元数据到模板中。模板被渲染以及一个可选的通知命令可以被运行来重起服务。

使用 [docker-gen][6]，我们可以自动的生成 Nginx 的配置并重载 Nginx当它们改变的时候。同样的方法可被用于 [Docker 日志管理][7]。


## Nginx 反向代理 Docker

这个例子 Nginx 模板可以被用于生成一个 Docker 容器使用虚拟主机路由的反向代理配置文件。这个模板是使用 [golang text/template package][8] 实现的。它使用一个通用的 `groupBy` 模板函数通过它们的 `VIRTUAL_HOST` 环境变量来分组运行的容器。这简化了遍历容器来生成一个负载均衡后端以及使得零停机部署可行。


```
{{ range $host, $containers := groupBy $ "Env.VIRTUAL_HOST" }}
upstream {{ $host }} {

{{ range $index, $value := $containers }}
    {{ with $address := index $value.Addresses 0 }}
    server {{ $address.IP }}:{{ $address.Port }};
    {{ end }}
{{ end }}

}

server {
    #ssl_certificate /etc/nginx/certs/demo.pem;
    #ssl_certificate_key /etc/nginx/certs/demo.key;

    gzip_types text/plain text/css application/json application/x-javascript text/xml application/xml application/xml+rss text/javascript;

    server_name {{ $host }};

    location / {
        proxy_pass http://{{ $host }};
        include /etc/nginx/proxy_params;
    }
}
{{ end }}
```

这个模板可以使用 `docker-gen` 来运行：

```
docker-gen -only-exposed -watch -notify "/etc/init.d/nginx reload" templates/nginx.tmpl /etc/nginx/sites-enabled/default
```

- `-only-exposed` - 仅仅使用容器已经暴露的端口。
- `-watch` - 启动之后，监控 docker 容器事件和重新生成模板。
- `-notify "/etc/init.d/nginx reload"` - 在模板生成后重载 nginx 配置。
- `templates/nginx.tmpl` - nginx 模板。
- `/etc/nginx/sites-enabled/default` - 目标文件。


这是两个容器配置了 `VIRTUAL_HOST=demo1.localhost` 和 ` VIRTUAL_HOST=demo2.localhost` 的渲染模板。

```
upstream demo1.localhost {
    server 172.17.0.4:5000;
    server 172.17.0.3:5000;
}

server {
    #ssl_certificate /etc/nginx/certs/demo.pem;
    #ssl_certificate_key /etc/nginx/certs/demo.key;

    gzip_types text/plain text/css application/json application/x-javascript text/xml application/xml application/xml+rss text/javascript;

    server_name demo1.localhost;

    location / {
        proxy_pass http://demo.localhost;
        include /etc/nginx/proxy_params;
    }
}

upstream demo2.localhost {
    server 172.17.0.5:5000;
}

server {
    #ssl_certificate /etc/nginx/certs/demo.pem;
    #ssl_certificate_key /etc/nginx/certs/demo.key;

    gzip_types text/plain text/css application/json application/x-javascript text/xml application/xml application/xml+rss text/javascript;

    server_name demo2.localhost;

    location / {
        proxy_pass http://demo2.localhost;
        include /etc/nginx/proxy_params;
    }
}
```

## 试验

我使用这个设置创建了一个[可信任的构建][9]来使得它非常容易试验。

运行 nginx-proxy 容器：

```
$ docker run -d -p 80:80 -v /var/run/docker.sock:/tmp/docker.sock -t jwilder/nginx-proxy
```

使用一个 `VIRTUAL_HOST` 环境变量启动你的容器：

```
docker run -e VIRTUAL_HOST=foo.bar.com -t ...
```

## 总结

为 docker 容器生成 nginx 反向代理配置可以通过使用 Docker APIs 和一些基础的模板自动化，这可以简化部署同时提升可用性。

虽然这非常适用于单台主机上运行的容器，为远程主机生成配置需要[自动发现][10]。看一下 [Docker 自动发现][11] 对这个问题的解决。


更新：这有一些类似观点和变种的其他文章值得一看：

- [Using Nginx, Confd, and Docker for Zero-Downtime Web Update][12] - Brian Ketelsen
- [Docker Events][13] - Michael Crosby
- [Haproxy As A Static Reverse Proxy for Docker Containers][14] - Oskar Hane


  [1]: https://github.com/jwilder
  [2]: http://jasonwilder.com/blog/2014/03/25/automated-nginx-reverse-proxy-for-docker/
  [3]: http://docs.docker.io/en/latest/reference/api/docker_remote_api_v1.10/#inspect-a-container
  [4]: http://docs.docker.io/en/latest/reference/api/docker_remote_api_v1.10/#monitor-docker-s-events
  [5]: https://github.com/jwilder/docker-gen
  [6]: https://github.com/jwilder/docker-gen
  [7]: http://jasonwilder.com/blog/2014/03/17/docker-log-management-using-fluentd/
  [8]: http://golang.org/pkg/text/template/
  [9]: https://index.docker.io/u/jwilder/nginx-proxy/
  [10]: http://jasonwilder.com/blog/2014/02/04/service-discovery-in-the-cloud/
  [11]: http://jasonwilder.com/blog/2014/07/15/docker-service-discovery
  [12]: http://brianketelsen.com/2014/02/25/using-nginx-confd-and-docker-for-zero-downtime-web-updates/
  [13]: http://crosbymichael.com/docker-events.html
  [14]: http://oskarhane.com/haproxy-as-a-static-reverse-proxy-for-docker-containers/