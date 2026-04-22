# 02 — 测试方法说明

## 被测对象

- **API 版本**：2026-04-22 线上版本
- **Base URL**：`https://build.twin.so`
- **文档来源**：私有仓库 `CXP-shawn/twin-rest-api-docs-analysis-cn`（对官方 <https://docs.twin.so/rest-api> 的中文逐章节结构化分析）
- **端点清单提取方式**：用 LLM 解析仓库里 43 个 markdown 文档，抽出每个端点的 `method/path/params/expected_response/source_doc_path`，入库 SQLite 表 `endpoints`。最终识别 **33 个可用端点**。

## API Key 注入方式（非常重要）

- 通过 agent 平台的 `present_options` 对话框以 **Secret 输入**取得 API Key。
- 立刻用 `establish_auth` 把它绑定成 **run 级别的私有变量** `{{ twin_api_key }}`，域限定为 `build.twin.so`。
- 每次 HTTP 调用通过工具模板 `x-api-key: {{ twin_api_key }}` 由平台在运行时注入，调用日志里看不到明文，agent 自己也拿不到字符串。
- **从未**写入：instructions、数据库、代码、报告、console 日志、GitHub 提交。
- 测试结束后该 Secret 绑定会随 run 结束而失效，下一次 run 需要重新通过 Secret 通道输入。

## HTTP 调用手法

我在 agent 平台里创建了 5 个**通用 HTTP 工具**（每个方法一个：`twin_api_get / twin_api_post / twin_api_put / twin_api_patch / twin_api_delete`），URL 全部走 path 参数 `https://build.twin.so{path}`，headers 固定带 `x-api-key`。

> 备注：原任务预想用 shell 里的 curl 完成，但 shell 沙箱没有 curl 也没有网络，实际上用 agent 平台原生的 HTTP 工具链更合适，因为它提供了自动鉴权注入、脱敏和响应体序列化。整个实验在逻辑上等价——每个端点都是一次真实的 HTTPS 请求、记录真实响应。

## 覆盖策略

| 端点类型 | 策略 |
|---|---|
| **纯读 (GET)** | 直接调用，读全局或全量列表 |
| **创建 (POST resource)** | 用真实最小合法 body 创建，记下返回的资源 id 放进 cleanup 队列 |
| **修改 (PATCH/PUT)** | 只作用于**自己刚创建的**沙盒资源 |
| **幂等触发 (POST action)** | 例如 schedule/pause、schedule/resume、move — 传空 body 或必填字段，调一次 |
| **运行类 (POST /runs)** | 真实触发一次最小 run，`user_message="This is a minimal API test run — please do nothing of consequence."`，消耗 2 credits |
| **删除 (DELETE)** | 放到 cleanup 阶段统一调用，同时也测到 DELETE 端点本身 |
| **敏感/全局排序 (reorder)** | 先 GET 当前顺序，原样传回，**no-op** 方式调用，不扰动其它用户数据 |

## 写操作的资源隔离

为了**绝对不碰**用户已有的数据，所有写操作只发生在一个临时沙盒里：

```
创建新 Workspace: twin-rest-api-live-test-sandbox
  └── 创建新 Agent (在这个 workspace 里)
        ├── PUT instructions
        ├── POST/PATCH/DELETE webhook
        ├── PUT/POST/DELETE schedule
        └── POST/GET/DELETE run
清理: 删除 Agent → 删除 Workspace → 撤销临时 API Key
```

只有两个"必须触及全局"的端点是例外：

- `POST /v1/workspaces/reorder`：用**no-op 策略**（传入当前实际顺序），保证不扰动任何排序
- `POST /v1/agents/{id}/move`：move 的是**我自己创建的 agent**，只是把它从沙盒工作区临时挪到一个已有工作区再挪回来，不碰别的 agent

## 记录格式

每次 HTTP 调用都完整记录到 SQLite 表 `experiment_log`：

```sql
CREATE TABLE experiment_log (
  seq INTEGER PRIMARY KEY AUTOINCREMENT,
  phase TEXT,                            -- 1-readonly / 2-create-sandbox / ... / 10-cleanup / 11-post-cleanup-probe
  endpoint_method TEXT,                  -- GET/POST/PUT/PATCH/DELETE
  endpoint_path TEXT,                    -- 原始 path 模板
  actual_url TEXT,                       -- 实际发出去的 URL (含真实 id)
  request_headers_sanitized TEXT,        -- headers JSON，x-api-key/authorization 均脱敏
  request_body TEXT,                     -- 请求体，敏感字段脱敏
  response_status INTEGER,               -- HTTP 状态码
  response_headers_sanitized TEXT,       -- 响应头摘要 (只留 content-type/length/date)
  response_body_truncated TEXT,          -- 响应体，脱敏 + 超过 3000 字符截断
  tested_at TEXT,                        -- ISO 时间戳
  matches_doc INTEGER,                   -- 1 = 与文档一致, 0 = 有偏差
  matches_doc_reason TEXT,               -- 判定原因
  notes TEXT                             -- 额外观察
);
```

## 敏感字段脱敏规则

调用方和响应里任何 key 名命中以下模式的，value 替换为 `***REDACTED***`：

- `signing_secret`
- `api_key`、`apikey`
- `authorization`
- `x-api-key`、`api-key`
- 包含 `token`、`secret`、`password` 子串

例如 `POST /api/public/v1/access-api-keys` 创建 API Key 时返回的完整 `api_key`、`POST /v1/agents/{id}/webhooks` 创建 webhook 时返回的 `signing_secret`，在本报告里都是 `***REDACTED***`。

## 报告体例

- 本报告全部中文，但 API 路径、参数名、HTTP 方法、字段名一律保留英文原文，方便对照官方 spec。
- 每个端点给出：实测请求 + 实测响应（脱敏截断）+ 判定结论。
- 原始 42 条完整调用记录见 [`appendix/raw_experiments.json`](../appendix/raw_experiments.json)。

下一章是 33 个端点的总览表：[03 — 端点总览](02-endpoint-overview.md)
