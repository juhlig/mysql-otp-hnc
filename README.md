MySQL/OTP + hnc
===============

**MySQL/OTP + hnc** provides connection pooling for
[MySQL/OTP](//github.com/mysql-otp/mysql-otp) using
[hnc](//github.com/juhlig/hnc). It contains convenience functions for
executing SQL queries on a connection in a pool and lets you choose between two
methods for creating and managing connection pools:

1. Use it as a library that helps you supervise your own MySQL connection pools.
2. Use it as an application that manages its own supervisor for connection pools.

I want to supervise my own connection pools
-------------------------------------------

Use `mysql_hnc:child_spec/3` to get a child spec for a pool that you can use
in your own supervisor.

```Erlang
%% my own supervisor
init([]) ->
    PoolOptions  = #{size => {5, 10}},
    MySqlOptions = [{user, "aladdin"}, {password, "sesame"}, {database, "test"},
                    {prepare, [{foo, "SELECT * FROM foo WHERE id=?"}]}],
    ChildSpecs = [
        %% MySQL pools
        mysql_hnc:child_spec(pool1, PoolOptions, MySqlOptions),
        %% other workers...
        {some_other_worker, {some_other_worker, start_link, []},
         permanent, 10, worker, [some_other_worker]}
    ],
    {ok, #{strategy => one_for_one}, ChildSpecs}}.
```

Let MySQL/OTP + hnc supervise my pools
--------------------------------------

This approach requires you to start the application `mysql_hnc`. Typically
this is done by adding `{applications, [mysql_hnc]}` to your `.app.src`
file and then relying on your favourite release tool for the rest.

Pools can be added at run-time using `mysql_hnc:add_pool/3`.

Pools can also be created at start-up by defining configuration parameters for
`mysql_hnc`. The name of each configuration parameter is the pool name and
the value is a pair on the form `{PoolOptions, MySqlOptions}`.

Example:

Start your Erlang node with `erl -config mypools.config` where `mypools.config`
is a file with the following contents:

```Erlang
[
 {mysql_hnc, [
    {pool1, {#{size => {5, 10}},
             [{user, "aladdin"}, {password, "sesame"}, {database, "test"},
              {prepare, [{foo, "SELECT * FROM foo WHERE id=?"}]}]}}
]}].
```

Using the connection pools
--------------------------

The most commonly used MySQL functions are available with wrappers in
`mysql_hnc`.

```Erlang
1> mysql_hnc:query(pool1, "SELECT * FROM foo WHERE id=?", [42]).
{ok,[<<"id">>,<<"bar">>],[[42,<<"baz">>]]}
2> mysql_hnc:execute(pool1, foo, [42]).
{ok,[<<"id">>,<<"bar">>],[[42,<<"baz">>]]}
```

For transactions, the connection pid is passed to the transaction fun as the
first parameter.

```Erlang
3> mysql_hnc:transaction(pool1, fun (Pid) ->
       ok = mysql:query(Pid, "INSERT INTO foo VALUES (?, ?)", [1, <<"banana">>]),
       ok = mysql:query(Pid, "INSERT INTO foo VALUES (?, ?)", [2, <<"kiwi">>]),
       hello
   end).
{atomic, hello}
```

Sometimes you need to checkout a connection to execute multiple queries on it,
without wrapping it in an SQL transaction. For this purpose you can use either
a pair of calls to `checkout/1` and `checkin/1` or a call to `with/2` with a
fun as in this example:

```Erlang
4> mysql_hnc:with(pool1, fun (Pid) ->
       {ok, _, [[OldTz]]} = mysql:query(Pid, "SELECT @@time_zone"),
       ok = mysql:query(Pid, "SET time_zone = '+00:00'"),
       %% Do some stuff in the UTC time zone...
       ok = mysql:query(Pid, "SET time_zone = ?", [OldTz])
   end).
ok
```

Using `checkout/1` and `checkin/1`
----------------------------------

The function `checkout/1` does not return a MySQL connection directly but a
connection _identifier_ (actually, a `hnc` worker identifier). You have to
unpack the real connection with a call to `get_connection/1` first.

Accordingly, the function `checkin/1` does not expect a MySQL connection but
the connection _identifier_ from which it was unpacked.

A note on using session data
----------------------------

Connections to a MySQL server will be established and kept open in the pool.
Using the `mysql_hnc` functions like `query`, `execute`, `transaction` etc
will use a random connection from the pool to execute a query, ie it is unlikely
that a `mysql_hnc:query` or similar call will be executed using the same connection
as the ones before.

If you are using session data like user variables, temporary tables etc, ie things
that are bound to the connection that creates them and gets destroyed when it
is closed, this means two things:

* Any session data you created by a `mysql_hnc:query` (or similar) call may not exist
  or have different values when you use `mysql_hnc:query` again, as it will likely
  be executed using a different connection. This behavior cannot be changed, you have
  to keep it in mind when writing your code.

* Any session data you created by a `mysql_hnc:query` (or similar) call will still
  exist when another process executes a query and is given this connection, ie the
  session is "dirty". Another possible scenario is that one process changed
  the connection to use another user, in which case it will be logged in as
  that user when another process gets this connection. This can be circumvented
  by using the on_return callback option of `hnc` (it will be called whenever a
  worker returns to the pool). You may use this callback to either call
  `mysql:reset_connection` or `mysql:change_user` in order to automatically reset
  the connection to a clean state when the connection is returned to the pool.

```Erlang
1> OnReturn = fun (Conn) -> mysql:change_user(Conn, "user0", "password0") end.
#Fun<erl_eval.44.97283095>
2> mysql_hnc:add_pool(pool1, #{on_return => {OnReturn, 5000}}, [{user, "user0"}, {password, "password0"}]).
{ok,<0.116.0>}
```

Use this as a dependency
------------------------

Using *erlang.mk*, put this in your `Makefile`:

```Erlang
DEPS = mysql_hnc
dep_mysql_hnc = git https://github.com/mysql-otp/mysql-otp-hnc 0.2.0
```

Using *rebar*, put this in your `rebar.config`:

```Erlang
{deps, [
    {mysql_hnc, ".*", {git, "https://github.com/mysql-otp/mysql-otp-hnc",
                           {tag, "0.2.0"}}}
]}.
```

License
-------

GNU Lesser General Public License (LGPL) version 3 or any later version.
Since the LGPL is a set of additional permissions on top of the GPL, both
license texts are included in the files [COPYING.LESSER](COPYING.LESSER) and
[COPYING](COPYING) respectively.
