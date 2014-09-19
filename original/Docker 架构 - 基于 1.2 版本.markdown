# Docker 架构 - 基于 1.2 版本

标签（空格分隔）： Docker 架构 Architecture CGroups Namespaces aufs vfs devmapper container

---

> 注：该文作者是 [rajdeep][1]，原文地址 [Docker Architecture (v1.2)][2]

> 注：该文是由一篇 slide 翻译而来。

## 在开始之前，我们需要了解

**什么是容器？**
   

 - 一组进程包含在隔离的环境 
 - 通过类似 cgroups 和 namespaces 的概念提供隔离

**什么是 Docker？**



  - 使用镜像的概念实现一个轻便的容器
  - 镜像是轻便且可发布的

## CGroups

 - 限制、记录（account）和隔离一组进程的资源使用（CPU，内存，磁盘 I/O，等等。）
 - 资源限制：组可以被设置不超过一组内存限制 - 这也包括文件系统 cache。
 - 优先级：一些组可能获得更大的 CPU 分配和磁盘 I/O 吞吐量
 - 记录（account）：为了测量某些系统使用了多少资源
 - 控制：冻结组或检查点和重起。

![此处输入图片的描述][3]
 
## Namespace

 - 分区必不可少的内核结构来创建虚拟环境
 - 不同的 Namespaces
    - pid（进程）
    - net（网络接口，路由。。。）
    - ipc（System V IPC）
    - mnt（挂载点，文件系统）
    - uts（hostname）
    - user（UIDs）

        ![此处输入图片的描述][4]

## Docker

 - 管理镜像和运行期容器
 - 后端支持多样的文件系统
 - 多个 Execdriver 容器实现
 - 客户端和服务器端组件 - 使用 HTTP 和 unix sockets 配合

## Docker 运行期组件

![此处输入图片的描述][5]
 
 
## Docker 引擎

 - Docker 核心：容器存储
 - 使用任务管理容器（类似 unix 的任务）
 - 容器处理封装了任务的函数
 - 所有的动作使用任务执行

![此处输入图片的描述][6]
 
 
## Docker 初始化

 - Docker 的主函数：docker.main()
 - 调用：mainDaemon()
 - 实例化引擎：eng := engine.New()
 - 内部注册：built-­‐ins builtsin.Register(eng)
 - 实例化任务：job := eng.Job(“initserver”) 
 - 为任务设置变量
 - 运行任务： job.run()
 - 启动接受的连接：eng.Job(“AcceptConnections”).run()
 - 内部注册：
    
    ```
    nstantiate daemon(eng) 
    eng.Register("initserver", server.InitServer) 
    eng.Register(“init_networkdriver”, bridge.InitDriver)
    ```
> 注：感觉作者这里凌乱了，处女座受不了啊。见图    
![此处输入图片的描述][7]
![此处输入图片的描述][8]
![此处输入图片的描述][9]
 
## Daemon

 - 主入口点管理容器的所有请求
 - 维护以下引用的数据结构：
       - ImageGraph
       - Volume Graph
       - Engine
       - ExecDriver
       - Server
       - ContainerStore
![此处输入图片的描述][10]
       
## Daemon - Graph

 - Graph 是一个存储系统文件版本和镜像关系的数据结构
 - 为每一个容器实例化一个 Graph
 - 引用一个 graphdriver.Driver
 - 在一个 Graph 上的动作：
     - 创建一个新的 Graph
     - 从一个 Graph 中获取镜像
     - 恢复一个 Graph
     - 创建一个镜像并且注册进 Graph
     - 在 Graph 上注册一个预先存在的镜像

![此处输入图片的描述][11]

## 在 Docker 中镜像和容器的概念

 - Docker 镜像是文件系统中的一层
 - 容器是两层：
     - 层一是基于镜像的初始化层
     - 层二是实际的容器内容

![此处输入图片的描述][12]

## Graph Driver

 - 被 Daemon 引用
 - 用于抽象多种后端存储
 - 加载以下后端文件系统：
     - aufs
     - Device mapper（devmapper）
     - vfs
     - btrfs

![此处输入图片的描述][13]
     
     
## 容器存储

 - 持久化后端的容器数据
 - 使用 SQLite 实现
 - 从 Daemon 引用
 - containGraph：graph
 - 在 Daemon 恢复期间用于加载容器信息

![此处输入图片的描述][14]
     
## Volume Graph

 - 基于 Graph 的简单 vfs，为了与容器卷保持联系
 - Volumes 使用在 Daemon 中的卷驱动器来创建和连接容器的卷
 - 每个容器被分配一个或多个卷
 
![此处输入图片的描述][15]
 
## ExecDriver

 - 对底层 Linux 控制的抽象
 - 从 daemon 调用
 - 支持以下实现
     - LXC
     - Native

![此处输入图片的描述][16]
     

## 驱动接口

 - 抽象接口与底层实现交互

![此处输入图片的描述][17]
 
 
## 驱动接口 - 网络

 - 抽象接口与底层实现交互

![此处输入图片的描述][18]
 

## libcontainer

 - 容器的底层原生实现
 - 被原生的驱动使用
 - Container.config - 一个容器数据的表示
 - 包装过的 cgroups 和 Namespaces

![此处输入图片的描述][19]
 

## 原生的驱动实现

![此处输入图片的描述][20]

## 创建容器的步骤

1. Engine -­‐> Daemon -­‐> ContainerCreate
2. ContainerCrea



      2.1 检查定义在配置文件中的内存是比 512k 大还是系统定义的限制小
      2.2 检查 SwapLimit
      2.3 调用 Daemon -­‐> Create
      
          2.3.1 daemon.repositories.LookupImage -­‐> tagStore.getImage()
          2.3.2 daemon.newContainer() 
              2.3.2.1 NewContainerMonitor()
          2.3.3 daemon.createRootFs()
              2.3.3.1 daemon.container.driver.Create()[Graph Driver -­‐ aufs, btrfs, devicemapper]
              2.3.3.2 container.ToDisk()//持久化容器

![此处输入图片的描述][21]


## 总结

 - Linux 控制原则
 - Docker 架构组件
 - 原生的驱动实现
 - libcontainer
 - 容器创建

![此处输入图片的描述][22]
 


  [1]: http://www.slideshare.net/rajdeep
  [2]: http://www.slideshare.net/rajdeep/docker-architecturev2
  [3]: http://image.slidesharecdn.com/docker-architecture-v2-140906002548-phpapp02/95/docker-architecture-v12-3-638.jpg?cb=1409981482
  [4]: http://image.slidesharecdn.com/docker-architecture-v2-140906002548-phpapp02/95/docker-architecture-v12-4-638.jpg?cb=1409981482
  [5]: http://image.slidesharecdn.com/docker-architecture-v2-140906002548-phpapp02/95/docker-architecture-v12-6-638.jpg?cb=1409981482
  [6]: http://image.slidesharecdn.com/docker-architecture-v2-140906002548-phpapp02/95/docker-architecture-v12-7-638.jpg?cb=1409981482
  [7]: http://image.slidesharecdn.com/docker-architecture-v2-140906002548-phpapp02/95/docker-architecture-v12-8-638.jpg?cb=1409981482
  [8]: http://image.slidesharecdn.com/docker-architecture-v2-140906002548-phpapp02/95/docker-architecture-v12-9-638.jpg?cb=1409981482
  [9]: http://image.slidesharecdn.com/docker-architecture-v2-140906002548-phpapp02/95/docker-architecture-v12-10-638.jpg?cb=1409981482
  [10]: http://image.slidesharecdn.com/docker-architecture-v2-140906002548-phpapp02/95/docker-architecture-v12-11-638.jpg?cb=1409981482
  [11]: http://image.slidesharecdn.com/docker-architecture-v2-140906002548-phpapp02/95/docker-architecture-v12-12-638.jpg?cb=1409981482
  [12]: http://image.slidesharecdn.com/docker-architecture-v2-140906002548-phpapp02/95/docker-architecture-v12-13-638.jpg?cb=1409981482
  [13]: http://image.slidesharecdn.com/docker-architecture-v2-140906002548-phpapp02/95/docker-architecture-v12-14-638.jpg?cb=1409981482
  [14]: http://image.slidesharecdn.com/docker-architecture-v2-140906002548-phpapp02/95/docker-architecture-v12-15-638.jpg?cb=1409981482
  [15]: http://image.slidesharecdn.com/docker-architecture-v2-140906002548-phpapp02/95/docker-architecture-v12-16-638.jpg?cb=1409981482
  [16]: http://image.slidesharecdn.com/docker-architecture-v2-140906002548-phpapp02/95/docker-architecture-v12-17-638.jpg?cb=1409981482
  [17]: http://image.slidesharecdn.com/docker-architecture-v2-140906002548-phpapp02/95/docker-architecture-v12-18-638.jpg?cb=1409981482
  [18]: http://image.slidesharecdn.com/docker-architecture-v2-140906002548-phpapp02/95/docker-architecture-v12-19-638.jpg?cb=1409981482
  [19]: http://image.slidesharecdn.com/docker-architecture-v2-140906002548-phpapp02/95/docker-architecture-v12-20-638.jpg?cb=1409981482
  [20]: http://image.slidesharecdn.com/docker-architecture-v2-140906002548-phpapp02/95/docker-architecture-v12-21-638.jpg?cb=1409981482
  [21]: http://image.slidesharecdn.com/docker-architecture-v2-140906002548-phpapp02/95/docker-architecture-v12-22-638.jpg?cb=1409981482
  [22]: http://image.slidesharecdn.com/docker-architecture-v2-140906002548-phpapp02/95/docker-architecture-v12-23-638.jpg?cb=1409981482