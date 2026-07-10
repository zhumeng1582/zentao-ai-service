# zentao-ai-service 接口文档

版本：v1.2

日期：2026-07-10

状态：接口设计基线

基础路径：`/api/v1`
适用范围：禅道扩展、zentao-ai-service、后台 Worker、管理端

## 1. 文档目标

本文档定义禅道与独立 AI 服务之间的接口契约，包括：

- 服务间鉴权和用户身份传递。
- 禅道对象同步、索引、任务和失败重试。
- Bug 相似检索和 AI 分析。
- 文档摘要、单文档问答和项目知识问答。
- 多轮聊天、会话管理和历史消息。
- Bug、文档、需求等对象的相似内容推荐。
- 客户反馈分类和草稿生成。
- 项目风险周报。
- AI 结果确认、驳回和写回。
- 模型配置、审计和用量查询。
- AI 服务回调禅道进行批量权限校验的契约。

本文档是前后端联调和服务实现的唯一接口基线。字段变更应通过新版本或兼容性扩展完成，不能直接破坏已发布字段。

## 2. 总体原则

1. 浏览器不直接访问 AI 服务，所有业务请求先进入禅道扩展控制器。
2. 禅道是用户、功能权限和对象权限的唯一权威来源。
3. AI 服务必须验证服务签名，不能信任请求正文中的用户名或权限列表。
4. 所有检索必须按 `tenant_id` 隔离，并在内容进入 LLM 前完成对象级权限校验。
5. 长耗时操作返回任务 ID，由 Worker 异步执行。
6. AI 结果默认不直接修改禅道业务对象，写回前必须由有权限的用户确认。
7. 所有 AI 结论和回答必须能追溯到来源对象。
8. 权限不明确、鉴权服务不可用或 ACL 版本落后时，默认拒绝访问。

## 3. 接口范围与发布阶段

| 模块 | 接口 | 阶段 |
| --- | --- | --- |
| 系统 | 健康检查、就绪检查 | MVP |
| 同步 | 单对象同步、批量同步 | MVP |
| 索引 | 状态、重试、重建 | MVP |
| 任务 | 查询异步任务 | MVP |
| Bug | 相似 Bug、Bug AI 分析 | MVP |
| 文档 | 摘要、单文档问答 | MVP |
| 项目知识问答 | 项目范围跨对象问答 | V1.1 |
| 全局 AI 客服 | `help`、Bug、文档范围会话、消息、流式回答、历史消息 | MVP |
| 推荐 | 文档、需求和跨类型内容通用推荐 | V1.1 |
| AI 结果 | 查询、采纳、驳回、添加评论写回 | MVP |
| 反馈 | 分类、生成 Bug/需求草稿 | V1.1 |
| 项目 | 风险周报 | V1.1 |
| 模型管理 | 查询、配置、连通性测试 | V1.1 |
| 审计与用量 | 调用日志、用量统计 | V1.1 |

### 3.1 接口总览

| Method | Path | Scope | 阶段 |
| --- | --- | --- | --- |
| GET | `/healthz` | 公开/内网 | MVP |
| GET | `/readyz` | 探针或服务鉴权 | MVP |
| POST | `/api/v1/sync/zentao/object` | `sync:object` | MVP |
| POST | `/api/v1/sync/zentao/objects/batch` | `sync:object` | MVP |
| GET | `/api/v1/index/status` | `index:read` | MVP |
| POST | `/api/v1/index/retry` | `index:retry` | MVP |
| POST | `/api/v1/index/rebuild` | `index:rebuild` | MVP |
| GET | `/api/v1/tasks/{task_id}` | 对象读取权限 | MVP |
| POST | `/api/v1/tasks/batch-get` | 对象读取权限 | MVP |
| POST | `/api/v1/bugs/{bug_id}/similar` | `bug:similar` | MVP |
| POST | `/api/v1/bugs/{bug_id}/analyze` | `bug:analyze` | MVP |
| POST | `/api/v1/docs/{doc_id}/summary` | `doc:summary` | MVP |
| POST | `/api/v1/docs/{doc_id}/ask` | `doc:ask` | MVP |
| POST | `/api/v1/projects/{project_id}/ask` | `project:ask` | V1.1 |
| POST | `/api/v1/conversations` | `chat:create` | MVP |
| GET | `/api/v1/conversations` | 会话所有者 | MVP |
| GET | `/api/v1/conversations/{conversation_id}` | 会话所有者 | MVP |
| PATCH | `/api/v1/conversations/{conversation_id}` | 会话所有者 | MVP |
| DELETE | `/api/v1/conversations/{conversation_id}` | 会话所有者 | MVP |
| POST | `/api/v1/conversations/{conversation_id}/messages` | `chat:send` | MVP |
| GET | `/api/v1/conversations/{conversation_id}/messages` | 会话所有者 | MVP |
| POST | `/api/v1/conversations/{conversation_id}/messages/{message_id}/feedback` | 会话所有者 | MVP |
| POST | `/api/v1/recommendations/similar` | `recommendation:read` | V1.1 |
| POST | `/api/v1/feedback/{feedback_id}/classify` | `feedback:classify` | V1.1 |
| POST | `/api/v1/projects/{project_id}/risk-reports` | `project:risk-report:create` | V1.1 |
| GET | `/api/v1/ai-results` | 源对象读取权限 | MVP |
| GET | `/api/v1/ai-results/{result_id}` | 源对象读取权限 | MVP |
| POST | `/api/v1/ai-results/{result_id}/accept` | 确认权限 | MVP |
| POST | `/api/v1/ai-results/{result_id}/reject` | 确认权限 | MVP |
| POST | `/api/v1/ai-results/{result_id}/writeback` | `ai-result:writeback` | MVP（仅添加评论） |
| GET | `/api/v1/admin/model-configs` | 管理员 | V1.1 |
| POST | `/api/v1/admin/model-configs` | 管理员 | V1.1 |
| PATCH | `/api/v1/admin/model-configs/{config_id}` | 管理员 | V1.1 |
| POST | `/api/v1/admin/model-configs/{config_id}/test` | 管理员 | V1.1 |
| GET | `/api/v1/admin/audit/llm-calls` | 管理员 | V1.1 |
| GET | `/api/v1/admin/usage` | 管理员 | V1.1 |

AI 服务依赖的禅道内部接口列在第 18 章，不属于上述 `/api/v1` 对外路径。

## 4. 通用约定

### 4.1 协议

- 生产环境必须使用 HTTPS。
- 请求和响应编码为 UTF-8。
- JSON 接口使用 `Content-Type: application/json`。
- 时间使用 ISO 8601，包含时区，例如 `2026-07-10T10:30:00+08:00`。
- ID 在 JSON 中统一使用字符串，避免不同语言的整数精度问题。

### 4.2 通用请求头

除明确标记为公开的接口外，请求必须携带：

```http
X-Client-Id: zentao_company_001
X-Tenant-Id: company_001
X-Actor-Type: user
X-Zentao-User: pm01
X-Timestamp: 1783650600
X-Nonce: 5f7e3198-0a92-4dee-96fa-c12bb34fd509
X-Request-Id: req_01JZABCDEF
X-Body-SHA256: 51e7f...
X-Signature: sha256=8d80a...
```

修改类请求还必须携带：

```http
X-Idempotency-Key: idem_01JZABCDEF
```

字段说明：

| Header | 必填 | 说明 |
| --- | --- | --- |
| `X-Client-Id` | 是 | AI 服务分配给禅道实例的客户端 ID |
| `X-Tenant-Id` | 是 | 租户 ID，必须与 Client 绑定关系一致 |
| `X-Actor-Type` | 是 | `user` 或 `service` |
| `X-Zentao-User` | 用户请求必填 | 禅道账号；系统任务使用 `system:sync` 等受限账号 |
| `X-Timestamp` | 是 | Unix 秒时间戳 |
| `X-Nonce` | 是 | 每次请求唯一的 UUID |
| `X-Request-Id` | 是 | 调用链请求 ID |
| `X-Body-SHA256` | 是 | 原始 HTTP Body 的 SHA-256 十六进制摘要；空 Body 也要计算 |
| `X-Signature` | 是 | HMAC-SHA256 签名 |
| `X-Idempotency-Key` | 修改类接口必填 | 防止客户端重试产生重复业务操作 |

### 4.3 HMAC 签名

客户端与 AI 服务共享 `client_secret`。签名原文为：

```text
{HTTP_METHOD} + "\\n" +
{NORMALIZED_PATH} + "\\n" +
{NORMALIZED_QUERY_STRING} + "\\n" +
{TENANT_ID} + "\\n" +
{CLIENT_ID} + "\\n" +
{ACTOR_TYPE} + "\\n" +
{ZENTAO_USER} + "\\n" +
{TIMESTAMP} + "\\n" +
{NONCE} + "\\n" +
{BODY_SHA256}
```

示例：

```text
POST
/api/v1/docs/88/ask

company_001
zentao_company_001
user
pm01
1783650600
5f7e3198-0a92-4dee-96fa-c12bb34fd509
51e7f...
```

计算方式：

```text
signature = hex(HMAC-SHA256(client_secret, canonical_request))
X-Signature = "sha256=" + signature
```

服务端验证要求：

- 时间偏差不得超过 300 秒。
- 使用 Redis 对 `client_id + nonce` 执行 `SET NX EX 300`，拒绝重放。
- 使用常量时间算法比较签名。
- `tenant_id`、用户、路径、Query 和 Body 摘要必须全部参与签名。
- 支持当前密钥和上一把密钥并行验证，以便无停机轮换。

### 4.4 服务账号

无具体用户的同步任务使用：

```http
X-Actor-Type: service
X-Zentao-User: system:sync
```

服务账号必须配置最小 Scope，例如：

```text
sync:object
sync:attachment
index:read
```

服务账号默认不能执行 AI 结果采纳、写回、模型配置和索引重建。

### 4.5 通用成功响应

所有 JSON 接口的顶层结构固定为 `code`、`msg`、`data`，不得在顶层增加业务字段。

```json
{
  "code": 0,
  "msg": "success",
  "data": {}
}
```

字段定义：

| 字段 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `code` | integer | 是 | `0` 表示成功，非 `0` 表示失败 |
| `msg` | string | 是 | 面向调用方的简短结果说明，不承载业务数据 |
| `data` | object、array 或 null | 是 | 实际业务内容；无返回内容或失败且无详情时为 `null` |

`request_id` 通过响应 Header 返回，不放在 JSON 顶层：

```http
X-Request-Id: req_01JZABCDEF
```

### 4.6 通用错误响应

```json
{
  "code": 40302,
  "msg": "无权访问该对象",
  "data": {
    "error_code": "PERMISSION_DENIED",
    "details": {}
  }
}
```

错误时：

- `code` 是稳定的数字业务码，客户端优先按它处理。
- `msg` 用于展示简短错误说明，不能依赖其文本进行程序判断。
- `data.error_code` 是便于日志和排查的稳定英文标识。
- `data.details` 只保存安全的字段校验结果或冲突信息。
- 生产环境的 `details` 不返回堆栈、SQL、模型密钥、签名原文或内部地址。

### 4.7 HTTP 状态码

| 状态码 | 使用场景 |
| --- | --- |
| `200` | 查询成功或同步处理成功 |
| `201` | 会话、配置等资源创建成功 |
| `202` | 已接受异步任务 |
| `400` | 请求字段或业务参数错误 |
| `401` | 签名无效、请求过期、Nonce 重放、Client 无效 |
| `403` | 调用方合法，但没有功能或对象权限 |
| `404` | 对象不存在；对无权知道其存在的对象也可返回 404 |
| `409` | 版本冲突、状态冲突或幂等键冲突 |
| `413` | 请求体或附件超过限制 |
| `422` | JSON Schema 或字段语义校验失败 |
| `429` | 超过租户、用户或模型调用限额 |
| `502` | 模型、Qdrant、禅道回调等上游服务异常 |
| `503` | 服务未就绪或关键依赖不可用 |

### 4.8 分页

列表接口统一使用游标分页：

```http
GET /api/v1/ai-results?limit=20&cursor=eyJpZCI6...
```

响应：

```json
{
  "code": 0,
  "msg": "success",
  "data": {
    "items": [],
    "next_cursor": "eyJpZCI6...",
    "has_more": false
  }
}
```

`limit` 默认 20，最大 100。

### 4.9 幂等

以下接口必须支持 `X-Idempotency-Key`：

- 对象同步和批量同步。
- 创建分析任务。
- 索引重试和重建。
- AI 结果采纳、驳回和写回。
- 模型配置修改。

同一租户、Client、接口和幂等键在 24 小时内：

- 请求摘要相同：返回第一次请求的结果。
- 请求摘要不同：返回 `409 IDEMPOTENCY_KEY_CONFLICT`。

## 5. 权限处理

### 5.1 权限边界

权限分为：

1. 服务身份：请求是否来自可信禅道实例。
2. 功能权限：用户是否能使用 Bug AI、文档问答、索引重试等功能。
3. 对象权限：用户是否能读取具体 Bug、文档、项目或反馈。
4. 动作权限：用户是否能采纳、修改字段或写回禅道。

### 5.2 检索权限流程

```text
验证 HMAC 和用户身份
  ↓
检查功能 Scope
  ↓
Qdrant 按 tenant_id、deleted、object_type 粗过滤
  ↓
AI 服务调用禅道批量权限校验接口
  ↓
删除所有未授权候选
  ↓
仅将授权内容发送给 LLM
  ↓
返回答案前再次核对引用对象位于授权集合
```

请求正文中的 `permission_scope` 不作为权限依据。

### 5.3 权限失败策略

- 禅道权限接口超时：返回失败，不降级为放行。
- 对象 ACL 版本不一致：重新校验或拒绝。
- 候选内容未完成权限校验：不得进入 Prompt。
- 用户失去源对象权限后，不得继续查看已有 AI 结果和历史引用。

## 6. 数据类型

本章定义的业务对象均放在响应 `data` 中。它们不是独立的顶层响应结构。

### 6.1 SourceObject

```json
{
  "source": "zentao",
  "object_type": "bug",
  "object_id": "123",
  "event_id": "action:90001",
  "event": "created",
  "source_revision": "90001",
  "occurred_at": "2026-07-10T10:30:00+08:00",
  "payload": {}
}
```

字段说明：

| 字段 | 必填 | 说明 |
| --- | --- | --- |
| `source` | 是 | 固定为 `zentao` |
| `object_type` | 是 | `bug`、`doc`、`feedback`、`story`、`task`、`case`、`project` |
| `object_id` | 是 | 禅道对象 ID |
| `event_id` | 是 | 禅道侧唯一事件 ID，用于事件幂等 |
| `event` | 是 | `created`、`updated`、`deleted`、`closed`、`restored`、`acl_changed` |
| `source_revision` | 是 | 单调递增版本，优先使用操作记录 ID，不使用普通时间字符串代替 |
| `occurred_at` | 是 | 事件发生时间 |
| `payload` | 非删除事件必填 | 对象快照 |

### 6.2 TaskSummary

```json
{
  "task_id": "01JZABCDEF123",
  "task_type": "bug_analysis",
  "status": "queued",
  "progress": 0,
  "created_at": "2026-07-10T10:30:00+08:00"
}
```

任务状态：

```text
queued -> running -> success
                  -> failed
                  -> superseded
```

`superseded` 表示对象出现了更新版本，旧任务不再允许写入索引或结果。

### 6.3 SourceReference

```json
{
  "object_type": "doc",
  "object_id": "88",
  "chunk_id": "doc:88:chunk:3",
  "title": "登录模块 PRD",
  "heading_path": "异常流程 > Token 过期",
  "source_url": "/doc-view-88.html",
  "snippet": "Token 过期后系统应先尝试刷新……"
}
```

## 7. 系统接口

### 7.1 存活检查

```http
GET /healthz
```

鉴权：可免鉴权，但只能通过内网或负载均衡器访问。

响应：

```json
{
  "code": 0,
  "msg": "success",
  "data": {
    "status": "ok",
    "service": "zentao-ai-service",
    "version": "1.0.0"
  }
}
```

该接口只证明进程存活，不检查外部依赖。

### 7.2 就绪检查

```http
GET /readyz
```

鉴权：需要服务鉴权，或仅允许集群探针访问。

响应：

```json
{
  "code": 0,
  "msg": "success",
  "data": {
    "status": "ready",
    "dependencies": {
      "postgres": "ok",
      "redis": "ok",
      "qdrant": "ok"
    }
  }
}
```

任一必需依赖不可用时返回 `503`。

## 8. 对象同步接口

### 8.1 同步单个对象

```http
POST /api/v1/sync/zentao/object
```

鉴权 Scope：`sync:object`。

Bug 请求：

```json
{
  "source": "zentao",
  "object_type": "bug",
  "object_id": "123",
  "event_id": "action:90001",
  "event": "updated",
  "source_revision": "90001",
  "occurred_at": "2026-07-10T10:30:00+08:00",
  "payload": {
    "title": "登录后无法进入首页",
    "steps": "复现步骤……",
    "severity": "3",
    "priority": "1",
    "status": "active",
    "product_id": "3",
    "project_id": "8",
    "execution_id": "21",
    "module_id": "12",
    "opened_by": "tester01",
    "assigned_to": "dev01",
    "comments": [],
    "files": [
      {
        "id": "889",
        "name": "error.png",
        "extension": "png",
        "size": 284102,
        "mime_type": "image/png",
        "url": "/file-read-889.html"
      }
    ],
    "source_url": "/bug-view-123.html"
  }
}
```

文档请求：

```json
{
  "source": "zentao",
  "object_type": "doc",
  "object_id": "88",
  "event_id": "action:90002",
  "event": "updated",
  "source_revision": "90002",
  "occurred_at": "2026-07-10T10:31:00+08:00",
  "payload": {
    "title": "登录模块 PRD",
    "content_type": "html",
    "content": "<h1>登录模块</h1><p>……</p>",
    "lib_id": "10",
    "project_id": "8",
    "product_id": "3",
    "execution_id": "0",
    "status": "normal",
    "added_by": "pm01",
    "edited_by": "pm01",
    "files": [],
    "source_url": "/doc-view-88.html",
    "acl_version": "27"
  }
}
```

删除事件：

```json
{
  "source": "zentao",
  "object_type": "doc",
  "object_id": "88",
  "event_id": "action:90003",
  "event": "deleted",
  "source_revision": "90003",
  "occurred_at": "2026-07-10T10:32:00+08:00",
  "payload": null
}
```

响应：`202 Accepted`

```json
{
  "code": 0,
  "msg": "success",
  "data": {
    "accepted": true,
    "duplicate": false,
    "task": {
      "task_id": "01JZTASK001",
      "task_type": "object_index",
      "status": "queued",
      "progress": 0,
      "created_at": "2026-07-10T10:30:01+08:00"
    }
  }
}
```

处理规则：

- `tenant_id + event_id` 唯一。
- 小于当前 `source_revision` 的事件返回成功，但任务状态为 `superseded`。
- AI 服务只能从配置的禅道主机下载附件，禁止访问请求中任意外部 URL。
- ACL 无法确认时可以保存快照，但不得使对象进入可检索状态。

### 8.2 批量同步对象

```http
POST /api/v1/sync/zentao/objects/batch
```

鉴权 Scope：`sync:object`。

限制：每批最多 100 个对象；压缩后请求体默认不超过 10 MB。

请求：

```json
{
  "objects": [
    {
      "source": "zentao",
      "object_type": "bug",
      "object_id": "123",
      "event_id": "action:90001",
      "event": "updated",
      "source_revision": "90001",
      "occurred_at": "2026-07-10T10:30:00+08:00",
      "payload": {}
    }
  ]
}
```

响应：

```json
{
  "code": 0,
  "msg": "success",
  "data": {
    "accepted_count": 1,
    "rejected_count": 0,
    "items": [
      {
        "object_type": "bug",
        "object_id": "123",
        "event_id": "action:90001",
        "accepted": true,
        "task_id": "01JZTASK001",
        "error": null
      }
    ]
  }
}
```

批量接口允许部分成功，每个对象必须返回独立处理结果。

## 9. 索引接口

### 9.1 查询对象索引状态

```http
GET /api/v1/index/status?object_type=doc&object_id=88
```

鉴权 Scope：`index:read`，并要求当前用户拥有源对象读取权限。

响应：

```json
{
  "code": 0,
  "msg": "success",
  "data": {
    "object_type": "doc",
    "object_id": "88",
    "source_revision": "90002",
    "sync_status": "synced",
    "parse_status": "success",
    "index_status": "indexed",
    "active_revision": "90002",
    "chunk_count": 12,
    "image_chunk_count": 2,
    "embedding_model": "bge-m3",
    "embedding_fingerprint": "bge-m3:1024:normalize-v1:chunker-v1",
    "last_indexed_at": "2026-07-10T10:31:30+08:00",
    "error": null
  }
}
```

状态值：

- `sync_status`：`pending`、`synced`、`failed`。
- `parse_status`：`pending`、`running`、`success`、`partial`、`failed`。
- `index_status`：`pending`、`indexing`、`indexed`、`failed`、`deleted`、`permission_unknown`。

### 9.2 重试失败索引

```http
POST /api/v1/index/retry
```

鉴权 Scope：`index:retry`；普通用户只能重试自己有权访问的单个对象，管理员可批量重试。

请求：

```json
{
  "object_type": "doc",
  "object_id": "88",
  "failed_task_id": "01JZTASK001"
}
```

响应：`202 Accepted`，返回新的 `TaskSummary`。

### 9.3 重建对象索引

```http
POST /api/v1/index/rebuild
```

鉴权 Scope：`index:rebuild`，默认仅管理员。

请求：

```json
{
  "object_type": "doc",
  "object_ids": ["88", "89"],
  "reason": "chunker_version_changed",
  "force_embedding": true
}
```

限制：单次最多 100 个对象。全租户重建必须使用独立运维任务，不通过本接口直接触发。

## 10. 任务接口

### 10.1 查询任务

```http
GET /api/v1/tasks/{task_id}
```

用户必须拥有任务关联对象的读取权限。管理员任务仅管理员可见。

响应：

```json
{
  "code": 0,
  "msg": "success",
  "data": {
    "task_id": "01JZTASK001",
    "task_type": "object_index",
    "status": "running",
    "progress": 65,
    "stage": "embedding",
    "object_type": "doc",
    "object_id": "88",
    "source_revision": "90002",
    "result_id": null,
    "retry_count": 1,
    "created_at": "2026-07-10T10:30:01+08:00",
    "started_at": "2026-07-10T10:30:03+08:00",
    "finished_at": null,
    "error": null
  }
}
```

### 10.2 批量查询任务

```http
POST /api/v1/tasks/batch-get
```

请求：

```json
{
  "task_ids": ["01JZTASK001", "01JZTASK002"]
}
```

每次最多查询 100 个任务。无权查看的任务不返回详情。

## 11. Bug 接口

### 11.1 相似 Bug 检索

```http
POST /api/v1/bugs/{bug_id}/similar
```

鉴权 Scope：`bug:similar`。需要当前 Bug 和所有候选 Bug 的读取权限。

请求：

```json
{
  "top_k": 10,
  "product_id": "3",
  "project_id": "8",
  "include_closed": true,
  "search_mode": "hybrid",
  "rerank": true
}
```

字段规则：

- `top_k` 默认 10，最大 50。
- `product_id`、`project_id` 只能缩小范围，不能扩大用户权限。
- `search_mode` 支持 `dense`、`sparse`、`hybrid`，默认 `hybrid`。
- 当前 Bug 本身必须从结果中排除。

响应：

```json
{
  "code": 0,
  "msg": "success",
  "data": {
    "query_bug_id": "123",
    "items": [
      {
        "object_id": "98",
        "title": "登录后页面空白",
        "score": 0.91,
        "vector_score": 0.89,
        "rerank_score": 0.91,
        "status": "active",
        "priority": "1",
        "severity": "3",
        "source_url": "/bug-view-98.html",
        "evidence": "复现步骤和 403 错误码相似"
      }
    ],
    "search_mode": "hybrid"
  }
}
```

### 11.2 Bug AI 分析

```http
POST /api/v1/bugs/{bug_id}/analyze
```

鉴权 Scope：`bug:analyze`。

请求：

```json
{
  "analysis_types": [
    "quality_check",
    "priority_suggestion",
    "risk_assessment",
    "similar_bugs"
  ],
  "include_comments": true,
  "include_images": true,
  "max_similar_bugs": 5,
  "mode": "async"
}
```

`mode`：

- `async`：推荐，返回 `202` 和任务 ID。
- `sync`：服务可在配置的同步超时内直接返回；超时自动转异步并返回 `202`。

异步响应：

```json
{
  "code": 0,
  "msg": "success",
  "data": {
    "task": {
      "task_id": "01JZTASK100",
      "task_type": "bug_analysis",
      "status": "queued",
      "progress": 0,
      "created_at": "2026-07-10T10:40:00+08:00"
    }
  }
}
```

分析完成后的响应：

```json
{
  "code": 0,
  "msg": "success",
  "data": {
    "result_id": "01JZRESULT100",
    "object_type": "bug",
    "object_id": "123",
    "analysis_type": "bug_triage",
    "summary": "登录成功后权限回调失败，导致首页请求返回 403。",
    "suggested_priority": "P1",
    "suggested_severity": "3",
    "risk_level": "high",
    "suggested_owner": "dev01",
    "reason": "问题阻断登录后的主流程，并已有相似线上问题。",
    "quality_issues": [
      {
        "field": "steps",
        "code": "MISSING_ENVIRONMENT",
        "message": "缺少浏览器和部署环境信息"
      }
    ],
    "similar_bugs": ["98"],
    "evidence": [],
    "confidence": 0.84,
    "insufficient_evidence": false,
    "status": "pending_confirmation"
  }
}
```

模型输出必须经过服务端 JSON Schema 校验；模型自己给出的 `confidence` 只能作为提示，不能当作权限或自动执行依据。

## 12. 文档接口

### 12.1 文档摘要

```http
POST /api/v1/docs/{doc_id}/summary
```

鉴权 Scope：`doc:summary`，需要文档库和文档读取权限。

请求：

```json
{
  "summary_type": "executive",
  "max_length": 800,
  "include_images": false,
  "mode": "async"
}
```

`summary_type` 支持：`brief`、`executive`、`outline`、`action_items`。

完成后的响应：

```json
{
  "code": 0,
  "msg": "success",
  "data": {
    "result_id": "01JZRESULT200",
    "object_type": "doc",
    "object_id": "88",
    "analysis_type": "doc_summary",
    "summary": "本文档定义登录模块的正常流程和异常处理……",
    "key_points": [
      "Token 过期后优先尝试刷新",
      "刷新失败后跳转登录页"
    ],
    "sources": [],
    "status": "generated"
  }
}
```

### 12.2 单文档问答

```http
POST /api/v1/docs/{doc_id}/ask
```

鉴权 Scope：`doc:ask`。

请求：

```json
{
  "question": "Token 过期后系统怎么处理？",
  "conversation_id": null,
  "top_k": 8,
  "include_images": true,
  "response_mode": "sync"
}
```

规则：

- 强制过滤 `object_type=doc AND object_id={doc_id}`。
- `conversation_id` 只用于对话上下文，不能扩大检索范围。
- 每次追问都必须重新鉴权。
- 问题和历史消息总长度超过限制时返回 `422 CONTEXT_TOO_LARGE`。

响应：

```json
{
  "code": 0,
  "msg": "success",
  "data": {
    "answer_id": "01JZANSWER001",
    "conversation_id": "01JZCONV001",
    "answer": "根据文档，系统会先尝试刷新 Token；刷新失败后跳转登录页。",
    "sources": [
      {
        "object_type": "doc",
        "object_id": "88",
        "chunk_id": "doc:88:chunk:3",
        "title": "登录模块 PRD",
        "heading_path": "异常流程 > Token 过期",
        "source_url": "/doc-view-88.html",
        "snippet": "当用户 Token 过期时，系统应尝试刷新 Token……"
      }
    ],
    "insufficient_evidence": false,
    "usage": {
      "input_tokens": 1250,
      "output_tokens": 96
    }
  }
}
```

没有足够证据时：

```json
{
  "code": 0,
  "msg": "success",
  "data": {
    "answer": "未在当前文档中找到足够依据。",
    "sources": [],
    "insufficient_evidence": true
  }
}
```

### 12.3 项目知识问答

```http
POST /api/v1/projects/{project_id}/ask
```

鉴权 Scope：`project:ask`，阶段：V1.1。

请求：

```json
{
  "question": "本版本登录模块有哪些已知风险？",
  "object_types": ["doc", "bug", "story", "task", "case"],
  "conversation_id": null,
  "top_k": 12,
  "response_mode": "sync"
}
```

项目 ID 只限定检索范围。每一个返回的来源对象仍须独立完成权限校验。

## 13. 客户反馈接口

### 13.1 反馈分类

```http
POST /api/v1/feedback/{feedback_id}/classify
```

鉴权 Scope：`feedback:classify`，阶段：V1.1。

请求：

```json
{
  "include_similar_feedback": true,
  "include_related_bugs": true,
  "generate_draft": true,
  "mode": "async"
}
```

完成后的响应：

```json
{
  "code": 0,
  "msg": "success",
  "data": {
    "result_id": "01JZRESULT300",
    "object_type": "feedback",
    "object_id": "56",
    "analysis_type": "feedback_classify",
    "category": "bug",
    "summary": "客户登录后无法进入首页",
    "impact_scope": "single_customer",
    "suggested_priority": "P1",
    "suggested_action": "create_bug",
    "similar_feedback": [],
    "related_objects": [],
    "draft": {
      "target_type": "bug",
      "title": "登录后无法进入首页",
      "description": "……"
    },
    "confidence": 0.81,
    "status": "pending_confirmation"
  }
}
```

AI 只能生成草稿，不能通过该接口直接创建正式 Bug 或需求。

## 14. 项目风险报告接口

### 14.1 生成项目风险周报

```http
POST /api/v1/projects/{project_id}/risk-reports
```

鉴权 Scope：`project:risk-report:create`，阶段：V1.1。

请求：

```json
{
  "period_start": "2026-07-06",
  "period_end": "2026-07-12",
  "include_feedback": true,
  "include_documents": true,
  "mode": "async"
}
```

完成后的响应：

```json
{
  "code": 0,
  "msg": "success",
  "data": {
    "result_id": "01JZRESULT400",
    "project_id": "8",
    "analysis_type": "project_risk_weekly",
    "period_start": "2026-07-06",
    "period_end": "2026-07-12",
    "summary": "本周项目总体风险中等。",
    "progress": [],
    "risks": [],
    "quality": [],
    "customer_feedback": [],
    "suggestions": [],
    "evidence": [],
    "status": "pending_confirmation"
  }
}
```

报告发布前必须由项目经理或管理员确认。

## 15. AI 结果接口

### 15.1 查询 AI 结果列表

```http
GET /api/v1/ai-results?object_type=bug&object_id=123&status=pending_confirmation&limit=20
```

AI 服务必须逐条确认当前用户仍拥有源对象读取权限。

### 15.2 查询 AI 结果详情

```http
GET /api/v1/ai-results/{result_id}
```

响应包括：

```json
{
  "code": 0,
  "msg": "success",
  "data": {
    "result_id": "01JZRESULT100",
    "object_type": "bug",
    "object_id": "123",
    "analysis_type": "bug_triage",
    "prompt_code": "bug_triage",
    "prompt_version": "1.0.0",
    "model": "default_chat",
    "result": {},
    "evidence": [],
    "confidence": 0.84,
    "status": "pending_confirmation",
    "created_by": "tester01",
    "created_at": "2026-07-10T10:40:10+08:00",
    "confirmed_by": null,
    "confirmed_at": null
  }
}
```

### 15.3 采纳 AI 结果

```http
POST /api/v1/ai-results/{result_id}/accept
```

请求：

```json
{
  "accepted_fields": ["suggested_priority", "risk_level"],
  "edited_result": {
    "suggested_priority": "P2"
  },
  "comment": "优先级调整后采纳",
  "expected_source_revision": "90001"
}
```

规则：

- 当前用户必须拥有源对象读取权限。
- 如果后续要写回字段，还必须拥有对应编辑权限。
- `expected_source_revision` 与禅道当前版本不一致时返回 `409 SOURCE_VERSION_CONFLICT`。
- 必须同时保存 AI 原始结果和用户修改后的最终结果。

响应：

```json
{
  "code": 0,
  "msg": "success",
  "data": {
    "result_id": "01JZRESULT100",
    "status": "accepted",
    "confirmed_by": "tester01",
    "confirmed_at": "2026-07-10T10:45:00+08:00"
  }
}
```

### 15.4 驳回 AI 结果

```http
POST /api/v1/ai-results/{result_id}/reject
```

请求：

```json
{
  "reason_code": "INCORRECT_SUGGESTION",
  "comment": "相似 Bug 与当前环境无关"
}
```

`reason_code`：

- `INCORRECT_CLASSIFICATION`
- `INCORRECT_SUGGESTION`
- `INCORRECT_SIMILARITY`
- `INCORRECT_CITATION`
- `INCOMPLETE`
- `NOT_ACTIONABLE`
- `OTHER`

### 15.5 写回禅道

```http
POST /api/v1/ai-results/{result_id}/writeback
```

鉴权 Scope：`ai-result:writeback`。

请求：

```json
{
  "actions": [
    {
      "type": "add_comment",
      "content": "AI 分析：该问题可能与权限回调失败有关。"
    }
  ],
  "expected_source_revision": "90001"
}
```

处理规则：

- 首批 MVP 的 `actions` 只允许 `add_comment`，字段修改、创建对象和发布报告进入 V1.1 或后续版本。
- 评论内容必须经过用户确认，并记录 AI 原始结果与用户最终提交内容。
- 只有 `accepted` 状态的结果允许写回。
- AI 服务不能直接写禅道数据库，必须调用禅道受保护的写回接口。
- 禅道在执行每个动作前重新检查当前用户权限和对象版本。
- 部分动作失败时返回每个动作的独立结果，整体状态为 `write_back_partial`。

## 16. 模型配置接口

以下接口默认仅管理员可用，阶段：V1.1。

### 16.1 查询模型配置

```http
GET /api/v1/admin/model-configs
```

响应不得返回 API Key 明文：

```json
{
  "code": 0,
  "msg": "success",
  "data": {
    "items": [
      {
        "id": "modelcfg_001",
        "provider": "openai_compatible",
        "base_url": "https://model.example.com/v1",
        "chat_model": "deepseek-chat",
        "embedding_model": "bge-m3",
        "vision_model": null,
        "has_api_key": true,
        "enabled": true,
        "public_model_allowed": true,
        "timeout_seconds": 60,
        "max_retries": 2
      }
    ]
  }
}
```

### 16.2 创建模型配置

```http
POST /api/v1/admin/model-configs
```

```json
{
  "provider": "openai_compatible",
  "base_url": "https://model.example.com/v1",
  "api_key": "secret",
  "chat_model": "deepseek-chat",
  "embedding_model": "bge-m3",
  "embedding_dimension": 1024,
  "timeout_seconds": 60,
  "max_retries": 2,
  "rate_limit_per_minute": 120,
  "public_model_allowed": true,
  "enabled": true
}
```

API Key 必须加密存储，响应和日志中不得回显。

### 16.3 修改模型配置

```http
PATCH /api/v1/admin/model-configs/{config_id}
```

未传 `api_key` 表示保留原密钥；传空字符串不代表删除，删除密钥使用独立管理动作。

### 16.4 测试模型配置

```http
POST /api/v1/admin/model-configs/{config_id}/test
```

请求：

```json
{
  "test_types": ["chat", "embedding"]
}
```

响应只返回连通性、耗时和模型信息，不返回完整模型输出或密钥。

## 17. 审计与用量接口

### 17.1 查询 AI 调用日志

```http
GET /api/v1/admin/audit/llm-calls?call_type=chat&success=false&limit=20
```

默认仅管理员可用。日志中的输入输出按租户配置保存摘要或脱敏内容。

### 17.2 查询用量统计

```http
GET /api/v1/admin/usage?from=2026-07-01&to=2026-07-31&group_by=model
```

响应：

```json
{
  "code": 0,
  "msg": "success",
  "data": {
    "period": {
      "from": "2026-07-01",
      "to": "2026-07-31"
    },
    "totals": {
      "requests": 1200,
      "input_tokens": 2500000,
      "output_tokens": 320000,
      "estimated_cost": 120.5,
      "currency": "CNY"
    },
    "groups": []
  }
}
```

费用为估算值，不能替代供应商账单。

## 18. AI 服务调用禅道的内部接口

本节接口由禅道扩展提供，不属于 AI 服务对外 API，但属于完整调用契约。

### 18.1 批量权限校验

```http
POST /api/ai/authorize/batch
```

AI 服务使用独立回调 Client 对请求签名。用户身份必须与原 AI 请求一致。

请求：

```json
{
  "actor": {
    "type": "user",
    "account": "pm01"
  },
  "action": "read",
  "resources": [
    {"type": "bug", "id": "98"},
    {"type": "doc", "id": "88"},
    {"type": "doc", "id": "91"}
  ],
  "expected_authz_version": "106"
}
```

响应：

```json
{
  "code": 0,
  "msg": "success",
  "data": {
    "authz_version": "106",
    "decisions": [
      {"type": "bug", "id": "98", "allowed": true, "reason": null},
      {"type": "doc", "id": "88", "allowed": true, "reason": null},
      {"type": "doc", "id": "91", "allowed": false, "reason": "OBJECT_DENIED"}
    ]
  }
}
```

禅道侧校验要求：

- Bug：功能权限、产品、项目、执行和删除状态。
- 文档：功能权限、文档库权限、具体文档 ACL、草稿和删除状态。
- 写回：必须校验具体动作对应的编辑权限和对象版本。

### 18.2 获取附件

```http
GET /api/ai/files/{file_id}/content
```

AI 服务必须携带源对象和原始用户上下文。禅道确认用户有权读取附件所属对象后才返回文件。

安全要求：

- 只允许 AI 服务访问。
- 设置文件大小、MIME 类型和扩展名白名单。
- 禁止重定向到任意外部地址。
- 文件下载地址使用短期一次性票据，或每次请求重新签名。

### 18.3 写回动作

```http
POST /api/ai/writeback
```

请求：

```json
{
  "actor": "tester01",
  "result_id": "01JZRESULT100",
  "object_type": "bug",
  "object_id": "123",
  "expected_source_revision": "90001",
  "actions": []
}
```

禅道侧必须重新检查 Session 对应用户、动作权限、对象版本和字段白名单，不得因为请求来自 AI 服务就自动放行。

## 19. 错误码

### 19.1 鉴权与权限

| code | error_code | HTTP | 说明 |
| --- | --- | --- | --- |
| `40101` | `MISSING_AUTH_HEADERS` | 401 | 缺少鉴权 Header |
| `40102` | `INVALID_CLIENT` | 401 | Client 不存在、禁用或不属于租户 |
| `40103` | `REQUEST_EXPIRED` | 401 | 时间戳超出窗口 |
| `40104` | `REPLAYED_REQUEST` | 401 | Nonce 已使用 |
| `40105` | `INVALID_BODY_HASH` | 401 | Body 摘要不一致 |
| `40106` | `INVALID_SIGNATURE` | 401 | HMAC 签名错误 |
| `40301` | `SCOPE_DENIED` | 403 | Client 或用户没有功能 Scope |
| `40302` | `PERMISSION_DENIED` | 403 | 没有对象或动作权限 |
| `50301` | `AUTHZ_UNAVAILABLE` | 503 | 禅道权限校验不可用 |
| `40901` | `AUTHZ_VERSION_MISMATCH` | 409 | 权限版本不一致 |

### 19.2 同步和索引

| code | error_code | HTTP | 说明 |
| --- | --- | --- | --- |
| `42201` | `UNSUPPORTED_OBJECT_TYPE` | 422 | 不支持的对象类型 |
| `42202` | `INVALID_SOURCE_REVISION` | 422 | 源版本无效 |
| `40902` | `STALE_SOURCE_REVISION` | 409 | 请求版本早于当前版本 |
| `50201` | `PARSE_FAILED` | 502 | 文档或内容解析失败 |
| `50202` | `OCR_FAILED` | 502 | OCR 失败；可作为部分成功处理 |
| `50203` | `EMBEDDING_FAILED` | 502 | Embedding 调用失败 |
| `50204` | `VECTOR_STORE_FAILED` | 502 | Qdrant 操作失败 |
| `40903` | `PERMISSION_UNKNOWN` | 409 | 对象 ACL 无法确认，禁止进入检索 |

### 19.3 模型和结果

| code | error_code | HTTP | 说明 |
| --- | --- | --- | --- |
| `50302` | `MODEL_UNAVAILABLE` | 503 | 模型不可用 |
| `50205` | `MODEL_TIMEOUT` | 502 | 模型调用超时 |
| `42901` | `MODEL_RATE_LIMITED` | 429 | 模型供应商限流 |
| `50206` | `INVALID_MODEL_OUTPUT` | 502 | 模型输出未通过 Schema 校验 |
| `42203` | `CONTEXT_TOO_LARGE` | 422 | 上下文超过限制 |
| `0` | `INSUFFICIENT_EVIDENCE` | 200 | 正常业务结果，通过 `data.insufficient_evidence=true` 表示 |
| `40904` | `INVALID_RESULT_STATUS` | 409 | 当前结果状态不允许该操作 |
| `40905` | `SOURCE_VERSION_CONFLICT` | 409 | 源对象已变化，需要重新确认 |
| `50207` | `WRITEBACK_FAILED` | 502 | 写回失败 |
| `0` | `WRITEBACK_PARTIAL` | 200 | 部分成功，通过 `data.status=write_back_partial` 表示 |

### 19.4 通用

| code | error_code | HTTP | 说明 |
| --- | --- | --- | --- |
| `42200` | `VALIDATION_ERROR` | 422 | 请求字段校验失败 |
| `40900` | `IDEMPOTENCY_KEY_CONFLICT` | 409 | 相同幂等键对应不同请求 |
| `42900` | `RATE_LIMIT_EXCEEDED` | 429 | 超过限额 |
| `50000` | `INTERNAL_ERROR` | 500 | 未分类内部错误 |
| `50300` | `DEPENDENCY_UNAVAILABLE` | 503 | 关键依赖不可用 |

## 20. 限流和超时

默认建议值：

| 接口类型 | 超时 | 限流维度 |
| --- | --- | --- |
| 健康检查 | 2 秒 | IP |
| 普通查询 | 10 秒 | Client、租户、用户 |
| 同步接口 | 10 秒 | Client、租户 |
| 文档问答同步模式 | 30 秒 | 用户、租户、模型 |
| 模型配置测试 | 60 秒 | 管理员、租户 |
| 异步任务创建 | 10 秒 | 用户、租户 |

超过同步超时的 AI 分析应转为异步任务，不应无限延长禅道 PHP 请求。

建议响应 Header：

```http
X-RateLimit-Limit: 120
X-RateLimit-Remaining: 118
X-RateLimit-Reset: 1783650660
Retry-After: 30
```

## 21. 日志与审计

每次请求至少记录：

- `request_id`
- `tenant_id`
- `client_id`
- `actor_type`
- 用户账号或服务账号
- 接口和 HTTP 状态
- 对象类型和 ID
- 权限校验结果
- 任务 ID、结果 ID
- 模型和 Prompt 版本
- Token、耗时、费用估算
- 错误码

不得记录：

- Client Secret。
- 模型 API Key。
- 完整 Cookie、Token 和签名原文。
- 未脱敏的敏感截图 OCR 文本。
- 用户无权访问的候选检索内容。

## 22. 兼容性规则

- URL 主版本使用 `/api/v1`。
- 新增可选字段属于兼容变更。
- 删除字段、修改字段类型、改变默认语义属于不兼容变更，必须发布 `/api/v2`。
- 客户端必须忽略未知响应字段。
- 服务端不能把新增必填字段直接加入已发布接口。
- 枚举新增值前应确认旧客户端能按未知值降级展示。

## 23. MVP 联调顺序

1. 实现 `/healthz` 和 `/readyz`。
2. 完成 HMAC Middleware、Nonce 防重放和 Client 密钥管理。
3. 实现单对象同步、任务查询和索引状态。
4. 实现禅道批量权限校验接口。
5. 实现 Bug 相似检索。
6. 实现单文档问答和引用。
7. 实现 `help`、`bug`、`doc` 范围的 Conversation、历史消息和流式回答。
8. 联调全局机器人入口、可信页面上下文和弹窗中断恢复。
9. 实现 Bug AI 分析和 AI 结果查询。
10. 实现采纳、驳回；写回先只支持添加评论。
11. 补充失败重试、审计日志和限流。
12. 在权限、幂等、乱序事件和模型失败场景全部通过后再扩展反馈、项目问答和项目周报。

## 24. MVP 验收检查

- 缺少签名或签名错误的请求返回 401。
- 超时请求和重复 Nonce 被拒绝。
- Client 不能跨租户访问数据。
- 请求正文伪造用户名不会改变实际调用身份。
- 无权限 Bug、文档和引用不会进入模型上下文。
- 私有文档必须同时通过文档库和文档 ACL。
- 同一事件重复同步只产生一个有效任务。
- 旧版本 Worker 不能覆盖新版本索引。
- 删除对象不再参与检索。
- 相似 Bug 不返回当前 Bug 本身。
- 文档问答无证据时明确返回 `insufficient_evidence=true`。
- `help` 会话只能检索经过审核的平台帮助知识，不能返回项目业务数据。
- Bug、文档会话连续追问不能扩大创建时的对象范围。
- 流式中断后可以通过历史消息接口恢复最终状态。
- 用户失去源对象权限后，相关历史消息返回 `restricted` 且隐藏内容和引用。
- AI 结果写回前重新检查权限和源对象版本。
- 模型或 AI 服务异常不影响禅道原有功能。
- 日志不包含密钥、完整签名和未脱敏敏感信息。

## 25. 聊天、相似内容推荐与历史消息

本章同时定义首批 MVP 的全局 AI 客服会话和 V1.1 的通用推荐接口。MVP 支持 `help`、`bug`、`doc` 范围的多轮聊天、流式回答和历史消息；项目问答、混合范围和通用跨类型推荐在 V1.1 或后续版本实现。所有入口复用相同的检索、权限、引用和消息存储能力。

全局入口约定：

- 登录后的禅道业务页面右下角展示机器人图标，点击后在当前页面打开聊天弹窗或抽屉，不刷新主页面。
- Bug 详情页由禅道服务端绑定 `bug` 范围，文档详情页绑定 `doc` 范围，其他页面绑定 `help` 范围。
- 前端只能提交消息和页面入口标识；租户、用户、对象类型与对象 ID 必须由禅道服务端根据当前路由生成并签名，AI 服务不得信任前端自报范围。
- 弹窗关闭后保留当前会话；重新打开时通过历史消息接口恢复。聊天接口失败不得阻断当前禅道页面操作。

### 25.1 会话模型

一个 Conversation 固定绑定创建时的租户、所有者和检索范围：

```json
{
  "conversation_id": "01JZCONV001",
  "title": "登录模块问题排查",
  "owner": "pm01",
  "scope": {
    "type": "doc",
    "object_type": "doc",
    "object_id": "88",
    "project_id": "8",
    "allowed_object_types": ["doc"]
  },
  "status": "active",
  "message_count": 6,
  "last_message_at": "2026-07-10T11:20:00+08:00",
  "created_at": "2026-07-10T11:00:00+08:00",
  "updated_at": "2026-07-10T11:20:00+08:00"
}
```

`scope.type` 支持：

| 类型 | 说明 | 阶段 |
| --- | --- | --- |
| `help` | 仅检索经过审核的平台使用手册、FAQ 和操作说明 | MVP |
| `doc` | 仅检索指定文档 | MVP |
| `bug` | 围绕指定 Bug，可检索授权的相似 Bug 和关联文档 | MVP |
| `project` | 检索指定项目内用户有权访问的对象 | V1.1 |
| `mixed` | 显式指定多个对象或对象类型 | V1.2 |
| `global` | 跨所有已授权项目和产品检索 | V1.2，默认关闭 |

规则：

- 会话创建后不能通过发送消息扩大 `scope`。
- `help` 与业务知识集合逻辑隔离，不能携带项目、产品或业务对象 ID。
- 切换文档、项目或 Bug 必须创建新会话。
- `scope` 只是检索上限，不代表用户天然拥有其中所有对象权限。
- 每次发送消息和读取历史消息时都必须重新校验当前权限。
- 会话默认仅创建者可见；后续如支持共享会话，必须设计独立 ACL，不能直接使用项目成员作为共享依据。

### 25.2 消息模型

```json
{
  "message_id": "01JZMSG002",
  "conversation_id": "01JZCONV001",
  "role": "assistant",
  "content": "根据文档，Token 过期后系统会先尝试刷新。",
  "status": "completed",
  "reply_to_message_id": "01JZMSG001",
  "sources": [
    {
      "object_type": "doc",
      "object_id": "88",
      "chunk_id": "doc:88:chunk:3",
      "title": "登录模块 PRD",
      "heading_path": "异常流程 > Token 过期",
      "source_url": "/doc-view-88.html",
      "snippet": "Token 过期后应优先尝试刷新……"
    }
  ],
  "insufficient_evidence": false,
  "model": "default_chat",
  "prompt_version": "doc_qa:1.0.0",
  "usage": {
    "input_tokens": 1250,
    "output_tokens": 96
  },
  "feedback": null,
  "created_at": "2026-07-10T11:01:03+08:00"
}
```

`role` 支持：

- `user`：用户原始消息。
- `assistant`：AI 回答。
- `system_event`：会话范围变化失败、来源权限失效等系统提示；不接受客户端直接创建。

消息状态：

```text
pending -> generating -> completed
                      -> failed
                      -> stopped
completed -> restricted
```

`restricted` 表示消息原来引用的来源已失去访问权限。读取历史记录时不得返回该 AI 消息的原始内容。

### 25.3 创建会话

```http
POST /api/v1/conversations
```

鉴权 Scope：`chat:create`。

平台帮助会话请求：

```json
{
  "title": null,
  "scope": {
    "type": "help",
    "allowed_object_types": ["help"]
  },
  "metadata": {
    "entry_page": "task-browse"
  }
}
```

文档会话请求：

```json
{
  "title": null,
  "scope": {
    "type": "doc",
    "object_type": "doc",
    "object_id": "88",
    "allowed_object_types": ["doc"]
  },
  "metadata": {
    "entry_page": "doc-view"
  }
}
```

Bug 会话请求：

```json
{
  "title": null,
  "scope": {
    "type": "bug",
    "object_type": "bug",
    "object_id": "123",
    "allowed_object_types": ["bug", "doc"]
  },
  "metadata": {
    "entry_page": "bug-view"
  }
}
```

项目会话请求（V1.1）：

```json
{
  "title": "项目风险讨论",
  "scope": {
    "type": "project",
    "project_id": "8",
    "allowed_object_types": ["doc", "bug", "story", "task", "case"]
  }
}
```

响应：`201 Created`

```json
{
  "code": 0,
  "msg": "success",
  "data": {
    "conversation_id": "01JZCONV001",
    "title": "新会话",
    "owner": "pm01",
    "scope": {
      "type": "doc",
      "object_type": "doc",
      "object_id": "88",
      "allowed_object_types": ["doc"]
    },
    "status": "active",
    "message_count": 0,
    "created_at": "2026-07-10T11:00:00+08:00"
  }
}
```

创建 Bug、文档或项目会话时必须校验用户对绑定对象的读取权限。`metadata.entry_page` 只用于审计和产品分析，不作为授权依据；禅道扩展必须根据服务端当前路由构造并签名 `scope`。客户端不能提交自定义 System Prompt、模型 API Key 或绕过检索范围的指令。

### 25.4 查询会话列表

```http
GET /api/v1/conversations?status=active&scope_type=doc&limit=20&cursor=...
```

响应：

```json
{
  "code": 0,
  "msg": "success",
  "data": {
    "items": [
      {
        "conversation_id": "01JZCONV001",
        "title": "登录模块问题排查",
        "scope": {
          "type": "doc",
          "object_type": "doc",
          "object_id": "88"
        },
        "status": "active",
        "message_count": 6,
        "last_message_preview": "根据文档，Token 过期后……",
        "last_message_at": "2026-07-10T11:20:00+08:00"
      }
    ],
    "next_cursor": null,
    "has_more": false
  }
}
```

如果最后一条 AI 消息的来源权限已失效，`last_message_preview` 必须为空或显示“部分历史内容已因权限变化而隐藏”。

### 25.5 查询、修改和删除会话

查询：

```http
GET /api/v1/conversations/{conversation_id}
```

修改标题或归档：

```http
PATCH /api/v1/conversations/{conversation_id}
```

```json
{
  "title": "登录模块 Token 问题",
  "status": "archived"
}
```

允许修改的字段仅包括：

- `title`
- `status`：`active` 或 `archived`

不能通过 PATCH 修改 `owner`、`tenant_id` 或 `scope`。

删除：

```http
DELETE /api/v1/conversations/{conversation_id}
```

删除采用软删除并返回：

```json
{
  "code": 0,
  "msg": "success",
  "data": null
}
```

后台按数据保留策略异步清除消息正文、引用快照和模型输入摘要。审计记录可保留最小必要字段，但不得继续展示会话内容。

### 25.6 发送聊天消息

```http
POST /api/v1/conversations/{conversation_id}/messages
```

鉴权 Scope：`chat:send`。

同步请求：

```json
{
  "content": "Token 刷新失败后会发生什么？",
  "response_mode": "sync",
  "top_k": 8,
  "include_images": true,
  "client_message_id": "web_01JZMSG001"
}
```

规则：

- `content` 去除首尾空白后不能为空。
- 默认最大 8,000 字符，超出返回 `422 MESSAGE_TOO_LARGE`。
- `client_message_id` 在会话内唯一，用于客户端重试幂等。
- 服务端保存用户消息后再生成回答。
- 检索只能在 Conversation `scope` 内进行。
- 历史上下文使用最近消息或服务端摘要，不能无限拼接全部历史消息。
- 历史消息、检索文本和附件内容均视为不可信数据，不能覆盖 System Prompt 或触发未授权工具调用。

同步响应：

```json
{
  "code": 0,
  "msg": "success",
  "data": {
    "user_message": {
      "message_id": "01JZMSG001",
      "role": "user",
      "content": "Token 刷新失败后会发生什么？",
      "status": "completed",
      "created_at": "2026-07-10T11:01:00+08:00"
    },
    "assistant_message": {
      "message_id": "01JZMSG002",
      "role": "assistant",
      "content": "刷新失败后系统会跳转登录页，并提示用户重新登录。",
      "status": "completed",
      "reply_to_message_id": "01JZMSG001",
      "sources": [],
      "insufficient_evidence": false,
      "created_at": "2026-07-10T11:01:03+08:00"
    }
  }
}
```

异步模式使用 `response_mode=async`，返回 `202`、用户消息和任务 ID。任务完成后通过任务接口取得 `assistant_message_id`。

### 25.7 流式聊天

客户端发送：

```http
Accept: text/event-stream
```

并在正文中指定：

```json
{
  "content": "总结当前文档的异常流程",
  "response_mode": "stream"
}
```

SSE 事件顺序：

```text
event: message.created
data: {"code":0,"msg":"success","data":{"user_message_id":"01JZMSG001","assistant_message_id":"01JZMSG002"}}

event: message.delta
data: {"code":0,"msg":"success","data":{"message_id":"01JZMSG002","delta":"根据文档"}}

event: message.sources
data: {"code":0,"msg":"success","data":{"message_id":"01JZMSG002","sources":[...]}}

event: message.completed
data: {"code":0,"msg":"success","data":{"message_id":"01JZMSG002","usage":{"input_tokens":1250,"output_tokens":96}}}
```

失败事件：

```text
event: message.failed
data: {"code":50205,"msg":"回答生成超时","data":{"error_code":"MODEL_TIMEOUT","message_id":"01JZMSG002","details":{}}}
```

流断开后的处理：

- 服务端可继续生成，并将最终状态写入历史消息。
- 客户端使用历史消息接口恢复最终状态，不依赖 SSE 重连重放全部文本。
- 未完成消息不得标记为 `completed`。

### 25.8 查询历史消息

```http
GET /api/v1/conversations/{conversation_id}/messages?limit=50&before=01JZMSG100
```

参数：

| 参数 | 说明 |
| --- | --- |
| `limit` | 默认 50，最大 100 |
| `before` | 返回该消息之前的更早消息，用于向上翻页 |
| `after` | 返回该消息之后的新消息，用于轮询恢复 |
| `include_sources` | 是否返回引用，默认 `true` |

响应按创建时间升序返回，便于客户端直接展示：

```json
{
  "code": 0,
  "msg": "success",
  "data": {
    "items": [
      {
        "message_id": "01JZMSG001",
        "role": "user",
        "content": "Token 过期后怎么处理？",
        "status": "completed",
        "created_at": "2026-07-10T11:01:00+08:00"
      },
      {
        "message_id": "01JZMSG002",
        "role": "assistant",
        "content": "系统会优先尝试刷新 Token。",
        "status": "completed",
        "sources": [],
        "created_at": "2026-07-10T11:01:03+08:00"
      }
    ],
    "previous_cursor": null,
    "has_more_before": false
  }
}
```

历史消息权限规则：

- 只能读取当前用户拥有的会话。
- 读取时重新检查会话绑定对象和每条 AI 消息引用对象的当前权限。
- 用户消息可以保留展示，但失去来源权限的 AI 消息必须返回 `status=restricted`，并隐藏 `content`、`sources` 和模型输入摘要。
- 禁止仅依靠“会话创建时有权限”继续展示历史答案。
- 被删除、归档或无权访问的源对象不能通过历史引用链接泄露标题和摘要。

### 25.9 消息反馈

```http
POST /api/v1/conversations/{conversation_id}/messages/{message_id}/feedback
```

请求：

```json
{
  "rating": "down",
  "reason_codes": ["INCORRECT_CITATION", "INCOMPLETE"],
  "comment": "遗漏了刷新失败后的跳转逻辑"
}
```

`rating` 支持 `up`、`down`、`clear`。同一用户对同一消息只有一条有效反馈，重复提交覆盖旧值并保留审计记录。

### 25.10 相似内容推荐

```http
POST /api/v1/recommendations/similar
```

鉴权 Scope：`recommendation:read`。

以禅道对象为种子：

```json
{
  "seed": {
    "type": "object",
    "object_type": "bug",
    "object_id": "123"
  },
  "target_object_types": ["bug", "doc", "story", "case"],
  "scope": {
    "product_id": "3",
    "project_id": "8"
  },
  "top_k": 10,
  "search_mode": "hybrid",
  "rerank": true,
  "include_closed": true
}
```

以用户输入文本为种子：

```json
{
  "seed": {
    "type": "text",
    "text": "登录后页面空白，控制台出现 403"
  },
  "target_object_types": ["bug", "doc"],
  "scope": {
    "project_id": "8"
  },
  "top_k": 10,
  "search_mode": "hybrid",
  "rerank": true
}
```

规则：

- `seed.type=object` 时必须先校验种子对象读取权限。
- `seed.type=text` 时必须显式指定项目、产品或会话范围，V1.1 不允许无范围全局检索。
- `target_object_types` 必须是租户启用的对象类型。
- 种子对象本身必须从结果中排除。
- 推荐候选必须逐个完成禅道对象级权限校验。
- `scope` 只能缩小检索范围，不能扩大权限。
- 对错误码、接口路径、版本号等内容应使用 dense + sparse 混合检索。

响应：

```json
{
  "code": 0,
  "msg": "success",
  "data": {
    "items": [
      {
        "object_type": "bug",
        "object_id": "98",
        "title": "登录后页面空白",
        "score": 0.91,
        "relation": "similar_issue",
        "reason": "复现步骤和 403 错误码相似",
        "status": "active",
        "source_url": "/bug-view-98.html",
        "evidence": {
          "chunk_id": "bug:98:chunk:1",
          "snippet": "登录后首页请求返回 403……"
        }
      },
      {
        "object_type": "doc",
        "object_id": "88",
        "title": "登录模块故障排查手册",
        "score": 0.84,
        "relation": "related_knowledge",
        "reason": "包含权限回调失败的排查流程",
        "status": "normal",
        "source_url": "/doc-view-88.html",
        "evidence": {
          "chunk_id": "doc:88:chunk:5",
          "snippet": "出现 403 时首先检查权限回调……"
        }
      }
    ],
    "search_mode": "hybrid",
    "filtered_unauthorized_count": 3
  }
}
```

`filtered_unauthorized_count` 仅用于说明过滤数量，不能返回被过滤对象的 ID、标题或任何摘要。

### 25.11 与原有问答接口的关系

- `POST /docs/{doc_id}/ask`：无会话场景可直接问答；传入 `conversation_id` 时，必须验证该会话属于当前用户且绑定同一文档。
- `POST /projects/{project_id}/ask`：传入会话时，必须绑定同一项目。
- 全局 AI 客服弹窗优先使用 `POST /conversations/{id}/messages`。
- 相似 Bug 页面可继续使用 `/bugs/{id}/similar`；跨对象推荐使用 `/recommendations/similar`。
- 两套入口必须复用同一检索、权限、引用和消息存储服务，不能分别实现不同权限逻辑。

### 25.12 历史消息存储与保留

建议关系表：

```text
conversations
conversation_messages
message_sources
message_feedback
conversation_summaries
```

关键约束：

- `conversations`：`tenant_id + conversation_id` 唯一。
- `conversation_messages`：`tenant_id + conversation_id + client_message_id` 唯一。
- `message_sources` 只保存已通过权限校验的引用。
- 历史消息默认保留时间由租户配置，例如 90、180 或 365 天。
- 用户删除会话后进入异步清除队列。
- 企业客户可配置不保存完整模型输入，只保存用户消息、最终答案、引用和调用摘要。
- 用于对话压缩的 Summary 必须继承会话权限，并在来源权限变化后重新生成或失效。

### 25.13 新增错误码

| code | error_code | HTTP | 说明 |
| --- | --- | --- | --- |
| `40401` | `CONVERSATION_NOT_FOUND` | 404 | 会话不存在或用户无权知道其存在 |
| `40906` | `CONVERSATION_SCOPE_MISMATCH` | 409 | 问答对象与会话绑定范围不一致 |
| `40907` | `CONVERSATION_ARCHIVED` | 409 | 已归档会话不能继续发送消息 |
| `42204` | `MESSAGE_TOO_LARGE` | 422 | 单条消息超过限制 |
| `50208` | `MESSAGE_GENERATION_FAILED` | 502 | 回答生成失败 |
| `40303` | `MESSAGE_RESTRICTED` | 403 | 历史消息引用权限已失效 |
| `42205` | `INVALID_RECOMMENDATION_SEED` | 422 | 推荐种子无效 |
| `42206` | `RECOMMENDATION_SCOPE_REQUIRED` | 422 | 文本推荐未提供检索范围 |
| `42207` | `INVALID_PAGE_CONTEXT` | 422 | 页面上下文缺失、与路由不一致或不支持 |

### 25.14 新增验收项

- MVP 用户可以创建 `help`、文档和 Bug 范围的会话；项目范围会话按 V1.1 验收。
- 连续追问不会扩大创建时的检索范围。
- 同一 `client_message_id` 重试不会生成重复消息。
- 流式中断后可以通过历史消息接口恢复最终状态。
- 历史消息按会话分页并保持稳定顺序。
- 用户失去源文档权限后，相关历史 AI 回答变为 `restricted`。
- 全局机器人入口在登录后的业务页面可见，点击后无需刷新页面即可打开、关闭和恢复聊天弹窗。
- Bug、文档页面自动绑定当前对象，其他页面仅使用隔离的平台帮助知识库。
- 文档快捷问答和全局 AI 客服使用相同的权限、检索和会话实现。
- V1.1 相似推荐支持 Bug、文档、需求和测试用例等目标类型。
- V1.1 推荐结果不包含种子对象自身，未授权候选不会返回标题、摘要、链接或证据。
