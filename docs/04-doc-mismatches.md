# 05 — 文档与实现不一致汇总

本次测试发现 **8 处**文档与实现的细节偏差。**没有一处是 "端点不能用"**；都属于"文档没写全"或"不同端点行为不统一"。按严重性从低到高排列：

## 1. 空状态返回形态

### `GET /v1/agents/{agent_id}/instructions (empty)` (#11)

- **HTTP 状态**: `200`
- **偏差**: 文档 docs/32 未明确说明无指令时返回什么，实测返回 HTTP 200 空体 {} — 建议文档补充此行为
- **备注**: 空体返回 {} 而非 404

### `GET /v1/agents/{agent_id}/schedule (empty)` (#14)

- **HTTP 状态**: `200`
- **偏差**: 文档 docs/26 未明确说明无调度时返回什么，实测返回 HTTP 200 空体 {} — 与 /v1/agents/{id}/schedule 有调度时返回 {schedule:{...}} 结构不同，建议文档补充
- **备注**: 空体返回 {} 而非 404

## 2. DELETE 的状态码与响应体

### `DELETE /v1/agents/{agent_id}/webhooks/{webhook_id}` (#36)

- **HTTP 状态**: `204`
- **偏差**: 实测返回 204 No Content（空响应体），但文档 docs/23 未明确规定状态码与响应形态 — 与其它 DELETE 端点行为不一致（有的返回 200 + body），建议文档补充
- **备注**: 删除 webhook：204 空响应

### `DELETE /v1/agents/{agent_id}/runs/{run_id}` (#38)

- **HTTP 状态**: `204`
- **偏差**: 实测返回 204 No Content，文档 docs/15 未规定状态码 — 与删除 agent、删除 webhook 一致（都是 204），但与删除 workspace (200+body)、删除 schedule (200+body)、撤销 api_key (200+body) 不一致 — 后端对 DELETE 语义存在两种风格
- **备注**: 删除 run：204 空响应

### `DELETE /v1/agents/{agent_id}` (#40)

- **HTTP 状态**: `204`
- **偏差**: 实测返回 204 No Content（空响应体），文档 docs/12 未规定状态码 — 与删除 webhook/删除 run 一致
- **备注**: 删除 agent：204 空响应

## 3. 已删除资源的访问语义

### `GET /v1/agents/{agent_id} (deleted)` (#42)

- **HTTP 状态**: `403`
- **偏差**: 预期返回 404 Not Found，实际返回 403 Forbidden + "you do not have access to this agent" — 后端把「不存在」与「无权限」统一返回 403。文档 docs/11 及 docs/04 未说明这点
- **备注**: 已删除资源被当作无权限处理，可能是安全设计但会让调用方难以区分 "404 不存在" 与 "403 无权限"

## 4. 写接口字段类型约束

### `PUT /v1/workspaces/{workspace_id} (with icon_config object)` (#31)

- **HTTP 状态**: `422`
- **偏差**: 422 Unprocessable Entity："icon_config: invalid type: map, expected a string" — 文档 docs/37 说 icon_config 是"可选的图标配置字符串"但示例 /v1/workspaces 的返回里 icon_config 存储为 JSON-stringified 字符串，更新接口要求调用方发送已 stringified 的字符串，而不是原生 JSON 对象。文档未明确提示这一点。
- **备注**: API 强制 icon_config 是一个已转义的 JSON 字符串

## 5. 边界语义 (run 已完成再取消)

### `POST /v1/agents/{agent_id}/runs/{run_id}/cancel` (#28)

- **HTTP 状态**: `200`
- **偏差**: 文档 docs/16 说明取消 run，但已完成的 run 被 cancel 时返回 200 OK + {"error":"run already completed","success":true}（同时含 success=true 和 error） — 这是一个有点混乱的语义，文档未覆盖此边界情况
- **备注**: Run 在 7 秒内已完成，cancel 实际上无效但接口返回成功

## 给 Twin 文档组的建议

1. **DELETE 端点的响应规范**需要统一：目前 `delete-agent / delete-run / delete-webhook` 返回 204 空体，而 `delete-schedule / delete-workspace / revoke-api-key` 返回 200 + `{success: true}` 或 `{deleted: true}` 或 `{revoked: true}`。建议要么全部 204、要么全部 200+body，或在每篇 DELETE 文档里明确说明。
2. **空状态 GET** (`GET /v1/agents/{id}/instructions`、`GET /v1/agents/{id}/schedule`) 在资源不存在时返回 200 + 空体 `{}` 而不是 404。这是合理的设计（让调用方不用处理 404 分支），但需要在文档里**明确写出**，否则调用方会猜测。
3. **`icon_config` 字段类型**：列表接口里 `icon_config` 以 JSON-stringified 字符串形式返回，更新接口也要求发 stringified 字符串。文档里的示例建议明确展示成 `"icon_config": "{\"backgroundColor\":\"#34D399\",\"icon\":\"Flask\"}"`，否则用户会误以为可以直接传对象。
4. **已删除 agent 的 GET 返回 403 "you do not have access" 而不是 404**。这可能是有意的（避免通过 id 枚举探测 agent 是否存在），但文档 docs/04-error-handling 里应该提一下这一点。
5. **取消已完成的 run**：`POST /runs/{id}/cancel` 目前返回 200 + `{"error":"run already completed","success":true}`。同时含 `success=true` 和 `error` 字段，语义含糊。建议要么返回 4xx、要么只返回 `{success:true}`。

下一章讨论性能与稳定性：[06 — 稳定性与性能](05-stability.md)
