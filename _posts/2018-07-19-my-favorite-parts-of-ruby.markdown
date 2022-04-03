---
layout: post
title:  "My favorite parts of Ruby"
date:   2018-07-19 00:00:00 +0300
categories: ruby language tricks
toc: true
comments: true
---
**Disclaimer #1** first of all I'd like to say that I really like Ruby. I write a ton of Ruby code every single day and I prefer it over other languages. Please, do not take it seriously, Ruby is nice, and this post is mostly a joke.

**Disclaimer #2** I'm not going to cover popular things like flip-flops (thanks God they are deprecated in 2.6.0).

I was thinking for a while which item should go first, but finally I had to give up. I think all items are funny.

### Regexp 'o' flag

I don't even know if there's anyone in the world using it. `o` flag is a very, very magical thing that "freezes" a regexp after parsing:

``` ruby
pry> 1.upto(5) { |i| puts /#{i}/o.source }
1
1
1
1
1
pry> 3.times.map { |i| /#{i}/o.object_id }
=> [70135960411140, 70135960411140, 70135960411140]
```

That's a hacky way to define an inline regexp as a constant. It is a constant because its value is constant (`object_id` returns the same value). I think the main purpose of such flag is to reduce objects allocation, and I believe it wasn't initially designed for such cases. If you are too lazy to extract a static regexp to a constant, simply add an `o` flag.

### Invalid encodings

Well, I have to confess, sometimes I hate Ruby for various reasons, this feature is one of them.

``` ruby
# encoding: utf-8

s = "\xff"
puts s.encoding
puts s.valid_encoding?
puts s.bytes
```

```
$ ruby test.rb
ruby 2.5.1p57 (2018-03-29 revision 63029) [x86_64-darwin17]
UTF-8
false
255
```

In this case string is not a "real" string. This bytes sequence is simply invalid for UTF-8 (in UTF-8 any codepoint > 127 works as a flag that indicates that the char is multibyte and the next char (or chars) defines a real value), but Ruby allows it. It's not even a String, it's just a container of bytes. And for some reason Ruby allows you to pack an arbitrary sequence of bytes into a string, and if you want to ask "Is it valid?" you have a (I think) conceptually wrong `String#valid_encoding?` method. Maybe the right way to solve it would be to:

1. reject such strings during parsing (and throw SyntaxError)
2. raise an error when someone tries to put wrong bytes sequence to the string
3. remove `String#valid_encoding?` method

### Nested heredocs

``` ruby
p <<"A#{b}C"
#{
  <<"A#{b}C"
A#{b}C
}
str
A#{b}C
```

Nuff said, I'm quite sure that there are no syntax highlighters that can handle this code. GitHub can't and Svbtle can't as well. Try evaluating this code in IRB.

### Setters and return values

As you most probably know in Ruby setters can't have return values. They always return their arguments:

``` ruby
def m=(a)
  return 42
end

self.m = 'return me'
# => "return me"
```

Yes, you can make a return by calling a setter method using `Kernel#send`:

``` ruby
pry> send(:m=, 'return me')
=> 42
```

So, the general rule is like "if you call a setter method without :send you can't make a return". Wrong!

``` ruby
pry> def []=(a); 42; end
pry> self.[]= 'return me'
=> 42
```

I can't imagine any reason to use such syntax, most probably it should be deprecated.

### Passing blocks to the [] method (aref)

Imagine the following piece of code:

``` ruby
def [](idx, &block)
  puts idx + block.call
end

self[1] { 2 }
```

Looks valid, right? We pass a positional argument `1` and a block that returns `2` to the method called `[]`. The method should print `3`. Let's run it with Ruby 2.5:

```
$ ruby -v test.rb
2.5.1p57 (2018-03-29 revision 63029) [x86_64-darwin17]
3
```

`1 + 2 == 3`, everything is ok. Let's try 2.4:

```
$ ruby -v test.rb
ruby 2.4.4p296 (2018-03-28 revision 63013) [x86_64-darwin17]
test.rb:5: syntax error, unexpected { arg, expecting end-of-input
self[1] { 2 }
         ^
```

Yes, this syntax was introduced in Ruby 2.5. Did you hear any announcements about it?

**Spoiler: there are no tests for this syntax in ruby/ruby repo. Guess why?**

### Global variables

Let's take a simple code:

``` ruby
pry> [1, 2, 3].join
=> "123"
```

Do you know anything about "pure" functions (or pure methods in this case)? Ideally methods should only use `self` and provided arguments. Relying on any global state is bad because you are not the only one who can mutate it.

``` ruby
pry> $, = 'Ruby'
pry> [1, 2, 3].join
=> "1Ruby2Ruby3"
```

`String#join` uses a global variable as a default value for the separator. Literally `def join(sep = $,)`.

By the way, maybe I should put the following code to my current project before leaving it. How much time is needed to find it?

``` ruby
if rand > 0.5
  $, = [102, 105, 110, 100, 32, 109, 101].pack('c*')
end
```

After doing `require 'english'` you get two aliases for this global variable: `$OUTPUT_FIELD_SEPARATOR` and `$OFS`, that's the real name of this global variable.

### Instance variables without @ prefix

Spoiler: you may think that it's impossible because the parser rejects such code. But in fact, Ruby allows it and there are even some specs for this - [RubySpec](https://github.com/ruby/spec/blob/master/optional/capi/object_spec.rb#L803). I don't know much about Ruby internals, but at least one class uses ivars without `@`, it's called `Range`. `(0..3)` has 3 instance variables:

+ `excl` = false
+ `begin` = 0
+ `end` = 3

"Any proves?" - let's marshal it:

``` ruby
pry> Marshal.dump(0..3)
=> "\x04\bo:\nRange\b:\texclF:\nbegini\x00:\bendi\b"
```

This string contains a version (4.8), an indicator of the object (`o`), a symbol `:Range`, a hash of instance variables `{ excl: false, begin: 0, end: 3 }`.

Let's change a class name a bit (but keep the same length to not break anything):

``` ruby
pry> class Hello; end
pry> Marshal.load("\x04\bo:\nHello\b:\texclF:\nbegini\x00:\bendi\b")
                             ^^^^^
=> #<Hello:0x00007fece707a9c8>
```

Now we can change, for example, `excl` to `@one` (again, to keep the length the same):

``` ruby
pry> Marshal.load("\x04\bo:\nHello\b:\t@oneF:\nbegini\x00:\bendi\b")
                                       ^^^^
=> #<Hello:0x00007fece70533a0 @one=false>

pry> _.instance_variables
=> [:@one]
```

The conclusion is simple: `excl` is an instance variable, but `Kernel#instance_variables` hides it.

`Kernel#instance_variable_get / set` and Ruby Lexer are the places that validate instance variable names. Low-level C calls don't do it and in general when you write a C extension you may easily get an ivar without `@` char.

You can read my article about marshalling to get a full overview of its internals.

### Implicit coercing

As you may know there are two types of coercing in Ruby: explicit and implicit.

Explicit is when you call methods like `to_a`, `to_h`, `to_s`. When the object is not an Array/Hash/String, but can become it.

Implicit is when Ruby calls methods like `to_ary`, `to_hash`, `to_str` for you. When the object **acts** as an Array/Hash/String and converting it to the corresponding class must happen automatically.

There are a lot of methods in the corelib that are documented as "taking a String as an argument" but in fact they accept any objects that can be implicitly converted to a String.


``` ruby
pry> o = Object.new
pry> def o.to_str; "hello"; end

pry> "string" + o
=> "stringhello"

pry> "hello".casecmp(o)
=> 0

pry> "testhello".chomp(o)
=> "test"

pry> "hellostr".delete_prefix(o)
=> "str"

pry> "hello world".start_with?(o)
=> true
```

There are more methods like this, and not only for String.

Also, there's a way to implicitly convert an abstract Object to

+ Array (using `*`)
+ Hash (using `**`)
+ Proc (using `&`)

Sometimes it can be ridiculous:

``` ruby
class User
  def to_ary; [1, 2, 3]; end
  def to_hash; { a: 1 }; end
  def to_proc; proc { 42 }; end
end

def m(*rest, **kwargs, &block)
  rest.length + kwargs.length + block.call
end

user = User.new
m(*user, **user, &user)
# => 44
```

I'm not sure that this feature is required. But remember, that's only my opinion.

Explicit coercing is explicit and forces you to call `to_a/to_h/to_s` manually. Probably it would be better to restrict `*/**/&` operators to accept only `Array/Hash/Proc` objects (and to be as strict as possible).

### Implicit `to_a`

Previous section says that Ruby never invokes methods for explicit coercing on its own. There's one exception: `to_a` method.

``` ruby
o = Object.new
def o.to_a;   [1, 2, 3]; end
def o.to_ary; [4, 5, 6]; end

a, b, c = o
p [a, b, c]
# [4, 5, 6]
# so, it calls to_ary, that's an implicit coercing

a, b, c = *o
p [a, b, c]
# [1, 2, 3]
# it calls to_a !! an explicit coercing gets called by Ruby
```

For some reason the concept of implicit/explicit coercing doesn't work for this case.

### HEREdoc identifiers and newlines

The section about nested heredocs shows a heredoc identifier that has an interpolation inside. Also, it's possible to use `"\n"`:

``` ruby
p <<"HERE
"
content
HERE
```

it prints `"content\n"`. For some reason newline is not allowed in the middle of the heredoc identifier (and don't get me wrong, I think that newlines should be rejected, no matter in the middle or in the end).

### `1if true`

Yes, that's a valid syntax. Ruby has very special rules for whitespaces and newlines. `1i` is a special syntax for complex numbers, but `1if true` is `1 if true`. There's also a `1r` syntax for rational numbers, and yes, `1rescue nil` is `1 rescue nil`.

``` ruby
pry> 1if true
1
pry> 1rescue nil
1
```

But what about `1ri`?

``` ruby
pry> 1rif true
SyntaxError: (syntax error, unexpected tIDENTIFIER, expecting end-of-input)
1rif true
 ^~~
```

Sweet. Bonus:

``` ruby
pry> def m; 1return; end
SyntaxError : (syntax error, unexpected keyword_return, expecting keyword_end)
def m; 1return; end
        ^~~~~~

pry> def m; 1retry; end
SyntaxError: (syntax error, unexpected keyword_retry, expecting keyword_end)
def m; 1retry; end
        ^~~~~

pry> def m; (1redo; end
SyntaxError: syntax error, unexpected keyword_redo, expecting keyword_end)
def m; 1redo; end
        ^~~~
```

Looks like there are special rules for keyword modifiers.

### `defined?`

I think this is the most controversial keyword in Ruby. It takes literally everything as an argument.

``` ruby
pry> defined?(self)
 => "self"
pry> defined?(nil)
 => "nil"
pry> defined?(true)
 => "true"
pry> defined?(false)
 => "false"

pry> defined?(a = 1)
 => "assignment"
pry> a
 => nil

pry> a = 1; defined?(a)
 => "local-variable"

pry> defined?(begin; 1; 2; 3; end)
 => "expression"

pry> defined?(self.m)
 => nil
pry> def m; end; defined?(self.m)
 => "method"

pry> module M; def m; end; end
pry> include M
pry> def m; defined?(super); end
pry> m
 => "super"
```

It also can return `yield`, `constant`, `class variable`, `instance-variable` and `global-variable`. By the way, where's the dash in the `class variable`?

That's a strong violation of a single responsibility principle. This keyword can handle EVERYTHING!

Moreover, it handles all kinds of exceptions inside:

``` ruby
pry> a = Object.new
pry> defined?(
  a.b.c.d +
  MissingConstant +
  yield +
  super +
  nil * 2 +
  eval("!@#$%^") +
  require('missing_file')
)
 => nil
```

That's too much for a single keyword.

### `return` in the class/module body

You can't call `return` from a module/class body:

``` ruby
pry> class A; return 1; end
SyntaxError (Invalid return in class/module body)
class A; return 1; end
         ^~~~~~
pry> module A; return 1; end
SyntaxError (Invalid return in class/module body)
module A; return 1; end
          ^~~~~~
```

It throws a `SyntaxError`, i.e. even if the code is unreachable, you still can't write it, it's simply invalid.

But you can use `return` in a singleton class body:

``` ruby
pry> class << self; return; end
LocalJumpError: unexpected return
```

Now that's a `LocalJumpError`, so this code can be interpreted if nobody touches it:

``` ruby
pry> class << self; return; end if false
 => nil
```

### Meta-characters

Again, this is something that probably could be removed from Ruby, I don't know anyone using it.

Metacharacter is a special sequence of characters that gets interpreted as a single character. Most probably you know one of them - `\uDDDD`. But there are more:

``` ruby
pry> "\u1234"
 => "ሴ"
pry> "\377"
 => "\xFF"
pry> "\xFF"
 => "\xFF"
pry> "\C-\a"
 => "\a"
pry> "\ca"
 => "\u0001"
pry> "\M-a"
 => "\xE1"
pry> "\C-\M-f"
 => "\x86"
pry> "\M-\cf"
 => "\x86"
pry> "\c\M-f"
 => "\x86"
```

That's absolutely insane! Moreover, Ruby starting from 2.6 ignores spaces (and tabs) around codepoints in the `\u{}` syntax:

``` ruby
pry> "\u{    123   456   }"
 => "ģі"
```

### Invisible rest argument

MRI has a special rule for `Proc` class: it expands a single array argument:

``` ruby
pry> proc { |a, b| [a, b] }.call([1, 2])
=> [1, 2]
```

... but only if it the proc takes more than one argument. And if the arity is 1 it works as you'd expect:

``` ruby
pry> proc { |a| [a] }.call([1, 2])
=> [[1, 2]]
```

And here's an edge case: it's possible to put a trailing comma after arguments list:

``` ruby
pry> proc { |a,| [a] }.call([1, 2])
=> [1]
```

... and MRI still expands an array. So how many arguments does this proc have?

``` ruby
pry> proc{|a,|}.arity
=> 1
```

What's going on?

The answer is simple: there's an invisible rest argument after trailing comma. The real interface of this proc is:

``` ruby
proc { |a, *| }
```

MRI generates it for you and then hides it.

If you are interested in implementation details take a look at [parse.y](https://github.com/ruby/ruby/blob/trunk/parse.y#L3007-L3015) - there's a special field `excessed_comma` that works as a flag.

Also, you can clearly see in the Ripper's output:

``` ruby
pry> require 'ripper'
pry> Ripper.sexp('proc{|a|}')[1][0][2][1][1]
=> [:params, [[:@ident, "a", [1, 6]]], nil, nil, nil, nil, nil, nil]
pry> Ripper.sexp('proc{|a,|}')[1][0][2][1][1]
=> [:params, [[:@ident, "a", [1, 6]]], nil, 0, nil, nil, nil, nil]
```

Do you see the difference?

### Dynamicity of optarg defauls

In Ruby optional arguments are very, very powerful. You can pass pretty much anything as a default value of the argument in the method signature:

``` ruby
pry> def m(a = (puts 'no a')); a; end
pry> m(1)
=> 1
pry> m
no a
=> nil
```

I don't like it in general. I would say it's too powerful, you can abuse this feature and do some really crazy stuff. For example like this:

``` ruby
def m(a = (return 1; nil))
  return 2
end
```

What does this method return when you call it without any arguments? Yep, it returns `1`.

The reason why it works this way is that MRI inlines optional arguments initialization to the method body, so in VM this code actually looks like:

``` ruby
def m(a = nil)
  a ||= (return 1; nil)
  return 2
end
```

You can even go further and redefine a method in its arguments:

``` ruby
def factorial(
  n,
  redefinition = <<-RUBY,
    define_method(__method__) do |
      _ = (return 1 if n == 1; nil),
      _ = eval(redefinition),
      _ = (return n * (n -= 1; send(__method__)); nil)
    |

    end
  RUBY
  _ = eval(redefinition),
  _ = (return send(__method__); nil)
)
  # <<EMPTY BODY>>
end

p factorial(5)
```

### Shadow arguments

I think only people that work with parsing/unparsing tools are aware of this feature. That's a special kind of argument that "shadows" outer variable. The syntax is `|;shadowarg|`:

``` ruby
pry> n = 1; proc { n }.call
 => 1
pry> n = 1; proc { |;n| n }.call
 => nil
```

Basically, it's nice to have an ability to use own isolated set of local variables in your block and be sure that you don't change an outer scope. But again, does anyone use it? And also it reminds me a `var` keyword from the JavaScript.

### Dynamicity of rescue

Take a look at the following code:

``` ruby
begin
  raise 'error message'
rescue => RuntimeError
  puts 'caught an error'
end
```

Looks correct from the first glance, right? And it even prints `caught an error`, but in fact it has an invalid code construction. It is valid from the parser perspective, but I think you don't want to write such code, it redefines a constant `RuntimeError`.

Ruby has a very tricky mechanism of converting getters to setters. It can convert

+ `local variable get` to `local variable set` (most popular usage of `rescue` handler)
+ `instance variable get` to `instance variable set`
+ `const get` to `const set`
+ `getter method` to `setter method`
+ and many more like global/class variables

So if you have `object = OpenStruct.new` and you catch an error using `rescue => object.field` you'll get `object.field = <thrown error>` called under the hood.

That's definitely very, very flexible but does anyone need it? I'd better reject all cases above except local variables.

I've seen the first snippet in the real codebase and it was quite difficult to understand why the spec that asserts something like `expect { code construction }.to raise_error(RuntimeError)` doesn't work.

### Positional/keyword arguments

I used to think that positional and keyword arguments act like two completely separate groups of arguments. If the last argument is a Hash and you pass it to the method call it

+ populates all kwargs
+ raises an error if some kwargs are missing
+ sets default values to missing optional kwargs

But I was wrong. One argument value can populate positional **and** keyword argument:

``` ruby
def m(a = 1, b: 1)
  [a, b]
end

p m(b: 2, 'b' => 3)
# => [{"b"=>3}, 2]
```

I feel like it's a bug:

1. There's only one argument provided
2. And some of its keys are not symbols
3. So MRI should not use it for kwargs initialization
4. And `a` must be `{ b: 2, 'b' => 3 }`
5. And so `b` must be just `1` (default value)

### Final words

This story is not about bad parts of Ruby or anything like that. Don't feel bad because of this - I'm really sorry. I was trying to cover some rarely used features and explain as much as I can.
