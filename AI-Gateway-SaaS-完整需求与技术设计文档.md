# AI Gateway SaaS 平台 —— 完整需求与技术设计文档（LiteLLM 版）

> 版本：v1.0 ｜ 状态：可落地实施版
> 技术栈：Spring Boot 3.x + Vue3 + PostgreSQL + Redis + Kafka + LiteLLM + K8s
> 目标：企业级 AI 模型聚合 / 计费 / 风控 / 财务平台，部署即可对外提供 OpenAI 兼容 API

---

## 文档说明

本文档在原始需求基础上补全为**可立即实施**的程度，新增/强化以下内容：

- 完整数据库 DDL（可直接执行）
- 完整对外 / 内部 API 契约
- **模型版本选择系统**（模型家族 → 具体版本，企业/项目可自选可用版本）
- **多渠道支付充值**：Stripe、支付宝、微信支付、USDT(TRC20/ERC20)、**Wise 卡支付 + Wise Platform 出金**
- LiteLLM 网关配置、负载均衡、故障转移落地方案
- 计费引擎、风控引擎、限流引擎详细设计
- docker-compose 与 K8s 部署清单
- 安全、监控、上线检查清单

---

# 一、项目定位

打造**企业级 AI 模型聚合平台**，核心价值：

- 一个 API Key 调用所有主流大模型，屏蔽各家协议差异
- 统一计费、统一额度、统一风控、统一账单
- 多租户：平台方 → 企业 → 项目 → API Key 四级隔离
- 充值消费闭环，支持全球支付与出金

## 1.1 接入的模型供应商

| 供应商 | 接入方式 | 主要模型家族 |
|---|---|---|
| OpenAI | 官方 API | GPT 系列 |
| Anthropic | 官方 API | Claude 系列 |
| Google | Vertex / AI Studio | Gemini 系列 |
| xAI | 官方 API | Grok 系列 |
| Mistral | 官方 API | Mistral 系列 |
| Cohere | 官方 API | Command 系列 |
| DeepSeek | 官方 API | DeepSeek 系列 |
| OpenRouter | 聚合 | 全量中转 |
| Azure OpenAI | 企业私有部署 | GPT 系列 |

## 1.2 对外统一接口（OpenAI 兼容）

```http
POST /v1/chat/completions          # 对话（支持流式 SSE）
POST /v1/embeddings                # 向量
POST /v1/images/generations        # 文生图
POST /v1/audio/transcriptions      # 语音转文字
POST /v1/audio/speech              # 文字转语音
POST /v1/rerank                    # 重排序
GET  /v1/models                    # 列出当前 Key 可用模型（含版本）
```

完全兼容 OpenAI SDK（Python / Node / Go），客户端只需替换 `base_url` 与 `api_key`：

```python
from openai import OpenAI
client = OpenAI(
    base_url="https://api.your-gateway.com/v1",
    api_key="sk-xxxxxxxxxxxxx"
)
resp = client.chat.completions.create(
    model="claude-opus-4-8",          # 可选具体版本
    messages=[{"role":"user","content":"hi"}]
)
```

---

# 二、系统角色与权限（RBAC）

## 2.1 角色层级

```
超级管理员（平台运营方）
  └── 企业（租户）
        ├── Owner（企业主）
        ├── Admin（企业管理员）
        ├── Developer（开发者）
        └── Viewer（只读）
```

## 2.2 权限矩阵

| 功能 | 超管 | Owner | Admin | Developer | Viewer |
|---|:--:|:--:|:--:|:--:|:--:|
| 平台用户管理 | ✅ | — | — | — | — |
| 平台财务/提现审核 | ✅ | — | — | — | — |
| 模型/渠道/套餐配置 | ✅ | — | — | — | — |
| 风控配置 | ✅ | — | — | — | — |
| 企业成员管理 | ✅ | ✅ | ✅ | — | — |
| 创建/删除 API Key | ✅ | ✅ | ✅ | ✅ | — |
| 充值 | — | ✅ | ✅ | — | — |
| 申请提现 | — | ✅ | — | — | — |
| 创建/管理项目 | ✅ | ✅ | ✅ | ✅ | — |
| 调用 API | — | ✅ | ✅ | ✅ | — |
| 查看账单 | ✅ | ✅ | ✅ | 本人 | ✅ |
| 选择模型版本 | — | ✅ | ✅ | ✅ | — |

权限实现：基于 Spring Security + 自定义 `@RequirePerm("billing:read")` 注解 + Redis 缓存权限点，企业级数据通过 `company_id` 做行级隔离。

---

# 三、用户系统

## 3.1 注册方式

- 邮箱注册（邮箱验证码激活）
- 手机号注册（短信验证码）
- Google OAuth 登录
- 企业邀请注册（带 invite_token，自动绑定企业 + 角色）

## 3.2 登录方式

| 方式 | 说明 |
|---|---|
| 邮箱 + 密码 | bcrypt(cost=12) 加盐存储 |
| 手机号 + 验证码 | 验证码 5 分钟有效，Redis 存储 |
| Google OAuth | 标准 OIDC 流程 |
| 2FA | 登录后强制 Google Authenticator（可企业策略强制） |

登录成功签发 **JWT(access 2h) + Refresh Token(30d, Redis)**，支持多端登录与单点登出。

## 3.3 安全中心

- 修改密码 / 邮箱 / 手机号（需二次验证）
- 绑定/解绑 Google Authenticator（TOTP）
- 登录日志（IP、设备、地理位置、时间）
- 登录设备管理（可远程踢出）
- 异地登录告警（邮件/短信）

## 3.4 用户状态机

```
待审核 → 正常 → 冻结 → 禁用
              ↘ 注销（软删除）
```

## 3.5 用户表 DDL

```sql
CREATE TABLE sys_user (
    id              BIGSERIAL PRIMARY KEY,
    uid             VARCHAR(32) UNIQUE NOT NULL,        -- 业务唯一ID U + 雪花
    email           VARCHAR(128) UNIQUE,
    mobile          VARCHAR(20) UNIQUE,
    nickname        VARCHAR(64),
    avatar          VARCHAR(512),
    password        VARCHAR(128),                       -- bcrypt
    google_id       VARCHAR(64) UNIQUE,
    totp_secret     VARCHAR(64),                        -- 2FA 密钥（加密存储）
    totp_enabled    BOOLEAN DEFAULT FALSE,
    status          SMALLINT DEFAULT 0,                 -- 0待审核 1正常 2冻结 3禁用 4注销
    register_ip     VARCHAR(45),
    register_time   TIMESTAMPTZ DEFAULT now(),
    last_login_time TIMESTAMPTZ,
    last_login_ip   VARCHAR(45),
    created_at      TIMESTAMPTZ DEFAULT now(),
    updated_at      TIMESTAMPTZ DEFAULT now()
);
CREATE INDEX idx_user_email ON sys_user(email);
CREATE INDEX idx_user_mobile ON sys_user(mobile);

CREATE TABLE sys_login_log (
    id           BIGSERIAL PRIMARY KEY,
    user_id      BIGINT NOT NULL,
    login_type   VARCHAR(20),
    ip           VARCHAR(45),
    location     VARCHAR(128),
    user_agent   VARCHAR(512),
    device_id    VARCHAR(64),
    status       SMALLINT,                              -- 1成功 0失败
    created_at   TIMESTAMPTZ DEFAULT now()
);
```

---

# 四、企业系统（多租户）

## 4.1 创建企业 / 企业认证

注册成 Owner 后创建企业，提交认证：营业执照、法人身份证、企业信息。

认证状态：`待审核 → 审核通过 / 审核拒绝`。未认证企业有调用上限（试用额度）。

## 4.2 企业表 DDL

```sql
CREATE TABLE biz_company (
    id               BIGSERIAL PRIMARY KEY,
    company_code     VARCHAR(32) UNIQUE NOT NULL,
    company_name     VARCHAR(128) NOT NULL,
    contact_name     VARCHAR(64),
    contact_mobile   VARCHAR(20),
    contact_email    VARCHAR(128),
    business_license VARCHAR(512),                      -- MinIO 文件 key
    legal_id_front   VARCHAR(512),
    owner_user_id    BIGINT NOT NULL,
    plan_id          BIGINT,                            -- 当前套餐
    plan_expire_at   TIMESTAMPTZ,
    auth_status      SMALLINT DEFAULT 0,                -- 0待审核 1通过 2拒绝
    status           SMALLINT DEFAULT 1,                -- 1正常 2冻结 3禁用
    created_at       TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE biz_company_member (
    id          BIGSERIAL PRIMARY KEY,
    company_id  BIGINT NOT NULL,
    user_id     BIGINT NOT NULL,
    role        VARCHAR(20) NOT NULL,                   -- OWNER/ADMIN/DEVELOPER/VIEWER
    status      SMALLINT DEFAULT 1,
    invited_by  BIGINT,
    joined_at   TIMESTAMPTZ DEFAULT now(),
    UNIQUE(company_id, user_id)
);
```

---

# 五、项目系统

企业可创建多个项目（客服机器人 / 知识库问答 / 代码助手 / 营销文案……）。
项目用于**成本归集与权限隔离**：每个项目独立绑定 API Key、模型权限、消费额度。

```sql
CREATE TABLE biz_project (
    id            BIGSERIAL PRIMARY KEY,
    company_id    BIGINT NOT NULL,
    name          VARCHAR(64) NOT NULL,
    description   VARCHAR(255),
    monthly_quota NUMERIC(18,6) DEFAULT 0,              -- 月消费上限(0不限)
    allowed_models JSONB,                               -- 允许模型版本白名单
    status        SMALLINT DEFAULT 1,
    created_at    TIMESTAMPTZ DEFAULT now()
);
```

---

# 六、API Key 系统

## 6.1 Key 生成

格式：`sk-` + 48 位随机串。展示时只显示一次，库内存哈希（SHA-256），明文不落库。

## 6.2 Key 权限与限制

- 可访问模型版本白名单
- RPM（每分钟请求数）
- TPM（每分钟 Token 数）
- 每日请求数 / 每日 Token 额度
- IP 白名单（可选）
- 过期时间

## 6.3 Key 表 DDL

```sql
CREATE TABLE biz_api_key (
    id            BIGSERIAL PRIMARY KEY,
    company_id    BIGINT NOT NULL,
    project_id    BIGINT,
    key_name      VARCHAR(64),
    key_prefix    VARCHAR(12),                          -- 展示用 sk-abc...
    key_hash      VARCHAR(64) UNIQUE NOT NULL,          -- SHA-256
    allowed_models JSONB,                               -- ["claude-opus-4-8","gpt-5"]
    rpm_limit     INT DEFAULT 60,
    tpm_limit     INT DEFAULT 100000,
    daily_quota   NUMERIC(18,6) DEFAULT 0,
    ip_whitelist  JSONB,
    status        SMALLINT DEFAULT 1,                   -- 1启用 2禁用 3删除
    expire_at     TIMESTAMPTZ,
    last_used_at  TIMESTAMPTZ,
    created_by    BIGINT,
    created_at    TIMESTAMPTZ DEFAULT now()
);
CREATE INDEX idx_apikey_hash ON biz_api_key(key_hash);
```

鉴权流程：网关收到 `Authorization: Bearer sk-xxx` → SHA-256 → 查 Redis 缓存（miss 回源 DB）→ 校验状态/过期/模型白名单/限流 → 放行。

---

# 七、模型系统（核心：模型版本选择）

> 这是平台的核心差异化。参考客户端模型选择器（如 Fable 5 / Opus 4.8 / Sonnet 4.6 / Haiku 4.5），
> 平台采用 **"模型家族 → 具体版本"** 两级结构。企业/项目/Key 可精确到**具体版本**做授权与计费。

## 7.1 两级模型结构

```
模型家族 (model_family)            具体版本 (model_version)
─────────────────────────────────────────────────────────
Claude                            claude-opus-4-8      "复杂任务"
                                  claude-sonnet-4-6    "日常高效"
                                  claude-haiku-4-5     "极速响应"
                                  claude-fable-5       "最难挑战"(可标记 unavailable)
GPT                               gpt-5
                                  gpt-4o
Gemini                            gemini-2.5-pro
                                  gemini-2.5-flash
Grok                              grok-3
DeepSeek                          deepseek-v3
                                  deepseek-r1
```

前端模型选择器渲染逻辑与图片一致：每个版本展示**显示名 + 一句话定位 + 可用状态徽标**（可用 / 维护中 / 当前不可用）。不可用版本置灰、不可选。

## 7.2 模型版本表 DDL

```sql
-- 模型家族
CREATE TABLE model_family (
    id          BIGSERIAL PRIMARY KEY,
    family_code VARCHAR(32) UNIQUE NOT NULL,            -- claude / gpt / gemini
    family_name VARCHAR(64),
    icon        VARCHAR(512),
    sort        INT DEFAULT 0,
    status      SMALLINT DEFAULT 1
);

-- 具体模型版本（对外可选项）
CREATE TABLE model_version (
    id              BIGSERIAL PRIMARY KEY,
    family_id       BIGINT NOT NULL,
    model_code      VARCHAR(64) UNIQUE NOT NULL,        -- 对外模型名 claude-opus-4-8
    display_name    VARCHAR(64),                        -- "Opus 4.8"
    tagline         VARCHAR(128),                       -- "For complex tasks / 复杂任务"
    capabilities    JSONB,                              -- ["chat","vision","tool","stream"]
    context_window  INT,                                -- 上下文长度
    max_output      INT,
    -- 计费（平台对客单价，单位：元 或 USD per 1M tokens）
    price_input     NUMERIC(12,6),                      -- 输入 / 1M tokens
    price_output    NUMERIC(12,6),                      -- 输出 / 1M tokens
    price_cached    NUMERIC(12,6),                      -- 缓存命中输入价
    currency        VARCHAR(8) DEFAULT 'USD',
    status          SMALLINT DEFAULT 1,                 -- 1启用 2维护 3禁用/不可用
    availability    VARCHAR(20) DEFAULT 'available',    -- available/maintenance/unavailable
    sort            INT DEFAULT 0,
    created_at      TIMESTAMPTZ DEFAULT now()
);

-- 模型版本 → 渠道映射（一个版本可由多个渠道供给，做负载均衡/故障转移）
CREATE TABLE model_channel_route (
    id            BIGSERIAL PRIMARY KEY,
    model_id      BIGINT NOT NULL,                      -- model_version.id
    channel_id    BIGINT NOT NULL,                      -- channel.id
    upstream_model VARCHAR(128),                        -- 该渠道真实模型名
    weight        INT DEFAULT 1,                        -- 权重
    priority      INT DEFAULT 0,                        -- 故障转移优先级（小优先）
    status        SMALLINT DEFAULT 1
);
```

## 7.3 GET /v1/models（按 Key 权限返回可选版本）

```json
{
  "object": "list",
  "data": [
    {
      "id": "claude-opus-4-8",
      "display_name": "Opus 4.8",
      "tagline": "For complex tasks",
      "family": "claude",
      "availability": "available",
      "context_window": 200000,
      "capabilities": ["chat","vision","tool","stream"]
    },
    {
      "id": "claude-sonnet-4-6",
      "display_name": "Sonnet 4.6",
      "tagline": "Most efficient for everyday tasks",
      "availability": "available"
    },
    {
      "id": "claude-haiku-4-5",
      "display_name": "Haiku 4.5",
      "tagline": "Fastest for quick answers",
      "availability": "available"
    },
    {
      "id": "claude-fable-5",
      "display_name": "Fable 5",
      "tagline": "For your toughest challenges",
      "availability": "unavailable"
    }
  ]
}
```

调用时客户端在 `"model"` 字段直接传 `model_code`（如 `claude-opus-4-8`），网关据此选路由。

## 7.4 LiteLLM 映射配置（config.yaml）

LiteLLM 作为底层网关，平台 `model_code` → LiteLLM `model_name`，每个 model_name 下挂多个 deployment 做负载均衡：

```yaml
model_list:
  # ---- Claude 版本 ----
  - model_name: claude-opus-4-8
    litellm_params:
      model: anthropic/claude-opus-4-8
      api_key: os.environ/ANTHROPIC_KEY_1
      weight: 2
  - model_name: claude-opus-4-8            # 同名第二渠道 → 故障转移
    litellm_params:
      model: openrouter/anthropic/claude-opus-4.8
      api_key: os.environ/OPENROUTER_KEY
      weight: 1
  - model_name: claude-sonnet-4-6
    litellm_params:
      model: anthropic/claude-sonnet-4-6
      api_key: os.environ/ANTHROPIC_KEY_1
  - model_name: claude-haiku-4-5
    litellm_params:
      model: anthropic/claude-haiku-4-5
      api_key: os.environ/ANTHROPIC_KEY_1

  # ---- GPT 版本 ----
  - model_name: gpt-5
    litellm_params: { model: openai/gpt-5, api_key: os.environ/OPENAI_KEY }
  - model_name: gpt-4o
    litellm_params: { model: openai/gpt-4o, api_key: os.environ/OPENAI_KEY }

  # ---- Gemini / Grok / DeepSeek ----
  - model_name: gemini-2.5-pro
    litellm_params: { model: gemini/gemini-2.5-pro, api_key: os.environ/GEMINI_KEY }
  - model_name: deepseek-v3
    litellm_params: { model: deepseek/deepseek-chat, api_key: os.environ/DEEPSEEK_KEY }

router_settings:
  routing_strategy: weighted-pick          # 权重负载均衡
  num_retries: 3
  retry_after: 1
  allowed_fails: 3
  cooldown_time: 30                         # 渠道熔断冷却(s)
  fallbacks:                                # 故障转移链
    - { "claude-opus-4-8": ["gpt-5"] }

litellm_settings:
  drop_params: true
  request_timeout: 120
```

业务层（Spring Boot）调用 LiteLLM：`POST http://litellm:4000/v1/chat/completions`，Header 带内部管理密钥。计费与限流在**业务网关层**完成，LiteLLM 只负责转发与底层负载均衡。

## 7.5 模型状态

`启用 / 维护 / 禁用(对外 unavailable)`。维护中版本：列表展示但置灰，调用返回 `503 model_in_maintenance`。

## 7.6 ⭐ 模型版本自动同步（通过 API + 定时任务，不写死 yaml）

> 你的要求：模型版本不要手写 yaml，而是**定时任务通过各供应商 API 自动拉取**，入库可配置。
> 这样上游一发新模型、一下线旧模型，平台自动同步，几乎零维护。

### A. 各供应商「模型列表」API

| 供应商 | 列表接口 | 是否带价格 |
|---|---|---|
| OpenAI | `GET https://api.openai.com/v1/models` | 否（价格需维护） |
| Anthropic | `GET https://api.anthropic.com/v1/models` | 否 |
| Google Gemini | `GET .../v1beta/models` (ListModels) | 否 |
| DeepSeek | `GET https://api.deepseek.com/models` (OpenAI兼容) | 否 |
| xAI Grok | `GET https://api.x.ai/v1/models` (OpenAI兼容) | 否 |
| Mistral | `GET https://api.mistral.ai/v1/models` | 否 |
| **OpenRouter** | `GET https://openrouter.ai/api/v1/models` | **是（含 pricing，可自动同步成本价）** |

### B. 定时任务 ModelSyncJob（每小时/每日）

```
对每个已启用渠道：
  1. 调该供应商 /models 接口，拿到当前可用模型清单
  2. upsert 到 model_version 表：
     - 新出现的模型 → 入库，status=禁用，availability=unavailable，标记「待定价」
     - 已存在的 → 更新元信息（上下文长度等）
     - 上游已下线的 → 标记 availability=unavailable
  3. 写同步日志，新模型告警通知超管去后台定价上架
```

```java
@Scheduled(cron = "0 0 */6 * * ?")              // 每6小时
public void syncModels() {
  for (Channel ch : channelService.listEnabled()) {
    ModelProvider p = providerFactory.get(ch.getProviderName());
    List<UpstreamModel> remote = p.listModels(ch);          // 调 /models
    for (UpstreamModel m : remote) {
      ModelVersion mv = modelVersionDao.findByCode(m.id());
      if (mv == null) {
        // 新模型：默认下架 + 待定价（关键防亏，未定价绝不对外售卖）
        modelVersionDao.insert(ModelVersion.draft(m, ch.getFamilyId()));
        notifyService.alertAdmin("发现新模型待定价: " + m.id());
      } else {
        modelVersionDao.touch(mv.getId());                  // 更新存活/元信息
      }
    }
    modelVersionDao.markStaleUnavailable(ch.getId(), remote); // 下线的置灰
  }
}
```

### C. 关键点：免 yaml + 防亏

- **不再依赖静态 LiteLLM yaml**：两种方式任选——
  1. 用 LiteLLM 管理 API 动态注册路由：`POST /model/new`、`POST /model/delete`（同步任务调用），LiteLLM 把模型存它自己的 DB，无需改 yaml。
  2. 平台自己维护 `model_channel_route` 表，网关层读 DB 选路由、直连上游（不经 LiteLLM 静态配置）。
- **新模型默认下架待定价**：自动拉到的新模型，在你没设置「对客价 + 成本价」之前一律不上架，**杜绝以未知成本价卖出导致亏损**。
- 价格策略：OpenRouter 的 `/models` 直接返回成本价 → `PriceSyncJob` 自动同步为参考成本；OpenAI/Anthropic 等无价格 API 的，成本价维护在后台价格表（见 §10.4），上游一旦调价由 `PriceSyncJob` 检测并告警/自动调对客价。

---

# 八、渠道管理系统（供应商账号池）

管理上游供应商账号，一个模型版本可挂多渠道做负载均衡 + 故障转移。

```sql
CREATE TABLE channel (
    id            BIGSERIAL PRIMARY KEY,
    provider_name VARCHAR(32) NOT NULL,                 -- openai/anthropic/google...
    channel_name  VARCHAR(64),
    api_key       VARCHAR(512),                         -- AES-256 加密存储
    base_url      VARCHAR(255),
    org_id        VARCHAR(128),
    quota         NUMERIC(18,6) DEFAULT 0,              -- 渠道额度上限
    used_quota    NUMERIC(18,6) DEFAULT 0,
    weight        INT DEFAULT 1,
    priority      INT DEFAULT 0,
    health        SMALLINT DEFAULT 1,                   -- 1健康 0熔断
    status        SMALLINT DEFAULT 1,
    created_at    TIMESTAMPTZ DEFAULT now()
);
```

## 8.1 负载均衡策略

`轮询 / 权重 / 随机 / 故障转移`。生产推荐：**权重 + 故障转移**。

## 8.2 自动切换（熔断）

触发条件：`429 / 5xx / 超时 / 余额不足`。
逻辑：单渠道连续失败 ≥ `allowed_fails` → 熔断 `cooldown_time` 秒 → 自动切下一优先级渠道 → 全部失败返回 `502 all_channels_unavailable`。健康探测每 30s 半开重试。

## 8.3 ⭐ 上游账户资金管理（你给上游充值 + 余额监控）

> 场景：你（平台方）在各模型平台（OpenAI/Anthropic…）都有自己的付费账户。客户的调用实际消耗的是**你在上游账户里的余额**。所以你必须**保证上游账户有钱**，否则客户调用会失败。

### A. 上游充值记录 + 余额表

```sql
-- 你给上游账户充值的记录
CREATE TABLE upstream_recharge_log (
    id           BIGSERIAL PRIMARY KEY,
    channel_id   BIGINT NOT NULL,
    amount       NUMERIC(18,6),                        -- 本次充值金额
    currency     VARCHAR(8),
    operator     BIGINT,                               -- 操作人
    remark       VARCHAR(255),
    created_at   TIMESTAMPTZ DEFAULT now()
);
-- channel 表已有：quota(已充值额度) / used_quota(累计成本)
-- 上游可用余额 = quota - used_quota（无余额API的供应商按此本地估算）
```

### B. 余额监控定时任务 UpstreamBalanceMonitorJob（每 5 分钟）

```
对每个渠道：
  1. 优先调供应商余额API（OpenRouter GET /credits 等有API的）
     无API的 → 本地估算 balance = quota - used_quota
  2. balance < 预警阈值(low_threshold) → 邮件/短信告警你去充值
  3. balance <= 0 或 < 单次调用预估成本 → 自动熔断该渠道(health=0)
     → 后续请求由 §8.2 故障转移走其他健康渠道
  4. (可选) 配了自动补给的 → 触发自动充值流程
```

| 供应商 | 余额获取方式 |
|---|---|
| OpenRouter | `GET https://openrouter.ai/api/v1/credits`（有API） |
| OpenAI / Anthropic / 其他 | 多无实时余额API → `quota - used_quota` 本地累计估算 |

### C. 资金安全要点

- 每次调用成功后，`channel.used_quota += 上游成本`，实时逼近真实消耗。
- 上游余额不足时**自动转移到其它渠道**，不影响客户体验，同时告警你补钱。
- 充值/消耗全部留痕，可做上游成本对账。
- 上游账户资金 ≠ 你的利润，平台利润单独沉淀（见 §10.4）。

## 8.4 ⭐ 上游账户自动充值（客户消费 → 成本自动回补上游，全自动闭环）

> 你的场景：客户充值并**绑定要用的模型**后，调用时我扣掉对应模型费用，再把这笔**成本自动回充到对应的上游模型账户**，让上游账户始终有钱，无需人工盯着充值。

### A. 资金拆分（每笔消费自动分流）

每笔调用扣客户 `对客价 charge`，平台把它**自动拆成两份**：

```
对客价 charge  =  上游成本 cost            +   毛利 profit
                     │                            │
                     ▼                            ▼
            该模型对应渠道的「上游资金储备」      平台利润账户(§10.4)
            upstream_reserve += cost            profit += (charge-cost)
```

- **成本部分**进入「该模型/渠道的上游资金储备」——这笔钱专门用来回补上游。
- **毛利部分**进入平台利润账户，**绝不动用**，从机制上保证你不会拿利润去垫上游（不亏）。

### B. 模型绑定 / 资金分配（绑定那个模型的账户）

客户可把余额**绑定/分配到指定模型**，各模型独立子额度，互不串用；调用时从所绑模型的子额度扣减。不绑定则走统一余额。

```sql
-- 客户资金 → 模型绑定（可选；不绑则用统一余额）
CREATE TABLE fin_model_binding (
    id          BIGSERIAL PRIMARY KEY,
    company_id  BIGINT NOT NULL,
    model_code  VARCHAR(64),                          -- 绑定的模型版本
    allocated   NUMERIC(18,6) DEFAULT 0,              -- 分配额度
    used        NUMERIC(18,6) DEFAULT 0,              -- 已消费
    status      SMALLINT DEFAULT 1,
    UNIQUE(company_id, model_code)
);

-- 上游资金储备（按渠道/供应商归集，专款回补上游）
CREATE TABLE upstream_reserve (
    id            BIGSERIAL PRIMARY KEY,
    channel_id    BIGINT UNIQUE NOT NULL,
    provider_name VARCHAR(32),
    reserve_amount NUMERIC(18,6) DEFAULT 0,            -- 待回补上游的成本资金池
    auto_recharge  BOOLEAN DEFAULT FALSE,              -- 是否开启自动回补
    threshold      NUMERIC(18,6) DEFAULT 0,            -- 上游余额低于此值触发
    recharge_amount NUMERIC(18,6) DEFAULT 0,           -- 每次回补金额
    updated_at     TIMESTAMPTZ DEFAULT now()
);
```

### C. 自动回补上游 UpstreamAutoRechargeJob（每 5 分钟）

```
对每个开启 auto_recharge 的渠道：
  1. 取上游真实余额(有API的查API，无API的用 quota-used_quota 估算)
  2. 若 余额 < threshold 且 reserve_amount >= recharge_amount：
       → 按该供应商支持的方式给上游账户充值 recharge_amount
       → 成功后：channel.quota += recharge_amount；reserve_amount -= recharge_amount
       → 写 upstream_recharge_log（标记 auto）
  3. reserve 不足或供应商不支持自动 → 生成回补任务 + 告警你手动充
```

### D. ⭐ 各供应商「自动充值」落地方式（技术现实，务必区分）

| 供应商 | 能否程序化充值 | 落地方式 |
|---|---|---|
| **OpenRouter** | ✅ 可 API/加密货币购买额度 | Job 直接调其充值接口，从 reserve 扣 |
| **OpenAI** | ⚠️ 无公开充值API，但控制台有 Auto-reload | 在 OpenAI 后台绑卡设「自动充值(阈值+金额)」；平台保证**绑定的卡/资金源**有钱（可用你的 Wise 卡/对公卡），平台只负责监控+保证 reserve 足够覆盖 |
| **Anthropic** | ⚠️ 同上，控制台有 Auto-reload | 同 OpenAI：控制台配 auto-reload + 绑卡，平台保证资金源充足 |
| **Google/Azure** | 多为后付费(按账单出账) | 不需预充，按月结算；平台只需保证云账户付款方式有效 |
| **其它无 API** | ❌ | Job 生成回补任务 + 通知你手动充，并显示 reserve 里有多少钱可充 |

> 关键认知：真正"全自动加钱到上游账户"的接口能力**取决于上游供应商**。
> 通用且可靠的做法 = **供应商自带 Auto-reload（绑卡自动续费）+ 平台把成本储备(reserve)对应的资金留在绑定卡/资金源上**。这样：客户消费 → 成本进 reserve → reserve 对应的钱在卡里 → 上游余额低时供应商自动从卡扣款续费，闭环自动完成，平台只监控不手动。

### E. 安全与不亏保障

- **专款专用**：reserve 只用于回补上游成本，毛利单独沉淀，永不挪用。
- **回补 ≤ 储备**：自动回补金额不超过 reserve 余额，绝不透支垫钱。
- **币种/汇率缓冲**：跨币种回补用 §10.4 的汇率安全系数，防汇率波动吃掉毛利。
- **失败可重试 + 留痕**：回补失败进重试队列并告警，全程 `upstream_recharge_log` 留痕对账。
- **绑定隔离**：客户绑定某模型的资金不会被其他模型消费占用。

---

# 九、套餐系统

```sql
CREATE TABLE plan (
    id           BIGSERIAL PRIMARY KEY,
    name         VARCHAR(64),                           -- 基础版/专业版/企业版/旗舰版
    price        NUMERIC(12,2),
    currency     VARCHAR(8) DEFAULT 'USD',
    duration     INT,                                   -- 有效天数
    token_limit  BIGINT,                                -- 月 Token 额度(0不限)
    rpm_limit    INT,
    max_api_keys INT,
    max_projects INT,
    allowed_models JSONB,                               -- 该套餐可用模型版本
    sort         INT DEFAULT 0,
    status       SMALLINT DEFAULT 1
);
```

| 套餐 | 月费 | 月 Token | RPM | 最大 Key | 最大项目 | 可用模型 |
|---|---|---|---|---|---|---|
| 基础版 | $0 | 100万 | 20 | 2 | 1 | Haiku/Flash/DeepSeek |
| 专业版 | $49 | 2000万 | 60 | 10 | 5 | + Sonnet/GPT-4o |
| 企业版 | $299 | 2亿 | 300 | 50 | 20 | 全部 |
| 旗舰版 | 议价 | 不限 | 1000+ | 不限 | 不限 | 全部 + 专属渠道 |

---

# 十、财务系统

## 10.1 账户余额

```sql
CREATE TABLE fin_account (
    id                BIGSERIAL PRIMARY KEY,
    company_id        BIGINT UNIQUE NOT NULL,
    available_balance NUMERIC(18,6) DEFAULT 0,          -- 可用
    freeze_balance    NUMERIC(18,6) DEFAULT 0,          -- 冻结(预扣)
    total_recharge    NUMERIC(18,6) DEFAULT 0,
    total_consume     NUMERIC(18,6) DEFAULT 0,
    currency          VARCHAR(8) DEFAULT 'USD',
    version           BIGINT DEFAULT 0,                 -- 乐观锁
    updated_at        TIMESTAMPTZ DEFAULT now()
);

-- 资金流水(每一笔变动都记账，对账依据)
CREATE TABLE fin_transaction (
    id           BIGSERIAL PRIMARY KEY,
    company_id   BIGINT NOT NULL,
    biz_type     VARCHAR(20),                           -- recharge/consume/refund/withdraw/freeze
    amount       NUMERIC(18,6),                         -- 正入账 负出账
    balance_after NUMERIC(18,6),
    ref_id       VARCHAR(64),                           -- 关联订单/账单ID
    remark       VARCHAR(255),
    created_at   TIMESTAMPTZ DEFAULT now()
);
```

余额扣减采用：**调用前预冻结 → 调用后按实际 Token 结算 → 释放差额**，乐观锁 + 事务保证一致性。

## 10.2 充值系统（多渠道）

```sql
CREATE TABLE fin_recharge_order (
    id            BIGSERIAL PRIMARY KEY,
    order_no      VARCHAR(40) UNIQUE NOT NULL,
    company_id    BIGINT NOT NULL,
    user_id       BIGINT NOT NULL,
    amount        NUMERIC(18,2),
    currency      VARCHAR(8),
    pay_channel   VARCHAR(20),                           -- stripe/alipay/wechat/usdt_trc20/usdt_erc20/wise
    pay_method    VARCHAR(20),                           -- card/wallet/crypto/bank
    status        SMALLINT DEFAULT 0,                    -- 0待支付 1已支付 2已取消 3超时 4退款
    third_txn_id  VARCHAR(128),                          -- 第三方流水号
    paid_at       TIMESTAMPTZ,
    expire_at     TIMESTAMPTZ,
    raw_callback  JSONB,                                 -- 原始回调留存
    created_at    TIMESTAMPTZ DEFAULT now()
);
```

### 支持的支付渠道

| 渠道 | 适用 | 接入方式 | 回调 |
|---|---|---|---|
| **Stripe** | 全球信用卡/借记卡（含 **Wise 卡**）| Checkout Session / PaymentIntent | Webhook `checkout.session.completed` |
| **支付宝** | 国内 | 电脑网站支付 / 当面付 | 异步 notify |
| **微信支付** | 国内 | Native 扫码 / JSAPI | 异步 notify |
| **USDT (TRC20)** | 全球加密 | 链上监听 / 第三方网关 | 区块确认 |
| **USDT (ERC20)** | 全球加密 | 链上监听 / 第三方网关 | 区块确认 |
| **Wise 卡** | 全球 | 见下方专节 | Stripe Webhook |

### ⭐ Wise 卡支付（重点说明）

技术事实：**Wise 本身不提供"收单（card acquiring）"能力**——它不能直接帮你向客户收信用卡款。
但 **Wise 卡是一张标准 Visa/Mastercard 多币种借记卡**，可以像任何国际卡一样付款。

因此"支持 Wise 卡支付"的正确落地方式是 **两条路线**：

**路线 A（推荐 / 收款用）—— 通过 Stripe 受理 Wise 卡：**
- 用户在结账页输入 Wise 卡号即可付款，Stripe 作为收单方处理。
- 平台无需单独对接 Wise，开通 Stripe 即自动支持 Wise 卡（及任意国际卡）。
- 前端可在支付方式里显式标注「💳 支持 Wise / 国际卡」提升转化。
- 多币种：Stripe 开启 `Adaptive Pricing` / 指定 `currency`，匹配 Wise 卡币种减少跨币种手续费。

```java
// Stripe Checkout：天然支持 Wise 卡（标准卡通道）
SessionCreateParams params = SessionCreateParams.builder()
    .setMode(SessionCreateParams.Mode.PAYMENT)
    .addPaymentMethodType(SessionCreateParams.PaymentMethodType.CARD) // Wise 卡走 CARD
    .setSuccessUrl(successUrl + "?order=" + orderNo)
    .setCancelUrl(cancelUrl)
    .addLineItem(LineItem.builder()
        .setQuantity(1L)
        .setPriceData(PriceData.builder()
            .setCurrency("usd")
            .setUnitAmount(amountCents)
            .setProductData(ProductData.builder().setName("AI Gateway 充值").build())
            .build())
        .build())
    .putMetadata("order_no", orderNo)
    .putMetadata("company_id", companyId.toString())
    .build();
Session session = Session.create(params);  // 返回 session.getUrl() 跳转付款
```

**路线 B（出金用）—— Wise Platform API 做全球提现：**
- 用于「提现」环节：把企业余额打到其全球银行账户/Wise 账户。
- 对接 Wise Platform `Transfers API`：创建 Quote → Recipient → Transfer → Fund。

```http
POST https://api.wise.com/v1/quotes
POST https://api.wise.com/v1/accounts          # 收款人
POST https://api.wise.com/v3/profiles/{id}/transfers
POST https://api.wise.com/v3/profiles/{id}/transfers/{id}/payments   # 出资
```

> 结论：**收款用 Stripe（自动覆盖 Wise 卡），出金用 Wise Platform**。这是合规且可立即落地的组合。

### 充值通用流程

```
1. 用户选金额+渠道 → 后端创建 recharge_order(status=0) → 返回支付参数/跳转URL
2. 用户在第三方完成支付
3. 第三方异步回调 → 验签 → 幂等校验(order_no) → 更新 order=1
4. 事务内：账户 available_balance += amount，写 fin_transaction(recharge)
5. 发通知(充值成功) + Webhook 推送
6. 主动轮询兜底：每分钟查未支付订单状态，防回调丢失
```

回调安全：Stripe 验 `Stripe-Signature`；支付宝/微信验 RSA2 签名；USDT 校验链上交易金额+到账地址+确认数。**所有回调强制幂等。**

## 10.2.1 ⭐ 支付通道配置中心（后台可配 + 客户自选自付 + 回调到账）

设计原则：**所有支付通道完全后台可配置**，代码里不写死任何渠道。
运营方在后台开关/配置好通道 → 客户在前端只看到已开启的通道、自己选、自己付 → 平台收回调判断是否到账 → 自动入账并对账。

### A. 后台可配置项（每个通道独立配置）

| 配置项 | 说明 | 示例 |
|---|---|---|
| 通道开关 | 一键启用/停用，停用后前端立即不展示 | 启用 / 停用 |
| 显示名称 / 图标 / 排序 | 前端展示用 | 「微信支付」+ 图标 + 排序值 |
| 凭证/密钥 | api_key / 商户号 / 私钥 / API v3 key 等（AES 加密存储） | sk_live_... |
| 环境模式 | 沙箱 / 生产 | sandbox / production |
| 支持币种 | 该通道可收币种 | USD / CNY / USDT |
| 手续费 | 费率% + 固定费，决定实际到账 | 2.9% + 0.30 |
| 最小/最大充值 | 单笔金额限制 | 最小 $5 / 最大 $5000 |
| 汇率来源 | 固定汇率 / 自动汇率(接汇率API) | 自动 |
| 到账确认方式 | webhook / 轮询 / 链上确认数 | webhook |
| 所需确认数 | 仅加密货币 | TRC20=19 / ERC20=12 |
| 回调地址 | 系统自动生成只读，复制到第三方后台 | /api/pay/callback/wechat |

### B. 配置表 DDL

```sql
CREATE TABLE pay_channel_config (
    id            BIGSERIAL PRIMARY KEY,
    channel_code  VARCHAR(20) UNIQUE NOT NULL,          -- stripe/alipay/wechat/usdt_trc20/usdt_erc20/wise
    display_name  VARCHAR(64),
    icon          VARCHAR(512),
    enabled       BOOLEAN DEFAULT FALSE,                -- 后台总开关
    env_mode      VARCHAR(12) DEFAULT 'production',     -- sandbox/production
    credentials   JSONB,                                -- 各通道密钥(AES加密)
    currencies    JSONB,                                -- ["USD","CNY"]
    fee_rate      NUMERIC(6,4) DEFAULT 0,               -- 费率 0.029=2.9%
    fee_fixed     NUMERIC(12,2) DEFAULT 0,              -- 固定费
    min_amount    NUMERIC(12,2) DEFAULT 0,
    max_amount    NUMERIC(12,2) DEFAULT 0,              -- 0=不限
    exchange_mode VARCHAR(12) DEFAULT 'auto',           -- fixed/auto
    fixed_rate    NUMERIC(12,6),
    confirm_type  VARCHAR(12) DEFAULT 'webhook',        -- webhook/polling/onchain
    confirm_blocks INT DEFAULT 0,                        -- 加密货币确认数
    callback_url  VARCHAR(255),                          -- 系统生成只读
    sort          INT DEFAULT 0,
    remark        VARCHAR(255),
    updated_by    BIGINT,
    updated_at    TIMESTAMPTZ DEFAULT now()
);
```

> 修改配置后写入 Redis 并发布失效事件，**实时生效**，无需重启。

### C. 客户端动态获取可用通道

前端不写死任何通道，调用接口拿后台已启用列表渲染：

```http
GET /api/pay/channels?amount=100&currency=USD
```
```json
{
  "channels": [
    { "code":"stripe", "name":"信用卡 (支持 Wise/国际卡)", "icon":"...", "fee":"2.9%+0.30", "min":5, "max":5000 },
    { "code":"alipay", "name":"支付宝", "icon":"...", "fee":"0.6%", "min":1 },
    { "code":"wechat", "name":"微信支付", "icon":"...", "fee":"0.6%", "min":1 },
    { "code":"usdt_trc20", "name":"USDT (TRC20)", "icon":"...", "fee":"1%", "confirm_blocks":19 }
  ]
}
```
- 后台「停用」的通道不出现在列表里。
- 不满足金额/币种限制的通道自动过滤。
- 客户**自己选一个通道、自己完成支付**,平台不代付。

### D. 统一支付下单 + 回调到账（与具体通道解耦）

```
                ┌─────────────────────────────────────────┐
客户选通道+金额 → │ POST /api/pay/create                       │
                │  读 pay_channel_config(校验开关/金额/币种)   │
                │  建 recharge_order(status=待支付)           │
                │  按通道生成支付参数(跳转URL/二维码/收款地址)  │
                └─────────────────────────────────────────┘
                              │
            客户在第三方/钱包自行支付（平台不介入资金）
                              │
        ┌─────────────────────┴──────────────────────┐
        ▼ (到账确认方式由后台配置决定)                  ▼
   webhook 回调                                   主动轮询/链上扫块兜底
   POST /api/pay/callback/{code}                 定时查未支付订单真实状态
        │
        ▼  统一回调处理器（PaymentCallbackHandler）
   1. 按 channel_code 取对应适配器验签
   2. 幂等校验 order_no（已处理直接返回成功）
   3. 校验金额/币种是否与订单一致
   4. 加密货币：校验确认数 ≥ confirm_blocks 才算"到账"
   5. 到账 → 事务：order=已支付 + available_balance += 实收 + 写 fin_transaction
   6. 发通知(充值成功) + Webhook 推送企业
   7. 落 raw_callback 原始报文（对账依据）
```

到账判定标准（`是否到账`）：

| 通道 | 到账判定 |
|---|---|
| Stripe / Wise 卡 | Webhook `checkout.session.completed` 且 `payment_status=paid` |
| 支付宝 | notify `trade_status=TRADE_SUCCESS` 且金额一致 |
| 微信 | 支付结果通知 `SUCCESS` 且金额一致 |
| USDT TRC20/ERC20 | 链上交易到账地址+金额匹配，且确认数 ≥ 配置值 |

### E. 适配器模式（新增通道零侵入）

```java
public interface PaymentAdapter {
    String code();                                   // 通道标识
    PrepayResult createOrder(RechargeOrder o, ChannelConfig c);  // 下单
    CallbackResult handleCallback(String body, Map<String,String> headers, ChannelConfig c); // 回调验签+解析
    OrderStatus query(String orderNo, ChannelConfig c);          // 主动查单兜底
}
// StripeAdapter / AlipayAdapter / WechatAdapter / UsdtTrc20Adapter / WiseAdapter ...
// 新增一个通道 = 实现一个 Adapter + 后台加一条 pay_channel_config，主流程不动
```

### F. 对账（每日自动）

```sql
CREATE TABLE pay_reconcile (
    id           BIGSERIAL PRIMARY KEY,
    channel_code VARCHAR(20),
    recon_date   DATE,
    platform_cnt INT,        platform_amount NUMERIC(18,2),   -- 平台侧
    channel_cnt  INT,        channel_amount  NUMERIC(18,2),   -- 第三方对账单
    diff_cnt     INT,        diff_amount     NUMERIC(18,2),   -- 差异
    status       SMALLINT DEFAULT 0,                          -- 0待处理 1已平 2有差异
    created_at   TIMESTAMPTZ DEFAULT now()
);
```

定时任务每日拉取各通道对账单 ↔ 平台 `recharge_order`，金额/笔数比对，差异告警人工处理。掉单（已付未回调）由查单兜底自动补单入账。

## 10.3 提现系统

```sql
CREATE TABLE fin_withdraw_order (
    id          BIGSERIAL PRIMARY KEY,
    order_no    VARCHAR(40) UNIQUE,
    company_id  BIGINT NOT NULL,
    amount      NUMERIC(18,2),
    fee         NUMERIC(18,2),
    channel     VARCHAR(20),                            -- wise/bank/usdt
    account_info JSONB,                                 -- 收款账户(加密)
    status      SMALLINT DEFAULT 0,                     -- 0待审核 1通过 2拒绝 3已打款 4失败
    audit_by    BIGINT,
    audit_at    TIMESTAMPTZ,
    paid_at     TIMESTAMPTZ,
    created_at  TIMESTAMPTZ DEFAULT now()
);
```

流程：企业申请 → 冻结余额 → 超管审核 → 通过则调 Wise Platform / 银行打款 → 成功解冻并扣减 → 失败回滚解冻。

## 10.4 ⭐ 资金流转与利润引擎（核心：保证赚钱、绝不亏损）

> 这是平台的盈利核心。把"客户的钱 → 你的利润 → 上游成本"三笔钱彻底分开核算，并用硬约束保证**每一笔都赚钱**。

### A. 资金流转全景

```
客户充值 $100
  │
  ├─(可选) 入金维护费 5% = $5  ──────────────▶ 平台利润账户 +$5
  │        可用余额到账 = $95
  │
  └─ 客户调用模型，按【对客价】扣可用余额
        对客价 = 上游成本价 × (1 + 加价率)        例如成本 $1.00 × 1.30 = $1.30
        每笔调用：
          收客户  $1.30  →  扣客户可用余额
          付上游  $1.00  →  计入 channel.used_quota（消耗你在上游的钱）
          毛利    $0.30  ─────────────────────▶ 平台利润账户 +$0.30
  │
  └─ 你用平台利润/收入，定期给上游账户充值（§8.3），维持上游余额充足

最终平台利润 = 入金维护费(可选) + Σ 每笔消费毛利
```

### B. 两种计费模式（后台可配，可叠加）

| 模式 | 说明 | 是否保证不亏 |
|---|---|---|
| **入金维护费** | 充值时抽 `fee%` 作为维护费直接进利润 | 只保证入金有收入，不保证每笔调用不亏 |
| **消费加价（推荐）** | 每笔调用按 `成本 ×(1+加价率)` 收费 | ✅ 每笔都有正毛利，根本上防亏 |
| 叠加 | 入金先抽一道 + 消费再加价 | 利润最大化 |

**推荐：以消费加价为主**（保证每一笔调用都赚），入金维护费为辅（相当于充值手续费）。

### C. 加价/定价配置表（多维度，后台可配）

```sql
-- 模型成本价 + 对客价（model_version 已有 price_input/price_output；这里管加价规则）
CREATE TABLE pricing_markup (
    id            BIGSERIAL PRIMARY KEY,
    scope         VARCHAR(20),                          -- global/family/model/channel/plan/company
    scope_ref     VARCHAR(64),                          -- 对应ID（global为空）
    markup_rate   NUMERIC(6,4) DEFAULT 0.30,            -- 加价率 0.30=+30%
    markup_fixed  NUMERIC(12,6) DEFAULT 0,              -- 固定加价/1M tokens
    min_margin    NUMERIC(6,4) DEFAULT 0.10,            -- ⭐最低毛利率(硬下限,绝不低于)
    enabled       BOOLEAN DEFAULT TRUE,
    updated_at    TIMESTAMPTZ DEFAULT now()
);

-- 入金维护费配置
CREATE TABLE recharge_fee_config (
    id          BIGSERIAL PRIMARY KEY,
    scope       VARCHAR(20),                            -- global/plan/company
    scope_ref   VARCHAR(64),
    fee_rate    NUMERIC(6,4) DEFAULT 0,                 -- 入金抽成 0.05=5%
    enabled     BOOLEAN DEFAULT FALSE
);

-- 平台利润账户 + 利润流水
CREATE TABLE platform_profit_account (
    id            INT PRIMARY KEY DEFAULT 1,
    total_revenue NUMERIC(20,6) DEFAULT 0,              -- 累计对客收入
    total_cost    NUMERIC(20,6) DEFAULT 0,              -- 累计上游成本
    total_profit  NUMERIC(20,6) DEFAULT 0,              -- 累计毛利
    updated_at    TIMESTAMPTZ DEFAULT now()
);
CREATE TABLE profit_ledger (
    id           BIGSERIAL PRIMARY KEY,
    source       VARCHAR(20),                           -- consume/recharge_fee
    ref_id       VARCHAR(64),
    company_id   BIGINT,
    model_code   VARCHAR(64),
    revenue      NUMERIC(18,6),                         -- 对客收费
    cost         NUMERIC(18,6),                         -- 上游成本
    profit       NUMERIC(18,6),                         -- 毛利
    created_at   TIMESTAMPTZ DEFAULT now()
);
```

加价规则优先级（就近覆盖）：`企业 > 套餐 > 模型 > 家族 > 全局`。

### D. ⭐ 防亏损硬约束（不能搞亏的关键）

1. **最低毛利率下限 `min_margin`**：系统计算对客价时强制
   `对客价 = max( 成本价 ×(1+加价率), 成本价 ×(1+min_margin) )`，
   **永远不会低于成本**，哪怕你加价率配错成 0 也兜底。
2. **新模型默认下架待定价**：自动同步来的新模型未设成本价/对客价前不上架，杜绝以未知成本卖出。
3. **汇率缓冲**：跨币种结算用 `实时汇率 × 安全系数(如 1.03)`，对冲汇率波动风险。
4. **上游涨价自动防护**：`PriceSyncJob` 检测到上游成本上涨 → 自动按规则同步上调对客价（或告警人工确认），避免成本倒挂。
5. **预冻结按对客价**：调用前按对客价预估冻结余额，余额不足直接拒绝，**杜绝透支/欠费**。
6. **实时毛利监控**：利润看板毛利率低于阈值（如 < 5%）即告警，可一键暂停该模型。

### E. 计费结算事务（每笔调用）

```
1. 取加价规则 → 算对客价 → 预冻结 freeze = 对客价估算
   （若客户绑定了模型，从该模型子额度 fin_model_binding 冻结/扣减）
2. 调上游，拿真实 usage → 真实上游成本 cost
3. 对客价 charge = max(cost×(1+rate), cost×(1+min_margin)) + 固定加价
4. 事务（资金拆分，见 §8.4）：
     客户 available_balance -= charge；释放冻结差额（绑定模型则扣子额度）
     channel.used_quota   += cost                       # 记上游消耗
     upstream_reserve     += cost                        # ⭐成本进上游储备，待自动回补
     platform_profit: revenue += charge, cost += cost, profit += (charge-cost)
     写 bill_token + profit_ledger
5. 异步 Kafka 落库；毛利<阈值告警
6. UpstreamAutoRechargeJob 异步检测：上游余额低 → 从 upstream_reserve 自动回补上游（§8.4 C/D）
```

> 闭环：**客户扣费 charge → 成本 cost 进 reserve → 上游低额时自动回补 → 毛利留存利润**。整笔自动完成，你不用手动给上游充钱。

### F. 利润看板（运营后台）

实时指标：今日/本月**收入、成本、毛利、毛利率**；各模型毛利率排行；各渠道上游余额；亏损/低毛利模型预警；入金维护费收入。让你随时看清"赚了多少、哪个模型在亏"。

---

# 十一、账单与计费系统

## 11.1 计费引擎

计费公式：

```
上游成本 cost = (prompt_tokens/1e6 × 成本_input)
              + (completion_tokens/1e6 × 成本_output)
              + (cached_tokens/1e6 × 成本_cached)

对客单次费用 charge = max( cost × (1 + 加价率),          # 正常加价
                          cost × (1 + min_margin) )     # ⭐ 最低毛利兜底，绝不低于成本
                    + 固定加价(可选)

毛利 = charge - cost   （恒 > 0，详见 §10.4 防亏损硬约束）
```

`加价率 / min_margin / 固定加价` 全部按 §10.4 的 `pricing_markup` 多维度配置（企业>套餐>模型>家族>全局）。图片/语音按次或按时长计价，同样套用加价。

`markup` 可按企业/套餐配置（如成本价 ×1.3）。图片/语音按次或按时长计价。

调用链路（计费 + 限流落点）：

```
客户端
  → 业务网关(鉴权/限流/模型白名单/预冻结)
    → LiteLLM(负载均衡/转发上游)
      → 上游供应商
    ← 返回 usage
  ← 按实际 usage 结算扣费 → 写账单(异步 Kafka) → 释放冻结差额
返回客户端
```

实时扣费走 Redis（保证性能），账单明细异步落 PostgreSQL（Kafka 削峰）。

## 11.2 Token 账单 + 调用日志 DDL

```sql
-- Token 账单(可按月分区，亿级)
CREATE TABLE bill_token (
    id              BIGSERIAL PRIMARY KEY,
    request_id      VARCHAR(40),
    company_id      BIGINT NOT NULL,
    user_id         BIGINT,
    project_id      BIGINT,
    api_key_id      BIGINT,
    model_code      VARCHAR(64),
    channel_id      BIGINT,
    prompt_tokens   INT,
    completion_tokens INT,
    cached_tokens   INT,
    total_tokens    INT,
    cost            NUMERIC(18,6),                       -- 对客费用
    upstream_cost   NUMERIC(18,6),                       -- 成本(利润分析)
    created_at      TIMESTAMPTZ DEFAULT now()
) PARTITION BY RANGE (created_at);

-- 调用日志
CREATE TABLE call_log (
    id            BIGSERIAL PRIMARY KEY,
    request_id    VARCHAR(40) UNIQUE,
    api_key_id    BIGINT,
    company_id    BIGINT,
    model_code    VARCHAR(64),
    channel_id    BIGINT,
    ip            VARCHAR(45),
    user_agent    VARCHAR(512),
    http_status   INT,
    biz_status    VARCHAR(40),                            -- success/rate_limited/insufficient_balance...
    response_time INT,                                    -- ms
    is_stream     BOOLEAN,
    created_at    TIMESTAMPTZ DEFAULT now()
) PARTITION BY RANGE (created_at);
```

## 11.3 账单聚合

`今日 / 昨日 / 本月 / 累计` 消费。实时数走 Redis 计数器，历史报表用 ClickHouse 或 PG 物化视图按天/项目/模型聚合。

---

# 十二、风控系统

## 12.1 限流（多维度）

| 维度 | 算法 | 存储 |
|---|---|---|
| API Key RPM/TPM | 滑动窗口 + 令牌桶 | Redis Lua 原子操作 |
| 企业级总限流 | 令牌桶 | Redis |
| 用户级限流 | 滑动窗口 | Redis |
| IP 限流 | 固定窗口 | Redis |

超限返回 `429 Too Many Requests`，带 `Retry-After` 头。

```lua
-- Redis 令牌桶 Lua（原子）
local key=KEYS[1]; local rate=tonumber(ARGV[1]); local cap=tonumber(ARGV[2])
local now=tonumber(ARGV[3]); local req=tonumber(ARGV[4])
local b=redis.call('HMGET',key,'tokens','ts')
local tokens=tonumber(b[1]) or cap; local ts=tonumber(b[2]) or now
tokens=math.min(cap, tokens+(now-ts)*rate)
local ok=0; if tokens>=req then tokens=tokens-req; ok=1 end
redis.call('HMSET',key,'tokens',tokens,'ts',now); redis.call('EXPIRE',key,3600)
return ok
```

## 12.2 黑名单

`IP 黑名单 / 国家黑名单(GeoIP) / 用户黑名单`。命中直接 `403`。

## 12.3 风险监控

实时检测：刷接口、异常 Token 消耗（突增 N 倍）、高频请求、代理/数据中心 IP、同 Key 多地登录。
处置：自动限流 → 临时封禁 → 告警超管。规则引擎可配置阈值。

```sql
CREATE TABLE risk_rule (
    id          BIGSERIAL PRIMARY KEY,
    rule_type   VARCHAR(40),                             -- token_spike/high_freq/proxy_ip
    threshold   JSONB,
    action      VARCHAR(20),                             -- alert/throttle/ban
    status      SMALLINT DEFAULT 1
);
CREATE TABLE risk_blacklist (
    id        BIGSERIAL PRIMARY KEY,
    type      VARCHAR(20),                               -- ip/country/user
    value     VARCHAR(64),
    reason    VARCHAR(255),
    expire_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ DEFAULT now()
);
```

---

# 十三、通知系统

通道：邮件 / 短信 / 站内信 / Webhook。

场景：充值成功、余额不足（阈值预警）、套餐到期、API 异常、提现状态、异地登录、风控告警。

模板化 + 异步（Kafka 解耦），Webhook 带签名 + 失败重试（指数退避）。

```sql
CREATE TABLE notify_log (
    id        BIGSERIAL PRIMARY KEY,
    company_id BIGINT,
    channel   VARCHAR(20),                               -- email/sms/inmail/webhook
    scene     VARCHAR(40),
    target    VARCHAR(255),
    content   TEXT,
    status    SMALLINT,                                  -- 0待发 1成功 2失败
    retry     INT DEFAULT 0,
    created_at TIMESTAMPTZ DEFAULT now()
);
```

---

# 十四、运营后台模块

```
Dashboard（实时调用量/收入/活跃企业/渠道健康）
用户管理 / 企业管理 / 企业认证审核
模型管理（家族/版本/定价/上下架/可用状态）
渠道管理（账号池/权重/熔断/额度）
套餐管理
支付通道配置中心（开关/密钥/手续费/金额限制/汇率/到账确认方式，实时生效）
财务管理（充值/提现审核/对账/利润分析）
账单管理（Token账单/调用日志/导出）
风控中心（规则/黑名单/告警）
通知中心（模板/记录）
系统设置（参数/密钥/汇率/加价系数）
```

---

# 十五、技术架构

## 15.1 技术栈

| 层 | 技术 |
|---|---|
| 前端 | Vue3 + TypeScript + Pinia + Element Plus + Vite |
| 后端 | Spring Boot 3.x + Spring Security + MyBatis-Plus |
| 模型网关 | **LiteLLM**（独立服务） |
| 缓存/限流 | Redis 7（Cluster） |
| 消息队列 | Kafka（账单/通知/风控削峰） |
| 主库 | PostgreSQL 16（账单按月分区，读写分离） |
| 分析库 | ClickHouse（账单聚合报表，可选） |
| 对象存储 | MinIO（营业执照/头像） |
| 监控 | Prometheus + Grafana + ELK |
| 部署 | Docker + Kubernetes + Nginx |
| 网关 | Spring Cloud Gateway / APISIX（鉴权+限流前置） |

## 15.2 部署拓扑

```
                    ┌──────────────┐
   客户端/SDK  ───▶ │ Nginx / LB    │
                    └──────┬───────┘
            ┌──────────────┼───────────────┐
            ▼              ▼                ▼
     ┌───────────┐  ┌────────────┐  ┌────────────┐
     │ API网关层  │  │ 管理后台API │  │ 前端静态资源 │
     │(鉴权/限流) │  └────────────┘  └────────────┘
     └─────┬─────┘
           ▼
     ┌───────────┐   ┌──────────┐
     │  LiteLLM   │──▶│ 上游供应商 │
     └───────────┘   └──────────┘
           │
     ┌─────┴───────────────────────┐
     ▼          ▼          ▼        ▼
   Redis      Kafka    PostgreSQL  MinIO
```

## 15.3 关键设计要点

- **计费/限流在业务网关层**，LiteLLM 只做转发与底层负载均衡，职责清晰。
- 账单写入走 Kafka 异步，避免阻塞调用主链路。
- API Key 鉴权信息 Redis 缓存（TTL + 失效主动清除），避免每次查库。
- 余额预冻结 + 实际结算，乐观锁防超额。
- 全链路 `request_id` 贯穿，便于排障与对账。

## 15.4 ⭐ 配置中心（大量配置入库 + 后台 + 实时生效，免改 yaml/重启）

> 你的要求：尽量不写死、不改 yaml，能后台配的都后台配。
> 做法：所有可变项进 DB 配置表 + Redis 缓存，后台改完发布失效事件，**实时生效不重启**。

```sql
CREATE TABLE sys_config (
    config_key   VARCHAR(64) PRIMARY KEY,               -- 配置键
    config_value JSONB,                                 -- 配置值
    group_name   VARCHAR(40),                           -- 分组：pay/model/risk/markup/notify/system
    remark       VARCHAR(255),
    updated_by   BIGINT,
    updated_at   TIMESTAMPTZ DEFAULT now()
);
```

已配置化的内容总览（均在后台可改）：支付通道（§10.2.1 `pay_channel_config`）、模型版本与定价（`model_version`）、加价规则（§10.4 `pricing_markup`）、入金维护费（`recharge_fee_config`）、渠道账号与权重（`channel`）、套餐（`plan`）、限流/黑名单/风控规则（§12）、通知模板、汇率安全系数、上游余额预警阈值、自动同步开关与频率等。**代码不写死，LiteLLM 走动态注册或 DB 路由，不再手维护 yaml。**

## 15.5 ⭐ 定时任务清单（Scheduled Jobs）

| 任务 | 频率 | 作用 | 关联 |
|---|---|---|---|
| ModelSyncJob | 每 6 小时 | 各供应商 /models 拉模型，新模型默认下架待定价 | §7.6 |
| PriceSyncJob | 每日 | 同步成本价(OpenRouter等)、检测上游涨价→自动调对客价/告警 | §10.4 |
| ExchangeRateJob | 每小时 | 同步汇率（×安全系数缓冲） | §10.4 |
| UpstreamBalanceMonitorJob | 每 5 分钟 | 监控你在上游的余额，低额告警/熔断/(可选)自动补给 | §8.3 |
| UpstreamAutoRechargeJob | 每 5 分钟 | 上游余额低 → 从 upstream_reserve 自动回补上游账户（消费即回补闭环） | §8.4 |
| OrderQueryJob | 每分钟 | 未支付充值订单主动查单，防回调丢失掉单 | §10.2.1 |
| ReconcileJob | 每日 | 各支付通道对账，差异告警 | §10.2.1 F |
| ProfitReportJob | 每日 | 生成利润日报（收入/成本/毛利/毛利率） | §10.4 F |
| BillAggregateJob | 每日 | 账单聚合到报表库 | §11.3 |
| KeyQuotaResetJob | 每日 0 点 | 重置 API Key 日额度计数 | §6.2 |
| LowBalanceAlertJob | 每 10 分钟 | 客户余额不足预警通知 | §13 |
| PartitionCreateJob | 每月 | 预建账单/日志下月分区 | §11.2 |

所有任务用分布式调度（XXL-JOB / ElasticJob）防多实例重复执行；任务开关与频率写在 `sys_config`，后台可调。

---

# 十六、可立即运行的部署配置

## 16.1 docker-compose.yml（一键起本地环境）

```yaml
version: "3.9"
services:
  postgres:
    image: postgres:16
    environment:
      POSTGRES_DB: ai_gateway
      POSTGRES_USER: gw
      POSTGRES_PASSWORD: ${PG_PASSWORD}
    ports: ["5432:5432"]
    volumes: ["pgdata:/var/lib/postgresql/data"]

  redis:
    image: redis:7
    command: redis-server --requirepass ${REDIS_PASSWORD}
    ports: ["6379:6379"]

  kafka:
    image: bitnami/kafka:3.7
    environment:
      KAFKA_CFG_NODE_ID: 0
      KAFKA_CFG_PROCESS_ROLES: controller,broker
      KAFKA_CFG_CONTROLLER_QUORUM_VOTERS: 0@kafka:9093
      KAFKA_CFG_LISTENERS: PLAINTEXT://:9092,CONTROLLER://:9093
    ports: ["9092:9092"]

  minio:
    image: minio/minio
    command: server /data --console-address ":9001"
    environment:
      MINIO_ROOT_USER: ${MINIO_USER}
      MINIO_ROOT_PASSWORD: ${MINIO_PASSWORD}
    ports: ["9000:9000","9001:9001"]
    volumes: ["miniodata:/data"]

  litellm:
    image: ghcr.io/berriai/litellm:main-latest
    command: --config /app/config.yaml --port 4000
    volumes: ["./litellm-config.yaml:/app/config.yaml"]
    environment:
      OPENAI_KEY: ${OPENAI_KEY}
      ANTHROPIC_KEY_1: ${ANTHROPIC_KEY_1}
      GEMINI_KEY: ${GEMINI_KEY}
      DEEPSEEK_KEY: ${DEEPSEEK_KEY}
      OPENROUTER_KEY: ${OPENROUTER_KEY}
    ports: ["4000:4000"]

  backend:
    image: your-registry/ai-gateway-backend:latest
    depends_on: [postgres, redis, kafka, litellm]
    environment:
      SPRING_PROFILES_ACTIVE: prod
      DB_URL: jdbc:postgresql://postgres:5432/ai_gateway
      DB_USER: gw
      DB_PASSWORD: ${PG_PASSWORD}
      REDIS_HOST: redis
      REDIS_PASSWORD: ${REDIS_PASSWORD}
      KAFKA_BROKERS: kafka:9092
      LITELLM_BASE: http://litellm:4000
      STRIPE_SECRET: ${STRIPE_SECRET}
      STRIPE_WEBHOOK_SECRET: ${STRIPE_WEBHOOK_SECRET}
      WISE_API_TOKEN: ${WISE_API_TOKEN}
      WISE_PROFILE_ID: ${WISE_PROFILE_ID}
      ALIPAY_APP_ID: ${ALIPAY_APP_ID}
      WECHAT_MCH_ID: ${WECHAT_MCH_ID}
      JWT_SECRET: ${JWT_SECRET}
    ports: ["8080:8080"]

  frontend:
    image: your-registry/ai-gateway-frontend:latest
    ports: ["80:80"]
    depends_on: [backend]

volumes:
  pgdata:
  miniodata:
```

## 16.2 .env 模板

```bash
# 数据库 / 缓存
PG_PASSWORD=change_me_strong
REDIS_PASSWORD=change_me_strong
MINIO_USER=admin
MINIO_PASSWORD=change_me_strong

# 安全
JWT_SECRET=$(openssl rand -base64 48)

# 上游模型供应商
OPENAI_KEY=sk-...
ANTHROPIC_KEY_1=sk-ant-...
GEMINI_KEY=...
DEEPSEEK_KEY=...
OPENROUTER_KEY=sk-or-...

# 支付：Stripe（同时覆盖 Wise 卡等国际卡收款）
STRIPE_SECRET=sk_live_...
STRIPE_WEBHOOK_SECRET=whsec_...

# 出金：Wise Platform
WISE_API_TOKEN=...
WISE_PROFILE_ID=...

# 国内支付
ALIPAY_APP_ID=...
ALIPAY_PRIVATE_KEY=...
WECHAT_MCH_ID=...
WECHAT_API_V3_KEY=...

# USDT 网关（如用第三方）
USDT_GATEWAY_KEY=...
```

## 16.3 启动顺序

```bash
# 1. 准备配置
cp .env.example .env && vim .env
# 2. 放置 litellm-config.yaml（见 7.4）
# 3. 起依赖 + 服务
docker compose up -d
# 4. 执行数据库初始化 DDL（本文档各表）
psql -h localhost -U gw -d ai_gateway -f init.sql
# 5. 访问后台 http://localhost ，初始化超管账号
# 6. 后台配置：渠道密钥 → 模型版本与定价 → 套餐 → 上架
# 7. 客户端用 sk-xxx 调用 http://localhost:8080/v1/chat/completions
```

> 配置 Stripe 后，**Wise 卡及全部国际卡自动支持收款**；提现侧填好 Wise Platform Token 即可全球出金。

---

# 十七、性能与可用性目标

| 指标 | 目标 |
|---|---|
| 注册用户 | 100 万 |
| 企业 | 10 万 |
| API Key | 100 万 |
| 峰值 QPS | 10,000 |
| 日调用量 | 1 亿次 |
| 账单记录 | 亿级（按月分区） |
| API 鉴权延迟 | < 5ms（Redis 缓存） |
| 网关额外延迟 | < 30ms（不含上游） |
| 可用性 | 99.9% |
| 扩容 | 无状态服务水平扩容 + DB 读写分离/分区 |

支撑手段：无状态网关 K8s HPA 自动扩缩；Redis Cluster；PG 分区 + 读写分离；Kafka 削峰；ClickHouse 扛报表查询；多渠道故障转移保障上游可用性。

---

# 十八、安全要求

- 上游 API Key、支付密钥 AES-256 加密落库，密钥走 KMS / Vault
- 客户端 API Key 仅存哈希，明文只展示一次
- 全站 HTTPS / TLS1.2+；HSTS
- 支付回调强制验签 + 幂等
- 敏感操作二次验证 + 操作审计日志
- 越权防护：行级 `company_id` 隔离 + 接口级权限点
- 风控：限流 / 黑名单 / 异常检测 / 自动封禁
- 数据合规：营业执照等 PII 加密存储，符合所在地法规

---

# 十九、上线检查清单（Go-Live Checklist）

- [ ] 所有 DDL 执行，分区表预建未来 3 个月分区
- [ ] LiteLLM 各模型版本路由配置 + 至少 2 渠道做故障转移
- [ ] 渠道密钥配置并通过健康探测
- [ ] 模型版本定价 / 加价系数 / 汇率配置完成
- [ ] ModelSyncJob 跑通，新模型自动入库且默认下架待定价
- [ ] 每个上架模型均已设「成本价 + 对客价」，对客价 ≥ 成本 ×(1+min_margin)（防倒挂自检）
- [ ] pricing_markup 最低毛利下限 min_margin 已配置（防亏硬约束）
- [ ] 上游账户已充值，UpstreamBalanceMonitorJob 告警/熔断验证通过
- [ ] 利润看板数据正确（收入/成本/毛利/毛利率），低毛利告警生效
- [ ] PriceSyncJob 上游涨价 → 自动调价/告警验证
- [ ] 套餐配置并上架
- [ ] Stripe 生产密钥 + Webhook 配置并验签（含 Wise 卡测试支付）
- [ ] Wise Platform 出金通道联调通过
- [ ] 支付宝/微信/USDT 回调联调通过 + 幂等验证
- [ ] 限流/黑名单/风控规则启用
- [ ] 通知通道（邮件/短信/Webhook）联调
- [ ] Prometheus/Grafana 监控面板 + 告警规则
- [ ] 压测达 10k QPS，HPA 自动扩缩验证
- [ ] 备份策略（PG 每日全量 + WAL）与演练
- [ ] 安全扫描（越权/注入/密钥泄露）通过

---

# 二十、客户端 API 调用与集成指南（对外消费方如何使用）

> 前面讲的是平台内部如何运作。本章讲**客户拿到 `sk-xxx` 之后，如何把本平台的 API 接进自己的产品**（客服机器人、知识库、代码助手等），完全兼容 OpenAI SDK，迁移成本极低。

## 20.1 三步接入

```
1. 在企业后台「API Key」创建一个 Key → 复制 sk-xxxxxxxx（只显示一次）
2. 把 base_url 指向平台： https://api.your-gateway.com/v1
3. 用 OpenAI 官方 SDK 或任意 HTTP 客户端调用（model 字段填想用的版本）
```

迁移已有 OpenAI 项目：**只改两行**——`base_url` 和 `api_key`，其余代码不动。

## 20.2 鉴权

| Header | 必填 | 说明 |
|---|---|---|
| `Authorization: Bearer sk-xxx` | ✅ | 平台 API Key |
| `X-Project-Id: <项目ID>` | 可选 | 指定成本归集到某项目（不传用 Key 默认项目） |
| `Content-Type: application/json` | ✅ | — |

## 20.3 端点与示例

### ① 对话 Chat Completions（非流式）

```bash
curl https://api.your-gateway.com/v1/chat/completions \
  -H "Authorization: Bearer sk-xxxxxxxx" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "claude-opus-4-8",
    "messages": [
      {"role":"system","content":"你是客服助手"},
      {"role":"user","content":"帮我查一下退货政策"}
    ],
    "temperature": 0.7,
    "max_tokens": 1024
  }'
```

响应（OpenAI 兼容）：

```json
{
  "id": "chatcmpl-xxx",
  "object": "chat.completion",
  "model": "claude-opus-4-8",
  "choices": [{"index":0,"message":{"role":"assistant","content":"我们的退货政策是…"},"finish_reason":"stop"}],
  "usage": {"prompt_tokens":42,"completion_tokens":88,"total_tokens":130}
}
```

### ② 对话（流式 SSE）

请求加 `"stream": true`，服务端按 `text/event-stream` 推送：

```
data: {"choices":[{"delta":{"content":"我"}}]}
data: {"choices":[{"delta":{"content":"们的"}}]}
data: {"choices":[{"delta":{"content":"退货"}}]}
...
data: [DONE]
```

### ③ 向量 Embeddings（做知识库检索）

```bash
curl https://api.your-gateway.com/v1/embeddings \
  -H "Authorization: Bearer sk-xxx" -H "Content-Type: application/json" \
  -d '{"model":"text-embedding-3-large","input":["退货流程是什么"]}'
```

### ④ 文生图 / 语音 / 重排序

```bash
# 文生图
POST /v1/images/generations   {"model":"gpt-image-1","prompt":"a red panda","size":"1024x1024"}
# 语音转文字（multipart 上传音频）
POST /v1/audio/transcriptions  file=@audio.mp3  model=whisper-1
# 文字转语音
POST /v1/audio/speech          {"model":"tts-1","input":"你好","voice":"alloy"}
# 重排序
POST /v1/rerank                {"model":"rerank-v3","query":"...","documents":["...","..."]}
```

### ⑤ 列出可用模型版本

```bash
curl https://api.your-gateway.com/v1/models -H "Authorization: Bearer sk-xxx"
```
返回**该 Key 被授权的模型版本**（含 display_name / tagline / availability，见 §7.3），客户端可据此渲染模型选择器。

## 20.4 选择模型版本

直接在 `"model"` 字段传 `model_code`，即可切换具体版本：

```
claude-opus-4-8      → 复杂任务
claude-sonnet-4-6    → 日常高效
claude-haiku-4-5     → 极速/低成本
gpt-5 / gpt-4o / gemini-2.5-pro / deepseek-v3 ...
```

调用不在授权白名单的版本 → 返回 `403 model_not_allowed`。

## 20.5 进阶能力

### 工具调用 / Function Calling

```json
{
  "model": "claude-opus-4-8",
  "messages": [{"role":"user","content":"北京今天天气？"}],
  "tools": [{
    "type":"function",
    "function":{"name":"get_weather","parameters":{"type":"object","properties":{"city":{"type":"string"}}}}
  }]
}
```

### 视觉（图片输入）

```json
{"model":"claude-opus-4-8","messages":[{"role":"user","content":[
  {"type":"text","text":"这张图是什么？"},
  {"type":"image_url","image_url":{"url":"https://.../cat.jpg"}}
]}]}
```

## 20.6 各语言 SDK 示例

```python
# Python (openai sdk)
from openai import OpenAI
client = OpenAI(base_url="https://api.your-gateway.com/v1", api_key="sk-xxx")
r = client.chat.completions.create(model="claude-opus-4-8",
        messages=[{"role":"user","content":"hi"}], stream=True)
for chunk in r:
    print(chunk.choices[0].delta.content or "", end="")
```

```javascript
// Node.js
import OpenAI from "openai";
const client = new OpenAI({ baseURL: "https://api.your-gateway.com/v1", apiKey: "sk-xxx" });
const res = await client.chat.completions.create({
  model: "gpt-5", messages: [{ role: "user", content: "hi" }]
});
```

```go
// Go (sashabaranov/go-openai)
cfg := openai.DefaultConfig("sk-xxx")
cfg.BaseURL = "https://api.your-gateway.com/v1"
client := openai.NewClientWithConfig(cfg)
```

> LangChain / LlamaIndex / Cursor / Dify 等工具：均支持自定义 `base_url + api_key`，填上即可把本平台当作 OpenAI 直接替换（drop-in）。

## 20.7 用量与计费回显

每次响应体含 `usage`（token 数）；响应头额外回显：

| 响应头 | 含义 |
|---|---|
| `X-Request-Id` | 请求唯一ID，排障/对账用 |
| `X-Tokens-Used` | 本次消耗 token |
| `X-Cost` | 本次扣费（对客价） |
| `X-Balance` | 调用后账户余额 |
| `X-RateLimit-Limit` / `X-RateLimit-Remaining` / `X-RateLimit-Reset` | 限流配额 |

## 20.8 错误码（OpenAI 兼容结构）

```json
{"error":{"type":"insufficient_balance","message":"账户余额不足","code":402}}
```

| HTTP | type | 含义 | 客户端处理 |
|---|---|---|---|
| 401 | `invalid_api_key` | Key 无效/禁用/过期 | 检查 Key |
| 402 | `insufficient_balance` | 余额不足 | 提示充值 |
| 403 | `model_not_allowed` | 该 Key 无此模型权限 | 换模型/申请权限 |
| 404 | `model_not_found` | 模型不存在/已下架 | 换模型 |
| 429 | `rate_limited` | 触发限流 | 按 `Retry-After` 重试 |
| 503 | `model_in_maintenance` | 模型维护中 | 稍后/换模型 |
| 502 | `all_channels_unavailable` | 上游全不可用 | 重试/换模型 |
| 500 | `internal_error` | 平台异常 | 携 `X-Request-Id` 反馈 |

## 20.9 调用最佳实践

- **重试**：对 429/5xx 指数退避重试（如 1s/2s/4s），尊重 `Retry-After`。
- **流式**：长回答用 `stream:true` 改善体验、避免超时。
- **超时**：客户端超时建议 ≥ 120s（大模型生成较慢）。
- **成本归集**：多业务用 `X-Project-Id` 或多个 Key 分项目，便于看各业务花费。
- **留痕**：保存 `X-Request-Id`，出问题可向平台溯源。
- **密钥安全**：Key 只放服务端，不要泄露到前端/客户端代码。

## 20.10 典型集成场景（客户怎么用出去）

| 场景 | 用到的端点 | 做法 |
|---|---|---|
| 客服机器人 | `/chat/completions`(stream) | system 设人设 + 多轮对话，流式回显 |
| 知识库问答(RAG) | `/embeddings` + `/chat/completions` | 文档切片→embedding入向量库；提问时检索→拼上下文→对话 |
| 代码助手 | `/chat/completions` + tools | 传代码上下文 + function calling 执行工具 |
| 营销文案 | `/chat/completions` | 批量 prompt 生成，用便宜版本(haiku/flash)降本 |
| 多模态 | vision / `/images/generations` | 图片理解 + 文生图 |

**RAG 最小示例（客户侧）：**
```python
# 1. 建库：文档→embedding→存向量库
emb = client.embeddings.create(model="text-embedding-3-large", input=chunks)
# 2. 提问：检索相关片段 → 拼进 prompt → 调对话
ctx = vector_db.search(query_emb, top_k=5)
ans = client.chat.completions.create(model="claude-sonnet-4-6",
    messages=[{"role":"system","content":f"参考资料：{ctx}"},
              {"role":"user","content":question}])
```

> 一句话：客户把本平台当成"一个能调所有大模型、还能切版本、统一计费"的 OpenAI 端点，照常写自己的应用即可。

---

# 二十一、迭代里程碑建议

| 阶段 | 周期 | 交付 |
|---|---|---|
| M1 MVP | 4-6 周 | 用户/企业/Key/单渠道转发/Stripe充值/基础账单 |
| M2 商业化 | 4 周 | 多渠道负载均衡+故障转移/套餐/多支付(含Wise卡)/计费引擎 |
| M3 企业级 | 4 周 | RBAC/项目/风控/限流/提现/通知 |
| M4 规模化 | 持续 | 分区+读写分离/ClickHouse/HPA/监控告警/SLA |

---

> 本文档为可落地实施版。按第十六章即可起本地环境验证主链路；生产上线前完成第十九章清单。
