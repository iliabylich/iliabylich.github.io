---
title: "Evaluating Ruby in Ruby"
date: "2020-01-26"
cover: ""
---

# TL;DR

This article is about instruction sequences and evaluating them using pure Ruby.

The repository is available [here](https://github.com/iliabylich/my.rb).

> Is it a Ruby implementation?

No. It's just a runner of instructions. It is similar to MRI's virtual machine, but it lacks many features and it's 100 times slower.

> Can I use it in my applications?

Of course, no. Well, if you want.

> Does it work at all?

Yes, and it even passes most language specs from RubySpec test suite.

# How Ruby evaluates your code.

Well, I think I should start with explaining basics. There is a plenty of articles about it, so I'll be short:

1. First, Ruby parses your code into AST (`parse.y`)
2. Then, it compiles it into instruction sequence (`compile.c`)
3. And every time when you let's say call a method it evaluates these instructions. (`insn.def`, `vm_eval.c`, `vm_insnhelper.c`)

Long time ago there was no YARV and Ruby used to evaluate AST.

`1+2` is just a small syntax tree with a `send` node in the root and two children: `int(1)` and `int(2)`. To evaluate it you need to "traverse" it by walking down recursively. Primitive nodes are getting substituted with values (integers 1 and 2 respectively), `send` node calls `+` on pre-reduced children and returns `3`.

On the other side, YARV is a stack-based Virtual Machine (VM) that evaluates an assembly-like language that consist of more than 100 predefined instructions that store their input/output values on the VM's stack.

These instructions have arguments, some of them are primitive values, some are inlined Ruby objects, some are flags.

You can view these instructions by running Ruby with `--dump=insns` flag:

```ruby
$ ruby --dump=insns -e 'puts 2 + 3'
== disasm: #<ISeq:<main>@-e:1 (1,0)-(1,10)> (catch: FALSE)
0000 putself                                                          (   1)[Li]
0001 putobject                    2
0003 putobject                    3
0005 opt_plus                     <callinfo!mid:+, argc:1, ARGS_SIMPLE>, <callcache>
0008 opt_send_without_block       <callinfo!mid:puts, argc:1, FCALL|ARGS_SIMPLE>, <callcache>
0011 leave
```

As you can see, there are 5 instructions:

1. `putself` - pushes `self` at the top of the stack
2. `putobject` - pushes given object (numbers 2 an 3)
3. `opt_plus` - optimized instruction for `+` method
4. `opt_send_without_block` - optimized instruction for a generic method call without block
5. `leave` - an equivalent of `return`

# The structure of the instruction sequence (ISeq)

Let's start with the example above.

MRI has an API to compile an arbitrary code into instructions:

```ruby
> pp RubyVM::InstructionSequence.compile("2+3").to_a
["YARVInstructionSequence/SimpleDataFormat",
 2,
 6,
 1,
 {:arg_size=>0,
  :local_size=>0,
  :stack_max=>2,
  :node_id=>4,
  :code_location=>[1, 0, 1, 3]},
 "<compiled>",
 "<compiled>",
 "<compiled>",
 1,
 :top,
 [],
 {},
 [],
 [1,
  :RUBY_EVENT_LINE,
  [:putobject, 2],
  [:putobject, 3],
  [:opt_plus, {:mid=>:+, :flag=>16, :orig_argc=>1}, false],
  [:leave]]]
```

Our code gets compiled into an array where:

+ The first element returns the type of the instruction sequence
+ Then comes `MAJOR`/`MINOR`/`PATCH` parts of the Ruby version that was used to compile it
+ Then comes the hash with a meta information about this iseq
+ Then comes relative/full/some other file name. It is possible to override them similar to passing `file/line` arguments to `{class,module}_eval` (`Object.module_eval("[__FILE__, __LINE__]", '(file)', 42)`)
+ Then comes the line where the code begins (1 by default)
+ Then comes the type of the iseq (`:top` here means the code was parsed and compiled as a top-level block of code)
+ Then we have a few empty arrays and hashes (we will return to them later)
+ And the last item is an array of instructions

Each instruction is either:

+ a number
+ a special symbol
+ or an array

Only arrays are "real" instructions, numbers and symbols are special debug entries that are used internally by the VM. In our case a number followed by the `:RUBY_EVENT_LINE` is a mark that MRI uses to know what is the number of the line that is currently being interpreted (for example, all backtrace entries include these numbers)

# Building a PoC

How can we evaluate instructions above? Well, we definitely need a stack:

```ruby
$stack = []
```

Then, we need to iterate over instructions (the last item of the iseq) and evaluate them one by one. We could write a giant `case-when`, but I promise that it won't fit on 10 screens. Let's use some meta-programming and dynamic dispatching:

```ruby
def execute_insn(insn)
  name, *payload = insn
  send(:"execute_#{name}", payload)
end
```

This way we need to write a method per instruction type, let's start with `putobject`:

```ruby
def execute_putobject(object)
  $stack.push(object)
end

def execute_opt_plus(options, flag)
  rhs = $stack.pop
  lhs = $stack.pop
  $stack.push(lhs + rhs)
end

def execute_leave
  # nothing here so far
end
```

This code is enough to get `5` in the stack once it's executed. Here's the runner:

```ruby
iseq = RubyVM::InstructionSequence.compile("2+3").to_a
insns = iseq[13].grep(Array) # ignore numbers and symbols for now
insns.each { |insn| execute_insn(insn) }
pp $stack
```

You should see `[5]`.

All instructions above simply pull some values from the stack, do some computations and push the result back.

# Self

Let's think about `self` for a minute:

```ruby
> pp RubyVM::InstructionSequence.compile("self").to_a[13]
[1, :RUBY_EVENT_LINE, [:putself], [:leave]]
```

`self` is stored somewhere internally, and even more, it's dynamic:

```ruby
puts self
class X
  puts self
end
```

The first `puts self` prints `main`, while the second one prints `X`.

Here comes the concept of frames.

# Frames

Frame is an object inside the virtual machine that represents a closure. Or a `binding`. It's an isolated "world" with its own set of locals, its own `self`, its own `file`/`line` information, it's own `rescue` and `ensure` handlers etc.

Frame also has a type:

1. it can be a `top` frame that wraps all the code in your file. All variables set in the global scope of your file belong to the top frame. Each parsed and evaluated file creates its own top frame.
2. it can be a `method` frame. All methods create it. One method frame per one method call.
3. it can be a `block` frame. All blocks and lambdas create it. One block frame per one block call.
4. it can be a `class` frame, however it does not mean that instantiating a class creates it. The whole `class X; ... end` does it. Later when you do `X.new` Ruby does not create any `class` frames. This frame represents a class definition.
5. it can be a `rescue` frame that represents a code inside `rescue => e; <...>; end` block
6. it can be an `ensure` frame (well, I'm sure you get it)
7. there are also `module` (for a module body), `sclass` (for a singleton class) and a very unique `eval` frame.

And they are stored internally as a stack.

When you invoke `caller` method you see this stack (or some information based on it). Each entry in the error backtrace is based on the state of this stack when the error is thrown.

OK, let's write some code to extend our VM:

```ruby
class FrameClass
  COMMON_FRAME_ATTRIBUTES = %i[
    _self
    nesting
    locals
    file
    line
    name
  ].freeze

  def self.new(*arguments, &block)
    Struct.new(
      *COMMON_FRAME_ATTRIBUTES,
      *arguments,
      keyword_init: true
    ) do
      class_eval(&block)

      def self.new(iseq:, **attributes)
        instance = allocate

        instance.file = iseq.file
        instance.line = iseq.line
        instance.name = iseq.name
        instance.locals = {}

        instance.send(:initialize, **attributes)

        instance
      end
    end
  end
end
```

I like builders and I don't like inheritance (the fact that frames share some common attributes does not mean that they should be inherited from an `AbstractFrame` class; also I don't like abstract classes, they are dead by definition)

Here's the first version of the `TopFrame`:

```ruby
TopFrame = FrameClass.new do
  def initialize(**)
    self._self = TOPLEVEL_BINDING.eval('self')
    self.nesting = [Object]
  end

  def pretty_name
    "TOP #{file}"
  end
end
```

Let's walk through the code:

1. each frame has a common set of attributes:
  1. `_self` - what `self` returns inside the frame
  2. `nesting` - what `Module.nesting` returns (used for the relative constant lookup)
  3. `locals` - a set of local variables
  4. `file` and `line` - currently running `__FILE__:__LINE__`
  5. `name` - a human-readable name of the frame, we will use it mostly for debugging
2. `FrameClass` is a builder that is capable of building a custom `Frame` class (similar to `Struct` class)
4. `FrameClass.new` takes:
  1. a list of custom frame-class-specific attributes
  2. and a block that is evaluated in a context of the created frame class

So, the `TopFrame` class is a `Struct`-like class that:

1. has a constructor that takes **only** common attributes (because we haven't specified any in `FrameClass.new`)
2. has a custom behavior in the constructor that sets `_self` and `nesting`
3. has a custom `pretty_name` instance method

We will create as many classes as we need to cover all kinds of frames (I will return to it later, I promise)

# Wrapping the ISeq

I don't like working with plain arrays, and as I mentioned above there's a ton of useful information in the instruction sequence that we get from `RubyVM::InstructionSequence.compile("code").to_a`.

Let's create a wrapper that knows the meaning of array items:

```ruby
class ISeq
  attr_reader :insns

  def initialize(ruby_iseq)
    @ruby_iseq = ruby_iseq
    @insns = @ruby_iseq[13].dup
  end

  def file
    @ruby_iseq[6]
  end

  def line
    @ruby_iseq[8]
  end

  def kind
    @ruby_iseq[9]
  end

  def name
    @ruby_iseq[5]
  end

  def lvar_names
    @ruby_iseq[10]
  end

  def args_info
    @ruby_iseq[11]
  end
end
```

Instance methods are self-descriptive, but just in case:

1. `file`/`line` return file/line where the iseq has been created
2. `kind` returns a Symbol that we will later use to distinguish frames (`:top` for a `TopFrame`)
3. `insns` returns a list of instructions
4. `name` is an internal name of the frame that is used in stacktraces (for `class X; end` it returns `<class:X>`)
5. `lvar_names` is an array of all local variable names that are used in the frame
6. `args_info` is a special Hash with a meta-information about arguments (empty for all frames except methods)

# Frame stack

Frames are organized as a stack internally, every time when we enter a frame Ruby pushes it on a stack. When the frame ends (i.e. when its list of instruction ends or there's a special `[:leave]` instruction) it pops it.

```ruby
class FrameStack
  attr_reader :stack

  include Enumerable

  def initialize
    @stack = []
  end

  def each
    return to_enum(:each) unless block_given?
    @stack.each { |frame| yield frame }
  end

  def push(frame)
    @stack << frame
    frame
  end

  def push_top(**args)
    push TopFrame.new(**args)
  end

  def pop
    @stack.pop
  end

  def top
    @stack.last
  end

  def size
    @stack.size
  end

  def empty?
    @stack.empty?
  end
end
```

Each entry in the stack is a frame that we entered at some point, so we can quickly build a `caller`:

```ruby
class BacktraceEntry < String
  def initialize(frame)
    super("#{frame.file}:#{frame.line}:in `#{frame.name}'")
  end
end

stack = FrameStack.new
code = '2 + 3'
iseq = ISeq.new(RubyVM::InstructionSequence.compile(code, 'test.rb', '/path/to/test.rb', 42).to_a)

stack.push_top(iseq: iseq)
caller = stack.map { |frame| BacktraceEntry.new(frame) }.join("\n")
puts caller
# => "/path/to/test.rb:42: in `<compiled>'"
```

# Writing the evaluator

Let's write it in a "script style" (it is simplified for a good reason, real code is much more complicated):

```ruby
iseq = ISeq.new(RubyVM::InstructionSequence.compile_file('path/to/file.rb').to_a)
frame_stack = FrameStack.new
stack = []

frame_stack.push_top(iseq: iseq)

until frame_stack.empty?
  current_frame = frame_stack.top
  if current_frame.nil?
    break
  end

  if current_frame.insns.empty?
    frame_stack.pop_frame
    next
  end

  current_insn = current_frame.insns.shift
  execute_insn(current_insn)
end
```

Generally speaking, the code above is the core of the VM. Once it's executed both `frame_stack` and `stack` must be empty. I added a bunch of consistency checks in my implementation, but for the sake of simplicity I'm going to omit them here.

# Instructions

I'll try to be short here, there are about 100 instructions in Ruby, and some of them look similar.

## `putself`, `putobject`, `putnil`, `putstring`, `putiseq`

All of these guys push a simple object at the top of the stack. `putnil` pushes a known global `nil` object, others have an argument that is used in `stack.push(argument)`

## optimized instructions like `opt_plus`

Ruby has a mode (that is turned on by default) that optimizes some frequently used method calls, like `+` or `.size`. It is possible to turn them off by manipulating `RubyVM::InstructionSequence.compile_option` (if you set `:specialized_instruction` to `false` you'll get a normal method call instead of the specialized instruction).

All of them do one specific thing, here's an example of the `opt_size`:

```ruby
def execute_opt_size(_)
  push(pop.size)
end
```

Of course we do it this way because we cannot optimize it. MRI does a different thing:

1. if there's a string/array/hash on top of the stack
2. and `String#size` (or `Array#size` if it's an array) is not redefined
3. then it **directly** calls a C method `rb_str_length` (or `RARRAY_LEN` if it's an array)
4. otherwise (if it's an object of some other type or a method has been redefined) it calls a method through the regular method dispatch mechanism (which is obviously slower)

We could do the same sequence of steps, but we can't invoke a C method, and so calling a check + `.size` afterwards is even slower. It's better for us to fall to the slow branch from the beginning.

You can print all available specialized instructions by running

```ruby
RubyVM::INSTRUCTION_NAMES.grep(/\Aopt_/)
```

On Ruby 2.6.4 there are 34 of them.

## `opt_send_without_block` (or `send` if specialized instructions are disabled)

This is an instruction that is used to invoke methods. `puts 123` looks like this:

```ruby
[:opt_send_without_block, {:mid=>:puts, :flag=>20, :orig_argc=>1}, false]
```

It has 2 arguments:

1. a hash with options
  1. `mid` - a method ID (method name)
  2. `flag` - a bitmask with a metadata about the invocation
  3. `orig_argc` - a number of arguments passed to a method call
2. a boolean flag that is called `CALL_DATA` in C. I have no idea what it does

Here's the rough implementation:

```ruby
def execute_opt_send_without_block(options, _)
  mid = options[:mid]
  args = options[:orig_argc].times.map { pop }.reverse
  recv = pop
  result = recv.send(mid, *args)
  push(result)
end
```

So here we

1. take a method from the `options` hash (`mid`)
2. then we pop N arguments from the stack (`args`)
3. then we pop the receiver of the method (`recv`)
4. then call `recv.send(mid, *args)` (in our case it's `self.send(:puts, *[123])`
5. and then we push the result back to the stack

## method definition

I intentionally started with method calls because Ruby defines methods via method calls. Yes.

Ruby has a special singleton object called `Frozen Core`. When you define a method via `def m; end` Ruby invokes `frozen_core.send("core#define_method", method_iseq)`:

```ruby
$ ruby --dump=insns -e 'def m; end'
== disasm: #<ISeq:<main>@-e:1 (1,0)-(1,10)> (catch: FALSE)
0000 putspecialobject             1                                   (   1)[Li]
0002 putobject                    :m
0004 putiseq                      m
0006 opt_send_without_block       <callinfo!mid:core#define_method, argc:2, ARGS_SIMPLE>, <callcache>
0009 leave
```

The object itself is defined [here](https://github.com/ruby/ruby/blob/beae6cbf0fd8b6619e5212552de98022d4c4d4d4/vm.c#L2983-L2996).

Of course, we don't have access to the Frozen Core. But we have an instruction that pushes it at the top of the stack. We can create our own `FrozenCore = Object.new.freeze` and check if `recv` is equal to this frozen core.

As you may notice there are also `putobject :m` and `putiseq` instructions. And `argc` of the method call is 2. Hmmm.

`core#define_method` takes two arguments:

1. a method name that is pushed by a `putobject` instruction
2. and an iseq that is ... pushed by the `putiseq` instruction. Yes, instruction is an argument for another instruction.

Here's the code:

```ruby
if recv.equal?(FrozenCore) && mid == :'core#define_method'
  method_name, body_iseq = *args
  result = __define_method(method_name: method_name, body_iseq: body_iseq)
end
```

Here's how `__define_method` looks:

```ruby
def __define_method(method_name:, body_iseq:)
  parent_nesting = current_frame.nesting
  define_on = MethodDefinitionScope.new(current_frame)

  define_on.define_method(method_name) do |*method_args, &block|
    execute(body_iseq, _self: self, method_args: method_args, block: block, parent_nesting: parent_nesting)
  end

  method_name
end
```

When we enter a method in Ruby it inherits `Module.nesting` of a frame that defines it. This is why we also copy `current_frame.nesting` to a method frame.

`define_on = MethodDefinitionScope.new(current_frame)` is also quite simple:

```ruby
class MethodDefinitionScope
  def self.new(frame)
    case frame._self
    when Class, Module
      frame._self
    when TOPLEVEL_BINDING.eval('self')
      Object
    else
      frame._self.singleton_class
    end
  end
end
```

+ If we define a method in a global scope it is defined on the `Object` class.
+ If we define a method inside a class/module context - well, class/module is where the method will be defined.
+ If we define a method inside some other context (inside `instance_eval` for example) - the method is defined on the singleton class of the object.

These code constructions are equivalent:

```ruby
def m; end
# and
Object.define_method(:m) {}

class X; def m; end; end
# and
X.define_method(:m) {}

o = Object.new
o.instance_eval { def m; end }
# and
o.singleton_class.define_method(:m) {}
```

Then comes this part:

```ruby
define_on.define_method(method_name) do |*method_args, &block|
  execute(body_iseq, _self: self, method_args: method_args, block: block, parent_nesting: parent_nesting)
end
```

We define a method that takes any arguments (it breaks `Method#parameters`, but let's ignore it) and an optional block and executes the ISeq of the method body in a context of `self`.

I admit that it's a very hacky trick, but it allows us to dynamically assign `self`.

Plus, we pass all other things that can (and in most cases will) be used in a method body:

1. `method_args` - what was given to a particular invocation of our method
2. `block` - a block given to a method call
3. `parent_nesting` - `Module.nesting` in the outer scope. We have to store in the beginning of the method definition because it may change before the method gets called.

`execute(iseq, **options)` is a tiny wrapper that pushes the frame into the `frame_stack` depending on the `kind` of the given iseq:

```ruby
def execute(iseq, **payload)
  iseq = ISeq.new(iseq)
  push_frame(iseq, **payload)
  evaluate_last_frame
  pop_frame
end

def push_frame(iseq, **payload)
  case iseq.kind
  when :top
    @frame_stack.push_top(
      iseq: iseq
    )
  when :method
    @frame_stack.push_method(
      iseq: iseq,
      parent_nesting: payload[:parent_nesting],
      _self: payload[:_self],
      arg_values: payload[:method_args],
      block: payload[:block]
    )
  else
    raise NotImplementedError, "Unknown iseq kind #{iseq.kind.inspect}"
  end
end
```

## local variables

there are 2 most-commonly used instructions to get/set locals:

+ `getlocal`
+ `setlocal`

Both take two arguments:

+ an offset of the frame where the variable is stored
+ an ID of the variable

Here's an example

```ruby
> pp RubyVM::InstructionSequence.compile('a = 10; b = 20; a; b').to_a[13]
[[:putobject, 10],
 [:setlocal_WC_0, 4],
 [:putobject, 20],
 [:setlocal_WC_0, 3],
 [:getlocal_WC_0, 3],
 [:leave]]
```

We push `10` to the stack, then we pop it and assign to a variable with ID = 4 in the current frame (`setlocal_WC_0` here is a specialized instruction that is `setlocal 0, 4` when the optimization is turned off).

Here's the code to maintain locals:

```ruby
require 'set'

# A simple struct that represents a single local variable;
# has a name, an ID and a value (or no value)
Local = Struct.new(:name, :id, :value, keyword_init: true) do
  def get
    value
  end

  def set(value)
    self.value = value
    value
  end
end

# A wrapper around "Set" that holds all locals for some frame;
# Absolutely each frame has its own instance of "Locals"
class Locals
  UNDEFINED = Object.new
  def UNDEFINED.inspect; 'UNDEFINED'; end

  def initialize(initial_names)
    @set = Set.new

    initial_names.reverse_each.with_index(3) do |arg_name, idx|
      # implicit args (like a virtual attribute that holds mlhs value) have numeric names
      arg_name += 1 if arg_name.is_a?(Integer)
      declare(name: arg_name, id: idx).set(Locals::UNDEFINED)
    end
  end

  def declared?(name: nil, id: nil)
    !find_if_declared(name: name, id: id).nil?
  end

  def declare(name: nil, id: nil)
    local = Local.new(name: name, id: id, value: nil)
    @set << local
    local
  end

  def find_if_declared(name: nil, id: nil)
    if name
      @set.detect { |var| var.name == name }
    elsif id
      @set.detect { |var| var.id == id }
    else
      raise NotImplementedError, "At least one of name:/id: is required"
    end
  end

  def find(name: nil, id: nil)
    result = find_if_declared(name: name, id: id)

    if result.nil?
      raise InternalError, "No local name=#{name.inspect}/id=#{id.inspect}"
    end

    result
  end

  def pretty
    @set
      .map { |local| ["#{local.name}(#{local.id})", local.value] }
      .sort_by { |(name, value)| name }
      .to_h
  end
end
```

So `locals` inside a frame is just a set. It is possible to declare a local, to check if it's declared and to get it.

Here's the implementation of `getlocal`:

```ruby
def execute_getlocal(local_var_id, n)
  frame = n.times.inject(current_frame) { |f| f.parent_frame }
  local = frame.locals.find(id: local_var_id)
  value = local.get
  if value.equal?(Locals::UNDEFINED)
    value = nil
  end
  push(value)
end
```

We jump out `n` times to get the Nth frame, we find a local, we return `nil` if it's `undefined` and we push the value back to the stack (so the result can be used by a subsequent instruction)

Here's the implementation of `setlocal`:

```ruby
def execute_setlocal(local_var_id, n)
  value = pop
  frame = n.times.inject(current_frame) { |f| f.parent_frame }
  local =
    if (existing_local = frame.locals.find_if_declared(id: local_var_id))
      existing_local
    elsif frame.equal?(current_frame)
      frame.locals.declare(id: local_var_id)
    else
      raise InternalError, 'locals are malformed'
    end

  local.set(value)
end
```

This one is a bit more complicated:

1. first, we `pop` the value from the stack (it was pushed by a previous instruction `putobject 10`)
2. then we find Nth frame
3. then we get a local from this frame
4. if it's there we use it; if not - we declare it in the current frame
5. and then we set the value

## method arguments

Every method has a list of arguments. Yes, sometimes it's empty, but even in such case we do an arity check. In general arguments initialization is a part of every method call.

This part of Ruby is really complicated, because we have 12 argument types:

+ required positional argument - `def m(x)`
+ optional positional argument - `def m(x = 42)`
+ rest argument - `def m(*x)`
+ post argument - `def m(*, x = 1)`
+ `mlhs` argument (can be used as a post argument too) - `def m( (x, *y, z) )`
+ required keyword argument - `def m(x:)`
+ optional keyword argument - `def m(x: 42)`
+ rest keyword argument - `def m(**x)`
+ block argument - `def m(&x)`
+ shadow argument - `proc { |;x| }` (I did not implement it because I never used it)
+ `nil` keyword argument (since 2.7) - `def m(**nil)`
+ arguments forwarding (also since 2.7) - `def m(...)`

First, let's take a look at the iseq to see what we have:

```ruby
> pp RubyVM::InstructionSequence.compile('def m(a, b = 42, *c, d); end').to_a[13][4][1]
["YARVInstructionSequence/SimpleDataFormat",
 2,
 6,
 1,
 {:arg_size=>4,
  :local_size=>4,
  :stack_max=>1,
  :node_id=>7,
  :code_location=>[1, 0, 1, 28]},
 "m",
 "<compiled>",
 "<compiled>",
 1,
 :method,
 [:a, :b, :c, :d],
 {:opt=>[:label_0, :label_4],
  :lead_num=>1,
  :post_num=>1,
  :post_start=>3,
  :rest_start=>2},
 [],
 [:label_0,
  1,
  [:putobject, 42],
  [:setlocal_WC_0, 5],
  :label_4,
  [:putnil],
  :RUBY_EVENT_RETURN,
  [:leave]]]
```

There are two entries that we are interested in:

1. a list of argument names (well, it's a list of all variables, but it works for our case)
2. a hash with arguments information

Let's prepare and group it first, it's hard to work with such format:

```ruby
class CategorizedArguments
  attr_reader :req, :opt, :rest, :post, :kw, :kwrest, :block

  def initialize(arg_names, args_info)
    @req = []
    @opt = []
    @rest = nil
    @post = []

    parse!(arg_names.dup, args_info.dup)
  end

  def parse!(arg_names, args_info)
    (args_info[:lead_num] || 0).times do
      req << take_arg(arg_names)
    end

    opt_info = args_info[:opt].dup || []
    opt_info.shift
    opt_info.each do |label|
      opt << [take_arg(arg_names), label]
    end

    if args_info[:rest_start]
      @rest = take_arg(arg_names)
    end

    (args_info[:post_num] || 0).times do
      post << take_arg(arg_names)
    end
  end

  def take_arg(arg_names)
    arg_name_or_idx = arg_names.shift

    if arg_name_or_idx.is_a?(Integer)
      arg_name_or_idx += 1
    end

    arg_name_or_idx
  end
end
```

I intentionally skip keyword arguments here, but they are not that much different from other types of arguments. The only noticeable difference is that optional keyword arguments have "inlined" default values if they are simple enough (like plain strings or numbers, but not expressions like `2+2`). If you are interested you can go to the repository and check this file.

Then, we should parse arguments and assign them into local variables **when we push a method frame** (so they are available once we start executing instructions of a method body):

```ruby
class MethodArguments
  attr_reader :args, :values, :locals

  def initialize(iseq:, values:, locals:)
    @values = values.dup
    @locals = locals
    @iseq = iseq

    @args = CategorizedArguments.new(
      iseq.lvar_names,
      iseq.args_info
    )
  end

  def extract(arity_check: false)
    if arity_check && values.length < args.req.count + args.post.count
      raise ArgumentError, 'wrong number of arguments (too few)'
    end

    # Required positional args
    args.req.each do |name|
      if arity_check && values.empty?
        raise ArgumentError, 'wrong number of arguments (too few)'
      end

      value = values.shift
      locals.find(name: name).set(value)
    end

    # Optional positional args
    args.opt.each do |(name, label)|
      break if values.length <= args.post.count

      value = values.shift
      locals.find(name: name).set(value)

      VM.jump(label)
    end

    # Rest positional argument
    if (name = args.rest)
      value = values.first([values.length - args.post.length, 0].max)
      @values = values.last(args.post.length)

      locals.find(name: name).set(value)
    end

    # Required post positional arguments
    args.post.each do |name|
      if arity_check && values.empty?
        raise ArgumentError, 'Broken arguments, cannot extract required argument'
      end

      value = values.shift
      locals.find(name: name).set(value)
    end

    # Make sure there are no arguments left
    if arity_check && values.any?
      raise ArgumentError, 'wrong number of arguments (too many)'
    end
  end
end
```

Here `values` is what we get in a method call in `*method_args`, `locals` is equal to `MethodFrame#locals` that is set to `Locals.new` by default.

Let's write `MethodFrame` class!

```ruby
MethodFrame = FrameClass.new do
  attr_reader :arg_values

  attr_reader :block

  def initialize(parent_nesting:, _self:, arg_values:, block:)
    self._self = _self
    self.nesting = parent_nesting

    @block = block

    MethodArguments.new(
      iseq: iseq,
      values: arg_values,
      locals: locals,
      block: iseq.args_info[:block_start] ? block : nil
    ).extract(arity_check: true)
  end

  def pretty_name
    "#{_self.class}##{name}"
  end
end
```

Method frame is just a regular frame that extracts arguments during its initialization.

## constants

A regular constant assignment (like `A = 1`) is based on a scope (`Module.nesting` for relative lookup) and two instructions:

1. `setconstant`
2. `getconstant`

Both have a single argument - a constant name. But how does Ruby distinguish relative and absolute constant lookup? I mean, what's the difference between `A` and `::A`?

Ruby uses a special instruction to set a "constant scope" that:

+ for relative lookup
  + in the optimized mode does `opt_getinlinecache` before `get/setconstant`
  + in the non-optimized mode does `pushnil` (that works as a flag)
+ for absolute lookup Ruby computes it via a sequence of instructions

Let's take a look at the non-optimized mode (because we can't optimize it anyway):

```ruby
> pp RubyVM::InstructionSequence.compile('A; ::B; Kernel::D').to_a[13]
[[:putnil],
 [:getconstant, :A],
 [:pop],

 [:putobject, Object],
 [:getconstant, :B],
 [:pop],

 [:putnil],
 [:getconstant, :Kernel],
 [:getconstant, :D],

 [:leave]]
```

`A` constant performs a relative lookup, so `putnil` is used.

`::B` constant performs a global lookup on the `Object` that is a known object, and so it's inlined in the `putobject` instruction.

`Kernel::D` first searches for `Kernel` constant locally, then it uses it as a "scope" for a constant `D`.

Quite easy, right? Not so fast. Ruby uses `Module.nesting` to perform a bottom-top search. This is why it's so important to maintain `nesting` value in frames. Thus, the lookup is performed on `current_scope.nesting` **in reverse order**:

```ruby
def execute_getconstant(name)
  scope = pop

  search_in = scope.nil? ? current_scope.nesting.reverse : [scope]

  search_in.each do |mod|
    if mod.const_defined?(name)
      const = mod.const_get(name)
      push(const)
      return
    end
  end

  raise NameError, "uninitialized constant #{name}"
end
```

If the scope is given (via `push` in a previous instruction) we use it. Otherwise we have a relative lookup and so we must use `current_scope.nesting.reverse`.

`setconstant` is a bit simpler, because it always defines a constant on a scope set by a previous instruction:

```ruby
> pp RubyVM::InstructionSequence.compile('A = 10; ::B = 20; Kernel::D = 30').to_a[13]
[[:putobject, 10],
 [:putspecialobject, 3],
 [:setconstant, :A],

 [:putobject, 20],
 [:putobject, Object],
 [:setconstant, :B],

 [:putobject, 30],
 [:dup],
 [:putnil],
 [:getconstant, :Kernel],
 [:setconstant, :D],

 [:leave]]
```

`putspecialobject` is an instruction that is (when called with 3) pushes a "current" scope.

```ruby
def putspecialobject(kind)
  case kind
  when 3
    push(current_frame.nesting.last) # push "current" scope
  else
    raise NotImplementedError, "Unknown special object #{kind}"
  end
end

def setconstant(name)
  scope = pop
  value = pop

  scope.const_set(name, value)
end
```

## Instance/Class variables

Instance variables are always picked from the `self` of the current frame (they literally look like a simplified version of local variables that are always stored in `self` of the current scope):

```ruby
> pp RubyVM::InstructionSequence.compile('@a = 42; @a').to_a[13]
[[:putobject, 42],
 [:setinstancevariable, :@a, 0],
 [:getinstancevariable, :@a, 0],
 [:leave]]
```

I guess you know how the code should look like:

```ruby
def execute_getinstancevariable(name, _)
  value = current_frame._self.instance_variable_get(name)
  push(value)
end

def execute_setinstancevariable(name, _)
  value = pop
  current_frame._self.instance_variable_set(name, value)
end
```

Class variables are similar, but it is possible to get it in the instance method, so it uses `self` if our current frame is a `ClassFrame` or `self.class` otherwise:

```ruby
> pp RubyVM::InstructionSequence.compile('@@a = 42; @@a').to_a[13]
[[:putobject, 42],
 [:setclassvariable, :@@a],
 [:getclassvariable, :@@a],
 [:leave]]

def execute_setclassvariable(name)
  value = pop
  klass = current_frame._self
  klass = klass.class unless klass.is_a?(Class)
  klass.class_variable_set(name, value)
end

def execute_getclassvariable(name)
  klass = current_frame._self
  klass = klass.class unless klass.is_a?(Class)
  value = klass.class_variable_get(name)
  push(value)
end
```

## Literals

But how can we construct arrays, hashes?

```ruby
> pp RubyVM::InstructionSequence.compile('[ [:foo,a,:bar], [4,5], 42 ]').to_a[13]
[[:putobject, :foo],
 [:putself],
 [:send, {:mid=>:a, :flag=>28, :orig_argc=>0}, false, nil],
 [:putobject, :bar],
 [:newarray, 3],

 [:duparray, [4, 5]],

 [:putobject, 42],
 [:newarray, 3],
 [:leave]]
```

As you can see the strategy of building an array depends on its dynamicity:

+ for dynamic `[:foo, a, :bar]` MRI uses `newarray` (because `a` has to be computed in runtime)
+ for primitive `[4, 5]` it uses `duparray` (because it's faster)

The whole array is also dynamic (because one of its elements is also dynamic). Let's define them:

```ruby
def execute_duparray(array)
  push(array.dup)
end

def execute_newarray(size)
  array = size.times.map { pop }.reverse
  push(array)
end
```

Do hashes support inlining?

```ruby
> pp RubyVM::InstructionSequence.compile('{ primitive: { foo: :bar }, dynamic: { c: d } }').to_a[13]
[[:putobject, :primitive],
 [:duphash, {:foo=>:bar}],

 [:putobject, :dynamic],
 [:putobject, :c],
 [:putself],
 [:send, {:mid=>:d, :flag=>28, :orig_argc=>0}, false, nil],
 [:newhash, 2],

 [:newhash, 4],
 [:leave]]
```

Yes! `duphash` contains an inlined hash that should be pushed to the stack as is. `newhash` has a numeric argument that represents the number of keys and values on the hash (i.e. `keys * 2` or `values * 2`, there's no difference). And once again, if at least one element of the hash is dynamic, the whole has is also dynamic and so it uses `newhash`:

```ruby
def execute_duphash(hash)
  push(hash.dup)
end

def execute_newhash(size)
  hash = size.times.map { pop }.reverse.each_slice(2).to_h
  push(hash)
end
```

Why do we need `.dup` in `duphash` and `duparray`? The reason is simple: this instruction can be executed multiple times (if it's a part of a method or block, for example), and so the same value will be pushed to the stack multiple times. One of the next instructions can modify it but literals have to stay static no matter what. Without using `.dup` the code like

```ruby
2.times do
  p [1, 2, 3].pop
end
```

would print `3` and `2`.

## Splats

Splat is one of the most beautiful features of Ruby. Splat is `foo, bar = *baz` (and also `[*foo, *bar]`):

```ruby
> pp RubyVM::InstructionSequence.compile('a, b = *c, 42').to_a[13]
[[:putself],
 [:send, {:mid=>:c, :flag=>28, :orig_argc=>0}, false, nil],
 [:splatarray, true],
 [:putobject, 42],
 [:newarray, 1],
 [:concatarray],
 [:dup],

 [:expandarray, 2, 0],

 [:setlocal_WC_0, 4],
 [:setlocal_WC_0, 3],
 [:leave]]
```

`splatarray` pops the object from the stack, converts it to `Array` by calling `to_a` (if it's not an array; otherwise there's no type casting), and pushes the result back to the stack.

`concatarray` constructs an array from two top elements and pushes it back. So it changes the stack `[a, b]` to `[ [a,b] ]`. If items are arrays it expands and merges them.

`expandarray` expands it by doing `pop` and pushing items back to the stack. It takes the number of elements that need to be returned, so if an array is bigger it drops some items, if it's too small - it pushes as many `nil`s as needed.

```ruby
def execute_splatarray(_)
  array = pop
  array = array.to_a unless array.is_a?(Array)
  push(array)
end

def execute_concatarray(_)
  last = pop
  first = pop
  push([*first, *last])
end

def execute_expandarray(size, _flag)
  array = pop

  if array.size < size
    array.push(nil) until array.size == size
  elsif array.size > size
    array.pop until array.size == size
  else
    # they are equal
  end

  array.reverse_each { |item| push(item) }
end
```

In fact `expandarray` is much, much more complicated, you can go to the repository and check it if you want.

Keyword splats (like `{ **x, **y }`) are really similar to array splats, I'm not going to cover them here.

## conditions (if/unless)

To handle conditions Ruby uses local `goto` (just like in C). Target of the `goto`-like instruction is a label:

```ruby
> pp RubyVM::InstructionSequence.compile("a = b = c = 42; if a; b; else; c; end").to_a[13]
[[:putobject, 42],
 [:dup],
 [:setlocal_WC_0, 3],
 [:dup],
 [:setlocal_WC_0, 4],
 [:setlocal_WC_0, 5],

 [:getlocal_WC_0, 5],
 [:branchunless, :label_20],
 [:jump, :label_16],

 :label_16,
 [:getlocal_WC_0, 4],
 [:jump, :label_22],

 :label_20,
 [:getlocal_WC_0, 3],

 :label_22,
 [:leave]]
```

Do you see these `:label_<NN>` symbols? They are used as markers. `branchunless` takes a single argument: a label that it to jump to if the value on the top of the stack is `false` or `nil`. If it's `true`-like it does nothing.

```ruby
def execute_branchunless(label)
  cond = pop
  unless cond
    jump(label)
  end
end

def jump(label)
  insns = current_frame.iseq.insns
  insns.drop_while { |insn| insn != label }
  insns.shift # to drop the label too
end
```

Here we do `pop`, check it and call `jump` if it's `false`. `jump` skips instructions until it sees a given label.

MRI also has `branchif` and `branchnil`:

+ `branchif` does `if cond`  as a main check
+ `branchnil` does `if cond.nil?`

## String interpolation/concatenation

Ruby has a few compile-time optimizations that optimize code like

```ruby
"a""b"
"#{'a'}#{'b'}"
```

into a string `"ab"`. However more complicated cases with dynamic interpolation involve a few new instructions:

```ruby
> pp RubyVM::InstructionSequence.compile('"#{a}#{:sym}"').to_a[13]
[[:putobject, ""],

 [:putself],
 [:send, {:mid=>:a, :flag=>28, :orig_argc=>0}, false, nil],
 [:dup],
 [:checktype, 5],
 [:branchif, :label_18],
 [:dup],
 [:send, {:mid=>:to_s, :flag=>20, :orig_argc=>0}, false, nil],
 [:tostring],
 :label_18,

 [:putobject, :sym],
 [:dup],
 [:checktype, 5],
 [:branchif, :label_31],
 [:dup],
 [:send, {:mid=>:to_s, :flag=>20, :orig_argc=>0}, false, nil],
 [:tostring],
 :label_31,

 [:concatstrings, 3],
 [:leave]]
```

Parts above are split into sections.

1. There's an "invisible" beginning of the string (`putobject ""`)
2. Then we have an interpolated method call `a`:
  1. First we call `a` via `send`
  2. then we run `checktype` instruction that checks for an argument `5` that what's popped is a string. it pushes back a boolean value
  3. then we conditionally invoke `to_s` if the object is not a string
3. then we have an interpolated symbol `:sym` that gets interpolated in the same way
4. and finally we invoke `concatstrings 3` that does `pop` 3 times, concatenates 3 strings and pushes the result back to the stack


First let's take a look at the `checktype` instruction:

```ruby
CHECK_TYPE = ->(klass, obj) {
  klass === obj
}.curry

RB_OBJ_TYPES = {
  0x00 => ->(obj) { raise NotImplementedError },     # RUBY_T_NONE

  0x01 => CHECK_TYPE[Object],                        # RUBY_T_OBJECT
  0x02 => ->(obj) { raise NotImplementedError },     # RUBY_T_CLASS
  0x03 => ->(obj) { raise NotImplementedError },     # RUBY_T_MODULE
  0x04 => ->(obj) { raise NotImplementedError },     # RUBY_T_FLOAT
  0x05 => CHECK_TYPE[String],                        # RUBY_T_STRING
  0x06 => ->(obj) { raise NotImplementedError },     # RUBY_T_REGEXP
  0x07 => ->(obj) { raise NotImplementedError },     # RUBY_T_ARRAY
  0x08 => ->(obj) { raise NotImplementedError },     # RUBY_T_HASH
  0x09 => ->(obj) { raise NotImplementedError },     # RUBY_T_STRUCT
  0x0a => ->(obj) { raise NotImplementedError },     # RUBY_T_BIGNUM
  0x0b => ->(obj) { raise NotImplementedError },     # RUBY_T_FILE
  0x0c => ->(obj) { raise NotImplementedError },     # RUBY_T_DATA
  0x0d => ->(obj) { raise NotImplementedError },     # RUBY_T_MATCH
  0x0e => ->(obj) { raise NotImplementedError },     # RUBY_T_COMPLEX
  0x0f => ->(obj) { raise NotImplementedError },     # RUBY_T_RATIONAL

  0x11 => ->(obj) { raise NotImplementedError },     # RUBY_T_NIL
  0x12 => ->(obj) { raise NotImplementedError },     # RUBY_T_TRUE
  0x13 => ->(obj) { raise NotImplementedError },     # RUBY_T_FALSE
  0x14 => ->(obj) { raise NotImplementedError },     # RUBY_T_SYMBOL
  0x15 => ->(obj) { raise NotImplementedError },     # RUBY_T_FIXNUM
  0x16 => ->(obj) { raise NotImplementedError },     # RUBY_T_UNDEF

  0x1a => ->(obj) { raise NotImplementedError },     # RUBY_T_IMEMO
  0x1b => ->(obj) { raise NotImplementedError },     # RUBY_T_NODE
  0x1c => ->(obj) { raise NotImplementedError },     # RUBY_T_ICLASS
  0x1d => ->(obj) { raise NotImplementedError },     # RUBY_T_ZOMBIE
  0x1e => ->(obj) { raise NotImplementedError },     # RUBY_T_MOVED

  0x1f => ->(obj) { raise NotImplementedError },     # RUBY_T_MASK
}.freeze

def execute_checktype(type)
  item_to_check = pop
  check = RB_OBJ_TYPES.fetch(type) { raise InternalError, "checktype - unknown type #{type}" }
  result = check.call(item_to_check)
  push(result)
end
```

I blindly took it from MRI and yes, this instruction supports many types. I implemented only two of them, but the rest looks simple (except `imemo` and friends). Honestly I have no idea why, but about 95% of specs from the RubySpec (only language group, I did not check the whole test suite) are passing with these missing parts. I have no idea how to trigger MRI to use them. Maybe it uses them internally?

`concatstrings` looks just like `newarray`:

```ruby
def execute_concatstrings(count)
  strings = count.times.map { pop }.reverse
  push(strings.join)
end
```

## Blocks

Blocks are passed to method calls as a third argument:

```ruby
> pp RubyVM::InstructionSequence.compile('m { |a| a + 42 }').to_a[13]
[[:putself],
 [:send,
  {:mid=>:m, :flag=>4, :orig_argc=>0},
  false,
  ["YARVInstructionSequence/SimpleDataFormat",
   2,
   6,
   1,
   {:arg_size=>1,
    :local_size=>1,
    :stack_max=>2,
    :node_id=>7,
    :code_location=>[1, 2, 1, 16]},
   "block in <compiled>",
   "<compiled>",
   "<compiled>",
   1,
   :block,
   [:a],
   {:lead_num=>1, :ambiguous_param0=>true},
   [[:redo, nil, :label_1, :label_9, :label_1, 0],
    [:next, nil, :label_1, :label_9, :label_9, 0]],
   [1,
    :RUBY_EVENT_B_CALL,
    [:nop],
    :label_1,
    :RUBY_EVENT_LINE,
    [:getlocal_WC_0, 3],
    [:putobject, 42],
    [:send, {:mid=>:+, :flag=>16, :orig_argc=>1}, false, nil],
    :label_9,
    [:nop],
    :RUBY_EVENT_B_RETURN,
    [:leave]]]]]
```

Block definitely needs a frame that looks pretty much like a `MethodFrame`:

```ruby
BlockFrame = FrameClass.new(:arg_values) do
  def initialize(arg_values:)
    self._self = parent_frame._self
    self.nesting = parent_frame.nesting

    MethodArguments.new(
      iseq: iseq,
      values: arg_values,
      locals: locals,
      block: nil
    ).extract(arity_check: false)
  end

  def pretty_name
    name
  end
end
```

(For simplicity let's ignore that blocks can also take blocks; also let's ignore lambdas, we will return to them later)

The code above looks **almost** like a method frame. The only difference is the `arity_check` value that we pass to the `MethodArguments` class.

But when should we create this frame? And how can we get a proc from it?

```ruby
VM_CALL_ARGS_BLOCKARG   = (0x01 << 1)

def execute_send(options, flag, block_iseq)
  _self = self
  mid = options[:mid]

  args = []

  block =
    if block_iseq
      proc do |*args|
        execute(block_iseq, self: _self, arg_values: args)
      end
    elsif flag & VM_CALL_ARGS_BLOCKARG
      pop
    else
      nil
    end

  args = options[:orig_argc].times.map { pop }.reverse
  recv = pop

  result = recv.send(mid, *args, &block)

  push(result)
end
```

It looks like a more generalized version of `opt_send_without_block`, because `opt_send_without_block` is a specialized implementation of `send`.

This instruction also pops a receiver and arguments, but what's important, it also computes the block.

1. If `block_iseq` is given we create a proc that (once called) executes a block instruction (i.e. a block body) with given arguments. This block uses `self` of the place where it was created. (i.e. `self == proc { self }.call` always returns true)
2. If there's no `block_iseq` the block can be given via a `&block` argument. MRI marks method call as `VM_CALL_ARGS_BLOCKARG` (this flag is just a bitmask)
3. and then we simply call a method with a generated proc object.

Implicit block like `b = proc {}; m(&b)` does not need any additional implementation. Method `proc` here takes a block (handled by the first `if` branch), it gets stored in a local variable and we pass it to the method as a block argument (`elseif` branch).

## Lambdas

It's complicated and I don't have a complete solution that covers all cases (I guess because MRI does not expose enough APIs to do it. Or I'm just not smart enough).

Arrow lambda (`->(){}`) is just a method call `FrozenCore#lambda`, and so we can easily determine that it's a lambda and not a proc. But what about `lambda {}`? It can be overwritten.

An incomplete (and somewhat unreliable) solution is to check that our receiver does not override `lambda` method inherited from a `Kernel` module:

```ruby
creating_a_lambda = false

if mid == :lambda
  if recv.equal?(FrozenCore)
    # ->{} syntax
    creating_a_lambda = true
  end

  if recv.class.instance_method(:lambda).owner == Kernel
    if Kernel.instance_method(:lambda) == RubyRb::REAL_KERNEL_LAMBDA
      # an original "lambda" method from a Kernel module
      creating_a_lambda = true
    end
  end
end
```

Then we can set it on our block frame as an attribute.

```ruby
# in the branch that creates a proc from the `block_iseq`
proc do |*args|
  execute(block_iseq, self: _self, arg_values: args, is_lambda: creating_a_lambda)
end

BlockFrame = FrameClass.new(:arg_values, :is_lambda) do
  def initialize(arg_values:, is_lambda:)
    self.is_lambda = is_lambda
    self._self = parent_frame._self
    self.nesting = parent_frame.nesting

    MethodArguments.new(
      iseq: iseq,
      values: arg_values,
      locals: locals,
      block: nil
    ).extract(arity_check: is_lambda)
  end
end
```

Arity check is enabled only if our proc is a lambda.

## Calling a block

If you remember when we define a method we tell it to save given block in a method frame:

```ruby
define_on.define_method(method_name) do |*method_args, &block|
  execute(body_iseq, _self: self, method_args: method_args, block: block, parent_nesting: parent_nesting)
end
```

And the frame itself saves it in the `attr_reader`.

So both explicit and implicit blocks are available in a method body via `current_frame.block`. It's possible to invoke it by calling `block.call(arguments)` (if it's available as an explicit block argument) or to call `yield(arguments)` (in such case it does not even have to be declared in a method signature).

```ruby
pp RubyVM::InstructionSequence.compile('def m; yield; end').to_a[13][4][1][13]
[[:invokeblock, {:mid=>nil, :flag=>16, :orig_argc=>0}],
 [:leave]]
```

Honestly even before I started working on this article I expected MRI to do something like this. `yield` is equivalent to `<current block>.call(args)`:

```ruby
def execute_invokeblock(options)
  args = options[:orig_argc].times.map { pop }.reverse

  frame = current_frame
  frame = frame.parent_frame until frame.can_yield?

  result = frame.block.call(*args)
  push(result)
end
```

Do you see `frame = frame.parent_frame until frame.can_yield?`? The reason for this line is that you may have a code like

```ruby
def m
  [1,2,3].each { |item| yield item }
end
```

^ `yield` here belongs to the method `m`, not to the `BlockFrame` of the `.each` method. There can be more nested blocks, so we have to go up until we see something that supports `yield`. Well, we know that only one frame can do `yield`: it's a `MethodFrame`.

Our frame class factory need to be extended to generate this method by default and return false from it. `MethodFrame` has to override it and return `true`. Polymorphism!

## Super

Calling `super` is very similar to calling `yield`: it can be replaced with `method(__method__).super_method.call(args)`.

`__method__` can be retrieved from `current_frame.name`, `args` are processed using `options[:orig_argc]`:

```ruby
def execute_invokesuper(options, _, _)
  recv = current_frame._self
  mid = current_frame.name
  super_method = recv.method(mid).super_method
  args = options[:orig_argc].times.map { pop }.reverse

  result = super_method.call(*args)
  push(result)
end
```

This implementation is incorrect, it can't handle a sequence of `super` calls (`class A < B < C`, each has a method that calls `super`). I guess it's possible to implement it by recording the class where the method was defined (i.e. by storing `current_frame._self` before calling `define_method` and passing it to the `MethodFrame` constructor as a `defined_in` attribute). This way we could do something like this:

```ruby
def execute_invokesuper(options, _, _)
  recv = current_frame._self
  mid = current_frame.name

  dispatchers = recv.class.ancestors
  current_dispatcher_idx = dispatchers.index(current_frame.defined_in)
  next_dispatcher = dispatchers[current_dispatcher_idx + 1]

  super_method = next_dispatcher.instance_method(mid).bind(recv)
  args = options[:orig_argc].times.map { pop }.reverse

  result = super_method.call(*args)
  push(result)
end
```

I did not implement it because MSpec does not rely on it and I usually try to avoid sequences of `super` calls.

## Global variables

Similar to locals and instance variables, there are `getglobal`/`setglobal` instructions. They also take a variable name as an argument.

Unfortunately, Ruby has no API to dynamically get/set global variables. But we have `eval`!

```ruby
def execute_getglobal((name))
  push eval(name.to_s)
end

def execute_setglobal((name))
  # there's no way to set a gvar by name/value
  # but eval can reference locals
  value = pop
  eval("#{name} = value")
end
```

## `defined?` keyword

As you may know this keyword can handle pretty much anything:

```ruby
> pp RubyVM::InstructionSequence.compile('defined?(42)').to_a[13]
[:putobject, "expression"],
 [:leave]]

> pp RubyVM::InstructionSequence.compile('a = 42; defined?(a)').to_a[13]
[[:putobject, 42],
 [:setlocal_WC_0, 3],
 [:putobject, "local-variable"],
 [:leave]]
```

In some simple cases it does not do any computations. It's obvious that `42` is an expression and `a` is a local variable (and there's no way to remove it by any code between assignment and `defined?` check)

More advanced checks use a `defined` instruction:

```ruby
> pp RubyVM::InstructionSequence.compile('@a = 42; defined?(@a)').to_a[13]
[[:putobject, 42],
 [:setinstancevariable, :@a, 0],
 [:putnil],
 [:defined, 2, :@a, true],
 [:leave]]
```

The first argument is a special `enum` flag that specifies what are we trying to check here:
```ruby
module DefinedType
  DEFINED_NOT_DEFINED = 0
  DEFINED_NIL = 1
  DEFINED_IVAR = 2
  DEFINED_LVAR = 3
  DEFINED_GVAR = 4
  DEFINED_CVAR = 5
  DEFINED_CONST = 6
  DEFINED_METHOD = 7
  DEFINED_YIELD = 8
  DEFINED_ZSUPER = 9
  DEFINED_SELF = 10
  DEFINED_TRUE = 11
  DEFINED_FALSE = 12
  DEFINED_ASGN = 13
  DEFINED_EXPR = 14
  DEFINED_IVAR2 = 15
  DEFINED_REF = 16
  DEFINED_FUNC = 17
end
```

I'll show you the branch that handles instance variables:

```ruby
def execute_defined(defined_type, obj, needstr)
  # used only in DEFINED_FUNC/DEFINED_METHOD branches
  # but we still have to do `pop` here (even if it's unused)
  context = pop

  verdict =
    case defined_type
    when DefinedType::DEFINED_IVAR
      ivar_name = obj
      if current_frame._self.instance_variable_defined?(ivar_name)
        'instance-variable'
      end
    # ... other branches
    end

  push(verdict)
end
```

All other branches are similar, they do some check and push a constant string or `nil` back to the stack.

## Range literals

For static ranges (like `(1..2)`) Ruby uses inlining and a well-known `putobject` instruction. But what if it's dynamic? Like `(a..b)`

```ruby
> pp RubyVM::InstructionSequence.compile('a = 3; b = 4; p (a..b); p (a...b)').to_a[13]
[[:putobject, 3],
 [:setlocal_WC_0, 4],
 [:putobject, 4],
 [:setlocal_WC_0, 3],
 [:putself],

 [:getlocal_WC_0, 4],
 [:getlocal_WC_0, 3],
 [:newrange, 0],

 [:send, {:mid=>:p, :flag=>20, :orig_argc=>1}, false, nil],
 [:pop],
 [:putself],

 [:getlocal_WC_0, 4],
 [:getlocal_WC_0, 3],
 [:newrange, 1],

 [:send, {:mid=>:p, :flag=>20, :orig_argc=>1}, false, nil],
 [:leave]]
```

There's a special `newrange` instruction that takes a flag as an argument to specify inclusion of the right side (i.e. to distinguish `..` vs `...`)


```ruby
def execute_newrange(flag)
  high = pop
  low = pop
  push(Range.new(low, high, flag == 1))
end
```

## Long jumps

This is probably the most complicated part. What if you have a method that has a loop inside a loop that does `return`? You want to stop executing both loops and simply exit the method, right?

```ruby
def m
  2.times do |i|
    3.times do |j|
      return [i,j]
    end
  end
end

m
```

Of course you can just find the closest frame that supports `return` (i.e. a `MethodFrame`), but you also need to stop execution of two running methods and blocks. In our case it's even more complicated because we don't control them (they are written in C).

The only way I was able to find is to throw an exception. An exception destroys all frames (including YARV's C frames) until it finds someone who can catch and handle it. If there's no such frame the programs exits with an error.

Let's create a special exception class called `VM::LongJumpError`. Each frame class has to know what it can handle (for example, you can do `break` in a block, but not in a method; `return` is normally supported only by methods and lambdas, etc):

```ruby
class LongJumpError  < InternalError
  attr_reader :value

  def initialize(value)
    @value = value
  end

  def do_jump!
    raise InternalError, 'Not implemented'
  end

  def message
    "#{self.class}(#{@value.inspect})"
  end
end

class ReturnError < LongJumpError
  def do_jump!
    frame = current_frame

    if frame.can_return?
      # swallow and consume
      frame.returning = self.value
    else
      pop_frame(reason: "longjmp (return) #{self}")
      raise self
    end
  end
end
```

Each `longjmp` exception wraps the value that it "returns" with (or "breaks" with, for `break` we need a separate class, but I'm going to skip it here. `break`/`next` and other friends are really similar to `return`).

But we need to catch them, right? Without a `rescue` handler we will have something conceptually similar to segfault:

```ruby
def execute(iseq, **payload)
  iseq = ISeq.new(iseq)
  push_frame(iseq, **payload)
  # here comes the difference:
  # we wrap executing instructions into a rescue handler
  begin
    evaluate_last_frame
  rescue LongJumpError => e
    e.do_jump!
  end
  pop_frame
end
```

The only missing thing is the implementation of `can_return?` method in our frames. All frames except `MethodFrame` (and `BlockFrame` it it's marked as `lambda`) must return `false`, `MethodFrame` must return `true`.

MRI uses a special instruction called `throw` that has a single argument that is a `throw_type` (an `enum`, for `return` it's 1, `break` is 3, `next` is 4, there are also `retry`/`redo` and a few more). The value that must be attached to the thrown exception comes from the stack (so this instruction does a single `pop`)

```ruby
VM_THROW_STATE_MASK = 0xff

RUBY_TAG_NONE = 0x0
RUBY_TAG_RETURN = 0x1
RUBY_TAG_BREAK = 0x2
RUBY_TAG_NEXT = 0x3
RUBY_TAG_RETRY = 0x4
RUBY_TAG_REDO = 0x5
RUBY_TAG_RAISE = 0x6
RUBY_TAG_THROW = 0x7
RUBY_TAG_FATAL = 0x8
RUBY_TAG_MASK = 0xf

def execute_throw(throw_type)
  throw_type = throw_type & VM_THROW_STATE_MASK
  throw_obj = pop

  case throw_type
  when RUBY_TAG_RETURN
    raise VM::ReturnError, throw_obj
  when RUBY_TAG_BREAK
    raise VM::BreakError, throw_obj
  when RUBY_TAG_NEXT
    raise VM::NextError, throw_obj
  # ...
  end
end
```

## `longjmp` in MRI

But does it work in the same way in MRI? C does not have exceptions. And at the same time there is a bunch of places where MRI does something like

```c
if (len > ARY_MAX_SIZE) {
    rb_raise(rb_eArgError, "array size too big");
}
// handle validated data
```

This `rb_raise` somehow exits a C function. Well, here's the trick: non-local `goto`.

There are C calls that perform a `goto` to any place (that was previously marked of course, similar to jump to labels for a local `goto`):

+ [`setjmp`](https://linux.die.net/man/3/sigsetjmp)
+ [`longjmp`](https://linux.die.net/man/3/longjmp)

> `setjmp()` saves the stack context/environment in `env` for later use by `longjmp`

... also known as "context switch". And it's relatively expensive.

Even if you don't `raise` an exception and only do `begin; ...; rescue; end` in your code you still have to save the context (to jump to it once you `raise` an error). MRI does not know at compile time which methods can throw an error (and do you throw them at all), so each `rescue` produces a `setjmp` call (and each `raise` triggers a `longjmp` and passes `closest rescue` -> `saved env` as an argument)

## `rescue`/`ensure`

So now we know that raise/rescue works via long jumps under the hood. Let's implement our own exceptions.

By sticking to MRI exceptions we can unwrap both internal and our stacks at the same time. I'm not going to override `raise`, it should do what it originally does, but we still need to support our own `rescue` blocks. Let's see what MRI gives us:

```ruby
> pp RubyVM::InstructionSequence.compile('begin; p "x"; rescue A; p "y"; end').to_a
[ # ...snip
 [[:rescue,
   [ # ...snip
    "rescue in <compiled>",
    "<compiled>",
    "<compiled>",
    1,
    :rescue,
    [:"\#$!"],
    {},
    [],
    [1,
     [:getlocal_WC_0, 3],
     [:putnil],
     [:getconstant, :A],
     [:checkmatch, 3],
     [:branchif, :label_11],
     [:jump, :label_19],
     :label_11,
     :RUBY_EVENT_LINE,
     [:putself],
     [:putstring, "y"],
     [:send, {:mid=>:p, :flag=>20, :orig_argc=>1}, false, nil],
     [:leave],
     :label_19,
     [:getlocal_WC_0, 3],
     [:throw, 0]]],
   :label_0,
   :label_7,
   :label_8,
   0],
  [:retry, nil, :label_7, :label_8, :label_0, 0]],
 [:label_0,
  1,
  :RUBY_EVENT_LINE,
  [:putself],
  [:putstring, "x"],
  [:send, {:mid=>:p, :flag=>20, :orig_argc=>1}, false, nil],
  :label_7,
  [:nop],
  :label_8,
  [:leave]]]
```

An instruction sequence that has some `rescue` blocks inside includes all information about them (in the element #12, right above an instructions list). Each rescue handler is a frame with its own list of variables and instructions. Its `kind` is `:rescue` and is has at least one local variable: `$!`. It starts with a dollar sign, but it's a local variable. According to its semantics it has to be a local variable, but unfortunately it can't look like a local variable (because it'd would potentially conflict with method calls). I mean, that's how I explain it to myself, I don't know for sure what was the initial reason to design it this way.

It also has a few labels at the bottom - `:label_7, :label_8, :label_0`:

+ the first label is a "begin" label. It marks where the (potentially) critical section of your code begins
+ the second label is an "end" label
+ the third label is an "exit" label. It marks where we should jump to if the error has been caught and handled.

A top-level instruction also contains these labels, and the meaning of them is:

+ if we evaluate instructions and we see a label that is a "begin" label of some rescue handler we **enable** the handler
+ if we see a label that is an "end" of some rescue handler we **disable** it
+ if we execute a single instruction and catch an exception we:
  + iterate over all **enabled** rescue handlers
  + and for each of them we push a `RescueFrame` with a rescue iseq
  + and we set a `$!` variable in this frame to the error that we have just caught
  + the rescue frame decides where to go next:
     + back to the original frame via `jump` to the "exit" label after doing `pop_frame`
     + or somewhere else via re-raise (if the iseq contains it)

Let's code it!

```ruby
class ISeq
  attr_reader :rescue_handlers
  attr_reader :ensure_handlers

  def initialize(ruby_iseq)
    @ruby_iseq = ruby_iseq
    reset!
    setup_rescue_handlers!
    setup_ensure_handlers!
  end

  # ... other existing methods on ISeq class

  def setup_rescue_handlers!
    @rescue_handler = @ruby_iseq[12]
      .select { |handler| handler[0] == :rescue }
      .map { |(_, iseq, begin_label, end_label, exit_label)| Handler.new(iseq, begin_label, end_label, exit_label) }
  end

  def setup_ensure_handlers!
    @ensure_handler = @ruby_iseq[12]
      .select { |handler| handler[0] == :ensure }
      .map { |(_, iseq, begin_label, end_label, exit_label)| Handler.new(iseq, begin_label, end_label, exit_label) }
  end

  class Handler
    attr_reader :iseq
    attr_reader :begin_label, :end_label, :exit_label
    def initialize(iseq, begin_label, end_label, exit_label)
      @iseq = iseq
      @begin_label, @end_label, @exit_label = begin_label, end_label, exit_label
    end
  end
end
```

So now each `iseq` object has two getters:

+ `rescue_handlers`
+ `ensure_handlers`

Frames must know which handlers are active (but not instruction sequences, because methods can recursively call themselves and so the same iseq will be reused; it's a per-frame property):

```ruby
class FrameClass
  def self.new
    Struct.new {
      # Both must be set to `Set.new` in the constructor
      attr_accessor :enabled_rescue_handlers
      attr_accessor :enabled_ensure_handlers
    }
  end
end
```

So this way all frames have it too. Every time when we see a label in our execution loop we need to check if it matches any `begin_label` or `end_label` of our `current_frame.iseq.rescue_handlers` (or `ensure_handlers`):

```ruby
def on_label(label)
  {
    current_iseq.rescue_handlers => current_frame.enabled_rescue_handlers,
    current_iseq.ensure_handlers => current_frame.enabled_ensure_handlers,
  }.each do |all_handlers, enabled_handlers|
    all_handlers
      .select { |handler| handler.begin_label == label }
      .each { |handler| enabled_handlers << handler }

    all_handlers
      .select { |handler| handler.end_label == label }
      .each { |handler| enabled_handlers.delete(handler) }
  end
end
```

side note: when we do a local `jump` we should also walk through skipped instructions and enable/disable our handlers; this is important

OK, now the only missing part is the reworked execution loop:

```ruby
# a generic runner that is used inside the loop
def execute_insn(insn)
  case insn
  when [:leave]
    current_frame.returning = stack.pop
  when Array
    name, *payload = insn

    with_error_handling do
      send(:"execute_#{name}", payload)
    end
  # -- new branch for labels --
  when Regexp.new("label_\\d+")
    on_label(insn)
  else
    # ignore
  end
end

# a wrapper that catches and handles error
def with_error_handling
  yield
rescue VM::InternalError => e
  # internal errors like LongJumpError should be invisible for users
  raise
rescue Exception => e
  handle_error(e)
end

def handle_error(error)
  if (rescue_handler = current_frame.enabled_rescue_handler.first)
    result = execute(rescue_handler.iseq, caught: error, exit_to: rescue_handler.exit_label)
    stack.push(result)
  else
    raise error
  end
end

# This guy also needs customization to support `jump(exit_label)`
def pop_frame
  frame = @frame_stack.pop
  if frame.is_a?(RescueFrame)
    jump(frame.exit_to)
  end
  frame.returning
end

# And here's the rescue frame implementation
RescueFrame = FrameClass.new do
  attr_reader :parent_frame, :caught, :exit_to

  def initialize(parent_frame:, caught:, exit_to:)
    self._self = parent_frame._self
    self.nesting = parent_frame.nesting

    @parent_frame = parent_frame
    @caught = caught
    @exit_to = exit_to

    # $! always has an ID = 3
    locals.declare(id: 3, name: :"\#$!")
    locals.find(id: 3).set(caught)
  end
end
```

But why do we handle only the first handler? Can there be multiple handlers? The answer is no, because:

+ MRI merges multiple consecutive `rescue` handlers (by using `case error` branching in a rescue body)
+ `rescue` itself is frame, and so nested `rescue` is a rescue handler of the rescue handler

## `throw`/`catch` methods

As a side (and I personally think a very interesting) note, while I was working on this project I have realized that specs for `Kernel#throw` are not working for me at all. They were literally completely broken (even after I finished working on a very basic implementation of exceptions):

```ruby
catch(:x) do
  begin
    throw :x
  rescue Exception => e
    puts e
  end
end
```

This code does not print anything. However, if you do just `throw :x` you get an exception:

```ruby
> throw :x
Traceback (most recent call last):
        2: from (irb):14
        1: from (irb):14:in `throw'
UncaughtThrowError (uncaught throw :x)
```

Huh, what's going on? Let's take a look at the implementation of `Kernel#throw`:

```c
void
rb_throw_obj(VALUE tag, VALUE value)
{
    rb_execution_context_t *ec = GET_EC();
    struct rb_vm_tag *tt = ec->tag;

    while (tt) {
      if (tt->tag == tag) {
          tt->retval = value;
          break;
      }
      tt = tt->prev;
    }
    if (!tt) {
      VALUE desc[3];
      desc[0] = tag;
      desc[1] = value;
      desc[2] = rb_str_new_cstr("uncaught throw %p");
      rb_exc_raise(rb_class_new_instance(numberof(desc), desc, rb_eUncaughtThrow));
    }

    ec->errinfo = (VALUE)THROW_DATA_NEW(tag, NULL, TAG_THROW);
    EC_JUMP_TAG(ec, TAG_THROW);
}
```

This code raises an instance of `rb_eUncaughtThrow` only in one case: if there's no frame above that may "catch" it (like in the case when we did just `throw :x`).

However if there's a frame somewhere above that has the same tag MRI performs a manual `longjmp`. This is why we can't catch this exception. There's simply no exception if there's a `catch(:x)` above (but there would be an exception if we would do `catch(:y) { throw :x }`).

Is it faster? Let's see

```ruby
require 'benchmark/ips'

Benchmark.ips do |x|
  x.config(:time => 3)

  x.report('raise') do
    begin
      raise 'x'
    rescue => e
    end
  end

  x.report('throw') do
    catch(:x) do
      throw :x
    end
  end

  x.compare!
end

$ ruby benchmark.rb
Warming up --------------------------------------
               raise   101.832k i/100ms
               throw   256.893k i/100ms
Calculating -------------------------------------
               raise      1.310M (± 4.2%) i/s -      3.971M in   3.037942s
               throw      4.853M (± 3.0%) i/s -     14.643M in   3.020227s

Comparison:
               throw:  4852821.4 i/s
               raise:  1309915.6 i/s - 3.70x  slower
```

As expected, it's faster. But what's more important, let's see what if there are more frames between `raise/rescue` and `throw/catch`. First, let's create a small method that wraps given code into N frames:

```ruby
def in_n_frames(depth = 0, n, blk)
  if depth == n
    blk.call
  else
    in_n_frames(depth + 1, n, blk)
  end
end

begin
  in_n_frames(1000, proc { raise 'err' })
rescue => e
  p e.backtrace.length
end
```

It prints `1003`, because there's also a `TopFrame` and a `BlockFrame` (and a small bug that does +1), but that's absolutely fine for us.

```ruby
$ ruby benchmark.rb
Warming up --------------------------------------
               raise     1.061k i/100ms
               throw     1.115k i/100ms
Calculating -------------------------------------
               raise     10.628k (± 1.3%) i/s -     32.891k in   3.095347s
               throw     11.183k (± 1.5%) i/s -     34.565k in   3.091514s

Comparison:
               throw:    11183.1 i/s
               raise:    10627.8 i/s - 1.05x  slower
```

There's almost no difference! The reason is simple: the only thing that is different is creation of the exception object. `throw` does not do it.

## Utility instructions

There are also a few interesting instructions that MRI uses to evaluate your complex code:

+ `adjuststack(n)` - does `n.times { pop }`
+ `nop` - literally does nothing
+ `dupn(n)` - does `pop` N times and then pushes them twice (or basically duplicates N last items)
+ `setn(n)` - does `stack[-n-1] = stack.top`
+ `topn(n)` - does `push(stack[-n-1])`
+ `swap` - swaps top two stack elements
+ `dup` - like `dupn(1)`
+ `reverse(n)` - reverses N stack elements (i.e. does `n.times.map { pop }.each { |value| push(value) }`)

# Final words

First of all, I'd like to say thank you to everyone who made YARV. I was not able to find a single place where MRI behaves inefficiently (and I spent many hours looking into instructions).

Once again, the code is available [here](https://github.com/iliabylich/my.rb), feel free to create issues/message me in Twitter. And please, don't use it in the real world.
