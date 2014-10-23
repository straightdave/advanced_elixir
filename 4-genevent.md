4-GenEvent
============

[事件管理器]()  
[注册事件]()  
[事件流]()  

本章探索GenEvent，Elixir和OTP提供的又一个行为抽象。它允许我们派生一个事件管理器，用来向多个处理者发布事件消息。

我们会激发两种事件：一个是每次bucket被加到注册表，另一个是从注册表中移除。

## 4.1-事件管理器
打开一个新```iex -S mix```对话，玩弄一下GenEvent的API：
```elixir
iex> {:ok, manager} = GenEvent.start_link
{:ok, #PID<0.83.0>}
iex> GenEvent.sync_notify(manager, :hello)
:ok
iex> GenEvent.notify(manager, :world)
:ok
```

函数```GenEvent.start_link/0```启动了一个新的事件管理器。不需额外的参数。
管理器创建好后，我们就可以调用```GenEvent.notify/2```函数和```GenEvent.sync_notify/2```函数来发送通知。

但是，当前还没有消息处理者绑定到该管理器，因此不管发啥通知叫破喉咙都不会有事儿发生。

现在，就在iex对话里，创建第一个事件处理器：
```elixir
iex> defmodule Forwarder do
...>   use GenEvent
...>   def handle_event(event, parent) do
...>     send parent, event
...>     {:ok, parent}
...>   end
...> end
iex> GenEvent.add_handler(manager, Forwarder, self())
:ok
iex> GenEvent.sync_notify(manager, {:hello, :world})
:ok
iex> flush
{:hello, :world}
:ok
```

我们创建了一个处理器，并通过函数```GenEvent.add_handler/3```把它“加入”到事件管理器上，传递的三个参数是：  

1. 刚启动的那个时间管理器
2. 定义事件处理者的模块（这里名字叫```Forwarder```）
3. 时间处理者的状态：在这里，即当前进程的id







