---
layout: post
title:  "HandlerSocket + Ruby"
date:   2015-11-10 00:00:00 +0300
categories: ruby mysql handlersocket databases sql nosql
toc: true
---
# What is HandlerSocket (HS)

+ a plugin for MySQL
+ which allows you to read/write to MySQL
+ and gives you a separate connection to MySQL
+ and doesn't allow you to run SQL queries
+ but allows to run simple CRUD queries *only* using indexes

HandlerSocket query language is very simple (I'd even say it's primitive), but it's much faster than MySQL's one. Though, of course, there are some limitations. Interested?

# Installation

You already have it if you are using Percona Server or MariaDB. If not, install it from [the source](https://github.com/DeNA/HandlerSocket-Plugin-for-MySQL).

To activate the plugin, run:
``` sql
INSTALL PLUGIN handlersocket SONAME 'handlersocket.so';
```

# Configuration

My configuration is the following:
```
# [mysqld] section
# the port number to bind to for read requests
loose_handlersocket_port = 9998
# the port number to bind to for write requests
loose_handlersocket_port_wr = 9999
# the number of worker threads for read requests
loose_handlersocket_threads = 16
# the number of worker threads for write requests
loose_handlersocket_threads_wr = 1
open_files_limit = 65535
```

You can find a detailed documentation of all available configuration options [here](https://github.com/ahiguti/HandlerSocket-Plugin-for-MySQL/blob/master/docs-en/configuration-options.en.txt)

Restart your MySQL server and run:

``` sql
show processlist\G
```

You should see a lot of rows like:
```
           Id: 1
         User: system user
         Host: connecting host
           db: NULL
      Command: Connect
         Time: NULL
        State: handlersocket: mode=rd, 0 conns, 0 active
         Info: NULL
    Rows_sent: 0
Rows_examined: 0
```
which means that HS daemon is up and running.

# Simple queries

You can test it locally using `telnet`:

``` sh
$ telnet 0.0.0.0 9999
Trying 0.0.0.0...
Connected to 0.0.0.0.
Escape character is '^]'.
```

Type `P -> 0 -> your_database -> your_table -> PRIMARY -> id,some_column` (where `->` is Tab). And press Enter. It should return `0 -> 1`.

This protocol looks ugly, but it may save you a lot of network usage. It's very compact, and parsing doesn't require any CPU usage.

# Use cases

If you don't have too much queries per second, probably you don't need HS. It doesn't optimize queries, but you may save some time on request parsing + some network. You may find it interesting if you have a lot of simple queries, like simple `SELECT`'s by primary key.

# Ruby adapter

Here goes my Ruby for HandlerSocket protocol. You can find it [here](https://github.com/iliabylich/handlersocket-ruby).

It has two implementations inside:
+ [Ruby-based](https://github.com/iliabylich/handlersocket-ruby/blob/master/lib/handlersocket/pure.rb)
+ [C-based](https://github.com/iliabylich/handlersocket-ruby/blob/master/ext/handlersocket_ext/handlersocket_ext.c)

Ruby implementation is very slow, it's there mainly to explain the protocol. C-based is quite fast.

# Adapter API

To require a specific implementation, run

``` ruby
# For slow pure Ruby implementation
require 'handlersocket/pure'
# For fast C implementation
require 'handlersocket/ext'
```

To create a connection, run

``` ruby
hs = Handlersocket.new('0.0.0.0', 9999)
```

Both pure Ruby and C implementations have the same API, so the `require` place is the only difference.

To open an index, run

``` ruby
hs.open_index('0', 'hs_test', 'users', 'PRIMARY', ['id', 'email'])
```

To read the data from that index, run

``` ruby
hs.find('0', '=', ['12'], ['100]'])
# Which is equal to
# SELECT id, name FROM hs_test.users WHERE id = 12 LIMIT 100
```

Other commands like auth/insert/update/delete are not there yet. But it's not that difficult to add them, check out [this file](https://github.com/iliabylich/handlersocket-ruby/blob/master/lib/handlersocket.rb#L37), implementation of other methods also takes ~2 lines of code.

# Benchmarks

The most interesting part. To run benchmarks locally, clone the gem repository on [github](https://github.com/iliabylich/handlersocket-ruby) and run `rake benchmark`. It compares Mysql2 gem to Ruby-based and C-based implementations. Here are my results:

```
Calculating -------------------------------------
             pure HS     2.000  i/100ms
              ext HS     3.149k i/100ms
              mysql2   669.000  i/100ms
-------------------------------------------------
             pure HS     49.979  (± 2.0%) i/s -    250.000
              ext HS    110.369M (±15.7%) i/s -    467.793M
              mysql2      5.218M (±16.3%) i/s -     23.869M

Comparison:
              ext HS: 110369437.2 i/s
              mysql2:  5218120.6 i/s - 21.15x slower
             pure HS:       50.0 i/s - 2208314.91x slower
```

I've run these benchmarks on 4 cores server with 4GB ram on Percona server 5.6. On both small and huge (30 millions records) datasets, with enabled and disabled query cache, with a small and a big value of `innodb_buffer_pool_size`.

There's a 20-30x performance difference between Mysql2 and a C-based version of HandlerSocket because:
+ almost no time is taken for request parsing
+ mysql2 is just a way more complex than my gem

And I was really disappointed by performance of Ruby-based implementation. It's 2 millions times slower than C-based. Why? There's a magical number `50.0 i/s`, but I cannot find what does it mean. If you have an answer, please, ping me on Twitter.


# Future plans

The gem *mostly* works, but there are some points that should be refined. Currently when network goes down there's no way to reconnect because HS protocol is stateful. There's no history tracking in HS objects, so if you open an index and then reconnect, you lose your opened index. I'm not sure if it should be implemented on a low level of abstraction in HS gem, probably it's better to make a separated high-level gem for AR that does this job.

# Conclusion

Once again, HandlerSocket saves your time on query parsing, buliding a query plan, it's more compact, but is very limited. If you don't have too many requests, don't even think about using it.

# Links

[HandlerSocket protocol](https://github.com/DeNA/HandlerSocket-Plugin-for-MySQL/blob/master/docs-en/protocol.en.txt)

[Ruby gem for HandlerSocket](https://github.com/iliabylich/handlersocket-ruby)

[HandlerSocket configuration](https://github.com/ahiguti/HandlerSocket-Plugin-for-MySQL/blob/master/docs-en/configuration-options.en.txt)
