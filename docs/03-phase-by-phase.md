# 04 — 每个阶段的详细调用记录

本章完整记录 42 次真实 HTTP 调用的**请求参数、响应状态、响应体、判定结论**。每条记录都是带着真实 id、真实时间戳的实测数据。响应体若超过 3000 字符会自动截断，`signing_secret`、`api_key`、`authorization` 等敏感字段均已脱敏。

每条调用带一个 `#seq` 序号，对应 SQLite 表 `experiment_log` 的主键，可在附录的 [`raw_experiments.json`](../appendix/raw_experiments.json) 里查到完整原始数据。

---

## 阶段 1 — 只读端点冒烟测试 (5 次调用)

### #1 `GET /v1/me` ✅

- **实际 URL**: `https://build.twin.so/v1/me`
- **调用时间**: 2026-04-22T14:47:30.908Z
- **HTTP 状态**: `200`

**请求 Headers (脱敏)**
```json
{
  "Accept": "application/json",
  "x-api-key": "***REDACTED***"
}
```

**请求 Body**: _（无）_

**响应 Headers (摘要)**
```json
{
  "content-length": "45",
  "content-type": "application/json",
  "date": "Wed, 22 Apr 2026 14:41:46 GMT"
}
```

**响应 Body (脱敏)**
```json
{
  "user_id": "user_01KPPW8HKFNH8CH1GEQ73Y6VB3"
}
```

**判定**: ✅ 与文档一致

**依据**: 响应 {user_id} 与文档 docs/08-get-current-user.md 描述一致

**备注**: 基础可达性冒烟测试

---

### #2 `GET /v1/agents` ✅

- **实际 URL**: `https://build.twin.so/v1/agents`
- **调用时间**: 2026-04-22T14:47:30.908Z
- **HTTP 状态**: `200`

**请求 Headers (脱敏)**
```json
{
  "Accept": "application/json",
  "x-api-key": "***REDACTED***"
}
```

**请求 Body**: _（无）_

**响应 Headers (摘要)**
```json
{
  "content-length": "5426",
  "content-type": "application/json",
  "date": "Wed, 22 Apr 2026 14:42:08 GMT"
}
```

**响应 Body (脱敏、已截断)**
```json
{
  "agents": [
    {
      "agent_id": "019db59b-8abb-7913-bb07-a0f546af631f",
      "agent_name": {
        "name": "Twin 文档中英对照镜像"
      },
      "deployment_state": "draft",
      "has_instruction_build": false,
      "has_runs": true,
      "icon_config": "{\"backgroundColor\":\"#4ADE80\",\"icon\":\"BookOpen\"}",
      "last_activity_at": "2026-04-22T14:32:42.961047100+00:00",
      "latest_run_has_pending_input": false,
      "latest_run_id": "019db59b-8ad0-76d3-a040-5f37d3083999",
      "latest_run_is_finished": false,
      "latest_run_status": "in_progress",
      "workspace_id": "019db568-4171-7e62-9497-c42e4afd2b93"
    },
    {
      "agent_id": "019db597-6d79-7aa0-97d3-8405c837ebf1",
      "agent_name": {
        "name": "Twin REST API 全量端点测试"
      },
      "deployment_state": "draft",
      "has_instruction_build": false,
      "has_runs": true,
      "icon_config": "{\"backgroundColor\":\"#34D399\",\"icon\":\"Flask\"}",
      "last_activity_at": "2026-04-22T14:28:13.323206473+00:00",
      "latest_run_has_pending_input": false,
      "latest_run_id": "019db597-6d8b-71e1-88f7-e386f1721833",
      "latest_run_is_finished": false,
      "latest_run_status": "in_progress",
      "workspace_id": "019db592-7eda-7053-8e92-10cc777de4b0"
    },
    {
      "agent_id": "019db57c-016b-78d2-8f8c-2eaf13e1c619",
      "agent_name": {
        "name": "Temporal Go 文档中文翻译"
      },
      "deployment_state": "draft",
      "has_instruction_build": false,
      "has_runs": true,
      "icon_config": "{\"backgroundColor\":\"#A78BFA\",\"icon\":\"Book\"}",
      "last_activity_at": "2026-04-22T14:00:17.203723589+00:00",
      "latest_run_has_pending_input": false,
      "latest_run_id": "019db57c-017c-75a3-a197-a3ba3b0410c8",
      "latest_run_is_finished": true,
      "latest_run_status": "stopped",
      "workspace_id": "019db579-1934-7a02-a5ff-82f7b459f49b"
    },
    {
      "agent_id": "019db56a-00cf-7001-bb24-de6ce7d1d962",
      "agent_name": {
        "name": "Twin REST API 文档分析"
      },
      "deployment_state": "draft",
      "has_instruction_build": true,
      "has_runs": true,
      "icon_config": "{\"backgroundColor\":\"#A78BFA\",\"icon\":\"Book\"}",
      "last_activity_at": "2026-04-22T13:48:31.755668868+00:00",
      "latest_instruction_build_run_id": "019db56a-00e0-71e3-a6f3-92c43d68b5e8",
      "latest_run_has_pending_input": false,
      "latest_run_id": "019db56a-00e0-71e3-a6f3-92c43d68b5e8",
      "latest_run_is_finished": true,
      "latest_run_status": "completed",
      "workspace_id": "019db568-4171-7e62-9497-c42e4afd2b93"
    },
    {
      "agent_id": "019db508-70cd-7de0-836a-15611ae97a4c",
      "agent_name": {
        "name": "Anthropic Usage Report"
      },
      "deployment_state": "draft",
      "has_instruction_build": false,
      "has_runs": true,
      "icon_config": "{\"backgroundColor\":\"#34D399\",\"icon\":\"BarChart\"}",
      "last_activity_at": "2026-04-22T11:52:02.526548555+00:00",
      "latest_run_has_pe
... [truncated, original length=6576]
```

**判定**: ✅ 与文档一致

**依据**: 响应包含 agents[] 与 next_cursor 分页游标，单项含 agent_id/agent_name/deployment_state/has_instruction_build/has_runs/icon_config/last_activity_at/latest_run_id/latest_run_is_finished/latest_run_status/workspace_id

**备注**: 返回 10 条记录

---

### #3 `GET /v1/workspaces` ✅

- **实际 URL**: `https://build.twin.so/v1/workspaces`
- **调用时间**: 2026-04-22T14:47:30.909Z
- **HTTP 状态**: `200`

**请求 Headers (脱敏)**
```json
{
  "Accept": "application/json",
  "x-api-key": "***REDACTED***"
}
```

**请求 Body**: _（无）_

**响应 Headers (摘要)**
```json
{
  "content-length": "11763",
  "content-type": "application/json",
  "date": "Wed, 22 Apr 2026 14:42:08 GMT"
}
```

**响应 Body (脱敏、已截断)**
```json
{
  "workspaces": [
    {
      "agent_count": 0,
      "created_at": "2026-04-22T14:26:04.128176329+00:00",
      "name": "Crimson Ridge",
      "position": 0,
      "updated_at": "2026-04-22T14:26:04.128176329+00:00",
      "workspace_id": "019db595-74e0-7391-b674-4159c0502432"
    },
    {
      "agent_count": 1,
      "created_at": "2026-04-22T14:22:50.074936372+00:00",
      "icon_config": "{\"icon\":\"Code\",\"backgroundColor\":\"#FFD93D\"}",
      "name": "API 测试与开发者工具",
      "position": 1,
      "updated_at": "2026-04-22T14:27:11.043137361+00:00",
      "workspace_id": "019db592-7eda-7053-8e92-10cc777de4b0"
    },
    {
      "agent_count": 0,
      "created_at": "2026-04-22T13:57:14.148581382+00:00",
      "name": "Coral Nexus",
      "position": 2,
      "updated_at": "2026-04-22T13:57:14.148581382+00:00",
      "workspace_id": "019db57b-0f24-7380-9473-d956e1dd4212"
    },
    {
      "agent_count": 1,
      "created_at": "2026-04-22T13:55:05.652488499+00:00",
      "icon_config": "{\"icon\":\"Book\",\"backgroundColor\":\"#A78BFA\"}",
      "name": "文档翻译 & 技术研究",
      "position": 3,
      "updated_at": "2026-04-22T13:58:16.164380731+00:00",
      "workspace_id": "019db579-1934-7a02-a5ff-82f7b459f49b"
    },
    {
      "agent_count": 2,
      "created_at": "2026-04-22T13:36:41.841262981+00:00",
      "icon_config": "{\"icon\":\"Book\",\"backgroundColor\":\"#A78BFA\"}",
      "name": "开发者文档研究",
      "position": 4,
      "updated_at": "2026-04-22T13:38:36.407866939+00:00",
      "workspace_id": "019db568-4171-7e62-9497-c42e4afd2b93"
    },
    {
      "agent_count": 0,
      "created_at": "2026-04-22T12:35:01.567077054+00:00",
      "name": "Steady Vault",
      "position": 5,
      "updated_at": "2026-04-22T12:35:01.567077054+00:00",
      "workspace_id": "019db52f-cb3f-7f72-9edf-0e9cba237600"
    },
    {
      "agent_count": 1,
      "created_at": "2026-04-22T11:41:11.682874521+00:00",
      "icon_config": "{\"icon\":\"BarChart\",\"backgroundColor\":\"#34D399\"}",
      "name": "API Usage Monitoring",
      "position": 6,
      "updated_at": "2026-04-22T11:51:43.675834540+00:00",
      "workspace_id": "019db4fe-8282-7df1-8468-19d392790e72"
    },
    {
      "agent_count": 1,
      "created_at": "2026-04-22T11:14:45.635831378+00:00",
      "icon_config": "{\"icon\":\"SparkleCircle\",\"backgroundColor\":\"#FB923C\"}",
      "name": "AI 应用 & 界面",
      "position": 7,
      "updated_at": "2026-04-22T11:15:48.922233682+00:00",
      "workspace_id": "019db4e6-4f03-7f43-bb1b-aefed12f5abe"
    },
    {
      "agent_count": 1,
      "created_at": "2026-04-22T11:03:51.509023781+00:00",
      "name": "Twin LLM Tool Relay 20260422-130350",
      "position": 8,
      "updated_at": "2026-04-22T11:03:51.509023781+00:00",
      "workspace_id": "019db4dc-53d5-7ad2-945b-f8721143ab0c"
    },
    {
      "agent_count": 1,
      "created_at": "2026-04-22T10:48:23.493092599+00:00",
      "name": "Twin API Relay 20260422-124823",
      "position": 9,
   
... [truncated, original length=14521]
```

**判定**: ✅ 与文档一致

**依据**: 响应包含 workspaces[]，单项含 workspace_id/name/position/created_at/updated_at/agent_count/icon_config

**备注**: 返回 48 个工作区

---

### #4 `GET /v1/schedules` ✅

- **实际 URL**: `https://build.twin.so/v1/schedules`
- **调用时间**: 2026-04-22T14:47:30.909Z
- **HTTP 状态**: `200`

**请求 Headers (脱敏)**
```json
{
  "Accept": "application/json",
  "x-api-key": "***REDACTED***"
}
```

**请求 Body**: _（无）_

**响应 Headers (摘要)**
```json
{
  "content-length": "173",
  "content-type": "application/json",
  "date": "Wed, 22 Apr 2026 14:42:08 GMT"
}
```

**响应 Body (脱敏)**
```json
{
  "schedules": [
    {
      "agent_id": "019daeb5-2f6c-75a3-88b1-362bd500df6a",
      "auto_paused": false,
      "consecutive_failures": 0,
      "cron": "0 0 12 * * * *",
      "next_run": 1776945600,
      "paused": false
    }
  ]
}
```

**判定**: ✅ 与文档一致

**依据**: 响应包含 schedules[]，单项含 agent_id/cron/next_run/paused/auto_paused/consecutive_failures

**备注**: 返回 1 条调度

---

### #5 `GET /api/public/v1/access-api-keys` ✅

- **实际 URL**: `https://build.twin.so/api/public/v1/access-api-keys`
- **调用时间**: 2026-04-22T14:47:30.909Z
- **HTTP 状态**: `200`

**请求 Headers (脱敏)**
```json
{
  "Accept": "application/json",
  "x-api-key": "***REDACTED***"
}
```

**请求 Body**: _（无）_

**响应 Headers (摘要)**
```json
{
  "content-length": "664",
  "content-type": "application/json",
  "date": "Wed, 22 Apr 2026 14:42:08 GMT"
}
```

**响应 Body (脱敏)**
```json
{
  "keys": [
    {
      "created_at": 1776867709,
      "display_prefix": "7fpJD8sY",
      "key_id": "89cfb82b7ab50872962c35f674dee255",
      "last_used_at": 1776868928,
      "name": "test"
    },
    {
      "created_at": 1776851089,
      "display_prefix": "R5-Heo94",
      "key_id": "443793c9a61f00db97eb396ce20fd544",
      "last_used_at": 1776859717,
      "name": "kimi"
    },
    {
      "created_at": 1776766653,
      "display_prefix": "T-kVfJif",
      "key_id": "21e93b5b5815d2b14a80812d21ec5593",
      "last_used_at": 1776767979,
      "name": "my-api-key"
    },
    {
      "created_at": 1776766653,
      "display_prefix": "Bll6y5fM",
      "key_id": "76f20e0cc77bbf3ed369226d2b994368",
      "name": "my-api-key"
    },
    {
      "created_at": 1776766561,
      "display_prefix": "68SsDlvI",
      "key_id": "7e945f0c4f998a51eb6dab95129e3d86",
      "name": "test-key"
    }
  ]
}
```

**判定**: ✅ 与文档一致

**依据**: 响应包含 keys[]，单项含 key_id/name/display_prefix/created_at/last_used_at（不含密钥原文，安全）

**备注**: 返回 5 个 API key 元数据

---

## 阶段 2 — 创建沙盒 Workspace + Agent (2 次调用)

### #6 `POST /v1/workspaces` ✅

- **实际 URL**: `https://build.twin.so/v1/workspaces`
- **调用时间**: 2026-04-22T14:47:30.909Z
- **HTTP 状态**: `201`

**请求 Headers (脱敏)**
```json
{
  "Accept": "application/json",
  "Content-Type": "application/json",
  "x-api-key": "***REDACTED***"
}
```

**请求 Body**
```json
{
  "name": "twin-rest-api-live-test-sandbox"
}
```

**响应 Headers (摘要)**
```json
{
  "content-length": "241",
  "content-type": "application/json",
  "date": "Wed, 22 Apr 2026 14:43:19 GMT"
}
```

**响应 Body (脱敏)**
```json
{
  "workspace": {
    "agent_count": 0,
    "created_at": "2026-04-22T14:43:19.117863929+00:00",
    "name": "twin-rest-api-live-test-sandbox",
    "position": 0,
    "updated_at": "2026-04-22T14:43:19.117863929+00:00",
    "workspace_id": "019db5a5-3fcd-71e3-80aa-640d6e4253bc"
  }
}
```

**判定**: ✅ 与文档一致

**依据**: 201 Created，响应 workspace 对象包含完整字段

**备注**: 创建测试沙盒工作区，id=019db5a5-3fcd-71e3-80aa-640d6e4253bc

---

### #7 `POST /v1/agents` ✅

- **实际 URL**: `https://build.twin.so/v1/agents`
- **调用时间**: 2026-04-22T14:47:30.909Z
- **HTTP 状态**: `201`

**请求 Headers (脱敏)**
```json
{
  "Accept": "application/json",
  "Content-Type": "application/json",
  "x-api-key": "***REDACTED***"
}
```

**请求 Body**
```json
{
  "workspace_id": "019db5a5-3fcd-71e3-80aa-640d6e4253bc",
  "is_subagent": false
}
```

**响应 Headers (摘要)**
```json
{
  "content-length": "364",
  "content-type": "application/json",
  "date": "Wed, 22 Apr 2026 14:43:24 GMT"
}
```

**响应 Body (脱敏)**
```json
{
  "agent": {
    "agent_id": "019db5a5-56b8-7a40-8b0e-ee557077c754",
    "deployment_state": "draft",
    "has_instruction_build": false,
    "has_runs": false,
    "icon_config": "{\"backgroundColor\":\"#A78BFA\"}",
    "last_activity_at": "2026-04-22T14:43:24.984599422+00:00",
    "latest_run_has_pending_input": false,
    "latest_run_is_finished": false,
    "workspace_id": "019db5a5-3fcd-71e3-80aa-640d6e4253bc"
  }
}
```

**判定**: ✅ 与文档一致

**依据**: 201 Created，响应 agent 对象包含 agent_id/workspace_id/deployment_state=draft/has_instruction_build=false/has_runs=false

**备注**: 创建测试 Agent，id=019db5a5-56b8-7a40-8b0e-ee557077c754

---

## 阶段 3 — 新 Agent 上的空状态读取 (7 次调用)

### #8 `GET /v1/agents/{agent_id}` ✅

- **实际 URL**: `https://build.twin.so/v1/agents/019db5a5-56b8-7a40-8b0e-ee557077c754`
- **调用时间**: 2026-04-22T14:48:15.811Z
- **HTTP 状态**: `200`

**请求 Headers (脱敏)**
```json
{
  "Accept": "application/json",
  "x-api-key": "***REDACTED***"
}
```

**请求 Body**: _（无）_

**响应 Headers (摘要)**
```json
{
  "content-length": "364",
  "content-type": "application/json",
  "date": "Wed, 22 Apr 2026 14:43:35 GMT"
}
```

**响应 Body (脱敏)**
```json
{
  "agent": {
    "agent_id": "019db5a5-56b8-7a40-8b0e-ee557077c754",
    "deployment_state": "draft",
    "has_instruction_build": false,
    "has_runs": false,
    "icon_config": "{\"backgroundColor\":\"#A78BFA\"}",
    "last_activity_at": "2026-04-22T14:43:24.984599422+00:00",
    "latest_run_has_pending_input": false,
    "latest_run_is_finished": false,
    "workspace_id": "019db5a5-3fcd-71e3-80aa-640d6e4253bc"
  }
}
```

**判定**: ✅ 与文档一致

**依据**: 响应 agent 对象含完整字段，与 docs/11 一致

---

### #9 `GET /v1/agents?workspace_id=...` ✅

- **实际 URL**: `https://build.twin.so/v1/agents?workspace_id=019db5a5-3fcd-71e3-80aa-640d6e4253bc`
- **调用时间**: 2026-04-22T14:48:15.811Z
- **HTTP 状态**: `200`

**请求 Headers (脱敏)**
```json
{
  "Accept": "application/json",
  "x-api-key": "***REDACTED***"
}
```

**请求 Body**: _（无）_

**响应 Headers (摘要)**
```json
{
  "content-length": "367",
  "content-type": "application/json",
  "date": "Wed, 22 Apr 2026 14:43:35 GMT"
}
```

**响应 Body (脱敏)**
```json
{
  "agents": [
    {
      "agent_id": "019db5a5-56b8-7a40-8b0e-ee557077c754",
      "deployment_state": "draft",
      "has_instruction_build": false,
      "has_runs": false,
      "icon_config": "{\"backgroundColor\":\"#A78BFA\"}",
      "last_activity_at": "2026-04-22T14:43:24.984599422+00:00",
      "latest_run_has_pending_input": false,
      "latest_run_is_finished": false,
      "workspace_id": "019db5a5-3fcd-71e3-80aa-640d6e4253bc"
    }
  ]
}
```

**判定**: ✅ 与文档一致

**依据**: workspace_id 查询参数生效，只返回沙盒 workspace 内的 1 个 agent

**备注**: query 过滤正常

---

### #10 `GET /v1/agents/{agent_id}/runs` ✅

- **实际 URL**: `https://build.twin.so/v1/agents/019db5a5-56b8-7a40-8b0e-ee557077c754/runs`
- **调用时间**: 2026-04-22T14:48:15.811Z
- **HTTP 状态**: `200`

**请求 Headers (脱敏)**
```json
{
  "Accept": "application/json",
  "x-api-key": "***REDACTED***"
}
```

**请求 Body**: _（无）_

**响应 Headers (摘要)**
```json
{
  "content-length": "74",
  "content-type": "application/json",
  "date": "Wed, 22 Apr 2026 14:43:35 GMT"
}
```

**响应 Body (脱敏)**
```json
{
  "all_run_summaries": [],
  "page": 1,
  "page_size": 100,
  "runs": [],
  "total_runs": 0
}
```

**判定**: ✅ 与文档一致

**依据**: 响应含 all_run_summaries=[] / page=1 / page_size=100 / runs=[] / total_runs=0，与 docs/13 一致

---

### #11 `GET /v1/agents/{agent_id}/instructions (empty)` ⚠️

- **实际 URL**: `https://build.twin.so/v1/agents/019db5a5-56b8-7a40-8b0e-ee557077c754/instructions`
- **调用时间**: 2026-04-22T14:48:15.811Z
- **HTTP 状态**: `200`

**请求 Headers (脱敏)**
```json
{
  "Accept": "application/json",
  "x-api-key": "***REDACTED***"
}
```

**请求 Body**: _（无）_

**响应 Headers (摘要)**
```json
{
  "content-length": "2",
  "content-type": "application/json",
  "date": "Wed, 22 Apr 2026 14:43:35 GMT"
}
```

**响应 Body (脱敏)**
```json
{}
```

**判定**: ⚠️ 与文档有偏差

**依据**: 文档 docs/32 未明确说明无指令时返回什么，实测返回 HTTP 200 空体 {} — 建议文档补充此行为

**备注**: 空体返回 {} 而非 404

---

### #12 `GET /v1/agents/{agent_id}/instructions/history (empty)` ✅

- **实际 URL**: `https://build.twin.so/v1/agents/019db5a5-56b8-7a40-8b0e-ee557077c754/instructions/history`
- **调用时间**: 2026-04-22T14:48:15.811Z
- **HTTP 状态**: `200`

**请求 Headers (脱敏)**
```json
{
  "Accept": "application/json",
  "x-api-key": "***REDACTED***"
}
```

**请求 Body**: _（无）_

**响应 Headers (摘要)**
```json
{
  "content-length": "27",
  "content-type": "application/json",
  "date": "Wed, 22 Apr 2026 14:43:35 GMT"
}
```

**响应 Body (脱敏)**
```json
{
  "instruction_versions": []
}
```

**判定**: ✅ 与文档一致

**依据**: 响应 {instruction_versions: []}，与 docs/34 一致

---

### #13 `GET /v1/agents/{agent_id}/webhooks (empty)` ✅

- **实际 URL**: `https://build.twin.so/v1/agents/019db5a5-56b8-7a40-8b0e-ee557077c754/webhooks`
- **调用时间**: 2026-04-22T14:48:15.811Z
- **HTTP 状态**: `200`

**请求 Headers (脱敏)**
```json
{
  "Accept": "application/json",
  "x-api-key": "***REDACTED***"
}
```

**请求 Body**: _（无）_

**响应 Headers (摘要)**
```json
{
  "content-length": "15",
  "content-type": "application/json",
  "date": "Wed, 22 Apr 2026 14:43:35 GMT"
}
```

**响应 Body (脱敏)**
```json
{
  "webhooks": []
}
```

**判定**: ✅ 与文档一致

**依据**: 响应 {webhooks: []}，与 docs/20 一致

---

### #14 `GET /v1/agents/{agent_id}/schedule (empty)` ⚠️

- **实际 URL**: `https://build.twin.so/v1/agents/019db5a5-56b8-7a40-8b0e-ee557077c754/schedule`
- **调用时间**: 2026-04-22T14:48:15.811Z
- **HTTP 状态**: `200`

**请求 Headers (脱敏)**
```json
{
  "Accept": "application/json",
  "x-api-key": "***REDACTED***"
}
```

**请求 Body**: _（无）_

**响应 Headers (摘要)**
```json
{
  "content-length": "2",
  "content-type": "application/json",
  "date": "Wed, 22 Apr 2026 14:43:35 GMT"
}
```

**响应 Body (脱敏)**
```json
{}
```

**判定**: ⚠️ 与文档有偏差

**依据**: 文档 docs/26 未明确说明无调度时返回什么，实测返回 HTTP 200 空体 {} — 与 /v1/agents/{id}/schedule 有调度时返回 {schedule:{...}} 结构不同，建议文档补充

**备注**: 空体返回 {} 而非 404

---

## 阶段 4 — Instructions 写入与读取 (3 次调用)

### #15 `PUT /v1/agents/{agent_id}/instructions` ✅

- **实际 URL**: `https://build.twin.so/v1/agents/019db5a5-56b8-7a40-8b0e-ee557077c754/instructions`
- **调用时间**: 2026-04-22T14:48:15.811Z
- **HTTP 状态**: `200`

**请求 Headers (脱敏)**
```json
{
  "Accept": "application/json",
  "Content-Type": "application/json",
  "x-api-key": "***REDACTED***"
}
```

**请求 Body**
```json
{
  "content": "# 测试 Agent\n这是一个由 Twin REST API 全量端点测试 agent 自动创建的测试指令。\n用于验证 PUT /v1/agents/{agent_id}/instructions 接口。"
}
```

**响应 Headers (摘要)**
```json
{
  "content-length": "16",
  "content-type": "application/json",
  "date": "Wed, 22 Apr 2026 14:43:45 GMT"
}
```

**响应 Body (脱敏)**
```json
{
  "success": true
}
```

**判定**: ✅ 与文档一致

**依据**: 响应 {success: true}，与 docs/33 一致

**备注**: 写入指令成功

---

### #16 `GET /v1/agents/{agent_id}/instructions (after write)` ✅

- **实际 URL**: `https://build.twin.so/v1/agents/019db5a5-56b8-7a40-8b0e-ee557077c754/instructions`
- **调用时间**: 2026-04-22T14:48:15.811Z
- **HTTP 状态**: `200`

**请求 Headers (脱敏)**
```json
{
  "Accept": "application/json",
  "x-api-key": "***REDACTED***"
}
```

**请求 Body**: _（无）_

**响应 Headers (摘要)**
```json
{
  "content-length": "650",
  "content-type": "application/json",
  "date": "Wed, 22 Apr 2026 14:43:50 GMT"
}
```

**响应 Body (脱敏)**
```json
{
  "instructions": {
    "agent_id": "019db5a5-56b8-7a40-8b0e-ee557077c754",
    "content": "# 测试 Agent\n这是一个由 Twin REST API 全量端点测试 agent 自动创建的测试指令。\n用于验证 PUT /v1/agents/{agent_id}/instructions 接口。",
    "content_hash": "da276a30d602cb8755e1fba12775faf63becf74f8ed988c548b5ef1e1f262fd9",
    "source_id": "checkpoint:instructions:2026-04-22T14:43:45.469076031+00:00",
    "storage_key": "agents/019db5a5-56b8-7a40-8b0e-ee557077c754/instructions/1776869025469-da276a30d602cb87.md",
    "updated_at": "2026-04-22T14:43:45.469076031+00:00",
    "user_email": "anadiabalsemao@halleysfifth.com",
    "user_id": "user_01KPPW8HKFNH8CH1GEQ73Y6VB3"
  }
}
```

**判定**: ✅ 与文档一致

**依据**: 响应 instructions 对象含 agent_id/content/content_hash/source_id/storage_key/updated_at/user_email/user_id，与 docs/32 一致

**备注**: 中文内容完整保留

---

### #17 `GET /v1/agents/{agent_id}/instructions/history (after write)` ✅

- **实际 URL**: `https://build.twin.so/v1/agents/019db5a5-56b8-7a40-8b0e-ee557077c754/instructions/history`
- **调用时间**: 2026-04-22T14:48:15.811Z
- **HTTP 状态**: `200`

**请求 Headers (脱敏)**
```json
{
  "Accept": "application/json",
  "x-api-key": "***REDACTED***"
}
```

**请求 Body**: _（无）_

**响应 Headers (摘要)**
```json
{
  "content-length": "558",
  "content-type": "application/json",
  "date": "Wed, 22 Apr 2026 14:43:50 GMT"
}
```

**响应 Body (脱敏)**
```json
{
  "instruction_versions": [
    {
      "content": "# 测试 Agent\n这是一个由 Twin REST API 全量端点测试 agent 自动创建的测试指令。\n用于验证 PUT /v1/agents/{agent_id}/instructions 接口。",
      "content_hash": "da276a30d602cb8755e1fba12775faf63becf74f8ed988c548b5ef1e1f262fd9",
      "created_at": "2026-04-22T14:43:45.469076031+00:00",
      "id": 109843,
      "source_id": "checkpoint:instructions:2026-04-22T14:43:45.469076031+00:00",
      "source_type": "checkpoint",
      "storage_key": "agents/019db5a5-56b8-7a40-8b0e-ee557077c754/instructions/1776869025469-da276a30d602cb87.md"
    }
  ]
}
```

**判定**: ✅ 与文档一致

**依据**: 响应 instruction_versions[1]，单项含 id/content/content_hash/created_at/source_id/source_type/storage_key，与 docs/34 一致

---

## 阶段 5 — Webhook CRUD (4 次调用，不含最后的 DELETE)

### #18 `POST /v1/agents/{agent_id}/webhooks` ✅

- **实际 URL**: `https://build.twin.so/v1/agents/019db5a5-56b8-7a40-8b0e-ee557077c754/webhooks`
- **调用时间**: 2026-04-22T14:48:15.811Z
- **HTTP 状态**: `201`

**请求 Headers (脱敏)**
```json
{
  "Accept": "application/json",
  "Content-Type": "application/json",
  "x-api-key": "***REDACTED***"
}
```

**请求 Body**
```json
{
  "url": "https://example.com/webhook-test-receiver",
  "events": [
    "run.started",
    "run.completed"
  ]
}
```

**响应 Headers (摘要)**
```json
{
  "content-length": "303",
  "content-type": "application/json",
  "date": "Wed, 22 Apr 2026 14:43:55 GMT"
}
```

**响应 Body (脱敏)**
```json
{
  "created_at": "2026-04-22T14:43:55.949962620+00:00",
  "events": [
    "run.started",
    "run.completed"
  ],
  "signing_secret": "***REDACTED***",
  "status": "active",
  "url": "https://example.com/webhook-test-receiver",
  "webhook_id": "019db5a5-cfac-71b1-9a8f-2d9320d7f3de"
}
```

**判定**: ✅ 与文档一致

**依据**: 201 Created，响应含 webhook_id/url/events/status=active/signing_secret（已脱敏），与 docs/19 一致

**备注**: webhook_id=019db5a5-cfac-71b1-9a8f-2d9320d7f3de

---

### #19 `GET /v1/agents/{agent_id}/webhooks/{webhook_id}` ✅

- **实际 URL**: `https://build.twin.so/v1/agents/019db5a5-56b8-7a40-8b0e-ee557077c754/webhooks/019db5a5-cfac-71b1-9a8f-2d9320d7f3de`
- **调用时间**: 2026-04-22T14:48:15.811Z
- **HTTP 状态**: `200`

**请求 Headers (脱敏)**
```json
{
  "Accept": "application/json",
  "x-api-key": "***REDACTED***"
}
```

**请求 Body**: _（无）_

**响应 Headers (摘要)**
```json
{
  "content-length": "358",
  "content-type": "application/json",
  "date": "Wed, 22 Apr 2026 14:44:08 GMT"
}
```

**响应 Body (脱敏)**
```json
{
  "agent_id": "019db5a5-56b8-7a40-8b0e-ee557077c754",
  "created_at": "2026-04-22T14:43:55.948950020+00:00",
  "events": [
    "run.started",
    "run.completed"
  ],
  "status": "active",
  "updated_at": "2026-04-22T14:43:55.948950020+00:00",
  "url": "https://example.com/webhook-test-receiver",
  "user_id": "user_01KPPW8HKFNH8CH1GEQ73Y6VB3",
  "webhook_id": "019db5a5-cfac-71b1-9a8f-2d9320d7f3de"
}
```

**判定**: ✅ 与文档一致

**依据**: 响应含 webhook_id/agent_id/url/events/status/user_id/created_at/updated_at，与 docs/21 一致

**备注**: GET 单个 webhook 不包含 signing_secret（安全）

---

### #20 `GET /v1/agents/{agent_id}/webhooks (after create)` ✅

- **实际 URL**: `https://build.twin.so/v1/agents/019db5a5-56b8-7a40-8b0e-ee557077c754/webhooks`
- **调用时间**: 2026-04-22T14:48:15.811Z
- **HTTP 状态**: `200`

**请求 Headers (脱敏)**
```json
{
  "Accept": "application/json",
  "x-api-key": "***REDACTED***"
}
```

**请求 Body**: _（无）_

**响应 Headers (摘要)**
```json
{
  "content-length": "373",
  "content-type": "application/json",
  "date": "Wed, 22 Apr 2026 14:44:08 GMT"
}
```

**响应 Body (脱敏)**
```json
{
  "webhooks": [
    {
      "agent_id": "019db5a5-56b8-7a40-8b0e-ee557077c754",
      "created_at": "2026-04-22T14:43:55.948950020+00:00",
      "events": [
        "run.started",
        "run.completed"
      ],
      "status": "active",
      "updated_at": "2026-04-22T14:43:55.948950020+00:00",
      "url": "https://example.com/webhook-test-receiver",
      "user_id": "user_01KPPW8HKFNH8CH1GEQ73Y6VB3",
      "webhook_id": "019db5a5-cfac-71b1-9a8f-2d9320d7f3de"
    }
  ]
}
```

**判定**: ✅ 与文档一致

**依据**: 响应 {webhooks: [{...}]} 含刚创建的 webhook

---

### #21 `PATCH /v1/agents/{agent_id}/webhooks/{webhook_id}` ✅

- **实际 URL**: `https://build.twin.so/v1/agents/019db5a5-56b8-7a40-8b0e-ee557077c754/webhooks/019db5a5-cfac-71b1-9a8f-2d9320d7f3de`
- **调用时间**: 2026-04-22T14:48:15.811Z
- **HTTP 状态**: `200`

**请求 Headers (脱敏)**
```json
{
  "Accept": "application/json",
  "Content-Type": "application/json",
  "x-api-key": "***REDACTED***"
}
```

**请求 Body**
```json
{
  "events": [
    "run.completed"
  ],
  "status": "disabled"
}
```

**响应 Headers (摘要)**
```json
{
  "content-length": "346",
  "content-type": "application/json",
  "date": "Wed, 22 Apr 2026 14:44:08 GMT"
}
```

**响应 Body (脱敏)**
```json
{
  "agent_id": "019db5a5-56b8-7a40-8b0e-ee557077c754",
  "created_at": "2026-04-22T14:43:55.948950020+00:00",
  "events": [
    "run.completed"
  ],
  "status": "disabled",
  "updated_at": "2026-04-22T14:44:08.532804991+00:00",
  "url": "https://example.com/webhook-test-receiver",
  "user_id": "user_01KPPW8HKFNH8CH1GEQ73Y6VB3",
  "webhook_id": "019db5a5-cfac-71b1-9a8f-2d9320d7f3de"
}
```

**判定**: ✅ 与文档一致

**依据**: 200 OK，响应完整 webhook 对象，events 变为 [run.completed]，status=disabled，与 docs/22 一致

**备注**: 支持部分字段更新

---

## 阶段 6 — Schedule 生命周期 (4 次调用，不含最后的 DELETE)

### #22 `PUT /v1/agents/{agent_id}/schedule` ✅

- **实际 URL**: `https://build.twin.so/v1/agents/019db5a5-56b8-7a40-8b0e-ee557077c754/schedule`
- **调用时间**: 2026-04-22T14:49:13.752Z
- **HTTP 状态**: `200`

**请求 Headers (脱敏)**
```json
{
  "Accept": "application/json",
  "Content-Type": "application/json",
  "x-api-key": "***REDACTED***"
}
```

**请求 Body**
```json
{
  "cron": "0 0 * * * * *"
}
```

**响应 Headers (摘要)**
```json
{
  "content-length": "16",
  "content-type": "application/json",
  "date": "Wed, 22 Apr 2026 14:44:13 GMT"
}
```

**响应 Body (脱敏)**
```json
{
  "success": true
}
```

**判定**: ✅ 与文档一致

**依据**: 200 OK 响应 {success: true}，与 docs/27 一致

**备注**: cron 为 7 字段 (second minute hour day month weekday year)

---

### #23 `GET /v1/agents/{agent_id}/schedule (after PUT)` ✅

- **实际 URL**: `https://build.twin.so/v1/agents/019db5a5-56b8-7a40-8b0e-ee557077c754/schedule`
- **调用时间**: 2026-04-22T14:49:13.752Z
- **HTTP 状态**: `200`

**请求 Headers (脱敏)**
```json
{
  "Accept": "application/json",
  "x-api-key": "***REDACTED***"
}
```

**请求 Body**: _（无）_

**响应 Headers (摘要)**
```json
{
  "content-length": "169",
  "content-type": "application/json",
  "date": "Wed, 22 Apr 2026 14:44:24 GMT"
}
```

**响应 Body (脱敏)**
```json
{
  "schedule": {
    "agent_id": "019db5a5-56b8-7a40-8b0e-ee557077c754",
    "auto_paused": false,
    "consecutive_failures": 0,
    "cron": "0 0 * * * * *",
    "next_run": 1776870000,
    "paused": false
  }
}
```

**判定**: ✅ 与文档一致

**依据**: 响应 schedule 对象含 agent_id/cron/next_run/paused=false/auto_paused=false/consecutive_failures=0，与 docs/26 一致

**备注**: next_run 返回 epoch 秒

---

### #24 `POST /v1/agents/{agent_id}/schedule/pause` ✅

- **实际 URL**: `https://build.twin.so/v1/agents/019db5a5-56b8-7a40-8b0e-ee557077c754/schedule/pause`
- **调用时间**: 2026-04-22T14:49:13.752Z
- **HTTP 状态**: `200`

**请求 Headers (脱敏)**
```json
{
  "Accept": "application/json",
  "Content-Type": "application/json",
  "x-api-key": "***REDACTED***"
}
```

**请求 Body**
```json
{}
```

**响应 Headers (摘要)**
```json
{
  "content-length": "16",
  "content-type": "application/json",
  "date": "Wed, 22 Apr 2026 14:44:24 GMT"
}
```

**响应 Body (脱敏)**
```json
{
  "success": true
}
```

**判定**: ✅ 与文档一致

**依据**: 200 OK 响应 {success: true}，与 docs/29 一致

**备注**: 请求体为空 {}

---

### #25 `POST /v1/agents/{agent_id}/schedule/resume` ✅

- **实际 URL**: `https://build.twin.so/v1/agents/019db5a5-56b8-7a40-8b0e-ee557077c754/schedule/resume`
- **调用时间**: 2026-04-22T14:49:13.752Z
- **HTTP 状态**: `200`

**请求 Headers (脱敏)**
```json
{
  "Accept": "application/json",
  "Content-Type": "application/json",
  "x-api-key": "***REDACTED***"
}
```

**请求 Body**
```json
{}
```

**响应 Headers (摘要)**
```json
{
  "content-length": "16",
  "content-type": "application/json",
  "date": "Wed, 22 Apr 2026 14:44:29 GMT"
}
```

**响应 Body (脱敏)**
```json
{
  "success": true
}
```

**判定**: ✅ 与文档一致

**依据**: 200 OK 响应 {success: true}，与 docs/30 一致

**备注**: 请求体为空 {}

---

## 阶段 7 — Run 生命周期 (4 次调用，不含最后的 DELETE)

### #26 `POST /v1/agents/{agent_id}/runs` ✅

- **实际 URL**: `https://build.twin.so/v1/agents/019db5a5-56b8-7a40-8b0e-ee557077c754/runs`
- **调用时间**: 2026-04-22T14:49:13.752Z
- **HTTP 状态**: `201`

**请求 Headers (脱敏)**
```json
{
  "Accept": "application/json",
  "Content-Type": "application/json",
  "x-api-key": "***REDACTED***"
}
```

**请求 Body**
```json
{
  "run_mode": "run",
  "user_message": "This is a minimal API test run — please do nothing of consequence."
}
```

**响应 Headers (摘要)**
```json
{
  "content-length": "471",
  "content-type": "application/json",
  "date": "Wed, 22 Apr 2026 14:44:36 GMT"
}
```

**响应 Body (脱敏)**
```json
{
  "run": {
    "agent_id": "019db5a5-56b8-7a40-8b0e-ee557077c754",
    "event_bytes": 0,
    "event_count": 0,
    "goal": "",
    "is_finished": false,
    "last_event_at": "2026-04-22T14:44:36.897083341+00:00",
    "policy_type": "runner_v2",
    "run_id": "019db5a6-6fa0-7c93-afb7-bb91a6434960",
    "run_number": 0,
    "started_at": "2026-04-22T14:44:36.897083341+00:00",
    "status": "in_progress",
    "step_count": 0,
    "user_email": "anadiabalsemao@halleysfifth.com",
    "user_id": "user_01KPPW8HKFNH8CH1GEQ73Y6VB3",
    "wrote_instructions": false
  }
}
```

**判定**: ✅ 与文档一致

**依据**: 201 Created，响应 run 对象含 run_id/run_number/status=in_progress/started_at/policy_type/user_id，与 docs/14 一致

**备注**: run_id=019db5a6-6fa0-7c93-afb7-bb91a6434960

---

### #27 `GET /v1/agents/{agent_id}/runs/{run_id}/events` ✅

- **实际 URL**: `https://build.twin.so/v1/agents/019db5a5-56b8-7a40-8b0e-ee557077c754/runs/019db5a6-6fa0-7c93-afb7-bb91a6434960/events`
- **调用时间**: 2026-04-22T14:49:13.760Z
- **HTTP 状态**: `200`

**请求 Headers (脱敏)**
```json
{
  "Accept": "application/json",
  "x-api-key": "***REDACTED***"
}
```

**请求 Body**: _（无）_

**响应 Headers (摘要)**
```json
{
  "content-length": "107120",
  "content-type": "application/json",
  "date": "Wed, 22 Apr 2026 14:44:42 GMT"
}
```

**响应 Body (脱敏、已截断)**
```json
{
  "events": [
    {
      "event": {
        "agent_id": {
          "id": "019db5a5-56b8-7a40-8b0e-ee557077c754"
        },
        "event": {
          "event": {
            "Started": {
              "agent_database_schema": null,
              "agent_id": "019db5a5-56b8-7a40-8b0e-ee557077c754",
              "agent_instructions_context": "# 测试 Agent\n这是一个由 Twin REST API 全量端点测试 agent 自动创建的测试指令。\n用于验证 PUT /v1/agents/{agent_id}/instructions 接口。",
              "agent_memory_context": null,
              "artifacts": {
                "agent_name_struct": null,
                "api_key_pattern_migrated": true,
                "builtin_tool_version": 1,
                "cua_goal_overrides": [],
                "instruction_storage_key": null,
                "recorded_tools": [],
                "required_auths": [],
                "run_usage_estimate": null,
                "templates_storage_key": null,
                "tools_cleared": true
              },
              "authenticated_services": [],
              "builtin_tool_version": 1,
              "chat_files": [],
              "dynamic_tools": [
                {
                  "application": {
                    "app_type": "api",
                    "application_id": "service:perplexity",
                    "canonical_domain": "perplexity.ai",
                    "canonical_slug": "perplexity",
                    "display_name": "Perplexity",
                    "icon_url": "https://favicon.im/perplexity.ai?larger=true",
                    "observed_domain": "api.perplexity.ai",
                    "resolution_source": "embedded_catalog",
                    "service": "perplexity"
                  },
                  "auth_variables": [],
                  "body_schema": {
                    "kind": {
                      "StructValue": {
                        "fields": {
                          "$schema": {
                            "kind": {
                              "StringValue": "http://json-schema.org/draft-07/schema#"
                            }
                          },
                          "description": {
                            "kind": {
                              "StringValue": "Web search arguments - delegates to Perplexity Sonar agent for web research"
                            }
                          },
                          "properties": {
                            "kind": {
                              "StructValue": {
                                "fields": {
                                  "comment": {
                                    "kind": {
                                      "StructValue": {
                                        "fields": {
                                          "description": {
                                            "kind": {
                                              "StringValue": "Short user-facing message describing what this tool call does (e.g. 'Fetching your l
... [truncated, original length=504900]
```

**判定**: ✅ 与文档一致

**依据**: 响应 {events: [...], total_count: 11}，包含 Started/RunPolicyInitialized/PolicyDecisionRequested/PolicyDecided/Finished 等 discriminated union 事件类型，与 docs/17 和 docs/18 一致

**备注**: events 数组较大 (107KB)，包含完整的 runner 内部状态

---

### #28 `POST /v1/agents/{agent_id}/runs/{run_id}/cancel` ⚠️

- **实际 URL**: `https://build.twin.so/v1/agents/019db5a5-56b8-7a40-8b0e-ee557077c754/runs/019db5a6-6fa0-7c93-afb7-bb91a6434960/cancel`
- **调用时间**: 2026-04-22T14:49:13.760Z
- **HTTP 状态**: `200`

**请求 Headers (脱敏)**
```json
{
  "Accept": "application/json",
  "Content-Type": "application/json",
  "x-api-key": "***REDACTED***"
}
```

**请求 Body**
```json
{
  "reason": "Cancelled by REST API test agent — minimal smoke run, no action needed."
}
```

**响应 Headers (摘要)**
```json
{
  "content-length": "48",
  "content-type": "application/json",
  "date": "Wed, 22 Apr 2026 14:44:42 GMT"
}
```

**响应 Body (脱敏)**
```json
{
  "error": "run already completed",
  "success": true
}
```

**判定**: ⚠️ 与文档有偏差

**依据**: 文档 docs/16 说明取消 run，但已完成的 run 被 cancel 时返回 200 OK + {"error":"run already completed","success":true}（同时含 success=true 和 error） — 这是一个有点混乱的语义，文档未覆盖此边界情况

**备注**: Run 在 7 秒内已完成，cancel 实际上无效但接口返回成功

---

### #29 `GET /v1/agents/{agent_id}/runs (after run)` ✅

- **实际 URL**: `https://build.twin.so/v1/agents/019db5a5-56b8-7a40-8b0e-ee557077c754/runs`
- **调用时间**: 2026-04-22T14:49:13.760Z
- **HTTP 状态**: `200`

**请求 Headers (脱敏)**
```json
{
  "Accept": "application/json",
  "x-api-key": "***REDACTED***"
}
```

**请求 Body**: _（无）_

**响应 Headers (摘要)**
```json
{
  "content-length": "1070",
  "content-type": "application/json",
  "date": "Wed, 22 Apr 2026 14:44:51 GMT"
}
```

**响应 Body (脱敏)**
```json
{
  "all_run_summaries": [
    {
      "credits_consumed": 2,
      "is_finished": true,
      "is_paused": false,
      "outcome": "success",
      "policy_model": "claude-sonnet-4-6",
      "policy_type": "runner_v2",
      "run_id": "019db5a6-6fa0-7c93-afb7-bb91a6434960",
      "run_kind": "run",
      "run_number": 1,
      "run_phase": "finished",
      "started_at": "2026-04-22T14:44:36.897083341+00:00",
      "summary": "This was a minimal test run with no consequential actions taken.",
      "tool_icons": [
        {
          "action_type": "running",
          "icon": "check-circle"
        }
      ],
      "total_action_count": 1,
      "wrote_instructions": false
    }
  ],
  "page": 1,
  "page_size": 100,
  "runs": [
    {
      "agent_id": "019db5a5-56b8-7a40-8b0e-ee557077c754",
      "event_bytes": 0,
      "event_count": 0,
      "goal": "",
      "is_finished": true,
      "last_event_at": "2026-04-22T14:44:36.897083341+00:00",
      "policy_model": "claude-sonnet-4-6",
      "policy_type": "runner_v2",
      "run_id": "019db5a6-6fa0-7c93-afb7-bb91a6434960",
      "run_kind": "run",
      "run_number": 1,
      "started_at": "2026-04-22T14:44:36.897083341+00:00",
      "status": "completed",
      "step_count": 0,
      "user_email": "anadiabalsemao@halleysfifth.com",
      "user_id": "user_01KPPW8HKFNH8CH1GEQ73Y6VB3",
      "wrote_instructions": false
    }
  ],
  "total_runs": 1
}
```

**判定**: ✅ 与文档一致

**依据**: 响应含 all_run_summaries[1]/runs[1]/total_runs=1，all_run_summaries 中的 run 含 credits_consumed/outcome/summary/run_phase 等更丰富字段

**备注**: outcome=success, credits_consumed=2

---

## 阶段 8 — API Key 创建 (1 次调用；撤销放在 cleanup)

### #30 `POST /api/public/v1/access-api-keys` ✅

- **实际 URL**: `https://build.twin.so/api/public/v1/access-api-keys`
- **调用时间**: 2026-04-22T14:49:13.760Z
- **HTTP 状态**: `201`

**请求 Headers (脱敏)**
```json
{
  "Accept": "application/json",
  "Content-Type": "application/json",
  "x-api-key": "***REDACTED***"
}
```

**请求 Body**
```json
{
  "name": "twin-rest-api-live-test-temp-key"
}
```

**响应 Headers (摘要)**
```json
{
  "content-length": "233",
  "content-type": "application/json",
  "date": "Wed, 22 Apr 2026 14:44:56 GMT"
}
```

**响应 Body (脱敏)**
```json
{
  "api_key": "***REDACTED***",
  "created_at": 1776869096,
  "display_prefix": "5XKODXoM",
  "key_id": "f28d7f04b58e24361d2b0dd4e8e96579",
  "name": "twin-rest-api-live-test-temp-key"
}
```

**判定**: ✅ 与文档一致

**依据**: 201 Created，响应含 key_id/name/display_prefix/created_at/api_key（完整密钥，已脱敏），与 docs/06 一致

**备注**: 完整 api_key 仅在创建时返回一次

---

## 阶段 9 — Workspace 更新 + Agent 跨区移动 + Reorder (5 次调用)

### #31 `PUT /v1/workspaces/{workspace_id} (with icon_config object)` ❌

- **实际 URL**: `https://build.twin.so/v1/workspaces/019db5a5-3fcd-71e3-80aa-640d6e4253bc`
- **调用时间**: 2026-04-22T14:49:13.760Z
- **HTTP 状态**: `422`

**请求 Headers (脱敏)**
```json
{
  "Accept": "application/json",
  "Content-Type": "application/json",
  "x-api-key": "***REDACTED***"
}
```

**请求 Body**
```json
{
  "name": "twin-rest-api-live-test-sandbox (updated)",
  "icon_config": {
    "backgroundColor": "#34D399",
    "icon": "Flask"
  }
}
```

**响应 Headers (摘要)**
```json
{
  "content-length": "127",
  "content-type": "text/plain; charset=utf-8",
  "date": "Wed, 22 Apr 2026 14:45:45 GMT"
}
```

**响应 Body (脱敏)**
```json
"Failed to deserialize the JSON body into the target type: icon_config: invalid type: map, expected a string at line 1 column 15"
```

**判定**: ❌ 与文档有偏差

**依据**: 422 Unprocessable Entity："icon_config: invalid type: map, expected a string" — 文档 docs/37 说 icon_config 是"可选的图标配置字符串"但示例 /v1/workspaces 的返回里 icon_config 存储为 JSON-stringified 字符串，更新接口要求调用方发送已 stringified 的字符串，而不是原生 JSON 对象。文档未明确提示这一点。

**备注**: API 强制 icon_config 是一个已转义的 JSON 字符串

---

### #32 `POST /v1/agents/{agent_id}/move (to other ws)` ✅

- **实际 URL**: `https://build.twin.so/v1/agents/019db5a5-56b8-7a40-8b0e-ee557077c754/move`
- **调用时间**: 2026-04-22T14:49:13.760Z
- **HTTP 状态**: `200`

**请求 Headers (脱敏)**
```json
{
  "Accept": "application/json",
  "Content-Type": "application/json",
  "x-api-key": "***REDACTED***"
}
```

**请求 Body**
```json
{
  "workspace_id": "019db592-7eda-7053-8e92-10cc777de4b0"
}
```

**响应 Headers (摘要)**
```json
{
  "content-length": "16",
  "content-type": "application/json",
  "date": "Wed, 22 Apr 2026 14:45:45 GMT"
}
```

**响应 Body (脱敏)**
```json
{
  "success": true
}
```

**判定**: ✅ 与文档一致

**依据**: 200 OK 响应 {success: true}，与 docs/40 一致

**备注**: 将 Agent 移到 API 测试与开发者工具工作区

---

### #33 `POST /v1/agents/{agent_id}/move (back)` ✅

- **实际 URL**: `https://build.twin.so/v1/agents/019db5a5-56b8-7a40-8b0e-ee557077c754/move`
- **调用时间**: 2026-04-22T14:49:13.760Z
- **HTTP 状态**: `200`

**请求 Headers (脱敏)**
```json
{
  "Accept": "application/json",
  "Content-Type": "application/json",
  "x-api-key": "***REDACTED***"
}
```

**请求 Body**
```json
{
  "workspace_id": "019db5a5-3fcd-71e3-80aa-640d6e4253bc"
}
```

**响应 Headers (摘要)**
```json
{
  "content-length": "16",
  "content-type": "application/json",
  "date": "Wed, 22 Apr 2026 14:45:52 GMT"
}
```

**响应 Body (脱敏)**
```json
{
  "success": true
}
```

**判定**: ✅ 与文档一致

**依据**: 200 OK 响应 {success: true}，与 docs/40 一致

**备注**: 把 Agent 移回沙盒工作区

---

### #34 `PUT /v1/workspaces/{workspace_id} (name only)` ✅

- **实际 URL**: `https://build.twin.so/v1/workspaces/019db5a5-3fcd-71e3-80aa-640d6e4253bc`
- **调用时间**: 2026-04-22T14:49:13.760Z
- **HTTP 状态**: `200`

**请求 Headers (脱敏)**
```json
{
  "Accept": "application/json",
  "Content-Type": "application/json",
  "x-api-key": "***REDACTED***"
}
```

**请求 Body**
```json
{
  "name": "twin-rest-api-live-test-sandbox (updated)"
}
```

**响应 Headers (摘要)**
```json
{
  "content-length": "251",
  "content-type": "application/json",
  "date": "Wed, 22 Apr 2026 14:46:05 GMT"
}
```

**响应 Body (脱敏)**
```json
{
  "workspace": {
    "agent_count": 1,
    "created_at": "2026-04-22T14:43:19.117863929+00:00",
    "name": "twin-rest-api-live-test-sandbox (updated)",
    "position": 0,
    "updated_at": "2026-04-22T14:46:05.264027353+00:00",
    "workspace_id": "019db5a5-3fcd-71e3-80aa-640d6e4253bc"
  }
}
```

**判定**: ✅ 与文档一致

**依据**: 200 OK 响应 workspace 对象含更新后的 name 和更新的 updated_at，与 docs/37 一致

**备注**: 不传 icon_config 时正常工作

---

### #35 `POST /v1/workspaces/reorder` ✅

- **实际 URL**: `https://build.twin.so/v1/workspaces/reorder`
- **调用时间**: 2026-04-22T14:49:13.760Z
- **HTTP 状态**: `200`

**请求 Headers (脱敏)**
```json
{
  "Accept": "application/json",
  "Content-Type": "application/json",
  "x-api-key": "***REDACTED***"
}
```

**请求 Body**
```json
{
  "workspace_ids": "(full current order, 49 workspace UUIDs — elided for brevity; see raw_data.json)"
}
```

**响应 Headers (摘要)**
```json
{
  "content-length": "16",
  "content-type": "application/json",
  "date": "Wed, 22 Apr 2026 14:46:29 GMT"
}
```

**响应 Body (脱敏)**
```json
{
  "success": true
}
```

**判定**: ✅ 与文档一致

**依据**: 200 OK 响应 {success: true}，与 docs/39 一致

**备注**: 测试采用 no-op 策略（传入当前顺序），不破坏已有排序

---

## 阶段 10 — Cleanup (6 个 DELETE 端点本身的测试)

### #36 `DELETE /v1/agents/{agent_id}/webhooks/{webhook_id}` ⚠️

- **实际 URL**: `https://build.twin.so/v1/agents/019db5a5-56b8-7a40-8b0e-ee557077c754/webhooks/019db5a5-cfac-71b1-9a8f-2d9320d7f3de`
- **调用时间**: 2026-04-22T14:49:49.450Z
- **HTTP 状态**: `204`

**请求 Headers (脱敏)**
```json
{
  "Accept": "application/json",
  "x-api-key": "***REDACTED***"
}
```

**请求 Body**: _（无）_

**响应 Headers (摘要)**
```json
{
  "content-length": "0",
  "date": "Wed, 22 Apr 2026 14:46:38 GMT"
}
```

**响应 Body (脱敏)**
```json
""
```

**判定**: ⚠️ 与文档有偏差

**依据**: 实测返回 204 No Content（空响应体），但文档 docs/23 未明确规定状态码与响应形态 — 与其它 DELETE 端点行为不一致（有的返回 200 + body），建议文档补充

**备注**: 删除 webhook：204 空响应

---

### #37 `DELETE /v1/agents/{agent_id}/schedule` ✅

- **实际 URL**: `https://build.twin.so/v1/agents/019db5a5-56b8-7a40-8b0e-ee557077c754/schedule`
- **调用时间**: 2026-04-22T14:49:49.450Z
- **HTTP 状态**: `200`

**请求 Headers (脱敏)**
```json
{
  "Accept": "application/json",
  "x-api-key": "***REDACTED***"
}
```

**请求 Body**: _（无）_

**响应 Headers (摘要)**
```json
{
  "content-length": "16",
  "content-type": "application/json",
  "date": "Wed, 22 Apr 2026 14:46:38 GMT"
}
```

**响应 Body (脱敏)**
```json
{
  "success": true
}
```

**判定**: ✅ 与文档一致

**依据**: 200 OK 响应 {success: true}，与 docs/28 一致

**备注**: 删除调度：200 + success=true

---

### #38 `DELETE /v1/agents/{agent_id}/runs/{run_id}` ⚠️

- **实际 URL**: `https://build.twin.so/v1/agents/019db5a5-56b8-7a40-8b0e-ee557077c754/runs/019db5a6-6fa0-7c93-afb7-bb91a6434960`
- **调用时间**: 2026-04-22T14:49:49.450Z
- **HTTP 状态**: `204`

**请求 Headers (脱敏)**
```json
{
  "Accept": "application/json",
  "x-api-key": "***REDACTED***"
}
```

**请求 Body**: _（无）_

**响应 Headers (摘要)**
```json
{
  "content-length": "0",
  "date": "Wed, 22 Apr 2026 14:46:39 GMT"
}
```

**响应 Body (脱敏)**
```json
""
```

**判定**: ⚠️ 与文档有偏差

**依据**: 实测返回 204 No Content，文档 docs/15 未规定状态码 — 与删除 agent、删除 webhook 一致（都是 204），但与删除 workspace (200+body)、删除 schedule (200+body)、撤销 api_key (200+body) 不一致 — 后端对 DELETE 语义存在两种风格

**备注**: 删除 run：204 空响应

---

### #39 `DELETE /api/public/v1/access-api-keys/{key_id}` ✅

- **实际 URL**: `https://build.twin.so/api/public/v1/access-api-keys/f28d7f04b58e24361d2b0dd4e8e96579`
- **调用时间**: 2026-04-22T14:49:49.450Z
- **HTTP 状态**: `200`

**请求 Headers (脱敏)**
```json
{
  "Accept": "application/json",
  "x-api-key": "***REDACTED***"
}
```

**请求 Body**: _（无）_

**响应 Headers (摘要)**
```json
{
  "content-length": "16",
  "content-type": "application/json",
  "date": "Wed, 22 Apr 2026 14:45:08 GMT"
}
```

**响应 Body (脱敏)**
```json
{
  "revoked": true
}
```

**判定**: ✅ 与文档一致

**依据**: 200 OK 响应 {revoked: true}，与 docs/07 一致

**备注**: 撤销 api_key：200 + revoked=true

---

### #40 `DELETE /v1/agents/{agent_id}` ⚠️

- **实际 URL**: `https://build.twin.so/v1/agents/019db5a5-56b8-7a40-8b0e-ee557077c754`
- **调用时间**: 2026-04-22T14:49:49.450Z
- **HTTP 状态**: `204`

**请求 Headers (脱敏)**
```json
{
  "Accept": "application/json",
  "x-api-key": "***REDACTED***"
}
```

**请求 Body**: _（无）_

**响应 Headers (摘要)**
```json
{
  "content-length": "0",
  "date": "Wed, 22 Apr 2026 14:46:43 GMT"
}
```

**响应 Body (脱敏)**
```json
""
```

**判定**: ⚠️ 与文档有偏差

**依据**: 实测返回 204 No Content（空响应体），文档 docs/12 未规定状态码 — 与删除 webhook/删除 run 一致

**备注**: 删除 agent：204 空响应

---

### #41 `DELETE /v1/workspaces/{workspace_id}` ✅

- **实际 URL**: `https://build.twin.so/v1/workspaces/019db5a5-3fcd-71e3-80aa-640d6e4253bc`
- **调用时间**: 2026-04-22T14:49:49.450Z
- **HTTP 状态**: `200`

**请求 Headers (脱敏)**
```json
{
  "Accept": "application/json",
  "x-api-key": "***REDACTED***"
}
```

**请求 Body**: _（无）_

**响应 Headers (摘要)**
```json
{
  "content-length": "16",
  "content-type": "application/json",
  "date": "Wed, 22 Apr 2026 14:46:49 GMT"
}
```

**响应 Body (脱敏)**
```json
{
  "deleted": true
}
```

**判定**: ✅ 与文档一致

**依据**: 200 OK 响应 {deleted: true}，与 docs/38 一致

**备注**: 删除 workspace：200 + deleted=true

---

## 阶段 11 — 负向探测 (已删除资源的 GET 行为)

### #42 `GET /v1/agents/{agent_id} (deleted)` ❌

- **实际 URL**: `https://build.twin.so/v1/agents/019db5a5-56b8-7a40-8b0e-ee557077c754`
- **调用时间**: 2026-04-22T14:49:49.450Z
- **HTTP 状态**: `403`

**请求 Headers (脱敏)**
```json
{
  "Accept": "application/json",
  "x-api-key": "***REDACTED***"
}
```

**请求 Body**: _（无）_

**响应 Headers (摘要)**
```json
{
  "content-length": "103",
  "content-type": "application/problem+json",
  "date": "Wed, 22 Apr 2026 14:46:56 GMT"
}
```

**响应 Body (脱敏)**
```json
{
  "detail": "you do not have access to this agent",
  "status": 403,
  "title": "Forbidden",
  "type": "about:blank"
}
```

**判定**: ❌ 与文档有偏差

**依据**: 预期返回 404 Not Found，实际返回 403 Forbidden + "you do not have access to this agent" — 后端把「不存在」与「无权限」统一返回 403。文档 docs/11 及 docs/04 未说明这点

**备注**: 已删除资源被当作无权限处理，可能是安全设计但会让调用方难以区分 "404 不存在" 与 "403 无权限"

---

