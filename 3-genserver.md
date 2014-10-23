3-GenServer
===========

>本章讲述的GenServer不论在Erlang还是Elixir中都是非常核心的内容。
GenServer是OTP提供的一个抽象物，对OTP编程中经常用到的一个事物---服务器模型其中的通用部分进行了封装，从而程序员不需要重复实现这些通用部分，而是实现关键的行为代码。该行为代码页分两部分，用户API和操作该GenServer的回调方法。详细内容请开始阅读。

>GenServer是一个Elixir模块，在使用时用```use```导入。该知识点可以参考即将推出的《Elixir元编程》内容。

[第一个GenServer]()  
[测试一个GenServer]()  
[需要监控]()  
[call，cast还是info？]()  
[监视器还是链接？]()  

上一章我们用agent实现了buckets，而根据第一章所描述的，我们的设计是给每个bucket赋予一个名字：
```
CREATE shopping
OK

PUT shopping milk 1
OK

GET shopping milk
1
OK
```

因为agent是个进程，因而每个bucket首先有一个进程的id（pid）而不是名字。
回忆在《入门》中的进程那章，我们学习过给进程注册名字。
鉴于此，我们可以使用这个方法来给bucket起名：
```elixir
iex> Agent.start_link(fn -> [] end, name: :shopping)
{:ok, #PID<0.43.0>}
iex> KV.Bucket.put(:shopping, "milk", 1)
:ok
iex> KV.Bucket.get(:shopping, "milk")
1
```

这可是个很差的主意！在Elixir中，这些名字都会存储为原子。这意味着我们从外部客户端输入的bucket名字，都会被转换成原子。
记住，__绝对不要把用户输入转换为原子__。这是因为原子是不会被垃圾收集器收集。一旦原子被创建，它就不会被撤下（你也没法主动释放一个原子，对吧）。使用用户输入生成原子就意味着用户可以插入足够不同的名字来耗尽系统内存空间！
在实际操作中，在它用完内存之前会先触及Erland虚拟机的最大原子数量，从而造成系统崩溃。

比起滥用名字注册机制，我们可以创建我们自己的_注册表进程_来维护一个字典，用该字典联系起每个bucket的名字和进程。

这个注册表要能够保证永远处于最新状态。如果有一个bucket进程因故崩溃，注册表必须清除该进程信息，以防止继续服务下次查找请求。
在Elixir中，我们描述这种情况会说“该注册表进程需要监视（monitor）每个bucket进程”。  

我们将使用一个GenServer来创建一个注册表进程用来监视bucket进程。
在Elixir和OTP中，GenServer是创建这样进程的首选抽象物。

## 3.1-第一个GenServer

一个GenServer实现分为两个部分：用户API和服务器回调函数。这两部分都要写在同一个模块里。
下面我们创建文件```lib/kv/registry.ex```，包含以下内容：
```elixir
defmodule KV.Registry do
  use GenServer

  ## Client API

  @doc """
  Starts the registry.
  """
  def start_link(opts \\ []) do
    GenServer.start_link(__MODULE__, :ok, opts)
  end

  @doc """
  Looks up the bucket pid for `name` stored in `server`.

  Returns `{:ok, pid}` if the bucket exists, `:error` otherwise.
  """
  def lookup(server, name) do
    GenServer.call(server, {:lookup, name})
  end

  @doc """
  Ensures there is a bucket associated to the given `name` in `server`.
  """
  def create(server, name) do
    GenServer.cast(server, {:create, name})
  end

  ## Server Callbacks

  def init(:ok) do
    {:ok, HashDict.new}
  end

  def handle_call({:lookup, name}, _from, names) do
    {:reply, HashDict.fetch(names, name), names}
  end

  def handle_cast({:create, name}, names) do
    if HashDict.get(names, name) do
      {:noreply, names}
    else
      {:ok, bucket} = KV.Bucket.start_link()
      {:noreply, HashDict.put(names, name, bucket)}
    end
  end
end
```

第一个函数是```start_link/1```，它启动了一个新的GenServer。
其调用GenServer模块的```start_link/3```函数所使用的三个参数：  

1. 定义和实现了服务器回调函数的模块名称。这里的```__MODULE__```是当前模块名
2. 初始参数，这里是```:ok```
3. 一个选项列表，它可以存放服务器的名字等

你可以向一个GenServer发送两种请求。__Call__是同步的，服务器__必须__发送回复给该类请求。
__Cast__是异步的，服务器__不会__发送回复消息。

再往下的两个方法，```lookup/2```和```create/2```，它们用了两种不同方式发送请求给服务器。
这两种请求，会被第一个参数所指认的服务器中的```handle_call/3```和```handle_cast/2```函数处理（因此你的服务器回调函数必须包含这两个函数）。```GenServer.call/2```和```GenServer.cast/2```除了指认服务器之外，还告诉服务器它们要发送的请求。
这个请求存储在元组里，这里即```{:lookup, name}```和```{:create, name}```，在下面写相应的回调处理函数时会用到。
这个消息元组第一个元素一般是要服务器做的事儿，后面的元素就是该动作的参数。

在服务器这边，我们要实现一系列服务器回调函数来实现服务器的启动、停止以及处理请求等。
回调函数是可选的，我们在这里只实现所关系的那几个。

第一个是```int/1```回调函数，它接受一个状态参数（你在用户API中调用```GenServer.start_link/3```中使用的那个），返回```{:ok, state}```。这里```state```是一个新建的```HashDict```。
我们现在已经可以观察到，GenServer的API中，客户端和服务器之间的界限十分明显。```start_link/3```在客户端发生。
而其对应的```int/1```在服务器端运行。

对于```call```请求，我们在服务器端必须实现```handle_call/3```回调函数。参数：接收某请求（那个元组）、请求来源(```_from```)以及当前服务器状态（```names```）。```handle_call/3```函数返回一个```{:reply, reply, new_state}```形式的元组。其中，```reply```是你要回复给客户端的东西，而```new_statue```是新的服务器状态。

对于```cast```请求，我们必须实现一个```handle_cast/2```回调函数，接受参数：```request```以及当前服务器状态（```names```）。这个函数返回```{:noreply, new_state}```形式的元组。

这两个回调函数，```handle_call/3```和```handle_cast/2```还可以返回其它几种形式的元组。还有另外几种回调函数，如```terminate/2```和```code_change/3```等。可以参考[完整的GenServer文档](http://elixir-lang.org/docs/stable/elixir/GenServer.html)来学习相关知识。

现在，来写几个测试来保证我们这个GenServer可以执行预期工作。

## 3.2-测试一个GenServer
测试一个GenServer和测试agent比没有多少不同。我们在测试的setup回调中启动该服务器进程用以测试。
用一下内容创建测试文件```test/kv/registry_test.exs```：
```elixir
defmodule KV.RegistryTest do
  use ExUnit.Case, async: true

  setup do
    {:ok, registry} = KV.Registry.start_link
    {:ok, registry: registry}
  end

  test "spawns buckets", %{registry: registry} do
    assert KV.Registry.lookup(registry, "shopping") == :error

    KV.Registry.create(registry, "shopping")
    assert {:ok, bucket} = KV.Registry.lookup(registry, "shopping")

    KV.Bucket.put(bucket, "milk", 1)
    assert KV.Bucket.get(bucket, "milk") == 1
  end
end
```

哈，居然都过了！

关闭这个注册表进程，我们只需在测试结束时简单地发送```:shutdown```信号给该进程（别忘记，GenServer也是个进程而已，发送结束进程信号就可以粗暴地停止它）。这个方法对于测试是还好啦，只是，最好你还是在GenServer的处理逻辑里加上关于停止的方法。
比如，定义一个```stop/1```函数来发送要求停止的```call```请求，就是极好的：
```elixir
## Client API

@doc """
Stops the registry.
"""
def stop(server) do
  GenServer.call(server, :stop)
end

## Server Callbacks

def handle_call(:stop, _from, state) do
  {:stop, :normal, :ok, state}
end
```
上面代码中，新的```handle_call/3```回调函数专门处理```:stop```请求。它返回的元组包括一个原子```:stop```，后面跟着原因```:normal```，然后是```:ok```和服务器状态。

## 3.3-需要监控

































