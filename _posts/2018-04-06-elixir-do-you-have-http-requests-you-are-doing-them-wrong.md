---
layout: post
title: 'Elixir: Do you have HTTP requests? You are doing them wrong!'
tags: [Elixir, HTTP, Performance, Security]
---

The process of making HTTP requests in Elixir seems to be obvious for every developer:
one should take [**HTTPoison**](https://hex.pm/packages/httpoison) (3,5M downloads from **Hex.pm**), and do whatever he wants!
But... Have you ever thought about alternatives? If not - follow this article,
and get the answer to the question:

> How to make HTTP request?
<!--more-->

## Why Elixir ships without HTTP client?

It is obvious: because **Erlang/OTP** has HTTP client included!
And you can call it from any **Elixir** application with easy,
using **Erlang** modules [as atoms](https://elixir-lang.org/crash-course.html#calling-functions).

This client is called **httpc**. It's simple enough, easy usable
and don't require modifications of your **mix.exs**.

You can find how to use in [docs](http://erlang.org/doc/man/httpc.html).

I'll show you small example of **get** HTTP request:

```elixir{% raw %}
# First, we should start `inets` application.
# `httpc` is part of it:
Application.ensure_all_started(:inets)

# We should start `ssl` application also,
# if we want to make secure requests:
Application.ensure_all_started(:ssl)

# Now we can make request:
{:ok, {{'HTTP/1.1', 200, 'OK'}, _headers, _body} =
  :httpc.request(:get, {'http://google.com', []}, [], [])

# `httpc` will follow redirect from `http` to `https` without
# additional config.
{% endraw %}```

Thus, why people don't use `httpc`? There are several reasons:

* **HTTPC** doesn't have caching system
* It doesn't use connection pools
* It has no streaming out of the box
* It doesn't ship with **MIME** types support
* ...
* Well. Fairly saying, it has nothing in it. It's dumb simple.

But! The question is:

> Do I need all this stuff in my project? Or my requests are **really** simple?

Let's for example look at very popular **Hex** package - [**tzdata**](https://hex.pm/packages/tzdata). It has only one dependency - **Hackney**.
Let's investigate a bit, how it's used.

We can [search](https://github.com/lau/tzdata/search?utf8=%E2%9C%93&q=%3Ahackney&type=)
repo by `:hackney` query, and find all places where it is used.
Except for `mix.exs` and `mix.lock`, it's used in only one place: **Tzdata.DataLoader**.

DataLoader is used to retrieve new timezone data. This data is stored in **tar.gz**
archive on remote server, and is fetched via `:hackney.get` method.

***

**tzdata/data_loader.ex**

```elixir
def download_new(url \\ @download_url) do
  Logger.debug("Tzdata downloading new data from #{url}")
  set_latest_remote_poll_date()
  {:ok, 200, headers, client_ref} = :hackney.get(url, [], "", [follow_redirect: true])
  {:ok, body} = :hackney.body(client_ref)
  content_length = byte_size(body)
  {:ok, last_modified} = last_modified_from_headers(headers)

  new_dir_name =
    "#{data_dir()}/tmp_downloads/#{content_length}_#{:random.uniform(100_000_000)}/"

  File.mkdir_p!(new_dir_name)
  target_filename = "#{new_dir_name}latest.tar.gz"
  File.write!(target_filename, body)
  extract(target_filename, new_dir_name)
  release_version = release_version_for_dir(new_dir_name)
  Logger.debug("Tzdata data downloaded. Release version #{release_version}.")
  {:ok, content_length, release_version, new_dir_name, last_modified}
end
```

***

What do you think, can you fetch one small file in a single day, using
**httpc**? I can!

Fairly saying, I don't know why **tzdata** maintainers are using **Hackney**
for this task. May be they know something, that I don't know.
But this is a good example of a place in code, where you can think about what to do - get side package with bench of dependencies, or use Erlang's *batteries*, because it's just **enough**.

## HTTPC alternatives

As we know now, **HTTPC** has one big benefit - it's included into **OTP**, and a lot of limitation.
So, great developers created there own HTTP clients, and shared them into open source.
In **Hex** registry we can find some good examples of HTTP client alternatives.
Two gigants:

* [**Hackney**](https://hex.pm/packages/hackney)
* [**IBrowse**](https://hex.pm/packages/ibrowse)

and banch of specific clients:

* [**fusco**](https://hex.pm/packages/fusco)
* [**gun**](https://hex.pm/packages/gun)
* [**lhttpc**](https://hex.pm/packages/lhttpc)

All of them have almost the same user experience and differs in bench of small and very useful *features*.

We'll not cover small libraries in this article, and probably in our projects because of simple **open source law**:

> With big popularity comes big quality

This quality comes from common cookbooks, solutions, documentation and issues on **Github**, and answered questions on **StackOverflow**.

That's why, lets try to compare a bit our gigant!

### Performance

The main thing, that is important for developer - library performance. Everybody loves benchmarks (even if they don't know what to do with them)!

I've found interesting repo - [httpcbench](https://github.com/ransomr/httpcbench),
which has some information about performance comparison of main HTTP clients.

I'll bring the **Results** table here:

***

| Client | runtime | wall_clock | mem | failures |
| ------ | --:| --:| ---:| --------:|
| hackney (default pool) | 38560 | 30912 | 16.110 | 0 |
| httpc | 34080 | 27913 | 54.083 | 0 |
| httpc (optimized) | 33540 | 26981 | 55.402 | 0 |
| ibrowse | 212720 | 112853 | 14.567 | 0 |
| ibrowse (optimized) | 22410 | 21029 | 59.849 | 0 |
| lhttpc | 27820 | 29276 | 12.893 | 0 |

***

Unfortunately, the repo's last commit was mad 3 years ago. But it's not a problem for us!
We can fix these test with all modern versions of these libraries.

You can find rewritten into **Elixir** code [in this repo](https://github.com/Virviil/ex_http_bench).

I'll bring this table also:

***

| Client                 | Runtime | Wall clock | Memory (MB) | Failures |
|------------------------|---------|------------|-------------|----------|
| Hackney (default pool) | 59691   | 50888      | 14.837      | 0        |
| HTTPC                  | 47906   | 42531      | 27.059      | 0        |
| HTTPC Optimized        | 50802   | 45300      | 28.631      | 0        |
| LHTTPC                 | 46361   | 38610      | 80.735      | 0        |
| IBrowse                | 58724   | 49948      | 15.141      | 0        |
| IBrowse (optimized)    | 58199   | 49791      | 11.781      | 0        |

***

Of course all benchmarks are synthetic. But they show, that there is almost no difference between our libraries.

**Conclusion**: they both performs almost the same. And we move next.

### Security

The second (and sometimes - the first) parameter - security. We should be sure, that our requests goes to right domains, can't be penetrated, nobody can steal our tokens and keys.

Here comes SSL comparison.

There is [great article about security](https://blog.voltone.net/post/7), and no sense to say something more.

**Conclusion**: Only **Hackney** validates **SSL** by dafault, using **certifi** library, but you can use this library in every other HTTP client. But:

* you can forget to do this
* you should create additional amount of code by yourself to implement this.

So, from security point of view, **Hackney** looks the most considered.

### Secondary parameters

Fairly saying, **Hackney** has some *secondary* advantages:

* 3.5M on **Hex** versus **IBrowse**'s 0.5M
* Binary strings with Unicode support, which is more common in **Elixir**, against charlists, that are more common in **Erlang** world
* **Hackney**'s last release was 1 day ago versus **IBrowse**'s previous year release (don't blame me for falsifications with this article's posting time :) )

### Results

Both **IBrowse** and **Hackeny** are great instruments to make good HTTP communication for your application. But, as for me, I'll choose **Hackney** for these reason:

* I have 5 time bigger chance to have it already installed in my project's deps.
* I don't need to think about HTTP client's security
* ...

No more reasons. Really... But these two should be enough!

## What about pure Elixir implementations?

I don't want to disappoint you, but there is no famous **Elixir** HTTP client.
All **Elixir**'s *HTTP client* packages are just wrappers for **Erlang** written
libraries. For example

* **HTTPoison** - is based on **Hackney**, and inspired by...
* **HTTPotion**, which is based on **IBrowse**
* **Tesla** - has adapters for both **Hackney** and **IBrowse**, and even to **httpc**
* **simplehttp** (you've never heard about it, yeah?) - based only on **httpc**

After getting this information, one should ask himself:

> Why should I wrap **Erlang** library into **Elixir** library, when I can call it directly?

This question have several possible answers:

* Wrapper makes the using experience smoother
* Wrapper brings additional functionality

Let's observe these reasons one by one.

### Smoother experience

When one says:

> This library makes my coding experience smoother

in terms of **Erlang** and **Elixir**, the meaning of this statement is:

1. **_Native_ calls.** In Elixir, we have common way to call functions, that looks something like:
    ```elixir
    MyNamespace.MyModule.my_function(arg_1, ..., arg_n)
    ```

    As you see, **MyModule** is **CamelCased** word, that can be prefixed with **MyNamespace** (also **CammelCased**), followed by function name with function arguments in brackets.

    In comparison, **Erlang** functions will be called somthing like this:
    ```elixir
    :erlang_module.my_function(arg_1, ..., arg_n)
    ```

    **Elixir** modules can be aliased, imported and required:

    * **alias** - simplifies module calles by removing namespace prefixes
    * **import** - imports module's functions in current scope
    * **require** - brings _macroses_ from module into current scope

    You can also import **Erlang** modules
    ```elixir
    iex> import :erlang
    iex> time() # here goes :erlang.time() call
    ```
    while _requiring_ and _aliasing_ has no sense here.

    **Erlang** calls even have autocomplition in **IEx**, the same as **Elixir** functions
    ```elixir
    iex> :timer.sl[Tab]
    # goes to
    iex> :timer.sleep
    ```

    **Conclusion**: There is **almost no** difference between these calls, so the experience should be almost the same.

1. **Documentation**. During development process, you can get documentation for your **Elixir** functions in two common different ways:

    * Using **IEx**, one can call `h/1` function to get documentation on given function:
        ```elixir
        iex> h Enum.at
        iex(7)> h Enum.at
                          def at(enumerable, index, default \\ nil)

            @spec at(t(), index(), default()) :: element() | default()

        Finds the element at the given index (zero-based).
        ...
        ```
    * This functionality can be included into you **IDE**.

    **None** of this works for **Erlang** functions for the moment. You can be lucky and get **spec**,
    but also not for every function:

    ```elixir
    iex> h :timer.sleep
                                      :timer.sleep/1

        @spec sleep(time) :: :ok when Time: timeout(), time: var

    Documentation is not available for non-Elixir modules. Showing only specs.
    ```

    This can be a big problem, when you shouldn't go googling with **Elixir** library, but need to do this for **Erlang** one. But this seems not to be working for wrappers.

    For example, this is the line in **HTTPosion** documentations:

    > :ssl - SSL options supported by the ssl erlang module

    Well, this mean that developer should go into **Erlang**'s `ssl` module documentation to understand what the parameters should be used in **Elixir**'s **HTTPoison** library.

    And both **HTTPoison** and **HTTPotion** lay on docs to **Hackney** and **IBrowse**, thus to make something more complicated then simple `get` you should read there docks also.

    **Conclusion**: in both versions you probably will go into google or docks via Internet, so the experience should also be **almost the same**.

### Additional functionality

Our **Elixir** wrappers comes with some portion of additional functionality:

* **HTTPoison** and **HTTPotion** wraps _errors_ into **Elixir** structs.
* They both also bring some _metaprogramming magic_ to build **API wrappers**
* **Tornado** entirely changes the way of building request and wrappers.

The question is the same again:

> Do I need all this stuff in my project?

Let's also make some investigations upon bench of libraries from **Hex**. We will do the same process, as with **tzdata**:
* Search for **HTTPoison** calls in repo.
* Analyze, can it be simplified by pure **Erlang**'s **Hackney** calls

1. [**Recaptacha**](https://hex.pm/packages/recaptcha) - performs communication with **recaptcha**'s servers to make captcha validation.
    * Search results can be find [here](https://github.com/samueljseay/recaptcha/search?utf8=%E2%9C%93&q=HTTPoison&type=). As we see, there is only **one** HTTP request (**POST**).
    * We can change `HTTPoison` into `:hackney` just in code with almost zero remaking. **HTTPoiso.Base** is not used here, **HTTPoison.Error.t()** - also. May be we even can use **:httpc** here, what do you think about that?
1. [**random_user_api**]() - communicates with server to get _random user_.
    * Search results can be find [here](https://github.com/PatNowak/random_user_api/search?utf8=%E2%9C%93&q=HTTPoison&type=). As we see, there is only **one** HTTP request (**GET**).
    * The same as previous library, nothing to add.
1. [**pubnux**]() - communicates with **PubNub** server.
    * Search results can be find [here](https://github.com/Liftitapp/pubnux/search?utf8=%E2%9C%93&q=HTTPoison&type=). It's really hard to find library with **HTTPoison.Base**, as you see :)
    * The same as previous, no sense to continue here anymore.
1. And so on, and so on... You can investigate them by yourself using for example [this list](https://hex.pm/packages?search=depends%3Ahttpoison) (yes, **HTTPoison** dependents).

## Conclusion

People are very lazy creatures. And software developers are the laziest people :)

It's very easy to bring **ten** dependencies into your project just to make single HTTP request.
**Harder** - to deal with possible **depndency hell**, when your project grows.
**Harder** - to track and support in **OSS** **N+1** library then **N**...

I think, that after reading this article, you will think a bit before planning your **API** communication. And this will simplify your development experience bigger, that _no use of erlang ibraries in elixir_.


And if you need concrete solutions, here they are:

1. If you need to make single call in your small pet project, use **:httpc**. It's _enough_ to do the task, while not bringing a lot of deps in code.
1. If you develop your own **API wrapper**, either for your project use, or for publishing in **Hex**, **Hackney** should be just _enough_ to do this. Moving from **Elixir** wrappers to pure library will reduce your deps folder, and will help to avoid _adapter bugs_.

## Acknowledgments

Special thanks to **maintainers** and **contributors** of dissected libraries. They are doing really great job and bring our **Elixir** community higher.
