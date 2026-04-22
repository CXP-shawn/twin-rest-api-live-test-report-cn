# 07 — 给开发者的使用建议

## TL;DR

**可以放心用**。33 个端点全部能正常工作，鉴权简单明了（一个 HTTP 头搞定），响应格式标准（JSON + RFC 7807 错误），整个测试期间 **0 次 5xx、0 次 429**。文档虽然在少数边界细节上有偏差，但不影响主流程开发。

## 建议踩的坑（提前看一下能省你两小时）

### 1. 鉴权是 `x-api-key`，不是 `Authorization: Bearer`

服务器会明确拒绝 Bearer token：
```
401 Unauthorized
Bearer tokens are not supported on the REST API.
Use the x-api-key header with an API key created from the Twin UI.
```

### 2. `icon_config` 永远是 JSON-stringified 字符串

无论是**读**还是**写**，`workspace.icon_config` 和 `agent.icon_config` 都是**字符串**，里面包着 JSON：

```json
{
  "icon_config": "{\"backgroundColor\":\"#34D399\",\"icon\":\"Flask\"}"
}
```

如果传原生对象 `{"icon_config": {"backgroundColor": ...}}` 会收到：
```
422 Unprocessable Entity
Failed to deserialize the JSON body into the target type:
icon_config: invalid type: map, expected a string
```

### 3. 空状态 GET 返回 `{}` 而不是 404

- `GET /v1/agents/{id}/instructions` 在 agent 还没写过 instructions 时返回 `200 + {}`
- `GET /v1/agents/{id}/schedule` 在 agent 还没配置调度时返回 `200 + {}`

客户端要用 `Object.keys(body).length === 0` 判断"空"，不要去 catch 404。

### 4. 已删除的 agent 返回 403 Not 404

GET 一个已删除的 agent 返回：
```
403 Forbidden
{ "detail": "you do not have access to this agent", "status": 403, ... }
```
这对区分"不存在"与"无权限"不友好。想确认是否删除成功，可以用 DELETE 返回的 204，而不是再 GET 一次。

### 5. DELETE 的返回体不统一

| 端点 | 成功响应 |
|---|---|
| `DELETE /v1/agents/{id}` | `204` + 空体 |
| `DELETE /v1/agents/{id}/runs/{run_id}` | `204` + 空体 |
| `DELETE /v1/agents/{id}/webhooks/{id}` | `204` + 空体 |
| `DELETE /v1/agents/{id}/schedule` | `200` + `{success: true}` |
| `DELETE /v1/workspaces/{id}` | `200` + `{deleted: true}` |
| `DELETE /api/public/v1/access-api-keys/{id}` | `200` + `{revoked: true}` |

**别依赖响应体判断成功**，用 `status_code >= 200 && status_code < 300` 就行。

### 6. `POST /runs/{id}/cancel` 不保证真的取消

短运行可能在你发 cancel 前就完成了。目前后端会返回：
```
200 OK
{ "error": "run already completed", "success": true }
```
语义是"操作本身没报错，但 run 已经完成了"。调用方应该先检查 run 是否还是 `in_progress`，或者直接把 cancel 当成**尽力而为**。

### 7. Run 事件接口响应很大

`GET /v1/agents/{id}/runs/{run_id}/events` 会返回一次调用里所有事件，测试时返回了 107KB / 11 条事件。生产环境里如果 run 很长（几十步工具调用），响应可能上 MB。建议：

- 流式或分页拿（目前文档未提供分页参数，可能需要后端加）
- 或者**用 webhook 代替轮询**：`POST /v1/agents/{id}/webhooks` 订阅 `run.completed` 等事件，让服务器主动推给你

### 8. Cron 是 7 字段而不是标准 5 字段

`PUT /v1/agents/{id}/schedule` 里的 `cron` 字段是 **7 字段格式**：`second minute hour day month weekday year`。例如 `"0 0 * * * * *"` = 每小时整点。标准 Unix crontab 是 5 字段，不要混用。

## 推荐的使用模式

### 最小可用接入（3 行代码跑一次 run）

```bash
# 1. 列出你的 agent，选一个
curl -H "x-api-key: $TWIN_API_KEY" https://build.twin.so/v1/agents

# 2. 触发一次运行
curl -X POST -H "x-api-key: $TWIN_API_KEY" -H "Content-Type: application/json" \
  -d '{"run_mode":"run","user_message":"请帮我执行工作流"}' \
  https://build.twin.so/v1/agents/<agent_id>/runs

# 3. 等 webhook 回推，或轮询 /runs/<run_id>/events
```

### 推荐的集成姿势

1. **用 OpenAPI 生成 SDK**：`https://build.twin.so/openapi.json` 是合规的 OpenAPI 3.1，直接 `openapi-generator` 生成你语言的 SDK。
2. **instructions 走 Git**：把 `instructions` 内容 checkin 到自己的 Git 仓库里，CI 里 `PUT /v1/agents/{id}/instructions` 同步。版本历史用 `GET .../instructions/history` 查。
3. **事件走 webhook**：不要轮询 `/events`，创建一个 webhook 订阅 `run.started/run.completed`。注意保存 `signing_secret` 用来验证签名。
4. **每个用户/租户一个 workspace**：`POST /v1/workspaces` + `POST /v1/agents/{id}/move`，通过 `workspace_id` 做查询隔离。

## 值得关注的小优点

- **错误响应用 RFC 7807**：`application/problem+json` + `{detail, status, title, type}`，比随意 JSON 错误格式清晰。
- **Next cursor 分页**：`GET /v1/agents` 用 `next_cursor` 游标而不是 page number，分页过程中即使有新数据也不会错位。
- **敏感凭据只在创建时返回一次**：`api_key` 和 `signing_secret` 分别只在创建 API Key 和创建 Webhook 时返回一次，后续 GET 都不会再给，符合最小暴露原则。
- **有 OpenAPI 3.1 spec**：`https://build.twin.so/openapi.json` 提供机器可读的 schema，可以生成 SDK / 导入 Postman / Insomnia。

## 我的总体打分

| 维度 | 评分 | 备注 |
|---|---|---|
| 可用性 | ⭐⭐⭐⭐⭐ | 33/33 端点可用 |
| 稳定性 | ⭐⭐⭐⭐⭐ | 0 次 5xx/429 |
| 文档完整度 | ⭐⭐⭐⭐ | 主流程清晰，边界细节有偏差 |
| 错误信息质量 | ⭐⭐⭐⭐⭐ | RFC 7807，提示明确 |
| 一致性 | ⭐⭐⭐ | DELETE 响应体不统一、空状态返回不统一 |
| 安全特性 | ⭐⭐⭐⭐⭐ | HSTS、CSP、敏感字段一次性返回 |

**总结**：如果你在做任何需要程序化管理 Twin Agent 的集成，这套 API 足够用且可信。只要你按本章列出的 8 个坑提前绕开，集成开发会很顺。

---

_报告完。原始调用数据请见 [`appendix/raw_experiments.json`](../appendix/raw_experiments.json)。_
