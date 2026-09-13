# Tests, examples and CI

## Two suites that are not the same thing

| | Unit suite | Integration examples |
|---|---|---|
| Where | `test/lib/**/*_test.rb` (17 files) | `spec/integration/*_spec.rb` (19 scripts) |
| Framework | Minitest/Spec (`describe`/`it`) + Mocha | none — plain Ruby scripts that raise on failure |
| Runner | `bundle exec rake test` (`Rakefile:8-13`) | `bin/integrations` (`Rakefile:17-51` wraps it) |
| Needs Postgres | **yes, all of it** | yes |

Both need a live PostgreSQL with the `pgmq` extension. `docker compose up -d` starts `ghcr.io/pgmq/pg18-pgmq:latest` on **port 5433** — chosen so it cannot collide with a system PostgreSQL. `test/test_helper.rb:24-30` and `spec/integration/support/example_helper.rb:15-21` both default to `localhost:5433/pgmq_test` as `postgres/postgres`, each overridable through `PG_HOST`, `PG_PORT`, `PG_DATABASE`, `PG_USER`, `PG_PASSWORD`.

There is no stubbed-database tier. Even `client_test.rb`'s queue-name validation cases build a real client. Mocha appears in exactly two files: `transaction_test.rb`, split into `context "with mocked connections"` (`:4`) and `context "with real database"` (`:90`); and `connection_test.rb:352-362`, which stubs `status` and expects `reset` on a real `PG::Connection` to drive `verify_connection!` down both branches.

## Test helpers

`test/test_helper.rb` starts SimpleCov **before** loading the gem, sets `minimum_coverage 96.5`, filters `/test/`, `/spec/`, `/examples/`, `/vendor/`, aliases `context` to `describe`, and mixes `JSONHelpers` and `DatabaseHelpers` into every `Minitest::Spec`.

`test/support/database_helpers.rb` supplies the shape every suite uses:

- `create_test_client(**options)`, `unique_queue_name(suffix = nil)` (a `SecureRandom.hex(4)` name, so tests never collide), `ensure_test_queue` (which swallows `PG::DuplicateTable`), and `wait_for(timeout: 5, &block)`, a 0.1s poll loop that raises on timeout.
- `setup_client_and_queue` / `teardown_client_and_queue` — the standard `before`/`after` pair. The teardown drops the queue, swallows the failure, and closes the client in an `ensure`.
- `pgmq_version` and `pgmq_supports_v1_11_features?` (aliased `pgmq_supports_set_vt_timestamp?` and `pgmq_supports_topic_routing?`), which read `pg_extension.extversion` so a test can skip features the installed extension predates.

`spec/integration/support/example_helper.rb` is the equivalent for examples: `ExampleHelper.run_example(name)` creates a client, collects queue names for cleanup, installs a SIGINT handler that flips an `interrupted` flag, and in its `ensure` restores the previous handler by type (`String`, `Proc`, else `"DEFAULT"`), cleans the queues and closes the client. `bin/integrations` runs each script as a subprocess with `-r example_helper`, so a spec file never requires the helper itself.

## Timing in tests

Nineteen `sleep` calls across `test/` wait out a visibility timeout or a poll (`sleep 2.5` after a `vt: 2` read at `test/lib/pgmq/client/consumer_test.rb:276`; a producer thread sleeping `0.5` before a long-poll assertion at `:288`). The house pattern is *the timeout plus a 0.5-second buffer* — `vt: 2` → `sleep 2.5`, `vt: 3` → `sleep 3.5` (`consumer_test.rb:219-227`) — so a new wait picks the shortest timeout that proves the point rather than a round number. These are real waits against a real server, so the suite is slow by construction and sensitive to a loaded machine.

## CI

`.github/workflows/ci.yml` runs on pull requests targeting `master` and nightly at 01:00 UTC, with in-progress runs cancelled per ref. Four jobs feed a `ci-success` gate that fails if any of them failed, was cancelled **or was skipped**:

- `tests` — a 3 × 5 matrix of Ruby `4.0.0`/`3.4`/`3.3` against Postgres `14`–`18` (`ghcr.io/pgmq/pg<N>-pgmq:latest` as a service on 5433) — 15 cells. The `include:` block does not add a sixteenth; it matches the Ruby 3.4 × Postgres 18 cell and adds `coverage: 'true'` to it, which the test step passes through as `GITHUB_COVERAGE`. It deletes `Gemfile.lock` on the Ruby 4.0 rows, creates the extension with `psql`, then runs `bundle exec rake test` and `bin/integrations`.
- `yard-lint` — `bundle exec yard-lint lib/` under `BUNDLE_GEMFILE=Gemfile.lint`.
- `rubocop` — `bundle exec rubocop` under the same lint bundle.
- `lostconf` — `npx lostconf --fail-on-stale`, the only Node step; `package.json` exists for it alone.

Both lint jobs pin Ruby `4.0.2`, matching `.ruby-version`. Every action is pinned to a commit SHA with the version in a trailing comment; Renovate (`renovate.json`) keeps them current and is restricted to `includePaths` — `.ruby-version`, the two Gemfiles, the gemspec, `.github/workflows/**`, `docker-compose*.yml`, `package.json` — with a 7-day `minimumReleaseAge`.

## Lint

There is no lint task in the `Rakefile`, and the default gem bundle has no linter in it. Both linters live in the second bundle: `BUNDLE_GEMFILE=Gemfile.lint bundle exec rubocop` and `… yard-lint lib/`. `.rubocop.yml` loads the `rubocop-minitest` and `rubocop-performance` plugins, inherits `standard`, `standard-performance` and `standard-minitest`, targets Ruby 3.3, and overrides exactly two cops: `Layout/LineLength: Max: 120` and `Layout/SpaceInsideHashLiteralBraces: EnforcedStyle: space`. `.yard-lint.yml` demands `RequiredCoverage: 100` on `lib/` with `FailOnSeverity: convention`, so an undocumented method or a missing `@param` fails CI — which is why every public method in `lib/` carries a full YARD block with `@example`.

## Release

`.github/workflows/push.yml` fires on `v*` tags and publishes through `rubygems/release-gem`, but only when `github.repository_owner == 'mensfeld'`. A tag pushed on the `zoolutions` fork publishes nothing. `DEVELOPMENT.md`'s release steps (bump `lib/pgmq/version.rb`, update `CHANGELOG.md`, tag, push) describe the upstream flow.
