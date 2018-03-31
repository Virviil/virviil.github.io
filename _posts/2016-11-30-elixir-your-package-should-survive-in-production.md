---
layout: post
title: 'Elixir: Your package should survive in production!'
tags: [Elixir, Deployment, Production, Distillery, Release]
---

You are very good person! Great person!
You have **just created you first (or second, or third, etc.) package** in Elixir language
and published it to the [*Hex*](https://hex.pm).
You are nice enough to follow some conventions. At least [this one](https://hex.pm/docs/publish).

The work is done, isn't it?
You've done everything as in **official tutorial™**. What problems can it lead to? "Nothing" is wrong answer...

Do you want to know why? Well... I'll try to answer this question.
<!--more-->

## Your `mix.exs` is wrong!

Of course it is. Let's find some problems!

### Build consolidation and application termination

Take a look at the function `project/0` in our **Hex** tutorial:

```elixir
def project do
  [app: :postgrex,
   version: "0.1.0",
   elixir: "0.13.0",
   description: description(),
   package: package(),
   deps: deps()]
end
```

Probably, you decided to follow this tutorial literally.
And deleted some incredibly important lines from `project/0` function.

```elixir
[build_embedded: Mix.env == :prod,
 start_permanent: Mix.env == :prod]
```

`:buid_embedded` tells our compiler to consolidate our project with all it's dependencies
in `_build` directory.
Moreover it consolidates all protocols.
It's not needed in you development environment, while all dependencies are installed on your working machine,
but imagine that you miss some of them on production server without **Elixir** or **Erlang** installation?

`:start_permanent` tells your virtual machine to terminate after your application is terminated.
This is not necessary in development, because you need your `iex` to survive even when your application dies.
But in production you should supervise your application at the point of the whole virtual machine too.
So, it's good when it crashes with your application.

What is application's behaviour in production without these lines? Nobody knows.
May be OK. May be not. Just don't forget to put them in your configuration, of course if they needed.

### What's in your `application/0`?

The second critical step. `application/0` function. In tutorial it's empty:

```elixir
def application do
  []
end
```

Is it empty in your code?
Seems to me, that no. You have there `:logger`, maybe `:httpoison` or `:ecto`,
and without these deps your application even doesn't start.

```elixir
def application do
  [applications: [:logger, :httpoison, :gproc]]
end
```

The questions are:

* Should you add `mod` of your own application?
* Or, maybe you should remove `:logger` from `:applications` list?

This is the problem for your brains. You can solve it - but you should know, that this **is** a problem.

Don't add `:mod` of your application in `application/0` function,
if the main module of your application doesn't have `start_link` or `start` functions!
That is redundantly, and can cause bad effects in the future.

And of course, fix your *README*. Don't force package users to add your application in `applications` list.
**Mix** will try to start you package before user's application, but how will it perform that without `start` function?
Nobody knows.

The answer on the second question is simple - remove everything,
that is not needed to be **started** before your application.
**Some packages** are not _startable_ - they doesn't have `start` or `start_link` functions in application module.
How will **Mix** start them? You already know the answer.

***

_**Update** (thanks to [George Guimarães](https://disqus.com/by/georgeguimaraes/))_:

**Elixir v1.4** changed the way you should list deps.
Check it out [here](https://github.com/elixir-lang/elixir/blob/v1.4/CHANGELOG.md#application-inference).
You still have to think about applications that shouldn't be started at runtime,
and list application from *OTP dist* that you use.

***

### Look at you `deps/0`!

I can bet that your package uses some dependencies.
They are defined in `deps/0` function, and of course they are up to date.
The problem is that then **next day** after you published the package **your deps are NOT up to date!**
Welcome to the reality!

You decided to use your friend Bob's package, and this is your `deps/0` function:

```elixir
defp deps do
  [{:bobs_lib, "= 0.2.1",
    :ex_doc, github: "elixir-lang/ex_doc"}]
end
```

Pretty good! But Bob unexpectedly released new bugfix with version **v0.2.2**.
And your friend Alex started to make package 2 days after you.
And this is his `deps/0` (he took data from *Hex* - no place for mistake!)

```elixir
defp deps do
  [{:bobs_lib, "= 0.2.2",
    :ex_doc, github: "elixir-lang/ex_doc"}]
end
```

What would be with **Mix**, if it's try to resolve `mix deps.get` in project with yours and Alex's packages?

Yes... you already know the answer.

So. Please. Be careful with your deps versioning.
To be familiar with them - read [here](http://semver.org/)...

But let's return to Alex's code:

```elixir
defp deps do
  [{:bobs_lib, "= 0.2.2",
    :ex_doc, github: "elixir-lang/ex_doc"}]
end
```

Does he really needs this **:ex_doc** package in release? It's just a junk in production.
It can increase deployment package size. But can't increase anything else.

So, we can use it just in deployment with:

```elixir
{:ex_doc, "~> 0.x.y", only: :dev}
```

Why don't to do that, if we can? Just be patient to your package users - don't force them
to pack all junk in there releases!

## Going deeper. Hold on: production is coming.

Today the most common way to generate releases - [Distillery](https://github.com/bitwalker/distillery).
This is easy to use, up-to-date, and almost silver-bullet library,
that helps with making a candy from your **Elixir** application.

If you think that using **Distillery** has no sense, your users **may think different**.
They will try to build there applications, fail because of your package, and come to your **Github's Issues** tab.
Ciao free time!

Of course, we can avoid this nerves.

### Distillery doesn't know your Erlang deps!

As you know, you can use Erlang modules in your code simply calling them via atom.
For example `:crypto.ec_curves`, `:observer.start` or `:calendar.local_time`.
While doing this operation, you don't list **Erlang** modules as deps in your `deps/0` function.
And this is the anthill!

**Distillery** doesn't check the code of your application trying to find **Erlang** module calls.
It just consolidates your deps from `deps` and `:applications` list.
So your modules `:crypto` and `:calendar` **would not be in release**.
Will application work? Of course not. What is the solution to this problem?

Well, you should **explicitly** put all this **Erlang** deps in your **:applications** list.
Even if you knows that they are not _startable_. This hack is dirty. But it works.
And your package also works in production.

_If you know better solution - use it and notify me in comments, so I can fix this article_

### Release doesn't have `Mix` application!

Forget about custom **Mix tasks** and all functions from **Mix** module in your code.
They **gonna fail in production**.
There are some alternatives that you can read about in
[**Distillery** docs](https://hexdocs.pm/distillery/boot-hooks.html#content).

### Your `config.exs` compiles into release!

Yes, it is.
You can't change configuration after you released your program.
For example, you have different machines for **:dev**, **:build**, **:test** and **:prod** purposes.
They all have different configuration.
But after you build the release on your build machine... fairy-tale goes bad.
Config now is **nailed** to your release. So... what to do?

Well, obviously programmers are clever people.
And you know how to deal with that problem, aren't you? **Environment variables**! And have all problems gone?... No

How do you read environments? Via `System.get_env`?
But values in **config.exs** are evaluated during the build time.
And `System.get_env` will be derived for your **current** environments!
The same with `Application.get_env`! What is the solution?

For first, don't use `System.get_env` or `Application.get_env` in your config files.
There current values will be nailed.

Other clever people decided - we will use tuples for system variables like this:

```elixir
config :my_application,
  my_key: {:system, "ENV_VAR_NAME"}
```

Thus, this tuple can be evaluated in runtime. Of course, you can configure some keys in old style way.

You can ask - but how is this connected to my package, if I don't want to use and don't use environment?
Well, look here:

```elixir
config :my_application,
  my_key: {:system, "ENV_VAR_NAME"}

config :your_library,
  your_key: {:system, "ENV_YOUR_KEY_NAME"}
```

What will be the result of `Application.get_env({:system, "ENV_YOUR_KEY_NAME"})` in release?
Yes... you know the answer.

So... what to do? **Get prepared!**. Because this approach has _backward compatibility_ - implement it in your package!
And if once somebody very clever will try to use your package as a nunchaku - it will work!
Here is the [package](https://github.com/Nebo15/confex) to help you
_(probably you can find some more with this functionality or create one by yourself)_.

And read about config in releases [here](https://hexdocs.pm/distillery/runtime-configuration.html#content).

## Conclusion

Well, my dear alchemist. I hope now you are ready to enter the **Hex.pm** hall of fame!
And don't loose your free time removing curses from users of your package!

Read this article before each `mix hex.publish` execution.

_Comment this article to help me keep it up-to date_.

Special thanks to [@noma4i](https://github.com/noma4i) for pointing the area of the problem. Big ups.
