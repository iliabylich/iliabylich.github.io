---
layout: post
title:  "What is Ruby DSL"
date:   2015-05-26 00:00:00 +0300
categories: ruby DSL
toc: true
---
As you already know, DSL means domain-specific language. It's like a language in a language. Here are some examples that we use every day:
``` ruby
class User
  attr_reader :name
end

class Profile < ActiveRecord::Base
  has_many :posts
end

class ApiController < ActionController::Base
  before_action :authenticate
end
```

But all these examples use rails-provided DSL, how about your own? First of all, you should know that all these methods (`attr_reader`, `has_many` and `before_action`) are actually class-methods of `Module`, `ActiveRecord::Base` and `ActionController::Base`. So you can write something like:
``` ruby
class TestClass
  def self.my_dsl_method(dsl_method_arg)
    puts "dsl method called with #{dsl_method_arg}"
  end
  my_dsl_method :some_argument
end
```

And the body of `my_dsl_method` can be more dynamic then just printing a string. It can define methods, execute some calculations, etc. Less words, more code!

For example, we want a DSL like this:
``` ruby
survey 'Survey title' do
  question 'Question 1 title' do
    answer 'Option 1'
    answer "Option 2"
  end

  question 'Question 2 title' do
    answer 'Option 3'
    answer 'Option 4'
  end
end
```

## Blocks and context

Ruby blocks are the heart of DSL in ruby. Blocks can be `call`-ed, `instance_eval`-ed and `instance_exec`-ed. The difference between `call`-ing and `instance_eval`-ing is the context of block execution:
``` ruby
require 'ostruct'

proc1 = proc { puts self.name }
proc2 = proc { |obj| puts obj.name }

object = OpenStruct.new(name: 'John')
object.instance_eval(&proc1) # self in proc1 is object
# => 'John'
proc2.call(object) # obj in proc2 is object
# => 'John'
```

`instance_exec` is pretty much like `instance_eval` but it also takes extra arguments that becomes accessible in the block.

## So?

Here is the draft of the code that simulates a survey:
``` ruby
class Survey
  def initialize(title, &block)
    @title = title
    instance_eval(&block)
  end

  def question(question_title, &block)
    # Storing question
  end

  def run
    # Printing stored questions
  end
end

def survey(title, &block)
  Survey.new(title, &block).run
end

survey 'Survey title' do
  question 'My Question1'
  question 'My Question2'
end
```

This DSL does not make anything yet, but it's a good prototype of how it should look. We `instance_eval` passed block, so DSL-method `question` becomes instance method of `Survey` class, and instance of `Survey` becomes the context of internal DSL block.

Let's implement `question` and `run` methods.

Method question should build instance of `Question` class with passed `title` and store it.

``` ruby
class Survey
  # ...
  def question(question_title, &block)
    questions << Question.new(question_title, &block)
  end

  def questions
    # With this code we have pre-defined value
    # of `questions` method
    @questions ||= []
  end
end
```

And the `Question` class:

``` ruby
class Question
  def initialize(title, &block)
    @title = title
    instance_eval(&block)
  end

  def answer(answer_text)
    answers << answer_text
  end

  def answers
    @answers ||= []
  end
end
```

Let's put all together:
``` ruby
survey 'Survey title' do
  # here `self` is an instance of Survey class
  # so we can call ANY instance methods of Survey
  question 'Question 1' do
    # here `self` is an instance of Question class
    answer 'Answer 1'
    answer 'Answer 2'
  end
end
```

The only missing method is `Survey#run`:
``` ruby
class Survey
  # ...
  def run
    puts @title
    questions.each(&:run)
  end
end

class Question
  # ...
  def run
    puts @title
    answers.each_with_index do |answer, index|
      puts "#{index + 1}. #{answer}"
    end
    result_number = gets.chomp.to_i - 1
    result = answers[result_number]
    puts "Your answer is #{result}"
  end
end
```

Full code is available on gist:
https://gist.github.com/iliabylich/9ea5dd31ee92571f8d59

Or a bit more complex example with tests on github:
https://github.com/iliabylich/multi-level-dsl

## Advanced DSL example

DSL that we've made before allows us to write some code using shortcuts, but usually we create DSL to define new methods (or change existing ones).

Let's build something like:
``` ruby
class User
  extend MyModuleWithDSL

  accessor_with_default_value :full_name, 'Fname Lname'
end

u = User.new
u.full_name
# => 'Fname Lname'
u.full_name = 'Another Name'
u.full_name
# => 'Another Name'
```

## Defining methods dynamically

Here we need `define_method` method. It takes two parameters: a method name and a block that becomes body of created method:

``` ruby
class User
  define_method :my_method do |arg|
    puts "Called with #{arg}"
  end
end

User.new.my_method(123)
# => Called with 123
```

A huge difference between defining method with `define_method` and `def method_name` is that `define_method` does not lose current context:
``` ruby
local_var = 'test'
define_method :method1 do
  puts local_var # Works here
end
method1
# => 'test'

def method2
  puts local_var # Does not work
end
method2
# => NameError: undefined local variable or method `local_var' for main:Object
```

This allows us to pass any variables to DSL and use them in dynamically defined methods.

## Where is the code?

Here is it:

``` ruby
module ModuleWithDSL
  def accessor_with_default_value(attr_name, default_value)
    # Here we need to build something like
    # def full_name
    #  @full_name || 'Fname Lname'
    # end
    # attr_writer :full_name
    define_method attr_name do
      instance_variable_get("@#{attr_name}") || default_value
    end
    attr_writer attr_name
  end
end

class User
  extend ModuleWithDSL
  # so all methods from ModuleWithDSL
  # become class-methods of User
  accessor_with_default_value :full_name, 'default'
end

u = User.new
u.full_name
# => 'default'
u.full_name = 'Full Name'
u.full_name
# => 'Full Name'
```

## What's wrong in the previous example?

What I personally don't like here is that there is no way to override defined method using `super` (I don't like `alias_method_chain`, we really can make it work using super method). Why super does not work? Because the method is defined directly on the class:
``` ruby
User.instance_method(:full_name)
# => #<UnboundMethod: User#full_name>
```

The following example is quite complex, but still:
``` ruby
module ModuleWithDSL
  def self.extended(base)
    mod = base.const_set(:AccessorsWithDefaultValues, Module.new)
    base.send(:include, mod)
    super
  end

  def accessor_with_default_value(attr_name, default_value)
    const_get(:AccessorsWithDefaultValues).module_eval do
      define_method attr_name do
        instance_variable_get("@#{attr_name}") || default_value
      end
      attr_writer attr_name
    end
  end
end
```

Here when this module `extend`-s some class/module, we define a sub-module called `AccessorsWithDefaultValues` in the class using `extended` hook, include it in the class and define ALL dynamic methods on this module. What does it give us?

``` ruby
User.instance_method(:full_name)
# => #<UnboundMethod: User(User::AccessorsWithDefaultValues)#full_name>
User.ancestors
# => [User, User::AccessorsWithDefaultValues, ...]
```

So now we can override our `accessor`-s on `User`:
``` ruby
class User
  extend ModuleWithDSL
  accessor_with_default_value :counter, 123

  def counter
    super + 10 # default behavior + 10
  end
end

user = User.new
user.counter
# => 133 (123 + 10 = default + custom)
user.counter = 15
user.counter
# => 25
```

ActiveRecord does the same job, you can override any column-method:
``` ruby
User.instance_method(:email)
# => #<UnboundMethod: User(#<#<Class:0x00000008c44ac0>:0x00000008c44b38>)#email(__temp__56d61696c6)>
# This module is anonymous, but it doesn't matter, it wasn't defined _directly_ on User:

class User < ActiveRecord::Base
  # If you have column `email`
  def email
    super.upcase
  end
end
```

A lot of developers build a DSL in their libraries to simplify the code, and this allows us to write our code in a more declarative fashion. But they forgot (quite often) about allowing us to override the code generated with this DSL. When the library doesn't support overriding generated code, use `alias_method`:

``` ruby
class User
  include SomeModule

  # here we call some DSL
  # and, let's say, it generates method 'run'
  # and we need to add before hooks
  # to this method
  alias_method :original_run, :run
  def run
    # before hook goes here..
    original_run
  end
end
```

## Conclusion

DSL is a tool for fast prototyping, here are some advantages:
1. It makes your code look nicer (the code that calls DSL)
2. Amazing level of abstraction, you can focus exactly on your logic and forget about any code-related stuff.

And disadvantages:
1. Your code becomes slow (usually just a little bit, but sometimes it can be critical)
2. The effort to maintain your DSL grows with its complexity. Yes, it's very easy to make DSL implementation look ugly

I really love to use DSL from third-party libraries (AR, RSpec, Capybara, FactoryGirl). But as I said before, I don't write it in enterprise and usually I ask people to avoid it.

But if you write DSL, please wrap generated stuff into module to let developers override your code and be happy, this is what DSL is made for :)
