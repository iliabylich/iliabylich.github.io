---
layout: post
title:  "Redis cluster. Quick overview"
date:   2015-04-13 00:00:00 +0300
categories: Redis databases
toc: true
comments: true
---
Today I have tested version 3.0.0 of Redis server which includes Redis cluster. Here are some first thoughts about this.

# Setup

Here are my servers:
```sh
# 212.71.252.54  / 192.168.171.141 / node1
# 176.58.103.254 / 192.168.171.142 / node2
# 178.79.153.89  / 192.168.173.227 / node3
```

Local hosts (on each server):
```sh
# local hosts
192.168.171.141 node1
192.168.171.142 node2
192.168.173.227 node3
```

And remote (on my PC):
```sh
# remote hosts
212.71.252.54  node1
176.58.103.254 node2
178.79.153.89  node3
```

First of all, let's download and extract it (on each node).

```sh
mkdir build && cd build
wget http://download.redis.io/releases/redis-3.0.0.tar.gz
tar -xvzf redis-3.0.0.tar.gz
cd redis-3.0.0/
```

Now we can build it
```sh
apt-get install -y make gcc build-essential
make MALLOC=libc # also jemalloc can be used
```

Let's run tests
```sh
apt-get install -y tk8.5 tcl8.5
make test
# a lot of output, should be green
```

Then we can start Redis
```sh
src/redis-server ./redis.conf
```

# Cluster configuration

We need to update our configuration file for each node

```
# redis.conf
bind node1 # for node1
cluster-enabled yes
cluster-config-file nodes-6379.conf
cluster-node-timeout 5000
```

Here's what should displayed:

```
29338:M 13 Apr 21:05:00.214 * No cluster configuration found, I'm a1eec932d923b55e23a5fe6a488ed7a97e27c826
                _._
           _.-``__ ''-._
      _.-``    `.  `_.  ''-._           Redis 3.0.0 (00000000/0) 64 bit
  .-`` .-```.  ```\/    _.,_ ''-._
 (    '      ,       .-`  | `,    )     Running in cluster mode
 |`-._`-...-` __...-.``-._|'` _.-'|     Port: 6379
 |    `-._   `._    /     _.-'    |     PID: 29338
  `-._    `-._  `-./  _.-'    _.-'
 |`-._`-._    `-.__.-'    _.-'_.-'|
 |    `-._`-._        _.-'_.-'    |           http://redis.io
  `-._    `-._`-.__.-'_.-'    _.-'
 |`-._`-._    `-.__.-'    _.-'_.-'|
 |    `-._`-._        _.-'_.-'    |
  `-._    `-._`-.__.-'_.-'    _.-'
      `-._    `-.__.-'    _.-'
          `-._        _.-'
              `-.__.-'

29338:M 13 Apr 21:05:00.246 # Server started, Redis version 3.0.0
29338:M 13 Apr 21:05:00.247 * DB loaded from disk: 0.000 seconds
29338:M 13 Apr 21:05:00.247 * The server is now ready to accept connections on port 6379
```

Quite a lot of noisy debug information, but here is one line that is extremely important:
```
No cluster configuration found, I'm a1eec932d923b55e23a5fe6a488ed7a97e27c826
```

So, our Redis server is running in cluster mode (... repeating same steps on other nodes ...)

# Connecting nodes

Now we have 3 nodes:
```
node1:6379
node2:6379
node3:6379
```

Let's connect them! Redis has a tool for connecting nodes called `redis-trib.rb`. And yes, it's a ruby script and it requires `redis` gem to be installed (not a problem in my case).

```
➜  redis-3.0.0 src/redis-trib.rb
Usage: redis-trib <command> <options> <arguments ...>

  create          host1:port1 ... hostN:portN
                  --replicas <arg>
  check           host:port
  fix             host:port
  reshard         host:port
                  --from <arg>
                  --to <arg>
                  --slots <arg>
                  --yes
  add-node        new_host:new_port existing_host:existing_port
                  --slave
                  --master-id <arg>
  del-node        host:port node_id
  set-timeout     host:port milliseconds
  call            host:port command arg arg .. arg
  import          host:port
                  --from <arg>
  help            (show this help)

For check, fix, reshard, del-node, set-timeout you can specify the host and port of any working node in the cluster.
```

For some reason, this tool does not support host names, we have to pass IP addresses manually.

```sh
➜  redis-3.0.0 src/redis-trib.rb create 192.168.171.141:6379 192.168.171.142:6379 192.168.173.227:6379

>>> Creating cluster
Connecting to node 192.168.171.141:6379: OK
Connecting to node 192.168.171.142:6379: OK
Connecting to node 192.168.173.227:6379: OK
>>> Performing hash slots allocation on 3 nodes...
Using 3 masters:
192.168.171.141:6379
192.168.171.142:6379
192.168.173.227:6379
M: 78a5bbdcd545848be8a66126a71dc69dd6d23bc4 192.168.171.141:6379
   slots:0-5460 (5461 slots) master
M: 1f6ed2478b461539f76b0b627de2e1b8565df719 192.168.171.142:6379
   slots:5461-10922 (5462 slots) master
M: 7a092b06c8c75e98176b7612e74d2e89e8b3eda7 192.168.173.227:6379
   slots:10923-16383 (5461 slots) master
Can I set the above configuration? (type 'yes' to accept):
```

Looks beautiful! Each node is responsible for 1/3 of data. Typing 'yes'...

```
>>> Nodes configuration updated
>>> Assign a different config epoch to each node
>>> Sending CLUSTER MEET messages to join the cluster
Waiting for the cluster to join.
>>> Performing Cluster Check (using node 127.0.0.1:7001)
M: 78a5bbdcd545848be8a66126a71dc69dd6d23bc4 192.168.171.141:6379
   slots:0-5460 (5461 slots) master
M: 1f6ed2478b461539f76b0b627de2e1b8565df719 192.168.171.142:6379
   slots:5461-10922 (5462 slots) master
M: 7a092b06c8c75e98176b7612e74d2e89e8b3eda7 192.168.173.227:6379
   slots:10923-16383 (5461 slots) master
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
```

That's it!

# Testing our cluster

Here is how to check the status of the cluster:

```
➜  redis-3.0.0 src/redis-cli -h node2 cluster nodes
7a092b06c8c75e98176b7612e74d2e89e8b3eda7 node1:6379 master - 0 1428949630273 3 connected 10923-16383
78a5bbdcd545848be8a66126a71dc69dd6d23bc4 node2:6379 myself,master - 0 0 1 connected 0-5460
1f6ed2478b461539f76b0b627de2e1b8565df719 node3:6379 master - 0 1428949629272 2 connected 5461-10922
```

Every single node knows about others, so the previous command can be executed on any node.

# Benchmarks

Let's setup [`redis-rb-cluster`](https://github.com/antirez/redis-rb-cluster)

```
➜  build wget https://github.com/antirez/redis-rb-cluster/archive/master.zip
➜  build unzip master.zip
➜  build cd redis-rb-cluster-master
```

We have the file `example.rb` which is pretty boring itself. It just writes random keys into our cluster and prints them:

```
➜  redis-rb-cluster-master ruby example.rb
1
2
3
...
```

Another example is more interesting.

```
➜  redis-rb-cluster-master ruby consistency-test.rb node1 6379
850 R (0 err) | 850 W (0 err) |
4682 R (0 err) | 4682 W (0 err) |
8490 R (0 err) | 8490 W (0 err) |
12196 R (0 err) | 12196 W (0 err) |
15785 R (0 err) | 15785 W (0 err) |
```

This tool writes a huge amount of data to Redis and check whether previously written data is still there.

# Testing the failover

This chapter is something that I still don't understand. I have tried to reproduce it many times and every time I have the same result.

Run `consistency-test.rb` again and kill any other node.

```
➜  redis-3.0.0 src/redis-cli -h node2 debug segfault
```

And here is the output that I'm getting every time:

```
70273 R (0 err) | 70273 W (0 err) |
Reading: Too many Cluster redirections? (last error: MOVED 9515 127.0.0.1:7002)
Writing: Too many Cluster redirections? (last error: MOVED 9515 127.0.0.1:7002)
72378 R (1 err) | 72378 W (1 err) |
Reading: Too many Cluster redirections? (last error: MOVED 9650 127.0.0.1:7002)
Writing: Too many Cluster redirections? (last error: MOVED 9650 127.0.0.1:7002)
72379 R (2 err) | 72379 W (2 err) |
Reading: Too many Cluster redirections? (last error: MOVED 5797 127.0.0.1:7002)
Writing: Too many Cluster redirections? (last error: MOVED 5797 127.0.0.1:7002)
72380 R (3 err) | 72380 W (3 err) |
Reading: Too many Cluster redirections? (last error: MOVED 9772 127.0.0.1:7002)
Writing: Too many Cluster redirections? (last error: MOVED 9772 127.0.0.1:7002)
72384 R (4 err) | 72384 W (4 err) |
Reading: Too many Cluster redirections? (last error: MOVED 10245 127.0.0.1:7002)
Writing: Too many Cluster redirections? (last error: MOVED 10245 127.0.0.1:7002)
72385 R (5 err) | 72385 W (5 err) |
Reading: Too many Cluster redirections? (last error: MOVED 7376 127.0.0.1:7002)
Writing: Too many Cluster redirections? (last error: MOVED 7376 127.0.0.1:7002)
72385 R (6 err) | 72385 W (6 err) |
Reading: Too many Cluster redirections? (last error: MOVED 6781 127.0.0.1:7002)
Writing: Too many Cluster redirections? (last error: MOVED 6781 127.0.0.1:7002)
72396 R (7 err) | 72396 W (7 err) |
Reading: Too many Cluster redirections? (last error: MOVED 10275 127.0.0.1:7002)
Writing: Too many Cluster redirections? (last error: MOVED 10275 127.0.0.1:7002)
72401 R (8 err) | 72401 W (8 err) |
Reading: Too many Cluster redirections? (last error: MOVED 8639 127.0.0.1:7002)
Writing: Too many Cluster redirections? (last error: MOVED 8639 127.0.0.1:7002)
72402 R (9 err) | 72402 W (9 err) |
Reading: Too many Cluster redirections? (last error: MOVED 8173 127.0.0.1:7002)
Writing: Too many Cluster redirections? (last error: MOVED 8173 127.0.0.1:7002)
72402 R (10 err) | 72402 W (10 err) |
Reading: Too many Cluster redirections? (last error: MOVED 9525 127.0.0.1:7002)
Writing: Too many Cluster redirections? (last error: MOVED 9525 127.0.0.1:7002)
72403 R (11 err) | 72403 W (11 err) |
Reading: Too many Cluster redirections? (last error: MOVED 9346 127.0.0.1:7002)
Writing: Too many Cluster redirections? (last error: MOVED 9346 127.0.0.1:7002)
72406 R (12 err) | 72406 W (12 err) |
Reading: Too many Cluster redirections? (last error: MOVED 6391 127.0.0.1:7002)
Writing: Too many Cluster redirections? (last error: MOVED 6391 127.0.0.1:7002)
72411 R (13 err) | 72411 W (13 err) |
Reading: Too many Cluster redirections? (last error: MOVED 6353 127.0.0.1:7002)
Writing: Too many Cluster redirections? (last error: MOVED 6353 127.0.0.1:7002)
72413 R (14 err) | 72413 W (14 err) |
Reading: Too many Cluster redirections? (last error: MOVED 10245 127.0.0.1:7002)
Writing: Too many Cluster redirections? (last error: MOVED 10245 127.0.0.1:7002)
72418 R (15 err) | 72418 W (15 err) |
Reading: Too many Cluster redirections? (last error: MOVED 6438 127.0.0.1:7002)
Writing: Too many Cluster redirections? (last error: MOVED 6438 127.0.0.1:7002)
72422 R (16 err) | 72422 W (16 err) |
Reading: Too many Cluster redirections? (last error: MOVED 6826 127.0.0.1:7002)
Writing: Too many Cluster redirections? (last error: MOVED 6826 127.0.0.1:7002)
72423 R (17 err) | 72423 W (17 err) |
Reading: Too many Cluster redirections? (last error: MOVED 9713 127.0.0.1:7002)
Writing: CLUSTERDOWN The cluster is down
Reading: CLUSTERDOWN The cluster is down
Reading: CLUSTERDOWN The cluster is down
Writing: CLUSTERDOWN The cluster is down
72423 R (295 err) | 72423 W (295 err) |
72423 R (2219 err) | 72423 W (2219 err) |
Reading: CLUSTERDOWN The cluster is down
Writing: CLUSTERDOWN The cluster is down
Reading: CLUSTERDOWN The cluster is down
Writing: CLUSTERDOWN The cluster is down
72423 R (4186 err) | 72423 W (4186 err) |
Reading: CLUSTERDOWN The cluster is down
Writing: CLUSTERDOWN The cluster is down
72423 R (6190 err) | 72423 W (6190 err) |
Writing: CLUSTERDOWN The cluster is down
72423 R (8207 err) | 72423 W (8207 err) |
Reading: CLUSTERDOWN The cluster is down
Reading: CLUSTERDOWN The cluster is down
Writing: CLUSTERDOWN The cluster is down
```

As you can see, the cluster goes down and errors amount quickly increases. After all this steps I'm getting broken cluster (and each separated node is broken, too)

```
➜  redis-3.0.0 src/redis-cli -h node3 ping
PONG
➜  redis-3.0.0 src/redis-cli -h node3 get 'test'
(error) CLUSTERDOWN The cluster is down
```

The cluster goes up after booting the first node manually

```
# booting node2 manually...
➜  redis-3.0.0  src/redis-cli -h  get qwe
(error) MOVED 757 127.0.0.1:7001
➜  redis-3.0.0  src/redis-cli -p 7001 get qwe
(nil)
```

I hope this will be fixed soon.

# Conclusion

According to [the repository](https://github.com/antirez/redis-rb-cluster#redis-rb-cluster) there are still a lot of things that need to be done before using Redis Cluster in production, but I would like to say Thanks to all contributors of Redis, `redis-rb` and `redis-rb-cluster`. Good job and looking forward to use in the real-world!

# Links

+ [official tutorial](http://redis.io/topics/cluster-tutorial)

+ [cluster specification](http://redis.io/topics/cluster-spec)

+ [changelog](https://raw.githubusercontent.com/antirez/redis/3.0/00-RELEASENOTES)
