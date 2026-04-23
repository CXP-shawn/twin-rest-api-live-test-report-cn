# 09 · present_options 人机交互回路 · REST API 实测调用实录（v2）

> v2 实验执行时间：2026-04-23T09:35Z ~ 09:37Z (UTC)
> 实验中创建的 v2 玩具 agent ID：`019db9b1-ecad-7222-992d-9e0a03231ac9`（实验结束已 `DELETE 204` 删除）
> Twin API key 脱敏规则：`x-api-key` header value 显示为 `***REDACTED***`

---

## 0 · v2 勘误说明

上一版（v1）报告基于错误的假设启动实验。本 v2 版基于用户提供的权威 openapi schema 重做。主要勘误：

| 维度 | v1 的做法 | 权威 schema 的正确做法 | v2 结果 |
|---|---|---|---|
| Base URL | 任务书写 `api.twin.so`，实测发现真 API 在 `build.twin.so` | 用户告知权威 base 应为 `builder.twin.so` | v2 实测 `builder.twin.so` 返回 SPA HTML（AmazonS3，UI 前端），**真正的 agent API 仍在 `build.twin.so`**。CORS header `access-control-allow-origin: https://builder.twin.so` 反向印证 builder=前端、build=后端 |
| 鉴权 | 一度用 `Authorization: Bearer` | `x-api-key` header | v2 全程 `x-api-key`，`build.twin.so` 正确接受 |
| `run_mode` 类型 | 试过 object、tagged enum、数字 | 字符串字面量 `"Builder"` / `"Run"` / `"SubAgent"` | v2 用 `"Run"`、`"Builder"` 各试一次，都仍 400 "must be explicitly set"（详见 §4）|

v2 的 `builder.twin.so` 实测结论并非用户预期——但这是对开发者极有价值的信息。v2 全部请求完整记录，覆盖写本文件和对应 raw JSON。

---

## 1 · 实验目的

在 Twin 平台中，`present_options` 是 agent 在 run 执行中暂停并向用户索取输入的原生机制。关键开发者问题：

1. 第三方能否通过公开 REST API 观察到 agent 进入了 interrupt 状态？
2. 能否通过 REST API 把用户答复回传给 agent，使之从 interrupt 恢复？
3. 如果题目是 `Secret` 类型，答复在 events 流里是否被 mask？

v2 版本基于用户给出的权威 base URL + 鉴权方式 + run_mode 字符串类型重做，但仍以实际响应为准。

---

## 2 · 玩具 agent 设计

| 题号 | type | prompt | options |
|---|---|---|---|
| 1 | `PickOne` | "请选一个颜色" | `["红","绿","蓝"]` |
| 2 | `OpenEnded` | "请输入一个 1 到 100 之间的整数" | `[]` |
| 3 | `Secret` | "请输入一个测试用的假 token（随便编，不要填真实密钥）" | `[]` |

v2 instructions 比 v1 更明确地限制 agent 只做这三步。实际写入路径：`PUT /v1/agents/{id}/instructions`（seq 5），响应 `200 {"success":true}`。

---

## 3 · 关键架构发现（v2 更新）

### 发现 α · `builder.twin.so` 是前端 SPA，不是 REST API

v2 首先按用户指示对 `https://builder.twin.so` 发起三次探测（见 seq 1-3）：

- `GET /v1/me` → **200 OK**, `server: AmazonS3`, `content-type: text/html`, body 是 Twin SPA index.html（含 `<title>Twin - Automate anything</title>`，GTM 脚本，根元素 `#root`）。
- `GET /openapi.json` → 同样返回 SPA HTML。
- `GET /api/v1/me` → 同样返回 SPA HTML。

这是典型 CloudFront+S3 托管 SPA 的 fallback 行为（未知路径统一返 `index.html`）。`builder.twin.so` 整个域托管的是 Web UI。**真正的 agent REST API 在 `build.twin.so`**（v1 已实测、v2 再次确认），且其 CORS header `access-control-allow-origin: https://builder.twin.so`（见 v2 seq 4-20）反向印证 builder 是 UI client、build 是 API server。

### 发现 β · `build.twin.so` 是 gRPC-over-HTTP 网关

v2 发现 8 个 probe 端点中 6 个（A/B/C/F/G/H）的响应具有以下特征：
- HTTP status: **200 OK**（非 404！）
- `content-type: application/grpc`
- `grpc-status: 12`（gRPC 标准 UNIMPLEMENTED）
- content-length: 0

参考 gRPC 规范，`grpc-status=12` 表示"方法未实现"。这说明 `build.twin.so` 是 gRPC-web/gRPC-HTTP 桥接网关：HTTP 路径被路由到某个 gRPC 服务，但该服务未实现此方法。**这不是 "端点不存在"（404 路由未注册）**——gateway 能路由到 handler，handler 只是 UNIMPLEMENTED。

对比组：`POST /runs/{rid}/cancel`（seq 18）返回 **404 with `{"detail":"run placeholder-run not found"}`**，这是 REST 业务层 404（含具体 run_id 文本），证明 /cancel 是真正实现的 REST 端点。两种响应的巨大结构差异可作为区分"已实现 REST" vs "未实现 gRPC stub"的实测信号。

### 发现 γ · `POST /v1/agents/{id}/runs` 对 x-api-key 调用方无法启动 run

v2 用 3 种 `run_mode` 字符串变体（`"Run"`、`"Builder"`、`"Run" + skip_deploy_check`）分别发 POST 请求，全部返回 400：

```json
{"detail": "run_mode must be explicitly set to Builder, Run, or SubAgent", "status": 400, "title": "Bad Request"}
```

openapi 声明 `StartRunBody.run_mode: string | null`，v1 的 422 响应（`"invalid type: map, expected a string"`，v1 trace seq 26）证实 serde 反序列化期望 string。但**即使传合法字符串 "Run" / "Builder"，业务层仍判定 run_mode 为"未设置"**。两条错误结合起来表明：openapi 声明与服务器内部反序列化规则存在分歧——内部 `RunMode` 很可能是 Rust enum，serde 以某种 alias/rename_all 模式匹配，`"Run"`/`"Builder"` 字面值没有命中。

结合 v1+v2 合计 11 次（8 次 v1 + 3 次 v2）变体尝试全部失败，可得出实测结论：**公开 REST API（通过 x-api-key 凭据）无法调用 `POST /v1/agents/{id}/runs` 启动 run**。该端点可能仅对 UI 会话（CORS 限定在 `builder.twin.so` 的浏览器）开放，或 run_mode 需要额外未公开字段配合。

---

## 4 · 完整调用实录（api_trace_v2 · 20 条）

按 `seq` 顺序列出 v2 所有真实 HTTP 请求。所有鉴权字段均已脱敏。

### seq 1 · `builder-spa-discovery` · GET (builder)/v1/me

- **phase**：`pre-A-v2`  |  **时间**：2026-04-23T09:35:20Z  |  **HTTP status**：`200`

**curl 等价命令**：
```bash
curl -sS -H 'x-api-key: ***REDACTED***' -H 'Accept: application/json' \
  'https://builder.twin.so/v1/me'
```
**请求 headers（脱敏）**：
```json
{"x-api-key":"***REDACTED***","Accept":"application/json"}
```
**请求 body**：（无）
**响应 headers（关键）**：
```json
{"server":"AmazonS3","content-type":"text/html; charset=utf-8","content-length":"3769"}
```
**响应 body（截断 5000 字符）**：
```json
<!doctype html>\n<html lang="en">... (Twin UI SPA 首页 HTML, content-length=3769)
```
**评注**：v2 关键发现：用户指定的 base `builder.twin.so` 实际是前端 SPA（CloudFront+S3），不承载 REST API；任何路径均 fallback 到 SPA index.html。真正的 agent API 在 `build.twin.so`（CORS header `access-control-allow-origin: https://builder.twin.so` 反向印证两者是 UI/API 对应关系）

---

### seq 2 · `builder-api-path-probe-1` · GET (builder)/openapi.json

- **phase**：`pre-A-v2`  |  **时间**：2026-04-23T09:35:29Z  |  **HTTP status**：`200`

**curl 等价命令**：
```bash
curl -sS -H 'x-api-key: ***REDACTED***' -H 'Accept: application/json' \
  'https://builder.twin.so/openapi.json'
```
**请求 headers（脱敏）**：
```json
{"x-api-key":"***REDACTED***","Accept":"application/json"}
```
**请求 body**：（无）
**响应 headers（关键）**：
```json
{"server":"AmazonS3","content-type":"text/html"}
```
**响应 body（截断 5000 字符）**：
```json
<!doctype html>... (相同 SPA HTML)
```
**评注**：探测 openapi.json 路径，仍返回 SPA HTML 而非 JSON

---

### seq 3 · `builder-api-path-probe-2` · GET (builder)/api/v1/me

- **phase**：`pre-A-v2`  |  **时间**：2026-04-23T09:35:29Z  |  **HTTP status**：`200`

**curl 等价命令**：
```bash
curl -sS -H 'x-api-key: ***REDACTED***' -H 'Accept: application/json' \
  'https://builder.twin.so/api/v1/me'
```
**请求 headers（脱敏）**：
```json
{"x-api-key":"***REDACTED***","Accept":"application/json"}
```
**请求 body**：（无）
**响应 headers（关键）**：
```json
{"server":"AmazonS3","content-type":"text/html"}
```
**响应 body（截断 5000 字符）**：
```json
<!doctype html>... (相同 SPA HTML)
```
**评注**：探测 /api/v1 前缀路径，仍 SPA HTML。确认 builder.twin.so 整个域都是前端 SPA

---

### seq 4 · `create-toy-agent-v2` · POST (build)/v1/agents

- **phase**：`A-v2`  |  **时间**：2026-04-23T09:35:38Z  |  **HTTP status**：`201`

**curl 等价命令**：
```bash
curl -sS -X POST -H 'x-api-key: ***REDACTED***' -H 'Content-Type: application/json' -H 'Accept: application/json' \
  'https://build.twin.so/v1/agents' \
  -d '{"is_subagent":false}'
```
**请求 headers（脱敏）**：
```json
{"x-api-key":"***REDACTED***","Content-Type":"application/json","Accept":"application/json"}
```
**请求 body**：
```json
{"is_subagent":false}
```
**响应 headers（关键）**：
```json
{"content-type":"application/json","access-control-allow-origin":"https://builder.twin.so"}
```
**响应 body（截断 5000 字符）**：
```json
{"agent":{"agent_id":"019db9b1-ecad-7222-992d-9e0a03231ac9","deployment_state":"draft","has_instruction_build":false,"has_runs":false,"icon_config":"{...}","last_activity_at":"2026-04-23T09:35:38Z","workspace_id":"019db93a-414e-7721-804e-896e375f1c7e"}}
```
**评注**：v2 toy agent 创建成功，agent_id=019db9b1-ecad-7222-992d-9e0a03231ac9（实验结束已 DELETE 204 删除）

---

### seq 5 · `put-instructions-v2` · PUT (build)/v1/agents/019db9b1-ecad-7222-992d-9e0a03231ac9/instructions

- **phase**：`A-v2`  |  **时间**：2026-04-23T09:35:55Z  |  **HTTP status**：`200`

**curl 等价命令**：
```bash
curl -sS -X PUT -H 'x-api-key: ***REDACTED***' -H 'Content-Type: application/json' \
  'https://build.twin.so/v1/agents/019db9b1-ecad-7222-992d-9e0a03231ac9/instructions' \
  -d '{"content":"# Purpose\n玩具 agent 指令：依次 present_options 三道题 (PickOne 颜色 / OpenEnded 1-100 / Secret 假 token)..."}'
```
**请求 headers（脱敏）**：
```json
{"x-api-key":"***REDACTED***","Content-Type":"application/json"}
```
**请求 body**：
```json
{"content":"# Purpose\n玩具 agent 指令：依次 present_options 三道题 (PickOne 颜色 / OpenEnded 1-100 / Secret 假 token)..."}
```
**响应 headers（关键）**：
```json
{"content-type":"application/json"}
```
**响应 body（截断 5000 字符）**：
```json
{"success":true}
```
**评注**：instructions 写入成功。对比 v1，v2 instructions 更明确地要求 agent "只做三步、不做其它"

---

### seq 6 · `start-run-attempt-1-via-dynamic-tool` · POST (build)/v1/agents/019db9b1-ecad-7222-992d-9e0a03231ac9/runs

- **phase**：`B-v2-start-run`  |  **时间**：2026-04-23T09:36:00Z  |  **HTTP status**：`400`

**curl 等价命令**：
```bash
curl -sS -X POST -H 'x-api-key: ***REDACTED***' -H 'Content-Type: application/json' \
  'https://build.twin.so/v1/agents/019db9b1-ecad-7222-992d-9e0a03231ac9/runs' \
  -d '{"run_mode":"Run","user_message":"开始 present_options 回路实验"}'
```
**请求 headers（脱敏）**：
```json
{"x-api-key":"***REDACTED***","Content-Type":"application/json"}
```
**请求 body**：
```json
{"run_mode":"Run","user_message":"开始 present_options 回路实验"}
```
**响应 headers（关键）**：
```json
{"content-type":"application/problem+json"}
```
**响应 body（截断 5000 字符）**：
```json
{"detail":"run_mode must be explicitly set to Builder, Run, or SubAgent","status":400,"title":"Bad Request"}
```
**评注**：v2 尝试 1：按用户指明的 run_mode="Run" 字符串，仍 400

---

### seq 7 · `start-run-attempt-2-builder` · POST (build)/v1/agents/019db9b1-ecad-7222-992d-9e0a03231ac9/runs

- **phase**：`B-v2-start-run`  |  **时间**：2026-04-23T09:36:44Z  |  **HTTP status**：`400`

**curl 等价命令**：
```bash
curl -sS -X POST -H 'x-api-key: ***REDACTED***' -H 'Content-Type: application/json' \
  'https://build.twin.so/v1/agents/019db9b1-ecad-7222-992d-9e0a03231ac9/runs' \
  -d '{"run_mode":"Builder","user_message":"测试 Builder 模式"}'
```
**请求 headers（脱敏）**：
```json
{"x-api-key":"***REDACTED***","Content-Type":"application/json"}
```
**请求 body**：
```json
{"run_mode":"Builder","user_message":"测试 Builder 模式"}
```
**响应 headers（关键）**：
```json
{"content-type":"application/problem+json"}
```
**响应 body（截断 5000 字符）**：
```json
{"detail":"run_mode must be explicitly set to Builder, Run, or SubAgent"}
```
**评注**：v2 尝试 2：run_mode="Builder"，同样 400。错误消息矛盾：服务器既接受 openapi 声明的 string 类型，又在所有字符串值（含其 enum 内部 variants 名称）上判定为"未设置"

---

### seq 8 · `start-run-attempt-3-skip-deploy` · POST (build)/v1/agents/019db9b1-ecad-7222-992d-9e0a03231ac9/runs

- **phase**：`B-v2-start-run`  |  **时间**：2026-04-23T09:37:10Z  |  **HTTP status**：`400`

**curl 等价命令**：
```bash
curl -sS -X POST -H 'x-api-key: ***REDACTED***' -H 'Content-Type: application/json' \
  'https://build.twin.so/v1/agents/019db9b1-ecad-7222-992d-9e0a03231ac9/runs' \
  -d '{"run_mode":"Run","skip_deploy_check":true,"user_message":"极简 body 第 3 次尝试"}'
```
**请求 headers（脱敏）**：
```json
{"x-api-key":"***REDACTED***","Content-Type":"application/json"}
```
**请求 body**：
```json
{"run_mode":"Run","skip_deploy_check":true,"user_message":"极简 body 第 3 次尝试"}
```
**响应 headers（关键）**：
```json
{"content-type":"application/problem+json"}
```
**响应 body（截断 5000 字符）**：
```json
{"detail":"run_mode must be explicitly set to Builder, Run, or SubAgent"}
```
**评注**：v2 尝试 3：加 skip_deploy_check:true，同样 400。结合 v1 trace seq 25（curl raw 直接发 `{"run_mode":"Run",...}` 也 400），可排除工具序列化问题。服务器此端点的 "Run" 字符串没有映射到内部 RunMode::Run variant。

---

### seq 9 · `poll-events-placeholder` · GET (build)/v1/agents/019db9b1-ecad-7222-992d-9e0a03231ac9/runs/placeholder-run/events

- **phase**：`B-v2-poll-events`  |  **时间**：2026-04-23T09:37:10Z  |  **HTTP status**：`200`

**curl 等价命令**：
```bash
curl -sS -H 'x-api-key: ***REDACTED***' -H 'Accept: application/json' \
  'https://build.twin.so/v1/agents/019db9b1-ecad-7222-992d-9e0a03231ac9/runs/placeholder-run/events'
```
**请求 headers（脱敏）**：
```json
{"x-api-key":"***REDACTED***","Accept":"application/json"}
```
**请求 body**：（无）
**响应 headers（关键）**：
```json
{"content-type":"application/json"}
```
**响应 body（截断 5000 字符）**：
```json
{"events":[],"total_count":0}
```
**评注**：v2 新发现：events 端点对 placeholder run_id 返回 200 {"events":[],"total_count":0}（非 404），证明此路径 REST 已实现且容忍未知 run_id（返回空数组）。对比 v1 对 api.twin.so 的同路径请求 404 content-length=22，可确认 api.twin.so 确实没这个路由

---

### seq 10 · `probe-A` · POST (build)/v1/agents/019db9b1-ecad-7222-992d-9e0a03231ac9/runs/placeholder-run/messages

- **phase**：`C-v2-probe`  |  **时间**：2026-04-23T09:37:23Z  |  **HTTP status**：`200`

**curl 等价命令**：
```bash
curl -sS -X POST -H 'x-api-key: ***REDACTED***' -H 'Content-Type: application/json' \
  'https://build.twin.so/v1/agents/019db9b1-ecad-7222-992d-9e0a03231ac9/runs/placeholder-run/messages' \
  -d '{"content":"红"}'
```
**请求 headers（脱敏）**：
```json
{"x-api-key":"***REDACTED***","Content-Type":"application/json"}
```
**请求 body**：
```json
{"content":"红"}
```
**响应 headers（关键）**：
```json
{"content-type":"application/grpc","grpc-status":"12","content-length":"0"}
```
**响应 body（截断 5000 字符）**：
```json
(空 body，grpc-status=12 UNIMPLEMENTED)
```
**评注**：v2 发现：probe-A 返回 HTTP 200 + grpc-status 12（UNIMPLEMENTED）。这**不是 REST 层 404**——gateway 接受该路径但底层 gRPC 服务未实现此方法。content-type=application/grpc 印证 build.twin.so 是 gRPC-web/gRPC-HTTP 桥接网关

---

### seq 11 · `probe-B` · POST (build)/v1/agents/019db9b1-ecad-7222-992d-9e0a03231ac9/runs/placeholder-run/input

- **phase**：`C-v2-probe`  |  **时间**：2026-04-23T09:37:23Z  |  **HTTP status**：`200`

**curl 等价命令**：
```bash
curl -sS -X POST -H 'x-api-key: ***REDACTED***' -H 'Content-Type: application/json' \
  'https://build.twin.so/v1/agents/019db9b1-ecad-7222-992d-9e0a03231ac9/runs/placeholder-run/input' \
  -d '{"value":"红"}'
```
**请求 headers（脱敏）**：
```json
{"x-api-key":"***REDACTED***","Content-Type":"application/json"}
```
**请求 body**：
```json
{"value":"红"}
```
**响应 headers（关键）**：
```json
{"content-type":"application/grpc","grpc-status":"12"}
```
**响应 body（截断 5000 字符）**：
```json
(空 body，grpc-status=12)
```
**评注**：probe-B 同样 HTTP 200 + grpc-status=12 UNIMPLEMENTED

---

### seq 12 · `probe-C` · POST (build)/v1/agents/019db9b1-ecad-7222-992d-9e0a03231ac9/runs/placeholder-run/answer

- **phase**：`C-v2-probe`  |  **时间**：2026-04-23T09:37:23Z  |  **HTTP status**：`200`

**curl 等价命令**：
```bash
curl -sS -X POST -H 'x-api-key: ***REDACTED***' -H 'Content-Type: application/json' \
  'https://build.twin.so/v1/agents/019db9b1-ecad-7222-992d-9e0a03231ac9/runs/placeholder-run/answer' \
  -d '{"answer":"红"}'
```
**请求 headers（脱敏）**：
```json
{"x-api-key":"***REDACTED***","Content-Type":"application/json"}
```
**请求 body**：
```json
{"answer":"红"}
```
**响应 headers（关键）**：
```json
{"content-type":"application/grpc","grpc-status":"12"}
```
**响应 body（截断 5000 字符）**：
```json
(空 body，grpc-status=12)
```
**评注**：probe-C 同样 HTTP 200 + grpc-status=12 UNIMPLEMENTED

---

### seq 13 · `probe-D` · PATCH (build)/v1/agents/019db9b1-ecad-7222-992d-9e0a03231ac9/runs/placeholder-run

- **phase**：`C-v2-probe`  |  **时间**：2026-04-23T09:37:23Z  |  **HTTP status**：`405`

**curl 等价命令**：
```bash
curl -sS -X PATCH -H 'x-api-key: ***REDACTED***' -H 'Content-Type: application/json' \
  'https://build.twin.so/v1/agents/019db9b1-ecad-7222-992d-9e0a03231ac9/runs/placeholder-run' \
  -d '{"user_input":"红"}'
```
**请求 headers（脱敏）**：
```json
{"x-api-key":"***REDACTED***","Content-Type":"application/json"}
```
**请求 body**：
```json
{"user_input":"红"}
```
**响应 headers（关键）**：
```json
{"allow":"DELETE","content-length":"0"}
```
**响应 body（截断 5000 字符）**：
```json
(空 body，Allow: DELETE)
```
**评注**：probe-D **PATCH 405 Method Not Allowed**，Allow header 明确只接受 DELETE（即 /v1/agents/{id}/runs/{rid} 路径在 openapi 里就是 DELETE）。这与 v1 结论一致

---

### seq 14 · `probe-E` · POST (build)/v1/agents/019db9b1-ecad-7222-992d-9e0a03231ac9/runs

- **phase**：`C-v2-probe`  |  **时间**：2026-04-23T09:37:23Z  |  **HTTP status**：`400`

**curl 等价命令**：
```bash
curl -sS -X POST -H 'x-api-key: ***REDACTED***' -H 'Content-Type: application/json' \
  'https://build.twin.so/v1/agents/019db9b1-ecad-7222-992d-9e0a03231ac9/runs' \
  -d '{"run_mode":"Run","run_id":"placeholder-run","user_message":"红"}'
```
**请求 headers（脱敏）**：
```json
{"x-api-key":"***REDACTED***","Content-Type":"application/json"}
```
**请求 body**：
```json
{"run_mode":"Run","run_id":"placeholder-run","user_message":"红"}
```
**响应 headers（关键）**：
```json
{"content-type":"application/problem+json"}
```
**响应 body（截断 5000 字符）**：
```json
{"detail":"run_mode must be explicitly set to Builder, Run, or SubAgent"}
```
**评注**：probe-E 复用 POST /runs 试图 resume：400，与 start-run 同样的 run_mode 识别失败；POST /runs 本身 openapi 声明的 StartRunBody 也没有 run_id 字段

---

### seq 15 · `probe-F` · POST (build)/v1/agents/019db9b1-ecad-7222-992d-9e0a03231ac9/runs/placeholder-run/reply

- **phase**：`C-v2-probe`  |  **时间**：2026-04-23T09:37:23Z  |  **HTTP status**：`200`

**curl 等价命令**：
```bash
curl -sS -X POST -H 'x-api-key: ***REDACTED***' -H 'Content-Type: application/json' \
  'https://build.twin.so/v1/agents/019db9b1-ecad-7222-992d-9e0a03231ac9/runs/placeholder-run/reply' \
  -d '{"reply":"红"}'
```
**请求 headers（脱敏）**：
```json
{"x-api-key":"***REDACTED***","Content-Type":"application/json"}
```
**请求 body**：
```json
{"reply":"红"}
```
**响应 headers（关键）**：
```json
{"content-type":"application/grpc","grpc-status":"12"}
```
**响应 body（截断 5000 字符）**：
```json
(空 body，grpc-status=12)
```
**评注**：probe-F HTTP 200 + grpc-status=12 UNIMPLEMENTED

---

### seq 16 · `probe-G` · POST (build)/v1/agents/019db9b1-ecad-7222-992d-9e0a03231ac9/runs/placeholder-run/respond

- **phase**：`C-v2-probe`  |  **时间**：2026-04-23T09:37:23Z  |  **HTTP status**：`200`

**curl 等价命令**：
```bash
curl -sS -X POST -H 'x-api-key: ***REDACTED***' -H 'Content-Type: application/json' \
  'https://build.twin.so/v1/agents/019db9b1-ecad-7222-992d-9e0a03231ac9/runs/placeholder-run/respond' \
  -d '{"response":"红"}'
```
**请求 headers（脱敏）**：
```json
{"x-api-key":"***REDACTED***","Content-Type":"application/json"}
```
**请求 body**：
```json
{"response":"红"}
```
**响应 headers（关键）**：
```json
{"content-type":"application/grpc","grpc-status":"12"}
```
**响应 body（截断 5000 字符）**：
```json
(空 body，grpc-status=12)
```
**评注**：probe-G HTTP 200 + grpc-status=12 UNIMPLEMENTED

---

### seq 17 · `probe-H` · POST (build)/v1/agents/019db9b1-ecad-7222-992d-9e0a03231ac9/runs/placeholder-run/resume

- **phase**：`C-v2-probe`  |  **时间**：2026-04-23T09:37:23Z  |  **HTTP status**：`200`

**curl 等价命令**：
```bash
curl -sS -X POST -H 'x-api-key: ***REDACTED***' -H 'Content-Type: application/json' \
  'https://build.twin.so/v1/agents/019db9b1-ecad-7222-992d-9e0a03231ac9/runs/placeholder-run/resume' \
  -d '{"input":"红"}'
```
**请求 headers（脱敏）**：
```json
{"x-api-key":"***REDACTED***","Content-Type":"application/json"}
```
**请求 body**：
```json
{"input":"红"}
```
**响应 headers（关键）**：
```json
{"content-type":"application/grpc","grpc-status":"12"}
```
**响应 body（截断 5000 字符）**：
```json
(空 body，grpc-status=12)
```
**评注**：probe-H HTTP 200 + grpc-status=12 UNIMPLEMENTED

---

### seq 18 · `cancel-placeholder-run` · POST (build)/v1/agents/019db9b1-ecad-7222-992d-9e0a03231ac9/runs/placeholder-run/cancel

- **phase**：`D-v2-cleanup`  |  **时间**：2026-04-23T09:37:35Z  |  **HTTP status**：`404`

**curl 等价命令**：
```bash
curl -sS -X POST -H 'x-api-key: ***REDACTED***' -H 'Content-Type: application/json' \
  'https://build.twin.so/v1/agents/019db9b1-ecad-7222-992d-9e0a03231ac9/runs/placeholder-run/cancel' \
  -d '{"reason":"实验结束 v2 清理"}'
```
**请求 headers（脱敏）**：
```json
{"x-api-key":"***REDACTED***","Content-Type":"application/json"}
```
**请求 body**：
```json
{"reason":"实验结束 v2 清理"}
```
**响应 headers（关键）**：
```json
{"content-type":"application/problem+json"}
```
**响应 body（截断 5000 字符）**：
```json
{"detail":"run placeholder-run not found","status":404,"title":"Not Found"}
```
**评注**：**关键对照**：cancel 端点返回业务层 404（detail 含具体 run_id），与 probe A-H 的 grpc-status=12 响应结构完全不同。证明 /cancel 是 openapi 声明的真 REST 端点（已实现），而 probe A-H 路径在 gateway 层能路由到某个 gRPC handler 但 handler 未实现——与 "端点不存在"（404 路由未注册）本质不同

---

### seq 19 · `list-runs-verify-no-run` · GET (build)/v1/agents/019db9b1-ecad-7222-992d-9e0a03231ac9/runs

- **phase**：`D-v2-cleanup`  |  **时间**：2026-04-23T09:37:35Z  |  **HTTP status**：`200`

**curl 等价命令**：
```bash
curl -sS -H 'x-api-key: ***REDACTED***' -H 'Accept: application/json' \
  'https://build.twin.so/v1/agents/019db9b1-ecad-7222-992d-9e0a03231ac9/runs'
```
**请求 headers（脱敏）**：
```json
{"x-api-key":"***REDACTED***","Accept":"application/json"}
```
**请求 body**：（无）
**响应 headers（关键）**：
```json
{"content-type":"application/json"}
```
**响应 body（截断 5000 字符）**：
```json
{"all_run_summaries":[],"page":1,"page_size":100,"runs":[],"total_runs":0}
```
**评注**：v2 确认：3 次 POST /runs 均未创建 run，total_runs=0，与 v1 一致

---

### seq 20 · `delete-toy-agent-v2` · DELETE (build)/v1/agents/019db9b1-ecad-7222-992d-9e0a03231ac9

- **phase**：`D-v2-cleanup`  |  **时间**：2026-04-23T09:37:35Z  |  **HTTP status**：`204`

**curl 等价命令**：
```bash
curl -sS -X DELETE -H 'x-api-key: ***REDACTED***' \
  'https://build.twin.so/v1/agents/019db9b1-ecad-7222-992d-9e0a03231ac9'
```
**请求 headers（脱敏）**：
```json
{"x-api-key":"***REDACTED***"}
```
**请求 body**：（无）
**响应 headers（关键）**：
```json
{"content-length":"0"}
```
**响应 body（截断 5000 字符）**：
```json
(空响应体)
```
**评注**：v2 清理完成：玩具 agent 已删除

---

## 5 · Interrupt 事件原始结构

**v2 实测结果：仍未能捕获到任何 `present_options` interrupt 事件。**

因 `POST /v1/agents/{id}/runs` 的 3 种 run_mode 变体全部 400（见 §3 发现 γ 和 §4 seq 6-8），v2 玩具 agent 也从未被实际启动。`interrupt_events_v2` 表唯一一条占位记录：

```json
{"_note":"v2 实验仍未能启动 run（POST /v1/agents/{id}/runs 在 3 种 run_mode 字符串变体下均返回 400），因此 events 流里没有任何 run 可观察。GET events?run_id=placeholder 返回 {\"events\":[],\"total_count\":0}，确认 placeholder 对应无事件。"}
```

### events 流的实际响应结构（旁证）

虽然没有真 run，但 `GET /v1/agents/{id}/runs/{rid}/events`（seq 9）对 placeholder run_id 返回 `200 {"events":[],"total_count":0}`，暴露了 events 接口的响应**顶层 schema**：

```json
{
  "events": [],
  "total_count": 0
}
```

即真正触发 interrupt 时，`events` 数组应含一个或多个 event 对象，但 event 对象的**内部字段形状**本次未能观察到（openapi 把 `RunEventItem` 声明为 opaque object，没有展开子字段）。

---

## 6 · 8 种 probe 的结果表格（v2 实测）

| probe | method + path | body | HTTP status | grpc-status | 响应摘要 | 结论 |
|---|---|---|---|---|---|---|
| A | `POST /v1/agents/{id}/runs/{rid}/messages` | `{"content":"红"}` | **200** | **12** | 空 body, `content-type: application/grpc` | ❌ **gRPC UNIMPLEMENTED**（未实现） |
| B | `POST /v1/agents/{id}/runs/{rid}/input` | `{"value":"红"}` | **200** | **12** | 空 body, application/grpc | ❌ gRPC UNIMPLEMENTED |
| C | `POST /v1/agents/{id}/runs/{rid}/answer` | `{"answer":"红"}` | **200** | **12** | 空 body, application/grpc | ❌ gRPC UNIMPLEMENTED |
| D | `PATCH /v1/agents/{id}/runs/{rid}` | `{"user_input":"红"}` | **405** | - | 空 body, `Allow: DELETE` | ❌ Method Not Allowed（该路径仅接受 DELETE） |
| E | `POST /v1/agents/{id}/runs` + resume body | `{"run_mode":"Run","run_id":"...","user_message":"红"}` | **400** | - | `run_mode must be explicitly set...` | ❌ run_mode 业务层校验失败（见 §3 发现 γ）；openapi StartRunBody 也无 run_id 字段 |
| F | `POST /v1/agents/{id}/runs/{rid}/reply` | `{"reply":"红"}` | **200** | **12** | 空 body, application/grpc | ❌ gRPC UNIMPLEMENTED |
| G | `POST /v1/agents/{id}/runs/{rid}/respond` | `{"response":"红"}` | **200** | **12** | 空 body, application/grpc | ❌ gRPC UNIMPLEMENTED |
| H | `POST /v1/agents/{id}/runs/{rid}/resume` | `{"input":"红"}` | **200** | **12** | 空 body, application/grpc | ❌ gRPC UNIMPLEMENTED |

**v2 结论：0 / 8 probe 成功。** v2 相对 v1 的关键补充：v1 在 `api.twin.so` 实测所有 probe 统一返回 `404 Not Found`（路由未注册）；v2 在 `build.twin.so` 上对同样路径的请求**并未 404**，而是返回 HTTP 200 + gRPC UNIMPLEMENTED（A/B/C/F/G/H）或 REST 405（D）或 run_mode 400（E）。gateway 层的路由表包含这些路径，只是底层 handler 未实现。

---

## 7 · run 最终状态

v2 实测路径上：

- **0 次** POST /v1/agents/{id}/runs 成功（3 种 run_mode 变体全部 400）。
- **0 次** events 接口观察到事件（`total_count: 0`，无 interrupt）。
- **玩具 agent 的 status 变化轨迹**：`draft / has_instruction_build=true / has_runs=false / total_runs=0`（seq 4、19）。agent 状态从创建到删除始终是 "draft 且无任何 run"。
- **清理**：`POST /runs/placeholder-run/cancel` → `404 run placeholder-run not found`（seq 18），`DELETE /v1/agents/{id}` → `204`（seq 20）。

---

## 8 · Secret 类型的可见性观察

**仍未能验证。** 理由同 v1：v2 的 run 也没能启动，三道 present_options 题（含第三题 Secret 假 token）全部未被触发。Secret 答复在 events 流里是否被 mask 的问题在 v1+v2 两版实验里均无可观测数据。

---

## 9 · 结论（v2 重写）

基于 v1（api.twin.so + build.twin.so，32 次请求）+ v2（builder.twin.so + build.twin.so，20 次请求）合计 **52 次真实 HTTP 请求**的实测：

1. **`api.twin.so` 是 LLM/goal-processing 辅助 API**，不含 agent / run / events 相关端点（v1 确认）。
2. **`builder.twin.so` 是 Web UI 前端的 CloudFront+S3 SPA** 部署，所有路径 fallback 到 index.html，不承载 REST API（v2 确认）。
3. **`build.twin.so` 是真正的 agent REST + gRPC 网关**，使用 `x-api-key` header 鉴权（响应 header 明示"Bearer tokens are not supported"）。
4. **在 `build.twin.so` 上可以做这些事情**（公开 REST API 表面能力）：
   - `POST /v1/agents` 创建 agent（201）
   - `GET /v1/agents` / `GET /v1/agents/{id}` 读 agent 元信息
   - `PUT /v1/agents/{id}/instructions` 写 instructions（200）
   - `GET /v1/agents/{id}/runs` 列 run（200，`total_runs: N`）
   - `GET /v1/agents/{id}/runs/{rid}/events` 读 events（200，即使 run_id 不存在也返回 200 空数组）
   - `POST /v1/agents/{id}/runs/{rid}/cancel` cancel run（run 不存在时 404）
   - `DELETE /v1/agents/{id}` 删 agent（204）
5. **在 `build.twin.so` 上无法做这些事情**（对公开 x-api-key 调用方）：
   - `POST /v1/agents/{id}/runs` **启动 run** — run_mode 字段在 openapi 声明与服务端业务校验之间存在分歧，3 种字符串变体（Run/Builder/Run+skip_deploy_check）与 v1 额外 5 种（tagged enum/map/int/null/空）合计 8 种都被判定为"未设置"。
   - 任何"回传用户答复"类 probe — 8 种候选路径在 gateway 层存在但 handler 全部 UNIMPLEMENTED（grpc-status=12）或 METHOD NOT ALLOWED。
6. **因此，本实验对核心问题的实测答案：**
   - Q1 能观察 interrupt 吗？ **✗** — events 接口本身可读，但因无法启动 run，观察不到任何 interrupt 事件；openapi 也未规范 event 对象的内部字段形状。
   - Q2 能回传答复吗？ **✗** — 8 种候选回传路径无一可用。
   - Q3 能闭合 present_options 回路吗？ **✗** — 两个前置条件（启动 run + 回传答复）都不成立。
   - Q4 Secret mask 行为？ **未能验证**。

> 以上结论仅基于 v1+v2 合计 52 次请求的实测数据。本报告不含对策建议。

---

## 10 · 附录：原始 api_trace_v2 JSON 导出

脱敏后的 v2 原始 `api_trace_v2` 与 `interrupt_events_v2` 完整 JSON 见同目录：

- [`09-present-options-roundtrip-raw.json`](./09-present-options-roundtrip-raw.json)

字段清单：
- `api_trace_v2`：20 条请求记录，按 `seq` 升序
- `interrupt_events_v2`：1 条记录（v2 仍为占位）
- 所有 `x-api-key` / `Authorization` header value 已脱敏为 `***REDACTED***`
- 响应 body 中未检出任何 `token/secret/password/api_key/authorization` 字段
