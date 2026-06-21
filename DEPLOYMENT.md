# 部署文档 / Deployment Guide — AI Gateway SaaS + OmniRoute

本文档覆盖**本地/单机(docker-compose)**与**生产(Kubernetes)**两种部署方式,
所有配置项均通过环境变量注入,**不需要改代码或改打进镜像的 `application.yml`**。

> 构建必须用 **JDK 17/21**(本机默认 JDK 25 会让 Lombok 注解处理器失效)。
> Docker 多阶段构建已固定用 `maven:3.9-eclipse-temurin-17`,无此问题。

---

## 1. 组件与端口

| 组件 | 作用 | 容器端口 | 对外 |
|---|---|---|---|
| backend | Spring Boot 网关+控制台 API+官网 CMS | 8080 | 经 LB/Ingress |
| frontend | 运营管理控制台 (Vue3) | 80 | 经 LB/Ingress |
| website | OmniRoute 官网 (Vue3) | 80(compose 映射 8081) | 经 LB/Ingress |
| litellm | 模型路由网关 | 4000 | **仅内网** |
| postgres | 主库 | 5432 | 仅内网 |
| redis | 缓存/限流/会话 | 6379 | 仅内网 |
| kafka | 异步账单/通知/风控 | 9092 | 仅内网 |
| minio | 对象存储(营业执照等) | 9000/9001 | 仅内网 |

架构见设计文档 §15.2。计费/鉴权/风控在 backend 业务网关层;LiteLLM 只做转发+负载均衡。

---

## 2. 配置项总表(全部可在部署文件替换)

每个变量对应 `backend/src/main/resources/application.yml` 里的 `${VAR:默认值}`。
**机密类**走 `.env` / K8s Secret;**非机密类**走 compose env / K8s ConfigMap。

### 2.1 基础设施
| 变量 | 类型 | 说明 |
|---|---|---|
| `DB_URL` | 配置 | `jdbc:postgresql://<host>:5432/ai_gateway`(生产指向 RDS 只改这里) |
| `DB_USER` / `DB_PASSWORD` | 配置/机密 | 数据库账号/口令 |
| `REDIS_HOST` / `REDIS_PORT` / `REDIS_PASSWORD` | 配置/机密 | Redis 连接 |
| `KAFKA_BROKERS` | 配置 | `host:9092` |
| `LITELLM_BASE` | 配置 | `http://litellm:4000`(内网) |
| `LITELLM_MASTER_KEY` | 机密 | 平台调 LiteLLM 管理 API 的主密钥 |
| `MINIO_USER` / `MINIO_PASSWORD` | 机密 | 对象存储凭据 |

### 2.2 安全(**上线必须重置**)
| 变量 | 类型 | 生成方式 |
|---|---|---|
| `JWT_SECRET` | 机密 | `openssl rand -base64 48`(≥32 字节,否则启动报错) |
| `AES_KEY` | 机密 | `openssl rand -base64 32`(必须 32 字节 base64) |
| `WEBHOOK_SIGNING_SECRET` | 机密 | 出站 Webhook HMAC 签名密钥 |
| `CORS_ALLOWED_ORIGINS` | 配置 | 逗号分隔的前端域名白名单 |

### 2.3 认证 / 支付 / 出款
| 变量 | 类型 | 说明 |
|---|---|---|
| `GOOGLE_OAUTH_CLIENT_ID` | 配置 | Google 登录;留空则关闭 |
| `STRIPE_SECRET` / `STRIPE_WEBHOOK_SECRET` | 机密 | Stripe(含 Wise 卡/国际卡收单) |
| `PAY_SUCCESS_URL` / `PAY_CANCEL_URL` | 配置 | 充值跳转(你的公网域名) |
| `ALIPAY_NOTIFY_URL` / `ALIPAY_RETURN_URL` / `WECHAT_NOTIFY_URL` | 配置 | 支付回调/跳转地址 |
| `WISE_ENV` | 配置 | `sandbox` / `production` |
| `WISE_API_TOKEN` / `WISE_PROFILE_ID` / `WISE_PUBLIC_KEY` | 机密 | Wise Platform 出款 + 回调验签 |
| `ALIPAY_APP_ID` / `ALIPAY_PRIVATE_KEY` / `WECHAT_MCH_ID` / `WECHAT_API_V3_KEY` | 机密 | 国内支付(也可在后台「支付渠道配置中心」加密录入,二选一) |
| `OPENAI_KEY` / `ANTHROPIC_KEY_1` / `GEMINI_KEY` / `DEEPSEEK_KEY` / `OPENROUTER_KEY` | 机密 | 上游模型供应商 key(也可在后台「渠道管理」加密录入) |

### 2.4 可调参数(有默认值,一般不用改)
`JWT_ACCESS_TTL` `JWT_REFRESH_TTL` `GATEWAY_TIMEOUT_MS` `PRICING_MARKUP_RATE`
`PRICING_MIN_MARGIN` `EXCHANGE_SAFETY_FACTOR`

> 大量运营参数(加价规则、阈值、任务开关、官网内容等)在**数据库 + 后台**配置,
> 改完即时生效,不需重启(设计文档 §15.4)。

---

## 3. 方式 A:docker-compose(本地/单机)

```bash
cd deploy
cp .env.example .env
vim .env                 # 至少改 PG/REDIS/MINIO 口令、JWT_SECRET、AES_KEY、上游 key
docker compose up -d --build
```

- `init.sql` 在 postgres 首次启动时**自动建表**(挂载到 `/docker-entrypoint-initdb.d`)。
- LiteLLM 不映射到宿主机(仅内网),动态路由持久化在同一个 PG。
- 访问:控制台 `http://localhost`(80)、官网 `http://localhost:8081`、API `http://localhost:8080`。

验证服务健康:
```bash
curl -s http://localhost:8080/actuator/health     # {"status":"UP"}
docker compose ps                                 # 全部 healthy/running
```

升级:`docker compose pull && docker compose up -d`(或重新 `--build`)。
日志:`docker compose logs -f backend`。

---

## 4. 方式 B:Kubernetes(生产)

清单在 `deploy/k8s/`。**应用顺序**(详见 `deploy/k8s/README.md`):

```bash
kubectl apply -f namespace.yaml
kubectl apply -f configmap.yaml
# 机密:不要直接用明文 secret.example.yaml!见 4.1
cp secret.example.yaml secret.yaml && vim secret.yaml && kubectl apply -f secret.yaml
kubectl apply -f networkpolicy.yaml
# 数据存储(生产建议用托管 RDS/ElastiCache/MSK,替换对应 host)
kubectl apply -f postgres-statefulset.yaml -f redis-statefulset.yaml \
              -f kafka-statefulset.yaml -f minio-deployment.yaml
# 首次需把 deploy/init.sql 导入 PG(见 4.2)
kubectl apply -f litellm-deployment.yaml -f litellm-service.yaml
kubectl apply -f backend-deployment.yaml -f backend-service.yaml
kubectl apply -f frontend-deployment.yaml -f frontend-service.yaml
kubectl apply -f website-deployment.yaml
kubectl apply -f hpa.yaml -f poddisruptionbudget.yaml -f ingress.yaml
```

backend 通过 `envFrom`(`backend-config` ConfigMap + `backend-secrets` Secret)注入全部变量——
**在这两个文件里改值即可替换配置**,无需动镜像。

### 4.1 机密管理(生产)
**不要**把明文 Secret 提交/直接 apply。三选一:
- External Secrets Operator(从 AWS Secrets Manager / GCP SM / Vault 同步)
- Sealed Secrets(加密后可提交 git)
- Vault Agent / CSI Secrets Store(运行时挂载)

`secret.example.yaml` 已拆分最小权限:`backend-secrets`(业务)、`litellm-secrets`(仅模型 key + 自己的库)、`datastore-secrets`(PG/Redis/MinIO)。LiteLLM 看不到 Stripe/JWT/AES 等。

### 4.2 数据库初始化
```bash
kubectl -n ai-gateway exec -i statefulset/postgres -- \
  psql -U gw -d ai_gateway < deploy/init.sql
```
> 生产建议改用 **Flyway/Liquibase** 做可回滚迁移(上线前置项,见 SECURITY.md 清单)。

### 4.3 网络与暴露
- `ingress.yaml`:TLS + 强制 HTTPS,`/`→frontend,`/api`+`/v1`→backend;官网域名指到 `website` Service。
- `networkpolicy.yaml`:默认拒绝,只放行 ingress→前端/后端、backend→pg/redis/kafka/litellm/minio;
  PG/Redis/Kafka/LiteLLM **只接受 backend**。需要支持 NetworkPolicy 的 CNI(Calico/Cilium)。
- `hpa.yaml`:backend 3~20 副本(CPU 70%);对应设计文档 §17 的 10k QPS 目标。

---

## 5. 上线后初始化(两种方式通用)

1. 打开控制台 → 创建**超级管理员**账号。
2. 「渠道管理」录入上游供应商 key(AES 加密存储)→ 充值/设置储备金。
3. 「模型管理」确认 `ModelSyncJob` 拉取的版本,设**成本价 + 对客价**并上架
   (对客价必须 ≥ 成本 ×(1+min_margin),否则系统阻止上架——防亏本)。
4. 「套餐管理」配置 plan;「支付渠道配置中心」开启 Stripe/支付宝/微信/USDT 并填密钥。
5. 「官网管理」编辑 OmniRoute 内容 → 预览 → **发布**(价格/模型可绑定 plan/model_version 自动同步)。
6. 客户端用 `sk-xxx` 调 `POST /api/v1/chat/completions` 验证。

---

## 6. 冒烟验证(curl)

```bash
# 健康
curl -s https://api.your-gateway.com/actuator/health
# 官网内容(公开)
curl -s "https://api.your-gateway.com/api/site/landing?lang=zh" | head
# 模型列表(需 sk- key)
curl -s https://api.your-gateway.com/v1/models -H "Authorization: Bearer sk-xxx"
# 一次对话
curl -s https://api.your-gateway.com/v1/chat/completions \
  -H "Authorization: Bearer sk-xxx" -H "Content-Type: application/json" \
  -d '{"model":"claude-opus-4-8","messages":[{"role":"user","content":"hi"}]}'
```

OpenAPI 规范见 `deploy/openapi/gateway-v1.yaml`。

---

## 7. 可观测 / 安全 / 运维

- **监控**:`deploy/monitoring/`(Prometheus 抓 `/actuator/prometheus`、告警规则、Grafana 面板、Alertmanager)。
- **日志**:`deploy/logging/`(Filebeat→Logstash→ES/Kibana,保留 `requestId` 全链路追踪)。
- **安全**:见 `SECURITY.md`(WAF、全局限流、JWT 吊销、SSRF 防护、依赖扫描等)+ 上线前清单。
  - 依赖漏洞扫描:`cd backend && mvn -P security-scan verify`
  - 阿里 P3C 规范检查:`mvn -P lint verify`
- **备份**:PG 每日全备 + WAL;Redis 持久化;定期演练恢复。
- **回滚**:k8s `kubectl rollout undo deployment/backend`;官网内容后台「回滚到历史版本」。

---

## 8. 构建镜像

```bash
# 后端(多阶段,内置 JDK17 构建)
docker build -t your-registry/ai-gateway-backend:1.0.0 backend
# 控制台
docker build -t your-registry/ai-gateway-frontend:1.0.0 frontend
# 官网
docker build -t your-registry/ai-gateway-website:1.0.0 website
docker push your-registry/ai-gateway-backend:1.0.0   # 等
```
把 k8s 清单里的 `your-registry/...:latest` 换成你的镜像 tag。

---

## 9. 常见问题

- **本机 `mvn` 编译报 Lombok 错** → 用 JDK 17/21:`JAVA_HOME=$(/usr/libexec/java_home -v 21) mvn -o clean package`。Docker 构建不受影响。
- **backend 起不来** → 多半是 PG/Redis 未就绪或机密未填;`kubectl logs`/`docker compose logs backend`。
- **NetworkPolicy 不生效** → CNI 不支持(换 Calico/Cilium)。
- **官网空白** → backend 未起或未发布;`GET /api/site/landing` 应返回快照,前端断网时显示内置兜底文案。
- **LiteLLM 没有新模型** → 路由是动态注册的;确认 `ModelSyncJob` 在跑且渠道已启用(设计文档 §7.6)。
