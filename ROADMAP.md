# Roadmap

Planned and in-progress work on `identity-hub`, maintained by Tunasoft Yazılım.

## Near term

### 🚧 WebAuthn passkey activation
Add WebAuthn-backed activation as an alternative to email-delivered tokens. After checkout, the user binds a passkey on the web checkout page; the client app pairs with the same passkey on first launch.

### 🚧 Signed audit log roll-up
Daily audit log rotation that produces a signed hash chain (like Transparency Logs). Gives operators a tamper-evident record without shipping raw audit rows off-box.

### 🚧 Activation-token resend rate limit
Current resend endpoint has generous limits. Tightening to per-order-per-hour bucket to reduce inbox spam and abuse vector.

## Next

### ⏳ Refund-to-revoke automation
Listen for `charge.refunded` webhooks from the payment provider and automatically revoke the order without operator intervention. Today this is operator-triggered from the admin panel.

- [ ] Webhook receiver that maps provider-specific refund events to internal `order.revoke`
- [ ] Idempotency key for refund → revoke
- [ ] Admin panel event showing "auto-revoked due to refund"

### ⏳ Slot auto-release on prolonged inactivity
A device that hasn't checked in for 90 days auto-releases its slot. Includes safety: user receives an email 14 days before the release deadline; the device can check in to reset the timer.

### ⏳ Enterprise SSO profile templates
Pre-baked SAML/OIDC configuration templates for Okta, Entra ID, Google Workspace, JumpCloud, and Ping. Removes the most common integration friction for B2B customers.

### ⏳ Device-bound signing keys
For high-trust tiers, move from plain refresh tokens to device-bound signing keys (Apple App Attest / Android Play Integrity). Binding is checked on every refresh.

## Later

### 📦 Policy engine for admin RBAC
Today admin permissions are a flat boolean. Move to OPA / Rego-based policies for role-based access control. Useful for teams with multiple admin tiers.

### 📦 Multi-region active-active
Support for active-active deployment across regions with CRDT-style merging on the `device_slots` table. This is non-trivial because slot consume is an exact-count invariant; leaning toward a single-writer design with region-affinity reads.

### 📦 WebAuthn FIDO2 security keys for admin accounts
Enforce hardware tokens for admin login (already supported as an option; planning to make it mandatory-by-default for new deployments).

### 📦 gRPC API alongside HTTP
For internal callers inside a VPC, gRPC is easier to integrate and more typesafe than HTTP+JSON.

## Accepted, not scheduled

- SCIM 2.0 provisioning endpoint for enterprise customers
- Activation tokens with graduated-access (trial slot free, paid slot paid)
- Machine-to-machine OAuth client credentials flow
- Kafka-based outbox pattern (we use Postgres outbox today; Kafka is a scale-up option)

## Non-goals

- ❌ General-purpose identity-provider competing with Auth0 / Okta — scope is commercial subscription apps
- ❌ Password-auth-as-primary — magic link + OAuth + passkey cover our use cases; password flows exist only because the upstream kernel ships them

## Release cadence

We cut release tags when a shipping feature lands. See [Releases](https://github.com/tunacosgun/identity-hub/releases).

## Contributing to the roadmap

Commercial engagements can prioritize roadmap items. Contact [info@tunahancosgun.dev](mailto:info@tunahancosgun.dev).
