# Resident Directory App — PostgreSQL Schema (Applied)

This document summarizes the PostgreSQL schema and minimal seed data that were **applied directly to the running database** using the CLI connection command in `db_connection.txt`, executing **one SQL statement at a time** (no `.sql` migration files were created).

## Connection

Use the command stored here:

- `resident-connect-platform-315682/database/db_connection.txt`

Example (current):

- `psql postgresql://appuser:dbuser123@localhost:5000/myapp`

## Extensions Enabled

- `pgcrypto` (UUID generation via `gen_random_uuid()`)
- `citext` (case-insensitive email)
- `pg_trgm` (trigram search indexes)

## Enum Types

- `user_role`: `resident`, `admin`
- `gdpr_request_type`: `export`, `delete`
- `gdpr_request_status`: `pending`, `processing`, `completed`, `rejected`

## Tables

### Auth / Users
- `users`
  - Core authentication identity table.
  - Key columns: `email (citext UNIQUE)`, `password_hash`, `role`, `is_active`, `is_email_verified`, timestamps.
  - Search index: trigram GIN on `email::text`.

- `user_sessions`
  - Refresh token sessions.
  - FK: `user_id -> users(id)` (cascade delete).
  - Index: `idx_user_sessions_user_id`.

- `password_reset_tokens`
  - Password reset flow token storage.
  - FK: `user_id -> users(id)` (cascade delete).
  - Index: `idx_password_reset_user_id`.

### Buildings / Units
- `buildings`
  - Unique by `name`.

- `units`
  - FK: `building_id -> buildings(id)` (cascade delete)
  - Unique: `(building_id, unit_label)`
  - Search index: trigram GIN on `unit_label`.

### Resident Directory / Privacy / Consent
- `resident_profiles`
  - 1:1 with users (`user_id UNIQUE`).
  - Optional links to `building_id`, `unit_id`.
  - Privacy flags: `is_directory_visible`, `show_email`, `show_phone`, `show_unit`.
  - Consent flags: `consent_directory`, `consent_messaging`, `consent_marketing`, plus `consent_updated_at`.
  - Search index: trigram GIN on computed full name `first_name || ' ' || last_name`.

- `consent_events`
  - Append-only consent change tracking.
  - FK: `user_id -> users(id)` (cascade delete)
  - Index: `(user_id, created_at DESC)`.

### Messaging
- `conversations`
- `conversation_participants`
  - Composite PK: `(conversation_id, user_id)`
  - Index: `idx_conv_participants_user`.

- `messages`
  - FK: `conversation_id -> conversations(id)` (cascade delete)
  - Timeline index: `(conversation_id, sent_at DESC)`.

### Announcements / Events / Notices
- `announcements`
  - Optional FK: `author_id`, `building_id`
  - Index: `(building_id, published_at DESC)`.

- `events`
  - Optional FK: `created_by`, `building_id`
  - Index: `(building_id, starts_at)`.

- `notices`
  - Optional FK: `created_by`, `building_id`
  - Index: `(building_id, created_at DESC)`.

### Admin / Audit / GDPR
- `audit_logs`
  - Optional FK: `actor_user_id`
  - Index: `(actor_user_id, created_at DESC)`
  - GIN index on `metadata (jsonb)` for flexible filtering.

- `gdpr_requests`
  - FK: `user_id -> users(id)` (cascade delete)
  - Optional FK: `processed_by -> users(id)` (set null)
  - Index: `(user_id, requested_at DESC)`.

## Minimal Seed Data (Development)

Inserted with `INSERT ... ON CONFLICT DO NOTHING` where applicable:

- Building:
  - `Sunset Towers` (Metropolis, CA)

- Units:
  - `1A` (floor 1)
  - `2B` (floor 2)

- Users (dev-only placeholder password hashes):
  - `admin@example.com` (role `admin`)
  - `alice@example.com` (role `resident`)
  - `bob@example.com` (role `resident`)

- Resident profiles:
  - Alice Wong (unit 1A)
  - Bob Smith (unit 2B)

- Announcement:
  - “Welcome to Resident Connect” (pinned, published now)

## Notes / Intended Usage

- This schema is designed to be normalized and extensible for the Resident Directory App.
- Fuzzy search support is provided via trigram GIN indexes (`pg_trgm`) on:
  - user email
  - resident full name
  - unit label

## Operational Rule (Important)

For future DB changes in this project:
- Read the connection command from `db_connection.txt`.
- Execute SQL via CLI **one statement at a time** using `psql ... -c "STATEMENT"`.
- Do **not** add `.sql` migration/seed files unless explicitly requested.
