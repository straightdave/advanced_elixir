2-Agent
========
[状态的问题]()  
[Agents]()  
[ExUnit回调函数]()  
[其它Agent行为]()  
[Agent中的C/S模式]()  

本章我们将创建一个名为```KV.Bucket```的模块。这个模块负责存储在某种程度上可以被不同进程读写的键值对。

如果你跳过了“入门”手册，或者是太久以前读的，那么建议你最好重新阅读一下关于**进程**的那一章。它是本节所描述内容的起点。

## 2.1-状态的问题
Elixir是一种“（变量值）不可变”的语言，默认情况下，没有什么是被共享的。若要创建bucket，从不同的地方存储和访问它们，我们有两种选择：
* 进程
* ETS（[Erlang Term Storage](http://sdbranch/Tree/Tree.aspx)）

我们之前介绍过进程，但ETS是个新东西，在之后的章节中再去探讨。这里，当讨论进程时，我们很少会去自己动手处理我们需要的进程，而是用Elixir和OTP中抽象出来的东西代替：  
* [Agent](http://elixir-lang.org/docs/stable/elixir/Agent.html) - 对状态简单的封装
* [GenServer](http://elixir-lang.org/docs/stable/elixir/GenServer.html) - “通用服务器”（本身是个进程）的意思。它封装了状态，提供了同步或异步调用，支持代码重新加载等等
* [GenEvent](http://elixir-lang.org/docs/stable/elixir/GenEvent.html) - “通用事件”管理器，允许向多个接收者发布事件消息
* [Task](http://elixir-lang.org/docs/stable/elixir/Task.html) - 计算处理的异步单元，可以派生出进程并稍后收集计算结果

我们在本“进阶”手册中会逐一讨论这些抽象物。记住它们都是在进程基础上实现的，使用了虚拟机提供的基本功能如```send```，```receive```，```spawn```和```link```。

## 2.2-Agents
[Agent](http://elixir-lang.org/docs/stable/elixir/Agent.html)是对状态简单的封装。
