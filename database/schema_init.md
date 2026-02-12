# Smart Calendar - Database Schema Initialization (PostgreSQL)

This container uses PostgreSQL. Connection details are stored in `db_connection.txt` and are created by `startup.sh`.

**Critical rule:** Apply SQL changes **one statement at a time** (one `psql -c "..."` invocation per statement).

## Connect

```bash
# From this directory:
cd smart-calendar-319576-319586/database

# Use the connection string that startup.sh writes:
$(cat db_connection.txt)

# Or run a one-off SQL command:
$(cat db_connection.txt) -c "SELECT version();"
```

## Schema overview

### `events`
Stores calendar events.

- `id` UUID primary key
- `title`, `description`
- `start_at`, `end_at` (`timestamptz`)
- `timezone` (IANA TZ string; stored for display)
- `all_day`
- optional fields like `location`, `color`
- `created_at`, `updated_at`
- check constraint ensures `end_at >= start_at`

### `notification_outbox`
Outbox/queue for notifications (backend worker can poll for due notifications).

- `event_id` FK to `events(id)` (cascade delete)
- `scheduled_for` when notification should be sent
- `channel` default `in_app` (backend can extend)
- `payload` JSONB
- `status` constrained to: `pending | processing | sent | failed | cancelled`
- `attempts`, `last_error`, `sent_at`
- `created_at`, `updated_at`

### `scheduler_state`
Key/value JSON store to keep scheduler cursors/locks.

## Apply schema (one statement at a time)

Run the following commands in order. Each bullet is a **single SQL statement**.

1) Extensions (UUID generation)

```bash
$(cat db_connection.txt) -c "CREATE EXTENSION IF NOT EXISTS pgcrypto;"
```

2) Events table

```bash
$(cat db_connection.txt) -c "CREATE TABLE IF NOT EXISTS events (id UUID PRIMARY KEY DEFAULT gen_random_uuid(), title TEXT NOT NULL, description TEXT NOT NULL DEFAULT '', start_at TIMESTAMPTZ NOT NULL, end_at TIMESTAMPTZ NOT NULL, timezone TEXT NOT NULL DEFAULT 'UTC', all_day BOOLEAN NOT NULL DEFAULT FALSE, location TEXT NOT NULL DEFAULT '', color TEXT NOT NULL DEFAULT '', created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(), updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(), CONSTRAINT events_end_after_start CHECK (end_at >= start_at));"
$(cat db_connection.txt) -c "CREATE INDEX IF NOT EXISTS idx_events_start_at ON events (start_at);"
$(cat db_connection.txt) -c "CREATE INDEX IF NOT EXISTS idx_events_end_at ON events (end_at);"
```

3) `updated_at` trigger helper

Note: use a string literal body to avoid shell `$$` expansion issues.

```bash
$(cat db_connection.txt) -c "CREATE OR REPLACE FUNCTION set_updated_at() RETURNS TRIGGER LANGUAGE plpgsql AS 'BEGIN NEW.updated_at = NOW(); RETURN NEW; END;';"
```

4) Add `updated_at` trigger to `events` (idempotent)

```bash
$(cat db_connection.txt) -c "DROP TRIGGER IF EXISTS trg_events_set_updated_at ON events;"
$(cat db_connection.txt) -c "CREATE TRIGGER trg_events_set_updated_at BEFORE UPDATE ON events FOR EACH ROW EXECUTE FUNCTION set_updated_at();"
```

5) Notification outbox

```bash
$(cat db_connection.txt) -c "CREATE TABLE IF NOT EXISTS notification_outbox (id UUID PRIMARY KEY DEFAULT gen_random_uuid(), event_id UUID NOT NULL REFERENCES events(id) ON DELETE CASCADE, scheduled_for TIMESTAMPTZ NOT NULL, channel TEXT NOT NULL DEFAULT 'in_app', payload JSONB NOT NULL DEFAULT '{}'::jsonb, status TEXT NOT NULL DEFAULT 'pending', attempts INTEGER NOT NULL DEFAULT 0, last_error TEXT NOT NULL DEFAULT '', created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(), updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(), sent_at TIMESTAMPTZ NULL, CONSTRAINT notification_outbox_status_chk CHECK (status IN ('pending','processing','sent','failed','cancelled')));"
$(cat db_connection.txt) -c "CREATE INDEX IF NOT EXISTS idx_notification_outbox_due ON notification_outbox (status, scheduled_for);"
$(cat db_connection.txt) -c "DROP TRIGGER IF EXISTS trg_notification_outbox_set_updated_at ON notification_outbox;"
$(cat db_connection.txt) -c "CREATE TRIGGER trg_notification_outbox_set_updated_at BEFORE UPDATE ON notification_outbox FOR EACH ROW EXECUTE FUNCTION set_updated_at();"
```

6) Scheduler state

```bash
$(cat db_connection.txt) -c "CREATE TABLE IF NOT EXISTS scheduler_state (key TEXT PRIMARY KEY, value JSONB NOT NULL DEFAULT '{}'::jsonb, updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW());"
$(cat db_connection.txt) -c "DROP TRIGGER IF EXISTS trg_scheduler_state_set_updated_at ON scheduler_state;"
$(cat db_connection.txt) -c "CREATE TRIGGER trg_scheduler_state_set_updated_at BEFORE UPDATE ON scheduler_state FOR EACH ROW EXECUTE FUNCTION set_updated_at();"
```

## Verify

```bash
$(cat db_connection.txt) -c "SELECT table_name FROM information_schema.tables WHERE table_schema='public' ORDER BY table_name;"
```
