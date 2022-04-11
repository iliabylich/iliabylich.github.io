---
layout: post
title:  "lib-ruby-parser"
date:   2020-11-23 00:00:00 +0300
categories: ruby parser rust
toc: true
comments: true
---
## Intro

So, I'm ready to announce that I have finished working on a new Ruby parser. It's called `lib-ruby-parser`.

Key features:

1. It's fast. It's written in Rust and it's slightly faster than Ripper. The difference is about 1-2% on my machine.
2. It has a beautiful interface. Every single node has its own type that is documented. For example, take a look at [`CSend` node](https://docs.rs/lib-ruby-parser/0.7.0/lib_ruby_parser/nodes/struct.CSend.html) that represents "conditional send" like `foo&.bar`. [Here's a list of all defined nodes](https://docs.rs/lib-ruby-parser/0.7.0/lib_ruby_parser/nodes/index.html). Both Ripper and `RubyVM::AST` have no documentation of their AST format. `whitequark/parser` [has a great documentation](https://github.com/whitequark/parser/blob/master/doc/AST_FORMAT.md), but its AST is not "static".
3. What's "static AST"? By saying that I mean that if documentation says that "N is not-nullable" then it's true no matter what. `whitequark/parser` does a great job, but the nature of dynamic language does not allow it to provide such guarantees. I'll show a few examples later.
4. It's precise. Unlike `whitequark/parser`, its lexer (or tokenizer if it sounds better for you) is based on MRI's `parse.y`. What does it mean? It means that I was not able to find any difference in tokenizing on 3 million lines of code that I have got by pulling sources of top 300 gems (by total downloads). I'll mention how I track it soon.
5. It does not depend on Ruby. In fact, it has absolutely no "required" dependencies (only a few optional ones). So, it's possible to write bindings for any other language, and I have made them for C/C++/Node.js. Of course, it's possible to have bindings for Ruby (because there are bindings for C and it's easy to reuse them)

## Implementation

Current performance (in release mode, with `jemalloc`) is ~200000 LOC/s. I think it can even be used for syntax highlighting (and in the browser, too, haha).


I don't want to dig too far, but some notes could be interesting.

Rust is a general purpose language that is based on LLVM (<3) and can be compiled directly into machine code. It does not need any VMs and it can be compiled to a ton of targets (or platforms). The code does not use pointers, and there are no `unsafe` calls that you might hear about.

Rust does support ADT (algebraic data type) and it has generics, so you can build data structures like

```rust
enum Tree<T> {
    Empty,
    Some(T, Box<Tree<T>>, Box<Tree<T>>),
}
```

Beautiful, right?

I designed `Node` struct in a very similar way:

```rust
enum Node {
    Variant1(Variant1),
    Variant2(Variant2),
    Variant3(Variant3),
    // ...
}

struct Variant1 {
    // fields
}

struct Variant2 {
    // fields
}

// ...
```

If you are familiar with C/C++ it might look similar to tagged union, and was I know that's exactly how they are implemented. Close equivalent in C:

```c
struct Node
{
  enum {
    VARIANT1,
    VARIANT2,
  } variant_no;

  union {
    struct {
      // variant1 data ...
    } variant1_data;

    struct {
      // variant2 data ...
    } variant2_data;
  } variant_value;
};
```

As I said, my lexer is based on MRI's `parse.y`. It's just a set of procedures that I turned into a few structs and interfaces.

But as some of you know, MRI's parser is compiled using `bison` (or `yacc` if you care about licenses). Bison is an LALR(1) parser generator, short summary:

1. you write a `.y` file using bison's DSL, with some interpolations in your programming language
2. this file defines rules on how you want your language constructions to be reduced (or combined)
2. then you convert it to `.{ext}` file where `ext` is what your language uses. done

Unfortunately, bison supports only C/C++/Java/D.

First I looked at what's available in the world of Rust. The most popular LALR parser generator is called [`lalrpop`](https://github.com/lalrpop/lalrpop) and I was very about it at the very beginning, I think has a very, very beautiful API:

```
pub Term: i32 = {
    <n:Num> => n,
    "(" <t:Term> ")" => t,
};

Num: i32 = <s:r"[0-9]+"> => i32::from_str(s).unwrap();
```

Plus, it's written in Rust, so to compile a grammar that is based on it you don't anything except Rust (that you need anyway to compile something written in Rust, right?).

Unfortunately, I have got a few reasons to abandon this idea:

1. No mid-rules (that are used A LOT in MRI) like

    `foo: bar { /* do something */ } baz { /* reduce */ }`

    I guess it's possible to emulate them by introducing rules like

    ```
    bar_with_mid_rule: bar { /* do something */ }
    foo: bar_with_mid_rule baz { /* reduce */ }
    ```
    but then I have no idea how such grammar can be maintained.

2. Compilation speed. I have got ~20% of Ruby grammar backported and noticed a huge performance degradation.
3. No stack introspection (and IIRC no "debug" mode at all). By saying "stack" here I mean parser stack, that's a feature of LR parsers. You can check how it looks like by running `ruby -ye '42'` locally
4. The grammar written with `lalrpop` has a different format comparing to bison, and so maintaining it (like backporting new changed from MRI) seems to be a nightmare.

But then I realized that Bison has a feature called "custom skeleton". It's like a custom Bison template that you can use to convert `.y` to your format, and it "takes" all the data (like token/transition tables) as an argument when called.

So I wrote my own skeleton for Rust and wrapped it into a [Rust library](https://github.com/iliabylich/rust-bison-skeleton). It uses m4 format that is a macro language. Here's [the main file](https://github.com/iliabylich/rust-bison-skeleton/blob/master/rust-bison-skeleton/bison/lalr1-rust.m4) and [an example](https://github.com/iliabylich/rust-bison-skeleton/blob/master/tests/src/calc.y) of how it can be used.

And then it took me about a week to backport the entire parser. The stack of it is a wrapper around `Vec<Value>` where `Value` is an enum type:

```rust
enum Value {
    Stolen,
    Uninitialized,
    None,
    Token(Token),
    TokenList(TokenList),
    Node(Node),
    NodeList(NodeList),
    Bool(bool),
    MaybeStrTerm(MaybeStrTerm),
    Num(Num),

    /* For custom superclass rule */
    Superclass(Superclass),

    // ... variants for other custom rules, there are ~25 of them
}
```

Initially result of each "reduce" action (that is stored in `yyval` variable) is set to `Value::Uninitialized`, reading `$<Node>5` in your action is compiled into something like

```rust
match yystack.steal_value_at(5) {
  Value::Node(node) => node,
  other => panic!("not a node: {:?}", other)
}
```

Doing `$$ = ...` is compiled into `yyval = ...`.

Why does reading "steal" the value from the stack? Because you can do `$$ = $<Node>1` and push an element of the vector to the same vector. At the same time you can do something like `$$ = combine($<Node>1, $<Node>1)` where you want both arguments to be mutable. You can't do it in Rust.

This is why when you read any stack value you actually steal it by replacing what's in the stack with `Value::Stolen`:

```rust
impl Stack {
    // ...

    fn steal_value_at(&mut self, i: usize) -> Value {
        let len = self.stack.len();
        std::mem::replace(&mut self.stack[len - 1 - i], Value::Stolen)
    }
}
```

`Value::Stolen` is just a placeholder value that indicates (when you see it) that your code previously has accessed the same stack entry. It's necessary to have it (or in general some kind of a default value that is set by `std::mem::take/replace`) to "fix" ownership model.

So then it was done and I started profiling it. At the very beginning it was incredibly slow, but I knew it, I had way too many `.clone()` calls in my code (in Rust that's a deep clone that is quite expensive in some cases). I added `jemalloc` and started profiling (`pprof-rs` <3), I removed most clones and got ~160-170 thousand LOC/s.

Great, but I wanted more. ~20% of time in my benchmark was spent on this `std::mem::replace` call that swaps non-overlapping bytes. Initially I thought that I can't improve it (that's the fastest way to take the value AND to put a placeholder instead of it). At some point when I was writing C++ bindings I noticed that `sizeof(Node)` is 120 bytes (`Node` here is a C++ `Node` class) and it literally opened my eyes.

I'll write it in C, take a look at this structure:

```c
struct Node
{
    enum { /* variants */ } variant;
    union {
        struct Args args;
        struct Def def;
        struct IfTernary if_ternary,
        // .. all other variant values
    } variant_value;
};

struct IfTernary
{
    Node *cond;
    Node *if_true;
    Node *if_false;
    // .. a few more Range field, snip
};
```

(Let's pretend that it can be compiled without forward declarations). This C `Node` is very, very similar to its Rust analogue. What's the size of this `Node`?

The size of `struct` is a sum of all of its nodes (let's simplify it and forget about memory alignment), the size of `union` is a maximum values its variant sizes.

Some specific node structures have multiple fields inside that are pointers (8 bytes on x86-64), and so the size of the generic `Node` is huge too.

Let's "swap" pointers and unions:

```c
struct Node
{
    enum { /* variants */ } variant;
    union {
        struct Args *args;
        struct Def *def;
        struct IfTernary (if_ternary,
        // .. all other variant values
    } variant_value;
};

struct IfTernary
{
    Node cond;
    Node if_true;
    Node if_false;
    // .. a few more Range field, snip
};
```

See? Now the size of the `union` is always `sizeof(void*)` and so `Node` is much smaller.

Why does it matter? Because `Vec<T>` in Rust is a wrapper for `*T` array. It's a contiguous and "flat" region of memory where all elements are "inlined":

```c
|item1|item2|item3|item4|...
```

and every item takes `sizeof(T)` memory. This is why doing `std::mem::replace(t1, t2)` can swap at most `sizeof(T)` bytes and this is why I want `T` to be as small as possible.

After turning my Rust model into

```rust
enum Node {
    Args(Box<Args>),
    IfTernary(Box<IfTernary>),
    /// ...
}

struct IfTernary {
    cond: Node,
    if_true: Node,
    if_false: Node,
    // ...
}

// other variants
```

I have got the same performance as Ripper.

## Future improvements

I keep thinking about turning `lib-ruby-parser` into zero copy parser and I'm believe it's very possible.

Currently all "values" that copy from source code (like numeric/string literals, token values) are copied into tokens and AST nodes:

```
"42"

tokens:
    Token { name: "tINTEGER", value: "42", loc: 0...2 }

nodes:
    Int { value: "42", expression_l: 0..2 }
```

`lib-ruby-parser` constructs these `"42"` strings twice by copying a part of the input. Of course, for this particular case it's possible to store only ranges like `start...end`, but there are exceptions where values of tokens and AST nodes are not equal to input sub-strings (like escape sequences, `"\n"` or `"\u1234"`).

Even this way it's possible to introduce the following type system:

```rust
enum Value<'a> {
    SubstringOfInput(&'a [u8]),
    ManuallyConstructed(Vec<u8>)
}
```

The first variant is a reference, the latter is owned. Total value (of both token and AST node) could be just a vector of such enums, and if you parse a string `"foo\n"` you'll get

```
tokens:
    Token {
        name: "tSTRING_CONTENT",
        value: vec![
            Value::SubstringOfInput(&input[1..3]),
            Value::ManuallyConstructed(vec![b'\n'])
        ],
        loc: 1..5
    }
```

However, then `input` must live as long as tokens and AST, and it sounds a bit problematic.

One option that I see is adding `Rc<Input>` to such values and store a range in `SubstringOfInput` enum variant. That's basically a `shared_ptr` from C++ world that wraps a raw pointer (like `T*`) + (pointer to) a number of existing "clones" of this pointer. Every time you copy it the shared number is incremented, destructor decreases it and once it's zero it also deletes `T`. It's quite cheap in terms of performance (something like `*n += 1` in constructor, `*n -= 1; drop(ptr) if n == 0;` in destructor)

## C bindings

GitHub repository - [https://github.com/lib-ruby-parser/c-bindings](https://github.com/lib-ruby-parser/c-bindings)

It took me a while to fix all segfaults, memory leaks and invalid memory access patterns (I'm not a C developer). `Valgrind` and `asan` are incredibly useful tools, I can't even imagine how much time it would take to write bindings without these guys.

The API is very similar, there's an additional layer between C and Rust that converts output of `lib-ruby-parser` into C structures.

It uses a combination of `enum` and `union` to represent a `Node`.

## C++ bindings

[https://github.com/lib-ruby-parser/cpp-bindings](https://github.com/lib-ruby-parser/cpp-bindings)

I personally like C++ much more than C. Smart pointers, containers, generics, but still worse for me than Rust. Upcoming standards are going to introduce even more features like modules and concepts, but, no.

The same story again, `valgrind`/`asan`, an extra layer that converts `Rust` objects to `C++` classes.

Also, my `valgrind` on Mac could not detect calling `free` on `C++` object (that's invalid, should be `delete` from C++), and so I had to setup a docker container locally to find and fix it.

It uses modern `std::variant<Nodes...>` to represent a `Node`.

## Node bindings

As a proof of concept I also created bindings for Node.js - [https://github.com/lib-ruby-parser/node-bindings](https://github.com/lib-ruby-parser/node-bindings).

I was actually impressed (in a good way) how elegant is the API of `node-addon-api`. I worked with V8 C++ API about a 10 months ago in Electron and back in the day it was quite verbose and painful. Also, I remember an interesting feature of `TryCatch` class:

```cpp
void DoSomething() {
    TryCatch trycatch;
    // do something that potentially does `throw err`
    if (trycatch.HasCaught()) {
        // process trycatch.Exception() value
    }
}
```

I personally think that this class abuses constructors/destructors in C++:

+ constructor of `TryCatch` registers `this` in some global context  (like `GetCurrentHandleScope().SetThrowHandler(this)`
+ any code that throws a JavaScript error sets it on this `TryCatch` handler
+ it's possible to access an exception object from any place
+ destructor of `TryCatch` drops it

I admit that it's smart, but it sounds way too implicit to me. I'd like this interface to perform a register in a more explicit way (by calling `register/unregsiter` for example).

And I like that `node-addon-api` handles it [even better](https://github.com/nodejs/node-addon-api/blob/master/doc/error_handling.md#handling-a-n-api-c-exception) by sticking to C++ exceptions.

Node.js bindings are based on C++ bindings from the previous section and they use a custom JavaScript class for each node type. Yes, there's also an extra layer that converts C++ objects to JavaScript.

## WASM

Rust can be compiled to WebAssembly, here's a demo - [https://lib-ruby-parser.github.io/wasm-bindings](https://lib-ruby-parser.github.io/wasm-bindings).

It worked out of the box with one minor change. I had to mark `onigurama` dependency as optional and disable it for WASM build. Oh, I love how Rust can turn dependency into a feature that is optional and enabled/disabled by default as you configure it.

## Final thoughts

This is one of the biggest open-source projects that I have ever made. It can be used from Rust/C/C++/Node.js/browser. It's fast (but remember, it can get even faster), it's precise and it's very strongly typed.

Also, if you need a custom setup (like custom C++ classes or maybe a completely different API) there's a [meta-repository](https://github.com/lib-ruby-parser/nodes) with all nodes information. You can use it to build your own bindings (or maybe new bindings for other languages), it has a `nodes` function that returns a `Vec<Node>` (`Node` here is just a "configuration" of a single `Node` variant from Rust - [source](https://github.com/lib-ruby-parser/nodes/blob/master/src/node.rs#L6))

Bindings still have a room for improvements (for example, there's no way to pass custom `ParserOptions` to the parser), and current version of `lib-ruby-parser` is `0.7.0`.

However, I think it's ready. It's stable and I'm not going to introduce any major changes anymore. I'm going to cut `3.0.0` once Ruby 3 is released. Don't hesitate to drop me a message if it works well for you.
