---
layout: post
title: 'Elixir: Brackets Hell in Supervision tree'
tags: [Elixir, OTP, Supervisor, GenServer, Erlang]
---

![Supervision tree](https://4everinbeta.files.wordpress.com/2015/01/supervisiontree.png)

The `Supervisor.Spec` module was deprecated and does not work with the module-based child specs,
introduced in **Elixir v1.5**. Thus, all methods for supervision tree declaring were sugnificantly changed.
It's great time to understand the _waterflow_ of passing arguments -
from top-level `Supervisor` to low-level workers aka `GenServers`.
<!--more-->

## The problem

I faced the problem with declaring arguments in **Supervisor -> GenServer** chain.
For example, look at the **Elixir's Supervisor**
[docks](https://hexdocs.pm/elixir/Supervisor.html#module-child-specification). You can find this code there:

```elixir
Stack.child_spec([:hello])
#=> %{
  id: Stack,
  start: {Stack, :start_link, [[:hello]]},
  restart: :permanent,
  shutdown: 5000,
  type: :worker
}
```

What the hack is going on here **o_O**!? Why `:hello` is wrapped into _single brackets_ in the first line,
but in _double brackets_ in the forth line?

I'm absolutely not sure how to pass, for example, `:ok` into `GenServer` - using `:ok`,
or maybe `[:ok]`, or maybe `[[:ok]]`...

How to match this value in `start_link`, `chils_spec` and `init` functions?..

If you feel the same - let's dive into this _brackets hell_ together.

## init/1 has arity 1 (_ONE_).

I hope that the good idea is to start from the ground - from the function,
when you **really** understand what to do with this arguments.
The only function with such parameters is function `init/1` in `GenServer`.

For sure, you are initializing your worker's state with passed arguments:

```elixir
def init(_something_) do
  state = ...
  # some arguments conversions
  {:ok, state}
end
```

The main idea here, and the only thing you have to learn by heart:

> init/1 has arity 1!

If you need more than one argument - you have to wrap them into some **data structure**,
so it can be passed into function with arity _1_.

Here comes two main errors:

* You have only one argument (for example, you done have arguments at all, or you want to pass `true` or `:ok`)

  ```elixir
  def init([true]) do
    {:ok, %{}}
  end
  ```
  Don't you think, that the brackets here are unnecessary? I'm sure, that they do!
  Brackets here have no sense, so - just don't use them!

* You have arguments, that already wraped into `tuple`, `map` or `list`.

  ```elixir
  map = %{name: "Joe", age: 21}
  tuple = {"Joe", 21}
  list = ["Joe", 21]
  ```
  And you are trying to wrap them into brackets?
  ```
  def init([%{name: "Joe", age: 21}]) do
    {:ok, %{name: "Joe", age: 21}}
  end

  def init([{"Joe", 21}]) do
    {:ok, {"Joe", 21}}
  end

  def init([["Joe", 21]]) do
    {:ok, ["Joe", 21]}
  end
  ```

  Pretty nice picture... With tones of unnecessary brackets!

One more thing to say: `use GenServer` brings us `init\1` function, defined like this:

```elixir
def init(state) do
  {:ok, state}
end
```

Basicly saying, it's the only one reason to add **one** argument in **brackets**:

* you **want** your **GenServer's state** to be **list** with **one** element after initialization
* you **don't want** to **redefine** `init/1`

And to conclude:

* when you see `GenServer` that doesn't have `init/1` defined in the code - you know how this `init` is done.
* if your `init/1` does the same as predefined `init/1` - don't override it manualy in the code!

Don't use brackets just because you've seen brackets in the documentation - probably,
the example there is not so good. Try to think your own head - and continue reading.

## GenServer.start_link - you already know what to do!

If one want's to pass the data into `init/1` function of your GenServer module -
he should start the server via `GenServer.start_link/3`  function. Let's look at it's spec:

```elixir
start_link(module: atom, args: any, options: Keyword.t) :: on_start
```

Well, here everything is easy - you have only *one* place for args,
and this args will pass into `init/1` function with arity **one**.
You should learn nothing - you already know what to do!

Let's look at some examples:

```elixir
# Good

GenServer.start_link(__MODULE__, true)
def init(true), do: ...

GenServer.start_link(__MODULE__, %{name: "Joe", age: 21})
def init(%{name: "Joe", age: 21}), do: ...

GenServer.start_link(__MODULE__, {"Joe", 21})
def init({"Joe", 21}), do: ...

GenServer.start_link(__MODULE__, ["Joe", 21])
def init(["Joe", 21]), do: ...

# Never Ever Do This at Home

GenServer.start_link(__MODULE__, [true])
def init(true), do: ...
** (FunctionClauseError) no function clause matching

GenServer.start_link(__MODULE__, [true])
def init([true]), do: ...
# 4 unnecessary brackets...

GenServer.start_link(__MODULE__, [{"Joe", 21}])
def init({"Joe", 21}), do: ...
** (FunctionClauseError) no function clause matching

GenServer.start_link(__MODULE__, [["Joe", 21]])
def init([["Joe", 21]]), do: ...
# C'mon, are you kidding me?
```

Yep, as you see - it's very easy. But the hard part is coming...

## \__MODULE__.start_link - what to do here?

The obvious place to call `GenServer.start_link/3` in your **GenServer**'s code - define your own `start_link` function.

First, lets look at the functions that come with `use GenServer` macro calling into our **GenServer**:

```elixir
iex> defmodule Foo, do: use GenServer
iex> Foo.__info__(:functions)
[child_spec: 1, code_change: 3, handle_call: 3, handle_cast: 2, handle_info: 2, init: 1, terminate: 2]
```

As we see, `use GenServer` have brought many functions to the module,
but `start_link` is not in the list. So, we have to define it by ourselves.

Oh, yes! Finally we got a place to play with arity. We can define `start_link` with as many arguments as we want!

```elixir
# Just example - kick off developer that will do this in real code
def start_link(_a,_b,_c, ... _z) do # Up to 255 - maximum Erlang arity
  GenServer.start_link(__MODULE__, true)
end
```

Then we can call this `start_link/255` function from our code:

```elixir
{:ok, pid} = MyGenServer.start_link(1,2,3,4,...255)
```

Well... Why you can need to define `start_link` with arity more then **1** in real life?
We'll not discuss neither "Do I really need this?", nor "Is this pattern or antipattern?", just bench of examples:

* you want to define dynamic name:

  ```elixir
  def start_link(init_arg, name) do
    GenServer.start_link(__MODULE__, init_arg, name: name)
  end
  ```

* manipulate with `module`, that defines callbacks for your **GenServer**:

  ```elixir
  def start_link(dynamic_module_name, init_arg) do
    GenServer.start_link(dynamic_module_name, init_arg)
  end
  ```

* do something else with opts:

  ```elixir
  def start_link(init_arg, opts) do
    GenServer.start_link(__MODULE__, init_arg, opts)
  end
  ```

As you see, nowbody knows what arity will be your `start_link` function.
And this is a place where problems are beginning. This is a portal to the **brackets hell**...

## Child spec: It's dangerous to go alone! Take this

Before passing through the hellgate - we should arm ourselves with child specification knowledge.
Fairly saying, child spec did not appear in **Elixir 1.5** for the first time.
But you really had no instruments to deal with it in Elixir -
until you didn't want to dig in some crazy metaprogramming or equal.

**Child specification** is used by your **Supervisor** to understand:

* **how** to **start** his children
* and **how** to **restart** his children

In return, **Supervisor*8 has his own specification, and it configures:

* **when** to restart children (**Restart strategy**)
* and **restart frequency until suicide**

Let's look at the child spec example for previously defined crazy **Foo** module with `start_link/255`:

```elixir
%{
  id: Foo,
  start: {Foo, :start_link, [1, 2, 3, ..., 255]},
  restart: :permanent,
  shutdown: 5000,
  type: :worker
}
```

Parameters are described [here](https://hexdocs.pm/elixir/Supervisor.html#module-child-specification),
so don't think that it's nessesary to repeat that again.

You see here thee crazy brackets inside `start` tuple. So the question here is:

> Do I really need these brackets here? May be, if I have only one argument, I can ommit them?

I don't want to dissapoint you, but these brackets have special sense, and should be here mandatory.

But I'll try to explain you this brackets step by step.

### Your start_link has unknown arity

The first step is easy: `start_link` in your module has unknown arity
(unknown at the moment of inventing child specification).

So, you should have and instrument to start `start_link` dynamically - with arity 1,
or may be 2, or may be 255. And here comes well-known Erlang's approach...

### Kernel.apply

Let's look at the `Kernel.apply/3` specification:

```elixir
apply(module: atom, function_name: atom, args: [any]) :: any
```

and examples:

```elixir
iex> Enum.reverse([1, 2, 3])
[3, 2, 1]
iex> apply(Enum, :reverse, [[1, 2, 3]])
[3, 2, 1]
```

As you see, `apply/3` doesn't know how many arguments your function has.
Thats why he asks you to put these arguments into list.
Event if you have **ne** argument - this arguments still should be in brackets.
This is separately pointed in `apply/3`**typespec**.

The most crazy moment appears, when you have only *one* argument, and it is a **list** (like in the example) -
you will have double brackets, and you can do nothing with this...

Ok, let's return to our child specification. Lets take tuple under `start` key, and _apply_ it's elements as the arguments to `Kernel.apply`:

```elixir
iex> Kernel.apply(Foo, :start_link, [1, 2, 3, ... 255])
# and it is the same as...
iex> Foo.start_link(1, 2, 3, ..., 255)
```

Do you see this? Yep, **child specification** does not invent anything new -
it just _copy-pastes_ tuple into `Kernel.apply`.
Thats why you need this brackets - and nothing to say more.

Let's for a moment return to the example from the beginning:

```elixir
Stack.child_spec([:hello])
#=> %{
  id: Stack,
  start: {Stack, :start_link, [[:hello]]},
  restart: :permanent,
  shutdown: 5000,
  type: :worker
}
```

Here, as you see, `Stack.start_link/1` is expecting to get **one** argument,
and this argument is **list**. That's why we need to put `:hello` into double brackets.

To sum up, one mnemonic advise:

> If you see the **function**, that is passed by it's **name** - remove **one** pair of **brackets**!

These brackets are used to pass arguments as **list**, so the notation will omit the brackets
and put these arguments as are into function call.

## child_spec/1 - dynamic point in your worker

Do you remember functions that our `use GenServer` brings into module?

```elixir
iex> defmodule Foo, do: use GenServer
iex> Foo.__info__(:functions)
[child_spec: 1, code_change: 3, handle_call: 3, handle_cast: 2, handle_info: 2, init: 1, terminate: 2]
```

Yep, something new, which can't be found neither in Erlang,
nor in Elixir 1.4 and previous - `child_spec/1` function.

Let's try to look into it's generation's source code to understand, what is this function for:

```elixir
spec = [
  id: opts[:id] || __MODULE__,
  start: Macro.escape(opts[:start]) || quote(do: {__MODULE__, :start_link, [arg]}),
  restart: opts[:restart] || :permanent,
  shutdown: opts[:shutdown] || 5000,
  type: :worker
]

@doc false
def child_spec(arg) do
  %{unquote_splicing(spec)}
end
```

As you see - nothing hard. It's expecting that you will pass some arguments in `use GenServer` statement, to somehow change attitude of **child specification**. For example, you can do something like this:

```elixir
use GenServer, restart: :transient
```

to redefine **Restart strategy**.

The only interesting place for us here - `{__MODULE__, :start_link, [arg]}` under `start` key in *child spec*.
As you see, `arg` which we pass into `child_spec/1` function is *wrapped into brackets*
and is prepared to be pushed into `start_link/1` function, that is defined in the same module.

As we know now, these **brackets will be omited**, so the call will be:

```elixir
MyGenServer.start_link(arg)
```

Obviously, you can redefine this `child_spec/1` function. But remember, that is has arity 1.
Thus, if you want to define everything dynamically - you should wrap your data into data structure:

```
def child_spec({id, {_module, _fun, args} = start, restart, shutdown}) do
	%{
      id: id,
      ...
    }
end

# or

def child_spec([id, {_module, _fun, args} = start, restart, shutdown]) do
	%{
      id: id
      ...
    }
end

# or (this function has no sence, but as example... )

def child_spec(%{id: id, start: {_module, _fun, args} = start, restart: restart, shutdown: shutdown} = spec) do
  spec
end



# Never Ever Do This At Home

def child_spec(id, {_module, _fun, args} = start, restart, shutdown) do
	%{
      id: id
      ...
    }
end

# Everybody will try to call YourModule.child_spec/1,
# but you defined YourModule.child_spec/4.
# Your function does not override child_spec/1.
```

And now we are on the finish line...

## Supervisor wants to call your worker - but doesn't know how

Starting your *GenServers* via `start_link` from the code - not the best idea.
We have perfect **OTP** framework, which forces us to start all processes under **_supervision tree_**.
It's defined by list of workers in casual way:

```elixir
Supervisor.start_link([
  _worker1_,
  _worker2_,
  ...
  _workerN_
], opts)
```

So, defining these _workers_ has three different approaches, that all comes at the end to the **child spec**. Let's dig into them one by one:

* A map representing the child specification itself - such as the child specification map outlined in the previous section:

  ```elixir
  Supervisor.start_link([
    %{
      id: "id",
      start: {MyModule, :start_link, [true]},
      restart: :transient,
      shutdown: 500,
      type: worker
    }
  ], opts)
  ```

  As you see, here *Supervisor* even doesn't touch `YourModule.child_spec/1` function -
  it's starting supervised proces directly from the spec.
  So, even you found perfect library in *Hex*, but it's main *GenServer's* `child_spec/1` is done awefull,
  you still can adopt it into your _supervision tree_ using this approach.

* A tuple with a module as first element and the start argument as second:

  ```elixir
  Supervisor.start_link([
    {MyModule, true}
  ], opts)
  ```

  When such format is used, the **Supervisor** will retrieve the **child specification** from **MyModule**.
  Do you remember, that `child_spec/1` has arity 1? This is a place, when this arity helps:

  ```elixir
  {MyModule, true}

  # inside Supervisor initialization process this turn into

  MyModule.child_spec(true)



  # Bad example

  {MyModule, [true]}

  # inside Supervisor initialization process this turn into

  MyModule.child_spec([true])

  # and probably this is not what you want...
  ```

* A module:

  ```elixir
  Supervisor.start_link([
    MyModule
  ], opts)
  ```

  In this case, it is equivalent to passing `{MyModule, []}`:

  ```elixir
  MyModule

  # inside Supervisor initialization process this turn into

  MyModule.child_spec([])
  ```

## Supervisor.start_child - one more tricky point.

Generally saying, you have one more way to start worker under **Supervisor** - using `start_child/2`.

Let's see it's specification:

```elixir
start_child(
  supervisor: supervisor,
  child_spec_or_args: :supervisor.child_spec | [term]
) :: on_start_child
```

In specification we see interesting variable name: `child_spec_or_args`. What does that mean?
Maybe, we can start our child **either** with _child spec_, or with args?

The answer is - **NO**. And the second argument of `start_child/2` function will depends on the **Supervisor's** strategy:

* if strategy is `:simple_one_for_one` - you should pass args
* if strategy is _not_ `:simple_one_for_one` - you should pass _child spec_

Why? Let's dig into it!

### NOT :simple_one_for_one

We are defining _child spec_ at the same time when we starting the worker.
That's why we simply pass _child spec_ - and new worker starts to work!

### :simple_one_for_one

With `:simple_one_for_one` startegy, we define _child spec_ for our children at the same time,
when we initialize the **Supervisor**, but children are started _dynamicly_. Here the problems come.

At the time when the **Supervisor** is initializing, we don't know what _args_ will be required by different workers,
that will be started in the future.

But this arguments in the form of **list** can be passed using `start_child/2` function!
This function will append these arguments to the **existing** arguments in predefined _child spec_ -
just appending two lists `list1++list2`:

```elixir
# For example our childspec is defined like this:
%{
  id: "id",
  start: {MyModule, :start_link, [true]},
  restart: :transient,
  shutdown: 500,
  type: worker
}

# We don't want to start our child, so we need to override our start
# part, and put there 0 arguments:
spec = Supervisor.child_spec(MyModule, start: {MyModule, :start_link, []})

# After this manipulation, child spec inside `spec` variable will be:
%{
  id: "id",
  start: {MyModule, :start_link, []},
  restart: :transient,
  shutdown: 500,
  type: worker
}

# Then we are starting our supervisor:
{:ok, pid} = Supervisor.start_link([spec], strategy: :simple_one_for_one)
# The worker will not be started, because `start_link` with 0 args is not defined

# Now, starting worker dynamically:
Supervisor.start_child(pid, [true]) # we have brackets here, because it's list!

# New child spec will be
%{
  id: "id",
  start: {MyModule, :start_link, []++[true]}, # or simply [true]
  restart: :transient,
  shutdown: 500,
  type: worker
}
# And new worker will start as we want
```

## Let's get some examples

Just to summarize - let's follow full _args waterfall_ with interesting conditions:

* The easiest one: I have the most simple **GenServer** without any usefull state -
and just want to write as little amount of code as possible:

  ```elixir
  # In supervisor:
  Supervisor.start_link([SimpleModule], opts)

  # This will call child_spec this way:
  SimpleModule.child_spec([])

  # I am to lazy to redefine child_spec, so the spec will be:
  %{
    id: SimpleModule,
    start: {SimpleModule, :start_link, [[]]}, # double brackets, but you already know why!
    restart: :permanent,
    shutdown: 5000,
    type: :worker
  }

  # Using this child spec, Supervisor will start our server this way:
  SimpleModule.start_link([])

  # I am to lazy to think about args, so I'm deciding to bypass it into init through start_link,
  # and defined start_link like this:
  def start_link(arg), do: GenServer.start_link(__MODULE__, arg)

  # Well, I'm not redefininig init also, so it will be called like this:
  SimpleModule.init([])

  # and will start server with state:
  {:ok, []}
  ```

* I don't like name `start_link`, and want to start my **GenServer** using `star_blink`:

  ```elixir
  # In supervisor:
  Supervisor.start_link([AstronomyModule], opts)

  # This will call child_spec this way:
  AstronomyModule.child_spec([])

  # Have to redefine child_spec here, to tell supervisor how to start my module:

  def child_spec(arg) do
    %{
      id: __MODULE__,
      start: {__MODULE__, :star_blink, [arg]},
      restart: :permanent,
      shutdown: 5000,
      type: :worker
  }
  end

  # This spec will return:
  %{
    id: AstronomyModule,
    start: {AstronomyModule, :star_blink, [[]]}, # as in previous example
    restart: :permanent,
    shutdown: 5000,
    type: :worker
  }

  # Using this child spec, Supervisor will start our server this way:
  AstronomyModule.star_blink([])

  # Bypassing arg into init through star_blink:
  def star_blink(arg), do: GenServer.start_link(__MODULE__, arg)

  # init will be called like this:
  AstronomyModule.init([])

  # and will start server with state:
  {:ok, []}
  ```

* I have **list** with **one** item as argument, but I didn't read this article
and decided to add one more pair of brackets - just to be sure that everything will be all right:

  ```elixir
  # In supervisor:
  Supervisor.start_link([
    {BracketsModule, [[:element]]}
  ], opts)

  # This will call child_spec this way:
  BracketsModule.child_spec([[:element]])

  # This spec will return:
  %{
    id: BracketsModule,
    start: {BracketsModule, :start_link, [[[:element]]]}, # OH SHI~
    restart: :permanent,
    shutdown: 5000,
    type: :worker
  }

  # Using this child spec, Supervisor will start our server this way:
  BracketsModule.start_link([[:element]])

  # Bypassing arg into init through start_link:
  def start_link(arg), do: GenServer.start_link(__MODULE__, arg)

  # init will be called like this:
  AstronomyModule.init([[:element]])

  # and will start server with state:
  {:ok, [[:element]]}
  ```

  Of course, we don't want *list* in *list* as our state. So, we can try to solve the problem in the different ways:

  ```elixir
  # The only right way - change the root aka supervisor

  Supervisor.start_link([
    {BracketsModule, [:element]}
  ], opts)



  # Never Ever Do This At Home

  # Redefine child_spec:
  def child_spec([arg]) do # trying to match brackets here
    %{
      id: __MODULE__,
      start: {__MODULE__, :star_blink, [arg]},
      restart: :permanent,
      shutdown: 5000,
      type: :worker
  }
  end

  # Redefine start_link
  def start_link([arg]), do: GenServer.start_link(__MODULE__, arg)

  # Or redefine init
  def init([arg]), do: {:ok, arg}}
  ```

## Conclusion

As you see, **brackets hell** is not so terrifying! From now you don't need to restart your programs
because of silly mistakes with brackets in **Supervisors**, **Specs**, **Start_links** and **Inits**.

I'll just try to give you last ideas explicetly - may be following these rules will help you and your team with communication and clearify your code:

* One should start architecturing **from** workers **to** supervisors.
**From** branches **to** roots. After all, **workers** will do the business job - they are main.
And, as you've seen in this article - you will have **all** instruments to start your workers properly -
doesn't metter how **workers** are defined.
So, don't make _data repacking_ in `init` or `start_link` functions - pass nessesary data directly from **supervisor**

* Try to follow `KISS` - don't bring unnesessary complicity to your program.
For example, don't redefine `init` in your **GenServer** if you can simply pass it's initial state
from `start_link` function. And don't redefine `child_spec` function,
if you are creating simple **GenServer** worker - better use arguments for `__using__` macro.

* The main _thick_ point of your _args flow_ should be `child_spec` function. Even if everything is absolutely unique,
and not standart, try to make `init`, `start_link` as usual as possible from one side,
and try not to use **child spec** directly in `Supervisor.init` - from the other.
`child_spec` function can hold all the logic about **how to start my worker**

## Acknowledgments

Thanks to [@Wunsh](https://wunsh.ru/) for translating this article to Russian! You can find the translation [here](https://wunsh.ru/articles/elixir-brackets-hell-in-supervision-tree.html#subscribeModal)!

## The End
