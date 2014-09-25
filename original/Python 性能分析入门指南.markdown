# Python 性能分析入门指南

标签（空格分隔）： Python  performance 性能分析

---

> 注： 本文的原作者是 [Huy Nguyen][1] ，原文地址为 [A guide to analyzing Python performance][2]

虽然并非你编写的每个 Python 程序都要求一个严格的性能分析，但是让人放心的是，当问题发生的时候，Python 生态圈有各种各样的工具可以处理这类问题。

分析程序的性能可以归结为回答四个基本问题：

 1. 正运行的多快
 2. 速度瓶颈在哪里
 3. 内存使用率是多少
 4. 内存泄露在哪里

下面，我们将用一些神奇的工具深入到这些问题的答案中去。

# 用 ```time``` 粗粒度的计算时间

 让我们开始通过使用一个快速和粗暴的方法计算我们的代码:传统的 unix ```time``` 工具。

```
 $ time python yourprogram.py
real    0m1.028s
user    0m0.001s
sys     0m0.003s
```
 
三个输出测量值之间的详细意义在这里 [stackoverflow article][3]，但简介在这：

 - real -- 指的是实际耗时
 - user -- 指的是内核之外的 CPU 耗时
 - sys  -- 指的是花费在内核特定函数的 CPU 耗时

你会有你的应用程序用完了多少 CPU 周期的即视感，不管系统上其他运行的程序添加的系统和用户时间。

如果 sys 和 user 时间之和小于 real 时间，然后你可以猜测到大多数程序的性能问题最有可能与 ```IO wait``` 相关。

# 用 ```timing context``` 管理器细粒度的计算时间

我们下一步的技术包括直接嵌入代码来获取细粒度的计时信息。下面是我进行时间测量的代码的一个小片段

timer.py

```
import time

class Timer(object):
    def __init__(self, verbose=False):
        self.verbose = verbose

    def __enter__(self):
        self.start = time.time()
        return self

    def __exit__(self, *args):
        self.end = time.time()
        self.secs = self.end - self.start
        self.msecs = self.secs * 1000  # millisecs
        if self.verbose:
            print 'elapsed time: %f ms' % self.msecs
```

为了使用它，使用 Python 的 ```with``` 关键字和 ```Timer ``` 上下文管理器来包装你想计算的代码。当您的代码块开始执行，它将照顾启动计时器,当你的代码块结束的时候，它将停止计时器。

这个代码片段示例：

```
from timer import Timer
from redis import Redis
rdb = Redis()

with Timer() as t:
    rdb.lpush("foo", "bar")
print "=> elasped lpush: %s s" % t.secs

with Timer() as t:
    rdb.lpop("foo")
print "=> elasped lpop: %s s" % t.secs
```

为了看看我的程序的性能随着时间的演化的趋势，我常常记录这些定时器的输出到一个文件中。

# 使用 ```profiler``` 逐行计时和分析执行的频率

罗伯特·克恩有一个不错的项目称为 [line_profiler][4] , 我经常使用它来分析我的脚本有多快，以及每行代码执行的频率：

为了使用它，你可以通过使用 ```pip``` 来安装它：

```
pip install line_profiler
```

安装完成后，你将获得一个新模块称为 ```line_profiler``` 和 ```kernprof.py``` 可执行脚本。

为了使用这个工具，首先在你想测量的函数上设置 ```@profile``` 修饰符。不用担心，为了这个修饰符，你不需要引入任何东西。```kernprof.py``` 脚本会在运行时自动注入你的脚本。

primes.py

```
@profile
def primes(n): 
    if n==2:
        return [2]
    elif n<2:
        return []
    s=range(3,n+1,2)
    mroot = n ** 0.5
    half=(n+1)/2-1
    i=0
    m=3
    while m <= mroot:
        if s[i]:
            j=(m*m-3)/2
            s[j]=0
            while j<half:
                s[j]=0
                j+=m
        i=i+1
        m=2*i+3
    return [2]+[x for x in s if x]
primes(100)
```

一旦你得到了你的设置了修饰符  ```@profile``` 的代码，使用 ```kernprof.py``` 运行这个脚本。

```
kernprof.py -l -v fib.py
```

```-l``` 选项告诉 ```kernprof ``` 把修饰符 ```@profile``` 注入你的脚本，```-v``` 选项告诉 ```kernprof ``` 一旦你的脚本完成后，展示计时信息。这是一个以上脚本的类似输出：

```
Wrote profile results to primes.py.lprof
Timer unit: 1e-06 s

File: primes.py
Function: primes at line 2
Total time: 0.00019 s

Line #      Hits         Time  Per Hit   % Time  Line Contents
==============================================================
     2                                           @profile
     3                                           def primes(n): 
     4         1            2      2.0      1.1      if n==2:
     5                                                   return [2]
     6         1            1      1.0      0.5      elif n<2:
     7                                                   return []
     8         1            4      4.0      2.1      s=range(3,n+1,2)
     9         1           10     10.0      5.3      mroot = n ** 0.5
    10         1            2      2.0      1.1      half=(n+1)/2-1
    11         1            1      1.0      0.5      i=0
    12         1            1      1.0      0.5      m=3
    13         5            7      1.4      3.7      while m <= mroot:
    14         4            4      1.0      2.1          if s[i]:
    15         3            4      1.3      2.1              j=(m*m-3)/2
    16         3            4      1.3      2.1              s[j]=0
    17        31           31      1.0     16.3              while j<half:
    18        28           28      1.0     14.7                  s[j]=0
    19        28           29      1.0     15.3                  j+=m
    20         4            4      1.0      2.1          i=i+1
    21         4            4      1.0      2.1          m=2*i+3
    22        50           54      1.1     28.4      return [2]+[x for x
```
寻找 ```hits ``` 值比较高的行或是一个高时间间隔。这些地方有最大的优化改进空间。

# 它使用了多少内存？

现在我们掌握了很好我们代码的计时信息，让我们继续找出我们的程序使用了多少内存。我们真是非常幸运， Fabian Pedregosa  仿照 Robert Kern 的 ```line_profiler``` 实现了一个很好的内存分析器 ```[memory profiler][5]```。

首先通过 pip 安装它：

```
$ pip install -U memory_profiler
$ pip install psutil
```

在这里建议安装 ```psutil``` 是因为该包能提升 ```memory_profiler``` 的性能。

想  ```line_profiler``` 一样， ```memory_profiler``` 要求在你设置 ```@profile``` 来修饰你的函数：

```
@profile
def primes(n): 
    ...
    ...
```

运行如下命令来显示你的函数使用了多少内存：

```
$ python -m memory_profiler primes.py
```

一旦你的程序退出，你应该可以看到这样的输出：

```
Filename: primes.py

Line #    Mem usage  Increment   Line Contents
==============================================
     2                           @profile
     3    7.9219 MB  0.0000 MB   def primes(n): 
     4    7.9219 MB  0.0000 MB       if n==2:
     5                                   return [2]
     6    7.9219 MB  0.0000 MB       elif n<2:
     7                                   return []
     8    7.9219 MB  0.0000 MB       s=range(3,n+1,2)
     9    7.9258 MB  0.0039 MB       mroot = n ** 0.5
    10    7.9258 MB  0.0000 MB       half=(n+1)/2-1
    11    7.9258 MB  0.0000 MB       i=0
    12    7.9258 MB  0.0000 MB       m=3
    13    7.9297 MB  0.0039 MB       while m <= mroot:
    14    7.9297 MB  0.0000 MB           if s[i]:
    15    7.9297 MB  0.0000 MB               j=(m*m-3)/2
    16    7.9258 MB -0.0039 MB               s[j]=0
    17    7.9297 MB  0.0039 MB               while j<half:
    18    7.9297 MB  0.0000 MB                   s[j]=0
    19    7.9297 MB  0.0000 MB                   j+=m
    20    7.9297 MB  0.0000 MB           i=i+1
    21    7.9297 MB  0.0000 MB           m=2*i+3
    22    7.9297 MB  0.0000 MB       return [2]+[x for x in s if x]

```



# **```line_profiler```** 和 **```memory_profiler```** 的 IPython 快捷命令

```line_profiler``` 和 ```memory_profiler``` 一个鲜为人知的特性就是在 IPython 上都有快捷命令。你所能做的就是在 IPython 上键入以下命令：

```
%load_ext memory_profiler
%load_ext line_profiler
```

这样做了以后，你就可以使用魔法命令 ```%lprun``` 和 ```%mprun``` 了，它们表现的像它们命令行的副本，最主要的不同就是你不需要给你需要分析的函数设置 ``` @profile``` 修饰符。直接在你的 IPython 会话上继续分析吧。

```
In [1]: from primes import primes
In [2]: %mprun -f primes primes(1000)
In [3]: %lprun -f primes primes(1000)
```

这可以节省你大量的时间和精力,因为使用这些分析命令，你不需要修改你的源代码。

# 哪里内存溢出了？

cPython的解释器使用引用计数来作为它跟踪内存的主要方法。这意味着每个对象持有一个计数器，当增加某个对象的引用存储的时候，计数器就会增加，当一个引用被删除的时候，计数器就是减少。当计数器达到0， cPython 解释器就知道该对象不再使用，因此解释器将删除这个对象，并且释放该对象持有的内存。

内存泄漏往往发生在即使该对象不再使用的时候，你的程序还持有对该对象的引用。

最快速发现内存泄漏的方式就是使用一个由  Marius Gedminas 编写的非常好的称为 ```[objgraph][6]``` 的工具。
这个工具可以让你看到在内存中对象的数量,也定位在代码中所有不同的地方,对这些对象的引用。

开始，我们首先安装 ```objgraph```

```
pip install objgraph
```

一旦你安装了这个工具，在你的代码中插入一个调用调试器的声明。

```
import pdb; pdb.set_trace()
```

## 哪个对象最常见

在运行时,你可以检查在运行在你的程序中的前20名最普遍的对象

```
pdb) import objgraph
(pdb) objgraph.show_most_common_types()

MyBigFatObject             20000
tuple                      16938
function                   4310
dict                       2790
wrapper_descriptor         1181
builtin_function_or_method 934
weakref                    764
list                       634
method_descriptor          507
getset_descriptor          451
type                       439
```


## 哪个对象被增加或是删除了？

我们能在两个时间点之间看到哪些对象被增加或是删除了。

```
(pdb) import objgraph
(pdb) objgraph.show_growth()
.
.
.
(pdb) objgraph.show_growth()   # this only shows objects that has been added or deleted since last show_growth() call

traceback                4        +2
KeyboardInterrupt        1        +1
frame                   24        +1
list                   667        +1
tuple                16969        +1
```


## 这个泄漏对象的引用是什么？

继续下去，我们还可以看到任何给定对象的引用在什么地方。让我们以下面这个简单的程序举个例子。

```
x = [1]
y = [x, [x], {"a":x}]
import pdb; pdb.set_trace()
```
为了看到持有变量 X 的引用是什么，运行 ```objgraph.show_backref() ``` 函数：
```
(pdb) import objgraph
(pdb) objgraph.show_backref([x], filename="/tmp/backrefs.png")
```

该命令的输出是一个 PNG 图片，被存储在 ```/tmp/backrefs.png```，它应该看起来像这样：

![此处输入图片的描述][7]

盒子底部有红色字体就是我们感兴趣的对象，我们可以看到它被符号 x 引用了一次，被列表 y 引用了三次。如果 x 这个对象引起了内存泄漏，我们可以使用这种方法来追踪它的所有引用，以便看到为什么它没有被自动被收回。

回顾一遍，objgraph 允许我们：

 - 显示占用 Python 程序内存的前 N 个对象
 - 显示在一段时期内哪些对象被增加了，哪些对象被删除了
 - 显示我们脚本中获得的所有引用

# Effort vs precision

在这篇文章中,我展示了如何使用一些工具来分析一个python程序的性能。通过这些工具和技术的武装，你应该可以获取所有要求追踪大多数内存泄漏以及在Python程序快速识别瓶颈的信息。


和许多其他主题一样,运行性能分析意味着要在付出和精度之间的平衡做取舍。当有疑问是，用最简单的方案，满足你当前的需求。

相关阅读：

- [stack overflow - time explained][8]
- [line_profiler][9]
- [memory_profiler][10]
- [objgraph][11]
 


  [1]: http://www.huyng.com/
  [2]: http://www.huyng.com/posts/python-performance-analysis/
  [3]: http://stackoverflow.com/questions/556405/what-do-real-user-and-sys-mean-in-the-output-of-time1
  [4]: http://packages.python.org/line_profiler/
  [5]: https://github.com/fabianp/memory_profiler
  [6]: http://mg.pov.lt/objgraph/
  [7]: http://www.huyng.com/media/3011/backrefs.png
  [8]: http://stackoverflow.com/questions/556405/what-do-real-user-and-sys-mean-in-the-output-of-time1
  [9]: http://packages.python.org/line_profiler/
  [10]: https://github.com/fabianp/memory_profiler
  [11]: http://mg.pov.lt/objgraph/