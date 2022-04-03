---
layout: post
title:  "Saving execution context for later debugging"
date:   2015-08-21 00:00:00 +0300
categories: ruby binding closure debugging
toc: true
---
Consider the following situation: you've got an exception in production. Of course, all of us are good developers, but you know, sometimes \*it just happens. What do you usually do to get some information about the error? You just grab the request parameters to test it locally, right? Then I might have a better solution for you: dump your memory once an error happens and restore the dump later to debug it.

## Binding

In Ruby the best candidate for doing this is `Binding` class. If you have a binding, your can easily do some debug using well-known `pry` gem. But the binding itself cannot be dumped (at least not, using default Ruby tools).

How to get a local binding? Just use `binding`. How to get a binding from an object? Just add a method to you class:
``` ruby
class MyClass
  def local_binding
    binding
  end
end

MyClass.new.local_binding
# => #<Binding>
```

Binding encapsulates the execution context at the place in your code where the interpreter is currently running and retains this context for future use. So, to dump and load back a binding we need to:
+ Dump the context of the binding (i.e. `binding.eval('self')`)
+ Dump all local variables (i.e. `binding.eval('local_variables')`)

In fact, that's all you need to restore your binding.

## Marshaling

How can we dump an arbitrary structure? Ruby has a class in stdlib called `Marshal`. The two core methods of this class are `dump` and `load`:
``` ruby
Point = Struct.new(:x, :y)
a = Point.new(34, 65)
marshaled = Marshal.dump(a)
# => "\x04\bS:\nPoint\a:\x06xi':\x06yiF"
Marshal.load(marshaled)
# => #<struct Point x=34, y=65>
```

## Limitations

Unfortunately, not everything can be marshaled. According to the documentation, the following objects cannot be dumped:
+ bindings (e.g., the return value of binding itself)
+ procedure or method objects (e.g., `proc {}` or `object.method(:method_name)`)
+ instances of class IO (e.g., `IO.new(1)` or `StringIO.new`)
+ anonymous classes and modules (e.g. `Class.new` or `Module.new`)

That sound really sad, but in most cases we can ignore these limitations. When was the last time you needed to debug an IO object that was doing something strange? In real life we rarely use any of these classes **during debugging process**. So, instead of dumping and loading back  an `IO` object we can just return a new one.

## Converting objects to marshalable data

Well, we can patch every single class in Ruby and add `marshal_load` and `marshal_dump` hooks to them, but that's just horrible. It would be much, much better to write a set of classes that are each responsible for converting a specific group of objects.

With that in mind I’ve implemented:
1. `PrimitiveDumper` - for dumping primitive objects, like numbers, booleans.
2. `ArrayDumper` - for arrays.
3. `HashDumper` - for hashes.
4. `ObjectDumper` - for custom objects.
5. `ClassDumper` - for classes.
6. `ProcDumper` - for proc/method objects
7. `MagicDumper` - for "magical objects" (see 'dumping magical objects' section)
8. `ExistingObjectDumper` - for existing objects (see 'dumping recurring objects' section)

Every dumper takes an object that we need to dump and returns its marshalable representation. Later you can use the same dumper to deconvert representation back and get the original object.

You can find the implementation of these dumpers [here](https://github.com/iliabylich/binding_dumper/tree/master/lib/binding_dumper/dumpers) and the specs for them [here](https://github.com/iliabylich/binding_dumper/tree/master/spec/binding_dumper/dumpers).

Here is, probably, the most complicated example:
``` ruby
def undumpable_recursive_object
  @undumpable_recursive ||= begin
    p = Point.allocate
    p.x = p
    p.y = StringIO.new
    p
  end
end
```

After converting this object using a system of dumpers result looks like this:
``` ruby
{
  _klass: Point,
  _ivars: {
    :@x => {
      _existing_object_id: 1234566 # or similar
    },
    :@y => {
      _klass: StringIO,
      _undumpable: true
    }
  },
  _old_object_id: 1234566 # same as above
}
```

This hash can be easily marshaled and restored back. But yes, we lose our `StringIO` instance - when the object is loaded back, that variable will be blank.

## Dumping Magical objects

After writing the first version of the library, I've tested it with a blank Rails application. The testing code was:
``` ruby
class UsersController < ApplicationController
  def index
    @users = User.all.to_a # 5 records
    local_proc = proc { 2 + 2 }
    render json: @users
    StoredBinding.create(data: binding.dump)
  end
end
```

The length of the dump was  ~30 screens and it took ~20 seconds to generate it. Most of the data was coming from objects related to Rails itself. Things like Rails configs, backtrace cleaners, arrays of middlewares, and so on. Do we need them? No. These objects are the same for every request, so we can ignore them.

But at the same time, we need to save and restore all references from 'dumpable' objects to 'magic' objects, we can't just omit them. This logic is implemented in [BindingDumper::MagicObjects](https://github.com/iliabylich/binding_dumper/blob/master/lib/binding_dumper/magic_objects.rb) module and here’s how you can use it:
``` ruby
class A
  @config = :config
end

BindingDumper::MagicObjects.register(A)
p BindingDumper::MagicObjects.pool
=> {10633360=>"A", 600668=>"A.instance_variable_get(:@config)"}
```

So, it builds a mapping between `object_id` and the way how to get this object. Using this functionality we can easily get whether existing object is 'magical', and if yes - dump its string representation (to eval it on loading phase). Let's say, we need to dump `Rails.application.config`, one of the 'magical' objects. We need to get its `object_id`, find it in the pool and remember the string that returns rails config after evaluation, i.e.:
``` ruby
"Rails.
  instance_variable_get(:@app_class).
  instance_variable_get(:@instance).
  instance_variable_get(:@config)"
```

After this optimization we have to spend ~20ms to build an object pool and ~200ms to dump a binding.

## Dumping recurring objects

We can optimize it even more. A lot of things like `request`, `response` are shared as instance variables across ~10 objects. We can dump our `request` object only once, remember its `object_id` and use a reference while dumping other objects that use it.

Let's say, we are in the initial memory (MEM1). We dump a binding, open another console with separated memory (MEM2) and restore a binding. In the example above (about recursive structure) there was a key `:_existing_object_id` that returns an `object_id` from MEM1.

In MEM2 we restore a binding and create a mapping
``` ruby
{
  object_id_from_MEM1 =>
  restored_object_in_MEM2
}
```

Using this mapping (in the gem it's called [memories](https://github.com/iliabylich/binding_dumper/blob/master/lib/binding_dumper/memories.rb)) we can restore the reference to duplicated objects.

## Restoring a binding

At this point you can be really confused, but relax, we are almost done.

So, we have a binding. To dump it we need to:
1. Build a `hash1` with the context of binding and local variables
2. Convert it to a marshalable nested `hash2`
3. `Marshal.dump(hash2)`
4. Store the result in any persistent storage.

To load it back:
1. `Marshal.load` the dump to get `hash2`
2. Convert `hash2` to `hash1` using the same converters
3. Load the context and all local variables from `hash1`
4. Patch the context a little bit to make it pretty.

Steps 1-4 and 1-3 are already implemented. The last step – making the context pretty – means that we need to inject local_binding method into the context and make it look like the “real” binding (inject local variables to the binding).

``` ruby
# we just have it,
# it's a `self` from the place where `binding.dump` was called
context

# and we have also local variables
local_variables

# here we need to get a binding that:
subject.local_binding.eval('self') == context
# => true
subject.local_binding.eval('local_variables') == local_variables
# => true
```

The pseudo-code for loading and patching the context looks like:
``` ruby
marshaled = StoredBinding.last.data
converted = Marshal.load(marshaled)
restored = Dumpers.load(converted)

context = undumped[:context]
locals = undumped[:locals]

class << context
  def local_binding
    result = binding

    locals.each do |lvar_name, lvar|
      result.local_variable_set(lvar_name, lvar)
    end

    result
  end
end
```

The actual implementation can be found [here](https://github.com/iliabylich/binding_dumper/blob/master/lib/binding_dumper/core_ext/binding_ext.rb). After calling `Binding.load(dumped).pry` you can start debugging it!

## Compatibility with old versions of Ruby

Currently the gem supports Ruby versions from 1.9.3 to 2.2.3. I had a few issues with porting the code from 2.0.0 to 1.9.3, like the lack of kwargs and `Module#prepend`. The funniest one was that in versions before 2.1.0 there is no `binding.local_variable_set` - there is only `binding.eval` that takes a string, not a block.

How can we pass a complex object to `eval`? The solution is not so difficult, because we have the object right here and right now, and the binding uses the same memory as the main thread. This means that we can pass the `object_id` of our object to `eval` string and get it there using `ObjectSpace._id2ref`:
``` ruby
undumped[:lvars].each do |lvar_name, lvar|
  result.eval("#{lvar_name} = ObjectSpace._id2ref(#{lvar.object_id})")
end
```

## Known issues

I've tested the gem locally with a few projects. Everything was fine, but:
1. Encoding. The data that the gem produces should be stored in UTF-8
2. The difference between Rails server and Rails console. There are some classes that are loaded only when the server is started (like `Rails::BacktraceCleaner` and some others from `NewRelic` gem). You have to require corresponding files manually before loading the binding in the console.

## Demo

To try it out, clone the [GitHub repository](https://github.com/iliabylich/binding_dumper), install dependencies, prepare the database using `bin/dummy_rake db:create db:migrate`, and start the server via `bin/dummy_rails s`. Then visit [http://localhost:3000/users](http://localhost:3000/users) to dump the binding of `UsersController#index`. After that you can open a console using `bin/dummy_rails c` and run `StoredBinding.last.debug`. You’re now in your controller, in the same state that it was in a moment ago when you hit that /users page!.

## Testing

The gem is fully tested with its specs running on [Travis CI](https://travis-ci.org/iliabylich/binding_dumper/). There's also a [script](https://github.com/iliabylich/binding_dumper/blob/master/bin/multitest) that can be used to run the whole test suite locally on **every** supported version of Ruby. But that's definitely not enough for a gem to become completely production-ready.

That's why I ask everyone who read this article: if you think that the idea of this gem should stay alive, that this method of debugging can be useful, and you would like to use it yourself, please, try it out locally and share your finding with me (via Twitter or Github).

## Links

[Github repo](https://github.com/iliabylich/binding_dumper)
