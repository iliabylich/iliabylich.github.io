---
title: "Writing bindings upside down"
date: "2021-12-28"
cover: ""
tags: ["Rust", "C", "C++", "Ruby", "JavaScript", "Node", "bindings", "LLVM", "LTO"]
---

## Bindings

Quite a long time ago I started writing C/C++/Ruby/Node.js/WASM bindings so I could call [my Rust project](https://github.com/lib-ruby-parser/lib-ruby-parser) from those languages. It is a Ruby language parser.

I tried multiple ways and found one that is very (VERY) controversial, but I think it deserves it's own article.

## Traditional way

Let's say you have a library in C. Just for simplicity, Rust is not special here.

![library-puzzle-piece.png](/writing-bindings-upside-down-1.png)

How can you use it in C++? A very simple solution is to wrap your header file with

```cpp
#ifdef __cplusplus
extern "C"
{
#endif

// C bindings

#ifdef __cplusplus
}
#endif
```

then change your includes to be slightly more compatible with C++ (`stdio.h` -> `cstdio` etc) and call it a day.

+ Pros: very simple.
+ Cons: your API has raw pointers, raw array/string structures that expose everything and are error-prone when used.

How can you use it in Ruby/Node.js? Again, let's start with "traditional" solution. For Ruby take your C library, for Node.js take C++ library and "wrap" it using a "native" extension.

It is possible to "attach" any arbitrary data to Ruby/Node.js objects
+ Ruby has [`TypedData_Wrap_Struct` function](https://docs.ruby-lang.org/en/2.4.0/extension_rdoc.html#label-C+struct+to+Ruby+object)
+ Node.js has [`Napi::External`](https://github.com/nodejs/node-addon-api/blob/main/doc/external.md)

In both cases it's possible to get pointer to attached data at any moment, both require you to specify a function that GC calls to free attached data.

Pros: still quite simple.
Cons: really error-prone, libraries designed this way quite frequently have memory leaks and segfaults.

![traditional-bindings.png](/writing-bindings-upside-down-2.png)

## Slow but more reliable solution

Of course it'd be unfair to not mention that it's always possible to type-cast structures from low-level language to high-level:

1. C -> C++ - by converting C structs to pretty C++ classes with smart pointers and collections
2. C -> Ruby - by creating Ruby objects from C structs
3. C++ -> Node.js - by creating JavaScript objects from C++ classes

+ Pros: this way it's actually less dangerous. You can cover it with unit-tests and check with ASAN/Valgrind/LSAN that you have no segfaults, memory leaks or incorrect memory access.
+ Cons: you have to copy a bunch of data and it will hurt performance. Many of your methods have to be rewritten to target language OR you have convert C++/Ruby/JS objects back to C to call original C functions.

A small note on copying: it is possible to "move" some data in certain cases and languages:

+ Ruby has a family of `rb_str_new_*` functions, but literally all of them take `const char *ptr, size_t len`, however, it's possible to create a "dummy" heap-allocated Ruby `String` and set `ptr` and `len` on it. It is possible only (and only) because internals of all Ruby objects (including `String`) are fully exposed on C layer.
+ Node.js has a [`Napi::String`](https://github.com/nodejs/node-addon-api/blob/main/doc/string.md) class for that but it takes either a C++ `const std::string&` or `const char *`. From what I know internals are not available, so copying is necessary.
+ C++ has a `std::string` that does not have a "take-and-store-what's-given" constructor that takes a `char *` (I would call it a "move" constructor, but this name has a different meaning in C++ :D). There's a `const char *` constructor that performs copying. Of course it's possible to create a compiler-dependent type-casting to a class with the same set of fields, move pointer + `len` + capacity there and convert it back to `std::string`. It's ugly, but definitely doable.

## A demo, small but representative

Let's start. I'd like to demonstrate a different solution on a tiny example. Let's write a micro-library that has several Rust structs, something like a function that, let's say, takes a `String` and returns a `Vec` of all non-ASCII chars. It's called `foo`.

```rust
pub fn foo(s: &str) -> Vec<char> {
    s.chars().filter(|c| !c.is_ascii()).collect()
}

#[test]
fn test_foo() {
    let chars = foo("abc😋中国def");
    assert_eq!(chars, vec!['😋', '中', '国']);
}
```

There are two structs that belong to Rust world exclusively: `Vec` and `char`.

## "Foreign" implementation

Can we expose these two types in C? `Vec` is defined in Rust standard library and it has `#[repr(Rust)]`, `char` is a 4-byte structure with no public layout.

We could define our own `repr(C)` structs together with `impl From<RustType> for CType` and call `.into()`, but that's not really the goal here.

Let's think for a moment about C++, Ruby and Node:

+ In C++ I'd really like to use `std::vector` for `Vec` and `std::string` for `char`.
+ For both Ruby and Node.js I want `Array` for `Vec` and `String` for `char`.

What if our library could depend on some contract that requires bindings to provide primitives? The contract will be the same for all bindings, but implementation will be different.

Rust does not know what is C++ `std::vector` or Ruby `String`, but we know it, our bindings know it and by providing a set of foreign utility functions (implemented on the bindings side) we could work with it just like with native `std::Vec<T>`.

![library-with-external-primitives.png](/writing-bindings-upside-down-3.png)

Here "Primitives" will be:

+ Rust structures for Rust version of the library
+ C structs for C version
+ C++ classes for C++ version
+ Ruby classes for Ruby version

"Functions to work with primitives" will be a set of `extern "C"` functions that take and return these "foreign" objects. Rust can call them without any knowledge of these objects.

By swapping implementations at link time we can get the same algorithm that works with a different set of structures from different languages. "Link-time polymorphism" seems to be a good name for this concept.

## Example

I think it makes sense to start with C, this is what I would like to get eventually:

```c
#ifndef STRUCTS_H
#define STRUCTS_H

#include <stddef.h>
#include <stdbool.h>

typedef struct Char
{
    char bytes[4];
} Char;

typedef struct CharList
{
    Char *ptr;
    size_t len;
} CharList;

#endif // STRUCTS_H
```

Now the question is: how can we return this `CharList` from our Rust code? We could use `bindgen` and something-something. No, no and no.

Instead, let's make Rust think that `CharList` on its side is some struct of some (AOT-known) size without any meaningful fields. To do that we need to dump sizes of our structs and make them available in C:

```c
#include <stdio.h>
#include "structs.h"

int main()
{
    printf("CHAR_SIZE=%lu\n", sizeof(Char));
    printf("CHAR_LIST_SIZE=%lu\n", sizeof(CharList));

    return 0;
}
```

We compile it, we run it, and we save its output to a text file called `sizes`, here's what I have locally

```c
CHAR_SIZE=4
CHAR_LIST_SIZE=16
```

Now our Rust library has to be changed to work in 2 different modes:

+ `native` - when Rust structs are used as fields.
+ `external` - when only size of structs is known, but fields and their positions are not.

We a new feature to our `Cargo.toml`:

```toml
[features]
default = []

# enables "external" mode, when structs have only size but no fields
external = []
```

and we create a build script:

```rust
#[cfg(feature = "external")]
fn main() {
    // read path of the `sizes` file from the ENV var
    let sizes_filepath = env!("SIZES_FILEPATH");
    // read `sizes` file
    let sizes = std::fs::read_to_string(sizes_filepath)
                  .expect("SIZES_FILEPATH has to point to a file");

    // parse it line by line and re-write to Rust
    let sizes_rs = sizes
        .lines()
        .map(|line| {
            let parts = line.split("=").collect::<Vec<_>>();
            let name = parts[0];
            let value = parts[1];
            format!("pub(crate) const {}: usize = {};", name, value)
        })
        .collect::<Vec<_>>()
        .join("\n");

    // write it back to sizes.rs
    std::fs::write("src/sizes.rs", sizes_rs).unwrap();
}

#[cfg(not(feature = "external"))]
fn main() {
    // dummy main for "native" mode when no work is needed
}
```


Note: also there should be a `rerun-if-changed` line, but for the sake of implicitly I ignore dependencies here.

After running `SIZES_FILEPATH=path/to/sizes cargo build --features=external` we get `src/sizes.rs`.

```rust
pub(crate) const CHAR_SIZE: usize = 4;
pub(crate) const CHAR_LIST_SIZE: usize = 16;
```

Time to use it! First we wrap existing definition of `Char` and `CharList` with

```rust
#[cfg(not(feature = "external"))]
mod native {
  pub type Char = char;
  pub type CharList = Vec<char>;
}
#[cfg(not(feature = "external"))]
use native::{Char, CharList};
```

(note that we use `not(feature = "external")` above, which means "native" mode, they are opposite, and so one feature is enough) and then we add conditionally included `external` module with **all** structs that we use:

```rust
#[cfg(feature = "external")]
mod sizes;
#[cfg(feature = "external")]
mod external {
    use crate::sizes::*;

    #[repr(C)]
    pub struct Char {
        bytes: [u8; CHAR_SIZE],
    }

    #[repr(C)]
    pub struct CharList {
        bytes: [u8; CHAR_LIST_SIZE],
    }
}
#[cfg(feature = "external")]
use external::{Char, CharList};
```

Now if we try to return `CharList` from our `foo` function we get compilation errors:

+ there's no way to construct a `Char` from `char`
+ we can't `collect()` `CharList` from `Iterator<char>`

How can we implement it? Let's take a look again at used APIs.

1. we iterate over given string (it'll be a `const *u8` in C API) - we have it
2. we check `c.is_ascii()` - that's also OK
3. we need to map `char` to foreign `Char` - **this requires a constructor**
4. we need a way to construct `CharList` from chars - **this requires methods like `CharList::new()` and `CharList::push()`**
5. we have a unit-test that needs `CharList::len()`, `CharList::at()`, `impl PartialEq for Char` (to compare individual chars) and `impl std::fmt::Debug for Char` (to print left/right-hand side if assertions fails)

So, here's a list of "external" (i.e. in the `mod external {}` block) methods we want to have:

```rust
impl From<char> for Char {
    fn from(_: char) -> Self { todo!() }
}
impl PartialEq for Char {
    fn eq(&self, other: &Self) -> bool { todo!() }
}
impl std::fmt::Debug for Char {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result { todo!() }
}

impl CharList {
    pub fn new() -> Self { todo!() }
    pub fn push(&mut self, item: Char) { todo!() }
    pub fn len(&self) -> usize { todo!() }
    pub fn at(&self, idx: usize) -> &Char { todo!() }
}
```

For now all of them can return `todo!()`, but we'll implement them later. Time to rewrite our Rust implementation to use these new language-independent functionality:

```rust
pub fn foo(s: &str) -> CharList {
    let mut char_list = CharList::new();
    for char in s.chars() {
        if !char.is_ascii() {
            char_list.push(Char::from(char));
        }
    }
    char_list
}

#[test]
fn test_foo() {
    let s = "abc😋中国";
    let chars = foo(s);
    assert_eq!(chars.len(), 3);
    assert_eq!(chars.at(0), &Char::from('😋'));
    assert_eq!(chars.at(1), &Char::from('中'));
    assert_eq!(chars.at(2), &Char::from('国'));
}
```

That was easy, right? Way less elegant but still easy.

## Implementing bindings

At this point you might have a guess on what's going to happen next. We are going to define a bunch of external C-ABI functions and blindly call them in all `todo!()` places. Later these functions will be implemented by the bindings library on C side:


```rust
extern "C" {
    fn char__new(c1: u8, c2: u8, c3: u8, c4: u8) -> Char;
    fn char__at(this: *const Char, idx: u8) -> u8;
}
impl Char {
    fn byte_at(&self, idx: u8) -> u8 {
        debug_assert!(idx <= 3);
        unsafe { char__at(self, idx) }
    }
}
impl From<char> for Char {
    fn from(c: char) -> Self {
        let mut buf = [0; 4];
        c.encode_utf8(&mut buf);
        unsafe { char__new(buf[0], buf[1], buf[2], buf[3]) }
    }
}
impl From<&Char> for char {
    fn from(c: &Char) -> Self {
        let c1 = c.byte_at(0);
        let c2 = c.byte_at(1);
        let c3 = c.byte_at(2);
        let c4 = c.byte_at(3);
        let bytes = [c1, c2, c3, c4];
        let mut zero_idx = 4;
        for idx in 0..4 {
            if bytes[idx] == 0 {
                zero_idx = idx;
                break;
            }
        }
        let bytes = &bytes[0..zero_idx];
        let s = std::str::from_utf8(bytes).unwrap();
        let chars = s.chars().collect::<Vec<_>>();
        debug_assert!(chars.len() == 1);
        chars[0]
    }
}
impl PartialEq for Char {
    fn eq(&self, other: &Self) -> bool {
        (0..4).all(|idx| self.byte_at(idx) == other.byte_at(idx))
    }
}
impl std::fmt::Debug for Char {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        write!(f, "{}", char::from(self))
    }
}

extern "C" {
    fn char_list__new() -> CharList;
    fn char_list__push(this: *mut CharList, item: Char);
    fn char_list__len(this: *const CharList) -> usize;
    fn char_list__at(this: *const CharList, idx: usize) -> *const Char;
}
impl CharList {
    pub fn new() -> Self {
        unsafe { char_list__new() }
    }
    pub fn push(&mut self, item: Char) {
        unsafe { char_list__push(self, item) }
    }
    pub fn len(&self) -> usize {
        unsafe { char_list__len(self) }
    }
    pub fn at(&self, idx: usize) -> &Char {
        unsafe { char_list__at(self, idx).as_ref().unwrap() }
    }
}
```

OK, now we have a contract. Rust relies on it, but C so far does not provide anything. If we try to run tests with `--features=external` we get a bunch of linkage errors, which is 100% expected. Time to implement in on C side.

This is a "shared" version that we'll use for C++/Ruby/Node.js bindings

```c
#ifndef BINDINGS_H
#define BINDINGS_H

#include DEFINITIONS_FILE

#ifdef __cplusplus
extern "C"
{
#endif

    Char_BLOB char__new(uint8_t c1, uint8_t c2, uint8_t c3, uint8_t c4);
    const uint8_t *char__bytes(const Char_BLOB *self);

    CharList_BLOB char_list__new();
    void char_list__push(CharList_BLOB *self, Char_BLOB item);
    size_t char_list__len(const CharList_BLOB *self);
    Char_BLOB char_list__at(const CharList_BLOB *self, size_t idx);

#ifdef __cplusplus
}
#endif

#endif // BINDINGS_H
```

It includes `DEFINITIONS_FILE` because we want it to be generic (and we'll pass it as a dynamic define via `-D` flag). Also you might notice that methods take/return `<type>_BLOB` type, that's because we want to pass C-compatible types. C types are C-compatible, and so we make another file `bindings-support.h` with

```c
#ifndef BINDINGS_SUPPORT_H
#define BINDINGS_SUPPORT_H

#include "structs.h"

typedef Char Char_BLOB;
typedef CharList CharList_BLOB;

#endif // BINDINGS_SUPPORT_H
```

... to create aliases. For C++ we'll have to convert our classes to something C understands (I'll cover it later).

Implementation time!

```c
Char_BLOB char__new(uint8_t c1, uint8_t c2, uint8_t c3, uint8_t c4)
{
    return (Char_BLOB){.bytes = {c1, c2, c3, c4}};
}
uint8_t char__at(const Char_BLOB *self, uint8_t idx)
{
    return self->bytes[idx];
}

CharList_BLOB char_list__new()
{
    return (CharList_BLOB){.ptr = NULL, .len = 0};
}
void char_list__push(CharList_BLOB *self, Char_BLOB item)
{
    Char *prev = self->ptr;
    self->ptr = malloc(sizeof(Char) * (self->len + 1));
    if (self->len > 0)
    {
        memcpy(self->ptr, prev, self->len * sizeof(Char));
        free(prev);
    }
    self->ptr[self->len] = item;
    self->len = self->len + 1;
}
size_t char_list__len(const CharList_BLOB *self)
{
    return self->len;
}
Char_BLOB char_list__at(const CharList_BLOB *self, size_t idx)
{
    return self->ptr[idx];
}
```

We compile it

```bash
$ clang c-bindings/bindings.c -g -c -o c-bindings/all.o
$ ar rc c-bindings/libbindings.a c-bindings/all.o
```

And change our `build.rs` script to link with it (the purpose of these environment variables is to make external primitives implementation "pluggable"):

```rust
let external_lib_path = env!("EXTERNAL_LIB_PATH");
println!("cargo:rustc-link-search={}", external_lib_path);

let external_lib_name = env!("EXTERNAL_LIB_NAME");
println!("cargo:rustc-link-lib=static={}", external_lib_name);
```

And now we can run tests:

```bash
$ cd rust-lib
$ EXTERNAL_LIB_PATH="../c-bindings" \
    EXTERNAL_LIB_NAME="bindings" \
    SIZES_FILEPATH="../c-bindings/sizes" \
    cargo test --features=external
```

and we get 1 passing test!

## C++ bindings, quickly

I want this:

```cpp
#ifndef STRUCTS_HPP
#define STRUCTS_HPP

#include <string>
#include <vector>

class Char
{
public:
    char bytes[4];
    Char() : Char(0, 0, 0, 0) {}
    explicit Char(char c1, char c2, char c3, char c4) : bytes{c1, c2, c3, c4} {}
    size_t size() const
    {
        size_t size = 2;
        if (bytes[2])
            size++;
        if (bytes[3])
            size++;
        return size;
    }
    std::string as_string() const
    {
        std::string s;
        s.reserve(4);
        for (size_t i = 0; i < size(); i++)
        {
            s.push_back(bytes[i]);
        }
        return s;
    }
};

using CharList = std::vector<Char>;

#endif // STRUCTS_HPP
```

C functions are the same, but these classes are incompatible with C FFI, so we need to define our blob structs:

```cpp
#ifndef BINDINGS_SUPPORT_HPP
#define BINDINGS_SUPPORT_HPP

#include "structs.hpp"

#define DECLARE_BLOB(T)               \
    extern "C"                        \
    {                                 \
        struct T##_BLOB               \
        {                             \
            uint8_t bytes[sizeof(T)]; \
        };                            \
    }                                 \
    union T##_UNION                   \
    {                                 \
        T value;                      \
        T##_BLOB blob;                \
                                      \
        ~T##_UNION() {}               \
        T##_UNION()                   \
        {                             \
            new (&value) T();         \
        }                             \
    };                                \
    T##_BLOB PACK_##T(T value)        \
    {                                 \
        T##_UNION u;                  \
        u.value = std::move(value);   \
        return u.blob;                \
    };                                \
    T UNPACK_##T(T##_BLOB blob)       \
    {                                 \
        T##_UNION u;                  \
        u.blob = blob;                \
        return std::move(u.value);    \
    }

DECLARE_BLOB(Char);
DECLARE_BLOB(CharList);

#endif // BINDINGS_SUPPORT_HPP
```

Here `DECLARE_BLOB` macro for given `Type` defines:

+ `Type_BLOB` struct that is C-compatible
+ `Type_UNION` union that is a union of `Type` and `Type_BLOB`
+ `PACK_Type` function that converts an instance of `Type` to `Type_BLOB` so we can return if **from** `extern "C"` function
+ `UNPACK_Type` function that converts `Type_BLOB` to `Type` so we can "unpack" blob that is passed **to** `extern "C"` function

This conversion with `union` is an equivalent of `std::mem::transmute` from Rust (C++20 has `std::bit_cast` for that, but `union` shows better what happens under the hood). Also we `std::move` the value to and from `union` on conversion, this is important (otherwise, copyable types are copied).

However, it has a requirement for `T` to be both movable and constructible with no arguments. We could do `std::memset(this, 0, sizeof(T))`, but some move constructors/assignment operators swap fields of `this` and given `other`, and so sometimes it's invalid to call a destructor on an object full of zeroes (of course it totally depends on the structure of the object).

With these blobs implementing binding functions is trivial:

```cpp
extern "C"
{
    Char_BLOB char__new(uint8_t c1, uint8_t c2, uint8_t c3, uint8_t c4) noexcept
    {
        return PACK_Char(Char(c1, c2, c3, c4));
    }
    uint8_t char__at(const Char_BLOB *self, uint8_t idx) noexcept
    {
        const Char *s = (const Char *)self;
        if (idx >= 4)
        {
            return 0;
        }
        return s->bytes[idx];
    }

    CharList_BLOB char_list__new() noexcept
    {
        return PACK_CharList(CharList());
    }
    void char_list__push(CharList_BLOB *self, Char_BLOB item) noexcept
    {
        ((CharList *)self)->push_back(UNPACK_Char(item));
    }
    size_t char_list__len(const CharList_BLOB *self) noexcept
    {
        return ((CharList *)self)->size();
    }
    Char_BLOB char_list__at(const CharList_BLOB *self, size_t idx) noexcept
    {
        return PACK_Char(((CharList *)self)->at(idx));
    }
}
```

We compile and build a static library:

```bash
$ clang++ -std=c++17 cpp-bindings/bindings.cpp -g -c -fPIE -o cpp-bindings/all.o
$ ar rc cpp-bindings/libbindings.a cpp-bindings/all.o
```

One extra step that is specific to C++ libraries - we need to link with C++ runtime, so this extra code goes to `Cargo.toml` and `build.rs` respectively:

```toml
[features]
# enables linking with C++ runtime
link-with-cxx-runtime = []
```

```rust
if cfg!(feature = "link-with-cxx-runtime") {
    if cfg!(target_os = "linux") {
        println!("cargo:rustc-link-lib=dylib=stdc++");
    } else {
        println!("cargo:rustc-link-lib=dylib=c++");
    }
}
```

And finally we can run our tests:

```sh
EXTERNAL_LIB_PATH="../cpp-bindings" \
  EXTERNAL_LIB_NAME="bindings" \
  SIZES_FILEPATH="../cpp-bindings/sizes" \
  cargo test --features=external,link-with-cxx-runtime
```

## Ruby bindings, also quickly

We want our `Char` to be a Ruby `String` and `CharList` to be an `Array`, in Ruby C API they are both represented as `VALUE` (that is technically a pointer to a tagged union (unless it's a small number/`true`/`false`/`nil`, then it's basically the value itself)).

```c
#ifndef STRUCTS_H
#define STRUCTS_H

#include <ruby.h>

typedef VALUE Char;
typedef VALUE CharList;

#endif // STRUCTS_H
```

and so sizes are these:

```c
CHAR_SIZE=8
CHAR_LIST_SIZE=8
```

`VALUE` is simply an alias to `unsigned long` on x86_64, so blobs are aliases:

```c
#ifndef BINDINGS_SUPPORT_H
#define BINDINGS_SUPPORT_H

#include "structs.h"

typedef Char Char_BLOB;
typedef CharList CharList_BLOB;

#endif // BINDINGS_SUPPORT_H
```

Bindings implementation (uses Ruby C API):

```c
#include "bindings-support.h"

Char_BLOB char__new(uint8_t c1, uint8_t c2, uint8_t c3, uint8_t c4)
{
    char c[4] = {(char)c1, (char)c2, (char)c3, (char)c4};
    long len = 2;
    if (c3) len++;
    if (c4) len++;
    return rb_utf8_str_new(c, len);
}
uint8_t char__at(const Char_BLOB *self, uint8_t idx)
{
    VALUE this = *self;
    return StringValuePtr(this)[idx];
}

CharList_BLOB char_list__new()
{
    return rb_ary_new();
}
void char_list__push(CharList_BLOB *self, Char_BLOB item)
{
    rb_ary_push(*self, item);
}
size_t char_list__len(const CharList_BLOB *self)
{
    return rb_array_len(*self);
}
Char_BLOB char_list__at(const CharList_BLOB *self, size_t idx)
{
    return rb_ary_entry(*self, idx);
}
```

Looks similar to C++ bindings, right? OK, can we run Rust tests with Ruby primitives? Unfortunately, no. These methods expect Ruby VM to be up and running and embedding Ruby is not something many Ruby developers do.

Instead, we need to re-compile it to a dynamically-loaded library that (once loaded) registers a `foo` method that takes a string, passes its `const char *` pointer to our `foo` function defined in Rust and returns `CharList` back to Ruby space.

```c
#include <ruby.h>
#include "structs.h"

CharList c_foo(const char *s);

VALUE rb_foo(VALUE self, VALUE s)
{
    (void)self;
    Check_Type(s, T_STRING);
    CharList chars = c_foo(StringValueCStr(s));
    return chars;
}

void Init_foo()
{
    rb_define_global_function("foo", rb_foo, 1);
}
```

`c_foo` is defined on Rust side with C linkage:

```rust
#[no_mangle]
pub extern "C" fn c_foo(s: *const i8) -> CharList {
    let s = unsafe { std::ffi::CStr::from_ptr(s).to_str().unwrap() };
    foo(s)
}
```

And now if we compile it to `.bundle` (this is for Mac, on Linux and Windows it's a `.so` extension)

```bash
$ clang -Ipath/to/ruby/includes ruby-bindings/init.c -c -o ruby-bindings/init.o
$ clang -Ipath/to/ruby/includes ruby-bindings/bindings.c -c -o ruby-bindings/bindings.o
$ ar rc ruby-bindings/libbindings.a ruby-bindings/bindings.o

$ cd rust-lib
$ EXTERNAL_LIB_PATH="../ruby-bindings" \
    EXTERNAL_LIB_NAME=bindings \
    SIZES_FILEPATH="../ruby-bindings/sizes" \
    cargo build --features=external
$ cd ..

$ clang \
    -dynamic \
    -bundle \
    -o ruby-bindings/foo.bundle \
    ruby-bindings/init.o rust-foo/librust-foo-rust.a \
    -Wl,-undefined,dynamic_lookup
```

.. we get `ruby-bindings/foo.bundle` that can be imported from Ruby:

```sh
$ ruby -r./ruby-bindings/foo -e 'p foo("abc😋中国")'
["😋", "中", "国"]
```

In fact there are many more options that are passed to `clang` above, check out [the repository](https://github.com/iliabylich/writing-bindings-upside-down) if you want to try it yourself.

## Memory leaks

All versions above have a memory leak that can be easily identified by compiling with

```sh
ASAN_OPTIONS=detect_leaks=1 CXXFLAGS="-fsanitize=address" clang++
```

or (to track it when running Rust tests)

```sh
RUSTFLAGS="-Z sanitizer=address" ... cargo test
```

(On Mac make sure to use `clang` from Homebrew, version that ships with OS does not support ASAN)

Running tests with this options shows that we have a leak somewhere in

```text
Direct leak of 96 byte(s) in 1 object(s) allocated from:
...
    #9 0x105905990 in char_list__push bindings.cpp:34
    #10 0x105904c04 in rust_foo::external::CharList::push::hf5bdccdb764b0c62 lib.rs:91
    #11 0x1059043bb in rust_foo::foo::hf9387b4435c6f8b6 lib.rs:108
    #12 0x10590442d in c_foo lib.rs:117
    #13 0x1059006e8 in cpp_foo(char const*) test.cpp:14
    #14 0x105900a10 in main test.cpp:24
```

And the reason is that our `CharList` is heap-allocated, but it has no destructor on the Rust side. To fix it we need to add 2 more functions to our bindings:

```rust
extern "C" {
    fn char__drop(this: *mut Char);
    fn char_list__drop(this: *mut CharList);
}

impl Drop for Char {
    fn drop(&mut self) {
        unsafe { char__drop(self) }
    }
}

impl Drop for CharList {
    fn drop(&mut self) {
        unsafe { char_list__drop(self) }
    }
}
```


Of course, we need to add it to `bindings.h`:

```c
#ifdef __cplusplus
extern "C"
{
#endif

    void char__drop(Char_BLOB *self);
    void char_list__drop(CharList_BLOB *self);

#ifdef __cplusplus
}
#endif
```

Here's a C implementation (depending on your implementation `char__drop` could to something):

```c
void char__drop(Char_BLOB *self)
{
    // noop, Char has no allocations
}

void char_list__drop(CharList_BLOB *self)
{
    if (self->len > 0)
    {
        free(self->ptr);
    }
}
```

C++ implementation:

```cpp
void char__drop(Char_BLOB *self)
{
    // noop, Char has no allocations
}
void char_list__drop(CharList_BLOB *self)
{
    ((CharList *)self)->~vector();
}
```

Ruby:

```c
void char__drop(Char_BLOB *self)
{
    // noop, Ruby has GC
}
void char_list__drop(CharList_BLOB *self)
{
    // noop, Ruby has GC
}
```

## Performance

How about performance? When we compile with Rust primitives we use LTO and so things from Rust standard library can be optimized together with our code. Luckily, there's a way to do that for external primitives too.

I would like to demonstrate it on the low level, first let's compile everything to LLVM IR:

```sh
$ clang-13 -S -emit-llvm c-bindings/bindings.c -o c-bindings/bindings.ll
$ grep -F "char__" c-bindings/bindings.ll
define dso_local i32 @char__new(i8 zeroext %0, i8 zeroext %1, i8 zeroext %2, i8 zeroext %3) #0 {
define dso_local zeroext i8 @char__at(%struct.Char* %0, i8 zeroext %1) #0 {
define dso_local void @char__drop(%struct.Char* %0) #0 {
```

^ that was the file with bindings implementation, C part. It defines all `char__` functions.

```sh
$ cd rust-foo
$ rustc --crate-name rust_foo src/lib.rs --crate-type staticlib --emit=llvm-ir --cfg 'feature="default"' --cfg 'feature="external"'
$ cd ..
$ grep -F "char__" rust-foo/rust_foo.ll
  %8 = call i32 @char__new(i8 zeroext %_8, i8 zeroext %_10, i8 zeroext %_12, i8 zeroext %_14)
declare i32 @char__new(i8 zeroext, i8 zeroext, i8 zeroext, i8 zeroext) unnamed_addr #1
```

^ that was the Rust part. It declares (as external) and calls some of our `char__` functions.

Let's link them together:

```sh
$ llvm-link-13 c-bindings/bindings.ll rust-foo/rust_foo.ll -S -o merged.ll
$ grep -F "char__" merged.ll
define dso_local i32 @char__new(i8 zeroext %0, i8 zeroext %1, i8 zeroext %2, i8 zeroext %3) #0 {
define dso_local zeroext i8 @char__at(%struct.Char* %0, i8 zeroext %1) #0 {
define dso_local void @char__drop(%struct.Char* %0) #0 {
  %8 = call i32 @char__new(i8 zeroext %_8, i8 zeroext %_10, i8 zeroext %_12, i8 zeroext %_14)
```

These 2 modules have been merged, time to optimize them together:

```sh
$ opt-13 -O3 merged.ll -S -o optimized.ll
$ grep -F "char__" optimized.ll
define dso_local i32 @char__new(i8 zeroext %0, i8 zeroext %1, i8 zeroext %2, i8 zeroext %3) local_unnamed_addr #0 {
define dso_local zeroext i8 @char__at(%struct.Char* nocapture readonly %0, i8 zeroext %1) local_unnamed_addr #1 {
define dso_local void @char__drop(%struct.Char* nocapture %0) local_unnamed_addr #0 {
```

Now we have only definitions, actual calls have been successfully inlined. Now we can compile it to an object file to see a final result:

```sh
$ llc-13 -O3 -filetype=obj optimized.ll -o optimized.o
$ clang-13 test.c optimized.o -O3 -o test-runner
$ objdump -D test-runner | grep "call" | grep "char__"
# no output
$ objdump -D test-runner | grep "call" | grep "char_list"
  406bba: e8 01 01 00 00        callq  406cc0 <char_list__drop>
  406d21: e8 fa fe ff ff        callq  406c20 <char_list__new>
```

As you can see all `char__new` or `char__at` calls have been inlined, `char_list__drop` and `char_list__new` haven't, because of how LLVM decides on what should or should not be inlined. Anyway, it works.

Of course, there's an easier way to get the same result:

```sh
$ export RUSTFLAGS="-Clinker-plugin-lto -Clinker=clang -Clink-arg=-fuse-ld=lld"
$ export CFLAGS="-flto"
```

^ this should be enough to get the same result. By adding `-Clinker-plugin-lto` we ask Rust to compile all object files to LLVM IR:

```sh
$ RUSTFLAGS="-Clinker-plugin-lto -Clinker=clang-13 -Clink-arg=-fuse-ld=lld" cargo build --features=external

$ mkdir objects
$ cd objects
$ ar x ../target/release/librust_foo.a
$ ls -l
# a ton of object files, that's what static library is about

$ file *.o

... snip ...
popcountti2.o:                                                                              ELF 64-bit LSB relocatable, x86-64, version 1 (SYSV), with debug_info, not stripped
powixf2.o:                                                                                  ELF 64-bit LSB relocatable, x86-64, version 1 (SYSV), with debug_info, not stripped
rust_foo-0f1418f7365bf15b.2nf5cb5qlud0f6qs.rcgu.o:                                          LLVM IR bitcode
rust_foo-0f1418f7365bf15b.rust_foo.f999abab-cgu.0.rcgu.o:                                   LLVM IR bitcode
rust_foo-0f1418f7365bf15b.rust_foo.f999abab-cgu.1.rcgu.o:                                   LLVM IR bitcode
rust_foo-0f1418f7365bf15b.rust_foo.f999abab-cgu.2.rcgu.o:                                   LLVM IR bitcode
rust_foo-0f1418f7365bf15b.rust_foo.f999abab-cgu.3.rcgu.o:                                   LLVM IR bitcode
rustc_demangle-7f98f837d3579544.rustc_demangle.5563b4d3-cgu.0.rcgu.o:                       ELF 64-bit LSB relocatable, x86-64, version 1 (SYSV), with debug_info, not stripped
... snip ...
```

Rust standard library is compiled directly to object files, but our code is `LLVM IR bitcode`. **This way we can get zero-cost bindings defined externally**.

Of course, it's not gonna work with Ruby or Node.js, both of them have a giant `libruby.so` (or `libnode.so`) that defines all functions and constants that your extension relies on. The extension itself is compiled with `-Wl,-undefined,dynamic_lookup` and symbol lookup is performed at runtime. I feel like technically it's possible, the entire Ruby/Node.js runtime could be compiled to a static `libruby.a`/`libnode.a` that defines all VM objects as external (because that's a singleton that we need to hook into), but all functions can provide their implementation (of course, in LLVM IR format), and so they can be inlined into our bindings implementation. I haven't experimented with it yet, and honestly I'm not going to :) If you know anything about existing discussions around it, please, ping me on Twitter.

## The whole picture, again

Demo repository is available [here](https://github.com/iliabylich/writing-bindings-upside-down)

To sum up:

1. Library maintainers can change their library to accept **"pluggable" external primitives**. If no external primitives provided native version is used.
2. Library maintainers can **provide headers** that define what should be provided (i.e. a set of functions that take/return primitives). This is the definition of the contract.
3. Authors of bindings **define structure as they want** (i.e. instead of vector they are free to use linked lists of hashmaps)
4. Authors of bindings **implement functions defined in headers**. This is the implementation of the contract.
5. To get full bindings the library is linked together with bindings. There might be a need to build a wrapper around functions defined in the library, but that's trivial as these functions return objects that are "native" for bindings.

Pros:

+ To implement bindings you don't wrap every single function, instead you focus on implementing types and contract around them. Moreover, the contract is quite simple and many languages have a rich standard library, so implementing these library-defined functions can be very easy.
+ Libraries can provide headers, so it's hard to make a mistake.
+ Libraries usually have a decent test suite, it can be used to verify bindings for correctness and memory leaks.

Cons:

+ The library needs quite a lot of changes to support "external" primitives. That's not an easy task.
+ The library needs to ship with a "reference implementation", to verify that existing external functions are enough to build everything.
+ There are minor differences between standard types in languages, for example C++ has a small string optimization, and small strings are not "containers". On the other side Rust strings are true containers, and library developers may expect "external" string to be a container, too, but on the bindings side `std::string` from C++ does not fully match the contract (i.e. you can't borrow a `char *` from `std::string` and expect it to live as long as the string lives). In some cases it can become really hard to track and fix.
