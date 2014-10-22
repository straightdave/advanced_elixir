3-GenServer
===========

[第一个GenServer]()  
[测试GenServer]()  
[需要监控]()  
[call，cast还是info？]()  
[监视器还是链接？]()  

上一章我们用agent实现了buckets，而第一章我们的设计是给每个bucket赋予一个名字：
```
CREATE shopping
OK

PUT shopping milk 1
OK

GET shopping milk
1
OK
```

因为agent是进程，因而每个bucket首先有一个进程的id（pid）而不是名字。
回忆在《入门》中进程那章，我们学习过给进程注册名字。
鉴于此，我们可以使用这个方法来给bucket进程加上名字：
```elixir
iex> Agent.start_link(fn -> [] end, name: :shopping)
{:ok, #PID<0.43.0>}
iex> KV.Bucket.put(:shopping, "milk", 1)
:ok
iex> KV.Bucket.get(:shopping, "milk")
1
```

这可是个很差的主意！在Elixir中，这些名字都会存储为原子。这意味着我们从外部客户端输入的bucket名字，都会被转换成原子。
记住，__绝对不要把用户输入转换为原子__。这是因为原子是不会被垃圾收集器收集的。一旦原子被创建，它就不会被撤下。
使用用户输入生成原子就意味着用户可以插入足够不同的名字来耗尽系统内存空间！
在实际操作中，在它用完内存之前会先触及Erland虚拟机的最大原子数量，从而造成系统崩溃。

比起滥用名字注册机制，我们可以创建我们自己的_注册表进程_来维护一个字典，用该字典联系起每个bucket的名字和对应的进程。

这个注册表要能够保证永远处于最新状态。日不，如果有一个bucket进程因故崩溃，注册表必须清理该进程信息，以防止继续服务下次查找请求。
在Elixir中，我们描述这种情况会说“该注册表需要监视（monitor）每个bucket”。

我们将使用一个GenServer来创建一个注册表进程用来监视bucket进程。
在Elixir和OTP中，GenServer是创建这样的通用服务器的首选抽象物。

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

第一个函数是```start_link/1```，它使用三个参数启动了一个新的GenServer：  
1. 实现了服务器回调函数的模块。这里的```__MODULE__```意为当前模块名
2. 初始参数，这里是```:ok```
3. 一个选项列表，它可以放服务器的名字等

你可以向一个GenServer发送有两种请求。__Call__是同步的，服务器__必须__发送回复给该类请求。
__Cast__是异步的，服务器不需要发送回复。

下面两个方法，```lookup/2```和```create/2```负责发送请求给服务器。




















