# 使用 psutil 和 MongoDB 做系统监控

标签（空格分隔）： psutil MongoDB Python jquery bottle

---

> 注：原文地址 [psutil and MongoDB for System Monitoring][1]

这篇入门文章描述了怎样创建一系列的图表来监控一台或多台服务器的负载。使用 Python（psutil 和 bottle），MongoDB 和 jquery。不管你使用什么样的数据库或 WEB 框架，思路都是一样的。

在最后，你将有一个 web 页面为每台服务器展示图表，图表中显示了 cpu， memory， 和 disk usage。

## 大纲

- Part 0: Get the Tools
- Part 1: Get the Data
    - About the MongoDB Collection.
    - The Data Gathering Code
- Part 2: Set up the bottle Server
- Part 3: Display the Data with jqplot
    - The HTML Page
    - The JavaScript (jqplot) Code

我们需要监控两台 FreeBSD  服务器来确保它们是正常的，运行期间没有内存和磁盘使用率问题。 为了这篇文章的目的，这两台服务器的名字是 example01 和 example02。

> 注意：这些恰好是相同的机器运行 MongoDB 副本集，它们中的一台运行 web 服务。没有理由它们必须是相同的机器 - 你完全可以让 MongoDB 运行在一台不同的服务器上，而不是你想监控的其中一台。这对于 web 服务也是一样的 - 可以在任何服务器上，不是必须得在你监控的其中一台服务器上。


我不想登录每台服务器，然后运行 `top` 或 `ds` 来找出发生了什么。我想要一个有最新图表的 web 页面可以看一眼，每台服务器一个页面。


整个工作流遵循以下三个步骤：

  - 获取系统数据存入 MongoDB（使用 `psutil` 和 `cron`）
  - 设置一个 Web 服务器来查询  MongoDB 的数据（使用 `bottle`）
  - 编写一个简单的 Web 页面来展示这些数据（`jqplot` + AJAX）

![此处输入图片的描述][2]

你可以在 GitHub 工程 [psmonitor][3] 上获取所有的代码。在这篇文章中，我使用一些代码片段。

在第一部分，你在一个 `cron` 任务中使用 `psutil` Python 包每五分钟写系统信息到 MongoDB 的一个 capped collection 中。这 5 分钟是完全任意的 - 你可以选择你喜欢的任意时段。我正在监控的系统，几分钟提供足够细的粒度。这部分把数据存入 MongoDB。

在第二部分， `bottle` 应用程序发出一个请求给 MongoDB，然后得到一个 JSON 数据格式的响应。这是在客户端的 HTML 页面和 MongoDB 数据直接创建了一个代理。

在第三部分，你有一个 HTML 文件匹配到你想监控的每台服务器上。该文件加载 `jqplot` 并发出一个 AJAX 请求给 `bottle` 应用程序。这部分是波动的原因 - 我们获取存储在 MongoDB 中的负载数据的时间序列图表。

这个我们创建的其中一个图表的示例：

![此处输入图片的描述][4]

你可以为你想要的任何数据创建图表。我为每台机器使用的图表是：

 - cpu user percent
 - cpu system percent
 - cpu irq
 - cpu nice percent
 - disk space free
 - memory free

## Part 0: Get the Tools

以下的工具需要通过安装来实现，但是你可以使用一个不同的数据库或是 Web 框架，如果你已经有的话。

`psutil` 是一个非常好用的跨平台的系统监控工具，如果想了解更多，请移步至其[项目主页][5]，你可以通过 `pip` 安装它。

NoSQL 数据库 [MongoDB][6] 是一个开源的文档型数据库。如果你有自然适合 JSON 结构的数据，MongoDB 会是一款非常适合存储数据的工具。对我来说，它已经成为了首选的工具，当我随时发现需要存储 JSON 格式数据的时候。有趣的是，我越使用它，越能找到新的用途。MongoDB 的文档型结构是惊人的有用，并且可以为许多信息类型提供 map 。

`bottle` Web 框架是一个用 Python 写的没有任何依赖的小型框架。你可以通过 `pip` 安装它。你可以使用包括开发服务器让事情继续，稍后把它放在一个不同的后端服务器。我把我的放在 Apache 服务器，一旦我得到我希望他们的。我稍后将写一篇文章关于这个的设置。

[jqplot][7] jquery 插件使得生产图表更容易，加上它们看起来也不错，并且它们是统一的很容易的比较图表。比如，当一个进程波动起来的时候，在图表中很容易看出来，因为你可以同时看到 cpu 和 内存飙升，这两个图表是同样的 X 轴位置。你可以使用这个插件做很多事情，这个练习仅仅只是皮毛。

## Part 1: Get the Data

当我们想遵循这样的模式创建一个数据结构的时候，在这步中你可以添加和删除任何 psutil 支持的数据。终点是会看 `jqplot` 图表的用户，因此在你的脑海里保持数据结构，`jqplot` 想要的是一个双元素列表的列表，其中一个是时间，像这样：

```
[[datetime1, y-value1], [datetime2, y-value2], and so on.]
```

那个结构不是收集数据的最有效方式，但是记住你的初心是什么一直是很重要的。我们将收集数据并且改变数据结构以便给 `jqplot` 需要的数据结构。

当第一步我不知道我需要什么和使用什么，我有更多一些测量。我认为，随着时间的流逝，我看到机器的实际需求将改变，并且它非常容易改变。

```
{
  'server': servername,
  'datetime': datetime.now(),
  'disk_root': ,
  'phymem':,
  'cpu': {'user':, 'nice':, 'system':, 'idle':, 'irq':,},
}
```

下面是 python 代码从系统获取数据（使用 psutil ）到 MongoDB 数据库。我在 MongoDB 中为每个被检测到的机器创建了一个集合。


你可以在 MongoDB 中创建一个单独的 document，包含了每台机器的数据；这个依赖于你的需求。因为我想每台机器有一个单独的页面，我用同样的方式拆分数据。如果你想把所有机器的图表渲染到一个页面中的话，你或许想让所有机器的数据在一个 document 中。


## About the MongoDB Collection

我有一个三个成员的 MongoDB 副本，设置名为 `rs1`。保存数据的机器恰巧是我监控的服务器，example01 和 example02。这第三个是一个仲裁者，不保存数据。这些 mongoDB 服务器不需要监控，它们可以在任何地方。

我有一个数据库 `reports`，我们将把新的 collections 放入这个数据库。对于每台机器，我将有一个 collection 来包含它的负载数据：1440 分每天，每 5 分组抽样一次，并保存 2 天的数据，我们需要 576 条记录（documents）。

```
(1440/5)*2 = 576 records per server
```

我不确定我最后要使用多少数据，因此我预估了 2k 每个 document，我为每个 document 预估一个比较大的大小， 因为这仅仅是个开始，后面我需要收集更多的数据，结果是 2k 真的是非常慷慨了，每个记录的平均大小是 200 bytes 左右，但是我没有包含任何的网络数据（并且 disk space 是低耗的）。

```
576 documents @ 2048 bytes per doc = 1,179,648 bytes
```

对于每一台机器需要监控的机器，我创建了一个固定集合的最大大小上限 1179648 和一个最大的数量上限 576 documents：

```
use reports
db.createCollection('example01', {capped:true, size:1179648, max:576})
db.createCollection('example02', {capped:true, size:1179648, max:576})
```

通过使用一个固定集合，我们将保证数据是以插入顺序保存的，随着时间流逝，老的 documents 会自动删除，因此我们始终有最新的 48 小时数据。


## The Data Gathering Code

首先，做必须的 imports 并且连接到 MongoDB 实例：

```
from datetime import datetime
import psutil
import pymongo
import socket

conn = pymongo.MongoReplicaSetClient(
    'example01.com, example02.com',
    replicaSet='rs1',
)
db = conn.reports
```

现在为你想要的每一个数据调用 `psutil`：

```
def main():
    cpu = psutil.cpu_times_percent()
    disk_root = psutil.disk_usage('/')
    phymem = psutil.phymem_usage()
```

创建一个字典包含你需要的时间序列结构数据。

```
    doc = dict()
    doc['server'] = socket.gethostname()
    doc['date'] = datetime.now()
    doc['disk_root'] = disk_root.free, 
    doc['phymem'] = phymem.free

    doc['cpu'] = {
        'user': cpu.user, 
        'nice': cpu.nice,
        'system': cpu.system, 
        'idle': cpu.idle,
        'irq': cpu.irq
    }
```

最后，把这个字典作为一个 document 添加进匹配的 MongoDB 集合中。它将被转换成一个 BSON document 当它被插入数据库的时候，但是结构是一样的。

```
    if doc['server'] == 'example01.com':
        db.example01.insert(doc)
    elif doc['server'] == 'example02':
        db.example02.insert(doc)
```

这里你有代码来获取数据并且存储进数据库集合中。所有剩下的部分是运行代码自动完成的。

在你想监控的每台服务器上设置一个定时任务，每 5 分钟运行一次：

```
*/5 * * * * /path/to/psutil_script
```

每个 MongoDB 集合包含 48 小时的系统性能数据，随你操弄。

## Part 2: Set up the bottle Server

创建一个 `bottle` 应用程序来查询 MongoDB集合。

连接 MongoDB，在收到每个请求服务器的数据后，给每个对应的服务器响应格式化的数据。

```
from bottle import Bottle
import pymongo
load = Bottle()

conn = pymongo.MongoReplicaSetClient(
    'example01.com, example02.com',
    replicaSet='rs1',
)
db = conn.reports
```

这是一个路由，一个 url 连接。当一个请求进来，从 url 中获取服务器的名字，然后创建并返回一个合适的数据结构（`jqplot` 需要的）。

```
@load.get('/<server>')
def get_loaddata(server):
    data_cursor = list()
    if server == 'example02':
        data_cursor = db.example02.find()
    elif server == 'example01':
        data_cursor = db.example01.find()

    disk_root_free = list()
    phymem_free = list()
    cpu_user = list()
    cpu_nice = list()
    cpu_system = list()
    cpu_idle = list()
    cpu_irq = list()

    for data in data_cursor:
        date = data['date']
        disk_root_free.append([date, data['disk_root'])
        phymem_free.append([date, data['phymem'])
        cpu_user.append([date, data['cpu']['user']])
        cpu_nice.append([date, data['cpu']['nice']])
        cpu_system.append([date, data['cpu']['system']])
        cpu_idle.append([date, data['cpu']['idle']])
        cpu_irq.append([date, data['cpu']['irq']])

    return {
            'disk_root_free': disk_root_free,
            'phymem_free': phymem_free
            'cpu_user': cpu_user,
            'cpu_irq': cpu_irq,
            'cpu_system': cpu_system,
            'cpu_nice': cpu_nice,
            'cpu_idle': cpu_idle,
            }
```

## Part 3: Display the Data with jqplot

**HTML Page**

HTML 页是非常简单的。

1. 在样式表中读取.
2. 编写一个 div 保存每个图表。这以下的示例中我仅仅展示了 cpu_user 数据。该模式对于其他变量是一样的。 
3. 加载 javascript。

你可以把 javascript 从 `psmonitor.js` 中直接放入页面，或者是以一个文件的形式调用它（如示例那样）：

```
<!doctype html>
<html>
<head>
<meta charset="utf-8">
<title>Load Monitoring</title>
<link rel="stylesheet" type="text/css" href="/css/jquery.jqplot.css" />
<style>div {margin:auto;}</style>
</head>
<body>

    <div id="cpu_user" style="height:400px;width:800px; "></div>

<script src="http://code.jquery.com/jquery-latest.min.js" ></script>
<script type="text/javascript" src="/js/jquery.jqplot.js"></script>
<script type="text/javascript" src="/js/jqplot.json2.js"></script>
<script type="text/javascript" src="/js/jqplot.dateAxisRenderer.js"></script>
<script type="text/javascript" src="/js/jqplot.highlighter.js"></script>
<script type="text/javascript" src="/js/jqplot.cursor.js"></script>
<!--[if lt IE 9]>
  <script language="javascript" type="text/javascript" src="excanvas.js"></script>
<![endif]-->
<script type="text/javascript" src="/js/psmonitor.js" />
</body>
</html>
```

## The JavaScript (jqplot) Code

完整的代码在 GitHub 工程中，但是这有 `jqplot` 代码片段用来设置显示数据。机器 `example01` 运行的 web 服务器将返回 json 格式的负载数据。同样，web 服务器可以运行在任何机器上，在我的示例中，它发生在被我们监控的服务器中的一台。

每个 plot 的代码遵循了相同的模式：

1. 发起一个 AJAX 请求调用运行着 bottle 应用的服务器。 
2. 把你想图表化的数据放入一个变量。
3. 把这个变量传递给 jqplot。


代码中的 `url` 包含了 `example01` 字符串：

```
url: "http://example01/load/example01"
```

`example01` 的第一个实例是 web 服务器的地址，因为该机器上运行着 `bottle` 应用。第二个实例是我们想要数据的这台服务器的名字。服务器名字（<server>）被传递进 `bottle` 路由（get_loaddata）检索 MongoDB 记录（documents）：

```
$(document).ready(function(){
    var jsonData = $.ajax({
      async: false,
      url: "http://example01/load/example01",
      dataType:"json"
    });

    var cpu_user = [jsonData.responseJSON['cpu_user']];

    $.jqplot('cpu_user',  cpu_user, {
        title: "CPU User Percent: EXAMPLE01",
        highlighter: {show: true, sizeAdjust: 7.5},
        cursor: {show: false},
        axes:{xaxis:{renderer:$.jqplot.DateAxisRenderer, 
              tickOptions:{formatString:"%a %H:%M"}}},
        series:[{lineWidth:1, showMarker: false}]
    });
});
```

该 javascript 划分 CPU 用户百分比；你可以用相同的方式添加另外的 plots，仅仅需要改变变量的名字和标题。 



  [1]: http://reachtim.com/articles/psutil-and-mongodb-for-system-monitoring.html
  [2]: http://reachtim.com/images/psflow.png
  [3]: https://github.com/tiarno/psmonitor
  [4]: http://reachtim.com/images/monitor01.png
  [5]: http://pythonhosted.org/psutil/
  [6]: http://www.mongodb.org/
  [7]: http://www.jqplot.com/