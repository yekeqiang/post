# Go 有 make 和 new 两个函数，怎么用？

标签（空格分隔）： Go make new

---

> 注：该文作者是 [Dave Cheney][1]，原文地址 [Go has both make and new functions, what gives ?][2]

这篇博客主要是讨论 Go 的内建函数 `make` 和 `new`。 

如 Rob Pike 在今年的 Gophercon 上提到的，Go 有多种方式初始化变量。其中之一的能力就是获取一个 struct literal 的地址，这导致了几种方式做同样的事情。

```
s := &SomeStruct{}
v := SomeStruct{}
s := &v              // identical
s := new(SomeStruct) // also identical
```

评论家指出，这种语言中的冗余是公平的，这有时会导致他们寻找其他的矛盾， `make` 和 `new` 之间的冗余最明显。

表面上 `make` 和 `new` 做着相同的事情，那究竟是什么原因两个都有呢？ 

## 为什么不可以使用 make 做一切事情？

Go 没有用户定义的泛型，但是它有一些内建的类型，可以操作如通用的  lists，maps， sets， 和 queues；  slices， maps 和 channels。

因为 `make` 是被设计成创建这三个内建的泛型，它必须由运行时提供，因为在 Go 中没有办法直接表达 `make` 的函数签名。

尽管 `make` 创建了通用的  slice，map， 和 channel 值，他们依然只是常规的值；`make` 不会返回指针类型的值。

如果 `new` 被移除了，支持 `make`，你将如何构建一个指针初始化值。

```
var x1 *int
var x2 = new(int)
```

x1 和 x2 有相同的类型，*int， x2 指向初始化的内存并且可以安全的解除引用。对于 x1 ，同样不是真的。

## 为什么不可以使用 new 做一切事情？

尽管 `new` 的使用是稀少的，其行为是指定好的。


`new(T)` 一直返回一个 `*T` 指向一个初始化的 `T`，正如 Go 没有构造器，这值将被初始化成 `T` 的 [zero value][3]（讲解了为什么值是零，以及它为什么有用）。

使用 `new` 来构造一个指向一个 slice，map，或是 channel zero 值的指针并和 `new` 的行为一致。

```
s := new([]string)
fmt.Println(len(*s))  // 0
fmt.Println(*s == nil) // true

m := new(map[string]int)
fmt.Println(m == nil) // false
fmt.Println(*m == nil) // true

c := new(chan int)
fmt.Println(c == nil) // false
fmt.Println(*c == nil) // true
```

## 确实，但这些只是规则，我们可以改变它们，对吧？

它们可能会导致混乱，`make` 和 `new` 是一致的；`make` 仅仅 make slices， maps， 和 channels，`new` 仅仅返回初始化内存的指针。

是的， `new` 可以被扩展成像 `make` 样操作 slices， maps 和 channels，但这将会导致它自己的不一致性。

 1. `new` 将有特殊的表现，如果传递给 `new` 的类型是 slices， maps 和 channels。这是每一个 Go 程序员应该记得的。
 2. 对于 slices 和 channels，`new` 将变成可变参数，采取可能的长度，buffer size 或是 capacity。在比较特殊的情况下一定要记住，然后之前 `new` 只有一个参数，一个类型。
 3. `new` 一直返回一个 *T 的 T 传递给它，这意味着像代码：
 
    ```
    func Read(buf []byte) []byte
    // assume new takes an optional length
    buf := Read(new([]byte, 4096))
    ```
 
    需要语法更特殊的情况下允许 *new([]byte, length) 将变得不再可能。
    

## 总结

`make` 和 `new` 做不同的事情。

如果你从其他语言转移过来，特别是有构造器的语言，那 `new` 应该就是你需要的。但是 Go 不是这些语言，它没有构造器。

我的建议的谨慎的使用 `new`。几乎总是有更容易和更清洁的方式来[写你的程序][4]。


作为一个代码评审者，使用 `new`，就像使用命名返回参数，这是一个信号，表明代码正在尝试更聪明的做一些事情，我需要特别注意。可能是代码真的是聪明,但更有可能，它可以被更清晰和更地道的重写。


**涉及的文章**：

 1. [What is the zero value, and why is it useful ?][5]
 2. [The empty struct][6]
 3. [Pointers in Go][7]
 4. [Channel Axioms][8]
    
 


  [1]: http://dave.cheney.net/
  [2]: http://dave.cheney.net/2014/08/17/go-has-both-make-and-new-functions-what-gives
  [3]: http://dave.cheney.net/2013/01/19/what-is-the-zero-value-and-why-is-it-useful
  [4]: http://dave.cheney.net/2014/05/24/on-declaring-variables
  [5]: http://dave.cheney.net/2013/01/19/what-is-the-zero-value-and-why-is-it-useful
  [6]: http://dave.cheney.net/2014/03/25/the-empty-struct
  [7]: http://dave.cheney.net/2014/03/17/pointers-in-go
  [8]: http://dave.cheney.net/2014/03/19/channel-axioms