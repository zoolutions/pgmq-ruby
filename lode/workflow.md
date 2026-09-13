# Workflow profile

Everything the shared workflow skills (`/lode:lfg`, `/lode:review-pr`, `/lode:finish-prs`, `/lode:debug-flaky`, `/lode:tdd`, `/lode:plan`) need to know about this repository that is not already in `../CLAUDE.md` or the rest of `lode/`.

## Commands

| Purpose | Command | Notes |
|---|---|---|
| fast loop (one file) | `bundle exec ruby -Ilib:test test/lib/pgmq/<name>_test.rb` | needs Postgres+pgmq on 5433; add `-n /pattern/` for one case |
| full suite | `bundle exec rake test` | `Rakefile:8-13`. No network, but it needs the database. **Not safe in two worktrees at once**: both point at the same `pgmq_test` database on 5433. Queue names are `SecureRandom` so rows will not collide, but pool size and connection limits will. |
| integration examples | `bin/integrations` (or `rake examples`, `rake examples:run`, `rake examples:run_one[name]`, `rake examples:list`) | 19 scripts, each a subprocess; same database |
| lint | `BUNDLE_GEMFILE=Gemfile.lint bundle exec rubocop` | second bundle; `standard` + `rubocop-minitest` + `rubocop-performance` |
| docs build / check | `BUNDLE_GEMFILE=Gemfile.lint bundle exec yard-lint lib/` | the only docs check there is; `RequiredCoverage: 100`, `FailOnSeverity: convention`. There is no generated docs site. |
| one CI cell locally | `docker compose up -d` then `PG_PORT=5433 bundle exec rake test` | the matrix varies Ruby and the Postgres image tag; `docker-compose.yml` pins `pg18` |
| config lint | `npx lostconf --fail-on-stale` | CI's `lostconf` job; needs `npm ci` |
| run the app | n/a — this is a library. `bundle exec ruby -Ilib -rpgmq -e '…'` is the closest thing; there is no `bin/console` despite `CLAUDE.md` and `DEVELOPMENT.md` mentioning one. |

## Branches and PRs

- Default branch: **`master`**, not `main`. `ci.yml` only runs on pull requests targeting `master`.
- `zoolutions/pgmq-ruby` is a fork of `mensfeld/pgmq-ruby` with no commits of its own; its `master` is an exact ancestor of upstream's and 110 commits behind it (here `0.6.1`, upstream `0.7.2`). Upstream is where the history and the releases are, so a non-fork-specific fix belongs there first — and check upstream before writing a feature: `read_grouped_head`, pool `reload`, time-like `delay`, and queue-name `normalize`/`sanitize!` already exist there.
- Work branches: `feat/*`, `fix/*`, `chore/*`, `docs/*`, `ci/*`, rooted off fresh `origin/master`.
- Commits: conventional, and upstream's history scopes them — `fix(connection):`, `feat(queue-name):`, `chore(deps):`. The body says why.
- PR body sections, in order: Summary, Test plan, Deviations & judgment calls, Gate.
- Merge policy: squash on `master` after green; never rebase a published branch.
- Attribution: no `Co-Authored-By` and no "Generated with" lines.

## Layers

| Layer | Files | Edit rule |
|---|---|---|
| Entry point | `lib/pgmq.rb` | Zeitwerk loader + the `PGMQ.new` shim; new files are picked up automatically, so only an inflection change belongs here |
| Client core | `lib/pgmq/client.rb` | small by design (139 lines) — new operations go in a module, not here |
| Operation modules | `lib/pgmq/client/*.rb` | owned here; one module per domain, `validate_queue_name!` first, `exec_params` for the SQL |
| Connection & transactions | `lib/pgmq/connection.rb`, `lib/pgmq/transaction.rb` | the only socket owner; changes here are the riskiest in the repo |
| Models & errors | `lib/pgmq/{message,metrics,queue_metadata,errors}.rb` | `Data.define` + a `new(row, **)` reader; raw Strings out |
| Unit tests | `test/lib/**` | Minitest/Spec, real database |
| Examples | `spec/integration/*.rb` | plain scripts run by `bin/integrations`; they raise instead of asserting |
| Generated / pinned | `Gemfile.lock`, `Gemfile.lint.lock`, `package-lock.json` | regenerate, never hand-merge |
| Upstream-owned in practice | everything, since this fork carries no diff | keep changes small enough to send upstream |

## Shapes

- **Queue names:** valid (`orders`, `_x`, `A1`), 47 characters (accepted) vs 48 (rejected), nil, `""`, whitespace-only, leading digit, hyphen, space, special characters.
- **Values back from PG:** always Strings — `"t"`/`"f"` for booleans, `"3"` for counts, a raw JSONB String for payloads. An assertion of `3` or `true` against a fresh call is a bug in the test.
- **Empty results:** `result.ntuples.zero?` — every read path has an "empty queue" case.
- **Collections:** empty Array / empty Hash short-circuits before the query in `produce_batch`, `delete_batch`, `archive_batch`, `set_vt_batch` and the three `*_multi` methods; `pop_batch` short-circuits on `qty <= 0`.
- **Multi-queue guards:** non-Array, empty Array, more than 50 queues.
- **Optional arguments that pick a SQL overload:** `headers` present/absent, `conditional` empty/non-empty, `vt` `Time`/Integer, topic `delay` zero/positive.
- **Connection inputs:** connection String, Hash, callable, an existing `PGMQ::Connection`, `nil` (→ `ConfigurationError`), empty Hash (→ `ConfigurationError`), a callable that returns the *same* connection twice (→ `ConfigurationError`).
- **Environments:** Ruby 3.3 / 3.4 / 4.0 × PostgreSQL 14–18; PGMQ before and after v1.11.0 (topic routing and `set_vt` with a timestamp need v1.11.0+, and `database_helpers.rb` has the version probe for skipping).

## Constraints

| Suggestion | Why it is wrong here |
|---|---|
| "Parse the timestamp / `to_i` the count before returning it" | The gem's contract is raw values; `type_map_for_results` is unset on purpose (`lib/pgmq/connection.rb:237-239`). |
| "Add a retry/backoff policy, a dead-letter queue, an ActiveJob adapter" | Out of scope by design — those belong to `pgmq-framework` (`lib/pgmq.rb:13-17`). |
| "Add a dependency for X" | Runtime deps are `pg`, `connection_pool`, `zeitwerk` and nothing else. |
| "Use `conn.exec` with interpolation, it's simpler" | Only `read_multi` and `pop_multi` may build SQL text, and only because a `UNION ALL` over N queues needs it. |
| "Mutex the `ExampleHelper` interrupt flag" | See `review/tooling.md` — deliberate, and the flag drives nothing irreversible. |
| "Recommend `-> { ActiveRecord::Base.connection.raw_connection }`" | It returns the same object per thread and trips the shared-connection guard at the default `pool_size: 5`. |
| "Rename the default branch / target `main`" | `master` is the default and `ci.yml` filters on it. |

## Docs

- User-facing docs live in `README.md` (single file, 896 lines). A new public method maps to a row in the **PGMQ Feature Support** table (`:41`) and to a code block under the matching API Reference heading (`:301`). Seven `### ` sections: Queue Management, Sending Messages, Reading Messages, Message Lifecycle, Monitoring, Transaction Support, Topic Routing — plus `#### ` subsections the Table of Contents links as if they were sections (Queue Naming Rules, Grouped Round-Robin Reading, Conditional Message Filtering, Topic Patterns).
- YARD in `lib/` is enforced by CI, so the doc block is part of the change, not a follow-up.
- Changelog: `CHANGELOG.md`, newest first, entries under `## Unreleased` until a release renames it to `## X.Y.Z (date)`. Inside a version, entries group under `### <Area>` headings (`Connection Management`, `Breaking Changes`, `Infrastructure`, `Testing`, …) and each bullet starts with a bold tag: `**[Fix]**`, `**[Feature]**`, `**[Breaking]**`, `**[Change]**`.
- `DEVELOPMENT.md` is a second developer guide that overlaps `CLAUDE.md`; a change to the commands or the project layout touches both.
- Files that pin a version and drift: `Gemfile.lock` pins `pgmq-ruby (X.Y.Z)` through the `gemspec` path, so it moves with `lib/pgmq/version.rb`; regenerate with `bundle install`. `Gemfile.lint.lock` and `package-lock.json` are Renovate's.

## CI

- Workflows: `.github/workflows/ci.yml` (pull requests to `master`, plus a nightly cron at 01:00 UTC) and `.github/workflows/push.yml` (`v*` tags → RubyGems, gated on `github.repository_owner == 'mensfeld'`).
- Matrix: Ruby `4.0.0`/`3.4`/`3.3` × Postgres `14`–`18` = 15 cells. Differences from local: `Gemfile.lock` is deleted on the Ruby 4.0 rows, the extension is created by an explicit `psql … CREATE EXTENSION`, and `GITHUB_COVERAGE=true` is set only on Ruby 3.4 × Postgres 18.
- Fetch a failure: `gh run view --job <id> --log-failed`.
- "Green" means the `ci-success` job passed — it fails when any of `rubocop`, `tests`, `yard-lint`, `lostconf` reported failure, cancellation **or skip**, so a skipped job is a red build.
- Concurrency is `${{ github.workflow }}-${{ github.ref }}` with `cancel-in-progress: true`: a new push cancels the previous run on the same branch.
- Shared or rate-limited services: none outside the per-cell Postgres service container. `lostconf` and `bundle install` reach the network.

## Flake sources

- **Real time.** 19 `sleep` calls in `test/` wait out visibility timeouts and long-poll windows, each timeout plus a 0.5-second buffer. The tightest margins are the bounded long-poll assertions: `multi_queue_test.rb:218-222` demands `elapsed >= 1.0` *and* `< 1.5`, `:204-207` demands `< 0.5`, and `consumer_test.rb:300-304` demands `>= 0.5` and `< 2.0`. A loaded machine is the first suspect.
- **Threads racing the pool.** `test/lib/pgmq/connection_test.rb` starts holder threads and asserts on pool availability; the pool-timeout test relies on `sleep 0.1` letting a thread acquire a connection first.
- **Probabilistic by design.** `spec/integration/shared_connection_thread_safety_spec.rb` prints "Race not triggered this run" instead of failing. That is intended; do not "fix" it into an assertion.
- **Shared database.** Every test and example runs against one `pgmq_test` database. Two suites at once exhaust connections rather than corrupt data.
- **Extension version.** Topic routing and timestamp `set_vt` need PGMQ v1.11.0+; a Postgres image without it changes which tests can pass.

## Conflicts

| File | Rule |
|---|---|
| `Gemfile.lock` | never hand-merge: take the base's, then `bundle install` |
| `Gemfile.lint.lock`, `package-lock.json` | take the base's and let Renovate re-resolve |
| `CHANGELOG.md` | union under the same `## Unreleased` heading, merging the `### <Area>` subheadings rather than repeating them |
| `lib/pgmq/version.rb` | take the base's; a release bumps it, a feature branch does not |
| `README.md` API Reference | append the new section in the order the Table of Contents lists, and update the ToC in the same edit |
| `test/support/database_helpers.rb` | append helpers; never reorder — every spec file depends on the `before`/`after` pair |

## Verification

- The manual check: `docker compose up -d`, then `bundle exec ruby -Ilib -rpgmq -e` with a five-line produce/read/delete against `localhost:5433/pgmq_test`, and read the raw values that come back (Strings, `"t"`/`"f"`).
- Anything touching `Connection` also runs `bin/integrations spec/integration/shared_connection_detection_spec.rb` and `…/shared_connection_thread_safety_spec.rb`.
- Stress iterations for a flake proof: 50 runs of the single test (`bundle exec ruby -Ilib:test <file> -n /name/` in a loop); the pool and thread tests are where a race shows.
- Evidence goes in `lode/tmp/` (git-ignored) unless the PR needs an auditable trail.
