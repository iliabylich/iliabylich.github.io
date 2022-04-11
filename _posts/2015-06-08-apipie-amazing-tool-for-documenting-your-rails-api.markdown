---
layout: post
title:  "Apipie - amazing tool for documenting your Rails API"
date:   2015-06-08 00:00:00 +0300
categories: Ruby Apipie documenting API
toc: true
comments: true
---
This article is about [Apipie gem](https://github.com/Apipie/apipie-rails) which provides a DSL for documenting your API. I will try to cover features that I personally use on my project.

Comparing to other tools for generating API documentation (`yardoc`, `sdoc`) I would say that the main thing that you gain with Apipie is that your documentation is a real ruby code, so you can write computations, concerns etc.

Here is a simple example of how it looks in code:

```ruby
class UsersController < ApplicationController

  resource_description do
    formats [:json]
    api_versions 'public'
  end

  api :POST, '/users' 'Create user'
  description 'Create user with specifed user params'
  param :user, Hash, desc: 'User information' do
    param :full_name, String, desc: 'Full name of the user'
    param :age, Fixnum, desc: 'Age of the user'
  end
  def create
    # Some application code
  end
end
```

So, you invoke a DSL from Apipie before your action method and it automatically comes to generated docs.

# Features

## Gathering information from your routes

In the example above we have passed an HTTP verb and a path of this action. We don't have to do it! Instead, we can simply write:

```ruby
api! 'Some description'
# other docs here...
def create
end
```

It automatically takes information from your routes that look like

```ruby
resources :users, only: [:create]
```

## API versioning

You can pass versions of your API that include this endpoint on
1. resource level (as in example)
2. action level (in pretty much the same way, `api_versions ...`

## Parameters typing

As you can see, in the first example we had 3 types:
1. `Hash`
2. `Array`
3. `Fixnum`

Apipie has also:
4. `Enum`
5. `Regexp`

## Parameters validation

We have already typed parameters, so why do we ignore it and write custom `before_action`-s for validating parameters manually? This feature is enabled by default, but if you don't think that it's a good idea to validate your parameters through documenting tool, just pass

```ruby
config.validate = false
```

The creator of the gem told me that 'People either love it or don't understand why it's even there', so it's up to you to decide if you need it.

## Concerns

I would say that it does not work as people usually expect.

`Apipie::DSL::Concern` allows you to document actions that are defined in concerns.

[A quick example](https://github.com/Apipie/apipie-rails#concerns)

Usually people think that it allows you to extract documentation from your controller in order to not mix code with documentation and application logic. And, to be honest, I think so too. The workaround for doing this goes below in the sections 'Extracting docs to mixins'

## Specs recording

Are you tired of writing examples manually? Me too :) With Apipie you can record request/response pairs to separated YAML file and display them in generated HTML. Pass `:show_in_doc` to metadata of your RSpec example and enjoy. Apipie embeds a module for recording requests to `    ActionController::TestCase::Behavior` which is the core of all requests specs for RSpec and Minitest (yes, both of them delegate performing requests internally to `ActionController::TestCase::Behavior` - [source](https://github.com/rspec/rspec-rails/blob/master/lib/rspec/rails/example/controller_example_group.rb#L12)).

## Other features
There is a plenty of other things in Apipie that are very cool, by I did not have a chance to use it yet.

1. Localization. Currently it supports English, Russian, Chinese and Brazilian. If you want to add support for your language, use this as an example - [source](https://github.com/Apipie/apipie-rails/blob/master/config/locales/en.yml)
2. Disqus integration. This is extremely useful when you have a decentralized team. If you have any questions - just leave a comment and wait for response! No need to define any models for storing users/comments/relations, everything is in the cloud.
3. Custom markup processors. Not sure that anyone can need it, default markup processor looks very stable.

# Customization

Some of these items can be difficult to explain, if you have any questions after reading it, just google it, answers for all of them should be in ruby docs (or ping me if you still don't get it).

## Extracting docs to mixins

This is the main question I have got after reading official README. I don't want to mix documentation and application logic. First of all we need to understand how exactly Apipie builds mapping between action names and compiled DSL. Even without reading source code the only guess that we may have is `method_added` hook. Every time when you define a method (on instance or on class, it does not matter), Ruby automatically fires `method_added` method on your class.

```ruby
class TestClass
  def self.method_added(method_name)
    puts "Added method #{method_name}"
  end

  def method1; end
  def method2; end
end
# => Added method method1
# => Added method method2
```

So, the algorithm is like this:
1. You invoke method `api` (or `api!`)
2. Apipie internally saves action name (let's say, as `Apipie.current_action_name`) and arguments (in `Apipie.current_action_data`, for example)
3. You invoke other DSL methods
4. Apipie appends this data to `Apipie.current_action_data`
5. You define a method (action in our case)
6. Apipie adds mapping `Apipie.current_action_name => Apipie.current_action_data` to a global Hash like `Apipie.all_docs`
7. When you visit `/docs`, Apipie displays all data from `Apipie.all_docs`

And if it's true we can write something like:

```ruby
# app/docs/users_doc.rb
module UsersDoc
  # we need the DSL, right?
  extend Apipie::DSL::Concern

  api :GET, '/users', 'List users'
  def show
    # Nothing here, it's just a stub
  end
end

# app/controller/users_controller.rb
class UsersController < ApplicationController
  include UsersDoc

  def show
    # Application code goes here
    # and it overrides blank method
    # from the module
  end
end
```

And yes, it works! Let's add resource description

```ruby
module UsersDoc
  extend Apipie::DSL::Concern

  resource_description do
    formats [:json]
    api_versions 'public'
  end
end
```

Which breaks it...

```
Apipie: Can not resolve resource UsersDoc name.
```

Yes, because our resource is `UsersController`. The error happens in the following lines in Apipie source code:

```ruby
    def get_resource_name(klass)
      if klass.class == String
        klass
      elsif @controller_to_resource_id.has_key?(klass)
        @controller_to_resource_id[klass]
      elsif Apipie.configuration.namespaced_resources? && klass.respond_to?(:controller_path)
        return nil if klass == ActionController::Base
        path = klass.controller_path
        path.gsub(version_prefix(klass), "").gsub("/", "-")
      elsif klass.respond_to?(:controller_name)
        return nil if klass == ActionController::Base
        klass.controller_name
      else
        raise "Apipie: Can not resolve resource #{klass} name."
      end
    end
```

Most of this code does not really matter, the thing is that Apipie does not know what to do with module `UsersDoc`. It has no `resource_id`, let's define it.

```ruby
resource_description do
  resource_id 'Users'
  # other dsl
end
```

Now we get an error

```ruby
undefined method `superclass' for UsersDoc:Module
```

But... Module (I mean, an instance of Module class) can't have a parent class. It's a module!

Forget that you are a ruby developer and define a method called `superclass` on your module :)

```ruby
# before calling DSL
def self.superclass
  UsersController
end
```

So, from this moment

```ruby
UsersDoc.superclass == UsersController
# => true
```

Refresh the page with documentation and see that it finally works!
Here is a gist with a full code that wraps ugly code and blank methods:
[full gist](https://gist.github.com/iliabylich/c8032b193405673062e7)

## The power of ruby-based doc

Imagine the following scenario:

I have two versions of API. When I was young and stupid, I implemented the first one (`v1`). And it has an authentication by plain username and password (in headers). Few years later something changed in my mind and I created another one (`v2`) which has token-based authentication.

First API contains 100 actions in 10 controllers, so does the second one, too. To make a documentation I have to copy-paste 2 lines of code 200 times (to put it for every action) or 20 times (to put it to every resource).

Instead of repeated copying the real description of authentication mechanism it would be really nice to have something like

```ruby
# for Api::V1
auth_with :password
# for Api::V2
auth_with :token
```

And describe authentication in a more declarative way. In order to make it we need to add our own method to DSL. Here is what it may look like:

```ruby
module BaseDoc
  # ... code from the gist

  AUTH_METHODS = {
    password: Api::Auth::PasswordDoc,
    token: Api::Auth::TokenDoc
  }

  def auth_with(auth_method)
    mod = AUTH_METHODS[auth_method] or
      raise "Unknown auth strategy #{auth_method}"
    send(:include, mod)
  end
end

# app/docs/api/auth/password_doc.rb
module Api::Auth::PasswordDoc
  def self.included(base)
    base.instance_eval do
      # documentation of password-
      #  based authentication
      header :username, required: true
      header :password, required: true
      error code: 401, desc: 'Unauthorized'
    end
  end
end

# app/docs/api/auth/token_doc.rb
module Api::Auth::TokenDoc
  def self.included(base)
    base.instance_eval do
      # documentation of token-
      #  based authentication
      header 'Token', 'Your API token'
      error code: 401, desc: 'Token is requried'
    end
  end
end
```

And from now we can write

```ruby
module Api::V1::UsersDoc
  extend BaseDoc

  doc_for :show do
    auth_with :password
    # or
    # auth_with :token
  end
end
```

## Ability to define default documentation for all actions in resource

So, we have a `resource_description` and `api` methods, but how can we define common parts for all of our actions? I hate copy-paste driven development, let's write a DSL for this.

We want to have a `defaults` method which takes a block and executes it for each action.

```ruby
module BaseDoc
  # ...

  def defaults(&block)
    @defaults = block
  end

  def doc_for(action_name, &block)
    instance_eval(&block)
    # Only the next line added
    instance_eval(&@defaults) if @defaults
    api_version namespace_name if namespace_name
    define_method(action_name) do
      # ... define it in your controller with the real code
    end
  end
end
```

So, we just store passed block and invoke it. This example is much simpler then the previous one.

# Demo

[URL](https://github.com/iliabylich/apipie-demo)

You can click by statically generated docs (thanks to GitHub pages):

+ [`Private::V1`](http://iliabylich.github.io/apipie-demo/doc/apidoc/private_v1.html)
+ [`Private::V2`](http://iliabylich.github.io/apipie-demo/doc/apidoc/private_v2.html)
+ [`Public`](http://iliabylich.github.io/apipie-demo/doc/apidoc/public.html)

# Conclusion

Apipie is an amazing library and its most significant advantage is that you document ruby using ruby (being a ruby developer), which gives you ability to define custom behaviors and scenarios. I can't even imagine myself writing API docs using `yardoc` (however, I use it to document plain ruby classes).

If you have any bugs (or ideas to implement), please, [create an issue on GitHub](https://github.com/Apipie/apipie-rails/issues), let's make it even better!
