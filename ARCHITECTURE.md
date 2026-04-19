# Architecture

Activation token design, device binding, and session management model for `identity-hub`.

## Context

This service is the **auth plane** of a commercial subscription product. Its neighbors:

```
            ┌─────────────┐
            │  Payment    │  (Stripe / iyzico webhooks)
            │  Provider   │
            └──────┬──────┘
                   │  payment_succeeded event
                   ▼
            ┌─────────────┐     admin-issued        ┌─────────────────┐
            │   Orders    │────activation token────▶│ identity-hub    │
            │   Service   │     via /admin/         │  (this repo)    │
            └─────────────┘                         └──────┬──────────┘
                                                           │  session
                                                           ▼
                                               ┌──────────────────────┐
                                               │ Client Apps          │
                                               │  (vpn-apple-kit,     │
                                               │   Flutter, web)      │
                                               └──────┬───────────────┘
                                                      │  authorized
                                                      ▼
                                               ┌──────────────────────┐
                                               │ proxy-platform       │
                                               │  (access nodes)      │
                                               └──────────────────────┘
```

## Activation token

### Design

A token is a single-use credential bound to an order, carrying enough information to identify the purchase but not enough to impersonate the user.

```
act_    TXdFVWhwSnVGMGVtNnFtMWtpOVFaV2JXTzNYaw
──      ──────────────────────────────────────
prefix  random 32 bytes, base32, URL-safe
```

### Storage

```sql
CREATE TABLE activation_tokens (
    id            uuid        PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id      text        NOT NULL,
    plan_id       text        NOT NULL,
    token_hash    bytea       NOT NULL UNIQUE,      -- SHA-256 of plaintext
    max_slots     int         NOT NULL DEFAULT 3,
    status        text        NOT NULL DEFAULT 'active',
    issued_at     timestamptz NOT NULL DEFAULT now(),
    expires_at    timestamptz NOT NULL,
    revoked_at    timestamptz
);
CREATE INDEX idx_tokens_order ON activation_tokens(order_id);
```

- Plaintext token is **shown exactly once** at issuance
- Only the SHA-256 hash is persisted
- `expires_at` is set conservatively (default 7 days from issuance)
- Expired / revoked tokens do not cascade — they remain in the table for audit

### Lifecycle

```
      issued ──▶ active ──▶ (partially|fully) consumed
                   │
                   ├── expired (passive)
                   └── revoked (admin action)
```

## Device binding

### Slot table

```sql
CREATE TABLE device_slots (
    id                uuid         PRIMARY KEY DEFAULT gen_random_uuid(),
    token_id          uuid         NOT NULL REFERENCES activation_tokens(id),
    device_id         text         NOT NULL,
    platform          text         NOT NULL,
    app_version       text,
    consumed_at       timestamptz  NOT NULL DEFAULT now(),
    last_seen_at      timestamptz  NOT NULL DEFAULT now(),
    revoked_at        timestamptz,
    UNIQUE (token_id, device_id)
);
CREATE INDEX idx_slots_token_active ON device_slots(token_id) WHERE revoked_at IS NULL;
```

- `(token_id, device_id)` uniqueness gives us **idempotent same-device consume** for free — a retry returns the existing slot row
- The partial index on `revoked_at IS NULL` makes active-slot counting a single index lookup

### Consume algorithm

```
BEGIN;

-- 1. Look up token by hash
SELECT id, status, max_slots, expires_at
  FROM activation_tokens
  WHERE token_hash = sha256($1) FOR UPDATE;

-- 2. Validate
IF status != 'active' OR expires_at < now() THEN
  ROLLBACK; RETURN token_invalid;
END IF;

-- 3. Try to insert a slot row. UNIQUE violation = same device, idempotent.
INSERT INTO device_slots (token_id, device_id, platform, app_version)
  VALUES (:token_id, :device_id, :platform, :version)
  ON CONFLICT (token_id, device_id) DO UPDATE
    SET last_seen_at = now()
  RETURNING id;

-- 4. Count active slots for this token
SELECT count(*) FROM device_slots
  WHERE token_id = :token_id AND revoked_at IS NULL;

-- 5. If count exceeds max_slots, roll back the insert
IF active_count > max_slots THEN
  ROLLBACK; RETURN slot_exhausted;
END IF;

-- 6. Create a session for the slot
INSERT INTO sessions (slot_id, access_token, refresh_token, expires_at) ...

COMMIT;
```

Key properties:

- **Atomic** — the whole flow is one transaction; no room for slot races
- **Idempotent for same device** — repeat consume returns the same slot
- **Explicit over-slot detection** — race-losers see `slot_exhausted` and can show "device limit reached"
- **Same-shape read** — admin panel shows the exact same data the user sees

## Sessions

```sql
CREATE TABLE sessions (
    id               uuid        PRIMARY KEY DEFAULT gen_random_uuid(),
    slot_id          uuid        NOT NULL REFERENCES device_slots(id),
    access_token     text        NOT NULL UNIQUE,
    refresh_token    text        NOT NULL UNIQUE,
    access_expires   timestamptz NOT NULL,
    refresh_expires  timestamptz NOT NULL,
    issued_at        timestamptz NOT NULL DEFAULT now(),
    last_used_at     timestamptz NOT NULL DEFAULT now(),
    revoked_at       timestamptz
);
```

- Access token: **EdDSA-signed** JWT, ~1 hour TTL, contains `slot_id`, `order_id`, `plan_id`
- Refresh token: **opaque**, rotated on each use, ~30 day TTL
- Rotation invalidates the previous refresh token — replay detection is built-in

## Revocation

Three flavors of revoke:

1. **Session revoke** — single device. `DELETE /v1/sessions/:id` or user-triggered logout.
2. **Slot revoke** — one device slot; the user reuses the slot for another device. Admin-triggered.
3. **Order revoke** — refund / chargeback. Cascades to all slots and sessions for that order.

All revokes emit an event on the outbox so downstream services (proxy-platform) can immediately kill active tunnels.

## Admin authentication

Admin flows are **completely separate** from end-user flows:

- Dedicated admin database user
- Separate HTTP server on a private interface
- SSO-only (OIDC via Google Workspace / Microsoft Entra)
- IP allow-list enforced at the reverse proxy
- All admin actions go through the audit log table with operator identity

## Audit log

Every state-changing call writes an append-only entry:

```sql
CREATE TABLE audit_log (
    id          bigserial    PRIMARY KEY,
    occurred_at timestamptz  NOT NULL DEFAULT now(),
    actor_type  text         NOT NULL,    -- 'user', 'admin', 'system'
    actor_id    text         NOT NULL,
    action      text         NOT NULL,    -- 'token.issue', 'slot.consume', 'session.revoke', ...
    subject_id  text         NOT NULL,    -- the affected entity
    metadata    jsonb
);
```

## Security assumptions

- Token plaintext exists only in transit and in the email delivered to the user
- Plaintext is never logged (structured logger has an activation-token scrubber)
- Refresh tokens are stored **hashed** too
- Admin access requires SSO with hardware-bound WebAuthn factor
- Signing keys rotate on a 30-day cadence with a rolling 3-key window

## What we changed from upstream

- Added `activation_tokens` + `device_slots` tables and endpoint surface
- Added **slot-aware consume** as an atomic transaction
- Added **order-level revoke cascade**
- Separated admin HTTP server into its own listener on its own port
- Added outbox events for cross-service revocation propagation
- Added the audit log table and structured logging scrubbers
