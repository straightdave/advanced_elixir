8-Task模块和通用TCP服务器（gen_tcp）
================
* [Echo服务器]()  
* [Tasks]()
* [Task的监督者]()

本章我们学习如何使用[Erlang的:gen_tcp模块](http://erlang.org/doc/man/gen_tcp.html)来处理请求。
在未来几章中我们还会扩展我们的服务器，使之能够服务于真正的命令。
这也是我们探索Elixir的Task模块的大好机会。

## 8.1-Echo服务器

我们首先实现一个Echo（回声）服务器来开始我们的TCP服务器之旅。它只是简单地返回从请求中收到的文字。
我们会慢慢地改进这个服务器，使它有监督者来监督，并且可以处理大量连接。

一个TCP服务器，总的来说，实现以下几步：  
1. 在可用端口建立socket连接，监听这个端口
2. 等待这个端口的客户端连接，有了就接受它
3. 读取客户端请求并且写回复

我们来实现这些步骤。在```apps/kv_server```程序中，打开文件```lib/kv_server.ex```，添加以下函数：
```elixir
def accept(port) do
  # The options below mean:
  #
  # 1. `:binary` - receives data as binaries (instead of lists)
  # 2. `packet: :line` - receives data line by line
  # 3. `active: false` - block on `:gen_tcp.recv/2` until data is available
  #
  {:ok, socket} = :gen_tcp.listen(port,
                    [:binary, packet: :line, active: false])
  IO.puts "Accepting connections on port #{port}"
  loop_acceptor(socket)
end

defp loop_acceptor(socket) do
  {:ok, client} = :gen_tcp.accept(socket)
  serve(client)
  loop_acceptor(socket)
end

defp serve(client) do
  client
  |> read_line()
  |> write_line(client)

  serve(client)
end

defp read_line(socket) do
  {:ok, data} = :gen_tcp.recv(socket, 0)
  data
end

defp write_line(line, socket) do
  :gen_tcp.send(socket, line)
end
```

我们通过调用```KVServer.accept(4040)```来启动服务器，其中4040是端口号。在```accept/1```中，第一步是去监听这个端口，知道socket变成可用状态，然后调用```loop_acceptor/1```。函数```loop_acceptor/1```只是一个循环，来接受客户端的连接。
对于每次接受的客户端连接，我们调用```serve/1```函数。

函数```serve/1```也是个循环，它一次从socket中读取一行，并将其写进给socket的回复。
注意```serve/1```使用了[管道运算符 ```|>```](http://elixir-lang.org/docs/stable/elixir/Kernel.html#%7C%3E/2)来表达操作流程。
管道运算符计算左侧函数计算的结果，并将其作为第一个参数传递给右侧函数调用。如：
```elixir
socket |> read_line() |> write_line(socket)
```

相当于：
```elixir
write_line(read_line(socket), socket)
```

>当使用```|>```运算符时，是否给函数调用加上括号是很重要的。举个例子：
```elixir
1..10 |> Enum.filter &(&1 <= 5) |> Enum.map &(&1 * 2)
```
会被翻译为：
```elixir
1..10 |> Enum.filter(&(&1 <= 5) |> Enum.map(&(&1 * 2)))
```
这个不是我们想要的，因为本应传给```Enum.filter/2```的那个匿名函数```&(&1<=5)```成了传给```Enum.map/2```的第一个参数。
解决方法就是加上括号：
```elixir
1..10 |> Enum.filter(&(&1 <= 5)) |> Enum.map(&(&1 * 2))
```

函数```read_line/2```中使用```:gen_tcp.recv/2```接收从socket传来的数据。
而```write_line/2```中使用```:gen_tcp.send/2```向socket写入数据。

这差不多就是我们为实现这个回声服务器所要做的。让我们试一试。

用```iex -S mix```在```kv_server```应用程序中启动对话，执行：
```elixir
iex> KVServer.accept(4040)
```
服务器就运行了，注意到此时该命令行会被阻塞。现在我们使用一个[telnet客户端](http://en.wikipedia.org/wiki/Telnet)
来访问这个服务器。基本上每个操作系统都有telnet客户端程序，命令也都差不多：
```
$ telnet 127.0.0.1 4040
Trying 127.0.0.1...
Connected to localhost.
Escape character is '^]'.
hello
hello
is it me
is it me
you are looking for?
you are looking for?
```

输入“hello”，按回车，你就会得到“hello”字样的回复。好牛逼！

退出telnet客户端方法不一，有些用```ctrl + ]```，有些是```quit```按回车。

一旦你退出telnet客户端，你会发现IEx会话中打印出一个错误信息：
```
** (MatchError) no match of right hand side value: {:error, :closed}
    (kv_server) lib/kv_server.ex:41: KVServer.read_line/1
    (kv_server) lib/kv_server.ex:33: KVServer.serve/1
    (kv_server) lib/kv_server.ex:27: KVServer.loop_acceptor/1
```
这是因为我们还期望从```:gen_tcp.recv/2```拿数据，但是客户端断了。我们将来要处理这个问题才行。

目前还有个更重要的bug要修：假如TCP接收者挂了怎么办？意为它没有监督者，不会自己重启，要是挂了我们将不能在处理更多的请求。
这就是为啥我们要将它挪进监督树。

## 8.2-Tasks

我们已经学习了Agent，通用服务器以及事件管理器。它们都可以进行多消息协作，或者管理状态。
但是，若是只需要处理一些任务，选什么呢？

[Task模块](http://elixir-lang.org/docs/stable/elixir/Task.html)为此提供了所需的功能。
例如，它有```start_link/3```函数，接受一个模块名、一个函数和函数的参数，从而执行这个传入的函数，并且还是作为监督树的一部分。

我们来试试。打开```lib/kv_server.ex```，修改下里```start/2```函数里的监督者：
```elixir
def start(_type, _args) do
  import Supervisor.Spec

  children = [
    worker(Task, [KVServer, :accept, [4040]])
  ]

  opts = [strategy: :one_for_one, name: KVServer.Supervisor]
  Supervisor.start_link(children, opts)
end
```

改动的意思是要让```KVServer.accept(4040)```成为一个工人来运行。目前我们暂时hardcode这个端口号，之后再讨论如何修改。

现在，这个服务器是监督树的一部分了，它应该会随着应用程序启动而自动运行。
在终端中输入```mix run --no-halt```，然后再次用telnet客户端来试试看是否还一切正常：
```
$ telnet 127.0.0.1 4040
Trying 127.0.0.1...
Connected to localhost.
Escape character is '^]'.
say you
say you
say me
say me
```
看，它还是好使！这回就算退了客户端，服务器挂了，你会看到又一个立马起来了。嗯，不错。。。不过它可伸缩性如何？

试着打开两个telnet客户端一起连接，你会注意到，第二个客户端根本不能回声：
```
$ telnet 127.0.0.1 4040
Trying 127.0.0.1...
Connected to localhost.
Escape character is '^]'.
hello
hello?
HELLOOOOOO?
```

看起来根本不工作嘛。这是因为处理请求和接受请求是在同一个进程。一个客户端连上来，就没法处理第二个了。

## 8.3-Task的监督者

为了让我们的服务器能够处理并发连接，我们需要让一个进程来当接收者，然后派生其它的进程来服务接收到的连接。
一个方案是：
```elixir
defp loop_acceptor(socket) do
  {:ok, client} = :gen_tcp.accept(socket)
  serve(client)
  loop_acceptor(socket)
end
```

函数```Task.start_link/1```类似```Task.start_link/3```，但是它可以接受一个匿名函数而不是（模块，函数，参数）的组合：
```elixir
defp loop_acceptor(socket) do
  {:ok, client} = :gen_tcp.accept(socket)
  Task.start_link(fn -> serve(client) end)
  loop_acceptor(socket)
end
```

我们翻过这个错了，记得吗？

和我们当时在注册表进程中调用```KV.Bucket.start_link/0```犯的错差不多。它意味着一个bucket挂会导致整个注册表进程挂。

上面的代码页犯了相同的错误：如果我们把```serve(client)```这个任务和接收者连接起来，那么在处理请求时发生的小事故就会导致请求接收者挂，继而导致连接都挂掉。

当时我们解决这个问题是用了一个简单的一对一监督者。这里我们也将使用相同的办法，除了一点：这个模式在Task中实在是太通用了，
所有Task已经为之提供了一个解决方案---一个简单的一对一监督者加上临时工（临时的工人），这个我们在之前的监督树中就是这么用的。

让我们再次修改下```start/2```函数，加个监督者：
```elixir
def start(_type, _args) do
  import Supervisor.Spec

  children = [
    supervisor(Task.Supervisor, [[name: KVServer.TaskSupervisor]]),
    worker(Task, [KVServer, :accept, [4040]])
  ]

  opts = [strategy: :one_for_one, name: KVServer.Supervisor]
  Supervisor.start_link(children, opts)
end
```

我们简单地启动了一个[```Task.Supervisor```](http://elixir-lang.org/docs/stable/elixir/Task.Supervisor.html)进程，
名字叫```Task.Supervisor```。记住，因为接收者任务依赖于这个监督者，因此该监督者必须先启动。

现在我们只需修改```loop_acceptor/2```，使用```Task.Supervisor```来处理每个请求：
```elixir
defp loop_acceptor(socket) do
  {:ok, client} = :gen_tcp.accept(socket)
  Task.Supervisor.start_child(KVServer.TaskSupervisor, fn -> serve(client) end)
  loop_acceptor(socket)
end
```

用命令```mix run --no-halt```启动新的服务器，现在就可以打开多个客户端来连接了。而且你会发现一个客户端退出不会让接收者挂掉。
好棒！

一下是完整的服务器实现，在单个模块中：
```elixir
defmodule KVServer do
  use Application

  @doc false
  def start(_type, _args) do
    import Supervisor.Spec

    children = [
      supervisor(Task.Supervisor, [[name: KVServer.TaskSupervisor]]),
      worker(Task, [KVServer, :accept, [4040]])
    ]

    opts = [strategy: :one_for_one, name: KVServer.Supervisor]
    Supervisor.start_link(children, opts)
  end

  @doc """
  Starts accepting connections on the given `port`.
  """
  def accept(port) do
    {:ok, socket} = :gen_tcp.listen(port,
                      [:binary, packet: :line, active: false])
    IO.puts "Accepting connections on port #{port}"
    loop_acceptor(socket)
  end

  defp loop_acceptor(socket) do
    {:ok, client} = :gen_tcp.accept(socket)
    Task.Supervisor.start_child(KVServer.TaskSupervisor, fn -> serve(client) end)
    loop_acceptor(socket)
  end

  defp serve(socket) do
    socket
    |> read_line()
    |> write_line(socket)

    serve(socket)
  end

  defp read_line(socket) do
    {:ok, data} = :gen_tcp.recv(socket, 0)
    data
  end

  defp write_line(line, socket) do
    :gen_tcp.send(socket, line)
  end
end
```

因为我们修改了监督者的需求，我们会问：我们的监督者策略还适用吗？

这里答案是Yes：如果接收者挂了，现存的连接是没理由一起挂的。另一方面，如果task监督者挂了，同样也没必要让接收者挂掉。
这和注册表进程那种情况相反，那种情况我们在一开始必须在注册表进程挂掉时让监督者也挂掉，直到后来我们用上了ETS来持久化保存状态。
而task是没有状态什么的，挂掉一个两个也不会拖谁的后腿。

下一章我们将开始解析客户请求，然后发送回复，从而完成我们的服务器。
