---
layout: post
title:  "Ruby Marshalling from A to Z"
date:   2016-01-26 00:00:00 +0300
categories: ruby marshalling serialization tlv
---
Marshalling is a serialization process when you convert an object to a binary string.
Ruby has a standard class `Marshal` that does all the job for serialization and deserialization.
To serialize an object, use `Marshal.dump`, to deserialize - `Marshal.load` or `Marshal.restore`.

``` ruby
marshalled = Marshal.dump([1, 2, 'string', Object.new])
# => "\x04\b[\ti\x06i\aI\"\vstring\x06:\x06ETo:\vObject\x00"

Marshal.load(marshalled)
# => [1, 2, "string", #<Object:0x00000002643000>]
```

This article explains the format of marshalling and shows how to write a pure Ruby marshalling library
compatible with the standard Ruby implementation.

# The gem

Let's try to make a pure Ruby gem that is compatible with standard `Marshal` class.

``` bash
$ bundle gem pure_ruby_marshal
$ tree pure_ruby_marshal
pure_ruby_marshal
├── bin
│ ├── console
│ └── setup
├── CODE_OF_CONDUCT.md
├── Gemfile
├── lib
│ ├── pure_ruby_marshal
│ │ └── version.rb
│ └── pure_ruby_marshal.rb
├── LICENSE.txt
├── pure_ruby_marshal.gemspec
├── Rakefile
└── README.md
```

Our main module `PureRubyMarshal` has the following interface:

``` ruby
module PureRubyMarshal
  extend self

  def dump(object)
    # ...
  end

  def load(data)
    # ...
  end
end
```

Also I'd like to extract all logic related into reading/writing encoded data
to separate classes - `ReadBuffer` for reading, `WriteBuffer` for writing.

``` ruby
module PureRubyMarshal
  extend self

  autoload :ReadBuffer,  'pure_ruby_marshal/read_buffer'
  autoload :WriteBuffer, 'pure_ruby_marshal/write_buffer'

  def dump(object)
    WriteBuffer.new(object).write
  end

  def load(data)
    ReadBuffer.new(data).read
  end
end

```

# Reading

As we decided before, `ReadBuffer` is our class responsible for reading an object from marshalled data.
Here is how it should look:

``` ruby
class PureRubyMarshal::ReadBuffer
  attr_reader :data

  def initialize(data)
    @data = data
  end

  def read
    # ...
  end
end

```

## Decompressing

First let's take a look at `Marshal.dump`.

``` ruby
marshalled = Marshal.dump([1, 2, 'string', Object.new])
data = marshalled.chars
# => [a long array of characters]
```

The first two characters represent a version of library that was used for marshalling:

``` ruby
data[0].ord
# => 4
data[1].ord
# => 8
```

Which represents a current version of Marshal library - `4.8`
([link to the source](https://github.com/ruby/ruby/blob/trunk/marshal.c#L54-L55)).
This values are constants for any marshalled data and they are stored in `Marshal::MAJOR_VERSION` and `Marshal::MINOR_VERSION`:

``` ruby
 :001 > Marshal::MAJOR_VERSION
 => 4
 :002 > Marshal::MINOR_VERSION
 => 8
```

Our `PureRubyMarshal` should have same constants:
``` ruby
module PureRubyMarshal
  MAJOR_VERSION = 4
  MINOR_VERSION = 8
end
```

The rest of the array is actually a marshalled object.
So, for now marshalled data looks like `['4.8', some unknown characters]`

To read the version of `Marshal` library we can modify `ReadBuffer#initialize`:

``` ruby
attr_reader :minor_version, :major_version

def initialize(data)
  @data = data.chars
  @major_version = read_byte
  @minor_version = read_byte
end

def read_char
  data.shift
end

def read_byte
  read_char.ord
end

# In irb:
marshalled = Marshal.dump(123)
read_buffer = PureRubyMarshal::ReadBuffer.new(marshalled)
read_buffer.major_version
 => 4
read_buffer.minor_version
 => 8
```

## Getting objects from the raw data

When `Marshal` converts your object to a string, it uses very simple rules:
+ All primitives like `Fixnum`, `Hash` or `Array` always use the same format of serialization,
prepended by a special character (each character for each unique type)
+ If an object has instance variables, these ivars are always dumped in the same way (`Iobject{hash:of,instance:variables}`)
+ You can't dump multiple objects in a single operation,
so there's always a root object in marshalled string (which is actually placed in the beginning of the string)
+ This object is followed by all other data, like ivars, string encodings etc.

## NilClass, TrueClass, FalseClass

``` ruby
ReadBuffer.new(Marshal.dump(nil)).data
 => ["0"]
```

Character `0` means that encoded object is `nil`. To handle it, we can write something like:
``` ruby
class PureRubyMarshal::ReadBuffer
  # ...

  def read
    char = read_char
    case char
    when '0' then nil
    else
      raise NotImplementedError, "Unknown object type #{char}"
    end
  end
end

 :001 > ReadBuffer.new(Marshal.dump(nil)).read
 => nil
```

``` ruby
 :001 > ReadBuffer.new(Marshal.dump(true)).data
 => ["T"]
 :002 > ReadBuffer.new(Marshal.dump(false)).data
 => ["F"]
```
`true` converts to `T`, `false` converts to `F`, nothing complex for now.

Let's extend our `case` statement:
``` ruby
when 'T' then true
when 'F' then false
```

## Tests

Of course, we can't develop without tests,

``` ruby
# fixtures.rb

FIXTURES = {
  'nil' => nil,
  'true' => true,
  'false' => false
}

# pure_ruby_marshal_spec.rb

require 'spec_helper'

describe PureRubyMarshal do
  describe '.load' do
    FIXTURES.each do |fixture_name, fixture_value|
      it "loads marshalled #{fixture_name}" do
        result = PureRubyMarshal.load(Marshal.dump(fixture_value))
        expect(result).to eq(fixture_value)
      end
    end
  end
end

# rspec --format documentation

PureRubyMarshal
  .load
    loads marshalled nil
    loads marshalled true
    loads marshalled false

3 examples, 0 failures
```

## Integer

All encoded integers are prepended with an `"i"` symbol. Added one more `when` to our `case`:

``` ruby
when 'i' then read_integer
```

The next byte defines a strategy of decoding the rest of the data. Then comes a bytes sequence representing a number.

Let's say that `byte = (first_char_code_after_i ^ 128) - 128` (it ranges from `-128 .. 128`).

There are 5 different cases depending on the byte value:
+ `byte is 0`
+ `byte in [4..128]`
+ `byte in [1..4]`
+ `byte in [-128..-4]`
+ `byte in [-5..-1]`

Let's write a utility lambda to simplify examples:

``` ruby
get_sequence = lambda do |n|
  buffer = ReadBuffer.new(Marshal.dump(n))
  buffer.read_char
  buffer.data.map(&:ord)
end
```

It:
+ marshalls passed number with native `Marshal`
+ loads it through our pure ruby `ReadBuffer`
+ reads one char (`"i"`)
+ takes the rest of the data
+ and converts characters to their codes

**The first case** is the simplest one:

``` ruby
get_sequence[0]
 => [0]
```
Zero is actually encoded as zero.

**The second case**:

``` ruby
get_sequence[7]
 => [12]
get_sequence[8]
 => [13]
get_sequence[120]
 => [125]
```
In this case `result = byte - 5` and it covers small positive numbers.

**The third case**:

``` ruby
get_sequence[99999]
 => [3, 159, 134, 1]
```

3 means three numbers representing a single number, they should be merged with binary `OR` and byte shifting:
``` ruby
(159 << (8*0)) | (134 << (8*1)) | (1 << (8*2))
 => 99999
```

**The fourth and the fifth examples** are just like 2nd and 3rd, but for negative numbers.

Let's implement our `read_integer` method!

``` ruby
def read_integer
    # c is our first byte
    c = (read_byte ^ 128) - 128

    case c
    when 0
      # 0 means 0
      0
    when (4..127)
      # case for small positive numbers
      c - 5
    when (1..3)
      # c next bytes is our big positive number
      c.
        times.
        map { |i| [i, read_byte] }.
        inject(0) { |result, (i, byte)| result | (byte << (8*i))  }
    when (-128..-6)
      # case for small negative numbers
      c + 5
    when (-5..-1)
      # (-c) next bytes is our number
      (-c).
        times.
        map { |i| [i, read_byte] }.
        inject(-1) do |result, (i, byte)|
          a = ~(0xff << (8*i))
          b = byte << (8*i)
          (result & a) | b
        end
    end
  end
```

To add tests for integers, we can just add more key-value pairs to our
`FIXTURES` constant.

Complex data types like
+ `Array`
+ `String`
+ `Hash`
+ `Regexp`

and others include numbers into their encoded structure to represent their length.

## Symbol

``` ruby
ReadBuffer.new(Marshal.dump(:a_symbol)).data
 => [":", "\r", "a", "_", "s", "y", "m", "b", "o", "l"]
```

Symbols are encoded using:
+ a special `":"` character
+ Symbol length (`"\r".ord - 5 = 8`) as Integer (we can fetch it using `read_integer` method)
+ The Symbol itself, character by character

``` ruby
# one more "when" statement
when ':' then read_symbol

# and implementation
def read_symbol
  read_integer.times.map { read_char }.join.to_sym
end
```

## String

``` ruby
ReadBuffer.new(Marshal.dump("a string")).data
 => ["\"", "\r", "a", " ", "s", "t", "r", "i", "n", "g"]
```

Where:
+ `"\""` is actually a `"` symbol which shows the beginning of the encoded string
+ the next encoded number `n = 8` is a length of the string
+ `n` following symbols are the string itself.

To support loading marshalled string, we can use the following code:
``` ruby
# ...
when '"' then read_string
# ...
def read_string
  read_integer.times.map { read_char }.join
end
```

## Array

``` ruby
ReadBuffer.new(Marshal.dump([1,2,3])).data
 => ["[", "\b", "i", "\x06", "i", "\a", "i", "\b"]
```

+ `"["` means that converted object is an `Array`
+ the next encoded number `n = 3` is a length of our array
+ `"i", "1"` is an `Integer 1`
+ `"i", "2"` is an `Integer 2`
+ `"i", "3"` is an `Integer 3`

Array items are encoded as separate objects, every item is prepended by its own service character:

``` ruby
# ...
when '[' then read_array
# ...
def read_array
  read_integer.times.map { read }
end
```

## Hash

`Hash` is encoded as an array of key-value pairs:

``` ruby
ReadBuffer.new(Marshal.dump({ 15 => 5 })).data
 => ["{", "\x06", "i", "\x14", "i", "\n"]
```

+ `"{"` is the beginning of hash
+ `1` is a number of `key => value` pairs
+ `"i" 15` is an `Integer 15`
+ `"i" 5` is an `Integer 5`


``` ruby
# ...
when '{' then read_hash
# ...
def read_hash
  pairs = read_integer.times.map { [read, read] }
  Hash[pairs]
end
```

## Float

`Float` is encoded as its string representation

``` ruby
ReadBuffer.new(Marshal.dump(1.5)).data
 => ["f", "\b", "1", ".", "5"]
```

Floats are encoded using `#to_s` method:
+ `"f"` is the beginning of `Float`
+ `3` is the length
+ `"1", ".", "5"` is its string representation

To get it back, we can use `String#to_f` method:

``` ruby
# ...
when 'f' then read_float
# ...
def read_float
  read_string.to_f
end
```

## Class

``` ruby
ReadBuffer.new(Marshal.dump(Array)).data
 => ["c", "\n", "A", "r", "r", "a", "y"]
```

Classes are represented by their names:
+ `"c"` means a `Class`
+ `5` is the length of its name
+ next 5 characters are the name itself

After reading the name we can do `Object.const_get(const_name)` to retrieve the actual class.
The only remark here is that when the class doesn't exist anymore,
`Marshal` re-raises `ArgumentError, "undefined class/module #{const_name}"` instead of a `NameError`.
Moreover, if the constant returned by `Object.const_get` is not a class,
`Marshal` raises `ArgumentError, "#{const_name} does not refer to a Class"`.
So, the constant **must** exist and it **must** be a `Class`

``` ruby
# ...
when 'c' then read_class
# ...
def marshal_const_get(const_name)
  Object.const_get(const_name)
rescue NameError
  raise ArgumentError, "undefined class/module #{const_name}"
end

def read_class
  const_name = read_string
  klass = marshal_const_get(const_name)
  unless klass.instance_of?(Class)
    raise ArgumentError, "#{const_name} does not refer to a Class"
  end
  klass
end
```

I've extracted `marshal_const_get` to a separate method to use it later for
reading a `Module` from the marshalled data.

## Module

Modules are similar to Classes, but the "magical" character is `"m"` instead of `"c"`
(and, of course, messages of exceptions are about modules).

``` ruby
# ...
when 'm' then read_module
# ...
def read_module
  const_name = read_string
  klass = marshal_const_get(const_name)
  unless klass.instance_of?(Module)
    raise ArgumentError, "#{const_name} does not refer to a Module"
  end
  klass
end
```

## Struct

Struct are encoded by their class names + their data:

``` ruby
Point = Struct.new(:x, :y)
a = Point.new(3, 7)
ReadBuffer.new(Marshal.dump(a)).data
 => ["S", ":", "\n", "P", "o", "i", "n", "t", "\a", ":", "\x06", "x", "i", "\b", ":", "\x06", "y", "i", "\f"]
```

This output can be split to:
+ `"S"` means a beginning of encoded `Struct`
+ `":", 5, "P", "o", "i", "n", "t"` is a Symbol `:Point`
+ `2, ":", 1, "x", "i", 2, ":", 1, "y", "i", 3` is a visually unmarked Hash (i.e. no `{` symbol) with 2 pairs:
  + `":", 1, "x"` is a Symbol `:x`
  + `"i", 2` is an Integer `2`
  + `":", 1, "y"` is a Symbol `:y`
  + `"i", 3` is an Integer `3`

So, we have a Struct defined with its class name and the Hash containing object's data

``` ruby
# ...
when 'S' then read_struct
# ...
def read_struct
  klass = marshal_const_get(read) # Point
  attributes = read_hash # { x: 3, y: 7 }
  values = attributes.values_at(*klass.members) # [3, 7]
  klass.new(*values)
end
```

Why is class name encoded as a `Symbol`, not a `String`? See section 'Symbol link'

## Regexp

``` ruby
ReadBuffer.new(Marshal.dump(/a_regexp/)).data
 => ["/", "\r", "a", "_", "r", "e", "g", "e", "x", "p", "\x00"]
```

Alright, there are:
+ `"/"` - a beginning of a regexp
+ `8` - length of its string representation
+ `"a_regexp"` - string representation
+ 0 - a [kcode](http://ruby-doc.org/core-2.1.1/Regexp.html#method-c-new) that was passed to a constructor

``` ruby
# ...
when '/' then read_regexp
# ...
def read_regexp
  string = read_string
  kcode = read_byte
  Regexp.new(string, kcode)
end
```

## Abstract object

``` ruby
class Point2
  attr_reader :x, :y

  def initialize(x, y)
    @x, @y = x, y
  end

  def ==(other)
    other.is_a?(Point2) &&
      other.x == self.x &&
      other.y == self.y
  end
end

point = Point2.new(5, 10)
ReadBuffer.new(Marshal.dump(point)).data
 => ["o", ":", "\v", "P", "o", "i", "n", "t", "2", "\a", ":", "\a", "@", "x", "i", "\n", ":", "\a", "@", "y", "i", "\x0F"]
```

+ `"o"` means the beginning of any non-standard object
+ `":", 6, "P", "o", "i", "n", "t", "2"` is a Symbol `:Point2`
+ `2, ":", 2, "@", "x", "i", 5, ":", 2, "@", "y", "i", 10` is a Hash with 2 key-value pairs:
  + key `":", 2, "@", "x"` is a Symbol `@x`
  + value `"i", 5` is an Integer `4`
  + key `":", 2, "@", "y"` is a Symbol `@y`
  + value `"i", 10` is an Integer `5`

``` ruby
# ...
when 'o' then read_object
# ...
def read_object
  klass = marshal_const_get(read) # Point2
  ivars_data = read_hash # { :@x => 5, :@y = 10 }
  object = klass.allocate # #<Point >
  ivars_data.each do |ivar_name, value|
    object.instance_variable_set(ivar_name, value)
  end
  object
end
```

## User class

User classes are classes inherited from default classes:

``` ruby
class MyArray < Array
end
my_array = MyArray[1,2,3]

ReadBuffer.new(Marshal.dump(my_array)).data
 => ["C", ":", "\f", "M", "y", "A", "r", "r", "a", "y", "[", "\b", "i", "\x06", "i", "\a", "i", "\b"]
```

Objects of these classes are encoded in the following way:
+ `"C"` - a beginning of a user class
+ `":", 7, "M", "y", "A", "r", "r", "a", "y"` - a symbol `:MyArray` - the name of the user class
+ the rest of the data that should be passed to constructor (array `[1, 2, 3]` for this example)

``` ruby
# ...
when 'C' then read_userclass
# ...
def read_userclass
  klass = marshal_const_get(read)
  data = read
  klass.new(data)
end
```

## Extended object

Sometimes your object is very, very complex. Like an object extended with some modules:

``` ruby
MyModule = Module.new
obj = []
obj.extend(MyModule)
```

In this case `Marshal` prepends you object structure with the list of extended modules:

``` ruby
ReadBuffer.new(Marshal.dump(obj)).data
 => ["e", ":", "\r", "M", "y", "M", "o", "d", "u", "l", "e", "[", "\x00"]
```

Where:
+ `"e"` means that marshalled data has the following scheme:
  + a `Symbol` that represents a name of a Ruby module (`:MyModule`)
  + an abstract dumped object that was extended with this module and should be retrieved

``` ruby
# ...
when 'e' then read_extended_object
# ...
def read_extended_object
  mod = marshal_const_get(read) # MyModule
  object = read # []
  object.extend(mod)
end
```

## Symbol links

I guess this idea was originally created for compressing marshalled output.
Consider the following situation: you have a collection of objects of the same class
(the simplest example is a result of calling `YourActiveRecordModel.all`).
When you pass this collection to `Marshal.dump`, it converts objects one by one, writing them to an output stream.
But an abstract object is represented by its class name and a hash of instance variables.
Symbol links save you from writing the same `class.name` again and again to the output.
Instead, it remembers all symbols that have been written, and when the symbol appears twice in the sequence,
it writes its sequence number.

``` ruby
a1 = [:symbol1, :symbol2]
a2 = [:symbol1, :symbol1]

dumped1 = Marshal.dump(a1)
 => "\x04\b[\a:\fsymbol1:\fsymbol2"
dumped1.length
 => 22

dumped2 = Marshal.dump(a2)
 => "\x04\b[\a:\fsymbol1;\x00"
dumped2.length
 => 15
```

For this example it saves us 31% of the initial size. Let's see how it works:

``` ruby
ReadBuffer.new(Marshal.dump(a2)).data
 => ["[", "\a", ":", "\f", "s", "y", "m", "b", "o", "l", "1", ";", "\x00"]
```

So we have an array of two items:
+ a `Symbol` `:symbol1`
+ a special character `";"` representing beginning of symbol link and an encoded `Integer` that represents a symbol with this index

The algorithm for parsing `Symbol` should be changed a little bit
to save all symbols to an internal array.

``` ruby
def initialize
  # ...
  @symbols_cache = []
  # ...
end

# ...

def read_symbol
  symbol = read_integer.times.map { read_char }.join.to_sym # no changes here
  @symbols_cache << symbol # save a symbol
  symbol
end

# ...

when ';' then read_symbol_link

# ...

def read_symbol_link
  @symbols_cache[read_integer]
end
```

I'll return to symbol links and my thoughts about how it can be used to compress marshalled output
in "Optimizations" section.

## Object links

Same story here, when you have an object that appears multiple times in your data, `Marshal` will serialize it only once:

``` ruby
obj1 = Object.new
obj2 = Object.new

a1 = [obj1, obj2]
a2 = [obj1, obj1]

dumped1 = Marshal.dump(a1)
dumped1.length
 => 18

dumped2 = Marshal.dump(a2)
dumped2.length
 => 16
```

A symbol that indicates a beginning of an object link is `"@"`. However the criteria for
dumping an object link instead of an object itself is objects equality (`equal?`).
Here's the problem: if you dump an array `[{}, {}]`, `Marshal` will dump
both objects without any object links, because these objects are not equal.

Also `Marshal` doesn't cache:
  + `true`/`false`/`nil`
  + `Integer`
  + `String` when it's a part of float/regexp
  + `Hash` when it's a hash of instance variables/struct members

Which is mostly correct.
  + `true`/`false`/`nil`/integers/floats always point to the same object in the memory
  + If we dump `[s, Regexp.new(s)]` it will not create an object link. Why? `Regexp.new` always creates a copy of the string inside, so there's no way to save any memory using object links, `regexp.source` is always a unique object in the memory.
  + If we dump two objects with the same hash of instance variables there are two cases:
    + If these objects have the same class, then it's almost 100% gurantee
      that these objects are the same. Then the output stream looks like `[Object, ObjectLink]`.
    + If these objects have a different class but the same hash of ivars -
      then we shouldn't create an object link and that's correct.

The code for object links is quite big to paste it here, you can find it
[here](https://github.com/iliabylich/pure_ruby_marshal/commit/a26aa1aecec20cee1c2a908673fe79275dcdfa58)

## Other cases

To be honest, there are a few more cases, but they are too complex
for implementing and pasting it here. I'm not going to cover here:
+ Encoding
+ User marshalled objects (i.e. with `marshal_dump/load`)
+ Bignum
+ Edge cases of `Float` like infinity
+ Some stuff that I just don't know from marshalling like UserDef/Hashdef/ModuleOld

# Writing

If you've read the previous part, I suppose it should be clear for you how to write it youself :)

# Optimizations

Let's try on the real-world examples. Stuff that usually gets serialized is your data.
And I can remember only one example where `Marshal` is used - model caching.

Imagine I have a model `User` with the following columns:
  + `id`
  + `email`
  + `created_at`/`updated_at`

``` ruby
Marshal.dump(User.first).length
 => 891
```

That's a lot... The easiest solution is to dump only `attributes` hash:
``` ruby
class User < ActiveRecord::Base
  def marshal_dump(*)
    attributes
  end

  def marshal_load(attributes)
    initialize(attributes)
    @new_record = id.blank?
  end
end

Marshal.load Marshal.dump(User.first)
 => #<User id: 131 ...>
Marshal.dump(User.first).length
 => 205
```

That's better, let's see what else we can improve.

By default `ActiveRecord::Base#attributes` returns a hash where keys are `String`. That sounds like
it can be optimized by calling `attributes.symbolize_keys` in `marshal_dump` to use symbol links
when we dump a collection. **But that's not true!**

``` ruby
User.all.map(&:attributes).map(&:keys).flatten.map(&:object_id).uniq.length
 => 4 (for 4 columns)
```

This is an amazing example of optimization!

``` ruby
User::AttrNames.constants
 => [:ATTR_9646, :ATTR_56d61696c6, :ATTR_36275616475646f51647, :ATTR_57074616475646f51647]

User::AttrNames.constants.map do |const_name|
  User::AttrNames.const_get(const_name)
end
 => ["id", "email", "created_at", "updated_at"]
```

Keys in the `attributes` hash are from these constants, so they are always the same
objects, so `Marshal` cashes them using object links.

By the way, why does ActiveRecord save attribute names as `String`? The answer is simple,
`Symbol` doesn't support encodings.

Well, currently our record is represented by:
  + class name as a symbol (so it's cacheable)
  + hash of attributes:
    1. `String` "id" / object link
    2. `Integer` id - can't do anything here
    3. `String` "email" / object link
    4. `String` email - nothing to optimize
    5. `String` "created_at" / object link
    6. `ActiveSupport::TimeWithZone` created_at - probably optimizable
    7. `String` "updated_at" / object link
    8. `ActiveSupport::TimeWithZone` udpated_at - see #6

Let's see what happens when we serialize `AS::TimeWithZone`:
``` ruby
Marshal.dump(Time.zone.now).length
 => 120
```

So, from 205 characters of our record 120 is taken for a time with zone. `AS::TimeWithZone` has
its custom implementation of `marshal_load` that converts it to `[utc, zone, time]`.
If you always use your application time zone,
then you can store it as epoch time:

``` ruby
class ActiveSupport::TimeWithZone
  def marshal_dump(*)
    to_i
  end

  def marshal_load(epoch)
    utc = Time.at(epoch)
    zone = Rails.application.config.time_zone
    local = utc.in_time_zone(zone)
    initialize(utc.utc, ::Time.find_zone(zone), local)
  end
end
```

``` ruby
Marshal.dump(User.first).length
 => 139
Marshal.dump(User.first(3)).length
 => 261
```

Personally I'm not sure that the next optimization is a good idea, but you can try:

``` ruby
class User < ActiveRecord::Base
  def self.marshallable_attributes
    @marshallable_attributes ||= %w(
      id
      email
      created_at
      updated_at
    )
  end

  def marshal_dump(*)
    self.class.marshallable_attributes.map do |attribute|
      attributes[attribute]
    end
  end

  def marshal_load(attributes)
    attributes = self.class.marshallable_attributes.zip(attributes)
    attributes = Hash[attributes]
    initialize(attributes)
    @new_record = id.blank?
  end
end
```

The idea is to store a hard-coded sequence of attribute names and using it
serialize only values of attributes. It allows us to not serialize object links
of attribute names.

Why can't we just get `attributes.keys.sort`? Because you can add one more
column that may come to the middle of that array. As a result, your
attributes may become shuffled. But probably you can avoid it by changing a cache key
right after adding a column.

You can extend this solution to something similar to `ActiveModel::Serializer`
where you have a separate serializer class for every model class.

Let's see what this optimization gives us:
``` ruby
Marshal.dump(User.first).length
 => 84
Marshal.dump(User.first(3).to_a).length
 => 190
```

That's 10 times less then initial size, good job!

Of course, if you have columns with type `text` there's nothing to
optimize, this is my example and my expirience.

# What is it for?

I've been working for about 3 weeks on implementing `Marshal` module for [opal](https://github.com/opal/opal).
It's almost compatible with MRI implementation. I believe `Marshal` is the thing
that can bring real isomorphism to opal applications. If you have a dumpable object
on the server and its class was compiled on the client, you can pass it **directly**
from the server without any serialization on the client/server.

# Links

[Github repo with PureRubyMarshal](https://github.com/iliabylich/pure_ruby_marshal)

[Possible Opal implementation of Marshal](https://github.com/opal/opal/pull/1191)
