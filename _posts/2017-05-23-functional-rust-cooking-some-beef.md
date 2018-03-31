---
layout: post
title: 'Functional Rust: Cooking some beef!'
categories: Rust
tag:
  - Rust
  - Cow
  - Esoteric languages
  - Functional programming
  - Architecture
---

![Cow logo](https://habrastorage.org/files/974/556/336/974556336a1f49a882e08295a72209c7.png){:class="img-responsive"}

One day I've stumbled upon Brainfuck-like language "Cow". And suddenly I've came up with an idea to write an interpreter for it in new hip language Rust. Rust is an multi-paradigm language, which means you have to chose in which style you want to write your code in. I've chosen functional programming.
<!--more-->

## Concept

State of our virtual [line-through]#cow# machine will be stored as immutable variable `state`. All actions will be made with functions like this:

```rust
fn change(previousState: CowVM) -> CowVM // newState
```

Looks a bit like **Elm**, **Redux**, and probably something else that I'm not aware of. Don't you think so?

The fact that we know from the very beginning how we going to store our data makes this task a good target for functional programming.

Let's look at the data our virtual machine stores:

* Program array - simple integer array;
* Memory - second simple integer array;
* Current command slot - index for value in program array;
* Current memory position - index for value in memory array;
* Register - almost simple integer value, why almost? Read further!
* ???
* **PROFIT**

From the very beginning I'm sure that at any point of development data will be store exactly like this. No additional _fields_ or _views_ will appear suddenly, no "just few little changes" from client. Living the dream!

## Keeping things immutable

There is only one way to accomplish the task of keeping everything immutable - every time when we need to change anything we will create completely new state of program, saving it and disposing of old one. To accomplish this we will use magical features of **Rust**:

* Trait **Copy**. If you came from OOP, think of structures with this traits as of _value type_ instead of _reference type_.
* Magical [struct update syntax](https://doc.rust-lang.org/book/structs.html#update-syntax). It copies fields from old structure into new one.

  **NOTE**: It doesn't actually copy fields that don't have trait **Copy** (in this case **`Vec<i32>`**).
It just moves them. And that's very important,
because you can stumble upon errors like [this](https://play.rust-lang.org/?code=%23%5Bderive(Clone%2C%20Debug)%5D%0Astruct%20Bar%20%7B%0A%20%20%20%20x%3A%20i32%2C%0A%20%20%20%20y%3A%20Vec%3Ci32%3E%0A%7D%0A%0Afn%20main()%20%7B%0A%20%20%20%20let%20a%20%3D%20Bar%7Bx%3A%201%2C%20y%3A%20vec!%5B1%2C2%2C3%5D%7D%3B%0A%20%20%20%20let%20b%20%3D%20Bar%7B..a%7D%3B%0A%20%20%20%20println!(%22%7B%3A%3F%7D%22%2C%20a.y)%3B%0A%7D&version=stable&backtrace=0).
**But** not in our case! We don't care about old structure and what happens to it.

## Model

Our code will have just one structure - the virtual cow machine itself:

```rust
#[derive(Debug, Default, Clone)]
pub struct CowVM {
    pub program: Vec<u32>,
    pub memory: Vec<i32>,
    pub program_position: usize,
    pub memory_position: usize,
    pub register: Option<i32>
}
```

Looks almost exactly like what I've described at concept phase, just adding few features -
**Debug**, **Default** and **Clone**. What it is and what it does you can read in my previous article.

## Reducer

Nothing too complicated there. Just following documentation and writing functions for every command of the language.
These functions are taking and returning virtual machine, each time creating new state.

For example let's look at very important command **mOo**  - this command moves index of current memory slot back by 1:

```rust
pub fn do_mOo(state: CowVM) ->CowVM {
    CowVM{
        memory_position: state.memory_position-1,
        program_position: state.program_position+1,
        ..state
    }
}
```

All that function does - it moves index of memory array back and moves index of program array forward.
Note that not all functions advance index of program array by 1, that's why we don't move this part to another function.
It doesn't take more space in code than calling a function anyways.

Okay, let's move to the interesting part. Remember me saying that register isn't that simple?
That's because the best (and probably the only) way to store **nullable** value in **Rust** is [**Option**](https://doc.rust-lang.org/std/option/) type.
It's made in functional manner itself, by the way.
Let's not get into [too much details](https://en.wikipedia.org/wiki/Option_type), just note that first of all,
this way is imposed by the language itself and second,
it's radically different from every language out there with **nil**, **null** or anything similar.
These languages are usually called classic OOP languages: **Java**, **C#**, **Python**, **Ruby**, **Go**...
List goes on, but it's not the point, just get used to the new order of things.

Anyways, let's get back to our register. It can be empty and can be not empty, that's why we use **Option**. And here is code for command that works with register:

```rust
pub fn do_MMM(state: CowVM) -> CowVM {
    match state.register {
        Some(value) => {
            let mut memory = state.memory;
            memory[state.memory_position] = value;
            CowVM{
                register: None,
                program_position: state.program_position+1,
                memory: memory,
                ..state
            }
        },
        None => {
            CowVM{
                register: Some(state.memory[state.memory_position]),
                program_position: state.program_position+1,
                ..state
            }
        },
    }
}
```

Do you see these 4 closing curly braces at the end?
Terrifying. That's why many functional languages aren't using braces.

While we're at it, let's not ignore this not so elegant way to change value in memory,
maybe reader can suggest something better?
Let's also note that in "pure functional" languages there are no arrays.
There are lists and dictionaries. Changing element in list takes **O(N)**, in dictionary - **O(logN)**, in our code we have **O(1)** and this is good. Regardless, data that looks like this:

```json
{"0": 0, "1": 4, ... , "255": 0}
```

scares me greatly. So let it be.

Rest of the code we just write according to language specification. You can just look it up on github.

## Main cycle

Very easy:

* Read file with source code;
* Create new virtual machine with empty memory and filled program array;
* Execute commands one by one until commands in program array end.

Since we use _functional_ way - everything has to be done recursively. So let's do it.

Let's define main recursive function - **execute**:

```rust
fn execute(state: CowVM) -> CowVM {
    new_state = match state.program[state.program_position] {
        0 => commands::do_moo(state),
        1 => commands::do_mOo(state),
        2 => commands::do_moO(state),
        3 => commands::do_mOO(state),
        4 => commands::do_Moo(state),
        5 => commands::do_MOo(state),
        6 => commands::do_MoO(state),
        7 => commands::do_MOO(state),
        8 => commands::do_OOO(state),
        9 => commands::do_MMM(state),
        10 => commands::do_OOM(state),
        11 => commands::do_oom(state),
        _ => state,
    }
  execute(new_state);
}
```

Simple logic - look up the new command, execute it and begin from the start. Keep doing it until nothing left to execute.

That's it. Interpreter for language **COW** is done!

## **Real** main cycle

Now you asking me - **"What was that? Some kind of joke?"**
The same question I asked myself when I've realized,
that "multi-paradigm" language **Rust** is lacking  **Tail Call Optimization**.
(What is this, read [here](http://stackoverflow.com/questions/310974/what-is-tail-call-optimization).)

Without this feature you will quickly find out why site **stackoverflow** is [called like that](https://is.gd/Tg5bid).

Well, guess we will have to do cycle instead.

Let's remove recursion from our **execute** function:

```rust
fn execute(state: CowVM) -> CowVM {
    match state.program[state.program_position] {
        0 => commands::do_moo(state),
        1 => commands::do_mOo(state),
        2 => commands::do_moO(state),
        3 => commands::do_mOO(state),
        4 => commands::do_Moo(state),
        5 => commands::do_MOo(state),
        6 => commands::do_MoO(state),
        7 => commands::do_MOO(state),
        8 => commands::do_OOO(state),
        9 => commands::do_MMM(state),
        10 => commands::do_OOM(state),
        11 => commands::do_oom(state),
        _ => state,
    }
}
```

And start cycle in main function itself:

```rust
fn main() {
    let mut state = init_vm();
    loop {
        if state.program_position == state.program.len() {
            break;
        }
        state = execute(state);
    }
}
```

Do you feel all the pain of the world functional programming?
It's not enough that this language made us forget our lovely recursions, it also made us use mutable variable!!!

We can't do this:

```rust
fn main() {
    let state = init_vm();
    loop {
        if state.program_position == state.program.len() {
            break;
        }
        let state = execute(state);
    }
}
```

because of the reasons unknown? Actually because variable we've introduced in loop goes out of the scope and can't be used afterwards.

## Reading MooMOOmOO source code

There is nothing functional about working with **IO** in **Rust**. At all.
That's why I'm not going to talk about it.
You can look at the source code yourself if you find it interesting.

## Conclusion

In my personal opinion, **Rust** language is already becoming _rusty_ without even being old.
OOP doesn't feel like OOP, FP doesn't feel like FP.
What we left with is "multi-paradigm".
Perhaps, exactly the mix of those paradigms will end up being cool!
Let's hope. And write in **Rust**.

However, there are still reasons in following functional way. In whole program we managed to:

* Never go OOP and never create single class.
* Never have problems with borrow checking. Never use references, muttable variables (almost),
never hearing word "ownership" from compiler. Let me tell you,
writing code knowing that it will always compile feels very good.
* Never touch lifetimes parameters - here, the actual dark side of **Rust**.
Let me tell you - I'm afraid of all there `(x: &'a mut i32)` and I'm happy I didn't have to do this.
* Never make single trait. Not really great achievement, but apparently you don't need traits in FP all that much.
* Make all these functions "pure" and easy to test (maybe I will write about this,
but differences in testing OOP and FP are well known and easilly googlable).

## Afterword

Thank you all for finishing this. I welcome all of you in comment section, there we can discuss different paradigms in **Rust**. Any suggestions are also welcome.

[Link to the source code](https://github.com/Virviil/cow.rs)

## Acknowledgments

Thanks to [@minizinger](https://github.com/minizinger) for developing especially hard for me (being non-functional) parts of the code, inspiration and general help.
