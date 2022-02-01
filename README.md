# pgmoon

![test](https://github.com/leafo/pgmoon/workflows/test/badge.svg)

> **Note:** Have you updated from an older version of OpenResty? You must update to
> pgmoon 1.12 or above, due to a change in Lua pattern compatibility to avoid incorrect 
> results from queries that return affected rows.

**pgmoon** is a PostgreSQL client library written in pure Lua (MoonScript).

**pgmoon** was originally designed for use in [OpenResty][] to take advantage
of the [cosocket
api](https://github.com/openresty/lua-nginx-module#ngxsockettcp) to provide
asynchronous queries but it also works in the regular any Lua environment where
[LuaSocket][] or [cqueues][] is available.

It's a perfect candidate for running your queries both inside OpenResty's
environment and on the command line (eg. tests) in web frameworks like [Lapis][].

## Install

```bash
$ luarocks install pgmoon
```

<details>
<summary>Using <a href="https://opm.openresty.org/">OpenResty's OPM</a></summary>

```bash
$ opm get leafo/pgmoon
```

</details>


### Dependencies

pgmoon supports a wide range of environments and libraries, so it may be
necessary to install additional dependencies depending on how you intend to
communicate with the database:

> **Tip:** If you're using OpenResty then no additional dependencies are needed
> (generally, a crypto library may be necessary for some authentication
> methods)

A socket implementation **is required** to use pgmoon, depending on the
environment you can chose one:

* [OpenResty][] &mdash; The built in socket is used, no additional dependencies necessary
* [LuaSocket][] &mdash; `luarocks install luasocket`
* [cqueues][] &mdash; `luarocks install cqueues`

If you're on PUC Lua 5.1 or 5.2 then you will need a bit libray (not needed for LuaJIT):

```bash
$ luarocks install luabitop
```

If you want to use JSON types you will need lua-cjson

```bash
$ luarocks install lua-cjson
```

SSL connections may require an additional dependency:

* OpenResty &mdash; No additional dependencies required
* LuaSocket &mdash; `luarocks install luasec`
* cqueues &mdash; `luarocks install luaossl`

Password authentication may require a crypto library, [luaossl][].

```bash
$ luarocks install luaossl
```

> **Note:** [LuaCrypto][] can be used as a fallback, but the library is abandoned and not recommended for use

> **Note:** Use within [OpenResty][] will prioritize built  in functions if possible

## Example

```lua
local pgmoon = require("pgmoon")
local pg = pgmoon.new({
  host = "127.0.0.1",
  port = "5432",
  database = "mydb",
  user = "postgres"
})

assert(pg:connect())

local res = assert(pg:query("select * from users where username = " ..
  pg:escape_literal("leafo")))
```

If you are using OpenResty you can relinquish the socket to the connection pool
after you are done with it so it can be reused in future requests:

```lua
pg:keepalive()
```

## Reference

Functions in table returned by `require("pgmoon")`:

### `new(options={})`

Creates a new `Postgres` object from a configuration object. All fields are
optional unless otherwise stated. The newly created object will not
automatically connect, you must call `connect` after creating the object.

Available options:

* `"database"`: the database name to connect to **required**
* `"host"`: the host to connect to (default: `"127.0.0.1"`)
* `"port"`: the port to connect to (default: `"5432"`)
* `"user"`: the database username to authenticate (default: `"postgres"`)
* `"password"`: password for authentication, may be required depending on server configuration
* `"ssl"`: enable ssl (default: `false`)
* `"ssl_verify"`: verify server certificate (default: `nil`)
* `"ssl_required"`: abort the connection if the server does not support SSL connections (default: `nil`)
* `"socket_type"`: the type of socket to use, one of: `"nginx"`, `"haproxy"`, `"luasocket"`, `"cqueues"` (default: `"nginx"` if in nginx, `"luasocket"` otherwise)
* `"application_name"`: set the name of the connection as displayed in `pg_stat_activity`. (default: `"pgmoon"`)
* `"pool"`: (OpenResty only) name of pool to use when using OpenResty cosocket (default: `"#{host}:#{port}:#{database}"`)
* `"pool_size"`: (OpenResty only) Passed directly to OpenResty cosocket connect function, [see docs](https://github.com/openresty/lua-nginx-module#tcpsockconnect)
* `"backlog"`: (OpenResty only) Passed directly to OpenResty cosocket connect function, [see docs](https://github.com/openresty/lua-nginx-module#tcpsockconnect)
* `"cqueues_openssl_context"`: Manually created `opensssl.ssl.context` to use when created cqueues SSL connections
* `"luasec_opts"`: Manually created options object to use when using LuaSec SSL connections

Methods on the `Postgres` object returned by `new`:

### success, err = postgres:connect()

Connects to the Postgres server using the credentials specified in the call to
`new`. On success returns `true`, on failure returns `nil` and the error
message.


### postgres:settimeout(time)

Sets the timeout value (in milliseconds) for all subsequent socket operations
(connect, write, receive). This function does not have any return values.

### success, err = postgres:disconnect()

Closes the socket to the server if one is open. No other methods should be
called on the object after this other than another call to connect.


### success, err = postgres:keepalive(...)

Relinquishes socket to OpenResty socket pool via the `setkeepalive` method. Any
arguments passed here are also passed to `setkeepalive`.

### result, num_queries = postgres:query(query_string)
### result, err, partial, num_queries = postgres:query(query_string)

Sends a query to the server. On failure returns `nil` and the error message.

On success returns a result depending on the kind of query sent.

`SELECT` queries, `INSERT` with `returning`, or anything else that returns a
result set will return an array table of results. Each result is a hash table
where the key is the name of the column and the value is the result for that
row of the result.

```lua
local res = pg:query("select id, name from users")
```

Might return:

```lua
{
  {
    id = 123,
    name = "Leafo"
  },
  {
    id = 234,
    name = "Lee"
  }
}
```

Any queries that affect rows like `UPDATE`, `DELETE`, or `INSERT` return a
table result with the `affected_rows` field set to the number of rows affected.


```lua
local res = pg:query("delete from users")
```

Might return:

```lua
{
  affected_rows = 2
}
```

Any queries with no result set or updated rows will return `true`.


This method also supports sending multiple queries at once by separating them
with a `;`. The number of queries executed is returned as a second return value
after the result object. When more than one query is executed then the result
object changes slightly. It becomes a array table holding all the individual
results:

```lua
local res, num_queries = pg:query([[
  select id, name from users;
  select id, title from posts
]])
```

Might return:

```lua
num_queries = 2

res = {
  {
    {
      id = 123,
      name = "Leafo"
    },
    {
      id = 234,
      name = "Lee"
    }
  },
  {
    {
      id = 546,
      title = "My first post"
    }
  }
}
```

Similarly for queries that return affected rows or just `true`, they will be
wrapped up in an addition array table when there are multiple of them. You can
also mix the different query types as you see fit.

Because Postgres executes each query at a time, earlier ones may succeed and
further ones may fail. If there is a failure with multiple queries then the
partial result and partial number of queries executed is returned after the
error message.


### escaped = postgres:escape_literal(val)

Escapes a Lua value for use as a Postgres value interpolated into a query
string. When sending user provided data into a query you should use this method
to prevent SQL injection attacks.

### escaped = postgres:escape_identifier(val)

Escapes a Lua value for use as a Postgres identifier. This includes things like
table or column names. This does not include regular values, you should use
`escape_literal` for that. Identifier escaping is required when names collide
with built in language keywords.

### str = tostring(postgres)

Returns string representation of current state of `Postgres` object.

## SSL connections

pgmoon can establish an SSL connection to a Postgres server. It can also refuse
to connect to it if the server does not support SSL. Just as pgmoon depends on
LuaSocket for usage outside of OpenResty, it depends on luaossl/LuaSec for SSL
connections in such contexts.

```lua
local pgmoon = require("pgmoon")
local pg = pgmoon.new({
  host = "127.0.0.1",
  ssl = true, -- enable SSL
  ssl_verify = true, -- verify server certificate
  ssl_required = true, -- abort if the server does not support SSL connections
  ssl_version = "tlsv1_2", -- e.g., defaults to highest available, no less than TLS v1.1 (LuaSec only)
  cafile = "...", -- certificate authority (LuaSec only)
  cert = "...", -- client certificate (LuaSec only)
  key = "...", -- client key (LuaSec only)
})

assert(pg:connect())
```

> **Note:** In Postgres 12 and above, the minium SSL version accepted by client
> connections is 1.2. When using LuaSocket + LuaSec to connect to an SSL
> server, if you don't specify an `ssl_version` then `tlsv1_2` is used.

In OpenResty, make sure to configure the
[lua_ssl_trusted_certificate](https://github.com/openresty/lua-nginx-module#lua_ssl_trusted_certificate)
directive if you wish to verify the server certificate.

## Authentication types

Postgres has a handful of authentication types. pgmoon currently supports
Trust and MD5 authentication.

## Type conversion

Postgres has a very rich set of types built in. pgmoon will do its best to
convert any Postgres types into the appropriate Lua type.

All integer, floating point, and numeric types are converted into Lua's number
type. The boolean type is converted into a Lua boolean. The JSON type is
decoded into a Lua table using Lua CJSON. Lua tables can be encoded to JSON as
described below.

Any array types are automatically converted to Lua array tables. If you need to
encode an array in Lua to Postgres' array syntax you can use the
`pgmoon.arrays` module. See below.

Any other types are returned as Lua strings.

## Handling arrays

Arrays are automatically decoded when they are returned from a query. Numeric,
string, and boolean types are automatically loaded accordingly. Nested arrays
are also supported.

Use `encode_array` to encode a Lua table to array syntax for a query:

```lua
local pgmoon = require("pgmoon")
local pg = pgmoon.new(auth)
pg:connect()

local encode_array = require("pgmoon.arrays").encode_array
local my_array = {1,2,3,4,5}
pg:query("insert into some_table (some_arr_col) values(" .. encode_array(my_array) .. ")")
```

### Empty Arrays

When trying to encode an empty array an error will be thrown. Postgres requires
a type when using an array. When there are values in the array Postgres can
infer the type, but with no values in the array no type can be inferred. This
is illustrated in the erorr provided by Postgres:


```
postgres=# select ARRAY[];
ERROR:  cannot determine type of empty array
LINE 1: select ARRAY[];
               ^
HINT:  Explicitly cast to the desired type, for example ARRAY[]::integer[].
```

## Handling JSON

`json` and `jsonb` types are automatically decoded when they are returned from
a query.

Use `encode_json` to encode a Lua table to the JSON syntax for a query:

```lua
local pgmoon = require("pgmoon")
local pg = pgmoon.new(auth)
pg:connect()

local encode_json = require("pgmoon.json").encode_json
local my_tbl = {hello = "world"}
pg:query("insert into some_table (some_json_col) values(" .. encode_json(my_tbl) .. ")")
```

## Handling hstore

Because `hstore` is an extension type, a query is reuired to find out the type
id before pgmoon can automatically decode it. Call the `setup_hstore` method on
your connection object after connecting to set it up.

```lua
local pgmoon = require("pgmoon")
local pg = pgmoon.new(auth)
pg:connect()
pg:setup_hstore()
```

Use `encode_hstore` to encode a Lua table into hstore syntax when updating and
inserting:

```lua
local encode_hstore = require("pgmoon.hstore").encode_hstore
local tbl = {foo = "bar"}
pg:query("insert into some_table (hstore_col) values(" .. encode_hstore(tbl) .. ")")
```

You can manually decode a hstore value from string using the `decode_hstore`
function. This is only required if you didn't call `setup_hstore`.

```lua
local decode_hstore = require("pgmoon.hstore").decode_hstore
local res = pg:query("select * from some_table")
local hstore_tbl = decode_hstore(res[1].hstore_col)
```

## Converting `NULL`s

By default `NULL`s in Postgres are converted to `nil`, meaning they aren't
visible in the resulting tables. If you want to convert `NULL`s to some visible
value set `convert_null` to `true` on the `Postgres` object:

```lua
local pgmoon = require("pgmoon")
local pg = pgmoon.new(auth)
pg:connect()

pg.convert_null = true
local res = pg:query("select NULL the_null")

assert(pg.NULL == res[1].the_null)
```

As shown above, the `NULL` value is set to `pg.NULL`. You can change this value
to make pgmoon use something else as `NULL`. For example if you're using
OpenResty you might want to reuse `ngx.null`.

# Contact

Author: Leaf Corcoran (leafo) ([@moonscript](http://twitter.com/moonscript))
Email: leafot@gmail.com
Homepage: <http://leafo.net>


# Changelog

* 1.13.0 — 2021-10-13 - Add support for scram_sha_256_auth (@murillopaula), 'backlog' and 'pool_size' options while using ngx.socket (@xiaocang), update LuaSec ssl_protocol default options (@jeremymv2), `application_name` option (@mecampbellsoup)
* 1.12.0 — 2021-01-06 - Lua pattern compatibility fix, Support for Lua 5.1 through 5.4 (@jprjr). Fix bug where SSL vesrion was not being passed. Default to TLS v1.2 when using LuaSec. Luabitop is no longer automatically installed as a dependency. New test suite.
* 1.11.0 — 2020-03-26 - Allow for TLS v1.2 when using LuaSec (Miles Elam)
* 1.10.0 — 2019-04-15 - Support luaossl for crypto functions, added better error when missing crypto library
* 1.9.0 — 2018-04-02 - nginx pool name includes user, connection reports name as `pgmoon`
* 1.8.0 — 2016-11-07 — Add cqueues support, SSL calling fix for Nginx cosocket (@thibaultCha)
* 1.7.0 — 2016-09-21 — Add to opm, add support for openresty pool, better default pool, support for hstore (@edan)
* 1.6.0 — 2016-07-21 — Add support for json and jsonb array decoding
* 1.5.0 — 2016-07-12 — Add SSL support (@thibaultCha), Add UUID array type (@edan), Add support for notifications (@starius)
* 1.4.0 — 2016-02-18 — Add support for decoding jsonb, add a json serializer (@thibaultCha)
* 1.3.0 — 2016-02-11 — Fix bug parsing a string that looked like a number failed, add support for using in ngx when in init context (@thibaultCha), add cleartext password auth, fix warning with md5 auth
* 1.2.0 — 2015-07-10 — Add support for PostgreSQL Arrays
* 1.1.1 — 2014-08-12 — Fix a bug with md5 auth
* 1.1.0 — 2014-05-21 — Add support for multiple queries in one call
* 1.0.0 — 2014-05-19 — Initial release

## License (MIT)

Copyright (C) 2021 by Leaf Corcoran

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in
all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN
THE SOFTWARE.

  [luaossl]: https://github.com/wahern/luaossl
  [LuaCrypto]: https://luarocks.org/modules/starius/luacrypto
  [LuaSec]: https://github.com/brunoos/luasec
  [Lapis]: http://leafo.net/lapis
  [OpenResty]: https://openresty.org/
  [LuaSocket]: http://w3.impa.br/~diego/software/luasocket/
  [cqueues]: http://25thandclement.com/~william/projects/cqueues.html
