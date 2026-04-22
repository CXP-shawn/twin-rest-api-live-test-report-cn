# 01 — Twin REST API 是什么、能干嘛

## 先说说 Twin 本身

[Twin](https://twin.so) 是一个让你"**写几段自然语言就能搭出自动化 Agent**"的平台。你在网页上告诉它"帮我每天早上汇总竞品动态"或"有新客户填完表就打个招呼"，它就会自己去调 API、抓网页、读邮件、写表格，把活干完。

它能做的事大致分四类：

- **信息类**：每天汇总新闻/竞品/项目动态
- **监控类**：盯着某个 KPI、某个页面、某个账户状态
- **外发类**：发邮件、发 Slack、发 LinkedIn
- **流水线类**：表单填完 → 自动去 CRM 建客户 → 自动发欢迎邮件

平时用 Twin 的人基本在 Web 界面里点点戳戳就行了。但如果你是开发者，想把 Twin 塞进**自家代码或 CI/CD 里**，就需要一套程序化的接口——这就是 **REST API** 的存在价值。

## REST API 让你能做什么

用官方 Twin UI 里"手点"的一切动作，REST API 都给你一个对应的程序入口：

| 你想干的事 | REST API 怎么帮你 |
|---|---|
| 批量创建 10 个 Agent（比如每个客户一个） | `POST /v1/agents` × 10 |
| 把某个 Agent 的调度从"每天 9 点"改成"每 2 小时" | `PUT /v1/agents/{id}/schedule` |
| 代码里触发一次 Agent 执行（比如 GitHub Action 里） | `POST /v1/agents/{id}/runs` |
| 把 Agent 产生的事件实时推到你自己系统里 | `POST /v1/agents/{id}/webhooks` |
| 查 Agent 最近 50 次运行的结果、成功率 | `GET /v1/agents/{id}/runs` |
| 版本化管理 Agent 的 instructions（类似"Git for prompts"） | `PUT /v1/agents/{id}/instructions` + `GET .../history` |
| 把 Agent 归到某个工作区、按团队隔离 | `POST /v1/workspaces` + `POST /v1/agents/{id}/move` |

## 什么场景下会用到

**场景 1：CI/CD 里定时触发回归测试 Agent**
在 GitHub Actions 里 `curl -X POST .../runs`，Agent 跑一遍端到端测试，失败了再通过 webhook 把事件回推到 Slack。

**场景 2：SaaS 产品集成**
你的 SaaS 里每来一个客户就用 Twin API 建一个客户专属 Agent（每个 Agent 挂一个 workspace），再通过 `POST /v1/workspaces/{id}` 做团队隔离。

**场景 3：一键部署 + 版本化 instructions**
把 `instructions` 存到自己的 Git 仓库里，CI/CD 同步 `PUT /v1/agents/{id}/instructions`——相当于给 Agent prompt 做 Git 管理。

**场景 4：把 Agent 事件接到自家数据湖**
通过 webhook 把 `run.completed` 等事件推到 Kafka → 数据仓库，用来统计成本、监控 token 消耗。

## 鉴权和基础 URL（一行说清楚）

- **Base URL**：`https://build.twin.so`（文档里也提到 `builder.twin.so` 是 openapi 里的正式 server，两者实测均可用）
- **鉴权方式**：HTTP 头 `x-api-key: <你的 API Key>`（**不能用 Bearer token**，试过会返回 401 且明确提示）
- **响应格式**：JSON（`application/json`），错误时返回 `application/problem+json`（RFC 7807）

## 它的能力边界（我这次测出来的）

- **全面支持**：Agent、Workspace、Run、Schedule、Webhook、Instruction、API Key 的 CRUD（33 个端点覆盖）
- **OpenAPI 规范**：`https://build.twin.so/openapi.json` 提供机器可读的 3.1 spec，可直接生成 SDK
- **没有**：批量操作、查询 DSL、长连接 SSE/WebSocket（events 目前要轮询）

下一章讲我是怎么测的：[02 — 测试方法说明](01-methodology.md)
