---
title: "Capybara and asynchronous stuff"
date: "2015-07-01"
cover: ""
tags: ["Ruby", "Capybara", "WebDriver"]
---

## What is Capybara, Poltergeist and PhantomJS?

In this part I will try to cover the following aspects:

1. Running asynchronous code in a web driver
2. Making the call synchronous
3. Wrapping it into some common solution
4. Advanced example - working with IndexedDB from Capybara

### PhantomJS

First of all, we need to know what is PhantomJS. I would say it's a 'tool that acts like a browser but can be controlled from outside using simple command interface'. In more common words, it's a web driver. It's a full-featured WebKit (an engine of Chrome/Safari/few last versions of Opera and other browsers), but in console. You can use it for scripting, automating or testing.

### Poltergeist

Poltergeist is a Ruby wrapper for PhantomJS. Usually you write the code for PhantomJS on JavaScript, with Poltergeist you can run it on Ruby.

### Capybara

Well, I'm pretty sure you know what it is. It's a test framework. And it supports different web drivers like:

`RackTest`  - web driver that does not support JavaScript, extremely fast, but deadly primitive; best solution for tests that don't require any JavaScript execution

`Selenium`  - the most popular web driver

Advantages:

1. Supports a lot of browsers (so you can run multiple test suites in different browsers)
2. Super stable

Disadvantages:

1. Requires X server to be installed on the machine that runs it (some cloud CI services just don't have it)
2. Opens a real browser that executes your test scenario, that slows your test suite.

`Capybara-webkit` - headless WebKit (does not run a browser), but still requires X server

`Poltergeist` - see above, headless WebKit, **does not require X server**, so you can run it everywhere.

## Asynchronous JavaScript code in your tests

When you write integration tests sometimes you need to run asynchronous JavaScript in context of your page and get a response back to Ruby. Here is an example from my current project: the client part of our application supports offline mode, so we store the data in WebSQL and sync it with server once connection is restored. The API of WebSQL is asynchronous, so we have a result of execution in some provided callback:

```js
// something like
WebSQLWrapper.execute('select 1', function(result) {
  // result of execution becomes available after some time
});
```

We have integration tests with Capybara+Poltergeist+PhantomJS combo that tests the whole stack of interaction between a client-side application and a server API. And WebSQL should be flushed between tests or populated with startup data before some specific tests. We also have delayed operations on the client that runs periodically.

All these features require us to run JavaScript code manually in PhantomJS between/before tests.

## Capybara/Poltergeist/PhantomJS installation

Quite simple:
1. PhantomJS: `sudo apt-get install phantomjs` or download binaries from the official site (you can even try 2.0 beta there).
2. Capybara: `gem 'capybara'` and that's it.
3. Poltergeist: `gem 'poltergeist'`.

Require both of them in your `spec_helper` and force Capybara to use Poltergeist as a web driver:

```ruby
require 'capybara/rails'
require 'capybara/poltergeist'

Capybara.register_driver :poltergeist_debug do |app|
  driver_options = {
    inspector: true,
    timeout: 5,
    js_errors: false,
    debug: false,
    phantomjs_logger: Logger.new('/dev/null')
  }
  if ENV['DEBUG_PHANTOMJS']
    driver_options.merge!({
      logger: Kernel,
      js_errors: true,
      debug: true,
      phantomjs_logger: File.open(Rails.root.join('log/phantomjs.log'), 'a')
    })
  end
  Capybara::Poltergeist::Driver.new(app, driver_options)
end

Capybara.javascript_driver = :poltergeist_debug
Capybara.default_driver = :poltergeist_debug
```

With this configuration Poltergeist does not print any noisy output, but you can enable it by passing `DEBUG_PHANTOMJS=true`

## Small example

Here is a simple of the code that returns it's response asynchronously:

```js
setTimeout(function() {
  alert('Thanks for waiting 1 second');
}, 1000);
```

This code actually does nothing, but it's a demonstration of how asynchronous stuff works.

## But... Capybara waits for my AJAX requests

Yes, when you write something like:

```ruby
it 'displays a message' do
  expect(page).to have_text('Hey')
end
```

and at the moment of running this expectation your page does not have this text **but receives it a second later** - the test will be green. Why?

Capybara has a setting called `default_wait_time` (was changed to `default_max_wait_time`, but is still acceptable) which is 2 seconds by default. [Here is how Capybara uses it](https://github.com/jnicklas/capybara/blob/master/lib/capybara/node/base.rb#L76).

It runs the code again and again, and stops if

1. It has a result of code execution - then it simple returns it
2. The time has come (2 seconds) - then it raises an error

(A little remark here. Capybara saves the time on the beginning of this method and on every iteration compares this time with `Time.now` - this is a very nice hack to save Capybara from wrapping API calls into `Timecop.freeze` - good job!)

## Can we reuse it?

Yes, of course. Let's simplify it a little bit:
```ruby
module WaitHelper
  extend self

  # Calls provided +block+ every 100ms
  #   and stops when it returns false
  #
  # @param timeout [Fixnum]
  # @yield block for execution
  #
  # @example
  #   current_time = Time.now
  #   WaitHelper.wait_until(3) do
  #     Time.now - current_time > 2
  #   end
  #
  #   # 2 seconds later ...
  #   # => true
  #
  #   current_time = Time.now
  #   WaitHelper.wait_until(3) do
  #     Time.now - current_time > 10
  #   end
  #
  #   # 3 seconds later (after timeout)
  #   # => false
  #
  def wait_until(timeout, &block)
    begin
      Timeout.timeout(timeout) do
        sleep(0.1) until value = block.call
        value
      end
    rescue TimeoutError
      false
    end
  end
end
```

Alright, now let's use it.

1. Modify JavaScript code to set result into some global variable (JavaScript function context, you know)
2. Run the code that gets response in callback.
3. Run the code that polls the response from global variable
4. Wrap this code with `WaitHelper.wait_until(timeout) {}`

```ruby
code_for_execution = <<-JS
  setTimeout(function() {
    window.asyncResponse = 'some response';
  }, 1000)
JS

code_for_polling = 'window.asyncResponse'

Capybara.current_session.evaluate_script(code_for_execution)
result = WaitHelper.wait_until(2) do
  Capybara.current_session.evaluate_script(code_for_polling)
end
puts result
# => 'some response'
```

## Can we organize it as a common reusable solution?

Why not. Here is a gem called [`capybara-async-runner`](https://github.com/iliabylich/capybara-async-runner). And here is how to use it.

Installation:

```ruby
# Gemfile
gem 'capybara-async_runner'
# spec/spec_helper.rb
require 'capybara/async_runner'
```

First of all, I don't like to mix JavaScript and Ruby code in a single file, so we need templates (like `.js.erb`).
You need to specify the directory with templates:

```ruby
# spec/spec_helper.rb
Capybara::AsyncRunner.setup do |config|
  config.commands_directory = Rails.root.join('spec/fixtures/async_commands')
end
```

Let's write our first command
```ruby
class TestCommand < Capybara::AsyncRunner::Command
  # global command name
  self.command_name = :test_command_name

  # .js.erb file in directory specified above
  self.file_to_run = 'template'

  response :parsed_json do |data|
    JSON.parse(data)
  end
end
```

This class follows the Command pattern, you can invoke it in the following way:

```ruby
Capybara::AsyncRunner.run(:test_command_name)
# or
TestCommand.new.invoke
```

Let's create our template for this command:

```js
// spec/fixtures/async_commands/template.js.erb
setTimeout(function() {
  var json = JSON.stringify([1,2,3]);
  <%= parsed_json(js[:json]) %>
})
```

There are few things that I need to explain:

1. `parsed_json` - this is an output point from the script that was defined in the command class. When you call it in the template, it embeds some JavaScript that stores passed data into the global variable. `parsed_json(123)` produces something like `window.parsed_json_result = 123` (so we can can grab this response later from the second script)
2. `js` - this is a proxy method from the gem that acts like a Hash. The method `[]` on this Hash returns passed key (and the whole method `js` is kind of "JavaScript memory"). `<%= parsed_json(js[:json]) %>` produces `window.parsed_json_result = json` which is exactly what we need.
3. When you call `Capybara::AsyncRunner.run(:test_command_name)`, the gem executes the script generated from your template in the context of `Capybara.current_session`. Then it subscribes to all defined responses (this is actually, the second script) and returns the first one that becomes defined (`window.parsed_json_result` in this case).
4. The block that we have specified for `parsed_json` is like a handler for transforming data which it returns (we invoke `parsed_json` method with `json` variable which contains raw JSON, our handler parses it)

If you are familiar with templates in Ruby, just quickly look at the [code](https://github.com/iliabylich/capybara-async-runner/blob/master/lib/capybara/async_runner/env.rb), this class is a context of rendering.

## Wrapping up

You need to follow these steps to create a command using a gem:
1. Specify the directory with templates in the gem configuration file
2. Create a command class
3. Specify its name
4. Specify the name of template
5. Create a template file
6. Define a response(s)
7. Call them in the template

## Passing data to template

You can pass any data to the template:

```ruby
class TestCommand < Capybara::AsyncRunner::Command
  self.command_name = :test
  self.template = 'template'
  # if you don't pass any block
  # it will return raw value
  response :done
end

data = {
  name: 'Ilya'
}
Capybara::AsyncRunner.run(:test, data)
# => 'Ilya'
```

And use it template:

```js
someLongRunningMethod(function() {
  // 'done' is a method that generates JavaScript
  // 'data' returns data that we've passed to 'run'
  // :name is a key in that Hash
  <%= done(data[:name]) %>
})
```

## Let's write something complex

As I mentioned before, on my current project we use WebSQL, but it's [deprecated](http://www.w3.org/TR/webdatabase/), so I'm not going to use it in examples. Instead let's write a wrapper for IndexedDB.

First of all, we need a JavaScript wrapper, I don't like the native IndexedDB API. First result from google = [`Dexie.js`](http://www.dexie.org/).

Let's plan our scenario:
1. Visit any page
2. Inject `Dexie.js` into the page
3. Create IndexedDB instance
4. Write some data to the database
5. Read them and print

### Visiting the page

```ruby
Capybara.current_session.visit('http://google.com')
```

### Injecting `Dexie.js` into the page

+ [Command](https://github.com/iliabylich/capybara-async-runner/blob/master/examples/indexeddb/commands/wrapper_loader.rb)
+ [Template](https://github.com/iliabylich/capybara-async-runner/blob/master/examples/indexeddb/templates/wrapper/inject.js.erb)

How to invoke:

```ruby
module IndexedDB
  URL = 'http://www.url.to.dexie.js.source'
end

Capybara::AsyncRunner.run('indexeddb:wrapper:inject', url: IndexedDB::URL)
```

Explanation:

1. JavaScript code detects whether the library has already been loaded
2. If not - appends `<script>` tag to the `<head>`
3. If yes - returns 'success'
4. If 3 seconds passed and we still have no library code - returns 'error'

The command raises an exception if response is `error`.

All this manipulations are synchronous for Ruby. The end of running the command means that we can continue execution.

Moreover, this command is safe, we can call it multiple times and it will inject the script into the page only once.

### Create IndexedDB instance

+ [Command](https://github.com/iliabylich/capybara-async-runner/blob/master/examples/indexeddb/commands/wrapper_initializer.rb)
+ [Template](https://github.com/iliabylich/capybara-async-runner/blob/master/examples/indexeddb/templates/wrapper/initialize.js.erb)

How to invoke:

```ruby
Capybara::AsyncRunner.run('indexeddb:wrapper:initialize')
```

Explanation:

1. JavaScript opens the database using `Dexie.js`
2. If `callback` was called, returns `sucess`
3. If `errback` was called, returns `error` with error message
4. Ruby command raises error if `error` was returned

### Write some data to the database

+ [Command](https://github.com/iliabylich/capybara-async-runner/blob/master/examples/indexeddb/commands/insert.rb)
+ [Template](https://github.com/iliabylich/capybara-async-runner/blob/master/examples/indexeddb/templates/commands/insert.js.erb)

How to invoke:

```ruby
user_data = { name: 'Some Name' }

user_id = Capybara::AsyncRunner.run('indexeddb:insert', store: 'users', data: user_data)
p "User ID: #{user_id}"
```

This step is quite simple if you understand the previous one.

### Reading data

+ [Command](https://github.com/iliabylich/capybara-async-runner/blob/master/examples/indexeddb/commands/query.rb)
+ [Template](https://github.com/iliabylich/capybara-async-runner/blob/master/examples/indexeddb/templates/commands/query.js.erb)

How to invoke:

```ruby
methods = [
  { method: 'where', arguments: ['id']},
  { method: 'equals', arguments: [user_id] },
  { method: 'toArray', arguments: [] }
]

p Capybara::AsyncRunner.run('indexeddb:query', store: 'users', methods: methods)
```

Here we pass an array of methods and their arguments to template, iterate over them and build a `Dexie.js` scope (just like `ActiveRecord::Relation`), and return a result back to ruby command.

After wrapping it even more we can get interface like [this](https://github.com/iliabylich/capybara-async-runner/blob/master/examples/indexeddb/indexdb.rb#L133)

Full example can be found [here](https://github.com/iliabylich/capybara-async-runner/tree/master/examples/indexeddb)

## Conclusion

I would say the topic of this article is not so popular. Single page applications that work in offline mode (and because of this, use WebSQL/IndexedDB/some other asynchronous storage, probably) are still not frequent today. Usually when you build an SPA, you just write your tests using Jasmine or something like that. You mock your requests to the server API and test your client in some isolated environment. But these tests are still functional (you verify a single component - client, in this case - but not the whole application).

Someday working on a rich client-side application, remember about the idea of this post. You can control the logical flow of your client-server communication in integration tests, and this is good. Wrap your client code, build a micro-framework on top of this gem and test every piece of your code.
