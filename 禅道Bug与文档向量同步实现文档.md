# 禅道 Bug 与文档向量同步实现文档

版本：v1.1
日期：2026-07-10
文档类型：开发实现文档  
适用范围：禅道扩展、zentao-ai-service、Qdrant 向量库、Bug 相似检索、文档问答

## 1. 实现目标

本实现文档用于指导后续开发，将禅道中的 Bug 和文档同步到独立 AI 服务，并写入向量数据库，支撑以下能力：

- Bug 相似问题检索。
- Bug AI 分析。
- 文档摘要和文档问答。
- 项目范围知识问答。
- Bug 和文档图片 OCR 检索。
- 后续扩展到客户反馈、需求、测试用例、项目周报。

第一阶段只做两类对象：

- `bug`
- `doc`

实现原则：

- 禅道侧只做轻量扩展，不把向量库、模型调用、任务队列放进禅道 PHP 应用。
- AI 服务负责同步状态、解析、切片、OCR、Embedding、Qdrant 入库。
- 向量库只存 chunk 文本、图片 OCR 文本、图片描述和 metadata。
- 业务状态、任务状态、分析结果、调用日志放 AI 服务 PostgreSQL。
- 所有向量数据必须带租户、对象、项目/产品、权限范围和来源链接。

## 2. 总体架构

```text
禅道
  ├── Bug 创建/编辑/查看
  ├── 文档创建/编辑/查看
  └── extension/custom AI 扩展
        ↓
        POST /api/v1/sync/zentao/object
        ↓
zentao-ai-service
  ├── ai-api 接收同步请求
  ├── PostgreSQL 保存同步状态
  ├── Redis/Celery 执行异步任务
  ├── parser 提取文本和图片
  ├── embedding 生成向量
  └── Qdrant 保存向量 chunk
```

查询链路：

```text
用户点击 Bug AI 分析 / 文档问答
  ↓
禅道扩展带当前用户身份调用 AI 服务
  ↓
AI 服务根据用户权限过滤 Qdrant
  ↓
检索相似 chunk
  ↓
调用 LLM 生成答案或分析结果
  ↓
返回禅道页面展示
```

## 3. 分阶段实现计划

### 3.1 阶段 0：基础骨架

目标：

- 建立 AI 服务基础工程。
- 建立数据库迁移。
- 建立 Qdrant 集合。
- 建立禅道扩展配置。

交付：

- FastAPI 服务。
- PostgreSQL 连接。
- Redis/Celery Worker。
- Qdrant 连接。
- `/healthz`、`/readyz`。
- 禅道侧 AI 服务地址和 Token 配置。

验收：

- AI 服务可以启动。
- Worker 可以启动。
- API 可以连接 PostgreSQL、Redis、Qdrant。
- 禅道扩展可以读取 AI 服务配置。

### 3.2 阶段 1：Bug 同步和向量入库

目标：

- 禅道 Bug 创建/编辑后同步到 AI 服务。
- AI 服务将 Bug 拼接成文本 chunk。
- 生成 embedding 并写入 Qdrant。

交付：

- 禅道 Bug 同步扩展。
- `POST /api/v1/sync/zentao/object`。
- `sync_objects` 表。
- `index_chunks` 表。
- Bug 文本构建器。
- Bug 向量化 Worker。

验收：

- 创建或编辑 Bug 后，AI 服务能收到同步请求。
- AI 服务能在 Qdrant 中生成 `bug:{id}:chunk:1`。
- 修改 Bug 后旧 chunk 被替换或标记删除。
- 删除 Bug 后向量 payload 标记 `deleted=true`。

### 3.3 阶段 2：Bug 相似检索

目标：

- 在 Bug 详情页增加“相似 Bug”或“AI 分析”入口。
- 根据当前 Bug 查询 Qdrant 中相似 Bug。

交付：

- `POST /api/v1/bugs/{id}/similar`。
- 禅道 Bug 详情页 AI 面板。
- 权限过滤。

验收：

- 用户只能看到自己有权限产品/项目下的相似 Bug。
- 相似结果包含 Bug ID、标题、相似度、状态、来源链接。
- 无相似 Bug 时返回空列表和明确提示。

### 3.4 阶段 3：文档同步和切片入库

目标：

- 禅道文档创建/编辑后同步到 AI 服务。
- AI 服务解析文档 HTML/Markdown 内容。
- 按标题和段落切片。
- 生成 embedding 并写入 Qdrant。

交付：

- 禅道文档同步扩展。
- 文档解析器。
- 文档切片器。
- 文档向量化 Worker。

验收：

- 文档更新后重新索引。
- 每个 chunk 保留 `heading_path`。
- Qdrant 中存在 `doc:{id}:chunk:{n}`。
- 文档详情页能查看索引状态。

### 3.5 阶段 4：文档问答

目标：

- 支持单文档问答。
- 支持项目范围文档问答。
- 回答必须带引用来源。

交付：

- `POST /api/v1/docs/{id}/ask`。
- `POST /api/v1/projects/{id}/ask`。
- RAG 检索。
- LLM 调用。
- 引用来源展示。

验收：

- 回答展示引用文档、章节和 source_url。
- 没有证据时回答“未在当前资料中找到依据”。
- 无权限文档不会被检索和引用。

### 3.6 阶段 5：Bug 和文档图片处理（V1.1）

目标：

- 处理 Bug 附件截图和文档图片。
- OCR 提取图片文字。
- 可选使用多模态模型生成图片描述。
- 图片作为 image chunk 写入 Qdrant。

交付：

- 文件下载或读取能力。
- OCR 服务封装。
- image chunk 构建器。
- 图片敏感信息脱敏。

验收：

- Bug 截图中的错误码可被检索。
- 文档流程图 OCR 文本可被检索。
- 图片 chunk 保留 `file_id`、`source_url`、`heading_path`。
- OCR 置信度低时返回提示。

### 3.7 阶段 6：索引状态和失败重试

目标：

- 禅道页面能看到对象同步、解析、向量化状态。
- 管理员可以重试失败任务。

交付：

- `GET /api/v1/index/status`。
- `POST /api/v1/index/retry`。
- 禅道索引状态面板。

验收：

- 能区分未同步、同步中、解析失败、向量化失败、已索引。
- 失败任务可以重试。
- 重试不会重复创建无效向量。

## 4. 禅道侧实现设计

### 4.1 扩展目录建议

不直接修改禅道核心文件，使用 `extension/custom`。

建议目录：

```text
zentaopms/extension/custom/common/ext/config/ai_service.php
zentaopms/extension/custom/common/ext/lang/zh-cn/ai_service.php

zentaopms/extension/custom/bug/ext/control/aisync.php
zentaopms/extension/custom/bug/ext/model/aisync.php
zentaopms/extension/custom/bug/ext/ui/view.ai.html.hook.php
zentaopms/extension/custom/bug/ext/js/view.ai.js
zentaopms/extension/custom/bug/ext/css/view.ai.css

zentaopms/extension/custom/doc/ext/control/aisync.php
zentaopms/extension/custom/doc/ext/model/aisync.php
zentaopms/extension/custom/doc/ext/ui/view.ai.html.hook.php
zentaopms/extension/custom/doc/ext/js/view.ai.js
zentaopms/extension/custom/doc/ext/css/view.ai.css
```

如果禅道当前版本的扩展加载路径有差异，以实际 `extension/custom` 机制为准。实现前先用最小 hook 验证 Bug 详情页和文档详情页能插入按钮。

### 4.2 当前禅道扩展机制确认

当前禅道代码可以通过 `extension/custom` 增加模块扩展方法和页面 hook。

已确认现有项目中已经存在 Bug AI 扩展示例：

```text
zentaopms/extension/custom/bug/ext/control/aianalyze.php
zentaopms/extension/custom/bug/ext/ui/view.ai.html.hook.php
```

该示例说明：

- 可以通过 `helper::importControl('bug')` 扩展 Bug 控制器。
- 可以新增 `bug-aiAnalyze` 这类自定义控制器方法。
- 可以通过 `ext/ui/*.hook.php` 在 Bug 详情页插入 AI 面板或按钮。

因此后续可以按同样方式新增：

```text
bug-aiSync
doc-aiSync
```

用于手动触发对象同步。

需要注意：

- Bug 控制器在创建、编辑、删除等流程中有 `executeHooks($bugID)` 调用。
- 但框架层在 open edition 下会直接返回空字符串，因此不能把 Open 版自动同步依赖在 `executeHooks` 上。
- Doc 控制器的创建、编辑、删除流程没有同样稳定的 `executeHooks` 入口。
- 如果要做到保存后立即自动同步，需要使用扩展控制器覆盖对应方法，或者修改核心流程；这会增加禅道升级维护成本。

基于以上限制，MVP 推荐采用：

1. `bug-aiSync`、`doc-aiSync` 手动同步。
2. AI 服务或禅道扩展提供定时增量扫描，按 `lastEditedDate`、`editedDate`、`deleted` 等字段补偿同步。
3. 在手动同步和定时补偿稳定后，再评估是否覆盖 Bug/Doc 的 `create`、`edit`、`delete` 方法实现实时触发。

### 4.3 禅道配置项

配置项：

```php
$config->aiService = new stdclass();
$config->aiService->baseUrl = 'http://ai-api:8000';
$config->aiService->token   = '';
$config->aiService->tenant  = 'default';
$config->aiService->timeout = 5;
```

安全要求：

- Token 不输出到前端。
- 前端只调用禅道扩展控制器。
- 禅道扩展控制器再调用 AI 服务。

### 4.4 Bug 同步触发点

MVP 先做两种触发：

1. Bug 详情页手动同步。
2. 定时增量补偿同步。

手动同步入口：

```text
Bug 详情页
  -> AI 面板
  -> 重新同步索引
```

自动同步入口：

- Bug 创建成功后。
- Bug 编辑成功后。
- Bug 关闭、激活、删除等状态变化后。

实现策略：

- 第一版不依赖 `executeHooks`，因为 Open 版框架层不会执行实际 hook。
- 第一版新增 `bug-aiSync` 控制器方法，供详情页按钮和后台任务调用。
- 第一版新增 Bug 变更扫描能力，按 `openedDate`、`lastEditedDate`、`deleted` 找出待同步 Bug。
- 第二版如需实时触发，再通过扩展覆盖 Bug `create`、`edit`、`delete`、`resolve`、`close`、`activate` 等方法，在父流程成功后调用同步接口。

Bug 自动触发不要直接写 Qdrant。禅道只调用 AI 服务：

```text
Bug create/edit/delete/resolve/close/activate
  -> 禅道扩展构造对象快照
  -> POST /api/v1/sync/zentao/object
  -> AI 服务异步写 PostgreSQL 和 Qdrant
```

### 4.5 文档同步触发点

MVP 先做两种触发：

1. 文档详情页手动同步。
2. 定时增量补偿同步。

同步内容：

- 文档标题。
- 文档正文 HTML。
- 文档库、项目、产品、执行信息。
- 附件列表。
- 文档权限信息。

实现策略：

- 第一版新增 `doc-aiSync` 控制器方法，供详情页按钮和后台任务调用。
- 第一版新增文档变更扫描能力，按 `addedDate`、`editedDate`、`deleted` 找出待同步文档。
- 文档模块没有和 Bug 一样稳定的 after hook 入口，不建议 MVP 覆盖完整 `create`、`edit`、`delete` 流程。
- 第二版如必须实时同步，再评估扩展覆盖 Doc `create`、`edit`、`delete` 方法，并保留定时补偿兜底。

文档自动触发同样不要直接写 Qdrant。禅道只负责发送对象快照和附件引用，AI 服务负责解析 HTML、下载附件、OCR、分段和向量化。

### 4.6 禅道发送到 AI 服务的数据

禅道扩展不要只发送对象 ID。建议发送对象快照，AI 服务也可以按需回调禅道补充附件。

统一请求：

```http
POST /api/v1/sync/zentao/object
X-Client-Id: zentao_default
X-Tenant-Id: default
X-Actor-Type: user
X-Zentao-User: admin
X-Request-Id: req_xxx
X-Timestamp: 1783580000
X-Nonce: 5f7e3198-0a92-4dee-96fa-c12bb34fd509
X-Body-SHA256: 51e7f...
X-Signature: sha256=8d80a...
X-Idempotency-Key: idem_xxx
Content-Type: application/json
```

Bug payload：

```json
{
  "source": "zentao",
  "tenant_id": "default",
  "object_type": "bug",
  "object_id": "123",
  "event": "created|updated|deleted|closed",
  "version": "2026-07-09T16:30:00+08:00",
  "payload": {
    "title": "登录后无法进入首页",
    "steps": "复现步骤...",
    "severity": "3",
    "priority": "1",
    "status": "active",
    "product_id": "3",
    "project_id": "8",
    "execution_id": "0",
    "module_id": "12",
    "opened_by": "tester01",
    "assigned_to": "dev01",
    "comments": [
      {
        "id": "9001",
        "actor": "dev01",
        "date": "2026-07-09T15:00:00+08:00",
        "content": "疑似权限回调失败"
      }
    ],
    "files": [
      {
        "id": "889",
        "name": "error.png",
        "extension": "png",
        "url": "/file-read-889.html"
      }
    ],
    "source_url": "/bug-view-123.html",
    "permission_scope": ["project:8", "product:3"]
  }
}
```

Doc payload：

```json
{
  "source": "zentao",
  "tenant_id": "default",
  "object_type": "doc",
  "object_id": "88",
  "event": "created|updated|deleted",
  "version": "2026-07-09T16:30:00+08:00",
  "payload": {
    "title": "登录模块 PRD",
    "content_type": "html",
    "content": "<h1>登录模块</h1><h2>异常流程</h2><p>...</p>",
    "lib_id": "10",
    "project_id": "8",
    "product_id": "3",
    "execution_id": "0",
    "added_by": "pm01",
    "edited_by": "pm01",
    "files": [
      {
        "id": "902",
        "name": "login-flow.png",
        "extension": "png",
        "url": "/file-read-902.html"
      }
    ],
    "source_url": "/doc-view-88.html",
    "permission_scope": ["project:8", "product:3"]
  }
}
```

## 5. AI 服务 API 设计

本章只说明同步实现所需的业务字段。请求头、签名原文、通用响应、错误码和接口路径以《AI服务接口文档》为唯一契约。

### 5.1 对象同步接口

```http
POST /api/v1/sync/zentao/object
```

职责：

- 鉴权。
- 校验签名。
- 保存对象快照。
- 生成异步索引任务。
- 返回 task_id。

响应：

```json
{
  "code": 0,
  "msg": "success",
  "data": {
    "task_id": "task_abc123",
    "status": "queued"
  }
}
```

### 5.2 索引状态接口

```http
GET /api/v1/index/status?object_type=bug&object_id=123
```

响应：

```json
{
  "code": 0,
  "msg": "success",
  "data": {
    "object_type": "bug",
    "object_id": "123",
    "sync_status": "synced",
    "parse_status": "success",
    "index_status": "indexed",
    "chunk_count": 2,
    "last_indexed_at": "2026-07-09T16:31:00+08:00",
    "error_message": null
  }
}
```

### 5.3 失败重试接口

```http
POST /api/v1/index/retry
```

请求：

```json
{
  "object_type": "doc",
  "object_id": "88"
}
```

### 5.4 Bug 相似检索接口

```http
POST /api/v1/bugs/{id}/similar
```

请求：

```json
{
  "top_k": 10
}
```

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
        "status": "active",
        "source_url": "/bug-view-98.html",
        "evidence": "复现步骤和 403 错误相似"
      }
    ]
  }
}
```

### 5.5 文档问答接口

```http
POST /api/v1/docs/{id}/ask
```

请求：

```json
{
  "question": "Token 过期后系统怎么处理？",
  "top_k": 8
}
```

响应：

```json
{
  "code": 0,
  "msg": "success",
  "data": {
    "answer": "根据文档，Token 过期后系统会先尝试刷新 Token，刷新失败后跳转登录页。",
    "sources": [
      {
        "object_type": "doc",
        "object_id": "88",
        "title": "登录模块 PRD",
        "heading_path": "登录模块 PRD > 异常流程 > Token 过期",
        "source_url": "/doc-view-88.html#chunk-3",
        "snippet": "当用户 Token 过期时，系统应尝试刷新 Token..."
      }
    ]
  }
}
```

## 6. AI 服务数据库设计

### 6.1 sync_objects

用途：保存禅道对象同步快照和索引状态。

字段：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| id | bigserial | 主键 |
| tenant_id | varchar | 租户 |
| source | varchar | zentao |
| object_type | varchar | bug、doc |
| object_id | varchar | 禅道对象 ID |
| version | varchar | 对象版本，通常用更新时间 |
| event | varchar | created、updated、deleted |
| payload_json | jsonb | 对象快照 |
| sync_status | varchar | pending、synced、failed |
| parse_status | varchar | pending、success、failed |
| index_status | varchar | pending、indexing、indexed、failed |
| chunk_count | int | chunk 数量 |
| error_message | text | 错误信息 |
| last_synced_at | timestamptz | 最近同步时间 |
| last_indexed_at | timestamptz | 最近索引时间 |
| created_at | timestamptz | 创建时间 |
| updated_at | timestamptz | 更新时间 |

唯一约束：

```text
tenant_id + object_type + object_id
```

### 6.2 index_chunks

用途：保存 AI 服务侧 chunk 元数据，便于调试和重建索引。

字段：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| id | bigserial | 主键 |
| tenant_id | varchar | 租户 |
| object_type | varchar | bug、doc |
| object_id | varchar | 禅道对象 ID |
| chunk_id | varchar | Qdrant point ID |
| chunk_type | varchar | text、image、comment、attachment |
| chunk_index | int | 序号 |
| title | varchar | 标题 |
| heading_path | text | 文档章节 |
| content | text | chunk 文本 |
| content_hash | varchar | 内容 hash |
| file_id | varchar | 图片或附件 ID |
| source_url | text | 来源链接 |
| metadata_json | jsonb | metadata |
| indexed_at | timestamptz | 入库时间 |

唯一约束：

```text
tenant_id + chunk_id
```

### 6.3 index_tasks

用途：保存异步任务状态。

字段：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| id | uuid | task_id |
| tenant_id | varchar | 租户 |
| object_type | varchar | 对象类型 |
| object_id | varchar | 对象 ID |
| task_type | varchar | sync、parse、index、ocr |
| status | varchar | queued、running、success、failed |
| retry_count | int | 重试次数 |
| error_message | text | 错误信息 |
| created_at | timestamptz | 创建时间 |
| started_at | timestamptz | 开始时间 |
| finished_at | timestamptz | 完成时间 |

### 6.4 llm_calls

用途：保存模型调用审计。

MVP 可以先只记录 embedding 调用摘要：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| id | bigserial | 主键 |
| request_id | varchar | 请求 ID |
| tenant_id | varchar | 租户 |
| call_type | varchar | embedding、chat、vision |
| provider | varchar | 模型供应商 |
| model | varchar | 模型名 |
| input_hash | varchar | 输入 hash |
| token_input | int | 输入 token |
| token_output | int | 输出 token |
| duration_ms | int | 耗时 |
| success | boolean | 是否成功 |
| error_message | text | 错误信息 |
| created_at | timestamptz | 创建时间 |

## 7. Qdrant 设计

### 7.1 集合

MVP 使用统一集合：

```text
zentao_knowledge
```

通过 payload 中的 `tenant_id` 做租户隔离。

### 7.2 point ID 规则

```text
bug:{object_id}:chunk:{index}
bug:{object_id}:image:{file_id}
doc:{object_id}:chunk:{index}
doc:{object_id}:image:{file_id}
```

示例：

```text
bug:123:chunk:1
bug:123:image:889
doc:88:chunk:3
doc:88:image:902
```

### 7.3 payload 标准

```json
{
  "tenant_id": "default",
  "source": "zentao",
  "object_type": "bug",
  "object_id": "123",
  "chunk_id": "bug:123:chunk:1",
  "chunk_type": "text",
  "chunk_index": 1,
  "title": "登录后无法进入首页",
  "heading_path": "",
  "content": "标题：登录后无法进入首页\n复现步骤：...",
  "project_id": "8",
  "product_id": "3",
  "execution_id": "0",
  "module_id": "12",
  "status": "active",
  "permission_scope": ["project:8", "product:3"],
  "source_url": "/bug-view-123.html",
  "file_id": "",
  "deleted": false,
  "updated_at": "2026-07-09T16:30:00+08:00"
}
```

### 7.4 更新策略

对象更新时：

1. 查询旧 chunk。
2. 删除旧 point 或标记 `deleted=true`。
3. 重新构建 chunk。
4. 重新生成 embedding。
5. upsert 新 point。
6. 更新 `index_chunks` 和 `sync_objects` 状态。

MVP 建议直接删除旧 point 后重建。

对象删除时：

- 优先更新 payload：`deleted=true`。
- 检索时过滤 `deleted=false`。

## 8. Bug 文本构建规则

Bug 通常整条作为一个 text chunk。

模板：

```text
标题：{title}
类型：Bug
产品：{product_name}
项目：{project_name}
模块：{module_name}
严重程度：{severity}
优先级：{priority}
状态：{status}

复现步骤：
{steps}

实际结果：
{actual_result}

期望结果：
{expected_result}

关键评论：
{important_comments}
```

评论处理：

- 过滤“收到”“已处理”“看看”“+1”等低价值评论。
- 保留包含错误原因、复现环境、处理结论、客户影响的评论。
- 评论过长时另建 `bug:{id}:comment:{index}`。

图片处理：

- 每张图片生成一个 image chunk。
- content 由 OCR 文本、图片描述和 Bug 标题拼接。

```text
Bug：登录后无法进入首页
图片：error.png
OCR 文本：403 Forbidden
图片描述：登录后页面空白，浏览器控制台显示权限错误。
```

## 9. 文档解析和切片规则

### 9.1 文档文本提取

输入可能是：

- HTML。
- Markdown。
- 纯文本。
- 附件解析文本。

处理步骤：

1. 清理 HTML 标签，保留标题层级、列表、表格文本。
2. 提取 H1/H2/H3 作为章节路径。
3. 提取正文段落。
4. 提取图片引用位置。
5. 生成文本 chunk 和 image chunk。

### 9.2 文档切片

规则：

- 优先按 H1/H2/H3 切分。
- 单个章节超过 1000 中文字时，按段落切分。
- 段落仍过长时，按句号、分号、换行切分。
- chunk 目标长度 500-1000 中文字。
- overlap 100-150 字。
- 每个 chunk 前补充文档标题和章节路径。

chunk 内容模板：

```text
文档：{doc_title}
章节：{heading_path}

{chunk_content}
```

### 9.3 文档图片处理

文档图片生成 image chunk。

content 模板：

```text
文档：{doc_title}
章节：{heading_path}
图片：{file_name}
OCR 文本：{ocr_text}
图片描述：{image_caption}
```

V1.1 可以先只做 OCR，`image_caption` 可为空。后续接视觉模型后补充描述。

## 10. 模型调用与网关

### 10.1 模型调用原则

业务模块不得裸调用模型供应商 API。

不允许在 `bug_ai`、`doc_ai`、`feedback_ai` 等业务模块中直接写：

```python
openai.chat.completions.create(...)
```

也不允许在业务模块中直接拼接不同厂商的 HTTP 请求。

所有模型调用必须经过内部网关：

```text
业务模块
  ↓
modules/llm/gateway.py
modules/embedding/gateway.py
  ↓
模型适配层
  ↓
OpenAI / DeepSeek / 通义 / Azure / 本地模型
```

这样后续更换模型、增加私有模型、调整重试、限流、token 统计和降级策略时，不需要修改 Bug 分析、文档问答等业务代码。

### 10.2 MVP 推荐方案

MVP 推荐使用：

- `LiteLLM` 作为多模型适配层。
- `qdrant-client` 直接操作 Qdrant。
- 自研轻量 RAG service，不重度依赖 Agent 框架。
- 文档切片先用项目内规则实现。

推荐调用关系：

```text
bug_ai / doc_ai / feedback_ai
  ↓
modules/llm/gateway.py
  ↓
LiteLLM
  ↓
OpenAI Compatible / DeepSeek / 通义 / Azure / 私有模型
```

暂不建议 MVP 重度使用 LangChain。原因是本项目核心是禅道对象同步、权限过滤、Qdrant 检索和人工确认写回，而不是通用 Agent 编排。

后续如果要做复杂文档索引、高级 RAG、工具调用或 Agent，再评估引入 LlamaIndex 或 LangChain。

### 10.3 聊天模型统一接口

业务模块只允许调用内部接口：

```python
result = llm.chat(
    messages=messages,
    model="default_chat",
    response_format="json",
    temperature=0.2,
    timeout=60
)
```

接口要求：

- 支持默认模型别名，例如 `default_chat`。
- 支持 JSON 输出。
- 支持超时。
- 支持重试。
- 支持 request_id。
- 记录输入摘要、输出摘要、token、耗时、模型名称和错误信息。

### 10.4 Embedding 统一接口

Embedding 也必须经过内部接口：

```python
vector = embedding.embed(
    text=chunk.content,
    model="default_embedding"
)
```

接口要求：

- 支持默认向量模型别名，例如 `default_embedding`。
- 支持批量 embedding。
- 支持内容 hash 去重。
- 支持失败重试。
- 记录模型调用日志。

### 10.5 视觉模型接口

图片理解能力可选，V1.1 可以先只做 OCR。

后续如接多模态模型，必须通过内部接口：

```python
caption = vision.describe(
    image_bytes=image,
    prompt="请描述这张 Bug 截图中的异常信息",
    model="default_vision"
)
```

视觉模型输出只作为 image chunk 的补充描述，不替代 OCR 结果。

### 10.6 配置示例

```env
LLM_ADAPTER=litellm
LLM_PROVIDER=openai_compatible
LLM_BASE_URL=https://model.example.com/v1
LLM_API_KEY=******
DEFAULT_CHAT_MODEL=deepseek-chat

EMBEDDING_PROVIDER=openai_compatible
EMBEDDING_BASE_URL=https://model.example.com/v1
EMBEDDING_API_KEY=******
DEFAULT_EMBEDDING_MODEL=bge-m3
EMBEDDING_DIMENSION=1024

VISION_PROVIDER=openai_compatible
VISION_BASE_URL=https://model.example.com/v1
VISION_API_KEY=******
DEFAULT_VISION_MODEL=
```

### 10.7 降级策略

- 聊天模型失败：重试后返回分析失败，保留任务状态。
- Embedding 模型失败：任务标记为 `index_status=failed`，允许重试。
- 视觉模型失败：不影响文本和 OCR 入库，只跳过图片描述。
- OCR 失败：不影响文本 chunk 入库，只记录图片处理失败。

## 11. Embedding 实现

### 11.1 模型配置

AI 服务配置：

```env
EMBEDDING_PROVIDER=openai_compatible
EMBEDDING_BASE_URL=https://model.example.com/v1
EMBEDDING_API_KEY=******
DEFAULT_EMBEDDING_MODEL=bge-m3
EMBEDDING_DIMENSION=1024
```

要求：

- Embedding 模型和聊天模型分开配置。
- Embedding 失败可重试。
- 同一内容 hash 未变化时可跳过重新向量化。

### 11.2 幂等策略

每个 chunk 计算 `content_hash`：

```text
sha256(tenant_id + object_type + object_id + chunk_type + content)
```

如果旧 chunk 的 `content_hash` 未变化：

- 可跳过重新生成 embedding。
- 只更新 metadata。

MVP 可以先简单重建，后续再优化 hash 跳过。

## 12. 权限过滤实现

### 12.1 同步时写入权限范围

禅道侧必须传下列权限 metadata，用于缩小 Qdrant 候选范围和提高检索效率：

```json
{
  "permission_scope": ["project:8", "product:3"]
}
```

如果对象是私有文档，还需要传：

```json
{
  "permission_scope": ["project:8", "user:pm01", "group:12"]
}
```

### 12.2 检索时过滤

检索必须包含：

- `tenant_id = 当前租户`
- `deleted = false`
- `permission_scope` 与当前用户权限范围有交集，作为候选预过滤条件

相似 Bug 检索额外过滤：

- `object_type = bug`
- 同产品或同项目优先。

文档问答检索过滤：

- 单文档问答：`object_type=doc AND object_id=当前文档 ID`
- 项目问答：`project_id=当前项目 ID`

### 12.3 权限兜底

`permission_scope` 只是检索预过滤 metadata，不是授权依据。AI 服务不能信任请求正文中的用户名、角色或权限范围。

首批 MVP 必须在候选内容进入 Prompt 前，调用禅道批量权限校验接口逐个确认当前用户对来源对象的读取权限；私有文档还必须同时校验文档库和文档 ACL。权限接口超时、返回不明确、ACL 版本落后或候选对象未完成校验时默认拒绝访问。读取已有 AI 结果和引用时也必须重新校验当前权限。

本节以《AI服务接口文档》第 5 章和第 18.1 节为最终契约。

## 13. 异常和重试

常见异常：

| 场景 | 处理 |
| --- | --- |
| 禅道同步请求失败 | 禅道页面提示失败，允许手动重试 |
| AI 服务保存对象失败 | 返回错误，记录 request_id |
| Worker 执行失败 | index_tasks 标记 failed |
| OCR 失败 | 跳过 image chunk，记录警告 |
| Embedding 失败 | 重试，超过次数标记 index_status=failed |
| Qdrant 写入失败 | 重试，失败后保留 chunk 记录 |
| 权限范围为空 | 不入向量库，标记权限不可确认 |

重试规则：

- Worker 自动重试 2 次。
- 管理员可手动重试。
- 重试必须幂等，不能重复创建多个有效 point。

## 14. 安全要求

- AI 服务 API 必须使用 Token 鉴权。
- 服务间请求增加 `X-Signature` 和 `X-Timestamp` 防重放。
- API Key 不进入禅道前端。
- 日志不记录完整 Token、Cookie、API Key。
- 图片 OCR 后必须脱敏。
- 外部模型调用前按配置脱敏。
- 不允许未确认权限的数据入向量库。

脱敏规则：

```text
手机号 -> [PHONE]
邮箱 -> [EMAIL]
身份证 -> [ID_CARD]
Token/API Key/Cookie -> [SECRET]
```

## 15. 代码模块建议

AI 服务目录：

```text
ai-service/
├── apps/
│   ├── api/
│   ├── worker/
│   └── scheduler/
├── core/
│   ├── config.py
│   ├── security.py
│   ├── logging.py
│   └── errors.py
├── modules/
│   ├── zentao_sync/
│   │   ├── schemas.py
│   │   ├── service.py
│   │   └── repository.py
│   ├── indexer/
│   │   ├── bug_builder.py
│   │   ├── doc_parser.py
│   │   ├── chunker.py
│   │   ├── image_ocr.py
│   │   └── service.py
│   ├── embedding/
│   │   ├── gateway.py
│   │   └── service.py
│   ├── llm/
│   │   ├── gateway.py
│   │   └── adapters.py
│   ├── vision/
│   │   └── gateway.py
│   ├── vectorstore/
│   │   └── qdrant.py
│   ├── bug_ai/
│   │   └── similar.py
│   └── doc_ai/
│       └── qa.py
├── repositories/
├── migrations/
└── tests/
```

禅道扩展目录：

```text
extension/custom/common/ext/config/ai_service.php
extension/custom/bug/ext/control/aisync.php
extension/custom/bug/ext/model/aisync.php
extension/custom/bug/ext/ui/view.ai.html.hook.php
extension/custom/doc/ext/control/aisync.php
extension/custom/doc/ext/model/aisync.php
extension/custom/doc/ext/ui/view.ai.html.hook.php
```

## 16. 详细实现步骤

### 步骤 1：搭建 AI 服务骨架

任务：

- 新建 FastAPI 应用。
- 新建 Celery Worker。
- 配置 PostgreSQL、Redis、Qdrant。
- 增加 `/healthz`、`/readyz`。

完成标准：

- `docker compose up` 后 API 和 Worker 正常运行。
- 健康检查返回成功。

### 步骤 2：创建数据库迁移

任务：

- 创建 `sync_objects`。
- 创建 `index_chunks`。
- 创建 `index_tasks`。
- 创建 `llm_calls`。

完成标准：

- Alembic 迁移可执行。
- 表结构符合本文档。

### 步骤 3：实现对象同步接口

任务：

- 实现 `POST /api/v1/sync/zentao/object`。
- 校验 Token。
- 保存 payload。
- 创建索引任务。

完成标准：

- 用 curl 发送 Bug payload 后，数据库有 `sync_objects` 和 `index_tasks`。

### 步骤 4：实现 Bug chunk 构建

任务：

- 从 payload 构建 Bug 文本。
- 过滤低价值评论。
- 生成 chunk_id。

完成标准：

- 输入 Bug payload，输出 `bug:{id}:chunk:1`。

### 步骤 5：实现模型网关、Embedding 和 Qdrant upsert

任务：

- 封装 `modules/llm/gateway.py`。
- 封装 Embedding 网关。
- 封装 Qdrant upsert。
- Worker 执行 Bug 索引。

完成标准：

- Qdrant 中能查到 Bug point。
- payload 包含权限、来源、项目、产品。

### 步骤 6：实现禅道 Bug 手动同步

任务：

- Bug 详情页插入“同步 AI 索引”按钮。
- 点击后调用禅道扩展控制器。
- 禅道扩展控制器调用 AI 服务同步接口。

完成标准：

- 从 Bug 页面点击同步，AI 服务收到对象。

### 步骤 7：实现 Bug 相似检索

任务：

- 实现 `/api/v1/bugs/{id}/similar`。
- 根据当前 Bug chunk 查询 Qdrant。
- 过滤当前对象自身。
- 返回相似 Bug 列表。

完成标准：

- Bug 详情页能展示相似 Bug。

### 步骤 8：实现文档同步接口复用

任务：

- 复用对象同步接口支持 `object_type=doc`。
- 文档详情页插入同步按钮。
- 发送文档 payload。

完成标准：

- AI 服务能收到文档对象快照。

### 步骤 9：实现文档解析和切片

任务：

- HTML 转文本。
- 提取标题路径。
- 按标题和段落切片。
- 生成 chunk。

完成标准：

- Qdrant 中有多个 `doc:{id}:chunk:{n}`。
- chunk 保留 `heading_path`。

### 步骤 10：实现单文档问答

任务：

- 实现 `/api/v1/docs/{id}/ask`。
- Qdrant 检索当前文档 chunk。
- 调用 LLM 生成回答。
- 返回引用来源。

完成标准：

- 文档详情页提问能返回答案和引用来源。

### 步骤 11：实现图片 OCR

任务：

- 从禅道附件下载图片。
- OCR 提取文字。
- 生成 image chunk。
- 写入 Qdrant。

完成标准：

- Bug 截图错误码可以被相似检索命中。
- 文档图片 OCR 文本可以被文档问答引用。

### 步骤 12：实现索引状态和重试

任务：

- 实现索引状态接口。
- 实现失败重试接口。
- 禅道页面展示状态。

完成标准：

- 用户能看到“已索引/索引失败/索引中”。
- 管理员能重试失败任务。

## 17. 测试计划

### 17.1 单元测试

- Bug 文本构建。
- 文档 HTML 解析。
- 文档切片。
- chunk_id 生成。
- 权限 scope 生成。
- Qdrant payload 构建。
- LLM Gateway 模型别名解析。
- Embedding Gateway 失败重试。

### 17.2 集成测试

- 禅道 Bug 手动同步到 AI 服务。
- AI 服务 Worker 写入 Qdrant。
- Bug 相似检索返回结果。
- 文档同步后可问答。
- 图片 OCR 后可检索。
- 切换模型配置后业务模块无需修改。

### 17.3 权限测试

- 用户 A 无权限项目数据不能检索。
- 私有文档不应出现在无权限用户问答中。
- 删除对象后检索不到。

### 17.4 幂等测试

- 同一个对象重复同步不产生重复有效 point。
- 更新对象后旧 chunk 不再参与检索。
- Worker 重试不会重复创建错误数据。

## 18. MVP 验收清单

Bug：

- Bug 详情页有 AI 同步入口。
- Bug 可同步到 AI 服务。
- Bug 可写入 Qdrant。
- Bug 相似检索可用。
- 相似结果带来源链接。

文档：

- 文档详情页有 AI 同步入口。
- 文档可同步到 AI 服务。
- 文档按章节切片。
- 文档 chunk 可写入 Qdrant。
- 单文档问答可用。
- 回答带引用来源。

安全：

- API 使用 HMAC 鉴权并防止 Nonce 重放。
- `permission_scope` 只做向量候选预过滤。
- 所有候选来源在进入 Prompt 前通过禅道批量接口完成对象级权限校验。
- 权限不明确、权限服务不可用或 ACL 版本落后时默认拒绝。
- 无权限对象不进入 Prompt、不返回、不出现在已有结果中。
- Token 和 API Key 不暴露到前端。

运维：

- 有索引状态。
- 有失败重试。
- 有基础日志。
- 有 request_id。
- 业务模块不直接调用模型供应商 API。
- 模型切换只修改配置或 adapter，不修改 Bug/文档业务代码。

## 19. 后续扩展

完成 Bug 和文档后，按同样模式扩展：

- 客户反馈：`feedback:{id}:chunk:1`。
- 需求：`story:{id}:chunk:1`。
- 测试用例：`case:{id}:chunk:1`。
- 会议纪要：按议题切片。
- 项目周报：基于结构化数据和 RAG 证据生成报告。

优先顺序：

1. Bug AI 分析完整输出和仅添加评论的人工确认写回。
2. 客户反馈分类和 Bug/需求草稿。
3. 项目范围知识问答、统一多轮聊天和通用推荐。
4. 项目风险周报。
5. 模型管理后台、审计与用量页面。
6. 版本发布风险检查。
