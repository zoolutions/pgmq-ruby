# Review rules: YARD documentation blocks

Accepted review findings from merged PR threads on `mensfeld/pgmq-ruby`, rewritten as rules about the system and verified against this tree. Documentation is a build step here — `yard-lint` with `RequiredCoverage: 100` and `FailOnSeverity: convention` gates CI — so these are code rules, not style preferences.

### An `@example` stays inside the API of the object it documents
- **Holds because:** the example is the first thing a reader runs, and one that mixes two objects' APIs cannot be pasted anywhere. A `Connection` method's example that calls `client.read(...)` and then `connection.reload` documents neither object; the accepted fix kept the whole example inside `connection.with_connection { |conn| conn.exec(...) }`. The same rule is why `Connection`'s class-level examples (`lib/pgmq/connection.rb:14-21`) construct a `PGMQ::Connection` and `Client`'s (`lib/pgmq/client.rb:20-25`, `51-61`) construct a `PGMQ::Client`.
- **Where:** every `@example` in `lib/`
- **Origin:** PR #142 review thread (accepted: "fixed in bceec49")

### An `@example` is parsed as Ruby, so it must also be callable
- **Holds because:** `Tags/ExampleSyntax` (`.yard-lint.yml:141-144`) runs the YARD parser over every `@example`, so a syntax error fails the lint — but syntactic validity is the floor, not the bar. The arity mistakes reviewers catch start in examples: an example showing `read_batch(queue, 5)` teaches a call that raises `ArgumentError`, and nothing in the toolchain checks arity. Write the example as the call you would actually make, with the keywords the signature declares.
- **Where:** every `@example` in `lib/`; the signatures they must match are in `lib/pgmq/client/`
- **Safe direction:** an example that is more explicit than necessary costs a line; one that is wrong is copied.
- **Origin:** PR #59 review thread (three arity corrections), PR #17 (`.yard-lint.yml` wording)

### Prose that opens a line with `Note:` or `Returns:` fails the lint
- **Holds because:** `Tags/InformalNotation` (`.yard-lint.yml:177-197`) maps `Note:`, `Returns:`, `Raises:`, `Example:`, `See:`, `Todo:`, `Warning:`, `Deprecated:`, `Author:`, `Version:` and `Since:` to their tags, with `RequireStartOfLine: true` and `CaseSensitive: false`. Severity is `warning`, which `FailOnSeverity: convention` still fails on. Use the tag.
- **Where:** `.yard-lint.yml`, every doc block in `lib/`
- **Origin:** PR #27 review thread (the config change that made `convention` the failure threshold)
