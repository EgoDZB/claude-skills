---
name: postgres-conventions
description: PostgreSQL naming and schema conventions (tables, columns, keys, constraints, indexes, relations, types) for .NET/EF Core, Node.js (Drizzle, Prisma, TypeORM, Knex) and Go (sqlc). Use when designing or reviewing schemas, writing migrations, entity configurations, sqlc queries or SQL, or when asked how to name database objects. Reports deviations to the user instead of silently changing them.
---

# PostgreSQL conventions

## How to apply

There are two modes. Decide which one applies before writing anything.

**Mode A — you choose the names.** The user asked for a schema, entity or migration and did not dictate names. Apply the
conventions below directly.

**Mode B — the names already exist or were dictated.** The user specified names, or existing code, migrations or the
database contain names that deviate from these conventions. Never rename silently. Report and ask.

Report format, in the user's language, before making changes:

```
Naming:
Must fix
- `"UserProfile"` (table) → `user_profiles` — quoted PascalCase must be quoted in every query forever
- `created` timestamp DEFAULT (CURRENT_TIMESTAMP AT TIME ZONE 'UTC') → `created_at timestamptz DEFAULT now()` — value shifts when session TimeZone is not UTC
Recommended
- `users.user_email` → `users.email` — table name is already the context
```

Then ask one question: apply all, apply only "Must fix", or keep as is. Wait for the answer.

Rules for reporting:

- **Must fix** = causes bugs, breaks queries or tooling: quoted mixed-case identifiers, reserved words, names over 63
  bytes, `timestamp` without time zone, wrong time defaults, `json`/`float`/`money` for data that needs `jsonb`/
  `numeric`, missing FK index, locks outside a transaction.
- **Recommended** = style and consistency.
- Report only objects touched by the current task. If you noticed other deviations elsewhere, add one line: "Also N
  similar deviations in <place> — list them?"
- Objects in applied migrations or a live database: a rename is a breaking change. Only suggest it; if approved,
  implement it as a new migration. Never edit an applied migration.
- If the codebase consistently uses a different convention, it wins. Mention the difference once, then follow the
  project convention without flagging each occurrence.
- Names produced by the ORM naming convention (for example EFCore.NamingConventions) are accepted as-is.
- Once the user confirms a non-conforming name, use it and do not raise it again in this session.

## General rules

- `snake_case`, lowercase, never quoted identifiers.
- Max 63 bytes per identifier; longer names are silently truncated. Shorten meaningfully, not by cutting.
- Tables plural: `users`, `orders`, `order_items`.
- Columns singular, no table prefix: `users.email`, not `users.user_email`.
- Full words except common abbreviations: `id`, `url`, `ip`, `sku`.
- Never use reserved or ambiguous words as names: `user`, `order`, `group`, `table`, `limit`, `offset`, `desc`, `end`,
  `date`, `time`, `timestamp`.
- No type or object prefixes: no `tbl_users`, `str_name`.
- Booleans are positive: `is_deleted`, never `is_not_deleted`.

## Standard columns

| Purpose           | Name                                                     | Type                                                  |
|-------------------|----------------------------------------------------------|-------------------------------------------------------|
| Primary key       | `id`                                                     | `bigint generated always as identity` or `uuid` v7    |
| Created / updated | `created_at`, `updated_at`                               | `timestamptz not null default now()`                  |
| Soft delete       | `deleted_at`                                             | `timestamptz null`                                    |
| Audit             | `created_by`, `updated_by`                               | FK to `users.id`                                      |
| Flags             | `is_active`, `is_verified`, `has_children`, `can_edit`   | `boolean not null default false`                      |
| Status            | `status`                                                 | `text` + check, enum, or lookup table                 |
| Date only         | `birth_date`, `due_on`                                   | `date`                                                |
| Validity          | `expires_at`, `valid_from`, `valid_to`                   | `timestamptz`                                         |
| Money             | `amount`, `unit_price`, `total_amount` + `currency_code` | `numeric(19,4)` + `char(3)`, or `amount_minor bigint` |
| Counters          | `items_count`, `retry_count`                             | `integer` / `bigint`                                  |
| Ordering          | `sort_order`                                             | `integer`                                             |
| Business key      | `code`, `slug`, `external_id`                            | `text` + unique                                       |
| Tenant            | `tenant_id`                                              | FK                                                    |
| Concurrency       | `version` or system `xmin`                               | `integer` / `xid`                                     |
| Free-form data    | `metadata`, `payload`, `settings`                        | `jsonb`                                               |

Suffixes carry meaning: `_at` instant, `_on`/`_date` calendar date, `_id` reference, `_count` counter, `_url`, `_path`,
`_hash`.

## Keys and constraints

| Object                  | Pattern                           | Example                            |
|-------------------------|-----------------------------------|------------------------------------|
| Primary key             | `pk_<table>`                      | `pk_orders`                        |
| Foreign key             | `fk_<table>_<ref_table>_<column>` | `fk_orders_customers_customer_id`  |
| Unique constraint       | `uq_<table>_<cols>`               | `uq_users_email`                   |
| Alternate key (EF Core) | `ak_<table>_<cols>`               | `ak_products_sku`                  |
| Check                   | `ck_<table>_<rule>`               | `ck_orders_amount_positive`        |
| Exclusion               | `ex_<table>_<cols>`               | `ex_bookings_room_id_period`       |
| Index                   | `ix_<table>_<cols>`               | `ix_orders_customer_id_created_at` |
| Unique index            | `ux_<table>_<cols>`               | `ux_users_email`                   |
| Partial index           | `ix_<table>_<cols>_<condition>`   | `ix_users_email_not_deleted`       |
| Index by method         | `ix_<table>_<col>_<method>`       | `ix_products_tags_gin`             |
| Trigger                 | `trg_<table>_<action>`            | `trg_orders_set_updated_at`        |
| Trigger function        | verb + object                     | `set_updated_at()`                 |
| Sequence                | PostgreSQL default                | `orders_id_seq`                    |

- FK column: `<referenced table, singular>_id`, or a role name when the role matters: `author_id`, `sender_id`,
  `recipient_id`, `parent_id`.
- The FK pattern includes the column, so several FKs to one table stay unique: `fk_messages_users_sender_id`,
  `fk_messages_users_recipient_id`.
- PostgreSQL does not index FK columns. Every FK needs an index unless a composite index already starts with it. EF Core
  creates these by convention; Drizzle, Prisma, TypeORM, Knex and sqlc do not.

## Relations

| Kind                         | Shape                            | Example                                          |
|------------------------------|----------------------------------|--------------------------------------------------|
| One-to-many                  | FK on the child                  | `order_items.order_id → orders.id`               |
| One-to-one                   | FK + unique, or PK = FK          | `user_profiles.user_id`                          |
| Many-to-many                 | Junction `<a>_<b>`, composite PK | `user_roles (user_id, role_id)`, `pk_user_roles` |
| Many-to-many with attributes | Meaningful entity name           | `memberships`, `enrollments`                     |
| Self-reference               | `parent_id` or role              | `categories.parent_id`, `employees.manager_id`   |

Polymorphic pairs (`commentable_type` + `commentable_id`) cannot have FKs. Prefer separate nullable FK columns with a
check that exactly one is set, or separate tables.

## Schemas, views, functions, types

| Object               | Pattern                | Example                                       |
|----------------------|------------------------|-----------------------------------------------|
| Schema               | domain or module       | `auth`, `billing`, `catalog`, `audit`         |
| View                 | `v_<name>`             | `v_active_users`                              |
| Materialized view    | `mv_<name>`            | `mv_daily_sales`                              |
| Function / procedure | verb + object          | `get_user_balance`, `calculate_order_total`   |
| Enum type            | `<entity>_<attribute>` | `order_status`                                |
| Enum values          | lowercase snake_case   | `pending`, `paid`, `refunded`                 |
| Database role        | `<app>_<access>`       | `shop_app_rw`, `shop_app_ro`, `shop_migrator` |

Enum values cannot be dropped without recreating the type. For sets that change, prefer `text` + check or a lookup
table.

## Types and defaults

- `timestamptz` for instants, default `now()`. `CURRENT_TIMESTAMP AT TIME ZONE 'UTC'` on a `timestamptz` column is a
  bug.
- `text` instead of `varchar(n)` unless the length is a business rule; prefer `check (char_length(col) <= n)` for that.
- `jsonb`, never `json`.
- `numeric` for money; never `real`, `double precision` or `money`.
- New identity columns use `generated always as identity`, not `serial`/`bigserial`.
- UUID primary keys use v7: PostgreSQL 18 `uuidv7()`, .NET 9 `Guid.CreateVersion7()`, Node `uuid` package `v7()`, Go
  `github.com/google/uuid` `uuid.NewV7()`. Random v4 keys fragment B-tree indexes.
- `gen_random_uuid()` is built in since PostgreSQL 13; `uuid-ossp` and `pgcrypto` are not needed for it.
- `citext` or a `lower(email)` unique index for case-insensitive uniqueness.

## Concurrency and query patterns

- Optimistic concurrency, EF Core + Npgsql: a `uint` property configured with `IsRowVersion()` maps to `xmin`.
- Optimistic concurrency, Node and Go: `version integer not null default 0`, updated with
  `where id = $1 and version = $2` and `version = version + 1`; zero affected rows means a conflict. In sqlc use
  `:execrows` to get the affected row count.
- `select ... for update` and `for update skip locked` only inside an explicit transaction; in autocommit the lock is
  released immediately.
- Advisory locks: `pg_try_advisory_xact_lock` inside a transaction. Session-level `pg_advisory_lock` only on a dedicated
  connection held open until unlock; with pooling, lock and unlock may land on different connections.
- Large or infinite lists use keyset pagination:
  `where (created_at, id) < ($1, $2) order by created_at desc, id desc limit $3`. `offset` only for small bounded lists.
- Grants for future tables need `alter default privileges`, not only `grant ... on all tables`.

## ORM setup for snake_case

| Stack            | Setup                                                        |
|------------------|--------------------------------------------------------------|
| EF Core + Npgsql | `EFCore.NamingConventions`, `UseSnakeCaseNamingConvention()` |
| Drizzle          | `casing: 'snake_case'`                                       |
| Prisma           | `@map` on fields, `@@map` on models                          |
| TypeORM          | `SnakeNamingStrategy` from `typeorm-naming-strategies`       |
| Knex / Objection | `knexSnakeCaseMappers()`                                     |
| Go + sqlc        | none; Go names are generated from snake_case automatically   |

If a project uses an ORM without snake_case configured, report it as Must fix before the first migration; after
migrations exist, report it as Recommended with the migration cost.

## Go + sqlc

sqlc is SQL-first: the schema comes from migration files, so every rule above applies to them directly.

- Point `schema` in `sqlc.yaml` at the migrations directory (goose, golang-migrate, dbmate and atlas formats are
  supported; down migrations are ignored). Never keep a separate hand-written schema file next to migrations.
- Plural snake_case tables give singular Go structs: `order_items` → `OrderItem`. Enum `order_status` → type
  `OrderStatus` with constants `OrderStatusPending`.
- Use `sql_package: pgx/v5`. Override `uuid` → `github.com/google/uuid.UUID` and, if wanted, `timestamptz` →
  `time.Time` instead of the `pgtype` wrappers.
- Fix initialisms with `rename` (`url: URL`, `ip: IP`) instead of renaming columns.
- Query names are PascalCase verb + object, with the result kind:

| Purpose    | Example                                     |
|------------|---------------------------------------------|
| One row    | `-- name: GetUserByEmail :one`              |
| Many rows  | `-- name: ListOrdersByCustomer :many`       |
| Insert     | `-- name: CreateOrder :one` + `returning *` |
| Update     | `-- name: UpdateOrderStatus :execrows`      |
| Delete     | `-- name: DeleteSession :exec`              |
| Bulk load  | `-- name: CreateOrderItems :copyfrom`       |

- Name parameters with `@name` or `sqlc.arg(name)`, and `sqlc.narg(name)` for nullable ones; positional `$1` produces
  `Column1`-style Go fields.
- Use `sqlc.embed(table)` for joins instead of selecting columns with ad-hoc aliases.
- Never use `select *` in queries that should survive schema changes, except `returning *` on writes.
