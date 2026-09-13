# Review rules: the client's public API

Accepted review findings from merged PR threads on `mensfeld/pgmq-ruby`, rewritten as rules about the system and verified against this tree.

### Only the queue name is positional on a read; counts are keywords
- **Holds because:** `read_batch(queue, 5, vt: 30)` raises `ArgumentError (given 2, expected 1)` — `read_batch`, `read_with_poll`, `read_grouped_rr` and `read_grouped_rr_with_poll` all take `qty:` as a keyword (`lib/pgmq/client/consumer.rb:74-79`, `125-132`, `:176`, `208-214`). `pop_batch(queue_name, qty)` is the one method in the gem that takes its count positionally (`lib/pgmq/client/message_lifecycle.rb:39`), so it is the shape reviewers and callers mistake for the others.
- **Where:** `lib/pgmq/client/consumer.rb`, `lib/pgmq/client/message_lifecycle.rb`
- **Proven by:** `spec/integration/fiber_scheduler_spec.rb:53`, `:82`, `:118` all call `read_batch(queue, vt: …, qty: …)` — the three sites the review fixed. No unit test asserts the `ArgumentError`.
- **Origin:** PR #59 review thread

### The 48-character queue-name limit is PGMQ's, not PostgreSQL's, and it is exclusive
- **Holds because:** PostgreSQL's identifier limit is 63, but PGMQ derives `pgmq.q_<name>` and `pgmq.a_<name>` from the queue name and needs room for the prefixes and suffixes — the comment at `lib/pgmq/client.rb:117-119` says exactly that. The check is `queue_name.to_s.length >= 48` (`:120`), so **47 characters is the longest name that passes** and 48 is rejected. A comment or test that says "48 characters is the maximum" is off by one, and one that reaches for 63 is validating the wrong database's rule.
- **Where:** `lib/pgmq/client#validate_queue_name!` (109-137)
- **Safe direction:** rejecting a name the extension would have accepted is a clear `InvalidQueueNameError` at the call site; letting one through produces a PostgreSQL identifier-truncation collision between two queues.
- **Proven by:** `test/lib/pgmq/client_test.rb:61-64` and `:117-123` sit on both sides of the boundary — the 48 case asserts the message, the 47 case asserts nothing and passes by not raising. The same method is the single injection boundary for `read_multi`/`pop_multi`, which interpolate the validated name into SQL text.
- **Origin:** PR #47 review thread

### A time-like argument takes the `timestamptz` overload only when it is literally a `Time`
- **Holds because:** `set_vt` and `set_vt_batch` branch on `vt.is_a?(Time)` (`lib/pgmq/client/message_lifecycle.rb:250`, `:292` — the only two `is_a?(Time)` tests in `lib/`). An `ActiveSupport::TimeWithZone`, which is what `Time.current` returns in the Rails apps this gem's README courts, is **not** a `Time`, so it falls through to the `integer` offset overload and fails at query execution rather than at the call. The fix upstream accepted for the same shape in `produce` was to accept any object that responds to `to_time` and treat only `Numeric` as an offset; this tree has not had that change, so the constraint stands as written.
- **Where:** `lib/pgmq/client/message_lifecycle.rb#set_vt` (246-266), `#set_vt_batch` (284-306); `set_vt_multi` inherits the behaviour by calling `set_vt_batch` (`:349`)
- **Safe direction:** narrowing to `Time` never sends a wrong timestamp; it only refuses to recognise one, and the failure is a query error naming the type.
- **Proven by:** `test/lib/pgmq/client/message_lifecycle_test.rb:181-192` covers the `Time` branch with a real `Time.now + 120` (and skips below PGMQ v1.11.0). Nothing in `test/` or `spec/` passes a time-like object that is not a `Time`.
- **Origin:** PR #119 review thread (accepted upstream for `produce`/`produce_batch`/`produce_batch_topic`, which this tree does not have)
