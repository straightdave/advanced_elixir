4-GenEvent
============

[事件管理器]()  
[注册表进程的事件]()  
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

但是，当前还没有任何消息处理者绑定到该管理器，因此不管它发啥通知，叫破喉咙都不会有事儿发生。

现在就在iex对话里创建第一个事件处理器：
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
2. 定义事件处理者的模块（如这里的```Forwarder```）
3. 事件处理者的状态：在这里，使用当前进程的id

加上这个处理器之后，可以看到，调用了```sync_notify/2```之后，```Forwarder```处理器成功地把事件转给了它的父进程（IEx），因此那个消息进入了我们的收件箱。

这里有几点需要注意：  

1. 事件处理器运行在事件管理器的同一个进程里
2. ```sync_notify/2```同步地运行事件处理器处理请求
3. ```notify/2```使事件处理器异步处理请求

这里```sync_notify/2```和```notify/2```类似于GenServer里面的```call/2```和```cast/2```。推荐使用```sync_notify/2```。
它以反向压力的机制工作，减少了“发消息快过消息被分发到处理者手中”的可能性。

记得去[GenServer的模块文档](http://elixir-lang.org/docs/stable/elixir/GenEvent.html)阅读其它一些函数。
目前我们的程序就用提到的这些知识就可以了。

## 4.2-注册表进程的事件

为了能发出事件消息，我们要稍微修改一下我们的注册表进程，使之同一个事件管理器进行协作。
我们需要在注册表进程启动的时候，事件管理器也能自动启动。
比如在```init/1```回调里面，最好能传递事件处理器的pid或名字什么的作为参数来```start_link```，以此将启动事件管理器与注册表进程分解开。

让我们首先修改测试中注册表进程的行为。打开```test/kv/registry_text.exs```，修改下目前的```setup```回调，然后再加上新的测试：
```elixir
defmodule Forwarder do
  use GenEvent

  def handle_event(event, parent) do
    send parent, event
    {:ok, parent}
  end
end

setup do
  {:ok, manager} = GenEvent.start_link
  {:ok, registry} = KV.Registry.start_link(manager)

  GenEvent.add_mon_handler(manager, Forwarder, self())
  {:ok, registry: registry}
end

test "sends events on create and crash", %{registry: registry} do
  KV.Registry.create(registry, "shopping")
  {:ok, bucket} = KV.Registry.lookup(registry, "shopping")
  assert_receive {:create, "shopping", ^bucket}

  Agent.stop(bucket)
  assert_receive {:exit, "shopping", ^bucket}
end
```

为了测试我们即将添加的功能，我们首先定义了一个```Forwarder```事件处理器，类似刚才在IEx中创建的那样。
在```Setup```中，我们启动了事件管理器，把它作为参数传递给了注册表进程，并且向该管理器添加了我们定义的```Forwarder```处理器。
至此，事件可以发向待测进程了。

在测试中，我们创建、停止了一个bucket进程，并且使用```assert_receive```断言来检查是否收到了```:create```和```:exit```事件消息。
断言```assert_receive```默认是500毫秒超时时间，这对于测试足够了。
同样要指出的是，```assert_receive```期待接收一个模式，而不是一个值。
这就是为啥我们用```^bucket```来匹配bucket的pid（参考《入门》关于变量的匹配内容）。

最终，注意我们调用了```GenEvent.add_mon_handler/3```来代替```GenEvent.add_handler/3```。该函数不但可以添加一个处理器，它还告诉事件管理器来监视当前进程。如果当前进程挂了，事件处理器也一并抹去。
这个很有道理，因为对于这里的```Forwarder```，如果消息的接收方（```self()```/测试进程）终止，我们理所应当停止转发消息。

好了，现在来修改注册表进程代码来让测试pass。打开```lib/kv/registry.ex```，输入以下新的内容（一些关键语句的解释写在注释里）：
```elixir
defmodule KV.Registry do
  use GenServer

  ## Client API

  @doc """
  Starts the registry.
  """
  def start_link(event_manager, opts \\ []) do
    # 1. start_link now expects the event manager as argument
    GenServer.start_link(__MODULE__, event_manager, opts)
  end

  @doc """
  Looks up the bucket pid for `name` stored in `server`.

  Returns `{:ok, pid}` in case a bucket exists, `:error` otherwise.
  """
  def lookup(server, name) do
    GenServer.call(server, {:lookup, name})
  end

  @doc """
  Ensures there is a bucket associated with the given `name` in `server`.
  """
  def create(server, name) do
    GenServer.cast(server, {:create, name})
  end

  ## Server callbacks

  def init(events) do
    # 2. The init callback now receives the event manager.
    #    We have also changed the manager state from a tuple
    #    to a map, allowing us to add new fields in the future
    #    without needing to rewrite all callbacks.
    names = HashDict.new
    refs  = HashDict.new
    {:ok, %{names: names, refs: refs, events: events}}
  end

  def handle_call({:lookup, name}, _from, state) do
    {:reply, HashDict.fetch(state.names, name), state}
  end

  def handle_cast({:create, name}, state) do
    if HashDict.get(state.names, name) do
      {:noreply, state}
    else
      {:ok, pid} = KV.Bucket.start_link()
      ref = Process.monitor(pid)
      refs = HashDict.put(state.refs, ref, name)
      names = HashDict.put(state.names, name, pid)
      # 3. Push a notification to the event manager on create
      GenEvent.sync_notify(state.events, {:create, name, pid})
      {:noreply, %{state | names: names, refs: refs}}
    end
  end

  def handle_info({:DOWN, ref, :process, pid, _reason}, state) do
    {name, refs} = HashDict.pop(state.refs, ref)
    names = HashDict.delete(state.names, name)
    # 4. Push a notification to the event manager on exit
    GenEvent.sync_notify(state.events, {:exit, name, pid})
    {:noreply, %{state | names: names, refs: refs}}
  end

  def handle_info(_msg, state) do
    {:noreply, state}
  end
end
```

这些改变很直观。我们给```GenServer```初始化过程传递一个事件管理器，该管理器是我们用```start_link```启动进程时作为参数收到的。
我们还改了cast和info两个回调，在里面调用了```GenEvent.sync_notify/2```。
最后，我们借这个机会还把服务器的状态改成了一个图，方便我们以后改进注册表进程。

执行测试，都是绿的。

## 4.3-事件流

最后一个值得探索的```GenEvent```的功能点是像处理流一样处理事件：

```elixir
iex> {:ok, manager} = GenEvent.start_link
{:ok, #PID<0.83.0>}
iex> spawn_link fn ->
...>   for x <- GenEvent.stream(manager), do: IO.inspect(x)
...> end
:ok
iex> GenEvent.notify(manager, {:hello, :world})
{:hello, :world}
:ok
```

上面的例子中，我们创建了一个```GenEvent.stream(manager)```，返回一个事件的流（即一个enumerable），并随即处理了它。
处理事件是一个_阻塞_的行为，我们派生新进程来处理事件消息，把消息打印在终端上。这一系列的操作，就像看到的那样，如实地执行了。
每次调用```sync_notify/2```或者```notify/2```，事件都被打印在终端上，后面跟着一个```:ok```（IEx输出语句的执行结果）。

通常事件流提供了足够多的内置功能来处理事件，使我们不必实现我们自己的处理器。
但是，若是需要某些自定义的功能，或是在测试时，定义自己的事件处理器回调才是正道。

至此，我们有了一个事件处理器，一个注册表进程以及可能会同时执行的许多bucket进程，是时候开始担心这些进程会不会挂掉了。

