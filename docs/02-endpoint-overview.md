# 03 — 端点总览表（33 个）

下表按文档的分类顺序列出 Twin REST API 的全部 33 个端点，每一行给出**实测状态、HTTP 状态码、跳转到详细调用记录**的锚点。

> 图例：
> - ✅ 实测通过且与文档一致
> - ⚠️ 实测通过但发现文档未覆盖的行为偏差（不是 bug，是文档不完整）
> - ❌ 实测失败
> - ❓ 无数据

| # | 分类 | Method | 路径 | 名称 | 状态 | HTTP | 请求参数 | 响应关键字段 |
|---|---|---|---|---|---|---|---|---|
| 1 | auth | `GET` | `/api/public/v1/access-api-keys` | 列出 API 密钥 | ✅ | 200 | （无） | 包含 keys 数组的对象，每条记录包含 key_id、name、display_prefix、cr… |
| 2 | auth | `POST` | `/api/public/v1/access-api-keys` | 创建 API 密钥 | ✅ | 201 | 必填: name | 包含 key_id、name、api_key、display_prefix、created_at 字… |
| 3 | auth | `DELETE` | `/api/public/v1/access-api-keys/{key_id}` | 撤销 API 密钥 | ✅ | 200 | 必填: key_id | 包含 revoked: true 的对象 |
| 4 | identity | `GET` | `/v1/me` | 获取当前用户信息 | ✅ | 200 | （无） | 包含 user_id 字段的对象 |
| 5 | agents | `GET` | `/v1/agents` | 列出 Agent 列表 | ✅ | 200 | 可选: workspace_id, cursor, limit | 包含 agents 数组的对象，每条记录含 agent_id、latest_run_id、deplo… |
| 6 | agents | `POST` | `/v1/agents` | 创建 Agent | ✅ | 201 | 可选: workspace_id, owner_user_id, is_subagent, derived_from_agent_id | 完整的 Agent 对象，包含 agent_id、deployment_state 等字段，HTTP… |
| 7 | agents | `GET` | `/v1/agents/{agent_id}` | 获取单个 Agent 详情 | ✅ | 200 | 必填: agent_id | 完整 Agent 对象，包含 agent_id、latest_run_id、last_activit… |
| 8 | agents | `DELETE` | `/v1/agents/{agent_id}` | 删除 Agent | ⚠️ | 204 | 必填: agent_id | 204 No Content，无响应体 |
| 9 | runs | `GET` | `/v1/agents/{agent_id}/runs` | 列出运行记录 | ✅ | 200 | 必填: agent_id / 可选: page, page_size, filter_status, filter_run_id, filter_started_after, filter_started_before, filter_policy_type, filter_policy_group | 包含 runs 数组和总记录数的对象，每条记录含 run_id、status 等字段 |
| 10 | runs | `POST` | `/v1/agents/{agent_id}/runs` | 启动一个 Run | ✅ | 201 | 必填: agent_id / 可选: run_mode, user_message, skip_deploy_check | Run 对象，包含 run_id、status、started_at 等字段，HTTP 201 |
| 11 | runs | `DELETE` | `/v1/agents/{agent_id}/runs/{run_id}` | 删除指定运行记录 | ⚠️ | 204 | 必填: agent_id, run_id | 204 No Content，无响应体 |
| 12 | runs | `POST` | `/v1/agents/{agent_id}/runs/{run_id}/cancel` | 取消运行中的 Run | ⚠️ | 200 | 必填: agent_id, run_id / 可选: reason | 包含 error 字段的对象，error 为空表示成功 |
| 13 | events | `GET` | `/v1/agents/{agent_id}/runs/{run_id}/events` | 列出运行事件 | ✅ | 200 | 必填: agent_id, run_id / 可选: limit, after_index | 包含事件数组的对象，每条事件含事件索引和事件类型等字段 |
| 14 | webhooks | `POST` | `/v1/agents/{agent_id}/webhooks` | 创建 Webhook | ✅ | 201 | 必填: agent_id, url, events | Webhook 对象，包含 webhook_id、url、events、signing_secret… |
| 15 | webhooks | `GET` | `/v1/agents/{agent_id}/webhooks` | 列出 Webhook 列表 | ✅ | 200 | 必填: agent_id | 包含 webhooks 数组的对象，每条记录含 webhook_id、url、events、stat… |
| 16 | webhooks | `GET` | `/v1/agents/{agent_id}/webhooks/{webhook_id}` | 获取指定 Webhook 详情 | ✅ | 200 | 必填: agent_id, webhook_id | Webhook 对象，包含 webhook_id、url、events、status、created… |
| 17 | webhooks | `PATCH` | `/v1/agents/{agent_id}/webhooks/{webhook_id}` | 更新 Webhook | ✅ | 200 | 必填: agent_id, webhook_id / 可选: url, events, status | 更新后的完整 Webhook 对象，包含 webhook_id、url、events、status、… |
| 18 | webhooks | `DELETE` | `/v1/agents/{agent_id}/webhooks/{webhook_id}` | 删除 Webhook | ⚠️ | 204 | 必填: agent_id, webhook_id | 204 No Content，无响应体 |
| 19 | schedules | `GET` | `/v1/agents/{agent_id}/schedule` | 获取 Agent 调度计划 | ⚠️ | 200 | 必填: agent_id | 包含 schedule 对象（含 cron 和 paused 字段）或 {"schedule": n… |
| 20 | schedules | `PUT` | `/v1/agents/{agent_id}/schedule` | 设置 Agent 定时调度 | ✅ | 200 | 必填: agent_id, cron | 包含 {"success": true} 的对象 |
| 21 | schedules | `DELETE` | `/v1/agents/{agent_id}/schedule` | 删除 Agent 调度计划 | ✅ | 200 | 必填: agent_id | 包含 {"success": true} 的对象 |
| 22 | schedules | `POST` | `/v1/agents/{agent_id}/schedule/pause` | 暂停 Agent 调度计划 | ✅ | 200 | 必填: agent_id | 包含 {"success": true} 的对象 |
| 23 | schedules | `POST` | `/v1/agents/{agent_id}/schedule/resume` | 恢复计划调度 | ✅ | 200 | 必填: agent_id | 包含 {"success": true} 的对象 |
| 24 | schedules | `GET` | `/v1/schedules` | 获取所有调度计划列表 | ✅ | 200 | （无） | 包含 schedules 数组的对象，每条记录含 agent_id、cron、paused 字段 |
| 25 | instructions | `GET` | `/v1/agents/{agent_id}/instructions` | 获取 Agent 指令 | ⚠️ | 200 | 必填: agent_id | 包含 instructions 对象（含 content 字段）的 JSON |
| 26 | instructions | `PUT` | `/v1/agents/{agent_id}/instructions` | 更新智能体指令 | ✅ | 200 | 必填: agent_id, content / 可选: source_type, source_id | 包含 {"success": true} 的对象 |
| 27 | instructions | `GET` | `/v1/agents/{agent_id}/instructions/history` | 获取指令历史版本 | ✅ | 200 | 必填: agent_id / 可选: limit | 包含 instruction_versions 数组的对象，每条记录含 content 和 crea… |
| 28 | workspaces | `GET` | `/v1/workspaces` | 列出工作区 | ✅ | 200 | （无） | 包含 workspaces 数组的对象，每条记录含 workspace_id 和 name 字段 |
| 29 | workspaces | `POST` | `/v1/workspaces` | 创建工作区 | ✅ | 201 | 必填: name | 包含 workspace_id 和 name 字段的对象，HTTP 201 |
| 30 | workspaces | `PUT` | `/v1/workspaces/{workspace_id}` | 更新工作区 | ⚠️ | 422 | 必填: workspace_id, name / 可选: icon_config | 更新后的工作区对象，包含 workspace_id、name、icon_config 字段 |
| 31 | workspaces | `DELETE` | `/v1/workspaces/{workspace_id}` | 删除工作区 | ✅ | 200 | 必填: workspace_id | 包含 {"deleted": true} 的对象 |
| 32 | workspaces | `POST` | `/v1/workspaces/reorder` | 重新排序工作区 | ✅ | 200 | 必填: workspace_ids | 包含 {"success": true} 的对象 |
| 33 | agents | `POST` | `/v1/agents/{agent_id}/move` | 将 Agent 移动到指定工作区 | ✅ | 200 | 必填: agent_id, workspace_id | 包含 {"success": true} 的对象 |


## 按分类汇总

| 分类 | 端点数 | 全部 ✅ | 有 ⚠️ |
|---|---|---|---|
| auth | 3 | 3 | 0 |
| identity | 1 | 1 | 0 |
| agents | 5 | 4 | 1 |
| runs | 4 | 2 | 2 |
| events | 1 | 1 | 0 |
| webhooks | 5 | 4 | 1 |
| schedules | 6 | 5 | 1 |
| instructions | 3 | 2 | 1 |
| workspaces | 5 | 4 | 1 |


## 所有 ⚠️ 偏差的端点（点进去看细节）

- **`DELETE /v1/agents/{agent_id}`** — 实测返回 204 No Content（空响应体），文档 docs/12 未规定状态码 — 与删除 webhook/删除 run 一致
- **`DELETE /v1/agents/{agent_id}/runs/{run_id}`** — 实测返回 204 No Content，文档 docs/15 未规定状态码 — 与删除 agent、删除 webhook 一致（都是 204），但与删除 workspace (200+body)、删除 schedule (200+body)、撤销 api_key (200+body) 不一致 — 后端对 DELETE 语义存在两种风格
- **`POST /v1/agents/{agent_id}/runs/{run_id}/cancel`** — 文档 docs/16 说明取消 run，但已完成的 run 被 cancel 时返回 200 OK + {"error":"run already completed","success":true}（同时含 success=true 和 error） — 这是一个有点混乱的语义，文档未覆盖此边界情况
- **`DELETE /v1/agents/{agent_id}/webhooks/{webhook_id}`** — 实测返回 204 No Content（空响应体），但文档 docs/23 未明确规定状态码与响应形态 — 与其它 DELETE 端点行为不一致（有的返回 200 + body），建议文档补充
- **`GET /v1/agents/{agent_id}/schedule`** — 文档 docs/26 未明确说明无调度时返回什么，实测返回 HTTP 200 空体 {} — 与 /v1/agents/{id}/schedule 有调度时返回 {schedule:{...}} 结构不同，建议文档补充
- **`GET /v1/agents/{agent_id}/instructions`** — 文档 docs/32 未明确说明无指令时返回什么，实测返回 HTTP 200 空体 {} — 建议文档补充此行为
- **`PUT /v1/workspaces/{workspace_id}`** — 422 Unprocessable Entity："icon_config: invalid type: map, expected a string" — 文档 docs/37 说 icon_config 是"可选的图标配置字符串"但示例 /v1/workspaces 的返回里 icon_config 存储为 JSON-stringified 字符串，更新接口要求调用方发送已 stringified 的字符串，而不是原生 JSON 对象。文档未明确提示这一点。


下一章是每个阶段的详细调用记录：[04 — 详细调用记录](03-phase-by-phase.md)
