# Database rules — MySQL 8.0+ (InnoDB)

Read with `backend.md` §4 §6 §7, `security.md` §14, `performance.md` §3. Every rule is MUST unless marked SHOULD.
**MariaDB:** if MariaDB is also a supported target, every migration and query runs in CI on both; differences are listed in §14.

## 1. Engine and connection

- InnoDB only. Character set `utf8mb4` (never `utf8`/`utf8mb3`, which is 3-byte and drops emoji/some scripts); set the connection charset/collation explicitly.
- **Strict SQL mode always on** (`STRICT_TRANS_TABLES` or stricter, `ONLY_FULL_GROUP_BY`, `NO_ZERO_DATE`, `NO_ZERO_IN_DATE`, `ERROR_FOR_DIVISION_BY_ZERO`); never disabled to "make an import work". Silent truncation and zero-dates are data corruption.
- **UTC only (NON-NEGOTIABLE, `backend.md` §7):** connection session time zone = `'+00:00'` on every connection (an offset, not a named zone: named zones need tz tables that shared hosts may lack); set the server's `default_time_zone = '+00:00'` where the host allows it. The DB never converts zones (`CONVERT_TZ`, `SET time_zone` in queries are forbidden); the app writes UTC.
- TLS required between app and DB (`require_secure_transport`); no anonymous or remote-root accounts; `local_infile` off; application role has DML only (`security.md` §14); a separate role runs migrations.
- Connection limits and pooling sized from measured concurrency; `max_allowed_packet`, `wait_timeout` set deliberately; long-lived connections handle reconnect.

## 2. Types and schema

- **Money:** `DECIMAL(p,s)` (e.g. `DECIMAL(12,2)`) + a currency code column; never `FLOAT`/`DOUBLE`.
- **Time: UTC only, never a local zone.** Prefer `DATETIME(6)` (or `TIMESTAMP(6)` knowing it ends in 2038 and converts by session zone); calendar-only values use `DATE`. Default/`ON UPDATE CURRENT_TIMESTAMP` follow the UTC session.
- **Booleans** `TINYINT(1)`/`BOOLEAN` NOT NULL with a default; **enums** as constrained `VARCHAR`/`TINYINT` + `CHECK` (MySQL 8.0.16+ enforces CHECK) or a lookup table — avoid native `ENUM` (reordering/adding is a table alteration, ordinal comparisons surprise).
- **Strings:** bounded `VARCHAR(n)` sized to the domain; `TEXT`/`BLOB` only when needed and never in hot filtered columns. **JSON** columns for genuinely schemaless payloads only; index JSON via generated columns; never store relational data in JSON.
- **NOT NULL by default**; NULL only when "unknown/not applicable" is a real state. Defaults are explicit.
- **Collation:** the default (`utf8mb4_0900_ai_ci`) is case- AND accent-insensitive: `unique(email)` treats `A@x.com` = `a@x.com`, `é` = `e`. Decide deliberately; use `utf8mb4_bin` (or `VARBINARY`) for tokens, hashes, codes and anything case-sensitive. Legacy `PAD SPACE` collations ignore trailing spaces in comparisons (see §14).
- Never compare a string column to a number (`WHERE varchar_col = 0` matches non-numeric strings, and defeats the index); bind parameters with the right type.
- `AUTO_INCREMENT` gaps are normal (rollbacks, failed inserts): never rely on gapless ids; never expose sequential ids externally.

## 3. Keys and identifiers

- Every table has a primary key. Clustered by PK in InnoDB, so **PK design is a performance decision**: keep it small and insert-ordered.
- Public/opaque ids: **time-ordered UUIDs (UUIDv7/ULID) stored as `BINARY(16)`**, or `BIGINT UNSIGNED` internal PK + separate public id column. **Random UUIDv4 as PK fragments the clustered index and bloats every secondary index — forbidden for large tables.**
- Foreign keys are declared and indexed (InnoDB auto-indexes the FK column if none exists, but design the composite index deliberately). `ON DELETE RESTRICT` by default; **no cascade delete** on business data.
- **[multi-tenant]** `tenant_id` is the first column of composite indexes; unique constraints are per tenant.

## 4. Constraints and integrity

- Integrity lives in the database: `NOT NULL`, `UNIQUE`, `FOREIGN KEY`, `CHECK` (enforced from 8.0.16). App pre-checks are not enough; a concurrency loser must hit a constraint and be mapped to 409/422 (SQLSTATE 23000, errors 1062/1451/1452).
- **No partial indexes in MySQL.** Conditional uniqueness ("one active pairing per phone", "one open run per item") uses a **generated column that is NULL when inactive plus a UNIQUE index** (UNIQUE permits many NULLs). Soft-delete + unique: same technique (unique on `(col, deleted_marker)` where the marker is NULL for live rows).
- Functional indexes (8.0.13+) or indexed generated columns for case-insensitive/derived lookups.
- `INSERT IGNORE`, `REPLACE INTO` and `ON DUPLICATE KEY UPDATE` are dangerous defaults: `IGNORE` swallows errors, `REPLACE` deletes+inserts (fires FK effects, changes ids), `ON DUPLICATE KEY UPDATE` fires on **any** unique key. Use them only with a single, understood unique key and a test.
- Never disable `foreign_key_checks` or `unique_checks` in application code or routine migrations.

## 5. Indexing

- Index every FK and every filtered/sorted/joined column; **composite index column order = equality columns first, then range, then sort**; leftmost-prefix rule; covering indexes for hot list queries.
- An index must earn its place (proven by `EXPLAIN`); drop duplicates and unused (`sys.schema_unused_indexes`, `performance_schema`). Each index costs writes and buffer pool.
- Key length limit 3072 bytes (DYNAMIC row format); `utf8mb4` uses up to 4 bytes/char — use prefix indexes or hashed/normalized columns for long strings.
- **Non-sargable patterns are bugs on hot paths:** functions on indexed columns (`WHERE DATE(created_at)=…`, `LOWER(email)=…` without a functional index), leading wildcard `LIKE '%x'`, implicit type/charset conversions, `OR` across different columns without index merge, `!=`/`NOT IN` on large sets.
- Full-text: InnoDB `FULLTEXT`; user input is escaped/stripped of BOOLEAN-mode operators and length-capped.

## 6. Queries and pagination

- Parameterized queries only (`security.md` §4). Select only needed columns.
- **Keyset pagination** for large lists (`WHERE (created_at, id) < (?, ?) ORDER BY created_at DESC, id DESC LIMIT n`); deep `OFFSET` scans and discards rows. Every paginated `ORDER BY` ends with a unique tie-breaker (`id`).
- `COUNT(*)` on big InnoDB tables is a scan: prefer keyset "has next", cached/rollup counts or estimates.
- Aggregate in SQL; use pre-computed daily rollups for dashboards.
- Avoid `ORDER BY RAND()`, `SELECT *`, correlated subqueries per row, and N+1 (`performance.md` §3). Use `EXISTS` for existence checks.
- `EXPLAIN` / `EXPLAIN ANALYZE` (8.0.18+) is attached to any PR that adds a hot query.

## 7. Transactions, isolation, locking

- Default isolation is **REPEATABLE READ** (consistent snapshot, gap/next-key locks); write paths that must not phantom-read or must serialize use explicit locks.
- **Deadlocks (1213) and lock-wait timeouts (1205) are normal under concurrency: the transaction is retried a bounded number of times** with jitter; keep transactions short, touch rows in a consistent order, no external calls or user waits inside.
- Claim/dequeue patterns: `SELECT … FOR UPDATE SKIP LOCKED` (8.0+; MariaDB 10.6+) or `NOWAIT`; single-row guards: `UPDATE … WHERE id=? AND version=?` and check affected rows.
- **DDL is not transactional** (implicit commit): a migration is designed to be re-runnable/forward-fixable; never mix DDL and data changes expecting rollback.
- Read replicas lag: read-your-writes paths use the primary.

## 8. Migrations (zero-downtime)

- Forward-only once pushed; **expand → backfill → switch → contract** over releases; the deployed code works with the schema before and after each step.
- Declare the algorithm explicitly so the statement **fails instead of silently copying or locking**: `ALTER TABLE … ALGORITHM=INSTANT` (add column, 8.0.12+) or `ALGORITHM=INPLACE, LOCK=NONE`. Anything that would copy a large table uses an online-schema-change tool (gh-ost / pt-online-schema-change) or a shadow-table plan.
- Backfills run in chunks (`WHERE id > ? LIMIT n`), throttled, resumable, outside the request path.
- Adding a NOT NULL/unique/FK on a populated table: backfill and verify first; measure lock time on production-sized data.
- Never drop or rename a column/table in the same release that stops using it.
- Migrations run on every supported engine version in CI and are tested against a dump with realistic volume for big tables.

## 9. Tenant isolation [multi-tenant]

- No row-level security in MySQL: isolation is enforced in the application (scoped accessors, every query has `tenant_id`) and verified by tests that another tenant's id returns 404. Consider one schema/database per tenant only when isolation requirements justify the operational cost.
- Composite unique/index keys start with `tenant_id`; FKs to tenant-owned parents should include `tenant_id` where the schema allows (prevents cross-tenant links).

## 10. Security

- Least-privilege accounts (no `SUPER`, `FILE`, `GRANT OPTION`, no DDL for the app role); separate migration user; host-restricted accounts; strong auth plugin; no shared credentials.
- Encryption in transit (TLS) and at rest (tablespace encryption or disk encryption); field-level encryption in the app for secrets; keys not stored in the same DB.
- Audit tables are append-only for the app role (no `UPDATE/DELETE` grant).
- Slow-query and general logs never contain bound secrets; `general_log` off in production.

## 11. Performance operations

- Tune deliberately: `innodb_buffer_pool_size` (≈ 50–75 % RAM on a dedicated host), `innodb_log_file_size`/redo capacity, `max_connections`, `tmp_table_size`; measure with `performance_schema`/`sys`.
- Slow query log on (`long_query_time` ≈ 0.2–1 s) with regular review; watch top statements by total time, rows examined vs sent, temporary tables on disk, filesorts, buffer pool hit rate, replication lag, deadlock log.
- Keep tables lean: purge/archive by retention job (chunked deletes); `OPTIMIZE`/rebuild only in maintenance windows; partitioning only with a proven need (time-based retention) and partition-key-in-PK constraints understood.
- Statistics: run `ANALYZE TABLE` after bulk loads or big deletes.

## 12. Backups and restore

- Automated daily logical (`mysqldump --single-transaction` / mydumper) or physical (Percona XtraBackup) backups, plus binary logs (`binlog_format=ROW`) for point-in-time recovery when RPO requires it.
- Backups are encrypted, stored off the DB host, retention enforced, and **restore is tested on a schedule** (an untested backup is not a backup). Health reports last-backup age.

## 13. Testing

- Test on the real engine and version (not SQLite) so collation, strict mode, generated columns, locking and JSON behave as in production.
- Tests cover: unique-violation → 409, FK restrict, generated-column uniqueness, deadlock retry path, pagination stability, UTC session zone, and a migration up/down on a realistic fixture.

## 14. MariaDB differences to respect when both are targets

- Collations: `utf8mb4_0900_*` do not exist in MariaDB; pick a collation valid on both or configure per engine.
- Functional key parts are unsupported: use an indexed (virtual) generated column instead.
- `RTRIM()`/`CHAR` inside generated-column expressions can raise error 1901: wrap CHAR columns in `RTRIM()`.
- `JSON` is an alias for `LONGTEXT` with validation: JSON functions and generated-column behaviour differ; test both.
- `EXPLAIN ANALYZE`, `SKIP LOCKED`, `CHECK` enforcement, `INSTANT` DDL and recursive CTE support arrived in different versions: pin minimum versions and test on both.
- Defaults (`sql_mode`, isolation, timestamp behaviour) differ by version: set them explicitly.
