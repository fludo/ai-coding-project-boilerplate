---
name: mariadb-data-access
description: Applies MariaDB data-access rules for CodeIgniter 4 - migrations, Query Builder, transactions, indexing, and connection config. Use when designing schema, writing migrations/queries, or reviewing database code.
---

# MariaDB Data Access (CodeIgniter 4)

## Prerequisite Detection

Inspect `app/Config/Database.php`, the `.env` `database.*` keys, existing `app/Database/Migrations` and `app/Database/Seeds`, and representative models before applying conventions. Confirm the server is MariaDB (`SELECT VERSION();`) and the driver in use. Treat a rule as project-specific only when configuration or an established pattern supports it; label inferences.

## Connection & Driver

- MariaDB speaks the MySQL wire protocol, so CodeIgniter's **`MySQLi`** driver is the default choice. Do not switch drivers without a stated reason.
- Configure the connection through `.env` (`database.default.hostname`, `.database`, `.username`, `.password`, `.port`, `.DBDriver`), never by editing `Database.php` with literal credentials.
- Use `utf8mb4` / `utf8mb4_general_ci` (or `utf8mb4_unicode_ci`) for full Unicode support. Avoid the legacy `utf8` (3-byte) charset.
- `DBDebug` is `true` in development (surfaces SQL errors) and **must be `false` in production**.
- Keep a separate `tests` connection group so the suite never touches live data. See [[codeigniter-testing]].

## Schema via Migrations

All schema changes go through migrations in `app/Database/Migrations`; never alter a live schema by hand. Filenames are timestamp-ordered: `YYYY-MM-DD-HHMMSS_Description.php`.

```php
<?php

namespace App\Database\Migrations;

use CodeIgniter\Database\Migration;

class CreateArticlesTable extends Migration
{
    public function up(): void
    {
        $this->forge->addField([
            'id'         => ['type' => 'INT', 'constraint' => 11, 'unsigned' => true, 'auto_increment' => true],
            'title'      => ['type' => 'VARCHAR', 'constraint' => 255],
            'body'       => ['type' => 'TEXT'],
            'published'  => ['type' => 'TINYINT', 'constraint' => 1, 'default' => 0],
            'created_at' => ['type' => 'DATETIME', 'null' => true],
            'updated_at' => ['type' => 'DATETIME', 'null' => true],
        ]);
        $this->forge->addKey('id', true);
        $this->forge->addKey('published');
        // InnoDB is required for foreign keys and transactions.
        $this->forge->createTable('articles', false, ['ENGINE' => 'InnoDB']);
    }

    public function down(): void
    {
        $this->forge->dropTable('articles');
    }
}
```

**Migration rules**
- Every `up()` has a matching, reversible `down()`.
- One logical change per migration; do not edit a migration that has already run in any shared environment — add a new one.
- Use **InnoDB** for any table needing transactions or foreign keys (it is MariaDB's default, but state it for clarity).
- Add foreign keys with `addForeignKey('user_id', 'users', 'id', 'CASCADE', 'CASCADE')` and index the referencing column.
- Choose types deliberately: `BIGINT UNSIGNED` for large id spaces, `DATETIME` vs `TIMESTAMP` per timezone needs, `DECIMAL` (never `FLOAT`) for money.
- Run with `php spark migrate`; roll back with `php spark migrate:rollback`. Seed reference data with `php spark db:seed`.

## Query Builder

Prefer the model API; drop to the Query Builder for queries beyond it. **Never concatenate user input into SQL.**

```php
$builder = $this->db->table('articles');
$rows = $builder
    ->select('id, title, published')
    ->where('published', 1)
    ->whereIn('id', $ids)          // values are bound, not interpolated
    ->orderBy('created_at', 'DESC')
    ->limit($perPage, $offset)
    ->get()
    ->getResultArray();
```

- Pass values as builder arguments (bound) rather than embedding them in strings. If a raw fragment is unavoidable, bind with `$db->query($sql, $params)` and justify it.
- Avoid `SELECT *` on wide tables; name the columns you use.
- Use the framework's `paginate()` on models for list endpoints instead of manual limit/offset math where possible.
- Guard against N+1: batch related lookups with `whereIn`/joins instead of querying inside a loop.

## Transactions

Wrap multi-statement writes that must succeed or fail together.

```php
$this->db->transException(true)->transStart();

$this->orderModel->insert($order);
$this->lineItemModel->insertBatch($items);

$this->db->transComplete();
```

- Use `transStart()`/`transComplete()` (auto rollback on failure) for the common case; use explicit `transBegin/transCommit/transRollback` only when you need conditional control.
- `transException(true)` turns a failed transaction into a `DatabaseException` so the failure is not silently swallowed.
- Keep transactions short; do no external I/O (HTTP calls, queue publishes) inside them.

## Indexing & Performance

- Index columns used in `WHERE`, `JOIN`, `ORDER BY`, and foreign keys; add them in the creating migration.
- Add composite indexes in the order queries filter (leftmost-prefix rule); do not create redundant single-column indexes already covered by a composite.
- Verify costly queries with `EXPLAIN` before adding or removing indexes; record the finding.
- Set sensible column lengths — an unbounded `VARCHAR(255)` that only holds a status string wastes index space.

## Data Integrity & Security

- Enforce invariants in the schema (`NOT NULL`, `UNIQUE`, foreign keys, `CHECK`) rather than relying only on application code.
- All external input reaches the database only after model/Query-Builder binding — this is the SQL-injection boundary. Never build queries from request strings.
- Store timestamps in UTC; convert at the presentation edge.
- Never log full rows containing secrets or personal data.
