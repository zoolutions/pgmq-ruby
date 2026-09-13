# Review rules: tests and examples

Accepted review findings from merged PR threads on `mensfeld/pgmq-ruby`, rewritten as rules about the system and verified against this tree.

### A concurrency test sequences the threads it needs; it never races them and hopes
- **Holds because:** "start two threads and expect them to collide" passes or fails on the scheduler. The shared-connection test instead *forces* the situation: a holder thread occupies slot one, the main thread then demands slot two, so the pool has to run the factory a second time. Handoff is a `Queue` — `ready_queue.pop` — not a `sleep … until flag`, because the holder can die before setting a flag and the waiter would then spin forever. Every exit path from the holder, including both rescue clauses, pushes to the queue.
- **Where:** `test/lib/pgmq/connection_test.rb:198-250`
- **Proven by:** the test itself; it is the only deterministic-concurrency pattern in the suite, and `spec/integration/shared_connection_thread_safety_spec.rb` is the deliberate opposite — a probabilistic script that prints "Race not triggered this run" rather than failing.
- **Origin:** PR #85 and PR #86 review threads (merged; #86 replaced #85's busy-wait flag with the queue)

### Everything a test or example opens is closed in an `ensure`
- **Holds because:** the suite creates a client per test against a real pool, and a leaked `ConnectionPool` holds server-side connections for the rest of the run — which is how a later test hits a pool timeout for no reason of its own. Three shapes are in the tree: `teardown_client_and_queue` drops the queue, swallows the failure and closes the client in an `ensure` (`test/support/database_helpers.rb:75-81`); a test that builds its own `Connection` closes it in a rescued `ensure` (`test/lib/pgmq/connection_test.rb:237-248`); an example that also opens a raw `PG::Connection` closes **both** the client and the raw connection, each in its own `begin/rescue nil`, because `client.close` only reaches connections the pool actually took (`spec/integration/shared_connection_thread_safety_spec.rb:49-61`, `spec/integration/shared_connection_detection_spec.rb:49-60`).
- **Where:** the three files above. A fourth shape closes without an `ensure` and gets away with it: `test/lib/pgmq/client/multi_queue_test.rb:14-21` rescues each `drop_queue` individually so the `@client.close` on the last line is always reached — the shape PR #78 asked for. Copy `teardown_client_and_queue` instead of inventing a fifth.
- **Proven by:** no assertion — this is hygiene the suite relies on; a regression shows up as an unrelated pool timeout.
- **Origin:** PR #78, PR #85 and PR #86 review threads

### Examples reach the pool through the public `Client#connection` reader
- **Holds because:** `attr_reader :connection` is public API (`lib/pgmq/client.rb:42`), so `instance_variable_get(:@connection)` buys nothing and breaks silently if the ivar is renamed. Both shared-connection examples use `client.connection.with_connection` (`spec/integration/shared_connection_detection_spec.rb:19`, `:37`).
- **Where:** `spec/integration/shared_connection_detection_spec.rb`
- **Proven by:** no assertion. The rule is honoured in `spec/`; `test/lib/pgmq/connection_test.rb:103` and `:390` still reach in with `instance_variable_get`, so this is a rule for new code, not a description of the whole tree.
- **Origin:** PR #85 review thread

### A grouping or ordering test asserts the distinct values, not just the count
- **Holds because:** `assert_equal 4, messages.size` passes just as happily when the implementation returns four messages from one group — the exact bug a round-robin read exists to prevent. The `read_grouped_rr` test produces messages for three distinct `user_id`s and then asserts each one appears in the result (`test/lib/pgmq/client/consumer_test.rb:250-255`).
- **Where:** `test/lib/pgmq/client/consumer_test.rb:237-256`
- **Proven by:** that test. The finding was raised against `read_grouped_head`, which this tree does not have; the rule is kept because `read_grouped_rr` and `read_grouped_rr_with_poll` have the same failure mode.
- **Origin:** PR #121 review thread

### Timestamps are compared as Strings on purpose; do not reach for `Time.parse`
- **Holds because:** the gem's whole contract is that nothing is converted, so a test that parses `msg.vt` before comparing it is asserting against a value the library never hands anyone. PostgreSQL's `timestamptz` text output is fixed-width and sorts lexicographically, which makes `assert_operator updated_msg.vt, :>, msg.vt` (`test/lib/pgmq/client/message_lifecycle_test.rb:178`) both correct and a statement of the contract. No test in the suite parses a time.
- **Where:** `test/lib/pgmq/client/message_lifecycle_test.rb:170-179`
- **Safe direction:** if the format ever stops sorting, the test fails loudly rather than passing on a parsed value that hides the contract change.
- **Origin:** PR #63 review thread (raised, deliberately not acted on — the tree still compares Strings)

### A wait is the timeout it is waiting out, plus half a second
- **Holds because:** these are real waits against a real server, 19 of them, and the suite's runtime is the sum. The tree's pattern is to pick the shortest timeout that proves the point and add a 0.5-second buffer: `vt: 2` → `sleep 2.5` (`test/lib/pgmq/client/consumer_test.rb:276`), `vt: 3` → `sleep 3.5` (`:219-227`). A round 3.5-second sleep guarding a 2-second timeout buys nothing and costs a second on all 15 CI cells.
- **Where:** every `sleep` in `test/`
- **Proven by:** no assertion — the cost shows as suite runtime, not as a failure.
- **Origin:** PR #119 review thread (accepted upstream across five test files)
