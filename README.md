# sqlite-cljs

ClojureScript (Node.js) client for SQLite. The point of the library is a
**data-driven DSL** (honeysql-inspired, SQLite dialect): ClojureScript maps and
vectors describe tables, indexes, views and queries, and the library renders
the DDL/SQL from that data. Values are *always* bound parameters, identifiers
are *always* strictly quoted, so SQL is never assembled from strings at call
sites.

The authoritative spec is `docs/sqlite-api-inventory.org` (driver choice,
SQLite grammar census, DSL design, pitfalls). The wrapped driver is
[`better-sqlite3`](https://github.com/WiseLibs/better-sqlite3) (synchronous,
N-API prebuilt binaries); it is the only npm runtime dependency and is required
from exactly one namespace. Public functions return promesa promises (resolved
from the driver's synchronous calls, documented as such); kebab-case keyword
maps in and out.

## Status

| Phase | Content | State |
|-------|---------|-------|
| 0 | scaffold: shadow-cljs `:node-test` / `:dev` builds, `run-tests.sh`, `util` (kebab/snake, strict identifier quoting, error envelope) | done |
| 1 | research inventory (`docs/sqlite-api-inventory.org`): driver decision, DDL/DML grammar census, PRAGMAs, DSL syntax proposal, pitfalls, uncertainties | done |
| 2 | `driver` (the one interop namespace), `core` (open!/close!, query!/execute!, transact!, pragmas, introspection) | planned |
| 3 | `ddl` (create-table/index/view, alter, drop) and `query` (select/insert/upsert/update/delete renderer) | planned |
| 4 | `migrate` (`user_version` + `sqlite_master` introspection, idempotent apply) | planned |
| 5 | live smoke against a file database (WAL, busy timeout, two connections), README quick start | planned |

`./run-tests.sh` — 7 tests / 74 assertions, no network, no database file.

## Hard style rules

* ClojureScript-only API: no `^js` hints and no JavaScript interop at call
  sites. All interop lives in `sqlite-cljs.driver`.
* Kebab-case keyword maps in and out (`:user-name` <-> `user_name`).
* Async operations return promesa promises.
* Zero runtime npm dependencies beyond the wrapped driver package.
* SQL-injection safety: every value is a bound parameter, never interpolated;
  every identifier goes through `util/sql-ident` (`"…"` with `""` doubling).
* The DSL is data: `{:create-table :users ...}`, `{:select [...] :from ... :where [...]}`,
  see inventory §6 for the exact shapes.

## Development

```
./run-tests.sh                                  # exec's node at the end — never chain it mid-command
npx shadow-cljs watch dev && node target/dev.js # REPL runtime
```
