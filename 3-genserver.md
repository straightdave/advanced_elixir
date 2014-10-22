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
回忆在《入门》中进程那章，我们学习过注册进程的名字。鉴于此，我们使用这个方法来给bucket进程加上名字：
```
iex> Agent.start_link(fn -> [] end, name: :shopping)
{:ok, #PID<0.43.0>}
iex> KV.Bucket.put(:shopping, "milk", 1)
:ok
iex> KV.Bucket.get(:shopping, "milk")
1
```





















