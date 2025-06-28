---
title: "ExceptionManager gem"
date: "2015-04-14"
cover: ""
---

# What is this?

`ExceptionManager` is a gem for getting extra information from your exception.

Source code: [https://github.com/iliabylich/exception_manager](https://github.com/iliabylich/exception_manager)

With this gem every time when you get an exception, it's possible to grab `subject` of exception (the instance of class where `raise` happened), `locals` - local variables, `subject_instance_variables` and `subject_class_variables`

Examples:

```ruby
require 'exception_manager'
ExceptionManager.enable!

class TestClassThatRaisesException
  @@class_variable = :class_value

  def test_error(*args)
    @instance_variable = :instance_value

    raise 'Test error'
  end
end

begin
  TestClassThatRaisesException.new.test_error(1, 2, 3)
rescue => e
  puts "Subject: #{e.subject}"
  puts "Locals: #{e.locals}"
  puts "Instance variables: #{e.subject_instance_variables}"
  puts "Class variables: #{e.subject_class_variables}"
  puts "Summary: #{e.summary.inspect}"
end
```

This code snippet prints:

```ruby
Subject: #<TestClassThatRaisesException:0x00000001512268>
Locals: {:args=>[1, 2, 3]}
Instance variables: {:@instance_variable=>:instance_value}
Class variables: {:@@class_variable=>:class_value}
Summary: {
  :locals => {
    :klass => TestClassThatRaisesException,
    :args=> [1, 2, 3]
  },
  :subject => #<TestClassThatRaisesException:0x00000001512268>,
  :subject_instance_variables => {
    :@instance_variable => :instance_value
  },
  :subject_class_variables => {
    :@@class_variable => :class_value
  }
}
```

So, you can get local variables of you exception, instance where it happened and the whole context.

If you have `pry` installed, you can inject pry session into stored binding:

```ruby
begin
  TestClassThatRaisesException.new.test_error(1, 2, 3)
rescue => e
  e._binding.pry
end
```

# Integration with NewRelic

Just use the following snippet:

```ruby
class StandardError
  def original_exception
    message_for_new_relic = [message, summary.inspect].join(' ')
    self.exception(message_for_new_relic)
  end
end
```

NewRelic gem by default calls `.original_exception.to_s` on every caught exception. Yes, looks dirty, but saves a lot of time!

# How it works

In old versions of ruby there was a `Kernel` method called `set_trace_func` which allows to set trigger for every executed line (including c-calls).

Ruby 2.0.0 (and newer) has a class called `TracePoint` which allows to subscribe to any specific method call (`raise` in our case).

So, simplified version of the main code looks like:

```ruby
class Exception
  attr_accessor :_binding
end

TracePoint.new(:raise) do |tp|
  tp.raised_exception._binding = tp.binding
end
```

And then we can get everything through this binding

# Compatibility

`ExceptionManager` is compatible (and tested on Travis CI) with the following versions of ruby

+ 2.0.0
+ 2.1.0
+ 2.2.0

(In other words, in all versions of ruby that has `TracePoint` class).

# Questions

If you have any questions/suggestions feel free [to create an issue on GitHub](https://github.com/iliabylich/exception_manager/issues).
