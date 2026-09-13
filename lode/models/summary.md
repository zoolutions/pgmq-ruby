# Models, errors and the raw-String contract

## Nothing is converted

`create_connection` (`lib/pgmq/connection.rb:232-243`) does not set `conn.type_map_for_results`, and the comment above the `PG.connect` call says the omission is deliberate: "Low-level library: return all values as strings from PostgreSQL / No automatic type conversion - let higher-level frameworks handle parsing / conn.type_map_for_results intentionally NOT set" (`:237-239`). Everything downstream follows from that.

- A message id is `"123"`, a read count `"1"`, a timestamp `"2025-01-15 10:30:00+00"`.
- A boolean column is `"t"` or `"f"`. Exactly six methods turn that into a Ruby boolean by comparing to `"t"`: `drop_queue` (`queue_management.rb:94`), `delete` (`message_lifecycle.rb:73`), `archive` (`:167`), `unbind_topic` (`topics.rb:64`), `validate_routing_key` (`:237`) and `validate_topic_pattern` (`:264`). Every other boolean is handed through as the `"t"`/`"f"` String — `QueueMetadata#partitioned?` and `#unlogged?` are aliases, not predicates, and their YARD says so (`lib/pgmq/queue_metadata.rb:29-35`).
- A JSONB payload is its unparsed String. `Message#message` is the raw text; callers `JSON.parse` it themselves, which is what every test and example does.
- Counts come back as Strings too. `purge_queue` returns `"3"`, not `3` (`test/lib/pgmq/client/maintenance_test.rb:14`).

Three YARD `@return` tags contradict this: `maintenance.rb:13` promises `Integer` for `purge_queue`, and `message_lifecycle.rb:80` / `:174` promise `Array<Integer>` for `delete_batch` / `archive_batch`. All three return Strings. The code, not the tag, is the contract. The fourth `Integer` tag in `lib/`, `topics.rb:76`, is correct: `produce_topic` is the single method that coerces (`topics.rb:106`, `.to_i`).

A practical consequence: tests compare timestamps as Strings (`assert_operator updated_msg.vt, :>, msg.vt`, `test/lib/pgmq/client/message_lifecycle_test.rb:178`), which works only because PostgreSQL's `timestamptz` text output sorts lexicographically in a fixed format. No test parses a time.

## The three row models

All three are `Data.define` subclasses with a class-level `new(row, **)` that reads the row by String key and calls `super` with keywords. A missing column yields `nil` rather than raising, which is how one class covers several queries.

| Model | Fields | Notes |
|---|---|---|
| `Message` | `msg_id, read_ct, enqueued_at, last_read_at, vt, message, headers, queue_name` | `alias_method :id, :msg_id` (`message.rb:45`). `queue_name` is populated only by `read_multi`/`pop_multi`, which select it as a literal `'<name>'::text as queue_name` column; it is `nil` for single-queue reads. `headers` is `nil` unless the message carried them. |
| `Metrics` | `queue_name, queue_length, newest_msg_age_sec, oldest_msg_age_sec, total_messages, scrape_time` | built by `metrics` and `metrics_all` |
| `QueueMetadata` | `queue_name, created_at, is_partitioned, is_unlogged` | `partitioned?` / `unlogged?` are aliases, not predicates — they return `"t"`/`"f"` |

The `Topics` listing methods do not use a model: `list_topic_bindings`, `test_routing` and `produce_batch_topic` return Arrays of Hashes with Symbol keys (`topics.rb:186-192`, `213-215`, `157-159`).

## Errors

`PGMQ::Errors` (`lib/pgmq/errors.rb`) is flat: `BaseError < StandardError` and seven subclasses — `ConnectionError`, `QueueNotFoundError`, `MessageNotFoundError`, `SerializationError`, `DeserializationError`, `ConfigurationError`, `InvalidQueueNameError`. `test/lib/pgmq/errors_test.rb` asserts each one's superclass.

Only three of the seven are ever raised by this gem:

- `InvalidQueueNameError` — from `validate_queue_name!` only.
- `ConfigurationError` — nil or malformed connection params, a bad connection string, and the shared-connection guard.
- `ConnectionError` — every failure that reaches a caller through `Connection#with_connection` or `Transaction#transaction`, including pool timeouts and a closed pool. A `PG::Error` from a bad query arrives here too, so a SQL-level failure and a dead socket are the same class to a caller; the message is what distinguishes them.

`QueueNotFoundError`, `MessageNotFoundError`, `SerializationError` and `DeserializationError` are defined and tested but never raised in `lib/` — a grep for `Errors::` across `lib/` finds only the other three — they exist for callers and for the planned framework gem. `ArgumentError` (plain Ruby) is what the `*_multi` and batch methods raise for shape violations, not a `PGMQ::Errors` class.
