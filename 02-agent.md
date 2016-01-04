2-Agent
========
[状态的麻烦](#)  
[Agents](#)  
[ExUnit回调函数](#)  
[其它Agent行为](#)  
[Agent中的C/S模式](#)  

本章我们将创建一个名为```KV.Bucket```的模块。这个模块负责存储可被不同进程读写的键值对。

如果你跳过了“入门”手册，或者是太久以前读的，那么建议你最好重新阅读一下关于 **进程** 的那一章。
它是本节所内容的起点。

## 2.1-状态的麻烦
Elixir是一种“（变量值）不可变”的语言。默认情况下，没有什么是被共享的。
如果想要提供某种状态，通过其创建可以从不同地方访问的“桶”，我们有两种选择：
* 进程
* ETS（[Erlang Term Storage](http://www.erlang.org/doc/man/ets.html)）

我们之前介绍过进程，但ETS是个新东西，在后面的章节中再去探讨。
而当用到进程时，我们很少会去自己动手从底层做起，而是用Elixir和OTP中抽象出来的东西代替：  
* [Agent](http://elixir-lang.org/docs/stable/elixir/Agent.html) -
对状态简单的封装
* [GenServer](http://elixir-lang.org/docs/stable/elixir/GenServer.html) -
“通用的服务器”（进程）。它封装了状态，提供了同步或异步调用，支持代码热更新等等
* [GenEvent](http://elixir-lang.org/docs/stable/elixir/GenEvent.html) -
“通用的事件”管理器，允许向多个接收者发布事件消息
* [Task](http://elixir-lang.org/docs/stable/elixir/Task.html) -
计算处理的异步单元，可以派生出进程并稍后收集计算结果

我们在本“进阶”手册中会逐一讨论这些抽象物。
记住它们都是在进程基础上实现的，使用Erlang虚拟机提供的基本特性，
如```send```，```receive```，```spawn```和```link```。

## 2.2-Agents
[Agent](http://elixir-lang.org/docs/stable/elixir/Agent.html)是对状态简单的封装。
如果你想要一个可以保存状态的地方（进程），那么Agent就是不二之选。
让我们在工程里启动一个```iex```对话：
```
$iex -S mix
```

然后“玩弄”一下Agent：
```elixir
iex> {:ok, agent} = Agent.start_link fn -> [] end
{:ok, #PID<0.57.0>}
iex> Agent.update(agent, fn list -> ["eggs"|list] end)
:ok
iex> Agent.get(agent, fn list -> list end)
["eggs"]
iex> Agent.stop(agent)
:ok
```
这里用某个初始状态（空列表）启动了一个agent，然后执行了一个命令来修改这个状态，
加了一个新的列表项到头部。```Agent.update/3```的第二个参数是一个匿名函数：
它使用agent当前状态为输入，返回想要的新状态。
最终我们获取整个列表。```Agent.get/3```函数的第二个参数是个匿名函数：
它使用当前状态为输入，返回的值就是```Agent.get/3```的返回值。
一旦我们用完agent，我们调用```Agent.stop/1```来终止agent进程。

现在我们用Agent来实现```KV.Bucket```。当时在开始之前，我们先写些测试。
新建文件```test/kv/bucket_test.exs```（回想一下```.exs```文件），内容是：
```elixir
defmodule KV.BucketTest do
  use ExUnit.Case, async: true

  test "stores values by key" do
    {:ok, bucket} = KV.Bucket.start_link
    assert KV.Bucket.get(bucket, "milk") == nil

    KV.Bucket.put(bucket, "milk", 3)
    assert KV.Bucket.get(bucket, "milk") == 3
  end
end
```

我们的第一条测试很直白：启动一个```KV.Bucket```，然后执行```get/2```和```put/2```操作。
最后判断结果。我们不需要显式地停止agent进程。
因为该test里面用到的agent进程是链接到测试进程的，测试进程一结束它就会跟着结束。

同时还要注意我们向```ExUnit.Case```传递了一个```async:true```的选项。
这个选项使得该测试用例与其它同样包含```:async```选项的测试用例并行执行。
这种方式能够更好地利用计算机多核的能力。但要注意，这样的话，测试用例不能依赖或改变某些全局的值。
比如测试需要向文件系统里写入文字，或者注册进程，或者访问数据库等。
你在放置```:async```标记前必须考虑会不会在两个测试之间造成资源竞争。

不管是不是异步执行的，很明显我们的测试会失败，因为该实现的功能一个都没实现。

为了修复失败的用例，我们来创建文件```lib/kv/bucket.ex```，输入以下内容。
你可以不看下方的代码，自己随便尝试着创建agent的行为：
```elixir
defmodule KV.Bucket do
  @doc """
  Starts a new bucket.
  """
  def start_link do
    Agent.start_link(fn -> %{} end)
  end

  @doc """
  Gets a value from the `bucket` by `key`.
  """
  def get(bucket, key) do
    Agent.get(bucket, &Map.get(&1, key))
  end

  @doc """
  Puts the `value` for the given `key` in the `bucket`.
  """
  def put(bucket, key, value) do
    Agent.update(bucket, &Map.put(&1, key, value))
  end
end
```
我们使用图(Map)来存储我们的键和值。函数捕捉符号```&```在《入门》中介绍过。
现在```KV.Bucket```模块定义好了，测试都通过了！你可以执行```mix test```试试。

## 2.3-ExUnit回调函数
在继续为```KV.Bucket```加入更多功能之前，先讲一讲ExUnit的回调函数。
你可能已经想到，每一个```KV.Bucket```的测试用例都需要用到bucket。
它要在该测试用例启动时设置好，还要在该测试用例结束时停止。
幸运的是，ExUnit支持回调函数，使我们跳过这重复机械的任务。

让我们使用回调机制重写刚才的测试：
```elixir
defmodule KV.BucketTest do
  use ExUnit.Case, async: true

  setup do
    {:ok, bucket} = KV.Bucket.start_link
    {:ok, bucket: bucket}
  end

  test "stores values by key", %{bucket: bucket} do
    assert KV.Bucket.get(bucket, "milk") == nil

    KV.Bucket.put(bucket, "milk", 3)
    assert KV.Bucket.get(bucket, "milk") == 3
  end
end
```

我们首先利用```setup/1```宏，创建了设置bucket的回调函数。
这个函数会在每条测试用例执行前被执行一次，并且是与测试在同一个进程里。

注意我们需要一个机制来传递创建好的```bucket```的pid给测试用例。
我们使用 _测试上下文_ 来达到这个目的。
当在回调函数里返回```{:ok, bucket: bucket}```的时候，
ExUnit会把该返回值元祖（字典）的第二个元素merge进测试上下文中。
测试上下文是一个图，我们可以在测试用例的定义中匹配它，从而获取这个上下文的值给用例中的代码使用：
```elixir
test "stores values by key", %{bucket: bucket} do
  # `bucket` is now the bucket from the setup block
end
```
更多信息可以参考[ExUnit.Case](http://elixir-lang.org/docs/stable/ex_unit/ExUnit.Case.html)模块文档，
以及[回调函数](http://elixir-lang.org/docs/stable/ex_unit/ExUnit.Callbacks.html)。

## 2.4-其它Agent行为

除了“读取”或者“修改”agent的状态，agent还允许我们使用
函数```Agent.get_and_update/2```“读取并修改”它维持的状态。
我们用这个函数来实现删除```KV.Bucket.delete/2```功能---从bucket中删除一个值，并返回该值：
```elixir
@doc """
Deletes `key` from `bucket`.

Returns the current value of `key`, if `key` exists.
"""
def delete(bucket, key) do
  Agent.get_and_update(bucket, &Map.pop(&1, key))
end
```

现在轮到你来给上面的代码写个测试啦。
你可以阅读[Agent模块的文档](http://elixir-lang.org/docs/stable/elixir/Agent.html)
获取更多信息。

## 2.5-Agent中的C/S模式

在进入下一章之前，让我们讨论一下agent中的C/S二元模式。
先来展开刚刚写好的```delete/2```函数：
```elixir
def delete(bucket, key) do
  Agent.get_and_update(bucket, fn dict->
    Map.pop(dict, key)
  end)
end
```

我们传递给agent的函数中的任何东西，都会出现在agent的进程里。
在这里，因为agent进程负责接收和回复我们的消息，因此可以说agent进程就是个服务器（服务端）。
而那个方法之外的任何东西，都被看成是在客户端的范围内。

这个区别很重要。如果有大量的工作要做，你必须考虑这个工作是放在客户端还是在服务器上执行。比如：
```elixir
def delete(bucket, key) do
  :timer.sleep(1000) # puts client to sleep
  Agent.get_and_update(bucket, fn dict ->
    :timer.sleep(1000) # puts server to sleep
    Map.pop(dict, key)
  end)
end
```

当服务器上执行一个很耗时的工作时，所有其它对该服务器的请求都必须等待，直到那个工作完成。
这会造成客户端的超时。

下一章我们会探索通用服务器GenServer，它在概念上对服务器与客户端的隔离更明显。
