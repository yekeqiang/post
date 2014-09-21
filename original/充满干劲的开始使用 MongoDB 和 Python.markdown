# 充满干劲的开始使用 MongoDB 和 Python

标签（空格分隔）： MongoDB Python

---

> 注：原文作者 [Stephen B. Morris][1]，原文地址 
[Hit the Ground Running with MongoDB and Python][2]


Stephen B. Morris 描述了怎样开始使用 MongoDB 和 Python。像往常一样使用 Python，你可以很快开始生产使用，不用担心复杂的 IDEs。MongoDB 有一个简单的数据模型和容易明白的语义。可以让你很方便的进入到这门有趣的技术当中。

NoSQL 数据库引擎比如 MongoDB 目前是相当时髦的。我过去谈过的每一个程序员看起来似乎都想用 其中之一的 NoSQL 数据库引擎工作。最经常被引用的论点是这种技术是可提高可伸缩性。我不太确定在所有情况下的可伸缩性问题。但是 NoSQL 技术肯定有一些坚实的用例，相对于关系型数据库。

除了程序员（包括我自己）工作想使用最新技术的愿望，有某些原因使用 MongoDB 和其他 NoSQL 产品。在 IT 的大多数情况下，语言选择通常归结为成本。最近我注意到，为了 努力简化 IT 开发和维护工作，倾向于使用更轻量级的技术。


我的文章“[通过使用 Python，保护 C++ 遗留程序][3]” 和 “[Exception Management in C++ and Python Development: Planning for the Unexpected][4]” 相当详细的讨论了程序语言，专注于 JAVA 和 Python。这些文章的主题是 Python 代码开发相对容易，与 JAVA 代码相比。当然，JAVA 是更重量级的解决方案，带来了如开箱即用的线程安全，强类型，和安全等好处。与之相比，在很多情况下 Python 允许你更快的得到原型。这就是现在很多机构使用 Python 构建原型的其中一个原因。Python 原型被测试，然后被 JAVA 实现代替（或是任何其他的主流高级语言）。

数据库区域是 IT 成本的另外一个关键因素。许可费用并不是真正的问题；高性能，开源数据库引擎比如 MySQL 是广泛普及的，多年的使用已经证明了。当然啦，关键的成本是软件开发和维护费用。

> 小便条：欲了解更多关于复合编码，一定要去看 Stephen 的博客“[Coding in the Multi-Language Era][5]”


## 安装和运行 MongoDB

考虑到这些问题，让我们来看看 MongoDB 和 Python 加起来的是怎样有趣的组合。一如既往，一个良好的开始是安装和可玩的测试系统。这篇文章的示例我使用 Ubuntu 12.04.4 LTS。但这不是强制性的（你可以使用其他系统）。

MongoDB 的启动和运行时非常简单的：在 Listing 1 中仅仅需要运行 5 个命令即可安装和运行 MongoDB。

### Listing 1 - 安装和启动 MongoDB

```
sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv 7F0CEB10
echo 'deb http://downloads-distro.mongodb.org/repo/ubuntu-upstart
 dist 10gen' | sudo tee /etc/apt/sources.list.d/mongodb.list
sudo apt-get update
sudo apt-get install mongodb-org
sudo service mongod start
```

Listing 1 中的最后一个命令实际上可能是多余的，因为安装的时候默认启动了服务。我包含这个命令是以防引擎没有自动启动。

为了验证 MongoDB 是否运行，使用这个命令：

```
/etc/init.d/mongod status
```

这个命令会产生类似 Listing 2 中的状态信息内容。

### Listing 2 — MongoDB 正在运行

```
Rather than invoking init scripts through /etc/init.d, use the service(8)

utility, e.g. service mongod status

Since the script you are attempting to invoke has been converted to an

Upstart job, you may also use the status(8) utility, e.g. status mongod

mongod start/running, process 11969
```

> 注意：如果你喜欢，你可以使用相等的 Upstart 验证 MongoDB 是否在运行。 `service mongod status`

如果安装的一切顺利，你现在应该已经有一个已经安装的正在工作的 MongoDB。

让我们尝试使用这个数据库引擎。以下的例子是基于 [MongoDB 在线手册][6]。

### 连接 MongoDB

连接 MongoDB 非常容易：只需键入：

```
mongo
```

这个命令应该启动 MongoDB shell 并显示在 Listing 3 中的文本。

### Listing 3 — 连接 MongoDB

```
MongoDB shell version: 2.6.3

connecting to: test

Server has startup warnings: 

2014-07-30T16:34:05.028+0100 [initandlisten] 

2014-07-30T16:34:05.028+0100 [initandlisten] ** NOTE: This is a 32 bit MongoDB binary.

2014-07-30T16:34:05.028+0100 [initandlisten] **       32 bit builds are limited to less than 2GB of data (or less with --journal).

2014-07-30T16:34:05.028+0100 [initandlisten] **       Note that journaling defaults to off for 32 bit and is currently off.

2014-07-30T16:34:05.028+0100 [initandlisten] **       See http://dochub.mongodb.org/core/32bit

2014-07-30T16:34:05.028+0100 [initandlisten] 

> 
```

在  Listing 3 中，在最后一行提供了一个实时的 (>) 来与 MongoDB 交互。在本文的其余部分，我将描述在 console 中可以被键入的一些命令。为了开始使用 console，使用这个命令显示当前的 MongoDB 数据库：

```
db
```

这个命令产生以下输出：

```
> db

test
```

为了查看所有的数据库，使用这个命令：

```
> show dbs

admin     (empty)

local     0.078GB

mydb      0.078GB

testData  0.078GB
```

> 注意：MySQL 用户或许觉得 MongoDB 命令有些相似。我第一次使用 Mongo 也有这样的感觉。

为了使用一个指定的数据库，键入以下命令，跟着使用 db 来验证正确的数据库上下文被选择。

```
> use mydb

switched to db mydb
```

现在，检查选择的数据库：

```
> db

mydb
```

我们已经验证选择的数据库就是 mydb，现在让我们添加一些数据进 mydb。记住 MongoDB 是一个开源的文档型数据库引擎。这个与其他关系型数据库的模型是非常不一样的。你必须创建 documents，然后添加进 Mongo。这听起来很困难，但是实际很容易：

```
> j = { name : "mongo" }

{ "name" : "mongo" }
```

这个命令创建了一个名为 j 的 document。它是一个  field-value  对。它可以这样被写入 MongoDB：

```
> db.mydb.insert(j)

WriteResult({ "nInserted" : 1 })
```

这个例子证明了 MongoDB 另外一个很重要的原则：MongoDB 是一个动态模式技术。

现在让我们添加另外一个名为 j 的 document：

```
> j = { name : "mongo", age : "25" }

> db.mydb.insert(j)
```

为了检索插入的 documents，键入：

```
> db.mydb.find()

{ "_id" : ObjectId("53d9156445db8daa9930cfe2"), "name" : "mongo" }

{ "_id" : ObjectId("53d9176445db8daa9930cfe3"), "name" : "mongo", "age" : "25" 
```

注意两条记录都从 MongoDB 检索出来。每条记录取得一个 ObjectId 属性，被 MongoDB 自动创建，用来区别于其他记录。另外一个需注意的是类 JSON 格式的数据结构。


### 游标和循环

它也可以使用游标遍历所有返回的数据，这个技术随着数据增长，非常有益。为了获取一个游标，使用以下命令：

```
> var c = db.mydb.find()
```

下一步，如下面这样迭代所有游标：

```
> while (c.hasNext()) printjson(c.next()){ "_id" : ObjectId("53d9156445db8daa9930cfe2"), "name" :
 "mongo" }{       "_id" : ObjectId("53d9176445db8daa9930cfe3"),       "name" : "mongo",       "age" : "25"}
```

你可以验证，我们早先保存的两个 documents 在游标迭代中被发现了。注意 printjson() 的使用，把数据渲染成类 JSON 格式。是否 JSON 与 JavaScript 相关？事实证明, 类json格式 的使用为 MongoDB 的使用提供了额外的灵活性。

## 使用 JavaScript 与 MongoDB

mongo shell并不是与 MongoDB 交互的唯一选择，你也可以使用 JavaScript 作为一个替代的访问方式。一个脚本接口比直接在 console 下提供了更多的便利。这是因为你的脚本可大可小，并包含任意数量的命令。尽管如此，JavaScript 命令格式是与直接使用  MongoDB shell 有点不同。使用 JavaScript 作为 MongoDB 的访问技术是在靠近其他 web 技术。

### 具体查询

为了查询数据库中的一个具体 document，只需使用相应的数据值：

```
> db.mydb.find( { name : "mongo", age : "25" } )

{ "_id" : ObjectId("53d9176445db8daa9930cfe3"), "name" : "mongo", "age" : "25" }
```

注意以上查询仅仅是返回数据库中两个 documents 中的其中一个。

我们的 MongoDB 安装基本完成和校验了。现在让我们安装一些 Python 工具和使用。

### 安装 PyMongo

PyMongo 使用 MongoDB 的发行版设计。通过使用 pip 是非常容易安装的，安装 pip：

```
sudo apt-get install python-pip
```

下一步，安装 PyMongo：

```
sudo pip install pymongo
```

这个命令应该产生如下信息：

```
Downloading/unpacking pymongo

  Running setup.py egg_info for package pymongo

Installing collected packages: pymongo

  Running setup.py install for pymongo

Successfully installed pymongo

Cleaning up...
```

当你看到以上结果，你应该准备开始运行 PyMongo 代码，像往常一样，先使用最简单的选择： Python console。键入以下命令：

```
python
```

你应该得到如下结果：

```
Python 2.7.3 (default, Feb 27 2014, 20:00:17) 
[GCC 4.6.3] on linux2
Type "help", "copyright", "credits" or "license" for more information.
>>>
```

现在校验 PyMongo 是否被安装：

```
>>> import pymongo
```

如果你没有从 import 语句得到错误，证明你已经安装成功。

现在引入 MongoClient：

```
>>> from pymongo import MongoClient

>>> client = MongoClient()
```

输出 client 对象的详细信息是非常有用的。注意，默认的主机和端口被使用了（如果你愿意，你可以在创建客户端时，指定这些详细信息）。

```
>>> clientMongoClient('localhost', 27017)
```

### Listing PyMongo Databases

为了查看数据库，使用客户端对象的 Python 语句：

```
>>> client.database_names()

[u'mydb', u'testData', u'local', u'admin']
```

现在获取一个 mydb 数据库的连接：

```
>>> db = client.mydb
```

### Listing PyMongo Database Collections

获取 collections 列表：

```
>>> db.collection_names()

[u'testData', u'system.indexes', u'mydb']
```

### 与一个 Collection 交互

为了从一个给定的 collection 中获取 第一个 document，使用这个命令：

```
>>> db.mydb.find_one()

{u'_id': ObjectId('53d9156445db8daa9930cfe2'), u'name': u'mongo'}
```

为了从一个 collection 获取指定的 item，使用这个命令：

```
>>> db.mydb.find_one()

{u'_id': ObjectId('53d9156445db8daa9930cfe2'), u'name': u'mongo'}
```

为了扩展查询参数，添加更多的属性：

```
>>> db.mydb.find_one({"name": "mongo", "age":"25"})

{u'age': u'25', u'_id': ObjectId('53d9176445db8daa9930cfe3'), u'name': u'mongo'}
```

为了看得查询失败是怎样的，添加一个无效的属性：

```
>>> db.mydb.find_one({"name": "mongo", "age":"26"})
```

正如你所看到的,很容易进入这个聪明的技术！现在让我们来写一些 Python 代码与 MongoDB 程序交互

## 使用 Python 把数据插入 MongoDB 数据库

让我们看一下如何使用 Python 把时间戳数据插入 MongoDB：

```
>>> import datetime
```

下一步，创建一个对象插入：

```
>>> post = {"author": "Mick",

...     "text": "My very first blog post",

...     "tags": ["mongodb", "python", "pymongo"],

...     "date": datetime.datetime.utcnow()}
```

查看最新创建的对象：

```
>>> post

{'date': datetime.datetime(2014, 7, 31, 11, 57, 11, 535966), 'text':
 'My very first blog post', 'tags': ['mongodb', 'python', 'pymongo'], 'author': 'Mick'}
```

在我们可以把数据插进数据库之前，我们需要创建一个新的 collection，这里叫做 posts：

```
>>> posts = db.posts
```

> 警告：你可以称呼任何你喜欢的 collection，我称呼我的为 posts。但是不要混淆我使用的这个词 post 和 HTTP post 命令。他们之间没有任何关联。

校验在数据库中已经被创建的新的 collection：

```
>>> db.collection_names()
[u'testData', u'system.indexes', u'mydb', u'posts']
```

现在你可以把 document 添加进 collection：

```
>>> post_id = posts.insert(post)
```

如果没有错误消息，posts 数据已经被写成功，你可以通过打印 post_id 值来校验结果：

```
>>> post_id
ObjectId('53da301d83cc1d283b1df18a')
```

像以前一样,我们可以查看新的 collection 中的第一个条目：

```
>>> posts.find_one()

{u'date': datetime.datetime(2014, 7, 31, 11, 57, 11, 535000), u'text':
 u'My very first blog post', u'_id': ObjectId('53da301d83cc1d283b1df18a'),
 u'author': u'Mick', u'tags': [u'mongodb', u'python', u'pymongo']}
```

注意 ObjectId 值和以上返回的 post_id 值匹配。

### 使用 PyMongo 搜索 MongoDB

在执行任何搜索之前，让我们添加另外一个 document 到新的 collection 中，如前所述，使用相同的过程：

```
>>> post = {"author": "Frank",

...     "text": "My first blog posting",

...     "tags": ["java","mongodb","python"],

...     "date": datetime.datetime.utcnow()}

>>> post

{'date': datetime.datetime(2014, 7, 31, 12, 12, 59, 74803), 'text': 'My first blog posting',
 'tags': ['java', 'mongodb', 'python'], 'author': 'Frank'}

>>> post_id = posts.insert(post)

>>> post_id

ObjectId('53da32dd83cc1d283b1df18b')
```

像以前那样，让我们搜索其中一个作者，因此强制 MongoDB 区分 collection 之间的记录。

```
>>> posts.find_one({"author":"Mick"})

{u'date': datetime.datetime(2014, 7, 31, 11, 57, 11, 535000), u'text':
 u'My very first blog post', u'_id': ObjectId('53da301d83cc1d283b1df18a'),
 u'author': u'Mick', u'tags': [u'mongodb', u'python', u'pymongo']}
```

我们找到了第一个作者，把作者的属性名改为 Frank，再搜索下：

```
>>> posts.find_one({"author":"Frank"})

{u'date': datetime.datetime(2014, 7, 31, 12, 12, 59, 74000), u'text': u'My first blog posting',
 u'_id': ObjectId('53da32dd83cc1d283b1df18b'), u'author': u'Frank', u'tags': [u'java', u'mongodb', u'python']}
```

再一次检索条目成功，但是如果我们搜索一个不存在的条目会发生什么？

```
>>> posts.find_one({"author":"John"})
```

没有任何返回，因为没有该属性名字的条目存在。



  [1]: http://www.informit.com/authors/bio/ff0e7b03-51ce-4983-9e3f-68485b1e8ceb
  [2]: http://www.informit.com/articles/article.aspx?p=2246943&utm_source=Python%20Weekly%20Newsletter&utm_campaign=ce99590cba-Python_Weekly_Issue_157_September_18_2014&utm_medium=email&utm_term=0_9e26887fc5-ce99590cba-312702461
  [3]: http://www.informit.com/articles/article.aspx?p=2175997
  [4]: http://www.informit.com/articles/article.aspx?p=2190334
  [5]: http://multicoding.blogspot.ie/
  [6]: http://docs.mongodb.org/manual/tutorial/getting-started/