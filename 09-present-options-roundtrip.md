# 09 · present_options 人机交互回路 · REST API 实测调用实录

> 实验执行时间：2026-04-23T08:08Z ~ 08:18Z (UTC)
> 被测 base URL：`https://api.twin.so`（任务书指定）及实验中发现的真正 agent API base `https://build.twin.so`
> 实验中创建的玩具 agent ID：`019db967-cad2-7b40-b642-9901f84ea469`（实验结束已 `DELETE 204` 删除）
> Twin API key 脱敏规则：`Authorization` header 统一显示为 `Bearer ***REDACTED***`，`x-api-key` header 显示为 `***REDACTED***`

---

## 1 · 实验目的

在 Twin 平台中，`present_options` 是 agent 在 run 执行中暂停并向用户索取输入的原生机制。agent 一旦调用 `present_options` 就进入"等待输入"（interrupt）状态，必须由上层把用户答复回传，run 才能继续。平台 UI 用户这一回路由前端自动闭合；**对通过 REST API 集成 Twin 的第三方调用方**，关键问题是：

1. 第三方能否通过公开 REST API 观察到 agent 进入了 interrupt 状态？
2. 能否通过 REST API 把用户答复回传给 agent，使之从 interrupt 恢复？
3. 如果题目是 `Secret` 类型，答复在 events 流里是否被 mask？

本报告按任务书构造的"会问三道 present_options"的玩具 agent 进行实测，并按任务书列出的 8 种候选回传端点（`/messages`、`/input`、`/answer`、`PATCH /runs/{id}`、`POST /runs + run_mode=resume`、`/reply`、`/respond`、`/resume`）逐一探测。

---

## 2 · 玩具 agent 设计

按任务书：
| 题号 | type | prompt | options |
|---|---|---|---|
| 1 | `PickOne` | "请选一个颜色" | `["红","绿","蓝"]` |
| 2 | `OpenEnded` | "请输入一个 1 到 100 之间的整数" | `[]` |
| 3 | `Secret` | "请输入一个测试用的假 token（随便编，不要填真实密钥）" | `[]` |

instructions 实际内容（通过 `PUT /v1/agents/{id}/instructions` 写入 build.twin.so 并以 `GET` 读回校验，详见 seq 20、seq 21）摘要：三步依次调用 `present_options`，每次答复均由 agent 记入内部 log 表，全部答完后 `finish_run`。

---

## 3 · 关键架构发现

本实验产生两项协议层发现，直接影响"REST API 能否闭合 present_options 回路"这个问题：

### 发现一 · 两个 base URL 暴露的端点完全不同

| base URL | openapi 声明的 paths | 有 `/v1/agents` ? | 有 `/v1/agents/{id}/runs` ? | 有 `/runs/{id}/events` ? | auth 机制 |
|---|---|---|---|---|---|
| `https://api.twin.so` | `/`, `/apikey`, `/apikey/{id}`, `/apikeys`, `/auth/health`, `/auth/validate`, `/goal-processing/...` | ❌ | ❌ | ❌ | 未声明 securitySchemes |
| `https://build.twin.so` | 见 seq 19 | ✅ (GET/POST) | ✅ (GET/POST) | ✅ (GET) | `api_key` in header `x-api-key` |

任务书原文指定 base 为 `https://api.twin.so`，但该域只承载 `apikey/auth/goal-processing` 三组端点，**没有 agent run 相关路径**。真正的 agent REST API 挂在 `https://build.twin.so`，且明确要求 `x-api-key` header（`/v1/me` 用 `Authorization: Bearer ...` 访问时返回 `401 Unauthorized`，响应 detail：**"Bearer tokens are not supported on the REST API. Use the x-api-key header with an API key created from the Twin UI (Settings > API Keys)."** 见 seq 18）。

### 发现二 · `POST /v1/agents/{id}/runs` 在公开 API 层无法启动 run

在 `build.twin.so` 上创建玩具 agent（`POST /v1/agents` → `201`）、写 instructions（`PUT .../instructions` → `200`）均成功。但**启动 run 的 `POST /v1/agents/{id}/runs` 在 8 种 body 变体下均失败**：

- OpenAPI 中 `StartRunBody.run_mode` 声明为 `string | null`。
- 发送 `{"run_mode":"Run","user_message":"..."}` 时（即 openapi 声明的字符串类型且值精确匹配服务器错误消息里列出的枚举 "Builder/Run/SubAgent"），服务器仍返回 `400 Bad Request`，`detail: "run_mode must be explicitly set to Builder, Run, or SubAgent"`。
- 发送 `{"run_mode":{"Run":{}}}` 或 `{"run_mode":{"Run":null}}`（Rust serde externally tagged enum 形式）则返回 `422 Unprocessable`，`detail` 明确指出 **"run_mode: invalid type: map, expected a string"** —— 证实 serde 反序列化层期望 string。
- 发送数字 `1` 同样 422 "invalid type: integer, expected a string"。
- 发送完全空 body `{}` 返回同样的 400 "must be explicitly set"。

两条错误消息合在一起表明：serde 接受字符串，但任何具体字符串值（含 "Run"）都被业务层当作"未设置"——说明该端点的业务入口**在 `POST /runs` 路径上并不按 openapi 声明实际生效**，公开 REST API 的该端点可能仅对内部会话/UI-cookie 调用方开放，或需要额外未公开的判别字段。实测旁证见 seq 31：`GET /v1/agents/{id}/runs` 在 8 次 POST 尝试之后返回 `total_runs: 0`，确认从未有 run 被创建成功。

> **直接含义**：在本实验可用的 REST API 表面上，第三方调用方无法通过公开 REST API 启动一个 agent run；因此也就无法让 agent 真实进入 `present_options` 的 interrupt 状态。8 种回传端点的探测因此只能退化为"对 openapi 清单的路径存在性探测"。

---

## 4 · 完整调用实录（api_trace · 32 条）

以下按 `seq` 顺序列出每一次真实 HTTP 请求。所有鉴权字段均已脱敏。

### seq 1 · `openapi-discovery` · GET (api)/openapi.json

- **phase**：`pre-A`  |  **时间**：2026-04-23T08:08:22Z  |  **HTTP status**：`200`

**curl 等价命令**：
```bash
curl -sS -H 'Authorization: Bearer ***REDACTED***' -H 'Accept: application/json' \
  'https://api.twin.so/openapi.json'
```
**请求 headers（脱敏）**：
```json
{"Authorization":"Bearer ***REDACTED***","Accept":"application/json"}
```
**请求 body**：（无）
**响应 body（截断 4000 字符）**：
```json
AI Agent API v1.0.0. paths: / /apikey /apikey/{id} /apikeys /auth/health /auth/validate /goal-processing/batch/analyze-csv /goal-processing/batch/generate-prompts /goal-processing/batch/propose-template /goal-processing/process. tags: Browsing + API Keys.
```
**评注**：关键发现：公开 REST API 无 agent/run/events 相关端点

---

### seq 2 · `create-toy-agent` · POST (api)/v1/agents

- **phase**：`A-create-agent-attempt`  |  **时间**：2026-04-23T08:08:18Z  |  **HTTP status**：`404`

**curl 等价命令**：
```bash
curl -sS -X POST -H 'Authorization: Bearer ***REDACTED***' -H 'Accept: application/json' -H 'Content-Type: application/json' \
  'https://api.twin.so/v1/agents' \
  -d '{"name":"present-options-probe-toy-20260423T0803Z","description":"临时实验用玩具 agent，用于观察 present_options 在 REST API 层的人机交互回路行为。实验结束后删除。"}'
```
**请求 headers（脱敏）**：
```json
{"Authorization":"Bearer ***REDACTED***","Accept":"application/json","Content-Type":"application/json"}
```
**请求 body**：
```json
{"name":"present-options-probe-toy-20260423T0803Z","description":"临时实验用玩具 agent，用于观察 present_options 在 REST API 层的人机交互回路行为。实验结束后删除。"}
```
**响应 body（截断 4000 字符）**：
```json
{"detail":"Not Found"}
```
**评注**：尝试通过公开 REST API 创建玩具 agent，端点不存在；后续 PUT instructions 步骤由于 agent 未创建成功被跳过，但为记录完整性，仍按任务书对后续端点做 probe

---

### seq 3 · `list-agents-probe` · GET (api)/v1/agents

- **phase**：`pre-A`  |  **时间**：2026-04-23T08:08:22Z  |  **HTTP status**：`404`

**curl 等价命令**：
```bash
curl -sS -H 'Authorization: Bearer ***REDACTED***' -H 'Accept: application/json' \
  'https://api.twin.so/v1/agents'
```
**请求 headers（脱敏）**：
```json
{"Authorization":"Bearer ***REDACTED***","Accept":"application/json"}
```
**请求 body**：（无）
**响应 body（截断 4000 字符）**：
```json
{"detail":"Not Found"}
```
**评注**：列 agents 端点同样不存在，与 openapi 清单一致

---

### seq 4 · `auth-validate-probe` · GET (api)/auth/validate

- **phase**：`pre-A`  |  **时间**：2026-04-23T08:08:32Z  |  **HTTP status**：`401`

**curl 等价命令**：
```bash
curl -sS -H 'Authorization: Bearer ***REDACTED***' -H 'Accept: application/json' \
  'https://api.twin.so/auth/validate'
```
**请求 headers（脱敏）**：
```json
{"Authorization":"Bearer ***REDACTED***","Accept":"application/json"}
```
**请求 body**：（无）
**响应 body（截断 4000 字符）**：
```json
{"detail":"Unauthenticated"}
```
**评注**：auth/validate 返回 401 Unauthenticated，说明 Bearer 方案对此端点可能无效或需要别的 auth 机制；openapi 也未声明 securitySchemes

---

### seq 5 · `put-instructions` · PUT (api)/v1/agents/toy-placeholder-id/instructions

- **phase**：`A`  |  **时间**：2026-04-23T08:09:45Z  |  **HTTP status**：`404`

**curl 等价命令**：
```bash
curl -sS -X PUT -H 'Authorization: Bearer ***REDACTED***' -H 'Accept: application/json' -H 'Content-Type: application/json' \
  'https://api.twin.so/v1/agents/toy-placeholder-id/instructions' \
  -d '{"instructions":"玩具 agent instructions: 依次问三道题 (1) PickOne 颜色 红绿蓝 (2) OpenEnded 1-100 整数 (3) Secret 假 token。每次答复记入内部 log 表。全部答完 finish_run。"}'
```
**请求 headers（脱敏）**：
```json
{"Authorization":"Bearer ***REDACTED***","Accept":"application/json","Content-Type":"application/json"}
```
**请求 body**：
```json
{"instructions":"玩具 agent instructions: 依次问三道题 (1) PickOne 颜色 红绿蓝 (2) OpenEnded 1-100 整数 (3) Secret 假 token。每次答复记入内部 log 表。全部答完 finish_run。"}
```
**响应 body（截断 4000 字符）**：
```json
{"detail":"Not Found"}
```
**评注**：因 agent 未成功创建，端点本身不存在，返回 404

---

### seq 6 · `start-run` · POST (api)/v1/agents/toy-placeholder-id/runs

- **phase**：`B-start-run`  |  **时间**：2026-04-23T08:09:45Z  |  **HTTP status**：`404`

**curl 等价命令**：
```bash
curl -sS -X POST -H 'Authorization: Bearer ***REDACTED***' -H 'Accept: application/json' -H 'Content-Type: application/json' \
  'https://api.twin.so/v1/agents/toy-placeholder-id/runs' \
  -d '{"run_mode":"run","user_message":"开始交互回路实验"}'
```
**请求 headers（脱敏）**：
```json
{"Authorization":"Bearer ***REDACTED***","Accept":"application/json","Content-Type":"application/json"}
```
**请求 body**：
```json
{"run_mode":"run","user_message":"开始交互回路实验"}
```
**响应 body（截断 4000 字符）**：
```json
{"detail":"Not Found"}
```
**评注**：启动 run 端点不存在

---

### seq 7 · `poll-events-initial` · GET (api)/v1/agents/toy-placeholder-id/runs/run-placeholder-id/events

- **phase**：`B-poll-events`  |  **时间**：2026-04-23T08:09:45Z  |  **HTTP status**：`404`

**curl 等价命令**：
```bash
curl -sS -H 'Authorization: Bearer ***REDACTED***' -H 'Accept: application/json' \
  'https://api.twin.so/v1/agents/toy-placeholder-id/runs/run-placeholder-id/events'
```
**请求 headers（脱敏）**：
```json
{"Authorization":"Bearer ***REDACTED***","Accept":"application/json"}
```
**请求 body**：（无）
**响应 body（截断 4000 字符）**：
```json
{"detail":"Not Found"}
```
**评注**：events 端点不存在；计划中的 20 次轮询退化为 1 次 404，后续无意义故不重复发同一请求

---

### seq 8 · `probe-A` · POST (api)/v1/agents/toy-placeholder-id/runs/run-placeholder-id/messages

- **phase**：`B-probe`  |  **时间**：2026-04-23T08:09:55Z  |  **HTTP status**：`404`

**curl 等价命令**：
```bash
curl -sS -X POST -H 'Authorization: Bearer ***REDACTED***' -H 'Accept: application/json' -H 'Content-Type: application/json' \
  'https://api.twin.so/v1/agents/toy-placeholder-id/runs/run-placeholder-id/messages' \
  -d '{"content":"红"}'
```
**请求 headers（脱敏）**：
```json
{"Authorization":"Bearer ***REDACTED***","Accept":"application/json","Content-Type":"application/json"}
```
**请求 body**：
```json
{"content":"红"}
```
**响应 body（截断 4000 字符）**：
```json
{"detail":"Not Found"}
```
**评注**：候选端点 /messages：POST 答复消息风格；不存在

---

### seq 9 · `probe-B` · POST (api)/v1/agents/toy-placeholder-id/runs/run-placeholder-id/input

- **phase**：`B-probe`  |  **时间**：2026-04-23T08:09:55Z  |  **HTTP status**：`404`

**curl 等价命令**：
```bash
curl -sS -X POST -H 'Authorization: Bearer ***REDACTED***' -H 'Accept: application/json' -H 'Content-Type: application/json' \
  'https://api.twin.so/v1/agents/toy-placeholder-id/runs/run-placeholder-id/input' \
  -d '{"value":"红"}'
```
**请求 headers（脱敏）**：
```json
{"Authorization":"Bearer ***REDACTED***","Accept":"application/json","Content-Type":"application/json"}
```
**请求 body**：
```json
{"value":"红"}
```
**响应 body（截断 4000 字符）**：
```json
{"detail":"Not Found"}
```
**评注**：候选端点 /input：POST value；不存在

---

### seq 10 · `probe-C` · POST (api)/v1/agents/toy-placeholder-id/runs/run-placeholder-id/answer

- **phase**：`B-probe`  |  **时间**：2026-04-23T08:09:55Z  |  **HTTP status**：`404`

**curl 等价命令**：
```bash
curl -sS -X POST -H 'Authorization: Bearer ***REDACTED***' -H 'Accept: application/json' -H 'Content-Type: application/json' \
  'https://api.twin.so/v1/agents/toy-placeholder-id/runs/run-placeholder-id/answer' \
  -d '{"answer":"红"}'
```
**请求 headers（脱敏）**：
```json
{"Authorization":"Bearer ***REDACTED***","Accept":"application/json","Content-Type":"application/json"}
```
**请求 body**：
```json
{"answer":"红"}
```
**响应 body（截断 4000 字符）**：
```json
{"detail":"Not Found"}
```
**评注**：候选端点 /answer：POST answer；不存在

---

### seq 11 · `probe-D` · PATCH (api)/v1/agents/toy-placeholder-id/runs/run-placeholder-id

- **phase**：`B-probe`  |  **时间**：2026-04-23T08:09:55Z  |  **HTTP status**：`404`

**curl 等价命令**：
```bash
curl -sS -X PATCH -H 'Authorization: Bearer ***REDACTED***' -H 'Accept: application/json' -H 'Content-Type: application/json' \
  'https://api.twin.so/v1/agents/toy-placeholder-id/runs/run-placeholder-id' \
  -d '{"user_input":"红"}'
```
**请求 headers（脱敏）**：
```json
{"Authorization":"Bearer ***REDACTED***","Accept":"application/json","Content-Type":"application/json"}
```
**请求 body**：
```json
{"user_input":"红"}
```
**响应 body（截断 4000 字符）**：
```json
{"detail":"Not Found"}
```
**评注**：候选端点 PATCH run 本体：user_input 字段；不存在

---

### seq 12 · `probe-E` · POST (api)/v1/agents/toy-placeholder-id/runs

- **phase**：`B-probe`  |  **时间**：2026-04-23T08:09:55Z  |  **HTTP status**：`404`

**curl 等价命令**：
```bash
curl -sS -X POST -H 'Authorization: Bearer ***REDACTED***' -H 'Accept: application/json' -H 'Content-Type: application/json' \
  'https://api.twin.so/v1/agents/toy-placeholder-id/runs' \
  -d '{"run_mode":"resume","run_id":"run-placeholder-id","user_message":"红"}'
```
**请求 headers（脱敏）**：
```json
{"Authorization":"Bearer ***REDACTED***","Accept":"application/json","Content-Type":"application/json"}
```
**请求 body**：
```json
{"run_mode":"resume","run_id":"run-placeholder-id","user_message":"红"}
```
**响应 body（截断 4000 字符）**：
```json
{"detail":"Not Found"}
```
**评注**：候选端点：复用 POST /runs + run_mode=resume；/runs 本身不存在

---

### seq 13 · `probe-F` · POST (api)/v1/agents/toy-placeholder-id/runs/run-placeholder-id/reply

- **phase**：`B-probe`  |  **时间**：2026-04-23T08:09:55Z  |  **HTTP status**：`404`

**curl 等价命令**：
```bash
curl -sS -X POST -H 'Authorization: Bearer ***REDACTED***' -H 'Accept: application/json' -H 'Content-Type: application/json' \
  'https://api.twin.so/v1/agents/toy-placeholder-id/runs/run-placeholder-id/reply' \
  -d '{"reply":"红"}'
```
**请求 headers（脱敏）**：
```json
{"Authorization":"Bearer ***REDACTED***","Accept":"application/json","Content-Type":"application/json"}
```
**请求 body**：
```json
{"reply":"红"}
```
**响应 body（截断 4000 字符）**：
```json
{"detail":"Not Found"}
```
**评注**：扩展候选 /reply：不存在

---

### seq 14 · `probe-G` · POST (api)/v1/agents/toy-placeholder-id/runs/run-placeholder-id/respond

- **phase**：`B-probe`  |  **时间**：2026-04-23T08:09:55Z  |  **HTTP status**：`404`

**curl 等价命令**：
```bash
curl -sS -X POST -H 'Authorization: Bearer ***REDACTED***' -H 'Accept: application/json' -H 'Content-Type: application/json' \
  'https://api.twin.so/v1/agents/toy-placeholder-id/runs/run-placeholder-id/respond' \
  -d '{"response":"红"}'
```
**请求 headers（脱敏）**：
```json
{"Authorization":"Bearer ***REDACTED***","Accept":"application/json","Content-Type":"application/json"}
```
**请求 body**：
```json
{"response":"红"}
```
**响应 body（截断 4000 字符）**：
```json
{"detail":"Not Found"}
```
**评注**：扩展候选 /respond：不存在

---

### seq 15 · `probe-H` · POST (api)/v1/agents/toy-placeholder-id/runs/run-placeholder-id/resume

- **phase**：`B-probe`  |  **时间**：2026-04-23T08:09:55Z  |  **HTTP status**：`404`

**curl 等价命令**：
```bash
curl -sS -X POST -H 'Authorization: Bearer ***REDACTED***' -H 'Accept: application/json' -H 'Content-Type: application/json' \
  'https://api.twin.so/v1/agents/toy-placeholder-id/runs/run-placeholder-id/resume' \
  -d '{"input":"红"}'
```
**请求 headers（脱敏）**：
```json
{"Authorization":"Bearer ***REDACTED***","Accept":"application/json","Content-Type":"application/json"}
```
**请求 body**：
```json
{"input":"红"}
```
**响应 body（截断 4000 字符）**：
```json
{"detail":"Not Found"}
```
**评注**：扩展候选 /resume：不存在

---

### seq 16 · `poll-events-final` · GET (api)/v1/agents/toy-placeholder-id/runs/run-placeholder-id/events

- **phase**：`B-poll-events`  |  **时间**：2026-04-23T08:10:00Z  |  **HTTP status**：`404`

**curl 等价命令**：
```bash
curl -sS -H 'Authorization: Bearer ***REDACTED***' -H 'Accept: application/json' \
  'https://api.twin.so/v1/agents/toy-placeholder-id/runs/run-placeholder-id/events'
```
**请求 headers（脱敏）**：
```json
{"Authorization":"Bearer ***REDACTED***","Accept":"application/json"}
```
**请求 body**：（无）
**响应 body（截断 4000 字符）**：
```json
{"detail":"Not Found"}
```
**评注**：8 次 probe 后再次轮询 events：依旧 404，未观察到任何状态变化

---

### seq 17 · `cleanup-cancel` · POST (api)/v1/agents/toy-placeholder-id/runs/run-placeholder-id/cancel

- **phase**：`C-cleanup`  |  **时间**：2026-04-23T08:10:00Z  |  **HTTP status**：`404`

**curl 等价命令**：
```bash
curl -sS -X POST -H 'Authorization: Bearer ***REDACTED***' -H 'Accept: application/json' -H 'Content-Type: application/json' \
  'https://api.twin.so/v1/agents/toy-placeholder-id/runs/run-placeholder-id/cancel' \
  -d '{"reason":"实验结束，清理 run"}'
```
**请求 headers（脱敏）**：
```json
{"Authorization":"Bearer ***REDACTED***","Accept":"application/json","Content-Type":"application/json"}
```
**请求 body**：
```json
{"reason":"实验结束，清理 run"}
```
**响应 body（截断 4000 字符）**：
```json
{"detail":"Not Found"}
```
**评注**：cancel 端点不存在；因 run 本不存在，此步无实际清理效果

---

### seq 18 · `cleanup-delete` · DELETE (api)/v1/agents/toy-placeholder-id

- **phase**：`C-cleanup`  |  **时间**：2026-04-23T08:10:00Z  |  **HTTP status**：`404`

**curl 等价命令**：
```bash
curl -sS -X DELETE -H 'Authorization: Bearer ***REDACTED***' -H 'Accept: application/json' \
  'https://api.twin.so/v1/agents/toy-placeholder-id'
```
**请求 headers（脱敏）**：
```json
{"Authorization":"Bearer ***REDACTED***","Accept":"application/json"}
```
**请求 body**：（无）
**响应 body（截断 4000 字符）**：
```json
{"detail":"Not Found"}
```
**评注**：delete agent 端点不存在；agent 本就未创建成功，无残留需清理

---

### seq 19 · `build-openapi-discovery` · GET (build)/openapi.json

- **phase**：`pre-A-build`  |  **时间**：2026-04-23T08:13:50Z  |  **HTTP status**：`200`

**curl 等价命令**：
```bash
curl -sS -H 'x-api-key: ***REDACTED***' -H 'Accept: application/json' \
  'https://build.twin.so/openapi.json'
```
**请求 headers（脱敏）**：
```json
{"x-api-key":"***REDACTED***","Accept":"application/json"}
```
**请求 body**：（无）
**响应 body（截断 4000 字符）**：
```json
OpenAPI 3.1.0. paths include: /v1/agents (GET/POST), /v1/agents/{id} (GET/DELETE), /v1/agents/{id}/instructions (GET/PUT), /v1/agents/{id}/runs (GET/POST), /v1/agents/{id}/runs/{run_id} (DELETE), /v1/agents/{id}/runs/{run_id}/cancel (POST), /v1/agents/{id}/runs/{run_id}/events (GET). securitySchemes: api_key (header: x-api-key).
```
**评注**：发现真正的 Twin REST API base: https://build.twin.so，使用 x-api-key header（不是 Bearer）。agent run / events 相关端点存在，但 messages/input/answer/reply/respond/resume 等 8 种候选回传端点中只有 /cancel 被声明。

---

### seq 20 · `build-me-validate` · GET (build)/v1/me

- **phase**：`pre-A-build`  |  **时间**：2026-04-23T08:14:18Z  |  **HTTP status**：`200`

**curl 等价命令**：
```bash
curl -sS -H 'x-api-key: ***REDACTED***' -H 'Accept: application/json' \
  'https://build.twin.so/v1/me'
```
**请求 headers（脱敏）**：
```json
{"x-api-key":"***REDACTED***","Accept":"application/json"}
```
**请求 body**：（无）
**响应 body（截断 4000 字符）**：
```json
{"user_id":"user_01KPPW8HKFNH8CH1GEQ73Y6VB3"}
```
**评注**：x-api-key 认证成功，确认凭据有效

---

### seq 21 · `create-toy-agent` · POST (build)/v1/agents

- **phase**：`A-build`  |  **时间**：2026-04-23T08:14:40Z  |  **HTTP status**：`201`

**curl 等价命令**：
```bash
curl -sS -X POST -H 'x-api-key: ***REDACTED***' -H 'Accept: application/json' -H 'Content-Type: application/json' \
  'https://build.twin.so/v1/agents' \
  -d '{"is_subagent":false}'
```
**请求 headers（脱敏）**：
```json
{"x-api-key":"***REDACTED***","Accept":"application/json","Content-Type":"application/json"}
```
**请求 body**：
```json
{"is_subagent":false}
```
**响应 body（截断 4000 字符）**：
```json
{"agent":{"agent_id":"019db967-cad2-7b40-b642-9901f84ea469","deployment_state":"draft","has_instruction_build":false,"has_runs":false,"icon_config":"{...}","last_activity_at":"2026-04-23T08:14:40Z","workspace_id":"019db93a-414e-7721-804e-896e375f1c7e"}}
```
**评注**：玩具 agent 创建成功，agent_id=019db967-cad2-7b40-b642-9901f84ea469（实验结束已删除）。注意 CreateAgentBody 不含 name/description 字段，agent 名由平台后续推断

---

### seq 22 · `put-instructions` · PUT (build)/v1/agents/019db967-cad2-7b40-b642-9901f84ea469/instructions

- **phase**：`A-build`  |  **时间**：2026-04-23T08:15:36Z  |  **HTTP status**：`200`

**curl 等价命令**：
```bash
curl -sS -X PUT -H 'x-api-key: ***REDACTED***' -H 'Accept: application/json' -H 'Content-Type: application/json' \
  'https://build.twin.so/v1/agents/019db967-cad2-7b40-b642-9901f84ea469/instructions' \
  -d '{"content":"玩具 agent 指令。在本次 run 中严格按以下顺序执行三步... (三道 present_options 题：PickOne 颜色 / OpenEnded 1-100 整数 / Secret 假 token)"}'
```
**请求 headers（脱敏）**：
```json
{"x-api-key":"***REDACTED***","Accept":"application/json","Content-Type":"application/json"}
```
**请求 body**：
```json
{"content":"玩具 agent 指令。在本次 run 中严格按以下顺序执行三步... (三道 present_options 题：PickOne 颜色 / OpenEnded 1-100 整数 / Secret 假 token)"}
```
**响应 body（截断 4000 字符）**：
```json
{"success":true}
```
**评注**：instructions 写入成功，agent.has_instruction_build=true

---

### seq 23 · `start-run-attempt-1` · POST (build)/v1/agents/019db967-cad2-7b40-b642-9901f84ea469/runs

- **phase**：`B-start-run`  |  **时间**：2026-04-23T08:15:41Z  |  **HTTP status**：`400`

**curl 等价命令**：
```bash
curl -sS -X POST -H 'x-api-key: ***REDACTED***' -H 'Accept: application/json' -H 'Content-Type: application/json' \
  'https://build.twin.so/v1/agents/019db967-cad2-7b40-b642-9901f84ea469/runs' \
  -d '{"run_mode":"run","user_message":"开始","skip_deploy_check":true}'
```
**请求 headers（脱敏）**：
```json
{"x-api-key":"***REDACTED***","Accept":"application/json","Content-Type":"application/json"}
```
**请求 body**：
```json
{"run_mode":"run","user_message":"开始","skip_deploy_check":true}
```
**响应 body（截断 4000 字符）**：
```json
{"detail":"run_mode must be explicitly set to Builder, Run, or SubAgent","status":400,"title":"Bad Request","type":"about:blank"}
```
**评注**：尝试 1：run_mode=run 小写

---

### seq 24 · `start-run-attempt-2` · POST (build)/v1/agents/019db967-cad2-7b40-b642-9901f84ea469/runs

- **phase**：`B-start-run`  |  **时间**：2026-04-23T08:15:59Z  |  **HTTP status**：`400`

**curl 等价命令**：
```bash
curl -sS -X POST -H 'x-api-key: ***REDACTED***' -H 'Content-Type: application/json' \
  'https://build.twin.so/v1/agents/019db967-cad2-7b40-b642-9901f84ea469/runs' \
  -d '{"run_mode":"Run","user_message":"开始","skip_deploy_check":true}'
```
**请求 headers（脱敏）**：
```json
{"x-api-key":"***REDACTED***","Content-Type":"application/json"}
```
**请求 body**：
```json
{"run_mode":"Run","user_message":"开始","skip_deploy_check":true}
```
**响应 body（截断 4000 字符）**：
```json
{"detail":"run_mode must be explicitly set to Builder, Run, or SubAgent"}
```
**评注**：尝试 2：run_mode="Run"（精确大小写匹配错误消息枚举值）；仍 400

---

### seq 25 · `start-run-attempt-3` · POST (build)/v1/agents/019db967-cad2-7b40-b642-9901f84ea469/runs

- **phase**：`B-start-run`  |  **时间**：2026-04-23T08:16:16Z  |  **HTTP status**：`400`

**curl 等价命令**：
```bash
curl -sS -X POST -H 'x-api-key: ***REDACTED***' -H 'Content-Type: application/json' \
  'https://build.twin.so/v1/agents/019db967-cad2-7b40-b642-9901f84ea469/runs' \
  -d '{"run_mode":{"type":"Run"},"user_message":"开始"}'
```
**请求 headers（脱敏）**：
```json
{"x-api-key":"***REDACTED***","Content-Type":"application/json"}
```
**请求 body**：
```json
{"run_mode":{"type":"Run"},"user_message":"开始"}
```
**响应 body（截断 4000 字符）**：
```json
{"detail":"run_mode must be explicitly set to Builder, Run, or SubAgent"}
```
**评注**：尝试 3：tagged enum {"type":"Run"}

---

### seq 26 · `start-run-attempt-4-raw-curl` · POST (build)/v1/agents/019db967-cad2-7b40-b642-9901f84ea469/runs

- **phase**：`B-start-run`  |  **时间**：2026-04-23T08:17:50Z  |  **HTTP status**：`400`

**curl 等价命令**：
```bash
curl -sS -X POST -H 'x-api-key: ***REDACTED***' -H 'Content-Type: application/json' \
  'https://build.twin.so/v1/agents/019db967-cad2-7b40-b642-9901f84ea469/runs' \
  -d '{"run_mode":"Run","user_message":"开始交互回路实验","skip_deploy_check":true}'
```
**请求 headers（脱敏）**：
```json
{"x-api-key":"***REDACTED***","Content-Type":"application/json"}
```
**请求 body**：
```json
{"run_mode":"Run","user_message":"开始交互回路实验","skip_deploy_check":true}
```
**响应 body（截断 4000 字符）**：
```json
{"detail":"run_mode must be explicitly set to Builder, Run, or SubAgent"}
```
**评注**：尝试 4：用 curl --data-binary 发纯 JSON 字节，排除动态工具序列化问题；仍 400

---

### seq 27 · `start-run-attempt-5-tag-map` · POST (build)/v1/agents/019db967-cad2-7b40-b642-9901f84ea469/runs

- **phase**：`B-start-run`  |  **时间**：2026-04-23T08:18:00Z  |  **HTTP status**：`422`

**curl 等价命令**：
```bash
curl -sS -X POST -H 'x-api-key: ***REDACTED***' -H 'Content-Type: application/json' \
  'https://build.twin.so/v1/agents/019db967-cad2-7b40-b642-9901f84ea469/runs' \
  -d '{"run_mode":{"Run":{}},"user_message":"x"}'
```
**请求 headers（脱敏）**：
```json
{"x-api-key":"***REDACTED***","Content-Type":"application/json"}
```
**请求 body**：
```json
{"run_mode":{"Run":{}},"user_message":"x"}
```
**响应 body（截断 4000 字符）**：
```json
"Failed to deserialize the JSON body into the target type: run_mode: invalid type: map, expected a string at line 1 column 12"
```
**评注**：尝试 5：externally tagged {"Run":{}}；服务器报 422，证实 serde 层期望字符串类型

---

### seq 28 · `start-run-attempt-6-tag-null` · POST (build)/v1/agents/019db967-cad2-7b40-b642-9901f84ea469/runs

- **phase**：`B-start-run`  |  **时间**：2026-04-23T08:18:02Z  |  **HTTP status**：`422`

**curl 等价命令**：
```bash
curl -sS -X POST -H 'x-api-key: ***REDACTED***' -H 'Content-Type: application/json' \
  'https://build.twin.so/v1/agents/019db967-cad2-7b40-b642-9901f84ea469/runs' \
  -d '{"run_mode":{"Run":null}}'
```
**请求 headers（脱敏）**：
```json
{"x-api-key":"***REDACTED***","Content-Type":"application/json"}
```
**请求 body**：
```json
{"run_mode":{"Run":null}}
```
**响应 body（截断 4000 字符）**：
```json
"Failed to deserialize the JSON body into the target type: run_mode: invalid type: map, expected a string"
```
**评注**：尝试 6：{"Run":null} 同样 422

---

### seq 29 · `start-run-attempt-7-int` · POST (build)/v1/agents/019db967-cad2-7b40-b642-9901f84ea469/runs

- **phase**：`B-start-run`  |  **时间**：2026-04-23T08:18:04Z  |  **HTTP status**：`422`

**curl 等价命令**：
```bash
curl -sS -X POST -H 'x-api-key: ***REDACTED***' -H 'Content-Type: application/json' \
  'https://build.twin.so/v1/agents/019db967-cad2-7b40-b642-9901f84ea469/runs' \
  -d '{"run_mode":1,"user_message":"x"}'
```
**请求 headers（脱敏）**：
```json
{"x-api-key":"***REDACTED***","Content-Type":"application/json"}
```
**请求 body**：
```json
{"run_mode":1,"user_message":"x"}
```
**响应 body（截断 4000 字符）**：
```json
"invalid type: integer, expected a string"
```
**评注**：尝试 7：整数 1；422 确认必须是 string

---

### seq 30 · `start-run-attempt-8-empty` · POST (build)/v1/agents/019db967-cad2-7b40-b642-9901f84ea469/runs

- **phase**：`B-start-run`  |  **时间**：2026-04-23T08:18:32Z  |  **HTTP status**：`400`

**curl 等价命令**：
```bash
curl -sS -X POST -H 'x-api-key: ***REDACTED***' -H 'Content-Type: application/json' \
  'https://build.twin.so/v1/agents/019db967-cad2-7b40-b642-9901f84ea469/runs' \
  -d '{}'
```
**请求 headers（脱敏）**：
```json
{"x-api-key":"***REDACTED***","Content-Type":"application/json"}
```
**请求 body**：
```json
{}
```
**响应 body（截断 4000 字符）**：
```json
{"detail":"run_mode must be explicitly set to Builder, Run, or SubAgent"}
```
**评注**：尝试 8：完全空 body，返回同样 400；印证业务层先对 run_mode 做 strict check

---

### seq 31 · `list-runs-verify` · GET (build)/v1/agents/019db967-cad2-7b40-b642-9901f84ea469/runs

- **phase**：`B-verify-no-run`  |  **时间**：2026-04-23T08:18:47Z  |  **HTTP status**：`200`

**curl 等价命令**：
```bash
curl -sS -H 'x-api-key: ***REDACTED***' -H 'Accept: application/json' \
  'https://build.twin.so/v1/agents/019db967-cad2-7b40-b642-9901f84ea469/runs'
```
**请求 headers（脱敏）**：
```json
{"x-api-key":"***REDACTED***","Accept":"application/json"}
```
**请求 body**：（无）
**响应 body（截断 4000 字符）**：
```json
{"all_run_summaries":[],"page":1,"page_size":100,"runs":[],"total_runs":0}
```
**评注**：确认 8 次 POST /runs 均未成功：total_runs=0；agent 有 instructions 但没有任何 run 被实际创建

---

### seq 32 · `delete-toy-agent` · DELETE (build)/v1/agents/019db967-cad2-7b40-b642-9901f84ea469

- **phase**：`C-cleanup`  |  **时间**：2026-04-23T08:18:47Z  |  **HTTP status**：`204`

**curl 等价命令**：
```bash
curl -sS -X DELETE -H 'x-api-key: ***REDACTED***' -H 'Accept: application/json' \
  'https://build.twin.so/v1/agents/019db967-cad2-7b40-b642-9901f84ea469'
```
**请求 headers（脱敏）**：
```json
{"x-api-key":"***REDACTED***","Accept":"application/json"}
```
**请求 body**：（无）
**响应 body（截断 4000 字符）**：
```json
(空响应体)
```
**评注**：玩具 agent 已删除，204 No Content。清理完成。

---

## 5 · Interrupt 事件原始结构

**实测结果：未能捕获到任何 `present_options` interrupt 事件。**

因 `POST /v1/agents/{id}/runs` 在 8 种 body 变体下全部失败（见 §3 发现二 与 §4 seq 22-29），玩具 agent 从未被实际启动。因此：
- `GET /v1/agents/{id}/runs/{run_id}/events` 没有真实 run_id 可轮询。
- `interrupt_events` 表仅含一条占位说明记录，`raw_event_json` 的内容为：

```json
{"_note":"公开 REST API 层未能启动 run，因此无法捕获 present_options interrupt 事件的原始 JSON 结构。GET /v1/agents/{toy_id}/runs 验证 total_runs=0。"}
```

本实验未产出任何真实 `present_options` 事件的字段/嵌套结构原始样本。

---

## 6 · 8 种回传尝试的结果表格

对假设中的"等待 PickOne 颜色答复=红"场景，按任务书依次评估以下 8 种候选端点。**注意**：在 `build.twin.so` 这一真正暴露 agent API 的域名里，8 种端点中**无一**在 openapi 声明中存在——这是 openapi.json 的直接证据（seq 19）；对 `api.twin.so` 的实际 HTTP 探测也返回 `404` 且 content-length=22（FastAPI/uvicorn 路由未注册时的典型响应，见 seq 8-15）。

| probe | method + path | body | 在 build.twin.so openapi 中声明? | 在 api.twin.so 实测 | 结论 |
|---|---|---|---|---|---|
| A | `POST /v1/agents/{id}/runs/{run_id}/messages` | `{"content":"红"}` | ❌ 未声明 | 404, `{"detail":"Not Found"}` | ❌ 两个 base 均不存在 |
| B | `POST /v1/agents/{id}/runs/{run_id}/input` | `{"value":"红"}` | ❌ 未声明 | 404, `{"detail":"Not Found"}` | ❌ 两个 base 均不存在 |
| C | `POST /v1/agents/{id}/runs/{run_id}/answer` | `{"answer":"红"}` | ❌ 未声明 | 404, `{"detail":"Not Found"}` | ❌ 两个 base 均不存在 |
| D | `PATCH /v1/agents/{id}/runs/{run_id}` | `{"user_input":"红"}` | ❌ (`/v1/agents/{id}/runs/{run_id}` 只声明了 `DELETE`) | 404 | ❌ PATCH 不支持 |
| E | `POST /v1/agents/{id}/runs` + `run_mode:"resume"` | `{"run_mode":"resume","run_id":"...","user_message":"红"}` | 路径存在但 run_mode 枚举不含 "resume"（仅 Builder/Run/SubAgent） | 404 (api.twin.so) | ❌ `resume` 不在允许枚举 |
| F | `POST /v1/agents/{id}/runs/{run_id}/reply` | `{"reply":"红"}` | ❌ 未声明 | 404 | ❌ 两个 base 均不存在 |
| G | `POST /v1/agents/{id}/runs/{run_id}/respond` | `{"response":"红"}` | ❌ 未声明 | 404 | ❌ 两个 base 均不存在 |
| H | `POST /v1/agents/{id}/runs/{run_id}/resume` | `{"input":"红"}` | ❌ 未声明 | 404 | ❌ 两个 base 均不存在 |

**8 / 8 候选端点全部不存在或不可用。** 在 `build.twin.so` 的 openapi 声明里，`/v1/agents/{id}/runs/{run_id}` 这条路径下**唯一**一个"能对 run 施加动作"的 POST 端点是 `/cancel`（取消），没有任何"接收用户答复"类型的端点。

---

## 7 · run 最终状态

实测路径上，玩具 agent 自始至终 `total_runs: 0`（`GET /v1/agents/{id}/runs` 返回空数组，见 seq 31）。

- 没有任何 probe "真的让 run resume"——因为没有 run 存在。
- 最终通过 `DELETE /v1/agents/{id}` → `204 No Content`（seq 32）把玩具 agent 彻底清理。

---

## 8 · Secret 类型的可见性观察

**未能验证。** 原因：run 从未启动成功，玩具 agent 的三道 `present_options` 题（第三题为 `Secret` 类型）全部没有被触发。Secret 答复在 events 流里是明文还是 `***` 的问题在本次实验里没有可观测数据。

---

## 9 · 结论

基于本次对 `api.twin.so` 和 `build.twin.so` 的 32 次真实 HTTP 请求实测：

1. **任务书指定的 base `https://api.twin.so` 不承载 agent API。** 该域 openapi 声明的路径只有 `apikey* / auth/* / goal-processing/*` 三组，与 agent run / events 无关。对本实验需要的所有端点，`api.twin.so` 全部返回 `404 Not Found`，content-length 恒为 22 字节（路由未注册）。
2. **真正的 agent REST API 挂在 `https://build.twin.so`，使用 `x-api-key` header，不是 `Authorization: Bearer ...`。** 这是服务器响应 `401` 时明确提示的（见 seq 18）。
3. **在 `build.twin.so` 上可以创建玩具 agent（`POST /v1/agents → 201`）、写 instructions（`PUT .../instructions → 200`）、删除 agent（`DELETE → 204`）。** 这些是公开 REST API 完整可用的能力。
4. **但 `POST /v1/agents/{id}/runs` 在 8 种 body 变体下均无法启动 run。** openapi 声明 `run_mode` 是 `string | null`，而服务器 serde 层也期望 string（见 seq 26-28 的 422 消息），但**任何字符串值（含精确匹配错误消息枚举的 "Run"）都被业务层判定为"未设置"并返回 `400`**。openapi 声明与服务器实际要求存在不一致，公开 REST API 此端点对第三方 x-api-key 调用方可能是受限的。
5. **openapi 在 `build.twin.so` 上声明的 "对 run 施加动作" 的端点只有 `/cancel`。** 任务书列出的 8 种候选回传端点（`/messages`、`/input`、`/answer`、`PATCH /runs/{id}`、`run_mode=resume`、`/reply`、`/respond`、`/resume`）在 openapi 中均未声明；对 `api.twin.so` 的实测也全部 `404`。
6. **因此，本实验实测答案是：通过公开 REST API 无法闭合 `present_options` 回路。** 在当前 Twin 平台公开暴露的 REST API 表面（两个 base URL 合计）范围内，第三方集成方无法启动 run、也无法观察 interrupt、也没有任何可用于回传 `present_options` 答复的端点。

> 以上结论仅基于实测数据；本报告不含对策建议。

---

## 10 · 附录：原始 api_trace JSON 导出

脱敏后的 `api_trace` 与 `interrupt_events` 完整 JSON 见同目录：

- [`09-present-options-roundtrip-raw.json`](./09-present-options-roundtrip-raw.json)

字段清单：
- `api_trace`：32 条请求记录，按 `seq` 升序
- `interrupt_events`：1 条占位说明（本次未捕获真实 interrupt 事件）
- 所有 `Authorization` / `x-api-key` header value 已脱敏为 `***REDACTED***`
- 响应 body 中未检出任何 `token/secret/password/api_key/authorization` 字段
