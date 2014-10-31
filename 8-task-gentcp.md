8-Task和gen_tcp
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











