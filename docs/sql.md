# BoilaSQL

BoilaSQL is a **documented PostgreSQL subset**. Anything outside this
page returns `0A000 feature_not_supported` with the construct name —
never a silent semantic difference.

Encoding: **UTF-8 only**. `server_encoding` / `client_encoding` are
`UTF8`. Another encoding → `0A000`. Identifiers are bare (`users`);
there are no double-quoted identifiers.

## Types

| SQL type | Storage tag | Notes |
|----------|-------------|--------|
| `BOOL` / `BOOLEAN` | 1 | |
| `BIGINT` / `INT8` | 2 | 64-bit signed |
| `TIMESTAMPTZ` | 3 | microseconds |
| `BYTEA` | 4 | binary-safe |
| `TEXT` | 5 | raw UTF-8 bytes |
| `JSONB` | 6 | document-as-column (no separate doc store) |
| `VECTOR(n)` | 1000+n | fixed-point ×1e6 i32; see [modalities.md](modalities.md) |
| `NUMERIC` / `DECIMAL` | 9 | bagadecimal 96-bit + scale; PG OID 1700 |

`FLOAT8` is reserved as tag 7 in the value codec (payload only, no
sort-order — gaps V1) but `CREATE TABLE` does **not** accept it yet.
Dual/expression arithmetic is integer-only (no floats).

`NULL` is three-valued: `NULL = NULL` is never true. `IS [NOT] NULL`
uses the secondary index when present.

**P12:** `NUMERIC` / `DECIMAL` (unconstrained; bagadecimal). Insert
via text (`'12.50'`) or bigint; `CAST` / `::numeric`.

**P20-3:** `SERIAL` / `BIGSERIAL` — auto-number PRIMARY KEY (bigint
underneath; the column must be the `PRIMARY KEY`).

**Not yet:** `REFERENCES`, arrays, usable `FLOAT8` columns,
`NUMERIC(p,s)` enforcement, unquoted `12.50` literals, `CHECK`
constraints.

Every `CREATE TABLE` requires a `PRIMARY KEY`.

## Databases

```sql
CREATE DATABASE name;
CREATE DATABASE IF NOT EXISTS name;
DROP DATABASE name;
DROP DATABASE IF EXISTS name;
USE name;                    -- MySQL convenience; PG startup already picks db
```

One session = one database. Switching mid-transaction → `0A000`.
`DROP` of the session’s current database → `55006`. Default database
after init: `boila`.

**P18:** `SELECT` may qualify `db.table` (or `db.public.table`) and
JOIN across local databases without `USE`. DML against a foreign
database → `0A000` (no 2PC). Local FDW aliases:

```sql
CREATE SERVER name OPTIONS (database 'other');
CREATE FOREIGN TABLE t SERVER name;   -- t → other.t
```

## DDL

```sql
CREATE TABLE [IF NOT EXISTS] t (
  id    BIGINT,
  name  TEXT NOT NULL,
  kind  TEXT NOT NULL DEFAULT 'std',
  ts    TIMESTAMPTZ DEFAULT now(),
  body  TEXT,
  PRIMARY KEY (id)
) [WITH (ttl_days = N | ttl_sec = N)];

ALTER TABLE t ADD COLUMN col TEXT;          -- nullable add-only
ALTER TABLE t RENAME TO u;
ALTER TABLE t RENAME [COLUMN] a TO b;
ALTER TABLE t DROP COLUMN col;              -- last non-PK only; CASCADE drops ix/FTS/HNSW

DROP TABLE [IF EXISTS] t;
TRUNCATE [TABLE] [IF EXISTS] t;             -- wipe data + modality CFs; keep schema

CREATE [UNIQUE] INDEX [IF NOT EXISTS] name ON t (col);
CREATE INDEX [IF NOT EXISTS] name ON t USING hnsw (emb);
CREATE FTS INDEX [IF NOT EXISTS] name ON t (body);
CREATE GRAPH [IF NOT EXISTS] name ON edges (src, dst [, w]);
CREATE ROLLUP [IF NOT EXISTS] name ON t USING time_bucket('1m', ts) [SUM(col)];

DROP INDEX [IF EXISTS] name ON t;           -- ON is required
DROP GRAPH [IF EXISTS] name ON t;
```

`SHOW TABLES` · `SHOW INDEX[ES] FROM t` · `SHOW COLUMNS FROM t` /
`DESCRIBE t` / `DESC t` · `SHOW CREATE TABLE t`.

**UNIQUE (P20-1):** `CREATE UNIQUE INDEX` rejects a duplicate non-NULL
value with `23505` on INSERT / UPDATE / `ON CONFLICT DO UPDATE`. NULLs
are distinct (many NULL rows are allowed, as in PostgreSQL). Creating a
unique index on a column that already contains duplicates fails with
`23505` and rolls back. `UNIQUE` is not accepted with `USING hnsw`.
Single-column only.

**Column constraints (P20-2):** `NOT NULL` and `DEFAULT` follow the
column type in any order. `DEFAULT` accepts a string / integer /
boolean literal matching the column type, or `now()` on a
`timestamptz` column (type mismatch → `42804` at CREATE). INSERT fills
omitted columns with their defaults before checking `NOT NULL`; a NULL
in a NOT NULL column → `23502` (also on UPDATE / `ON CONFLICT DO
UPDATE`). `ALTER TABLE ADD COLUMN` remains nullable-only.

**SERIAL / BIGSERIAL (P20-3):** an auto-number PRIMARY KEY. INSERT
that omits the column gets the next per-table counter value
(`RETURNING` shows it); multi-row INSERTs take consecutive values.
The counter rides the transaction buffer — a ROLLBACK gives the number
back (unlike PostgreSQL sequences). Explicit values are allowed and do
not advance the counter. A SERIAL column must be the PRIMARY KEY
(`0A000` otherwise) and cannot carry a DEFAULT (`42601`).

`TABLE t [WHERE …] [ORDER BY …] [LIMIT …]` is sugar for
`SELECT * FROM t …`.

## DML

```sql
INSERT INTO t (c1, c2) VALUES (1, 'a'), (2, 'b')
  [ON CONFLICT DO NOTHING]
  [ON CONFLICT DO UPDATE SET c2 = EXCLUDED.c2, c3 = c3 + 1]
  [RETURNING * | col, …];

UPDATE t SET c2 = upper(c2), c3 = c3 + 1
  WHERE id = 1 OR length(c2) = 3
  [RETURNING …];

DELETE FROM t WHERE id IN (1, 2) [RETURNING …];
```

`ON CONFLICT` SET may mix literals and expressions. `EXCLUDED.col` is
the proposed insert row (PG semantics). `UPDATE SET` expressions see
the **old** row.

## SELECT

```sql
SELECT [ALL|DISTINCT] items
  [FROM t]
  [JOIN u ON t.a = u.b]          -- one INNER or LEFT; ON is equality (+ AND eqs)
  [WHERE pred]
  [GROUP BY cols|exprs]
  [HAVING pred]
  [ORDER BY col|alias [ASC|DESC] [NULLS FIRST|LAST]]
  [LIMIT n | LIMIT ALL | FETCH FIRST/NEXT n]
  [OFFSET n];
```

**WHERE** (structural, can use indexes):

- `=` `<>` `!=` `<` `<=` `>` `>=`
- `BETWEEN` / `NOT BETWEEN`
- `IN` / `NOT IN`
- `LIKE` / `ILIKE` / `NOT LIKE` (`%` `_`, optional `ESCAPE`)
- `IS [NOT] NULL`
- `AND` of the above
- `col = a OR col = b` rewrites to `IN`

**WHERE** (expression span — sequential filter, or AND-tail after a
structural predicate): arithmetic, `||`, functions, `CASE`, `CAST`/`::`,
`AND`/`OR`/`NOT`, `IS [NOT] DISTINCT FROM`. Top-level `OR` of mixed
columns is a seq-filter.

**Window (P13):** `ROW_NUMBER()` / `RANK()` / `DENSE_RANK()` /
`SUM`/`AVG`/`COUNT`/`MIN`/`MAX(col)` `OVER ( [PARTITION BY cols]
[ORDER BY cols [ASC|DESC]] )`. Default frame = RANGE UNBOUNDED
PRECEDING (ties share the running agg). One OVER spec per query.
`ROWS`/`RANGE`/`GROUPS` → `0A000`. No `LAG`/`LEAD`/`NTILE`, no
named `WINDOW`.

**COPY (P14):** `COPY t [(cols)] FROM STDIN | TO STDOUT`
`[WITH (FORMAT text|csv)]`. Text is tab-separated (`\N` null, `\.`
end). CSV via csvbaga. Bad row → `22P04` and the implicit txn
rolls back. No `FROM 'file'`, no binary, no `HEADER`.

**Not yet:** subqueries, more than one JOIN, `INTERSECT`/`EXCEPT ALL`
on tables (set ops exist on dual only).

## Dual (`SELECT` without `FROM`)

Literals, session builtins, scalar functions, arithmetic, `CASE`,
`CAST`/`::`, `VALUES`, `generate_series`, `UNION` / `INTERSECT` /
`EXCEPT`, `DISTINCT`, `ORDER BY`, `LIMIT`/`OFFSET`, `WHERE` on aliases.

```sql
SELECT 1 + 2 AS n, current_database(), now();
SELECT * FROM generate_series(1, 5);
VALUES (1, 'a'), (2, 'b');
```

Bare keywords (no `()`): `current_user`, `session_user`, `user`,
`current_database`, `current_schema`, `version`, `current_date`, `now`.

## Scalar functions

Unknown name → `42883`. Nested calls and expression arguments work in
projection / dual / WHERE expression spans.

| Function | Result |
|----------|--------|
| `length` / `char_length` / `character_length` | bigint (bytes, not graphemes) |
| `upper` / `lower` / `trim` / `btrim` / `reverse` | text (`upper` is ASCII-only) |
| `substr` / `substring` / `left` / `right` / `replace` / `repeat` / `lpad` / `rpad` | text |
| `strpos` / `abs` / `sign` / `mod` / `power` / `pow` | bigint |
| `to_char(timestamptz, fmt)` | text (P20-5; `YYYY MM DD HH24 HH12 HH MI SS`) |
| `starts_with` / `ends_with` | boolean |
| `coalesce` / `nullif` / `greatest` / `least` / `concat` | first-arg type or text |
| `quote_literal` / `quote_ident` / `quote_nullable` / `pg_typeof` | text |
| `version` / `current_*` / `now` / `current_setting` / `pg_backend_pid` / `pg_is_in_recovery` | session |

`avg` over BIGINT is **integer** division (`sum/cnt` in i64) — gaps A1.
Over NUMERIC it is decimal (P20-4).

## Aggregates

`COUNT(*)` / `COUNT(col)` / `SUM` / `AVG` / `MIN` / `MAX`, including
`agg(expr)` and `GROUP BY expr`. `HAVING` is a boolean tree (`AND`
tighter than `OR`, parentheses). Mix of agg and row expressions on the
group’s first row.

**NUMERIC (P20-4):** `SUM` / `AVG` / `MIN` / `MAX` over a NUMERIC
column stay decimal-exact — the result is NUMERIC (PG type 1700), no
i64 truncation. `AVG` keeps at least 6 decimal places. NULLs are
skipped.

No `DISTINCT` inside aggregates.

## Transactions

```sql
BEGIN [READ ONLY];
BEGIN ISOLATION LEVEL SERIALIZABLE [READ ONLY];
BEGIN ISOLATION LEVEL REPEATABLE READ;
COMMIT;          -- alias: END
ROLLBACK;        -- alias: ABORT
VACUUM;          -- flush + full compact; MVCC GC (P16)
```

Default / `READ COMMITTED` is latest-read + write buffer (no SSI).
`REPEATABLE READ` sees versions with `lsn ≤` snapshot. `SERIALIZABLE`
is the same plus commit check: a concurrent write of a key this txn
read or wrote → `40001`. `DEFERRABLE` → `0A000`. No predicate locks.

`BEGIN` in an open txn → `25001`. Writes in `READ ONLY` → `25006`.
Buffer overflow → `54000` (`BOILA_TXN_MAX`).

## Session GUC

```sql
SET name TO value;     -- or SET name = value
SHOW name;
RESET [name | ALL];
DISCARD ALL;           -- clears GUC; on PG also prepared statements
SET ROLE name [PASSWORD '…'];
SET ROLE NONE;         -- aliases: RESET ROLE, SET SESSION AUTHORIZATION
```

Defaults: `server_version`, encodings `UTF8`, `DateStyle`, `TimeZone`.
No real GUC side-effects (you cannot change encoding).

`EXPLAIN` / `EXPLAIN ANALYZE` print a text plan; `ANALYZE` adds actual
rows and execution time in ms.

## SQLSTATE (common)

| Code | When |
|------|------|
| `0A000` | Feature not supported |
| `22023` / type errors | Bad literal / dimension |
| `23505` | Unique violation |
| `25001` | Already in a transaction |
| `25006` | Read-only transaction |
| `28P01` | Password authentication failed |
| `3D000` | Unknown database (DROP IF EXISTS → no-op) |
| `42P01` | Undefined table |
| `42P04` | Duplicate database |
| `42P07` | Duplicate table (IF NOT EXISTS → no-op) |
| `42710` | Duplicate object (index/FTS/…) |
| `42803` | Grouping error |
| `42883` | Undefined function |
| `53300` | Too many connections |
| `54000` | Program limit (budget scan/rows, txn buffer) |
| `55006` | Drop current database |
| `40001` | Serialization failure (P17) |
| `57014` | Query canceled (wall deadline) |
