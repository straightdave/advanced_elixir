1-Mix简介
=========

>因为比较忙，这个进阶篇完善和更新得非常缓慢。但是因为OTP才是Elixir的重点，所以一有空一定会努力完善下去的。

在这份指导手册中，我们将学习创建一个完整的Elixir应用程序，以及监督树、配置、测试等高级概念。

这个程序是一个分布式的键-值存储数据库。我们会把键-值存储在“桶”中，分布存储到多个节点。
我们还会创建一个简单的客户端工具，可以连接任意一个节点，并且能够发送类似以下的命令：
```
CREATE shopping
OK

PUT shopping milk 1
OK

PUT shopping eggs 3
OK

GET shopping milk
1
OK

DELETE shopping eggs
OK
```

为了编写这个程序，我们将主要用到以下三个工具：
*  __OTP(Open Telecom Platform)__
OTP是一个随Erlang发布的代码库集合。Erlang开发者使用OTP来创建健壮的、高度容错的程序。在本章中，我们将来探索与Elixir整合在一起的OTP，包括监督树、事件管理等等；  

*  __Mix__
Mix是随Elixir发布的构建工具，用来创建、编译、测试你的应用程序，还可以用来管理依赖等；  

*  __ExUnit__
ExUnit是一个随Elixir发布的单元测试工具

本章会用Mix来创建我们第一个工程，探索OTP、Mix以及ExUnit的各种特性。

>注意：  
本手册需要Elixir v0.15.0（1.2.0发布后，这里被改为1.2.0了）或以上。
你可以使用命令```elixir -v```查看版本。
如果需要，可以参考《Elixir入门手册》第一章节内容安装最新的版本。   
如果发现任何错误，请开issue或者发pull request。

## 1.1-第一个应用程序

当你安装Elixir时，你不仅得到了```elixir```，```elixirc```和```iex```命令，
还得到一个可执行的Elixir脚本叫做```mix```。

从命令行输入```mix new```命令来创建我们的第一个工程。
我们需要传递工程名称作为参数（在这里，比如叫做 ```kv```），
然后告诉mix我们的主模块的名字是全大写的```KV```。
否则按照默认，mix会创建一个主模块，名字是第一个字母大写的工程名称（```Kv```）。
因为K和V的含义在我们的程序中上是平等关系，所以最好是都用大写：
```sh
$ mix new kv --module KV
```

Mix将创建一个文件夹名叫```kv```，里面有一些文件：
```
* creating README.md
* creating .gitignore
* creating mix.exs
* creating config
* creating config/config.exs
* creating lib
* creating lib/kv.ex
* creating test
* creating test/test_helper.exs
* creating test/kv_test.exs
```

现在简单看看这些创建的文件。

>注意：  
Mix是一个Elixir可执行脚本。这意味着，你要想用mix为名直接调用它，
需要提前将Elixir目录放进系统的环境变量中。否则，你需要使用elixir来执行mix：
```sh
$ bin/elixir bin/mix new kv --module KV
```
你也可以用```-S```选项来执行elixir，它不管你有没有把mix的目录加入环境变量：
```sh
$ bin/elixir -S mix new kv --module KV
```

## 1.2-工程的编译

一个名叫```mix.exs```的文件会被自动创建在工程目录中。
它的主要作用是配置你的工程。它的内容如下（略去代码中的注释）：
```elixir
defmodule KV.Mixfile do
  use Mix.Project

  def project do
    [app: :kv,
     version: "0.0.1",
     elixir: "~> 1.2",
     build_embedded: Mix.env == :prod,
     start_permanent: Mix.env == :prod,
     deps: deps]
  end

  def application do
    [applications: [:logger]]
  end

  defp deps do
    []
  end
end
```

我们的```mix.exs```定义了两个公共函数：
一个是```project```，它返回工程的配置信息，如工程名称和版本；
另一个是```application```，它用来生成应用程序文件。

还有一个私有函数叫做```deps```，它被```project```函数调用，里面定义了工程的依赖。
不一定非要把```deps```定义为一个独立的函数，但是这样做可以使工程的配置文件看起来整洁美观。

Mix还生成了文件```lib/kv.ex```，其内容是个简单的模块定义：
```elixir
defmodule KV do
end
```

以上这个结构就足以编译我们的工程了：
```sh
$ cd kv
$ mix compile
```
将生成：
```
Compiled lib/kv.ex
Generated kv app
Consolidated List.Chars
Consolidated Collectable
Consolidated String.Chars
Consolidated Enumerable
Consolidated IEx.Info
Consolidated Inspect
```

注意文件```lib/kv.ex```被编译，生成了程序manifest文件：```kv.app```
及一些 _协议(参考入门手册)_。根据```mix.exs```的配置，所有编译产出被放在```_build```目录中。

一旦工程被编译成功，便可以从工程目录启动一个```iex```会话：
```sh
$ iex -S mix
```

## 1.3-执行测试

Mix还生成了合适的文件结构，来测试我们的工程。Mix工程一般沿用一些命名规则：
在```test```目录中，测试文件一般以```<filename>_test.exs```模式命名。
每一个```<filename>```对应一个```lib```目录中的文件名。
根据这个命名规则，我们已经有了测试```lib/kv.ex```所需的```test/kv_test.exs```文件。
只是目前它几乎什么也没做：
```elixir
defmodule KVTest do
  use ExUnit.Case
  doctest KV

  test "the truth" do
    assert 1 + 1 == 2
  end
end

```

需要注意几点：
1. 测试文件使用的扩展名(.exs)即Elixir脚本文件。这很方便，我们不用在跑测试前还编译一次。
2. 我们定义了一个测试模块名为```KVTest```，用```ExUnit.Case```来注入测试API，
并使用宏```test/2```定义了一个简单的测试；

Mix还生成了一个文件名叫```test/test_helper.exs```，它负责设置测试框架：
```
ExUnit.start()
```

每次Mix执行测试时，这个文件将自动被导入（required）。执行测试，使用命令```mix test```：
```
Compiled lib/kv.ex
Generated kv app
[...]
.

Finished in 0.04 seconds (0.04s on load, 0.00s on tests)
1 tests, 0 failures

Randomized with seed 540224
```

注意，每次运行```mix test```时，Mix会重新编译源文件，生成新的应用程序。
这是因为Mix支持多套执行环境，我们稍后章节会详细介绍。

另外，ExUnit为每一个成功的测试结果打印一个点，它还会自动随机安排测试顺序。
让我们把测试改成失败看看会发生啥。修改```test/kv_test.exs```里面的断言，改成：
```elixir
assert 1 + 1 == 3
```

现在再次运行```mix test```（注意这次没有编译行为发生）：
```
1) test the truth (KVTest)
   test/kv_test.exs:5
   Assertion with == failed
   code: 1 + 1 == 3
   lhs:  2
   rhs:  3
   stacktrace:
     test/kv_test.exs:6

Finished in 0.05 seconds (0.05s on load, 0.00s on tests)
1 tests, 1 failures
```

ExUnit会为每个失败的测试结果打印一个详细的报告。其内容包含了测试名称，失败的代码，
失败断言中```==```号的左值（lhs）和右值(rhs)。

在错误提示的第二行（测试名称下面那行），是该测试的代码位置。
将这个位置作为参数给```mix test```命令，则将仅执行该条测试：
```
$ mix test test/kv_test.exs:5
```
这个十分有用是吧。

最后是关于错误的追踪栈信息，给出关于测试的额外信息。
包括测试失败的地方，还有原文件中产生失败的具体位置等。

## 1.4-环境

Mix支持“环境”的概念。它允许开发者为某些场景定义不同的编译等动作。
默认地，Mix理解三种环境：
* ```:dev``` - Mix任务的默认执行环境（如编译等操作）
* ```:test``` - ```mix test```使用的环境
* ```:prod``` - 用来将应用程序发布到产品环境

环境配置只对当前工程有效。我们之后会看到，向工程中添加的依赖默认在```:prod```环境下工作。

可以通过访问```mix.exs```工程配置文件中的```Mix.env```函数定义不同的环境配置，
它会以原子形式返回当前的环境。
比如我们用之于```:build_embedded```和```:start_permanent：```这两个选项：
```elixir
def project do
  [...,
   build_embedded: Mix.env == :prod,
   start_permanent: Mix.env == :prod,
   ...]
end
```
上面代码的含义就是程序在```:prod```环境中运行的话，则使用那两个选项。

当你编译代码的时候，Elixir把编译产出都置于```_build```目录。
但是，有些时候Elixir为了避免一些不必要的复制操作，
会在```_build```目录中创建一些链接指向特定文件而不是copy。
当```:build_embedded```选项被设置为true时可以制止这种行为，
从而在```_build```目录中提供执行程序所需的所有文件。

类似地，当```:start_permanent```选项设置为true的时候，程序会以“Permanent模式”执行。
意思是如果你的程序的监督树挂掉，Erlang虚拟机也会挂掉。
注意在:dev和:test环境中，我们可能不需要这样的行为。
因为在这些环境中，为了troubleshooting等目的，需要保持虚拟机持续运行。

Mix默认使用```:dev```环境，除非在执行测试时需要用到```:test```环境。
环境可以随时更改：
```
$ MIX_ENV=prod mix compile
```

或在Windows上：
```
> set /a "MIX_ENV=prod" && mix compile
```

## 1.5-探索

关于Mix，内容还有很多，我们在编写这个工程的过程中还会陆续接触到一些。
详细信息可以参考[Mix的文档](http://elixir-lang.org/docs/stable/mix)。

记住，你可以使用mix的帮助信息来帮助理解一些任务的操作方法，如：
```
$ mix help TASK
```
