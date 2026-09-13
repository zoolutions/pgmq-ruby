# Review rules: runner scripts and Rake tasks

Accepted review findings from merged PR threads on `mensfeld/pgmq-ruby`, rewritten as rules about the system and verified against this tree.

### An interrupted example run exits 130; it is not reported as a failure
- **Holds because:** `system` returns three things, and two of them are falsy. `nil` means the child was signalled (Ctrl-C), `false` means it exited non-zero. Testing `unless success` folds a user's Ctrl-C into the failure list and prints a red summary for something nobody broke. The runner branches on `success.nil?` first and `exit(130)`, and only then records a failure (`bin/integrations:39-45`); its own `Signal.trap("INT")` exits 130 as well (`bin/integrations:16-19`).
- **Where:** `bin/integrations` (33-48)
- **Safe direction:** treating a real failure as an interrupt would hide a broken example, so the `nil` test is narrow and everything else falls through to the failure branch.
- **Proven by:** no test — `bin/integrations` is a script CI invokes, and nothing exercises its exit codes.
- **Origin:** PR #47 review thread

### Rake tasks resolve `bin/integrations` against the Rakefile, not the working directory
- **Holds because:** `exec("bin/integrations")` only works when rake was started from the repository root, and rake can be run from anywhere. Both example tasks build the path with `File.expand_path("bin/integrations", __dir__)` (`Rakefile:20`, `:34`), matching how `examples_dir` is resolved (`:25`, `:39`).
- **Where:** `Rakefile#examples:run` (18-21), `#examples:run_one` (23-35)
- **Proven by:** no test.
- **Origin:** PR #86 review thread

### The example helper restores the SIGINT handler by the type `Signal.trap` returned
- **Holds because:** `Signal.trap` hands back a `String` (`"DEFAULT"`, `"IGNORE"`, `"SYSTEM_DEFAULT"`), a `Proc`, or `nil`, and the three need different restore calls — a `Proc` must be passed with `&`, a String positionally. A single `|| "DEFAULT"` fallback silently discards a previous handler. The `ensure` in `run_example` is a three-branch `case` (`spec/integration/support/example_helper.rb:80-84`).
- **Where:** `spec/integration/support/example_helper.rb#run_example` (55-88)
- **Proven by:** no test.
- **Origin:** PR #47 review thread

### Not a rule here: the `ExampleHelper` interrupt flag is an unsynchronised local
- **Holds because:** `run_example` sets a plain local from inside the signal handler and exposes it as `-> { interrupted }` (`spec/integration/support/example_helper.rb:63-72`), with no mutex or atomic. A reviewer flagged the visibility question; the design was left as it is. It is deliberate for what it drives — examples poll the lambda between iterations, and the worst outcome of a missed update is one extra loop before the script stops — so a change that introduces a mutex here is fixing something that is not broken. It would matter if an example ever gated something irreversible on the flag.
- **Where:** `spec/integration/support/example_helper.rb#run_example` (63-72)
- **Origin:** PR #47 review thread (raised, not acted on)

### Not a rule here: `FailOnSeverity: convention` weakens documentation enforcement
- **Holds because:** it reads like a downgrade and is the opposite. `FailOnSeverity` is a *threshold*, not a filter: `convention` is the lowest of the three severities yard-lint emits, so failing at `convention` fails at `warning` and `error` too. Setting it back to `error` — the change a reviewer proposed for exactly this reason — would stop the build failing on the `convention`-severity validators (`Documentation/EmptyCommentLine`, `Documentation/BlankLineBeforeDefinition`, `Tags/RedundantParamDescription`) and on the two `warning`-severity ones (`Tags/ExampleSyntax`, `Tags/InformalNotation`). The tree keeps `convention` (`.yard-lint.yml:15`) on purpose.
- **Where:** `.yard-lint.yml:15`
- **Origin:** PR #27 review thread (raised, deliberately not acted on)
