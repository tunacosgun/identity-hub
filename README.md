# identity-hub

> Authentication and identity management server in Go. Issues short-lived activation tokens, manages device binding, enforces per-purchase slot limits, and serves as the auth plane for commercial subscription products.

<p align="left">
  <img alt="Go" src="https://img.shields.io/badge/Go-1.22%2B-00ADD8?style=flat-square&logo=go&logoColor=white">
  <img alt="Database" src="https://img.shields.io/badge/DB-PostgreSQL-4169E1?style=flat-square&logo=postgresql&logoColor=white">
  <img alt="Auth" src="https://img.shields.io/badge/Auth-OIDC%20%7C%20OAuth2%20%7C%20SAML-blue?style=flat-square">
  <img alt="License" src="https://img.shields.io/badge/License-MIT-green?style=flat-square">
  <img alt="Maintained" src="https://img.shields.io/badge/Maintained%20by-Tunasoft%20Yazılım-181717?style=flat-square">
</p>

---

## What this is

`identity-hub` is Tunasoft Yazılım's Go-based identity and activation server. It handles:

- **OIDC / OAuth2 / SAML** federation with social providers, enterprise SSO, and magic-link flows
- **Activation tokens** — single-use, hashed-at-rest, revocable tokens issued after payment
- **Device binding** — one purchase, up to N device slots, atomic consume
- **Session management** — short-lived access tokens + refresh tokens, revocable per-device
- **Admin authentication** — bastion-friendly, IP-restricted admin login separate from end-user auth

It is the auth plane we run in front of [`proxy-platform`](https://github.com/tunacosgun/proxy-platform) access nodes and consumer-grade mobile apps.

See [`ARCHITECTURE.md`](./ARCHITECTURE.md) for the activation flow, token design, and database schema. See [`ROADMAP.md`](./ROADMAP.md) for planned work.

## Why this fork exists

We run commercial subscription products where authentication has unusual requirements:

1. **No user-visible account on first launch.** The client app does not show a login screen — the user paid via web checkout and received an activation link. Identity is bootstrapped from that link.
2. **Strict slot limits.** One purchase = up to 3 devices. This must be enforced atomically; concurrent consume requests must not race.
3. **Fast revoke.** When a refund happens, all active sessions for that order must terminate within seconds.
4. **Admin auth is separate.** End-user OAuth flows and admin SSO must not share the same token surface.

Upstream [`supabase/auth`](https://github.com/supabase/auth) (gotrue) is an excellent general-purpose auth server and we use its core as the foundation. This fork adds the activation-token, device-binding, and slot-management layer on top.

## Features

- ✅ **Activation token issuance** — `POST /admin/tokens/issue` after a paid order
- ✅ **Atomic consume** — `POST /v1/activate { token, device_id }` binds in a single transaction
- ✅ **Slot enforcement** — per-order max device count, extra consumes return `409 slot_exhausted`
- ✅ **Same-device idempotency** — re-consuming with the same `device_id` is a no-op
- ✅ **Revocation** — `POST /admin/orders/:id/revoke` terminates all active sessions for that order
- ✅ **Federated login** — Google, Apple, Microsoft, GitHub, plus SAML/OIDC for enterprise
- ✅ **Magic link & OTP** — email / SMS one-time codes for low-friction flows
- ✅ **Refresh tokens** — rotated on use, revocable per-device
- ✅ **Audit log** — every issue / consume / revoke / session event, append-only, queryable

## Getting started

### Build

```bash
git clone https://github.com/tunacosgun/identity-hub.git
cd identity-hub
make build
```

### Run

```bash
cp example.env .env
# edit DATABASE_URL and secrets
./identity-hub run
```

### Issue your first activation token

```bash
curl -X POST http://localhost:9999/admin/tokens/issue \
  -H "X-Admin-Token: $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "order_id": "ord_abc123",
    "plan_id": "plan_pro_monthly",
    "max_slots": 3,
    "ttl_hours": 168
  }'
```

Response:
```json
{
  "token": "act_TXdFVWhwSnVGMGVtNnFtMWtpOVFaV2JXTzNYaw",
  "token_hash": "5e884898da28047151d0e56f8dc62927...",
  "expires_at": "2026-04-26T12:00:00Z"
}
```

The plaintext `token` is shown **once** at issuance. Only the hash is stored.

## Activation endpoint (client-facing)

```bash
curl -X POST https://auth.example.com/v1/activate \
  -H "Content-Type: application/json" \
  -d '{
    "token": "act_TXdFVWhwSnVGMGVtNnFtMWtpOVFaV2JXTzNYaw",
    "device_id": "DEVC-9A7F-41B2",
    "platform": "ios",
    "app_version": "1.4.2"
  }'
```

Success response:
```json
{
  "session_id": "sess_01HZ…",
  "access_token": "eyJhbGciOiJFZERTQSI…",
  "refresh_token": "rt_…",
  "expires_in": 3600,
  "slot": 1,
  "slots_used": 1,
  "slots_total": 3
}
```

## Related Tunasoft projects

- [`proxy-platform`](https://github.com/tunacosgun/proxy-platform) — access-node service that consumes sessions issued by this server
- [`vpn-apple-kit`](https://github.com/tunacosgun/vpn-apple-kit) — iOS/macOS client that calls `/v1/activate`
- [`clean-go-starter`](https://github.com/tunacosgun/clean-go-starter) — Go modular-monolith template we use for the surrounding control-plane services

## Credits

Built on top of [`supabase/auth`](https://github.com/supabase/auth) (gotrue), one of the most battle-tested Go auth servers in the open-source ecosystem. This fork is maintained by Tunahan Coşgun and Duygu Durmuş at Tunasoft Yazılım and extended for commercial subscription activation-token and device-binding use cases.

## License

MIT — see [`LICENSE`](./LICENSE).

## Contact

- 📧 [info@tunahancosgun.dev](mailto:info@tunahancosgun.dev)
- 🌐 [tunahancosgun.dev](https://tunahancosgun.dev)
- 📅 [Book a consultation](https://cal.com/tunacosgun/intro)
