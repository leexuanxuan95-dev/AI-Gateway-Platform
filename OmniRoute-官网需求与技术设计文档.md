# OmniRoute 官网 —— 需求与技术设计文档（独立）

> 版本：v1.0 ｜ 状态：可落地实施版
> 配套交付：`OmniRoute-官网.html`（设计与渲染模板参考）
> 关系：本文档是「AI Gateway SaaS 平台」主文档的子模块，独立成文，便于单独排期开发。
> 核心原则：**官网所有内容、配色、文案、套餐、FAQ 等均可在运营后台配置，前端读 API 渲染，不改代码即可上线变更。**

---

## 一、定位与目标

OmniRoute 官网是产品的对外门面与获客入口，单一目标：**让开发者/企业一眼看懂"一个 Key 调用所有大模型、可切版本"，并完成注册拿到 API Key。**

衡量指标：
- 首屏跳出率、注册转化率（CTA 点击 → 注册 → 首次创建 Key）
- 文档/快速开始点击率
- SEO 自然流量

非目标：本文档只覆盖**对外营销官网**；控制台/后台界面在主文档与单独的「控制台 UI」文档中描述。

---

## 二、信息架构（页面区块）

官网为单页长滚动 + 锚点导航，区块顺序如下（每个区块都对应后台一份可配置数据，见第四章）：

| # | 区块 | 作用 | 数据来源 |
|---|---|---|---|
| 1 | 顶部导航 Nav | 品牌、锚点、CTA | site_brand + nav |
| 2 | Hero 首屏 | 价值主张 + 路由控制台动效 + 主 CTA | section: hero |
| 3 | 供应商条 Providers | 信任背书（接入了哪些供应商） | section: providers |
| 4 | 核心能力 Features | 6 张能力卡 | section: features |
| 5 | 如何接入 How | 四步接入流程 | section: how |
| 6 | 代码示例 Code | Drop-in 替换（cURL/Python/Node） | section: code |
| 7 | 支持模型 Models | 家族 → 版本 | section: models（可绑定 model_version 自动同步）|
| 8 | 价格 Pricing | 套餐卡 | section: pricing（可绑定 plan 表自动同步）|
| 9 | 安全合规 Security | 4 张安全卡 | section: security |
| 10 | 常见问题 FAQ | 折叠问答 | faq |
| 11 | 行动召唤 CTA | 注册引导 | section: cta |
| 12 | 页脚 Footer | 链接、备案、版权 | footer |

> 区块可在后台**启用/停用、排序**；停用的区块前端不渲染。

---

## 三、内容清单（默认文案，均可后台改）

### Hero
- 主标题：一个 Key，接入**所有大模型**。
- 副标题：GPT-5、Claude Opus 4.8、Gemini、Grok、DeepSeek…… 在 `model` 字段一填即换版本。统一计费、统一额度、智能故障转移。完全兼容 OpenAI SDK，迁移只改两行。
- 主按钮：免费获取 API Key → 注册页
- 次按钮：看如何接入 → #how
- 徽标：9+ 供应商接入 · 数十个模型版本 · 多渠道充值（含 Wise 卡）

### 供应商
OpenAI / Anthropic / Google Gemini / xAI Grok / DeepSeek / Mistral / Cohere / OpenRouter / Azure OpenAI

### 核心能力（6）
1. 全模型聚合 —— 9+ 供应商一个入口，屏蔽协议差异。
2. 版本自由切换 —— 像选 Opus 4.8 / Sonnet 4.6 / Haiku 4.5 一样填 `model`。
3. 统一计费与额度 —— 按 token 实时计费，预冻结防超支，响应回显花费与余额。
4. 负载均衡 + 故障转移 —— 一家挂了自动切，客户端无感。
5. 企业多租户 + 风控 —— 四级隔离、RBAC、限流、黑名单、异常监控。
6. 多渠道充值 —— Stripe（含 Wise/国际卡）、支付宝、微信、USDT。

### 如何接入（四步）
01 创建 API Key → 02 base_url 指向 `api.omniroute.ai/v1` → 03 选版本调用 → 04 后台充值看账单。

### 代码示例
cURL / Python / Node 三段，强调"改两行"。

### 支持模型
Claude（opus-4-8 / sonnet-4-6 / haiku-4-5）、GPT（gpt-5 / gpt-4o）、Gemini（2.5-pro / 2.5-flash）、Grok（grok-3）、DeepSeek（v3 / r1）、更多（Mistral / Cohere / OpenRouter 全量）。

### 价格（4 档）
基础版 $0 / 专业版 $49 / 企业版 $299 / 旗舰版 议价。

### 安全合规（4）
密钥只存哈希、全链路 TLS + 回调验签、企业级数据隔离、预冻结防超支。

### FAQ（6）
怎么计费 / 迁移要改多少代码 / 支持哪些支付 / 模型挂了怎么办 / 能否切版本 / 数据是否安全。

### 页脚
产品 / 开发者 / 公司 三组链接 + 版权 + （如需）ICP 备案号。

---

## 四、⭐ 后台内容管理（Landing CMS）—— 核心需求

**官网不写死，全部在运营后台配置，前端读 API 渲染。**超管改完→保存草稿→预览→发布，前端经 CDN 刷新即实时生效，无需改代码或重新部署。

### 4.1 后台「官网管理」模块功能

| 子模块 | 可配置项 |
|---|---|
| 站点设置 | 品牌名、Logo、favicon、域名、默认语言、ICP/版权 |
| SEO | 各页 title / description / keywords / OG 图 / 站点地图开关 |
| 主题 | 配色 token（背景/主色 amber/辅色 mint/文字…）、字体、圆角 |
| 区块管理 | 各区块启用/停用、排序、标题、副标题 |
| Hero | 主/副标题、两个 CTA 文案+链接、徽标项、是否显示路由动效 |
| 能力卡 | 增删改：图标、标题、描述、排序 |
| 接入步骤 | 增删改：编号、标题、描述 |
| 代码示例 | 各语言代码片段、tab 顺序、域名变量 |
| 模型展示 | 手动维护 或 **绑定 model_version 自动同步**（推荐：上新自动显示）|
| 价格卡 | 手动维护 或 **绑定 plan 表自动同步**；高亮哪一档、按钮文案/链接 |
| 安全卡 | 增删改 |
| FAQ | 增删改、排序、默认展开项 |
| 页脚 | 链接分组、版权、备案号 |
| 多语言 | 中/英（可扩展）；每个文案字段多语言版本 |
| 发布管理 | 草稿 / 预览 / 发布 / 回滚（版本历史）|

### 4.2 数据模型

```sql
-- 站点级配置（品牌/SEO/主题/页脚，单条 JSON 即可）
CREATE TABLE site_setting (
    id          INT PRIMARY KEY DEFAULT 1,
    lang        VARCHAR(8) DEFAULT 'zh',
    brand       JSONB,        -- {name,logo,favicon,domain,copyright,icp}
    seo         JSONB,        -- {title,description,keywords,ogImage}
    theme       JSONB,        -- {bg,amber,mint,text,font,radius...}
    footer      JSONB,        -- 链接分组
    updated_by  BIGINT,
    updated_at  TIMESTAMPTZ DEFAULT now()
);

-- 区块（hero/features/how/code/models/pricing/security/cta…）
CREATE TABLE site_section (
    id          BIGSERIAL PRIMARY KEY,
    section_key VARCHAR(32),                 -- hero/features/...
    lang        VARCHAR(8) DEFAULT 'zh',
    enabled     BOOLEAN DEFAULT TRUE,
    sort        INT DEFAULT 0,
    title       VARCHAR(255),
    subtitle    VARCHAR(512),
    payload     JSONB,                        -- 该区块结构化内容（卡片数组/CTA/代码等）
    bind_source VARCHAR(20),                  -- 可选：model_version / plan（自动同步）
    updated_at  TIMESTAMPTZ DEFAULT now(),
    UNIQUE(section_key, lang)
);

-- FAQ 单列便于增删排序
CREATE TABLE site_faq (
    id        BIGSERIAL PRIMARY KEY,
    lang      VARCHAR(8) DEFAULT 'zh',
    question  VARCHAR(255),
    answer    TEXT,
    sort      INT DEFAULT 0,
    enabled   BOOLEAN DEFAULT TRUE
);

-- 发布版本（草稿/已发布快照，支持回滚）
CREATE TABLE site_publish (
    id          BIGSERIAL PRIMARY KEY,
    lang        VARCHAR(8),
    status      SMALLINT,                     -- 0草稿 1已发布
    snapshot    JSONB,                         -- 整站内容快照
    published_by BIGINT,
    published_at TIMESTAMPTZ DEFAULT now()
);
```

> 也可统一用主文档 §15.4 的 `sys_config`（group_name='site'）承载，二选一。推荐独立表，便于版本管理与多语言。

### 4.3 接口

后台管理（鉴权：超管）：
```http
GET    /admin/site                 # 取草稿内容（全部区块）
PUT    /admin/site/setting         # 改站点设置/主题/SEO
PUT    /admin/site/section/{key}   # 改某区块
POST   /admin/site/faq             # 增 FAQ
PUT    /admin/site/faq/{id}        # 改/排序
DELETE /admin/site/faq/{id}
POST   /admin/site/preview         # 生成预览（带 token 的预览链接）
POST   /admin/site/publish         # 发布（生成 snapshot + 刷 CDN）
POST   /admin/site/rollback/{id}   # 回滚到某历史版本
```

前端渲染（公开、只读、强缓存）：
```http
GET /api/site/landing?lang=zh      # 返回已发布的整站内容 JSON
```
返回结构示例：
```json
{
  "brand": {"name":"OmniRoute","logo":"...","domain":"omniroute.ai"},
  "seo": {"title":"...","description":"..."},
  "theme": {"amber":"#F6B53C","mint":"#57E3BE","bg":"#0B1719"},
  "sections": [
    {"key":"hero","enabled":true,"title":"一个 Key，接入所有大模型。",
     "payload":{"subtitle":"...","ctas":[{"text":"免费获取 API Key","href":"/signup"}],
                "badges":["9+ 供应商接入","..."],"showConsole":true}},
    {"key":"features","payload":{"cards":[{"icon":"hub","title":"全模型聚合","desc":"..."}]}},
    {"key":"pricing","bind_source":"plan","payload":{"highlight":"pro"}}
  ],
  "faq": [{"q":"怎么计费?","a":"..."}],
  "footer": {"groups":[...], "copyright":"© 2026 OmniRoute"}
}
```

### 4.4 生效机制

```
后台编辑 → 存草稿(site_section/site_faq…)
       → 预览(带token读草稿，不影响线上)
       → 发布：生成 snapshot 写 site_publish(status=1) → 刷新 CDN/缓存
前端：GET /api/site/landing 读「已发布 snapshot」→ 渲染
       内容走 CDN 强缓存(如 5min)或发布即主动失效，改完近实时生效
```

### 4.5 自动同步（少维护）

- 价格区块 `bind_source=plan`：直接读主文档 `plan` 表，后台改套餐 → 官网价格自动更新。
- 模型区块 `bind_source=model_version`：读 `model_version` 中"已上架"版本（主文档 §7.6 定时任务自动同步上游新模型）→ 官网模型列表自动更新，无需手改。

---

## 五、设计规范

视觉以「路由控制台」为母题，详见交付的 `OmniRoute-官网.html`。

- 配色 token（后台可改）：背景 `#0B1719`、主色 amber `#F6B53C`、辅色 mint `#57E3BE`、文字 `#E8F1EF`、描边 `#21454B`。
- 字体：标题 Space Grotesk、中文 Noto Sans SC、代码 IBM Plex Mono。
- **Signature 元素**：Hero 路由控制台 —— 一个 API 请求扇出到模型芯片，琥珀色高亮"已路由"循环切换（呼应控制台模型版本选择器）。后台可开关此动效。
- 结构化编号仅用于"四步接入"等真实序列。
- 动效克制：首屏入场、滚动揭示、hover 微交互；遵循 `prefers-reduced-motion`。

---

## 六、技术实现

| 项 | 方案 |
|---|---|
| 前端 | Vue3 + TypeScript + Vite（与主平台前端栈一致）；或纯静态 + 模板 |
| 渲染数据 | 启动/构建时拉 `GET /api/site/landing`，按 JSON 渲染区块组件 |
| SEO | **SSR / 预渲染（Nuxt 或构建期 SSG）**，保证首屏 HTML 含内容；输出 sitemap.xml、OG 标签 |
| 缓存/加速 | 静态资源 + landing JSON 走 CDN；发布时主动失效 |
| 多语言 | i18n，按 `lang` 取对应内容；路由 `/` 与 `/en` |
| 埋点 | CTA/注册转化埋点（GA/自建），后台看转化漏斗 |
| 表单 | 注册/联系销售提交到主平台 API |
| 可访问性 | 语义标签、键盘焦点、对比度达标、reduced-motion |
| 响应式 | 桌面/平板/手机；手机折叠导航、单列布局 |
| 性能目标 | 首屏 LCP < 2.5s，Lighthouse ≥ 90 |

渲染组件与区块一一对应：`HeroSection / FeaturesSection / PricingSection …`，每个组件吃对应 `section.payload`，区块顺序由 `sort` 决定。

---

## 七、与主平台的关系

- 官网消费主平台的：`plan`（价格）、`model_version`（模型展示）数据。
- CTA「获取 API Key / 注册」跳转主平台注册页；「文档」跳 API 文档站。
- 「官网管理」是主平台运营后台的一个模块（见主文档 §14 应新增「官网管理」入口）。

---

## 八、验收清单

- [ ] 官网所有文案/配色/区块均可在后台改，发布后前端近实时生效
- [ ] 草稿 / 预览 / 发布 / 回滚 全流程可用
- [ ] 价格区块绑定 `plan`、模型区块绑定 `model_version`，自动同步验证
- [ ] 区块可启用/停用/排序，停用即不渲染
- [ ] 多语言（中/英）切换正常
- [ ] SSR/预渲染生效，查看源码含首屏内容；OG/sitemap 正常
- [ ] 移动端、键盘可达、reduced-motion 通过
- [ ] Lighthouse 性能/可访问性/SEO ≥ 90
- [ ] CTA 转化埋点上报，后台可看漏斗
- [ ] CDN 缓存与发布失效联通

---

> 配合 `OmniRoute-官网.html` 作为设计与交互参考即可开发。后台「官网管理」模块建议与主平台同栈实现，数据表见第四章。
