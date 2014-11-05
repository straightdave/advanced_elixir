6-ETS
======
[ETS当缓存用]()  
[竞争条件？]()  
[ETS当持久存储用]()  

每次我们要找一个bucket时，都要发消息给注册表进程。在某些情况下，这意味着注册表进程会变成性能瓶颈！

本章我们将学习ETS（Erlang Term Storage），以及如何把它当成缓存使用。
之后我们会拓展它的功能，把数据从监督者保存到其孩子上。这样即使崩溃，数据也能存续。

>严重注意！绝对不要冒失地把ETS当缓存用。仔细分析你的程序，看看到底哪里才是瓶颈。这样来决定是否需要缓存以及缓存什么。
本章仅仅讲解ETS是如何工作的一个例子，具体怎么做得由你自己决定。

## 6.1-ETS当缓存用

ETS可以把Erlang/Elixir的词语（term）存储在内存表中。
使用[Erlang的```:ets```模块](http://www.erlang.org/doc/man/ets.html)来操作：
```elixir
iex> table = :ets.new(:buckets_registry, [:set, :protected])
8207
iex> :ets.insert(table, {"foo", self})
true
iex> :ets.lookup(table, "foo")
[{"foo", #PID<0.41.0>}]
```

在创建一个ETS表时，需要两个参数：表名和一组选项。对于在上面的例子，在可选的选项中我们传递了表类型和访问规则。
我们选择了```:set```类型，意思是键不能有重复（集合论）。
我们选择的访问规则是```:protected```，意思是对于这个表，只有创建该表的进程可以修改，而其它进程只能读取。
这两个选项是默认的，这里就不多说了。

ETS表可以被命名，可以通过名字访问：
```elixir
iex> :ets.new(:buckets_registry, [:named_table])
:buckets_registry
iex> :ets.insert(:buckets_registry, {"foo", self})
true
iex> :ets.lookup(:buckets_registry, "foo")
[{"foo", #PID<0.41.0>}]
```

好了，现在我们使用ETS表，修改```KV.Registry```。
我们对事件管理器和bucket的监督者使用相同的技术，显式传递ETS表名给```start_link```。
记住，有了服务器以及ETS表的名字，本地进程就可以访问那个表。

打开```lib/kv/registry.ex```，修改里面的实现。加上注释来标明我们的修改：
```elixir
defmodule KV.Registry do
  use GenServer

  ## Client API

  @doc """
  Starts the registry.
  """
  def start_link(table, event_manager, buckets, opts \\ []) do
    # 1. We now expect the table as argument and pass it to the server
    GenServer.start_link(__MODULE__, {table, event_manager, buckets}, opts)
  end

  @doc """
  Looks up the bucket pid for `name` stored in `table`.

  Returns `{:ok, pid}` if a bucket exists, `:error` otherwise.
  """
  def lookup(table, name) do
    # 2. lookup now expects a table and looks directly into ETS.
    #    No request is sent to the server.
    case :ets.lookup(table, name) do
      [{^name, bucket}] -> {:ok, bucket}
      [] -> :error
    end
  end

  @doc """
  Ensures there is a bucket associated with the given `name` in `server`.
  """
  def create(server, name) do
    GenServer.cast(server, {:create, name})
  end

  ## Server callbacks

  def init({table, events, buckets}) do
    # 3. We have replaced the names HashDict by the ETS table
    ets  = :ets.new(table, [:named_table, read_concurrency: true])
    refs = HashDict.new
    {:ok, %{names: ets, refs: refs, events: events, buckets: buckets}}
  end

  # 4. The previous handle_call callback for lookup was removed

  def handle_cast({:create, name}, state) do
    # 5. Read and write to the ETS table instead of the HashDict
    case lookup(state.names, name) do
      {:ok, _pid} ->
        {:noreply, state}
      :error ->
        {:ok, pid} = KV.Bucket.Supervisor.start_bucket(state.buckets)
        ref = Process.monitor(pid)
        refs = HashDict.put(state.refs, ref, name)
        :ets.insert(state.names, {name, pid})
        GenEvent.sync_notify(state.events, {:create, name, pid})
        {:noreply, %{state | refs: refs}}
    end
  end

  def handle_info({:DOWN, ref, :process, pid, _reason}, state) do
    # 6. Delete from the ETS table instead of the HashDict
    {name, refs} = HashDict.pop(state.refs, ref)
    :ets.delete(state.names, name)
    GenEvent.sync_notify(state.events, {:exit, name, pid})
    {:noreply, %{state | refs: refs}}
  end

  def handle_info(_msg, state) do
    {:noreply, state}
  end
end
```

注意，修改前的```KV.Registry.lookup/2```给服务器发送请求；修改后，它就直接从ETS表里面读取数据了。该表是对各进程都共享的。
这就是我们实现的缓存机制的大体想法。

为了让缓存机制工作，新建的ETS起码需要```:protected```访问规则（默认的），这样客户端才能从中读取数据。
否则就只有```KV.Registry```进程才能访问。
我们还在启动ETS表时设置了```:read_concurrency```，为表的并发访问稍作优化。

我们以上的改动导致测试都挂了。一个重要原因是我们在启动注册表进程时，需要多传递一个参数给```KV.Registry.start_link/3```。
让我们重写```setup```回调来修复测试代码```test/kv/registry_test.exs```：
```elixir
setup do
  {:ok, sup} = KV.Bucket.Supervisor.start_link
  {:ok, manager} = GenEvent.start_link
  {:ok, registry} = KV.Registry.start_link(:registry_table, manager, sup)

  GenEvent.add_mon_handler(manager, Forwarder, self())
  {:ok, registry: registry, ets: :registry_table}
end
```

注意我们传递了一个表名```:registry_table```给```KV.Registry.start_link/3```，
其后返回了```ets: :registry_table```，成为了测试的上下文。

修改了这个回调后，测试仍有fail，差不多都是这个样子：
```
1) test spawns buckets (KV.RegistryTest)
   test/kv/registry_test.exs:38
   ** (ArgumentError) argument error
   stacktrace:
     (stdlib) :ets.lookup(#PID<0.99.0>, "shopping")
     (kv) lib/kv/registry.ex:22: KV.Registry.lookup/2
     test/kv/registry_test.exs:39
```

这是因为我们传递了注册表进程的pid给函数```KV.Registry.lookup/2```，而它期待的却是ETS的表名。
为了修复我们要把所有的：
```elixir
KV.Registry.lookup(registry, ...)
```
都改为：
```elixir
KV.Registry.lookup(ets, ...)
```

其中获取```ets```的方法跟我们获取注册表一个样子：
```elixir
test "spawns buckets", %{registry: registry, ets: ets} do
```

像这样，我们对测试进行修改，把```ets```传递给```lookup/2```。一旦我们完成这些修改，有些测试还是会失败。
你还会观察到，每次执行测试，成功和失败不是稳定的。例如，对于“派生bucket进程”这个测试来说：
```elixir
test "spawns buckets", %{registry: registry, ets: ets} do
  assert KV.Registry.lookup(ets, "shopping") == :error

  KV.Registry.create(registry, "shopping")
  assert {:ok, bucket} = KV.Registry.lookup(ets, "shopping")

  KV.Bucket.put(bucket, "milk", 1)
  assert KV.Bucket.get(bucket, "milk") == 1
end
```

有可能会在这行失败：
```elixir
assert {:ok, bucket} = KV.Registry.lookup(ets, "shopping")
```

但是假如我们在这行之前创建一个bucket，还会失败吗？

原因在于（嗯哼！基于教学目的），我们犯了两个错误：    
1. 我们过于冒进地使用缓存来优化  
2. 我们使用的是```cast/2```，它应该是```call/2```

## 6.2-竞争条件？
用Elixir编程不会让你避免竞争状态。但是Elixir关于“没啥是共享”的这个特点可以帮助你很容易找到导致竞争状态的根本原因。

我们测试中发生的事儿是__延迟__---介于我们操作和我们观察到ETS表被改动之间。下面是我们期望发生的：  
1. 我们执行```KV.Registry.create(registry, "shopping")```  
2. 注册表进程创建了bucket，并且更新了缓存表  
3. 我们用```KV.Registry.lookup(ets, "shopping")```从表中获取信息  
4. 上面的命令返回```{:ok, bucket}```  

但是，因为```KV.Registry.create/2```使用cast操作，命令在真正修改表之前先返回了结果！换句话说，其实发生了下面的事：  
1. 我们执行```KV.Registry.create(registry, "shopping")```  
2. 我们用```KV.Registry.lookup(ets, "shopping")```从表中获取信息  
3. 命令返回```:error```  
4. 注册表进程创建了bucket，并且更新了缓存表  

要修复这个问题，只需要让```KV.Registry.create/2```同步操作，使用```call/2```而不是```cast/2```。
这就能保证客户端只会在表被修改后才能继续下面的操作。让我们来修改相应函数和回调：
```elixir
def create(server, name) do
  GenServer.call(server, {:create, name})
end

def handle_call({:create, name}, _from, state) do
  case lookup(state.names, name) do
    {:ok, pid} ->
      {:reply, pid, state} # Reply with pid
    :error ->
      {:ok, pid} = KV.Bucket.Supervisor.start_bucket(state.buckets)
      ref = Process.monitor(pid)
      refs = HashDict.put(state.refs, ref, name)
      :ets.insert(state.names, {name, pid})
      GenEvent.sync_notify(state.events, {:create, name, pid})
      {:reply, pid, %{state | refs: refs}} # Reply with pid
  end
end
```

我们只是简单地把回调里的```handle_cast/2```改成了```handle_call/3```，并且返回创建的bucket的pid。

现在执行下测试。这次，我们要使用```--trace```选项：
```elixir
$ mix test --trace
```

如果你的测试中有死锁或者竞争条件时，```--trace```选项非常有用。因为它可以同步执行所有测试（而```async: true```没啥效果），并且显式每条测试的详细信息。这次我们应该只有一条失败（可能也是间歇性的）：
```
1) test removes buckets on exit (KV.RegistryTest)
   test/kv/registry_test.exs:48
   Assertion with == failed
   code: KV.Registry.lookup(ets, "shopping") == :error
   lhs:  {:ok, #PID<0.103.0>}
   rhs:  :error
   stacktrace:
     test/kv/registry_test.exs:52
```

根据错误信息，我们期望表中没有bucket，但是它却有。
这个问题和我们刚刚解决的相反：之前的问题是创建bucket的命令与更新表之间的延迟，而现在是bucket处理退出操作与清除它在表中的记录之间的延迟。

不幸的是，这次我们无法简单地把```handle_info/2```改成一个同步的操作。但是我们可以用事件管理器的通知来修复该失败。
先来看看我们```handle_info/2```的实现：
```elixir
def handle_info({:DOWN, ref, :process, pid, _reason}, state) do
  # 5. Delete from the ETS table instead of the HashDict
  {name, refs} = HashDict.pop(state.refs, ref)
  :ets.delete(state.names, name)
  GenEvent.sync_notify(state.event, {:exit, name, pid})
  {:noreply, %{state | refs: refs}}
end
```

注意我们在发通知__之前__就从ETS表中进行删除操作。这是有意为之的。
这意味着当我们收到```{:exit, name, pid}```通知的时候，表即已经是最新了。让我们更新剩下的代码：
```elixir
test "removes buckets on exit", %{registry: registry, ets: ets} do
  KV.Registry.create(registry, "shopping")
  {:ok, bucket} = KV.Registry.lookup(ets, "shopping")
  Agent.stop(bucket)
  assert_receive {:exit, "shopping", ^bucket} # Wait for event
  assert KV.Registry.lookup(ets, "shopping") == :error
end
```

我们对测试稍作调整，保证先收到```{:exit, name, pid}消息，再执行```KV.Registry.lookup/2```。

你看，我们能够通过修改程序逻辑来使测试通过，而不是使用诸如```:timer.sleep/1```或者其它小技巧。这很重要。
大部分时间里，我们依赖于事件，监视以及消息机制来确保系统处在期望状态，在执行测试断言之前。

为方便，下面给出能通过的测试全文：
```elixir
defmodule KV.RegistryTest do
  use ExUnit.Case, async: true

  defmodule Forwarder do
    use GenEvent

    def handle_event(event, parent) do
      send parent, event
      {:ok, parent}
    end
  end

  setup do
    {:ok, sup} = KV.Bucket.Supervisor.start_link
    {:ok, manager} = GenEvent.start_link
    {:ok, registry} = KV.Registry.start_link(:registry_table, manager, sup)

    GenEvent.add_mon_handler(manager, Forwarder, self())
    {:ok, registry: registry, ets: :registry_table}
  end

  test "sends events on create and crash", %{registry: registry, ets: ets} do
    KV.Registry.create(registry, "shopping")
    {:ok, bucket} = KV.Registry.lookup(ets, "shopping")
    assert_receive {:create, "shopping", ^bucket}

    Agent.stop(bucket)
    assert_receive {:exit, "shopping", ^bucket}
  end

  test "spawns buckets", %{registry: registry, ets: ets} do
    assert KV.Registry.lookup(ets, "shopping") == :error

    KV.Registry.create(registry, "shopping")
    assert {:ok, bucket} = KV.Registry.lookup(ets, "shopping")

    KV.Bucket.put(bucket, "milk", 1)
    assert KV.Bucket.get(bucket, "milk") == 1
  end

  test "removes buckets on exit", %{registry: registry, ets: ets} do
    KV.Registry.create(registry, "shopping")
    {:ok, bucket} = KV.Registry.lookup(ets, "shopping")
    Agent.stop(bucket)
    assert_receive {:exit, "shopping", ^bucket} # Wait for event
    assert KV.Registry.lookup(ets, "shopping") == :error
  end

  test "removes bucket on crash", %{registry: registry, ets: ets} do
    KV.Registry.create(registry, "shopping")
    {:ok, bucket} = KV.Registry.lookup(ets, "shopping")

    # Kill the bucket and wait for the notification
    Process.exit(bucket, :shutdown)
    assert_receive {:exit, "shopping", ^bucket}
    assert KV.Registry.lookup(ets, "shopping") == :error
  end
end
```

随着测试通过，我们只需更新监督者```init/1```回调函数的代码（文件```lib/kv/supervisor.ex```），传递ETS表的名字作为参数给注册表工人：
```elixir
@manager_name KV.EventManager
@registry_name KV.Registry
@ets_registry_name KV.Registry
@bucket_sup_name KV.Bucket.Supervisor

def init(:ok) do
  children = [
    worker(GenEvent, [[name: @manager_name]]),
    supervisor(KV.Bucket.Supervisor, [[name: @bucket_sup_name]]),
    worker(KV.Registry, [@ets_registry_name, @manager_name,
                         @bucket_sup_name, [name: @registry_name]])
  ]

  supervise(children, strategy: :one_for_one)
end
```

注意我们仍使用```KV.Registry```作为ETS表的名字，好让debug方便些，因为它指明了使用它的模块。ETS名和进程名分别存储在不同的注册表，以避免冲突。

## 6.3-ETS当持久存储用

到目前为止，我们在初始化注册表的时候创建了一个ETS表，而没有操心在注册表结束时关闭该ETS表。
这是因为ETS表是“连接”（某种修辞上说）着创建它的进程的。如果那进程挂了，表也会自动关闭。

这作为默认行为实在是太方便了，我们可以在将来更多地利用这个特点。
记住，注册表和bucket监督者之间有依赖。注册表挂，我们希望bucket监督者也挂。
因为一旦注册表挂，所有连接bucket进程的信息都会丢失。
但是，假如我们能保存注册表的数据怎么样？
如果我们能做到这点，就可以去除注册表和bucket监督者之间的依赖了，让```:one_for_one```成为监督者最合适的策略。

要做到这点需要些小改动。首先我们需要在监督者内启动ETS表。其次，我们需要把表的访问类型从```:protected```改成```:public```。
因为表的所有者是监督者，但是进行修改操作的仍然是时间管理者。

让我们从修改```KV.Supervisor```的```init/1```回调开始：
```elixir
def init(:ok) do
  ets = :ets.new(@ets_registry_name,
                 [:set, :public, :named_table, {:read_concurrency, true}])

  children = [
    worker(GenEvent, [[name: @manager_name]]),
    supervisor(KV.Bucket.Supervisor, [[name: @bucket_sup_name]]),
    worker(KV.Registry, [ets, @manager_name,
                         @bucket_sup_name, [name: @registry_name]])
  ]

  supervise(children, strategy: :one_for_one)
end
```

接下来，我们修改```KV.Registry```的```init/1```回调，因为它不再需要创建一个表，而是需要一个表作为参数：
```elixir
def init({table, events, buckets}) do
  refs = HashDict.new
  {:ok, %{names: table, refs: refs, events: events, buckets: buckets}}
end
```

最终，我们修改```test/kv/registry_test.exs```中的```setup```回调，来显式地创建ETS表。
我们还将用这个机会分离```setup```的功能，放到一个方便的私有函数中：
```elixir
setup do
  ets = :ets.new(:registry_table, [:set, :public])
  registry = start_registry(ets)
  {:ok, registry: registry, ets: ets}
end

defp start_registry(ets) do
  {:ok, sup} = KV.Bucket.Supervisor.start_link
  {:ok, manager} = GenEvent.start_link
  {:ok, registry} = KV.Registry.start_link(ets, manager, sup)

  GenEvent.add_mon_handler(manager, Forwarder, self())
  registry
end
```

这之后，我们的测试应该都绿啦！

现在只剩下一个场景需要考虑：一旦我们收到了ETS表，可能有现存的bucket的pid在这个表中。
这是我们这次改动的目的。
但是，新启动的注册表进程没有监视这些bucket，因为它们是作为之前的注册表的一部分创建的，现在那些注册表已经不存在了。
这意味着表将被严重拖累，因为我们都不去清除已经挂掉的bucket。

来增加一个测试来暴露这个bug：
```elixir
test "monitors existing entries", %{registry: registry, ets: ets} do
  bucket = KV.Registry.create(registry, "shopping")

  # Kill the registry. We unlink first, otherwise it will kill the test
  Process.unlink(registry)
  Process.exit(registry, :shutdown)

  # Start a new registry with the existing table and access the bucket
  start_registry(ets)
  assert KV.Registry.lookup(ets, "shopping") == {:ok, bucket}

  # Once the bucket dies, we should receive notifications
  Process.exit(bucket, :shutdown)
  assert_receive {:exit, "shopping", ^bucket}
  assert KV.Registry.lookup(ets, "shopping") == :error
end
```

执行这个测试，它将失败：
```
1) test monitors existing entries (KV.RegistryTest)
   test/kv/registry_test.exs:72
   No message matching {:exit, "shopping", ^bucket}
   stacktrace:
     test/kv/registry_test.exs:85
```

这是我们期望的。如果bucket不被监视，在它挂的时候，注册表将得不到通知，因此也没有事件发生。
我们可以修改```KV.Registry```的```init/1```回调来修复这个问题。给所有表中的现存条目设置监视器：
```elixir
def init({table, events, buckets}) do
  refs = :ets.foldl(fn {name, pid}, acc ->
    HashDict.put(acc, Process.monitor(pid), name)
  end, HashDict.new, table)

  {:ok, %{names: table, refs: refs, events: events, buckets: buckets}}
end
```

我们用```:ets.foldl/3```来遍历表中所有条目，类似于```Enum.reduce/3```。它为每个条目执行提供的函数，并且用一个累加器累加结果。
在函数回调中，我们监视每个表中的pid，并相应地更新存放引用信息的字典。
如果有某个条目是挂掉的，我们还能收到```:DOWN```消息，稍后可以清除它们。

本章让监督者拥有ETS表，并且使其将表作为参数传递给注册表进程。通过这样的方法，我们让程序变得更加健壮。
我们还探索了把ETS当作缓存，并且讨论了如果在客户端和服务器共享数据时会进入的竞争状态。
