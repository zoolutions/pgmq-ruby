# Lode map

The index of this repository's durable memory. Read this first; it beats a directory listing. Every file describes the system as it is now, with rationale; `../CHANGELOG.md` records what changed.

- `summary.md` — what pgmq-ruby is, the three invariants (nothing is converted, every queue name is validated, the pool owns connection identity), and where this fork sits relative to `mensfeld/pgmq-ruby`
- `terminology.md` — the words this repo uses (raw value, `vt`, produce/consume, batch versus multi, conditional, callable connection, shared-connection guard, lint bundle…)
- `practices.md` — what reading the code teaches and `../CLAUDE.md` does not say: staying a low-level client, bind-don't-interpolate, returning values unconverted, YARD as a build step, concurrency
- `workflow.md` — the profile the shared `/lode:` workflow skills read: commands, branches, layers, input shapes, constraints, docs, CI, flake sources, conflict rules, verification
- `plans/README.md` — where plans live (GitHub issues upstream and on the fork); handovers in `tmp/`, not committed

## Subsystems

- `client/summary.md` — `PGMQ::Client` and its nine included modules: the 39 public methods, queue-name validation as the injection boundary, the two methods that interpolate SQL, argument shapes, SQL-overload selection, return shapes
- `connection/summary.md` — `PGMQ::Connection` and `PGMQ::Transaction`: construction and parameter normalisation, the shared-connection guard, checkout/health-check/single-retry, closing, and why a `TransactionalClient` operation lands in the transaction
- `models/summary.md` — the raw-String contract, the six places a `"t"` becomes a Ruby boolean, the three `Data.define` row models, and which of the seven error classes are ever raised
- `testing-and-ci/summary.md` — the unit suite versus the integration examples, the helpers both rely on, timing in tests, the 15-cell CI matrix and the `ci-success` gate, the two linters, the release workflow

## Review rules (`review/`)

Accepted review findings from merged PR threads on `mensfeld/pgmq-ruby`, rewritten as rules about the system, verified against this tree, each with the test that proves it or an honest "no test". `/lode:gate` reads every file here before reviewing a diff; `/lode:learn` adds to them.

- `review/connection.md` — the guard's mutex, the remedy its message names, identity-based detection versus `pool_size`, class-before-message error matching, nil parameters
- `review/client.md` — keyword `qty:` versus positional `pop_batch`, the exclusive 48-character queue-name limit, `Time`-only overload selection
- `review/testing.md` — deterministic concurrency with a `Queue`, closing in an `ensure`, the public `connection` reader, asserting distinct values rather than counts, comparing timestamps as Strings, the timeout-plus-half-a-second wait
- `review/tooling.md` — `bin/integrations` exit 130, Rake paths resolved against `__dir__`, restoring a SIGINT handler by type, and two *Not a rule here* entries (the unsynchronised interrupt flag, `FailOnSeverity: convention`)
- `review/docs.md` — `@example` stays inside the documented object's API and must be callable, and the informal-notation lint

## Not memory

- `tmp/` — git-ignored: gate diffs and reports, handovers, scratch
