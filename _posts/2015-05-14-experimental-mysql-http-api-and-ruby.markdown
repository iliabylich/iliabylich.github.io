---
layout: post
title:  "Experimental MySQL HTTP API and Ruby"
date:   2015-05-14 00:00:00 +0300
categories: ruby MySQL databases
toc: true
comments: true
---
Yes, MySQL has an HTTP API which is:
1. an experimental feature
2. it ships as a native plugin only for 5.7 version (http://labs.mysql.com/?id=3)

Basically, it allows you to work with your database in the following ways:
1. as an SQL endpoint
2. as a CRUD endpoint
3. as a JSON document endpoint

## SQL endpoint

It gives an ability to run queries like

```
GET http://host:port/sql/:database/:query
```

For example

```
GET http://localhost:8080/sql/testdb/SELECT+1
```

is a synonym of SQL's `SELECT 1`.

## CRUD endpoint

Interface is:

```
GET http://host:port/crud/:database/:table/:id
```

And

```
GET http://localhost:8080/crud/test_db/test_table/101'
```

produces:

```
SELECT * FROM `test_db`.`test_table` WHERE `test_table`.`id` = 101
```

## JSON document endpoint

Basically, with this endpoint your table has only 3 fields:
```
mysql> describe test_db.test_table;
+--------+---------------------+------+-----+---------+-------+
| Field  | Type                | Null | Key | Default | Extra |
+--------+---------------------+------+-----+---------+-------+
| _id    | varchar(36)         | NO   | PRI | NULL    |       |
| _rev   | bigint(20) unsigned | NO   | PRI | 0       |       |
| _extra | blob                | NO   |     | NULL    |       |
+--------+---------------------+------+-----+---------+-------+
```

All the data is stored in `_extra` column, `_id` is a regular document id, `_rev` is a revision of your document. There is no schema for API perspective and the documents are versioned. As for me it looks prety much like CouchDB (of course, without querying the entire documents data :smile:)

# MySQL 5.7 installation

It's dead simple (until you want to install it from source):
+ Download package from mysql site

```
wget http://downloads.mysql.com/snapshots/pb/mysql-5.7.5-labs-http/mysql-5.7.5-labs-http-linux-glibc2.5-x86_64.tar.gz
```
+ Extract it

```
tar -zxvf mysql-5.7.5-labs-http-linux-glibc2.5-x86_64.tar.gz
```
+ Create mysql:mysql user:group

```
groupadd mysql
useradd -r -g mysql mysql
```
+ Link extracted dir

```
ln -s /root/mysql-5.7.5-labs-http-linux-glibc2.5-x86_64 /usr/local/mysql
cd /usr/local/mysql
```
+ Run `mysql_install_db`

```
bin/mysql_install_db --user=mysql --datadir=/var/lib/mysql
```
+ Run mysql server

```
bin/mysqld_safe --user=mysql --datadir=/var/lib/mysql &
```
+ Change root password

```
cat ~/.mysql_secret
# Here goes your root password!
mysql -uroot -p
# <Paste your root password from the file>
mysql> SET PASSWORD=PASSWORD('root'); # Yes, I'm mad
mysql> exit
```
+ Create configs and init script

```
cp support-files/my-default.cnf /etc/my.cnf
cp support-files/mysql.server /etc/init.d/mysql
```
And change `basedir` and `datadir` in `/etc/init.d/mysql`
```
# In my case
basedir=/usr/local/mysql
datadir=/var/lib/mysql
```
+ Run mysql server again with configs

```
/etc/init.d/mysql start
```

# HTTP API plugin installation

It's already included into our MySQL installation:

```
ls -l /usr/local/mysql/lib/plugin/libmyhttp.so
```

But it's disabled by default, to enable it run
```
mysql> INSTALL PLUGIN myhttp SONAME 'libmyhttp.so';
...
mysql> SHOW PLUGINS;
```

Once plugin is enabled, some MySQL variables become accessible:
```
mysql> show variables like 'myhttp_%';
+----------------------------------+------------------------------+
| Variable_name                    | Value                        |
+----------------------------------+------------------------------+
| myhttp_basic_auth_user_name      | basic_auth_user              |
| myhttp_basic_auth_user_passwd    | basic_auth_passwd            |
| myhttp_crud_url_prefix           | /crud/                       |
| myhttp_default_db                | myhttp                       |
| myhttp_default_mysql_user_host   | 127.0.0.1                      |
| myhttp_default_mysql_user_name   | username                     |
| myhttp_default_mysql_user_passwd | userpass                     |
| myhttp_document_url_prefix       | /doc/                        |
| myhttp_http_enabled              | ON                           |
| myhttp_http_port                 | 8080                         |
| myhttp_https_enabled             | OFF                          |
| myhttp_https_port                | 8081                         |
| myhttp_https_ssl_key_file        | lib/plugin/myhttp_sslkey.pem |
| myhttp_sql_url_prefix            | /sql/                        |
+----------------------------------+------------------------------+
```

1. **myhttp_basic_auth_user_name** - username that should be used for basic http authentication
2. **myhttp_basic_auth_user_passwd** - password that should be used for basic http authentication
3. **myhttp_crud_url_prefix** - prefix of CRUD endpoint
4. **myhttp_default_db** - database that will be used if no database specified in request
5. **myhttp_default_mysql_user_host**  - MySQL user that will be used for performing query provided with HTTP request
6. **myhttp_default_mysql_user_passwd** - password of this MySQL user
7. **myhttp_document_url_prefix** - prefix of JSON document endpoint
8. **myhttp_http_enabled** - boolean setting to enable/disable non-SSL HTTP API
9. **myhttp_http_port** - port of non-SSL HTTP API
10. **myhttp_https_enabled** - boolean setting to enable/disable SSL HTTP API (yes, it's available)
11. **myhttp_https_port** - port of SSL HTTP API
12. **myhttp_https_ssl_key_file** - path to SSL certificate for SSL API
13. **myhttp_sql_url_prefix** - prefix of SQL endpoint

Let's try it!

```
$ curl -v localhost:8080
> GET / HTTP/1.1
> User-Agent: curl/7.35.0
> Host: 127.0.0.1:8080
> Accept: */*
>
< HTTP/1.1 404 Not Found
< Cache-control: must-revalidate
< Connection: Keep-Alive
< Pragma: no-cache
* Server MyHTTP 1.0.0-alpha is not blacklisted
< Server: MyHTTP 1.0.0-alpha
< Content-Length: 36
<
{"error":404, "message":"Not Found"}
```

Well, at least it returns something (be patient, this feature is experimental)

# HTTP API plugin configuration

First of all, we need to create MySQL user and use his credentials in config file.

```
mysql> CREATE USER 'username'@'%' IDENTIFIED BY 'userpass';
mysql> GRANT ALL PRIVILEGES ON * . * TO 'username'@'%';
mysql> FLUSH PRIVILEGES;
```
Then we can change plugin's settings in `/etc/my.cnf` to something like:

```
[mysqld]
myhttp_default_db = myhttp
myhttp_default_mysql_user_name = username
myhttp_default_mysql_user_passwd = userpass
myhttp_default_mysql_user_host = 127.0.0.1
myhttp_basic_auth_user_name = basic_auth_user
myhttp_basic_auth_user_passwd = basic_auth_passwd
```

Here is how it should look using curl:

```
curl --user basic_auth_user:basic_auth_passwd --url http://localhost:8080/crud/myhttp/simple/1
 {"id":"1","col_a":"Hello"}
```

# Connecting with Ruby

First of all this is an HTTP API, so it's available for both server-side and client-side.

## Playing with CRUD endpoint

If we need to work with MySQL over HTTP API (let's imagine for a second that this is the only available way to fetch the data)
we need something like [ActiveResource](https://github.com/rails/activeresource).

ActiveResourse is a gem for building ActiveRecord-like classes that actually fetch data over HTTP API.

Install it:

```
$ gem install activeresource
```
And use:

``` ruby
require 'activeresource'

# We should have a table `users` (yes, Rails convention)
class User < ActiveResourse::Base
  self.site = 'http://localhost:8080/crud'
  self.user = 'basic_auth_user'
  self.password = 'basic_auth_passwd'
end

user = User.find(1)
user.first_name
# => 'Sherlock'
user.last_name
# => 'Holmes'
user.first_name = 'John'

User.create(first_name: 'John', last_name: 'Watson)
# => ActiveResource::MethodNotAllowed: Failed.  Response code = 405.  Response message = Method Not Allowed.
```

By default ActiveResource sends POST request to create a new record. However MySQL HTTP API has some other standards and it uses PUT for creating new record. Moreover, its required to specify primary key (`id`) of the record you want to insert (even if `id` has autoincrement)

ActiveResource doesn't allow us to specify HTTP verbs for its API calls
[source](https://github.com/rails/activeresource/blob/master/lib/active_resource/base.rb#L1435)
Without this we can't even create a single record in the database!

Let's check for alternatives. [`Her`](https://github.com/remiprev/her) is a popular (according to stars on GitHub it's even more popular then ActiveResource) alternative with a nice syntax sugar. Here is an example:

```
$ gem install her
```

``` ruby
require 'her'

Her::API.setup url: "http://localhost:8080/crud/testdb/" do |c|
  # Basic auth
  c.use Faraday::Request::BasicAuthentication,
    "basic_auth_user", "basic_auth_passwd"

  # Encode request
  c.use Faraday::Request::UrlEncoded

  # Parse response
  c.use Her::Middleware::DefaultParseJSON

  # Adapter
  c.use Faraday::Adapter::NetHttp
end

class User
  include Her::Model
  method_for :create, :put
end

user = User.find(1)
# => instance of user
user.first_name = 'another-name'
user.save
# => false
```

No errors messages, just 400 Bad Request.
The thing is that body should be JSON-encoded, here is my middleware that encodes it:

``` ruby
class MySqlHttpApiFormatter < Faraday::Middleware
  def call(env)
    if env['body'] # this middleware calls also for 'get' requests
      env['body'].delete(:id) # id is already in the url
      env['body'] = env['body'].to_json
    end
    @app.call(env)
  end
end

Her::API.setup ... do |c|
  c.use MySqlHttpApiFormatter
end

user = User.create(first_name: 'Test', last_name: 'User', id: 25)
# => instance of user
```
And finally it works! The only thing that I personally don't like in design of this API is that both `INSERT` and `UPDATE` have to be called using PUT because it produces query:

```
REPLACE INTO db.table SET ..., pk = ...
```
That's why we have to pass id, `create` and `update` actions are combined into single `replace` (and I don't see any explanation for that).

Theoretically, this API can be used for both client-side and server-side (but it's important to make sure that you can allow users to change the data in your database)

## Playing with SQL endpoint

Honestly saying nothing interesting.
+ This API can't be used for client-side applications because of potential SQL injections (yes, you can restrict user from deleting/updating data and grant only `Select_priv` privilege, but it still looks dangerous). Even if you are 100% sure that you are completely secured from any application-related vulnerabilities, CRUD API still looks better then passing raw SQL queries over HTTP.
+ Yes, it's possible to use it on the server, but for what? mysql2 gem is still much faster (and can be easily combined with ActiveRecord)... nothing to discuss.

## Playing with JSON document endpoint

First of all, it looks quite strange comparing with regular MySQL tables. The database that is compatible with this API **must** be created using the same API. And as I wrote before, it creates 3 fields:
+ `_id` - some unique identifier
+ `_rev` - revision of document (I can't call it 'record')
+ `_ext` - serialized mapping `attribute_name => value`

Here is how it looks it MySQL:
```
mysql> select * from json_types\G
*************************** 1. row ***************************
   _id: 1
  _rev: 1
_extra: {"json_null": null, "json_string": "string", "json_number": 123.456, "json_bool": true}
1 row in set (0.00 sec)
```

CouchDB has similar interface (HTTP API, schema-less documents, MVCC) but it provides one core feature of all databases: ability to run queries. You can't run any query when your data is serialized (no, I don't know what is `where _rev LIKE '%"some":json_structure%'"`). Moreover MVCC for this table is not real :) It stores only latest revision.

From my expirience when you need to store schema-less documents you have two options:
1. Use real schema-less storage (yes, like Mongo)
2. Serialize fields **that need to be serialized** into single column (and Rails provides functionality for this out-of-box: `ActiveRecord::Base.serialize`) and query the database using unserialized column.

JSON document API can be useful when you already have MySQL and don't want to setup any other databases to store tons of unstructured data. Yes, sounds awful, but looks like a true.

# Conclusion

HTTP API is a tool, and it solves a lot of possible issues. It's not released yet (and I suppose it will not be included into default MySQL 5.7 installation), but I'm waiting for it so much. Today, when SPAs are so popular and Rails application may have only HTTP API, it can simplify our server code. Let's give it a chance!
