# TikTok Skill 优化 TODO

| 项 | 内容 |
| --- | --- |
| 日期 | 2026-11-04 |
| 范围 | `tiktok-skill`（JS CLI Skill）+ 对照 `guaikei/server`（TikTok REST / MCP / 任务队列） |
| 用途 | 项目根目录主台账：问题清单 + 优化路线 + 勾选跟进 |
| 版本基线 | skill `1.1.0`；后端 TikTok Create/Get + `mcp/tiktok` |
| 归档副本 | 历史快照见 `docs/archive/`（以本文件为准） |

处理完将下方 `[ ]` 改为 `[x]`，可在条目后补 PR / commit。

---

## 跟踪清单（先改这里）

### P0

- [ ] **B-01** 后端搜索空结果应正式结束（勿一直「处理中」至 TIMEOUT）
- [ ] **A-03 / A-04** 统一 CLI 与 MCP 的等待上限与 TIMEOUT 续等文案；默认阻塞缩短 + 同参续等协议
- [ ] **A-01** 瘦身 `SKILL.md`（决策树置顶，细则进 references）
- [ ] **A-02** frontmatter description 中英双语 WHAT+WHEN
- [ ] **P-01** 降低 Token 开通摩擦（官网一键配置片段 / doctor）

### P1

- [x] **P-02** 包月 Token 上限与文档冲突（`14→PLAN_LIMIT`；双上限写进 SKILL/readme/options；版本 1.1.1）
- [x] **P-03** 其余数字错误码命名（`15/16` 余额等）→ `INSUFFICIENT_BALANCE` / `RATE_LIMITED` / `DAILY_LIMIT`（`next_action` 仍可选）
- [ ] **A-05** 消除 sort 同号异义（文档强化或字符串枚举）
- [ ] **A-06 / A-07 / A-09** 信封增强；`--agent`（无 Banner/ANSI、compact JSON）
- [ ] **A-08** Cursor / Codex / Workbuddy 适配说明
- [ ] **A-12 / P-04** `doctor` / smoke / 连通探测
- [ ] **B-02** Create 静默改写 vs Skill 失败即停：二选一对齐
- [ ] **B-03 / B-04 / B-05** 复用窗口文档、入队失败、费用预估 ceil（guaikei）
- [ ] **B-06 / A-13** URL/关键词规则与 MCP 对齐
- [ ] **E-01 / E-02** 单测 + CI；`GUAIKEI_BASE_URL` 可覆盖
- [ ] **S-01** 企业向数据外发说明
- [x] **P-05** 支持手机号移出 Skill 正文（仅留官网；不升版本号）

### P2

- [ ] **B-07** 短链解析（产品可选）
- [ ] **B-08 / B-09 / B-10** 播放地址 TTL 提示；字段形态统一；0 值策略
- [ ] **A-10 / A-11** 示例占位符；运行时 schema 抽检
- [ ] **E-03 / E-04** request_id；版本单源
- [ ] **P-06** 统一对外命名
- [ ] **S-02** Token 失效快速失败

### P3

- [ ] **B-11 / B-12 / E-05 / E-06** Header 标准化评估；MCP user_url；日志/代理边角

### 进度

| 优先级 | 条目约数 | 已完成 |
| --- | ---: | ---: |
| P0 | 5 | 0 |
| P1 | 15 | 1（P-02） |
| P2 | 6 | 1（P-05） |
| P3 | 1 组 | 0 |

---

## 0. 一句话结论

Skill 在「CLI 契约、失败即停、stdout 信封、路由表」上已经比较成熟，适合强模型（Claude / GPT / Codex）在 shell 里编排；真正卡住「普罗大众 + 多端 Agent + 中小模型」的，是：**异步等待过长且与 MCP 不一致、空搜索被伪装成 TIMEOUT、错误码仍是数字魔法、文档过长且偏中文、开通摩擦大、与 guaikei MCP 双轨未统一**。

---

## 1. 架构速览（对照）

```
用户 / Agent
  ├─ Cursor / Workbuddy / 其它：读 SKILL.md → shell 调 node CLI
  ├─ Codex / 脚本：直接 node scripts/tiktok/*-cli.js
  └─ （另一条线）guaikei MCP：tiktok_search / tiktok_post / tiktok_comment
        │
        ▼
  www.guaikei.com  REST
  POST /api/tiktok/{search/keyword|post/url|comment/url}
  GET  /api/tiktok/{search|post|comment}/info
        │
        ▼
  DB 任务行 status=1 → Redis/队列 → 采集 worker → status=2/3 → 计费
```

双轨现状：

| 面 | 入口 | 轮询 | 超时文案 | URL 归一 |
| --- | --- | --- | --- | --- |
| JS Skill | CLI + SKILL.md | ~15 分钟（间隔封顶 60s） | 「请稍后重试」 | 整段入参严格校验 |
| MCP | `mcp/tiktok` | ~1–2 分钟（间隔封顶 5s） | 「用完全相同参数再调」 | 可从长文本里抽出 URL |

同一后端、两套客户端语义 → Agent 行为必然分叉。

---

## 2. 问题全清单

优先级：`P0` 必须先处理 / `P1` 显著影响准确率或转化 / `P2` 质量与规模化 / `P3` 打磨。

状态列默认 `open`；关闭后改 `done`，并同步勾选上方「跟踪清单」。

### 2.1 产品与触达（面向普罗大众）

| ID | Pri | 状态 | 问题 | 影响 | 证据 / 位置 |
| --- | --- | --- | --- | --- | --- |
| P-01 | P0 | open | Token 开通依赖微信/官网自助，Skill 内无引导完成态检测以外的「一键开通」 | 大众用户装完技能就卡死；Agent 只能复读官网链接 | `SKILL.md` §0、`readme.md` §4 |
| P-02 | P1 | done | 包月 limit 上限（搜/作品 30、评论 40）与 CLI 允许 1–10000 落差大 | 模型常先开大 limit → errcode 14 → 多一轮失败 | 2026-11-04：对齐 douyin-search-claw；`backendCodes.js` 映射 `14→PLAN_LIMIT`；文档双上限；v1.1.1 |
| P-03 | P1 | done | 计费/余额/频控/日限映射为稳定码（errmsg 优先；搜索 16/频控 1 特例） | 弱模型可分支；`next_action` 仍可选 | 2026-11-09 v1.1.2 |
| P-04 | P2 | open | 价值主张偏「给 AI 的手」，但缺少「装完 60 秒看到第一条结果」的 onboarding 脚本 | 转化漏斗长 | 无 `scripts/doctor.js` / smoke |
| P-05 | P2 | done | 支持微信手机号写进 Skill 正文 | 隐私、骚扰面、企业合规观感差 | 2026-11-04：已从 SKILL/readme 去掉；仅留官网；不升版本 |
| P-06 | P3 | open | 命名三套：`tiktok-creator-videos` / `tiktok-skill` / 文案「洞察助手」 | 目录、registry、用户口口相传易乱 | `SKILL.md` name vs `package.json` |

### 2.2 Agent / 多模型可用性

| ID | Pri | 状态 | 问题 | 影响 | 证据 / 位置 |
| --- | --- | --- | --- | --- | --- |
| A-01 | P0 | open | `SKILL.md` 过长（~220 行），决策树、参数表、错误表、合规挤在一起 | GLM / Kimi / 小参数 DeepSeek / 上下文紧张的 Agent **丢指令、乱路由** | `SKILL.md` 全文 |
| A-02 | P0 | open | description / 正文几乎全中文；英文触发词弱 | GPT / Claude / Gemini / Codex / 英文 Workbuddy **召回不足或误拒** | frontmatter `description` |
| A-03 | P0 | open | 单次任务可阻塞至约 15 分钟；多数 Agent 工具超时远短于此 | 宿主杀进程 → 模型当成失败乱重试；与 MCP「约 2 分钟」体验冲突 | `constants.js` + `pollTaskResult`；`mcp/helper` `RetryIntervalMax=5s` |
| A-04 | P0 | open | TIMEOUT 策略在 JS Skill 文档与 MCP 文案相反倾向（Skill：搜索 TIMEOUT 勿立刻连打；MCP：必须同参再调） | 同用户换客户端就「学坏」；计费与复用窗口被打穿 | `SKILL.md` §0/§4；`mcp/helper.TimeoutMessage` |
| A-05 | P1 | open | `search --sort 1`（最多赞）与 `post --sort 1`（最热）同号异义 | 高发误用；中小模型尤其容易照搬 | `SKILL.md` §2 表格；两边 SCHEMA |
| A-06 | P1 | open | 无机器可读 `retry_policy` / `next_action` 字段 | Agent 只能读中文 message 猜下一步 | `scripts/utils/output.js` 信封 |
| A-07 | P1 | open | stderr 有 Banner + ANSI 色；空参既打 help 又出 error JSON | 占用 token；部分宿主把 stderr 混进上下文导致「念日志」 | `utils.js` `printBanner`；`*-cli.js` args.length===0 |
| A-08 | P1 | open | 无 Codex / Cursor / Workbuddy 专用薄适配层（agents 清单、tool schema 导出、安装说明） | 「普罗大众」换 Agent 产品就要人肉改路径 | 仅 OpenClaw 风格 metadata |
| A-09 | P1 | open | 成功 stdout 默认 pretty JSON（缩进） | 大结果烧上下文；Claude/GPT 长会话更痛 | `printSuccessEnvelope` |
| A-10 | P2 | open | 示例里 `@username` / 假 aweme_id 可被字面执行 | 弱模型不替换占位符 | `SKILL.md` examples |
| A-11 | P2 | open | Schema 只作文档，运行时不校验 results 形态 | 后端字段漂移时 Agent 静默吃到半残数据 | `assets/*.schema.json` |
| A-12 | P2 | open | 无「干跑 / 契约自检」命令 | 没 Token 时 Agent 无法验证自己会不会用 | 缺 `doctor` / `--dry-run` |
| A-13 | P3 | open | 关键词校验：MCP 禁 `&`，JS 不禁 | 双轨输入合法集不一致 | `mcp/tiktok/search.go` vs `validate/keyword.js` |

### 2.3 与 guaikei 后端契约

| ID | Pri | 状态 | 问题 | 影响 | 证据 / 位置 |
| --- | --- | --- | --- | --- | --- |
| B-01 | P0 | open | 搜索采集空结果时 `UpdateSearch` 直接 `errcode:3 结果为空`，**不落 status=2/空数组** | 客户端一直「处理中」直到 TIMEOUT；用户以为系统挂了；Skill 只能用文字解释 | `app/tiktok/search.go` UpdateSearch |
| B-02 | P0 | open | Create 侧非法 sort/time/limit **静默改写**；Skill 侧 **失败即停** | 绕过 Skill 直调 API 与 Skill 行为不一致；排障难 | `CreateSearchTask` 等 vs `assertSearchOptions` |
| B-03 | P1 | open | 任务复用/查询窗口多套：创建搜 9m、作品/评 20m；查询完成搜 15m、作品 30m；LoadAll 仅 10m status=1 | Skill 文档写「搜 9m / 作品评 20m」不完整；中间态丢队列后靠 LoadAll 也可能接不上 | `Create*` / `Get*Result` / `*LoadAll` |
| B-04 | P1 | open | `AddTask` 失败被忽略（后端已有 TODO H1） | DB 已有 status=1 → Skill 空转至 TIMEOUT | `guaikei/server/TODO.md` H1；`CreateSearchTask` |
| B-05 | P1 | open | 余额预估用整除（搜 `Limit/19`，作品评 `Limit/20`） | 小 limit 低估；门禁与实扣不一致 | `Create*Task` dayEstimate |
| B-06 | P1 | open | URL：Skill 要求整段是规范链接；MCP `extractTiktokURL` 可从句子抽链 | 用户说「看看这个 https://…」时 CLI 失败、MCP 成功 | `validate/url.js` vs `mcp/tiktok/url.go` |
| B-07 | P2 | open | 短链 /photo/ 两端都不支持，但无「解析短链」能力 | 大众分享链路几乎全是短链 → 转化断点 | 文档已声明；产品缺口 |
| B-08 | P2 | open | 无视频详情 API；播放地址可能过期且无 TTL 字段 | Agent 易幻觉出 detail；或拿过期 URL 当可播 | `SKILL.md` § 开头说明 |
| B-09 | P2 | open | 搜索 `play_addr` 字符串 vs 作品 `video_url` 数组 | 二次编排易写错字段 | schema + options.md |
| B-10 | P2 | open | omitempty 省略 0 / 空串 | 模型把「缺字段」当成「未知」或编造 | Go 结果结构体 `omitempty` |
| B-11 | P3 | open | Header 名 `TOKEN` 非 `Authorization` | 与生态习惯不符；部分网关/日志中间件不好挂 | `request.js` |
| B-12 | P3 | open | MCP comment 补 `user_url` 在仅有 sec_uid 时可能拼出非法路径 | 与 JS `profileUrl(user_uid)` 行为不一致 | `mcp/tiktok/comment.go` processCommentResults |

### 2.4 工程与质量

| ID | Pri | 状态 | 问题 | 影响 | 证据 / 位置 |
| --- | --- | --- | --- | --- | --- |
| E-01 | P1 | open | 零测试（无 `*.test.js`）、无 CI、无 `.gitignore` 规范可见 | 回归全靠人工；改校验规则易伤 Agent | 仓库扫描 |
| E-02 | P1 | open | `BASE_URL` 写死 `www.guaikei.com`，无 staging 覆盖 | 联调/预发困难 | `constants.js` |
| E-03 | P2 | open | 信封无 `request_id` / `task_hint` | 客服与日志无法对齐一次调用 | `output.js` |
| E-04 | P2 | open | 版本三处维护风险：`package.json` / `constants` fallback / SKILL frontmatter | 漂移后 Agent 按旧文档行动 | 已有读 package 逻辑，但 SKILL version 手写 |
| E-05 | P3 | open | Windows 日志权限依赖临时目录隔离（已文档化） | 多用户机边缘风险 | `log.js`；changelog |
| E-06 | P3 | open | 无依赖是优点，但也无标准 HTTP 客户端特性（连接复用、代理、HTTP/2） | 企业代理环境可能难用 | `https.request` |

### 2.5 合规与安全（Skill 侧可见部分）

| ID | Pri | 状态 | 问题 | 影响 | 证据 / 位置 |
| --- | --- | --- | --- | --- | --- |
| S-01 | P1 | open | 数据外发第三方写得偏软，缺企业向「最小字段 / 保留期 / 区域」说明 | ToB / 跨境团队不敢接 | `readme` §10 |
| S-02 | P2 | open | Token 格式校验通过后无本地「探测连通」；失效要等真请求 | Agent 会先跑完整轮询 | `token.js` |
| S-03 | P2 | open | 后端支付/鉴权等 Critical 项见 guaikei `TODO.md`（间接影响 Skill 可用性） | 充值异常、Token 软删仍可用等 | `guaikei/server/TODO.md` C*/H* |

---

## 3. 优化方向（宏观 → 可落地）

### 3.1 总原则

1. **一套契约，多端适配**：REST 结果语义以「信封 + 命名错误码 + next_action」为唯一真相；CLI Skill 与 MCP 只做传输壳。
2. **为最弱常用模型写 Skill**：假设模型会漏读长文、混淆同名参数、超时后乱重试——用短决策树 + 机器字段约束它。
3. **把长等待从 Agent 主路径拿开**：Agent 适合「提交 / 轻量轮询 / 续等」，不适合单工具卡 15 分钟。
4. **中英双语触发，中文深讲可选**：召回用英+中 description；细则放 `references/`。

### 3.2 路线图建议

#### Phase A — 契约对齐（1–2 周，P0）

| 做什么 | 怎么做 | 消解问题 |
| --- | --- | --- |
| 空搜索正式结束 | 后端空结果写 `status=2` + `[]`（或明确 empty status）；Skill 映射 `empty/NO_MATCH` | B-01 |
| 统一 TIMEOUT 续等策略 | 信封增加 `retryable` + `retry_with_same_args`；JS/MCP 文案与文档同文 | A-04, A-06 |
| 缩短默认阻塞 / 可续等 | CLI 增加 `--wait-ms` / 默认 90–120s；超时返回 `TIMEOUT` 但带 `continuation: same_args`；文档教 Agent 同参续等 | A-03 |
| 对齐 MCP 与 CLI 轮询上限说明 | 对外只承诺一种「建议等待」；实现可以不同但文案禁止打架 | A-03, A-04 |
| 关键词 / URL 规则对齐 | JS 与 MCP 共用同一套规则表（禁 `&`、从文本抽 URL 可选开关） | A-13, B-06 |

#### Phase B — Agent 友好 Skill（并行 1–2 周，P0/P1）

| 做什么 | 怎么做 | 消解问题 |
| --- | --- | --- |
| 瘦身 `SKILL.md` | 正文只留：何时用、三命令路由、3 条金例、退出码决策树；参数表/字段进 `references/` | A-01 |
| 双语 description | frontmatter WHAT+WHEN 中英并列；保留 TikTok 触发词 | A-02 |
| `--agent` 模式 | 无 Banner、无 ANSI、compact JSON、stderr 仅关键事件行 | A-07, A-09 |
| 命名错误码 | `MONTHLY_LIMIT` / `INSUFFICIENT_BALANCE` / `RATE_LIMITED`；数字仅作 `provider_code` | P-03, A-06 |
| `next_action` 枚举 | `ask_user` / `fix_token` / `reduce_limit` / `retry_same` / `retry_later` / `stop` | A-06 |
| 适配碎片 | 增加 `adapters/cursor.md`、`adapters/codex.md`、`adapters/workbuddy.md`（安装路径、env、超时建议） | A-08 |
| doctor | `node scripts/tiktok/doctor.js`：查 node 版本、Token 形态、可选 ping | P-04, A-12, S-02 |

#### Phase C — 后端稳健（与 guaikei 排期，P1）

| 做什么 | 消解问题 |
| --- | --- |
| Create 入队失败回滚/标失败 | B-04 |
| 预估费用 ceil，与实扣一致 | B-05 |
| 文档与实现统一复用窗口；LoadAll 覆盖查询窗 | B-03 |
| Create 对非法枚举改为拒绝（与 Skill 一致）或 Skill 明确「以服务端为准」二选一 | B-02 |
| 短链解析服务（可选付费） | B-07 |
| 结果字段补 `_meta.duration_unit` / `url_expires_hint` | B-08, B-09, B-10 |

#### Phase D — 大众转化与规模（P1/P2）

| 做什么 | 消解问题 |
| --- | --- |
| 官网「复制一段配置」：Cursor env / Codex config 一键片段 | P-01 |
| 默认推荐 limit 策略写进 Skill：未指定用 10；包月用户显式提示 ≤30/40 | P-02 |
| 支持渠道改为官网工单/社群链接，手机号移出 Skill 正文 | P-05 |
| 统一对外名与 npm/skill id | P-06 |
| 最小测试集：args / url / keyword / envelope / 错误码映射 | E-01 |
| `GUAIKEI_BASE_URL` 可覆盖 | E-02 |

---

## 4. 分模型策略（让 Skill「更好装、更少错」）

### 4.1 能力画像（写 Skill 时的假设，非跑分榜）

| 模型族 | 相对优势 | 对本 Skill 的典型失败模式 | Skill 侧对策 |
| --- | --- | --- | --- |
| **Claude**（含 Cursor 常用） | 长文档、工具纪律好 | 仍可能在 15min 超时后「换参重试」 | 短决策树 + `retry_same` 机器字段；`--agent` |
| **GPT** | 路由与英文指令强 | 中文细则漏读；爱用大 limit | 英中 description；默认 limit 强调；errcode 命名 |
| **Gemini** | 长上下文 | 易扫读跳过「短链不支持」 | 路由表置顶否定句；INVALID_URL 带 next_action |
| **Grok** | 跟热点、口语 | 可能把抖音/TikTok 混用 | description 明确「不要用于抖音」 |
| **DeepSeek** | 性价比、中文代码 | 长 SKILL 易丢；参数同号异义 | 必须瘦身；sort 写成 `sort_by=likes\|relevance` 字符串枚举（中长期） |
| **GLM** | 中文场景、国产栈 | 英文 registry 召回弱；对魔法数字弱 | 中文触发保留；错误码命名；少表格嵌套 |
| **Kimi** | 长上下文中文 | 「全要」倾向拉满 limit | 文档 + 运行时：未说数量强制 10；包月预检 |

### 4.2 分 Agent 产品

| Agent | 建议交付形态 | 注意 |
| --- | --- | --- |
| **Cursor** | 瘦 `SKILL.md` + scripts；env `GUAIKEI_API_TOKEN`；推荐 `--agent` | description 决定自动唤起；控制长度 |
| **Codex** | 同 CLI；另给 `adapters/codex.md`：工作目录、超时、续等 | 偏 shell；少依赖 MCP |
| **Workbuddy 等** | 优先提供 **MCP**（guaikei 已有）或同一信封的 HTTP；CLI 作 fallback | 工具超时通常更短 → 必须续等协议 |
| **Claude Desktop 等 MCP 宿主** | 直接 MCP；与 CLI 共享错误码与 next_action | 禁止两套 TIMEOUT 文案 |

### 4.3 中长期 API 形态（降低模型负担）

把易混的数字枚举升级为字符串（可双端兼容一版）：

- `sort`: `relevance` \| `most_liked`（search）；`latest` \| `popular`（post）
- `time`: `all` \| `1d` \| `7d` \| `30d` \| `90d` \| `180d`
- 保留旧数字为 deprecated alias

链式编排固定口令（写进 Skill 置顶）：

1. 搜 → 用 `aweme_id` 拉评论；用 `author_sec_uid` / `author_url` 拉作品  
2. **禁止**把 `author_id` / `user_id` 交给 post  
3. 用户未说数量 → limit=10  
4. `TIMEOUT` → 同参再来；`NO_MATCH` → 换词勿狂重试  

---

## 5. 建议的目标信封（示意，尚未实现）

```json
{
  "status": "error",
  "error_code": "TIMEOUT",
  "provider_code": "1",
  "message": "任务仍在处理中",
  "next_action": "retry_same",
  "retryable": true,
  "retry_with_same_args": true,
  "suggested_wait_seconds": 60,
  "request": { "command": "search", "keyword": "AI", "sort": 1, "time": 0, "limit": 20 },
  "metadata": {
    "skill_version": "1.2.0",
    "runtime_version": "22.x",
    "execution_time": 90000,
    "request_id": "optional"
  },
  "results": null
}
```

---

## 6. 已做得好的地方（避免误改）

- 失败即停的枚举校验（相对后端静默改写，更利于 Agent）
- stdout 统一信封；空结果 exit 0 + `NO_MATCH`
- 链接归一（www / https / 去 query）与短链明确拒绝
- 作者/评论者 ID 分流写清楚（sec_uid vs 数字 uid）
- 零 npm 依赖，易分发
- Auth 与明确业务错误不重试
- Schema 与 options/changelog 相对完整

---

## 7. 修订记录

| 日期 | 作者 | 说明 |
| --- | --- | --- |
| 2026-11-04 | Cursor Agent 审查 | 初版：问题全量 + 优化路线 + 多模型策略 |
| 2026-11-04 | — | 提升为项目根目录 `todo.md`（合并跟踪清单；`docs/archive/` 改为指向本文件） |
| 2026-11-04 | — | P-02 落地（对齐 douyin-search-claw）：`PLAN_LIMIT` + 文档双上限；版本 1.1.1 |
| 2026-11-04 | — | P-05：移除 SKILL/readme 中微信手机号，仅保留官网；不升版本 |
| 2026-11-14 | — | P1-10 方向 A：faq + 双壳例；版本 1.1.3 |
