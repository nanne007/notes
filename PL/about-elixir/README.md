## 关于 Elixir


### 异常处理

### Elixir 中的 try-catch

Elixir  有三种异常类型：

- `:error`, 由 `Kernel.raise` 产生
- `:throw`, 由 `Kernel.throw/1` 产生
- `:exit`, 由 `Kernel.exit/1` 产生

try-catch 中的子句中，

```elixir
try do
   do_something_that_may_fail(some_arg)
rescue
  x in [ArgumentError] ->
    IO.puts "Invalid argument given"
catch
  class, value ->
    IO.puts "#{class} caught #{value}"
else
  value ->
    IO.puts "Success! The result was #{value}"
after
  IO.puts "This is printed regardless if it failed or succeed"
end
```

- `rescue` 部分只能用来处理 `:error` 类型，
  常见的包括： `RuntimeError`、`ArgumentError`、`ErlangError`，也可通过 `Kernel.defexception/1` 自定义类型。
- `catch` 部分对这三种类型都可以处理，类型通过`class` 进行绑定（未指定 class 时，默认是 `:throw`）。
- `else` 部分在未出错的情况下执行。
- `after` 部分无论什么情况都会执行。

### Erlang 中的 try-catch

Erlang 有两个表达式来处理，一个是 `catch`，另一个是 `try-catch`。

#### Catch me

`catch` 将表达式中产生的三种异常转换成信息。

- `error` => {'EXIT', {Reason, Stacktrace}}
- `exit(Term)` => {'EXIT', Term}
- `throw(Term)` => Term

``` erlang
%%% catch EXPR
catch 1+a. % => {'EXIT',{badarith,[...]}}
catch exit(invalidargs) %=> {'EXIT', invalidargs}
catch throw(1). % => 1
```

#### Catch me and distinguish me

`try-catch` 是增强版的 `catch`， 可以拿到异常的类型。

``` erlang
%%% try-catch
try EXPR
catch
  CLASS:EXCEPTION ->
    %% do something with the exception.
end
```

可以看出，Elixir 和 Erlang 中的 try-catch 基本是相对应的，Erlang 中的 `catch` 没有对应的 Elixir 版本。
这是可以理解的，毕竟用到 `catch` 的地方一般都可以用 `try-catch` 代替。


### 进程退出

总结一下 __进程退出(proccess exiting)__ 这个话题。

如下我站在进程 A 的的视角解释它会以哪一种方式狗带。

#### 进程内部

从进程自身来说：

- 第一种是自然死亡：进程执行完（或者显式的调用 `exit(:normal)`），留下 `:normal` 消息。
- 其次就是自杀：进程调用 `exit(:kill)` 方法结束自己的生命，留下 `:kill` 消息。
- 最后是生病死亡：进程在工作的过程中，不小心出错了，比如：`ArgumentError`、`ArithmeticError`，留下的消息就是这些错误信息。
  （死前也可以显示地调用 `exit(other_reason)` 来说明自己死去的原因）。

#### 进程外部

外部环境也可以通过 `Process.exit/2` 方法来杀死进程 A：

- `exit(pid, :normal)` 对 A 来说是无效的，毕竟任何外部因素无法使一个人自然死亡。
  （如果调用这个方法的是 A 自己，就和从进程内部显示调用 `exit(:normal)` 的效果一样）
- `exit(pid, :kill)` 能够成功杀死 A，A 会留下消息 `killed`。
- `exit(pid, other_reason)` 同样也会将病传给了 A，致其死亡，A 留下消息 `other_reason`。

内部和外部导致进程 A 死亡的情况差不多，要么自然死亡，要么染病死亡，在要么被人暗杀或者自杀。

#### 进程防御

对于外部环境的暗杀，进程 A 是有自己的防御措施的。
A 通过 `Process.flag(:trap_exit, true)` 给自己加上 `trap_exit` 防御后，

- 受到外部的 `:normal` 攻击时，A 不但不会受到任何伤害，还会记录该攻击的具体情况（`{:EXIT, _from_who, :normal}`）。
- 受到外部的 `:kill` 攻击时，A 防御失败，最终死亡。
- 受到外部的 `other_reason` 攻击时，A 防御成功，也会记录该攻击的情况（`{:EXIT, _from_who, _other_reason}`）。

以上是我理解的进程退出。有任何遗漏或者错误，欢迎讨论。


那么问题来了， `exit(:kill)` 和 `Process.exit(pid, :kill)` 有什么区别？
想清楚这个问题， __Process Exiting__ 就尘埃落定了，具体可参见 [This PR in Erlang/OTP](https://github.com/erlang/otp/pull/854)。


### Macro 使用

第一次用 Elixir 写代码，拿 [stockfighter](https://github.com/lerencao/stockfighter) 练手。
期间遇到一个编译错误，总是提示 `__struct__ is not defined`，
排查了一会，发现是 `use HTTPoison.Base` 引起的。

在它的 `__using__` 中发现了这样一行代码： `defoverridable Module.definitions_in(__MODULE__)`。
我的 module 中使用了 `defstruct`，
所以这行代码会使得 module 中 `__struct__` 方法（用 `defstruct` 时会编译出该方法）被惰性定义，
用 `%__MODULE__{}` 作 pattern matching 会出现上面提到的错误。

起初我很自然地认为这行代码是罪魁祸首，就给 httpoison 提了一个 [PR](https://github.com/edgurgel/httpoison/pull/110)。
后来再审视的时候，突然想起来，Elixir 中，macro 的使用顺序是很重要的。
在 using module 之前使用 `defstruct`，`__struct__` 会被 overridable，
但如果是在之后的话，`defoverridable Module.definitions_in(__MODULE__)` 这行代码在执行时，
`__struct__` 还不存在，就不会被 overridable 了。
试了试，发现问题确实出在这里。


``` elixir
defmodule FooBar do
  defmacro __using__(_) do
    quote do
      defoverridable Module.definitions_in(__MODULE__)
      Module.definitions_in(__MODULE__) |> IO.inspect
    end
  end
end

defmodule MyModule do
  use FooBar # it should be used first
  defstruct bar: nil
  def foo(%__MODULE__{bar: bar}), do: bar
end
```

### Plug 源码

[Plug](https://github.com/elixir-lang/plug)

从 `Plug` 开始，`Plug` 本身是一个 `Application`，里面定义了 `start` 回调方法。
同时，它还是一个 *behaviour*，用于实现各种各样的 Web 模块。


作为 application 时，`Plug` 依托于 `Plug.Supervisor`，它做了两件事情：

* 启动 `Plug.Upload`，用于管理文件上传。
* 另外定义了 `Plug.Key` ets。

#### 作为 `Behaviour`

`Plug.Conn`: *unset* ---> (*set* | *file*) ---> *sent* | *chunked*

- `put_status`: ---> *sent*
- `resp`: unsent(*unset* | *set*) ---> *set*
- `send_resp`: *set* --> *set* => `run_before_send` => `adapter.send_resp` --> *sent*
- `send_file`: unsent([`unset`, `set`]) --> *file* => `run_before_send` => `adapter.send_file` --> *sent*
- `send_chunked`: unsent --> *chunked* => `run_before_send` => `adapter.send_chunked` --> *chunked*
`chunk`: *chunked* => `adapter.chunk`
