# 06 — 稳定性与性能观察

## HTTP 状态码分布

| 状态码 | 次数 | 含义 |
|---|---|---|
| 200 OK | 32 | 读与部分写操作 |
| 201 Created | 5 | 资源创建成功 |
| 204 No Content | 3 | 部分 DELETE |
| 403 Forbidden | 1 | 负向探测：已删除资源访问 |
| 422 Unprocessable Entity | 1 | `icon_config` 字段类型校验失败 |
| 5xx | **0** | 整个测试没遇到任何服务器错误 |

## 延迟观察

- agent 平台的 HTTP 工具没有暴露逐请求的精确延迟数字，但可以从响应头 `date` 推断：整个 42 次调用从 **14:41:46** 到 **14:46:56**，共 **~5 分钟**。扣除 agent 内部决策时间，平均**单次 HTTP 往返估计 < 500ms**。
- 唯一一个**明显慢**的请求是 `GET /v1/agents/{id}/runs/{run_id}/events`，因为它返回了 **107KB** 的事件流（11 条事件，含完整的 runner 内部状态树）。大响应体需要调用方在客户端做分页或流式处理。
- `POST /v1/agents/{id}/runs` 只需 < 1 秒就返回 `201 Created + status=in_progress`，真正的 policy 执行是异步的——这是正确的异步设计。

## 没遇到的问题

- **rate limit 触发**: 0 次（测试期间未触发 HTTP 429）
- **网络错误**: 0 次
- **5xx 服务器错误**: 0 次
- **响应乱码**: 0 次（UTF-8 在 request body 和 response body 里都正常）

## 响应头里的安全特征

- `content-security-policy: default-src 'none'` — 严格 CSP
- `strict-transport-security: max-age=31536000; includeSubDomains; preload` — HSTS 启用
- `access-control-allow-origin: https://builder.twin.so` — 只允许官方前端跨域
- `referrer-policy: strict-origin-when-cross-origin`
- `x-content-type-options: nosniff`
- `x-frame-options: SAMEORIGIN`
- `x-xss-protection: 1; mode=block`

服务器后端是 `uvicorn`（Python ASGI）；错误响应用的是 `application/problem+json`（RFC 7807）标准格式，含 `detail`、`status`、`title`、`type` 字段。

下一章给调用方的使用建议：[07 — 使用建议](06-recommendations.md)
