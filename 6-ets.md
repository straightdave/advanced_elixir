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

































