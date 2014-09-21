# 怎样确定 Web 应用程序的线程池大小

标签（空格分隔）： Thread Pool Web 

---

> 本文原文是 [How To Determine Web Application Thread Pool Size][1]


继续[当扩展 Web 应用程序时面临的架构问题][2]，在这篇博客中，我将介绍一个常见的问题，怎样确定 Web 应用程序的线程池大小？当部署 Web 应用程序到生产或是当 Web 应用程序性能测试的时候会显示出来。

## 线程池

在 Web 应用程序中，线程池大小决定了它们能在任何时候处理的并发请求数。如果一个 Web 应用程序得到的请求比线程池多，多余的请求要么排队要么被拒绝。


请注意并发并不是并行。并发请求是在任何时间点只有少数请求可以运行在 CPU 上正在被处理的请求数。并行请求是在任何时间点所有的请求都可以运行在 CPU 上的正在处理的请求数。

在非阻塞 I/O 应用程序中，比如 NodeJS，一个单进程（线程）能处理并发的处理多个请求。在多核 CPU 环境，可以通过增加进程或线程数来处理并行请求。

在阻塞 I/O 应用程序中，比如 Java SpringMVC，一个单进程仅仅可以并发的处理一个请求。为了并发的处理更多的请求，我们不得不增加线程数。

## CPU 限制的应用程序

CPU 限制的应用程序，线程池的大小应该等于系统里面 CPU 的数量。增加更多的线程将中断请求处理，由于线程上下文切换，这样也会增加响应时间。

非 I/O 阻塞的应用程序将被 CPU 限制，因为它们没有线程等待时间，当请求得到处理的时候。

## I/O 限制的应用程序

确定 I/O 限制应用程序的线程池的大小是复杂的多，并依赖于下游系统的响应时间，因为一个线程会被阻塞直到其他系统响应了。我们将不得不增加线程的数量,以便更好地利用 CPU，正如在 [Reactor Pattern Part 1 : Applications with Blocking I/O][3] 中的讨论。


## 利特尔法则

利特尔法则被用于非技术领域，比如银行需要计算出处理进入银行的客户所需要的出纳台的数量。

[Little’s law][4]

```
The long-term average number of customers in a stable system L is equal to the long-term 
average effective arrival rate, λ, multiplied by the average time a customer spends 
in the system, W; or expressed algebraically: 
L = λW.
```

利特尔法则应用于 Web 应用程序。

```
在一个系统中的平均线程数量等于平均的 web 请求到达率（每秒的 web 请求数），乘以平均响应时间（ResponseTime）。
```

线程 = 线程数
每秒 web 请求数 = 在 1s 内能被处理的 web 请求数
响应时间 = 处理一个 web 请求所花费的时间

```
线程 = (每秒 web 请求数) X 响应时间
```

当以上的方程式提供了需要的线程数来处理传入的请求，它没有提供线程 CPU 利用率的信息。比如，多少个线程应该被分配在给定系统的 X CPU。

## 测试确定线程池大小

为了找出吞吐量和响应时间之间最佳平衡的线程池大小。每个 CPU 最小线程启动开始（Threads Pool Size = No of CPUs），应用程序线程池大小直接与下游系统的平均响应时间成正比，直到 CPU 使用率爆了，或是响应时间降级。

下面的图说明了多少个请求数，CPU 和连接响应时间指标。

CPU Vs 请求数图表显示了增加 Web 应用程序负载的时候，CPU 的利用率。

响应时间 Vs 请求数图表显示了由于增加 Web 应用程序的负载，对响应时间的影响。

绿点表示最佳吞吐量和响应时间。

**线程池大小 = CPU 数量**

![此处输入图片的描述][5]

以上图表描述了阻塞 I/O 限制应用程序，当线程数等于 CPU 数时。应用程序线程被阻塞等待下游系统响应。因为线程被阻塞，响应时间随着请求进入队列增加。即使 CPU 利用率非常低，随着所有的请求被阻塞，应用程序开始拒绝请求。

**大线程池**

![此处输入图片的描述][6]

以上图表描述了当大量的线程在 Web 应用程序中被创建时，阻塞 I/O 限制了应用程序。由于大量的线程，线程切换将非常频繁。应用程序的 CPU 使用率会爆满，即使吞吐量没有增加，由于不必要的线程上下文切换。因为上下文切换造成的请求被中断，响应时间降级。

**最佳线程池大小**

![此处输入图片的描述][7]


以上图表描述了当最佳线程池大小在 Web 应用程序中被创建的时，阻塞 I/O 限制了应用程序。CPU 得到有效使用，并由良好的吞吐量和更少的线程上下文切换。我们注意到响应时间是适合由于请求被高效率的处理（很少的中断）。

## 线程池隔离

在大部分 Web 应用程序中，一些类型的 web 请求相对于其他类型的 web 请求需要更长的时间来处理。最慢的请求将夯住所有的请求，并降低整个应用程序的性能。

两种方法来处理这个问题：

 - 有单独的系统来处理缓慢的Web请求
 - 在同一应用程序中分配给慢的 web 请求一个单独的线程池

确定一个阻塞 I/O Web 应用程序的最佳线程池大小是一个非常困难的任务。通常通过进行一些性能运行完成。有几个线程池在 Web 应用程序中会更加复杂化确定最佳线程池大小的过程。
 
  [1]: http://venkateshcm.com/2014/05/How-To-Determine-Web-Applications-Thread-Poll-Size/
  [2]: http://venkateshcm.com/2014/05/Architecture-Issues-Scaling-Web-Applications/
  [3]: http://venkateshcm.com/2014/04/Reactor-Pattern-Part-1-Non-blocking-I-O/
  [4]: http://en.wikipedia.org/wiki/Little%27s_law
  [5]: http://venkateshcm.com/img/blog/MinimumThreads.png
  [6]: http://venkateshcm.com/img/blog/MaximumThreads.png
  [7]: http://venkateshcm.com/img/blog/OptimalThreads.png