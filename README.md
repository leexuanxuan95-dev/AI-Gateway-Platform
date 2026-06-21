# AI Gateway SaaS Platform

Enterprise AI model aggregation / billing / risk-control / finance platform.
One OpenAI-compatible API endpoint fronts all major model providers; unified
metering, quota, risk control, and accounting. Built per
`AI-Gateway-SaaS-完整需求与技术设计文档.md`.

## Stack

| Layer | Tech |
|---|---|
| Frontend | Vue3 + TypeScript + Pinia + Element Plus + Vite |
| Backend | Spring Boot 3.2 + Spring Security + MyBatis-Plus (Java 17) |
| Model gateway | LiteLLM (separate service) |
| Cache / rate-limit | Redis 7 |
| Message queue | Kafka (billing / notify / risk fan-out) |
| Primary DB | PostgreSQL 16 (partitioned billing tables) |
| Object store | MinIO |
| Deploy | Docker Compose / Kubernetes |

## Layout

```
backend/      Spring Boot modular monolith
  src/main/java/com/aigateway
    common/         result, exception, security (JWT/RBAC), config, crypto
    modules/
      user/         auth, login, 2FA, login log
      company/      multi-tenant company + members + RBAC + projects
      apikey/       API key issue / hash / permissions
      model/        model family/version/route + ModelSyncJob + providers
      channel/      upstream account pool, reserve, balance monitor, auto-recharge
      finance/      account, transaction, recharge, withdraw, pricing + profit engine
      payment/      pay-channel config center, adapters, unified callback
      gateway/      OpenAI-compatible /v1/* proxy, api-key auth, rate limit, billing
      plan/         subscription plans
      risk/         rate limit, blacklist, risk rules
      notify/       email/sms/inmail/webhook
      billing/      token bill + call log + aggregation jobs
      system/       sys_config center
frontend/     Vue3 admin console (scaffold)
deploy/       docker-compose.yml, init.sql, litellm-config.yaml, .env.example
```

## Quick start (local)

```bash
cd deploy
cp .env.example .env && vim .env        # fill secrets
docker compose up -d                     # starts pg/redis/kafka/minio/litellm/backend/frontend
# init.sql is auto-applied by the postgres container on first boot
```

Then:
1. Open the console at http://localhost and create the initial super-admin.
2. Configure: upstream channel keys → model versions & pricing → plans → publish.
3. Clients call `http://localhost:8080/v1/chat/completions` with `sk-xxx`.

## Production deploy & observability

Production assets live under `deploy/` (design doc §15.2 / §17 / §19):

- **`deploy/k8s/`** — Kubernetes manifests (namespace, config/secret templates,
  backend/frontend/LiteLLM deployments + services, PG/Redis/Kafka StatefulSets,
  MinIO, HPA 3..20, TLS Ingress, zero-trust NetworkPolicies, PDB). LiteLLM and all
  datastores are ClusterIP-only and reachable only from the backend. Start with
  `deploy/k8s/README.md` for the apply order.
- **`deploy/monitoring/`** — Prometheus (scrapes `/actuator/prometheus`), alert
  rules (error rate, p99 >30ms, low channel balance, margin <5%, pod/DB/Redis down,
  429 spikes), Alertmanager, and a provisioned Grafana overview dashboard.
- **`deploy/logging/`** — ELK pipeline (Filebeat → Logstash → Elasticsearch/Kibana)
  preserving the `requestId` trace field.
- **`deploy/nginx/`** — standalone TLS reverse proxy (HSTS/CSP, rate-limit zones,
  SSE passthrough) for non-k8s deployments.
- **`deploy/openapi/gateway-v1.yaml`** — OpenAPI 3 spec for the customer `/v1/*` API.

## Customer-facing API

OpenAI-compatible. Clients only change `base_url` + `api_key`:

```python
from openai import OpenAI
client = OpenAI(base_url="https://api.your-gateway.com/v1", api_key="sk-xxx")
client.chat.completions.create(model="claude-opus-4-8",
    messages=[{"role":"user","content":"hi"}])
```

See design doc §20 for the full endpoint reference and error codes.

## Status

This is a from-spec implementation. Core modules, schema, gateway proxy,
billing/pricing engine, payment channel framework, and scheduled jobs are
implemented. Production hardening items (full provider SDK coverage, exhaustive
test suites, K8s manifests, ClickHouse reporting) are tracked against the
Go-Live checklist in design doc §19.
