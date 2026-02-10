# Habit Tracker PostgreSQL Schema (Applied via CLI)

This repository's PostgreSQL schema was applied **directly via CLI** using the connection command stored in:

- `habit_tracker_database/db_connection.txt` (e.g., `psql postgresql://...`)

Per project rules, **no `.sql` files** were created; each statement was executed one-at-a-time using:

- `psql ... -c "SINGLE_SQL_STATEMENT"`

## Extensions

- `pgcrypto` (for `gen_random_uuid()` UUID primary keys)

## Tables

### `app_users`
Stores application users.

**Columns (key ones):**
- `id uuid PK default gen_random_uuid()`
- `email text NOT NULL` (unique case-insensitive via index on `lower(email)`)
- `password_hash text NOT NULL`
- `display_name text`
- `timezone text NOT NULL default 'UTC'`
- `is_email_verified boolean NOT NULL default false`
- `created_at timestamptz default now()`
- `updated_at timestamptz default now()`

**Constraints / Indexes:**
- Check: email contains `@` (basic sanity check)
- Unique index: `app_users_email_uq` on `lower(email)`
- Trigger: updates `updated_at` on row updates

### `refresh_sessions`
Stores hashed refresh tokens / sessions.

- `user_id` FK → `app_users(id)` ON DELETE CASCADE
- Unique index on `refresh_token_hash`
- Index on `user_id`

### `habits`
Stores habits created by users.

- `user_id` FK → `app_users(id)` ON DELETE CASCADE
- `title`, `description`, `category`
- `is_archived`
- `start_date`, `end_date`

Indexes:
- `(user_id)`
- `(user_id, is_archived)`
- `(user_id, category)`

Trigger:
- updates `updated_at` on updates

### `habit_schedule`
Defines schedule/frequency rules for a habit.

- `habit_id` FK → `habits(id)` ON DELETE CASCADE
- `frequency` in `('daily','weekly','monthly')`
- `target_count >= 1`
- Optional schedule fields:
  - `days_of_week smallint[]` (length constraint only)
  - `day_of_month smallint` (1..31)
  - `week_start smallint` (0..6)

Indexes:
- Unique `(habit_id)` to enforce one schedule per habit

Trigger:
- updates `updated_at` on updates

### `habit_completions`
Completion records by time period.

- `habit_id` FK → `habits(id)` ON DELETE CASCADE
- `user_id` FK → `app_users(id)` ON DELETE CASCADE
- `period_start date`, `period_end date`
- `completed_on date`
- `completion_count >= 1`
- Optional `note`

Indexes:
- Unique `(habit_id, user_id, period_start, period_end)`
- `(user_id, completed_on desc)`
- `(habit_id, period_start desc)`

### `habit_streak_snapshots`
Precomputed/cached streak metrics.

- `habit_id` FK → `habits(id)` ON DELETE CASCADE
- `user_id` FK → `app_users(id)` ON DELETE CASCADE
- `granularity` in `('daily','weekly','monthly')`
- `as_of_date date`
- `current_streak`, `longest_streak` (>= 0)
- `completion_rate numeric(5,2)`

Index:
- Unique `(habit_id, user_id, granularity, as_of_date)`

### `reminders`
Reminder scheduling rules.

- `user_id` FK → `app_users(id)` ON DELETE CASCADE
- `habit_id` FK → `habits(id)` ON DELETE SET NULL
- `channel` in `('email','push','sms','inapp')`
- `remind_at time`
- `days_of_week smallint[]` (length constraint only)
- `timezone text`
- `is_enabled boolean`

Index:
- `(user_id, is_enabled)`

Trigger:
- updates `updated_at` on updates

### `exports`
Tracks export/report generation requests.

- `user_id` FK → `app_users(id)` ON DELETE CASCADE
- `export_type` in `('progress_report','habits','completions','full')`
- `format` in `('csv','json','pdf')`
- `status` in `('queued','processing','completed','failed')`
- timestamps + optional `file_url`, `error_message`

Index:
- `(user_id, requested_at desc)`

### `audit_log`
Audit events.

- `user_id` FK → `app_users(id)` ON DELETE SET NULL
- `entity_type`, `entity_id`
- `action` in `('create','update','delete','login','logout','complete','export','reminder')`
- `details jsonb`
- `ip_address inet`, `user_agent text`, `created_at`

Indexes:
- `(user_id, created_at desc)`
- `(entity_type, entity_id)`

## Seed Data (Minimal)

Inserted idempotently (`ON CONFLICT DO NOTHING`):

- Demo user:
  - `id = 00000000-0000-0000-0000-000000000001`
  - `email = demo@habit.local`
- Sample habits:
  - `Drink Water` (daily)
  - `Read 10 pages` (weekly with days_of_week)
- One reminder (in-app)
- One completion row (today)
- One streak snapshot (daily, today)

## Notes

- Some deep array element validation (e.g., ensuring each `days_of_week` element is 0..6) was intentionally kept minimal because PostgreSQL `CHECK` constraints cannot use subqueries; stricter validation can be enforced in application logic or via triggers if needed.
