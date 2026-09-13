# Review rules: connection pooling and reconnection

Accepted review findings from merged PR threads on `mensfeld/pgmq-ruby`, rewritten as rules about the system and verified against this tree.

### The shared-connection guard is a check-then-insert, so it holds a Mutex
- **Holds because:** without it the guard loses the race it exists to win. Two threads creating pool slots at the same moment can both see the connection as unseen and both insert it, so the same `PG::Connection` reaches two slots — the exact corruption (nil `PG::Result`, segfaults, wrong rows) the check was written to prevent. `create_pool` allocates one `seen_mutex` outside the `ConnectionPool.new` block and wraps the `key?` test and the assignment in a single `synchronize` (`lib/pgmq/connection.rb:199`, `208-220`). The `WeakKeyMap` itself gives no atomicity.
- **Where:** `lib/pgmq/connection.rb#create_pool` (196-227)
- **Safe direction:** raising `ConfigurationError` on a connection that was in fact fresh is harmless; letting a shared one through is not.
- **Proven by:** `test/lib/pgmq/connection_test.rb:"raises ConfigurationError when callable returns same connection to multiple slots"` (199-249) — it proves the guard fires, not that the lock is held under contention.
- **Origin:** PR #85 review thread

### The guard's error message names a fix that actually produces a distinct connection
- **Holds because:** the message is the only guidance a user gets at the moment their app stops booting, and the obvious suggestion is wrong: `ActiveRecord::Base.connection.raw_connection` returns the same object on repeated calls in one thread, so recommending it would send the reader back into the error. The message ends "by calling `PG.connect` inside the callable" (`lib/pgmq/connection.rb:215-216`).
- **Where:** `lib/pgmq/connection.rb#create_pool` (210-217)
- **Proven by:** `test/lib/pgmq/connection_test.rb:236` asserts only the `/same PG::Connection object/` fragment; the remedy sentence has no assertion on it.
- **Origin:** PR #85 review thread

### Detection is by object identity, so any memoising callable raises once `pool_size > 1`
- **Holds because:** the guard keys on the `PG::Connection` object, not on connection parameters, and `pool_size` defaults to 5. Every callable that returns a cached connection — the Rails `-> { ActiveRecord::Base.connection.raw_connection }` pattern included — hits it as soon as the pool needs a second slot. A change that documents or recommends a callable has to say which of the two escapes applies: return a fresh connection per call, or pass `pool_size: 1`.
- **Where:** `lib/pgmq/connection.rb#create_pool` (198-221); `CHANGELOG.md` 0.6.0 marks it `[Breaking]`
- **Proven by:** `spec/integration/shared_connection_detection_spec.rb` (the unsafe and the safe pattern, side by side)
- **Origin:** PR #85 review thread (the changelog half was applied; `README.md:212` and `:257` still recommend the raw_connection pattern without the caveat)

### `connection_lost_error?` matches by class first and by message only as a fallback
- **Holds because:** libpq promises a dedicated class for a dead socket but not a stable message across OS, pooler and TLS versions, so class-matching catches the next variant without waiting for it to hit production and then be added to a list. `PG::ConnectionBad` and `PG::UnableToSend` return `true` immediately; only a bare `PG::Error` reaches the eleven-entry `LOST_CONNECTION_MESSAGES` substring scan (`lib/pgmq/connection.rb:141-146`). Adding a message to the list therefore changes nothing for those two classes.
- **Where:** `lib/pgmq/connection.rb#connection_lost_error?` (141-146), `LOST_CONNECTION_MESSAGES` (115-128)
- **Safe direction:** a false positive costs one extra retry on a fresh connection; a false negative fails the user's call.
- **Proven by:** `test/lib/pgmq/connection_test.rb:252-323` — but **no test reaches the substring path**. Six of the seven cases are positive and every one of them constructs `PG::ConnectionBad` (`:262`, `:274`, `:291`, `:299`, `:311`) or `PG::UnableToSend` (`:319`), both of which the class check answers at line 142; the only `PG::Error` case (`:280-284`) is a negative. Emptying `LOST_CONNECTION_MESSAGES` would keep this suite green, and no other file in `test/` or `spec/` names the constant or the method.
- **Origin:** PR #99 review thread (raised, not yet addressed)

### `Connection` requires parameters; nil is a `ConfigurationError`, not a default
- **Holds because:** this gem reads no `ENV` of its own — `CLAUDE.md` states the user manages their own `ENV.fetch` calls — so a nil parameter set has no sensible fallback and failing at construction beats connecting to whatever `PG.connect` would guess from the environment.
- **Where:** `lib/pgmq/connection.rb#initialize` (45-50)
- **Proven by:** no test. `PGMQ::Client.new` with no argument passes `nil` straight through, and no file under `test/` or `spec/` constructs a `Client` or `Connection` with `nil`, an empty Hash or a malformed connection string — the only `ConfigurationError` assertions in the suite are the shared-connection ones at `connection_test.rb:213`, `:227`, `:235`. All three `ConfigurationError` paths in `normalize_connection_params` and `parse_connection_string` are uncovered.
- **Origin:** PR #5 review thread
