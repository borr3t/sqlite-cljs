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
| 2 | `core` (the one interop namespace: `db`/`close!`, pragmas, versions, value binding, statement cache, error mapping) and `query` (`query-all!`/`query-one!`/`query-value!`/`execute!`/`execute-all!`/`exec!`/`transact!`) | done |
| 3 | `ddl` (create-table/index/view/trigger, alter, drop, STRICT validation), `dsl` (select/insert/upsert/update/delete renderer, json_each IN lists, chunked bulk inserts) and `migrate` (`user_version` + `sqlite_master` introspection, idempotent apply with drift detection) | done |
| 4 | live smoke (`dev/sqlite_cljs/smoke.cljs`, `:smoke` build): a real file database in a temp dir + `:memory:` side by side, 11 PASS/FAIL sections 0–10, no network | done |
| 5 | user manual (`docs/sqlite-cljs.org`) + README | done |

`./run-tests.sh` — 88 tests / 560 assertions, no network; database tests use `:memory:` plus one
temporary file under `target/`. `npx shadow-cljs compile smoke && node target/smoke.js` — the live
smoke (11/11 sections, exit 0 only when every section passes).

## Documentation

- **User manual** — [`docs/sqlite-cljs.org`](docs/sqlite-cljs.org): install and quick start,
  the db value and effective pragmas, the DDL DSL (STRICT default, constraints, indexes,
  views/triggers, alters with version gates), the query DSL (render model, operator table,
  joins, CTEs, upserts, `:raw`, big IN lists, chunking), `migrate!`, executing queries,
  transactions, value binding, error handling, design decisions, tests and the live smoke,
  and a full API index per namespace.
- **API inventory** — [`docs/sqlite-api-inventory.org`](docs/sqlite-api-inventory.org): the
  verified spec behind the library (driver choice, SQLite grammar census, PRAGMAs, DSL
  design, pitfalls and uncertainties with house positions, the as-shipped API in section 9).
  Manual references such as "inventory 7.5" point there.
- Namespace and function docstrings in `src/sqlite_cljs/` are the API truth; the manual's
  examples are rendered/run against them.

## Quick look

```clojure
(require '[sqlite-cljs.core :as core] '[sqlite-cljs.query :as q]
         '[sqlite-cljs.migrate :as migrate] '[promesa.core :as p])

(def d (core/db "app.db"))                ; WAL, foreign_keys on, busy_timeout 5000, STRICT by default
(p/let [_ (migrate/migrate! d [{:create-table :users
                                :columns [[:id :integer {:primary-key? true}]
                                          [:email :text {:not-null? true :unique? true :collate :nocase}]
                                          [:active :integer {:not-null? true :default true}]      ; booleans are 0/1
                                          [:created-at :integer {:not-null? true                   ; dates are epoch ms
                                                                 :default [:raw "(unixepoch('subsec') * 1000)"]}]]}])
        _ (q/execute! d {:insert-into :users :values [{:email "a@x" :active true}]
                         :on-conflict [:email] :do-update-set [:active] :returning [:id]})
        rows (q/query-all! d {:select [:id :email :active] :from :users :where [:and [:= :active true] [:like :email "%@x"]]}
                            {:coerce {:active :boolean}})]
  rows)
;; => [{:id 1 :email "a@x" :active true}]
```

Every `:where` value is a `?` parameter; `{:select .. :where [:= :name "x'; DROP TABLE users; --"]}` binds
the string. The driver is synchronous, so the promises are already settled when returned; inside a
`transact!` body (which runs synchronously) deref them: `@(q/execute! tx q)`.

## Hard style rules

* ClojureScript-only API: no `^js` hints and no JavaScript interop at call
  sites. All driver interop lives in `sqlite-cljs.core` (property access via
  `unchecked-get`, calls via `js-invoke`; no type hints anywhere).
* Kebab-case keyword maps in and out (`:user-name` <-> `user_name`).
* Async operations return promesa promises.
* Zero runtime npm dependencies beyond the wrapped driver package.
* SQL-injection safety: every value is a bound parameter, never interpolated;
  every identifier goes through `util/sql-ident` (`"…"` with `""` doubling).
* The DSL is data: `{:create-table :users ...}`, `{:select [...] :from ... :where [...]}`,
  see inventory §6 for the exact shapes and the namespace docstrings of
  `sqlite-cljs.dsl` / `sqlite-cljs.ddl` / `sqlite-cljs.migrate` for the implemented grammar.
* Tables are `STRICT` by default; `:boolean`/`:datetime` column types are rejected
  (INTEGER 0/1 and INTEGER epoch milliseconds are the house conventions);
  legacy type names only via `[:raw "VARCHAR(255)"]` with `:strict? false`.

## Development

```
./run-tests.sh                                          # exec's node at the end — never chain it mid-command
npx shadow-cljs compile smoke && node target/smoke.js   # live smoke: temp-dir file db + :memory:, 11 sections, no network
npx shadow-cljs watch dev && node target/dev.js         # REPL runtime
```
