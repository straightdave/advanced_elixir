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
用```@manager_name```和```@registry_name```这两个模块属性给我们监督者的俩孩子声明名字，然后在“工作者（worker）”的定义中引用这两个属性值。尽管不是必须用模块属性来定义名字，但很实用，因为这样读起来很醒目。














