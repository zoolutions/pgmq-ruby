# Practices

`../CLAUDE.md` holds the project's own guidance (architecture, commands, code style). There is no `.claude/rules/` directory. This file adds the practices that reading the code teaches and `CLAUDE.md` does not state; the specific rules with their proofs are in `review/`.

## Staying a low-level client

- The scope line is in the code, not just the README: `lib/pgmq.rb:13-17` says job processing, retries and framework integration belong to `pgmq-framework`. Before adding a method, check that it maps to a PGMQ SQL function or to a PostgreSQL primitive (a transaction, a pool). Anything that encodes a *policy* — a retry schedule, a dead-letter convention, a serialiser — is out of scope by construction.
- Serialisation is the caller's job in both directions. `produce` takes a JSON String and `Message#message` hands one back. Nothing in `lib/` parses JSON, and `lib/` never requires the `json` stdlib — its six `require` lines are `zeitwerk`, `time` (three of them), `pg` and `connection_pool`. The only JSON calls in `lib/` are three occurrences of `conditional.to_json` on the filtered read paths (`client/consumer.rb:42`, `:91`, `:146`), which therefore depend on the caller having loaded `json`; `test/test_helper.rb:20` and `spec/integration/support/example_helper.rb:10` both do.

## SQL

- Bind, do not interpolate. Of the 48 SQL calls in `lib/pgmq/client/`, 43 are `exec_params` and three more are `conn.exec` with constant SQL; the two that build a string are `read_multi` and `pop_multi`, where a `UNION ALL` over N queues cannot be expressed with bound parameters. If a new method needs the same shape, copy their discipline: `validate_queue_name!` every name, `gsub("'", "''")` it anyway, and `to_i` every number that lands in the string.
- Pick the SQL overload from the Ruby value's shape and keep the branches shallow: `headers` present, `conditional.empty?`, `vt.is_a?(Time)`. Each branch spells out the full `exec_params` call rather than assembling a parameter list, which is verbose but keeps the argument order visible next to the `$n::type` markers.
- Every `$n` carries an explicit cast (`$1::text`, `$2::jsonb`, `$3::bigint[]`). PostgreSQL cannot resolve PGMQ's overloaded functions from untyped parameters, so dropping a cast changes which function is called.

## Returning values

- Return the row's value, unconverted. A method that starts parsing a timestamp or `to_i`-ing a count has taken a decision away from the caller, and the whole gem's contract is that it does not. `produce_topic`'s `.to_i` is the single exception and it should stay single.
- Guard on `result.ntuples.zero?` before touching `result[0]`, and return `nil` (for a model) or `false`/`[]` (for a scalar) — `pop`, `read`, `set_vt`, `metrics`, `drop_queue` and `delete` all do.
- New YARD `@return` tags describe what the method actually hands back. Four tags in `lib/` name `Integer`; three of them are wrong because the method returns PostgreSQL's String: `maintenance.rb:13` (`Integer`, `purge_queue` returns `"3"`) and `message_lifecycle.rb:80` / `:174` (`Array<Integer>`, `delete_batch` and `archive_batch` return Arrays of Strings). The fourth, `topics.rb:76`, is accurate because `produce_topic` is the one method that calls `.to_i`. Do not add a fifth wrong one.

## Documentation as a build step

- `yard-lint` runs with `RequiredCoverage: 100` and `FailOnSeverity: convention` (`.yard-lint.yml:15-16`), and it excludes `spec/**` and `test/**`, so documentation of `lib/` is not optional: a new public method needs a description, a `@param` per argument and a `@return`. `@example` is house style, not config: all 39 public methods in `lib/pgmq/client/` carry at least one, while `Connection`, `Transaction`, the three models and `Client`'s own `close`/`stats` do not. Tag order *is* enforced: `param`, `option`, `return`, `raise`, `example` (`Tags/Order`).
- Prose that says `Note:` or `Returns:` at the start of a line fails `Tags/InformalNotation`; use `@note` and `@return`.
- An `@example` is parsed as Ruby (`Tags/ExampleSyntax`), so it has to be syntactically valid — and, being the first thing a user reads, it should also be callable: the arity mistakes reviewers catch usually start in an example.

## Concurrency

- A `Client` is shared across threads by design; a `PG::Connection` never is. Anything new that hands out a connection goes through `Connection#with_connection` so the pool stays the single owner.
- Work started inside `client.transaction` stays on the calling thread. The transactional client rides `connection_pool`'s per-thread re-entrant checkout, so a `Thread.new` inside the block silently takes a different connection and leaves the transaction.
