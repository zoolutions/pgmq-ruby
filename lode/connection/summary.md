# Connection pooling, reconnection and transactions

`PGMQ::Connection` (`lib/pgmq/connection.rb`, 245 lines) is the only place this gem opens a socket. `PGMQ::Transaction` (`lib/pgmq/transaction.rb`, 92 lines) rides on top of it.

## Construction

`Connection#initialize` (`connection.rb:39-57`) raises `Errors::ConfigurationError` for nil params, normalises the rest, and builds the `ConnectionPool` object. The pool itself is lazy: `connection_pool` runs the block that makes a connection only when a checkout finds no idle one, so nothing connects until the first `with_connection`. `normalize_connection_params` (`169-175`) accepts, in this order: anything that `respond_to?(:call)` (kept as the callable), a String (parsed by `PG::Connection.conninfo_parse`, falling back to `{ conninfo: string }` when that method is missing), or a non-empty Hash. Anything else — including an empty Hash — is a `ConfigurationError`. A bad connection string is caught at `parse_connection_string`'s `rescue PG::Error` and re-raised as `ConfigurationError` (`190-191`).

`create_connection` (`232-243`) calls the callable if there is one, otherwise `PG.connect(params[:conninfo] || params)`. It deliberately does not set `conn.type_map_for_results`; the comment at `237-239` says so, and that one omission is what makes every value this gem returns a String. Its own `rescue PG::Error` turns a failed connect into `ConnectionError` (`241-242`).

Defaults: `DEFAULT_POOL_SIZE = 5`, `DEFAULT_POOL_TIMEOUT = 5`, `auto_reconnect: true`.

## The shared-connection guard

`create_pool` (`196-227`) wraps the block `ConnectionPool` calls for each slot. When the value is a `PG::Connection`, the pool records it in an `ObjectSpace::WeakKeyMap` under a `Mutex`; a connection already in the map raises `ConfigurationError` naming the `object_id` and telling the caller to return a distinct connection per invocation.

The check is by object identity, not by parameters, so it fires for any callable that memoises — which includes `-> { ActiveRecord::Base.connection.raw_connection }` on a single Rails checkout with the default `pool_size: 5`. `WeakKeyMap` is used so a garbage-collected connection drops out of the map on its own. The whole `create_pool` body is wrapped in `rescue => e` that re-raises as `ConnectionError` (`225-226`), but that rescue only covers building the pool object — the block runs later, on a checkout, so the guard's `ConfigurationError` reaches the caller as itself. `with_connection` does not rescue it either: its three rescue clauses are `PG::Error` and the two `ConnectionPool` errors. That is what the tests assert (`test/lib/pgmq/connection_test.rb:227-236`).

## Checkout, health check and the one retry

`with_connection` (`64-87`) is the single path to a connection:

1. `retries` is 1 when `auto_reconnect` is on, 0 otherwise.
2. Inside `@pool.with`, `verify_connection!` runs — only when `auto_reconnect` is on.
3. `PG::Error` is caught; the call is retried at most once, and only when `connection_lost_error?` says the socket is gone. Otherwise, and after the retry is spent, it becomes `Errors::ConnectionError` with the original message appended.
4. `ConnectionPool::TimeoutError` and `ConnectionPool::PoolShuttingDownError` become `ConnectionError` too, each with its own prefix (`"Connection pool timeout: "`, `"Connection pool is closed: "`). They are not retried.

`verify_connection!` (`158-163`) resets a connection that is `finished?` **or** whose `status` is `PG::CONNECTION_BAD`, and returns `nil` for a healthy one. The second check is the one that catches a socket torn down server-side by PostgreSQL or a pooler such as PgBouncer, where the client-side object still reports `finished? == false`.

`connection_lost_error?` (`141-146`) matches in two steps: first by class (`PG::ConnectionBad` or `PG::UnableToSend`, which libpq raises for connection failure regardless of message), then by downcased substring against the eleven-entry private `LOST_CONNECTION_MESSAGES` list (`115-128`). The class check short-circuits, so it decides every `PG::ConnectionBad`; the substring list only ever runs for a bare `PG::Error`.

## Closing

`Connection#close` (`91-93`) shuts the pool down, closing each connection that is not already `finished?`. After that, `with_connection` raises `ConnectionError` from the `PoolShuttingDownError` branch. `Client#close` (`client.rb:82-84`) and `Client#stats` (`:92-94`) are one-line forwards to the `Connection`. `Connection#stats` (`101-106`) returns `{ size: @pool_size, available: @pool.available }` — `size` is the configured size, not a count of live connections.

## Transactions

`Transaction#transaction` (`transaction.rb:39-47`) checks out a connection, opens `conn.transaction`, and yields a `TransactionalClient.new(self, conn)`. It rescues `PG::Error, StandardError` and re-raises everything as `ConnectionError` prefixed `"Transaction failed: "` — so a `RuntimeError` raised inside the block reaches the caller as a `ConnectionError`, and the tests assert exactly that (`test/lib/pgmq/transaction_test.rb:170-177`).

`TransactionalClient` is a `method_missing` wrapper (`transaction.rb:52-90`). Understand what actually routes the SQL: `method_missing` calls `@parent.__send__(method, ...)`, so the operation runs with `self` bound to the **parent** `Client`, and the `with_connection` it reaches is `Client#with_connection` → the pool — not the `with_connection` defined on `TransactionalClient` at `82-84`. The operations still land inside the transaction because `ConnectionPool#checkout` is re-entrant per thread: the outer `@pool.with` has already stored the connection in `Thread.current`, and a nested checkout on the same thread returns that same object and only bumps a counter. Two consequences worth knowing before changing this code:

- A transaction body that hands work to **another thread** loses the transaction — that thread's checkout takes a different slot.
- Each nested checkout re-runs `verify_connection!` when `auto_reconnect` is on, so a mid-transaction `conn.reset` is reachable in principle; nothing in the tests covers it.

`TransactionalClient#connection` returns the parent's `Connection`, which is what `transaction_test.rb:226-231` uses to run raw SQL inside a transaction.
