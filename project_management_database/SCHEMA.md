# Project Management Database (PostgreSQL)

This container provides a PostgreSQL database for the digital agency management dashboard.

## Connection

A helper connection command is stored in:

- `db_connection.txt`

Example:

- `psql postgresql://appuser:dbuser123@localhost:5000/myapp`

> Note: `db_connection.txt` includes the `psql ` prefix. If you want just the URI, strip the prefix (e.g. `sed 's/^psql //'`).

## Schema overview

Tables:

- `users`
  - `id` UUID PK
  - `email` unique (case-insensitive uniqueness enforced by index on `lower(email)`)
  - `password_hash`, `full_name`, `avatar_url`
  - `is_active`
  - `created_at`, `updated_at`

- `clients`
  - `id` UUID PK
  - `owner_user_id` UUID FK → `users(id)` (cascade delete)
  - `name`, `email`, `phone`, `company`, `notes`
  - `created_at`, `updated_at`
  - Unique per owner (case-insensitive): `(owner_user_id, lower(name))`

- `projects`
  - `id` UUID PK
  - `owner_user_id` UUID FK → `users(id)` (cascade delete)
  - `client_id` UUID FK → `clients(id)` (set null on delete)
  - `name`, `description`
  - `status` in: `active | completed | paused | cancelled`
  - `start_date`, `due_date`
  - `budget_cents`, `revenue_cents` (non-negative)
  - `created_at`, `updated_at`
  - Frequently queried indexes:
    - `status`
    - `created_at DESC`
    - `owner_user_id`
    - `client_id`
  - Unique per owner (case-insensitive): `(owner_user_id, lower(name))`

- `user_settings`
  - `user_id` UUID PK + FK → `users(id)` (cascade delete)
  - `theme` in: `light | dark`
  - `created_at`, `updated_at`

### Updated-at behavior

A trigger function keeps `updated_at` current on updates:

- `set_updated_at()` (plpgsql)
- Triggers exist on `users`, `clients`, `projects`, and `user_settings`

## Minimal seed data (dev/local)

Seeded records (idempotent inserts):

- User: `demo@agency.test`
- Client: `Acme Corp`
- Projects:
  - `Website Redesign` (active)
  - `Brand Kit` (completed)
  - `Monthly SEO` (paused)
- `user_settings` for demo user with theme `light`

## Re-applying schema / seed (no .sql files)

This project prefers applying SQL directly using `psql -c`, one statement per call.

From the `project_management_database/` directory:

1) Ensure PostgreSQL is running (see `startup.sh`)
2) Use the connection from `db_connection.txt`

Example pattern:

```bash
CONN="$(cat db_connection.txt | sed 's/^psql //')"
psql "$CONN" -v ON_ERROR_STOP=1 -c "SELECT 1;"
```

Then apply DDL/DML statements one-at-a-time as needed.
