# GO 开发者对 GO 初学者的建议

标签（空格分隔）： GO golang

---

> 注：原文地址为 [Advise from Go developers to Go programming newbies][1]

以促进 India 的 go 编程作为 GopherConIndia 承诺的一部分。我们采访了 [40 位 Gophers][2]（一个 Gopher 代表一个 GO 项目或是任何地方的 GO 程序员），得到了他们关于 GO 的意见。从 2014 年的八月到十一月，我们将每个星期发表两篇采访稿。


如果你正好刚刚开始 go 编程，他们对于我们一些问题的答案可能会对你有非常有用。看看这些。

应该做：

 - 通读 [the Go standard library][3] 和 [Effective Go][4]，为了学习 GO 的规范，Effective Go 是被高度推荐的，尤其是如果你有其他语言的背景。
 - 在 [Go tour][5] 上做练习
 - 看完[语言参考][6]
 - 练习 [Go by Example][7]，只要你不仅仅是复制粘贴！
 - 坚持编写 GO 代码，在几周内你将会在这门语言上变得高效
 - 理解接口的功能，他们是 GO 最大的礼物之一，可能比 channels 和 goroutines 还重要。这个关于接口的文章 [article on interfaces][8] 和 Andrew Gerrand 在 GopherCon 2014 上的 keynote [接口的描述][9] 会对你非常有帮助。
 - 抛弃你的 OO 的思想包袱，如果你来自于其他语言，比如动态语言 Python 或是 Ruby，或者是一个编译型语言如 Java 或 C#。GO 是一个面向对象的语言，但是它不是一个基于 class 的语言和不支持继承。
 - 了解继承从 GO 语言中移除了。实践组合的用法而不是继承的机会显现了，并且纠结于继承只会导致你沮丧
 - 不要以其他语言的风格编写 GO
 - 寻找更加有经验的 Gophers，他们能帮助你 review 代码片段和给你反馈。在 GO 社区能得到真正的支持和帮助
 - 用 GO 实现你想法中的一个项目或是找到一个项目来工作。然后随着你学习的更多，不断重构你的应用。利用[邮件列表][10]和参加 [Gopher Academy Slack group][11] 来见其他的 Gophers 来得到帮助。[Dave Cheney][12] 的博客和 [GoingGo][13] 的博客也是一个非常好的开始
 - 不要等待泛型和函数式被添加进语言；屏住呼吸并学习爱上我们在今天拥有的这门语言

> 注：私人添加，可以订阅 Newspaper.io 的 Golang Daily，以及 [@ASTA谢][14] 的 [《Go Web 编程》][15] 【作者也出了实体书，大家可以购买】和 订阅 Golang Ask News，社区 http://golanghome.com/，[@无闻Unknown][16] [《Go编程基础》][17]，[《Go Web基础》][18] 和 [《Go名库讲解》][19]

**给 go 初学者分享的一些问题**

 - 对于任何人来说学习一门新语言可能都是令人挫折的。GO 社区是不可置信的活跃，你不是孤单的。利用所有的文档，博客，本地的 Meetups 和用户组，比如 Slack。不要害怕问问题和参与
 - 如果你对 GO 感兴趣，使用它的一侧涉足，或是专业的使用它，如果本地有 Go meetup，考虑参与。如果你有货，考虑去分享它
 - 如果你有计划旅行，并且有能力，努力去访问 GO 社区目的地
 - 来访的用户群体是个证明这个社区有众多的用户，支持者和雇员的途径
 - 不要浪费时间去和其他语言比较，如果你喜欢 GO，就爱上他并且去使用它
 - 接受 Go 的文化和 GO 做事情的方式。你的代码会感谢你，如果你这样做了，你会得到很多
 - 不要冲动的引入依赖
 - 简单是 GO 最重要的特征。避免过分设计，使用简单的代码片段而不是单一的庞大的代码库
 - 从其他语言移植库到 GO 是一个很好的做法，它允许你剥离他人的代码并且以符合 GO 语言的方式粘合起来。


> 注：How do you see the market for Go Programmers in the work place? What is the future for Go 这部分不翻译，请读者自己看



 


  [1]: http://www.gophercon.in/blog/2014/08/23/adviceforgonewbies/
  [2]: http://list.ly/list/Pak-gopher-interviews
  [3]: http://golang.org/pkg/
  [4]: http://golang.org/doc/effective_go.html
  [5]: http://tour.golang.org/#1
  [6]: https://golang.org/ref/spec
  [7]: https://gobyexample.com/
  [8]: http://mwholt.blogspot.in/2014/08/maximizing-use-of-interfaces-in-go.html
  [9]: http://talks.golang.org/2014/go4gophers.slide#5
  [10]: https://groups.google.com/forum/#!forum/golang-nuts
  [11]: https://gophers.slack.com/
  [12]: http://dave.cheney.net/
  [13]: http://www.goinggo.net/
  [14]: http://weibo.com/533452688
  [15]: https://github.com/astaxie/build-web-application-with-golang
  [16]: http://weibo.com/Obahua
  [17]: https://github.com/Unknwon/go-fundamental-programming
  [18]: https://github.com/Unknwon/go-web-foundation
  [19]: https://github.com/Unknwon/go-rock-libraries-showcases