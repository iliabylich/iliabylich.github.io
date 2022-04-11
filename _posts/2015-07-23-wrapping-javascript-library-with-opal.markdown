---
layout: post
title:  "Wrapping JavaScript library with Opal"
date:   2015-07-23 00:00:00 +0300
categories: ruby opal
toc: true
comments: true
---
# Introduction

The task that is solved here is not real, but it's still a good example of (probably?) real work with Opal. I could choose some complex enough JavaScript library and write a simple wrapper using Opal, but there's no fun. Instead, let's write a wrapper for existing rich client-side application (it may show you how to wrap your existing application logic). Well, wrapper for something like a client-side scheduler may sound boring, so I have chosen a JavaScript-based browser game called [`BrowserQuest`](http://browserquest.mozilla.org) [written](https://github.com/mozilla/BrowserQuest) by Mozilla, and I'll show you how to write a bot for it using Opal.

# Opal

There are so many posts about [Opal](https://github.com/opal/opal), so I'm just going to say "it's a Ruby to JavaScript" compiler, that's enough.

# Environment

First of all, we need something that runs the game and injects a bot into the page. I, personally, while writing integration tests (this is the place, where we usually face to web drivers), prefer PhantomJS, but it's headless, so you can't enjoy watching how your bot works. We have to use something like Capybara + Selenium:
```ruby
# Gemfile
gem 'capybara'
gem 'selenium-webdriver'

# runner.rb
require 'capybara'

Capybara.register_driver :selenium do |app|
  Capybara::Selenium::Driver.new(app, :browser => :firefox)
end

Capybara.javascript_driver = :selenium
Capybara.default_driver = :selenium
Capybara.run_server = false
```

So, the script registers a driver, specifies its browser (Firefox), makes it default and runs Capybara in browser mode (i.e. without own server in the background)

# Opening the page

Dead simple:

```ruby
Capybara.current_session.visit('http://browserquest.mozilla.org')
# or in object-oriented style
class Game
  include Capybara::DSL

  def initialize(url)
    # all methods of
    # Capybara.current_session
    # are available here
    visit(url)
  end

  def play
    # logic of the bot
  end
end

Game.new('http://browserquest.mozilla.org/').play
```

Now it runs a Firefox and opens the page with the game.

# Compiling Opal

So, there are two ways to compile Ruby into JavaScript:
+ compiling Ruby code as a string to JavaScript string
+ compiling specified Ruby file to JavaScript string

To compile a file with Ruby, run:
```ruby
Opal.append_path('some/path/to/dir/with/your/files')
Opal::Builder.build('relative/path/from/that/dir/to/you/file')
```

To compile a string with ruby:
```ruby
Opal.compile("plain ruby code")
```

The first way is what we really need:
1. create a directory with all opal files
2. add it to Opal's load path
3. (all `require` commands work as in MRI)
4. create a file called `app.rb` that `require`-s other files
5. embed `app.rb` to the page

# Fetching the data from the game

[This](https://github.com/mozilla/BrowserQuest/blob/master/client/js/main.js#L7) is the place where the main `App` class is created. But! It's defined in anonymous function, so this variable is not available outside the context.

This game uses `CommonJS` to load files. This library caches all previously required files and instantly returns cached result on the seconds `require`.

We can use it:
1. require `app` file.
2. wrap any of its methods with some logic that stores current app instance globally and then call `super`

I have chosen a method called `start`:
```ruby
# opal/bot.rb
module Patch
  def self.apply
    %x{
      var app = require('app');
      oldStart = app.prototype.start;
      app.prototype.start = function(username) {
        window.currentApplication = this;
        oldStart.apply(this, arguments);
      }
    }
  end
end

Patch.apply
```

Some explanations:
+ `%x{js code}` just passes provided JavaScript into compiled version (i.e. runs it without any translation)
+ After compiling this file we have access to a variable `currentApplication` that contains an instance of `App` class

# Starting the game

As you can see, to start the game you need to:
1. type your player name
2. wait for 'Play' button to activate (become red)
3. press 'Play' button
4. wait until all assets will be loaded
5. close instructions that the game opens for any new player

After all of these steps the game will be ready, but the point here is that most of the steps are asynchronous. You can't just type your name and **immediately** press 'Play' button (and you can't press 'Play' without waiting for loading)

This is the place where promises shine. Opal has its own standard library that
+ ships with Opal's code, so it's already in the page
+ has a class called `Promise` that acts pretty much like a `jQuery.Deferred()`

```ruby
require 'promise'

promise = Promise.new
promise.then { puts 'Done' }
promise.fail { puts 'Fail' }
promise.resolve
# => 'Done' (in JavaScript console)
# or
promise.reject
# => 'Fail'
```

(`Promise` is like an object that is a combination of `callback`-s and `errback`-s, but you don't invoke callbacks manually, instead you just switch the state of your promise-object and it automatically triggers `callbacks`/`errbacks`)

Here is a little helper module that saves our time:
```ruby
module Utils
  def wait_for(promise = Promise.new, &waiting)
    result = waiting.call
    if !!result
      promise.resolve
    else
      after 0.1 do
        wait_for(promise, &waiting)
      end
    end
    promise
  end
end

# and usage

class MyClass
  include Utils

  def call
    some_async_method_without_ability_to_pass_callback
    wait_for do
      method_called && result_is_success
    end
  end
end
```

`wait_for` method takes a promise (which is a blank promise object by default) and a block (which will be converted to JavaScript function). It calls the block and `resolve`-s the promise if it is returned true. If not, it calls itself again after 100 ms (`after` = `setTimeout`) with **the same promise object**

To type player's name we should run:
```ruby
# I'm using opal-jquery here
# I think it does not require any explanation
input = Element.find('#nameinput')
input.value = @player_name
wait_for do
  Element.find('.play.button.disabled').length == 0
end.then do
  # Button is ready, we can click it here
end
```

To click the button, run:
```ruby
button = Element.find('#createcharacter .play.button div')
button.trigger(:click)
wait_for do
  Element.find('#instructions').has_class?('active')
end.then do
  # The game is ready here
  # And it shows us instructions
  # We are almost ready to start the game
end
```

To close instructions, run:
```ruby
Element.find('#instructions').trigger(:click)
```

[And put everything together](https://github.com/iliabylich/opal-browserquest-bot/blob/master/bot/opal/commands/start_game.rb)

To run this command, call
```ruby
StartGame.new('Bot player').invoke.then do
  alert("I'm in the game")
end
```

# Time to wrap the code of the game

As an enter point we are going to use global JavaScript variable `currentApplication`. It has a property `game` (that, unexpectedly, returns instance of `Game` class). `game` has a `player` property (instance of `Player`) and `entities` property which is an object containing all entities on the map, their types and coordinates. You can easily find their JavaScript implementations in the GitHub repository of the game.

So, our main objects are:
+ `currentApplication`
+ `currentApplication.game`
+ `currentApplication.game.player`
+ `currentApplication.game.entities`

First class for wrapping is definitely an `App`:
```ruby
class Application
  include Native

  def self.current
    self.new(`currentApplication`)
  end

  def initialize(native)
    @native = native
  end

  alias_native :game, :game, as: Game

  def to_n
    @native
  end
end
```
So, we have a class called `Application` that wraps some native JavaScript object and has a ruby-method `game` that calls JavaScript-method `game` and wraps it using `Game` class (see below). As a bonus, we have a class-method `current` that returns wrapped `currentApplication`.

The next class is a `Game`:
```ruby
class Game
  include Native

  def self.current
    Application.current.game
  end

  def initialize(native)
    @native = native
  end

  alias_native :player, :player, as: Player
  alias_native :say

  def to_n
    @native
  end

  def entities
    res = []
    native_entities = `currentApplication.game.entities`
    Native::Hash.new(native_entities).each do |e_id, e|
      res << Native(e)
    end
    EntityCollection.new(res)
  end
end
```

And again, this class can wrap any JavaScript game object, has methods `player`, `say` and `entities` (`EntityCollection` is our next class to implement).

(we can test method `say` write now, just put `Game.current.say('Hello')` to the block where the game is ready and start chatting with other players)

# Entities

The game provides a global JavaScript object `Types` with all mobs/items/armors/weapons information, it allows to identify unknown entity, compare armors and weapons by rank. Basically, it provides everything for writing a bot logic.

To convert it to Ruby, use ```Types = Native(`Types`)```and use this object in the Ruby world!

Here is my definition of `Entity` class:
```ruby
class Entity
  include Native

  def initialize(native)
    @native = native
  end

  def to_n
    @native
  end

  alias_native :kind

  def player?
    Types.isPlayer(kind)
  end

  # some other methods
  # like mob?
  # or heal?

  def weapon_rank
    Types.getWeaponRank(kind)
  end

  def armor_rank
    Types.getArmorRank(kind)
  end
end
```

Well, this class can wrap player/mob/armor/weapon/healing, but this is only a value-object, we still need to implement our collection-object `EntityCollection`:

```ruby
class EntityCollection
  def initialize(native_entities)
    @entities = native_entities.map do |native_entity|
      Entity.new(native_entity.to_n)
    end
  end

  def players
    entities = @entities.select(&:player?)
    EntityCollection.new(entities)
  end

  # similar methods like
  # mobs/weapons/armors/healings
  # are omitted and are just like 'players' method
end
```

# Player class

(quickly and without any explanation):

```ruby
class Player
  include Utils
  include Native

  def self.current
    Game.current.player
  end

  def initialize(native)
    @native = native
  end

  alias_native :distance_to, :getDistanceToEntity
  alias_native :moving?, :isMoving
  alias_native :attacking?, :isAttacking
  alias_native :hp, :hitPoints
  alias_native :max_hp, :maxHitPoints

  def full_hp?
    hp == max_hp
  end

  alias_native :weapon_name, :getWeaponName
  alias_native :armor_name, :getArmorName
end
```

# Writing the code of the bot

It's not as difficult once we have all these classes prepared. The algorithm of  farming is like:
1. Find a closest mob and kill it
2. Find a closest weapon (and pick up if it's enough close)
3. Find a closest armor (and pick up if it's enough close)
4. Find a closest healing (and pick up if it's enough close)
5. `GOTO` 1

All of these steps will be our methods, and all of them **must** be asynchronous.

Just one method is missing here (`closest`):
```ruby
class EntityCollection
  def by_distance
    entities = @entities.sort_by do |entity|
      Player.current.distance_to(entity)
    end
    EntityCollection.new(entities)
  end

  def first
    @entities.first
  end

  def last
    @entities.last
  end

  def closest
    by_distance.first
  end
end
```

## Killing a mob

```ruby
def kill_mob
  closest_mob = Game.current.entities.mobs.closest
  `#{Game.current.to_n}.makePlayerAttack(#{closest_mob.to_n})`
  # TODO: move this method to the game class
  # using alias_native :)
end
```

## Picking up an abstract item

```ruby
def pick_up
  `#{Game.current.to_n}.makePlayerGoToItem(#{item.to_n});`
end
```

## Picking up a weapon

```ruby
def get_armor
  current_weapon_name = Player.current.weapon_name
  weapons = Game.current.entities.weapons
  closest_weapon = weapons.better_than(current_weapon_name).closest
  if closest_weapon.nil?
    # No weapon, probably next time
    return
  end
  if Player.current.distance_to(closest_weapon) > 100
    # Weapon is too far away, next time
    return
  end
  pick_up(closest_weapon)
end
```

## Picking up an armor

Just like a previous snippet, but with `armors` instead of `weapons`

## What's missing?

All of these steps should return promises, every single method written below should wait for player to stop moving and attacking. To make this we need some common method like:
```ruby
def wait_until_inactive
  promise = Promise.new
  wait_for do
    !Player.current.moving? && !Player.current.attacking?
  end.then do
    # Wait 1 more second to continue
    after 1 do
      promise.resolve
    end
  end
  promise
end
```

And put it to the end of each action method.

# Wrapping a wrapper

We need the main method `farm`, right?

```ruby
def farm
  kill_mob.then do
    get_weapon.then do
      get_armor.then do
        heal.then do
          farm
        end
      end
    end
  end
end
```

This, is probably the thing that I have personally learned during writing this article. Even if you think in Ruby, you still have to deal with asynchronous components like callbacks/promises. When you need to make an HTTP request in Ruby, you just get you favorite HTTP adapter (mine is `RestClient`), send a request and your interpreter waits for response. In JavaScript you have to process response in some callback, because you can't just stop your interpreter (you know, it blocks UI).

# Conclusion

As for me, the main thing Opal gives to you is some ability to think in terms of Ruby classes/modules/inheritance system. But it does not let you completely escape from JavaScript ecosystem (no callbacks? - block is a callback). I would say, most of Opal functionality related to Ruby classes can be replaced with, for example, [`JsClass`](http://jsclass.jcoglan.com/) library (which is really wonderful). Opal allows you to compile **existing** Ruby libraries to JavaScript and use them on the client - this is probably the main feature. Some day significant amount of Ruby libraries will be ported to client-side and probably some day we will think in terms of Ruby even on the client.
