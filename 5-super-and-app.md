5-监督者和应用程序
======================

- [第一个监督者]()  
- [理解应用程序]()  
  - [启动应用程序]()  
  - [应用程序的回调函数]()  
  - [工程还是应用程序？]()  
- [简单的监督者]()  
- [监督树]()  

到目前为止，我们的程序需要一个事件管理器和一个注册表进程。它还会有不是成百，就是上千的bucket进程。
你是不是觉得这个还不错？没有东西是完美的，也许马上就要出现bug了。

当有东西挂了，我们的第一反应是：“快拯救这些错误”。但是，像在《入门》中学到的那样，不同于其它多数语言，Elixir不太做“防御性编程”。
相反，我们说“要挂快点挂”，或是“就让它挂”。
如果有bug要让我们的注册表进程挂掉，啥也不怕，因为我们即将用监督者启动一个新的注册表进程。

本章，我们就要学习监督者，还会讲到应用程序。一个不够，我们要创建两个监督者，用它们监督我们的进程。

## 5.1-第一个监督者
创建一个监督者跟创建GenServer比没多少不同。
定义一个名为```KV.Supervisor```的模块，这个模块使用[Supervisor](http://elixir-lang.org/docs/stable/elixir/Supervisor.html)行为抽象。此文件```lib/kv/supervisor.ex```的内容：
```elixir
defmodule KV.Supervisor do
  use Supervisor

  def start_link do
    Supervisor.start_link(__MODULE__, :ok)
  end

  @manager_name KV.EventManager
  @registry_name KV.Registry

  def init(:ok) do
    children = [
      worker(GenEvent, [[name: @manager_name]]),
      worker(KV.Registry, [@manager_name, [name: @registry_name]])
    ]

    supervise(children, strategy: :one_for_one)
  end
end
```

我们的监督者有两个孩子：事件管理器和注册表进程。通常会给监督者旗下的进程起个名字，好让别的进程用名字而不是pid访问它们。
这很有用，因为一个被监督的进程可能会挂，一挂再重启，pid就变了。  
用```@manager_name```和```@registry_name```这两个模块属性给我们监督者的俩孩子声明名字，然后在“工人（worker）”的定义中引用这两个属性值。尽管不是必须用模块属性来定义名字，但很实用，因为这样读起来很醒目。

举个例子，```KV.Registry```工人接受两个参数，第一个是事件管理器的名字，第二个是一个选项键值列表。
在这里，我们设置一个名字选项```[name: KV.Registry]```（使用之前定义的模块属性：```@registry_name```），确保我们可以在整个应用程序中通过名字```KV.Registry```访问注册表进程。用定义它们的模块名称给监督者孩子起名的做法十分普遍，因为在调试一个正则运行的系统时很有用。

监督者中孩子们声明的顺序也是有区别的。因为注册表依赖于事件管理器，我们必须先启动前者。这也是在孩子列表中，```GenEvent```工人的位置靠前的原因。

最后，我们调用了```supervisor/2```，给它传递了一个孩子列表，以及策略：```:one_for_one```。

监督的策略指明了当一个孩子进程挂了会发生什么。```:one_for_one```意思是如果一个孩子进程挂了，只有一个它的“复制品”会启动来替代它。
这个策略现在是说得通的。如果事件管理器挂了，没理由连注册表进程也重启。反之亦然。
但是如果监督者旗下的孩子越来越多时，这个策略就需要改变了。```Supervisor```行为抽象支持许多不同的策略，我们在本章中将会讨论其中三种。

如果我们在工程中启动命令行对话```iex -S mix```，我们可以手动启动监督者：
```elixir
iex> KV.Supervisor.start_link
{:ok, #PID<0.66.0>}
iex> KV.Registry.create(KV.Registry, "shopping")
:ok
iex> KV.Registry.lookup(KV.Registry, "shopping")
{:ok, #PID<0.70.0>}
```

当我们启动监督树，事件管理器和注册表进程都自动被启动，允许我们创建bucket。不再需要手动启动它们。

尽管在实战中，我们很少手动启动应用程序的监督者。相反，它的启动是应用程序回调的一部分。

## 5.2-理解应用程序




















