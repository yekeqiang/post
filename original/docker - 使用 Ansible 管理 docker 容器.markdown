# docker - 使用 Ansible 管理 docker 容器 

标签（空格分隔）： docker Ansible container

---

> 注：作者是 Cove Schneider，Joshua Conner， Pavel Antonov。原文是 Ansible 的官方文档中 [docker - manage docker containers][1]


## 大纲

在 Ansible 的 1.4 版本及以上支持。

管理 docker 容器的生命周期。

## 选项

属性|必需|默认值|选择|说明| 
:---------------|:---------------|:---------|:-------|:-------|
command|no|||在启动的时候设置一个命令来运行容器|
count|no|1||设置运行的容器的数量|
detach|no|true||以 detached 模式启动，让容器在后台运行|
dns|no|||为容器设置自定义的 DNS 服务器|
docker_api_version|no|docker-py default remote API version||使用的 Remote API  版本，默认的是当前 docker-py 默认指定的版本（在 Ansible 1.8 中添加）|
docker_url|no|unix://var/run/docker.sock|| docker 主机发出命令的 URL|
env|no|||设置环境变量，（比如：env="PASSWORD=sEcRe7,WORKERS=4"）|
expose|no|||设置容器 expose 的端口，用于映射和 links。（如果这个端口已经在 Dockerfile 中 EXPOSE 了，你不需要再次 expose 它）。在  Ansible 1.5 中添加|
hostname|no|||设置容器的主机名|
image|yes|||设置使用的镜像|
links|no|||把容器与其他容器连接起来（比如：links=redis,postgresql:db）。在 Ansible 1.5 中添加|
lxc_conf|no|||LXC 配置参数，比如： lxc.aa_profile:unconfined|
memory_limit|no|256 MB||设置给容器分配的内存|
name|no|||设置容器的名字，不能与 count 参数同时使用，在  Ansible 1.5 中添加|
net|no|||为 容器设置网络模式（bridge, none, container:<name|id>, host）. 要求 docker >= 0.11. （在 Ansible 1.8 中被添加））|
password|no|||设置远程API 的密码|
ports|no|||使用 docker CLI-style 语法，设置私有到公共端口的映射技术参数， [([<host_interface>:[host_port]])|(<host_port>):]<container_port>[/udp] （在 Ansible 1.5 中添加）|
privileged|no|||设置容器是否应该允许在 privileged  模式|
publish_all_ports|no|||发布所有的暴露端口给主机接口，在 Ansible 1.5 中添加|
registry|no|||用于拉取镜像的远程仓库的 URL，（在 Ansible 1.8 中添加 ）|
state|no|present|present running  stopped   absent  killed restarted|设置容器的状态|
stdin_open|no|||保持 stdin  打开。（在 Ansible 1.6 中添加）|
tty|no|||分配一个 pseudo-tty （在 Ansible 1.6 添加）|
username|no|||设置远程 API 的名称|
volumes|no|||设置挂载在容器中的 volume|
volumes_from|no||从另外一个容器中设置共享卷|

> 注意：要求  docker-py >= 0.3.0
> 注意：docker >= 0.10.0

## 示例：


在 web 组的每台主机上启动一个运行 tomcat 的容器，并且把 tomcat 的监听端口绑定到主机的 8080：

```
- hosts: web
  sudo: yes
  tasks:
  - name: run tomcat servers
    docker: image=centos command="service tomcat6 start" ports=8080
```

tomcat 服务器的端口是 NAT 到主机上的一个动态端口，但是你可以决定服务器哪个端口与 Docker 容器做映射：

```
- hosts: web
  sudo: yes
  tasks:
  - name: run tomcat servers
    docker: image=centos command="service tomcat6 start" ports=8080 count=5
  - name: Display IP address and port mappings for containers
    debug: msg={{inventory_hostname}}:{{item['HostConfig']['PortBindings']['8080/tcp'][0]['HostPort']}}
    with_items: docker_containers
```

正如前面的例子，但是是用一个 sequence 迭代 docker 容器的列表：

```
- hosts: web
  sudo: yes
  vars:
    start_containers_count: 5
  tasks:
  - name: run tomcat servers
    docker: image=centos command="service tomcat6 start" ports=8080 count={{start_containers_count}}
  - name: Display IP address and port mappings for containers
    debug: msg="{{inventory_hostname}}:{{docker_containers[{{item}}]['HostConfig']['PortBindings']['8080/tcp'][0]['HostPort']}}"
    with_sequence: start=0 end={{start_containers_count - 1}}
```

停止、移除所有正在运行的 tomcat 容器，并且从停止的容器中列出退出码：

```
- hosts: web
  sudo: yes
  tasks:
  - name: stop tomcat servers
    docker: image=centos command="service tomcat6 start" state=absent
  - name: Display return codes from stopped containers
    debug: msg="Returned {{inventory_hostname}}:{{item}}"
    with_items: docker_containers
```

创建一个命名的容器：

```
- hosts: web
  sudo: yes
  tasks:
  - name: run tomcat server
    docker: image=centos name=tomcat command="service tomcat6 start" ports=8080
```

创建多个命名的容器：


```
- hosts: web
  sudo: yes
  tasks:
  - name: run tomcat servers
    docker: image=centos name={{item}} command="service tomcat6 start" ports=8080
    with_items:
      - crookshank
      - snowbell
      - heathcliff
      - felix
      - sylvester
```

创建容器并通过使用 sequence 命名：

```
- hosts: web
  sudo: yes
  tasks:
  - name: run tomcat servers
    docker: image=centos name={{item}} command="service tomcat6 start" ports=8080
    with_sequence: start=1 end=5 format=tomcat_%d.example.com
```

创建两个连接的容器：

```
- hosts: web
  sudo: yes
  tasks:
  - name: ensure redis container is running
    docker: image=crosbymichael/redis name=redis

  - name: ensure redis_ambassador container is running
    docker: image=svendowideit/ambassador ports=6379:6379 links=redis:redis name=redis_ambassador_ansible
```

创建容器，指定选项作为键值对和列表：

```
- hosts: web
  sudo: yes
  tasks:
  - docker:
        image: namespace/image_name
        links:
          - postgresql:db
          - redis:redis
```

Create containers with options specified as strings and lists as comma-separated strings:

```
- hosts: web
  sudo: yes
  tasks:
  docker: image=namespace/image_name links=postgresql:db,redis:redis
```

创建一个没有 networking 的容器：

```
- hosts: web
  sudo: yes
  tasks:
  docker: image=namespace/image_name net=none
```


  [1]: http://docs.ansible.com/docker_module.html