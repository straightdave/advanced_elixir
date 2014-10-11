1-Mix简介
=========
[第一个应用程序](#11-%E7%AC%AC%E4%B8%80%E4%B8%AA%E5%BA%94%E7%94%A8%E7%A8%8B%E5%BA%8F)  
[工程的编译](#12-%E5%B7%A5%E7%A8%8B%E7%9A%84%E7%BC%96%E8%AF%91)  
[执行测试](#13-%E6%89%A7%E8%A1%8C%E6%B5%8B%E8%AF%95)  
[环境](#14-%E7%8E%AF%E5%A2%83)  
[探索](#15-%E6%8E%A2%E7%B4%A2)  

在这份指导手册中，我们将学习创建一个完整的Elixir应用程序，以及它自己的监督树、配置、测试等等内容。

这个程序是一个分布式的键-值数据库。我们会把键-值存储在“桶”中，分布存储到多个节点。我们还会创建一个简单的客户端工具，让我们可以连接任意一个节点，并且能够发送类似以下的命令：
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

为了编写这样的程序，我们将用到一下三个主要工具：
* OTP是一个随Erlang发布的代码库的集合。Erlang开发者使用OTP来创建健壮的、高度容错的程序。在本章中，我们将来探索与Elixir整合在一起的OTP，包括监督树、事件管理等等；
* Mix是随Elixir发布的构建工具，用来创建、编译、测试你的应用程序，还可以用来管理依赖等；
* ExUnit是一个随Elixir发布的单元测试工具

本章我们就是要用Mix来创建我们第一个工程，探索OTP，Mix以及ExUnit的各种特性。

>注意：本手册需要Elixir v0.15.0或以上。你可以使用命令```elixir -v```查看版本。如果需要，可以按照《Elixir入门》第一章节安装最新的版本。

##1.1-第一个应用程序

当你安装Elixir时，你不仅得到了```elixir```，```elixirc```和```iex```命令，还得到了一个可执行的Elixir脚本叫做```mix```。

从命令行输入```mix new```命令来创建我们的第一个工程。我们需要传递工程名称作为参数（在这里，```kv```），然后告诉mix我们的主模块的名字是全大写的```KV```，否则按照默认，mix会创建一个主模块，名字是第一个字母大写的工程名称（```Kv```。因为K和V在含义上是平等关系，所以最好是都大写）。
```
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

>注意：Mix是一个Elixir可执行脚本。这意味着，你要想用mix为名直接调用它，需要提前将Elixir目录放进系统的环境变量中。否则，你需要使用elixir来运行mix：
```
$ bin/elixir bin/mix new kv --module KV
```
你可以用```-S```选项来执行，它不管你有没有把Elixir目录加入环境变量：
```
$ bin/elixir -S mix new kv --module KV
```

##1.2-工程的编译

一个名叫```mix.exs```的文件会被自动创建在工程目录中。它的主要工作是配置你的工程。看看它的内容（略去注释）：
```
defmodule KV.Mixfile do
  use Mix.Project

  def project do
    [app: :kv,
     version: "0.0.1",
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

我们的```mix.exs```定义了两个公共函数：```project```，它返回工程的配置信息如工程名称和版本；```application```，它用来生成应用程序文件。

还有一个私有函数叫做```deps```，它被```project```函数调用，里面定义了工程的依赖。不一定非要把```deps```定义为一个独立的函数，但是这样做可以使工程的配置文件看起来更整齐。

Mix还生成了文件```lib/kv.ex```，内容是个简单的模块定义：
```
defmodule KV do
end
```

以上这个结构就足以编译我们的工程了：
```
$ mix compile
```
将生成：
```
Compiled lib/kv.ex
Generated kv.app
```

注意文件```lib/kv.ex```被编译，生成了文件```kv.app```。这个```.app```文件是根据配置文件```mix.exs```中```application/0```函数的信息编译出来的。我们之后会详细介绍配置文件。

一旦工程被编译成功，便可以从工程目录启动一个```iex```会话来执行：
```
$ iex -S mix
```

## 1.3-执行测试

Mix还生成了合适的结构用以测试我们的工程。在```test```目录中，测试文件一般以```<filename>_test.exs```模式命名，每一个<filename>对应一个```lib```目录中的文件名。我们以及有了测试```lib/kv.ex```所需的```test/kv_test.exs```文件。目前它几乎什么也没做：
```
defmodule KVTest do
  use ExUnit.Case

  test "the truth" do
    assert 1 + 1 == 2
  end
end
```

需要注意几点：
1. 测试文件使用的扩展名(.exs)意为Elixir脚本文件。这很方便，我们不用为每次测试文件的改动而编译一次。
2. 我们定义了一个测试模块名为```KVTest```，它用```ExUnit.Case```来注入测试API，并使用宏```test/2```定义了一个简单的测试；

Mix还生成了一个文件名叫```test/test_helper.exs```，它负责设置测试框架：
```
ExUnit.start
```

每次Mix执行测试时，这个文件将自动被导入（required）。执行测试，使用命令```mix test```：
```
Compiled lib/kv.ex
Generated kv.app
.

Finished in 0.04 seconds (0.04s on load, 0.00s on tests)
1 tests, 0 failures

Randomized with seed 540224
```

注意在运行```mix test```时，Mix会重新编译源文件，再次生成新的应用程序。这涉及到Mix的执行环境，我们稍后章节会详细介绍。

另外，ExUnit为每一个成功的测试结果打印一个点，它还会自动随机安排测试顺序。让我们把测试改成失败看看。
修改```test/kv_test.exs```里面的断言，改成：
```
assert 1 + 1 == 3
```

现在再次运行```mix test```（注意，没有编译行为发生）：
```
1) test the truth (KVTest)
   test/kv_test.exs:4
   Assertion with == failed
   code: 1 + 1 == 3
   lhs:  2
   rhs:  3
   stacktrace:
     test/kv_test.exs:5

Finished in 0.05 seconds (0.05s on load, 0.00s on tests)
1 tests, 1 failures
```

为每一个测试失败，ExUnit打印一个详细的报告，包含了测试名称，失败的代码，失败断言中==号的左值和右值。

在错误提示的第二行，测试名称右边，是测试定义的位置。将这个位置作为参数给```mix test```命令，将仅仅执行那段测试代码：
```
$ mix test test/kv_test.exs:4
```

最后是关于错误的追踪栈信息，给出关于测试的额外信息，包括测试失败的地方，还有原文件中产生失败的代码位置等。

## 1.4-环境

Mix支持“环境”的概念。它允许开发者为某些场景定义不同的编译或是其它动作。默认地，Mix理解三种环境：
* ```:dev``` - Mix的默认环境（编译等操作）
* ```:test``` - ```mix test```使用的环境
* ```:prod``` - 用来将应用程序发布到产品环境

>注意：如果你向工程中添加了依赖，它们都不会使用你工程的环境，而是使用它们的```:prod```环境的设置！

默认情况下，这些环境行为都差不多，我们目前为止写的配置文件对它们都有效。
为某个特别的环境定义不同的配置，需要编写```mix.exs```文件中的```Mix.env```函数，它会以原子形式返回当前的环境：
```
def project do
  [deps_path: deps_path(Mix.env)]
end

defp deps_path(:prod), do: "prod_deps"
defp deps_path(_), do: "deps"
```

Mix默认使用```:dev```环境，除非在执行测试时使用的是```:test```环境。环境可以随时更改：
```
$ MIX_ENV=prod mix compile
```

## 1.5-探索

关于Mix，内容还有很多，我们在编写这个工程的过程中还会陆续接触到一些。可以参考[Mix的概览](http://elixir-lang.org/docs/stable/mix)。

记住，你可以使用帮助来获取你需要的信息，如：
```
$ mix help TASK
```

