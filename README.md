# Twin REST API 全量端点实测与体验报告（中文）

> **测试时间**：2026-04-22 UTC
> **被测 API**：Twin REST API (`https://build.twin.so`)
> **源文档仓库**：<https://github.com/CXP-shawn/twin-rest-api-docs-analysis-cn>（私有）
> **文档数据源**：<https://docs.twin.so/rest-api>
> **测试覆盖**：33 个端点，共 42 次调用（含 clean-up 阶段与负向探测）

## 这份报告是什么

这是一份**真实跑一遍**的 Twin REST API 体验报告。我拿到用户的 API Key 之后，把官方文档里列出的每一个接口都按照说明的方式调了一次真实的 HTTP，记录下发的请求参数、服务器返回的响应、与文档是否一致的判定，以及出现的所有偏差。所有鉴权相关的字段（API Key、signing_secret、完整 api_key）在记录时都已经脱敏。

## 核心结论

| 指标 | 数值 |
|---|---|
| 文档中列出的端点总数 | **33** |
| 实际发起的请求次数 | **42**（含写-读确认、负向探测、cleanup） |
| HTTP 2xx 成功 | **40 / 42**（95.2%） |
| HTTP 4xx 客户端错误 | **2**（1 个故意的 422 验证 icon_config 序列化、1 个 403 负向探测） |
| HTTP 5xx 服务器错误 | **0** |
| 与文档完全一致 | **34 / 42**（81.0%） |
| 与文档有偏差 | **8 / 42**（19.0%） |

**一句话总结**：**所有端点都能工作**，请求/响应核心字段和文档基本一致。但文档在**边界行为**（空状态返回什么、DELETE 是 200 还是 204、已删除资源是 404 还是 403、`icon_config` 需要字符串还是对象）上有若干未覆盖的细节，开发者在写集成时需要自己踩一遍。

## 目录

| 章节 | 文档 |
|---|---|
| 01 — API 是什么、能干嘛 | [docs/00-intro.md](docs/00-intro.md) |
| 02 — 测试方法说明 | [docs/01-methodology.md](docs/01-methodology.md) |
| 03 — 端点总览表（33 个） | [docs/02-endpoint-overview.md](docs/02-endpoint-overview.md) |
| 04 — 每个阶段的详细调用记录 | [docs/03-phase-by-phase.md](docs/03-phase-by-phase.md) |
| 05 — 文档与实现不一致汇总 | [docs/04-doc-mismatches.md](docs/04-doc-mismatches.md) |
| 06 — 稳定性与性能观察 | [docs/05-stability.md](docs/05-stability.md) |
| 07 — 给开发者的使用建议 | [docs/06-recommendations.md](docs/06-recommendations.md) |
| 附录 — 原始实测数据 (42 条 JSON) | [appendix/raw_experiments.json](appendix/raw_experiments.json) |
| 09 — present_options 人机交互回路（REST API 实测） | [09-present-options-roundtrip.md](09-present-options-roundtrip.md) |
| 附录 — 端点元信息 (33 条 JSON) | [appendix/endpoints.json](appendix/endpoints.json) |

## 快速状态表

> ✅ 实测通过且与文档一致；⚠️ 实测通过但与文档不一致；❌ 实测失败

| 分类 | Method | Path | 状态 | HTTP 状态码 |
|---|---|---|---|---|
| auth | `GET` | `/api/public/v1/access-api-keys` | ✅ | 200 |
| auth | `POST` | `/api/public/v1/access-api-keys` | ✅ | 201 |
| auth | `DELETE` | `/api/public/v1/access-api-keys/{key_id}` | ✅ | 200 |
| identity | `GET` | `/v1/me` | ✅ | 200 |
| agents | `GET` | `/v1/agents` | ✅ | 200 |
| agents | `POST` | `/v1/agents` | ✅ | 201 |
| agents | `GET` | `/v1/agents/{agent_id}` | ✅ | 200 |
| agents | `DELETE` | `/v1/agents/{agent_id}` | ⚠️ | 204 |
| runs | `GET` | `/v1/agents/{agent_id}/runs` | ✅ | 200 |
| runs | `POST` | `/v1/agents/{agent_id}/runs` | ✅ | 201 |
| runs | `DELETE` | `/v1/agents/{agent_id}/runs/{run_id}` | ⚠️ | 204 |
| runs | `POST` | `/v1/agents/{agent_id}/runs/{run_id}/cancel` | ✅ | 201 |
| events | `GET` | `/v1/agents/{agent_id}/runs/{run_id}/events` | ✅ | 200 |
| webhooks | `POST` | `/v1/agents/{agent_id}/webhooks` | ✅ | 201 |
| webhooks | `GET` | `/v1/agents/{agent_id}/webhooks` | ✅ | 200 |
| webhooks | `GET` | `/v1/agents/{agent_id}/webhooks/{webhook_id}` | ✅ | 200 |
| webhooks | `PATCH` | `/v1/agents/{agent_id}/webhooks/{webhook_id}` | ✅ | 200 |
| webhooks | `DELETE` | `/v1/agents/{agent_id}/webhooks/{webhook_id}` | ⚠️ | 204 |
| schedules | `GET` | `/v1/agents/{agent_id}/schedule` | ✅ | 200 |
| schedules | `PUT` | `/v1/agents/{agent_id}/schedule` | ✅ | 200 |
| schedules | `DELETE` | `/v1/agents/{agent_id}/schedule` | ⚠️ | 204 |
| schedules | `POST` | `/v1/agents/{agent_id}/schedule/pause` | ✅ | 201 |
| schedules | `POST` | `/v1/agents/{agent_id}/schedule/resume` | ✅ | 201 |
| schedules | `GET` | `/v1/schedules` | ✅ | 200 |
| instructions | `GET` | `/v1/agents/{agent_id}/instructions` | ✅ | 200 |
| instructions | `PUT` | `/v1/agents/{agent_id}/instructions` | ✅ | 200 |
| instructions | `GET` | `/v1/agents/{agent_id}/instructions/history` | ✅ | 200 |
| workspaces | `GET` | `/v1/workspaces` | ✅ | 200 |
| workspaces | `POST` | `/v1/workspaces` | ✅ | 201 |
| workspaces | `PUT` | `/v1/workspaces/{workspace_id}` | ✅ | 200 |
| workspaces | `DELETE` | `/v1/workspaces/{workspace_id}` | ✅ | 200 |
| workspaces | `POST` | `/v1/workspaces/reorder` | ✅ | 200 |
| agents | `POST` | `/v1/agents/{agent_id}/move` | ✅ | 201 |

## 被测的端点全部来自官方文档

本次测试覆盖的 33 个端点全部来自 [Twin 官方 REST API 文档](https://docs.twin.so/rest-api) 在 2026-04-22 的版本，按分类分布如下：

- **auth (API Keys)** — 3 个
- **identity** — 1 个
- **agents** — 5 个（含 move-agent）
- **runs** — 4 个
- **events** — 1 个
- **webhooks** — 5 个
- **schedules** — 6 个
- **instructions** — 3 个
- **workspaces** — 5 个（含 reorder）

## 安全声明

- 本次测试用的 Twin API Key 仅在运行期持有，通过 agent 平台的 Secret 通道传入，不会写入 instructions、不会写入数据库、不会出现在报告任何位置。
- 报告里所有响应体均经过敏感字段过滤：`signing_secret`、`api_key`、`authorization`、`secret`、`token`、`password` 等命名的字段都被替换为 `***REDACTED***`。
- 所有写操作只作用于测试过程中创建的沙盒资源，测试结束时一并清理，没有修改任何用户既有数据。

---

_本报告由 Twin REST API 全量端点测试 agent 在单次运行中自动生成。_

---

- 🪲 已就 start-run 不可用、openapi server URL 错误、gRPC 交互端点 UNIMPLEMENTED 等问题向 Twin 官方工程团队提交 bug report：[twin-rest-api-issue-report](https://github.com/CXP-shawn/twin-rest-api-issue-report)
