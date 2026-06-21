# Security Posture

Security controls implemented in this platform and the checklist for production
hardening. Maps to design doc §18 (security requirements) and §12 (risk control).

## Implemented in code

### Authentication & session
- Passwords hashed with **bcrypt (cost=12)**; never stored or logged in plaintext.
- Console auth via **JWT access token (2h)** + opaque **refresh token (30d, Redis)**.
- **2FA / TOTP** (Google Authenticator compatible); TOTP secret **AES-256-GCM encrypted** at rest.
- **Account lockout**: 5 consecutive failed logins → 15-min lock (`LoginAttemptService`).
- **IP throttle** on the public auth surface: 30 req/min/IP (`AuthRateLimitFilter`) to blunt
  credential stuffing and account enumeration.

### Authorization
- **RBAC** via `@RequirePerm` + `PermissionAspect`; role→permission matrix in `RolePermissions`.
- **Row-level multi-tenant isolation**: every tenant query is scoped by `company_id` from the
  authenticated context; cross-tenant access is structurally prevented.
- Platform super-admin endpoints gated separately.

### API gateway (`/v1/*`)
- API keys are `sk-` + 48 random chars, shown **once**, stored only as **SHA-256 hash**.
- Per-key model allow-list, RPM/TPM token buckets (Redis Lua), daily quota, IP whitelist,
  expiry — enforced before any upstream call.
- Pre-deduct (freeze) balance before the call; settle on actual usage → no overspend.

### Secrets
- Upstream provider keys, payment credentials, withdrawal account info: **AES-256-GCM** at rest.
- Decrypted only in-memory at call time; **never logged**; masked in API responses.
- LiteLLM reached over the internal network only (not published to host); message logging off.
- Keys sourced from env/`.env`; production should use **KMS / Vault** (config supports swap-in).

### Payments
- Every webhook callback: **signature verification** (Stripe `Webhook.constructEvent`, RSA2 for
  Alipay/WeChat, on-chain confirmations for USDT) + **idempotency** (Redis lock + DB status check).
- Callback amount/currency validated against the stored order — never trust the callback amount.
- Active polling fallback (`OrderQueryJob`) so dropped callbacks don't lose payments; daily reconcile.

### Transport & headers
- HSTS (1y, includeSubDomains), `X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`,
  CSP `frame-ancestors 'none'`, `Referrer-Policy`, `Permissions-Policy`.
- CORS restricted to an explicit origin allow-list (no wildcard-with-credentials).
- Error responses never include stack traces or internal messages.
- Request size caps (form 2MB, multipart 10MB, header 16KB).

### Auditing & tracing
- `X-Request-Id` propagated across the whole chain (MDC + response header) for traceability.
- `@AuditLog` aspect persists actor / company / ip / outcome / latency for sensitive operations.

### Risk control (design doc §12)
- Multi-dimension rate limiting (key/company/user/IP).
- IP / country (GeoIP) / user blacklists.
- Anomaly detection rules (token spikes, high frequency, datacenter/proxy IPs, multi-IP per key)
  with auto throttle / temp-ban / alert actions.

## Defense-in-depth additions (anti-attack / anti-pen-test layer)

- **Application WAF** (`WafFilter`): blocks SQLi / XSS / path-traversal / command+JNDI(log4shell)
  signatures in URI/query/headers (with double-URL-decode evasion handling) and known scanner
  user-agents (sqlmap/nikto/nmap/…); logs every block.
- **Global per-IP flood backstop** (`GlobalRateLimitFilter`, 600/min) on top of the auth-surface
  throttle and per-API-key limits — fail-open on Redis outage.
- **JWT hardening**: bound `issuer` (verified on parse), per-token `jti`, ≥256-bit key enforced,
  and **revoke-all** via a per-user token epoch (`TokenEpochService`) — password change, 2FA
  disable, and "revoke all sessions" bump the epoch so every previously-issued access token is
  rejected on its next request.
- **Verification-code anti-bombing**: 60s per-target cooldown + daily cap; codes no longer logged.
- **Upload validation** (`UploadValidator`): object-key / document / audio checks — extension &
  content-type allow-list, size caps, path-traversal rejection.
- **SSRF guard** (`SafeUrlValidator`): tenant webhook URLs are resolved and private/loopback/
  link-local/metadata (169.254.169.254) targets are rejected.
- **Dependency CVE scan**: `mvn -P security-scan verify` runs OWASP dependency-check (fails on
  CVSS ≥ 8); P3C static analysis via `mvn -P lint verify`.
- **Security test coverage**: 37 dedicated tests (WAF, rate limits, SSRF, JWT revocation, upload
  validation, code throttle) — part of the 100-test suite, all green.

## Production hardening checklist (do before go-live)

- [ ] Replace all default secrets in `.env`; generate `JWT_SECRET` (`openssl rand -base64 48`)
      and a real 32-byte `AES_KEY` (`openssl rand -base64 32`).
- [ ] Move secrets to **KMS / HashiCorp Vault**; rotate provider & payment keys.
- [ ] Terminate **TLS 1.2+** at the LB; enable HSTS preload; redirect HTTP→HTTPS.
- [ ] Put the management API behind the LB only; do **not** expose Postgres/Redis/Kafka/MinIO/
      LiteLLM/actuator-metrics to the public internet (compose already keeps LiteLLM internal).
- [ ] Bind `/actuator/prometheus` + `/metrics` to an internal management network/port.
- [ ] Add a **WAF** + DDoS protection in front of the gateway.
- [ ] Enable DB encryption at rest + automated backups (PG daily full + WAL, design doc §19).
- [ ] Pen-test: auth bypass, IDOR/tenant-crossing, injection, key leakage, replay of webhooks.
- [ ] Tighten CORS `aigateway.cors.allowed-origins` to the real console domain(s).
- [ ] Verify rate-limit/blacklist/risk rules are enabled and alerting works.
- [ ] PII (e.g. business licenses) stored encrypted in MinIO; comply with local data-residency law.
