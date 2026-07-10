# 基于禅道的 AI 项目管理与客户反馈分析系统项目文档

版本：v1.3
日期：2026-07-10
项目方向：基于中文开源项目管理平台禅道，建设独立 AI 分析服务，通过禅道扩展页面嵌入 AI 能力，实现项目流程把控、文档问答、客户反馈分析、Bug 智能分析和版本风险预警。

## 1. 项目概述

本项目不从零开发完整项目管理软件，而是采用成熟中文开源平台作为业务底座，重点建设 AI 能力。

推荐方案：

```text
禅道开源版
  + 禅道扩展页面
  + 独立 AI 服务
  + 向量数据库
  + 项目/文档/反馈/Bug 分析能力
```

禅道负责项目管理基础能力：

- 产品、项目、执行、需求、任务、Bug、测试用例、反馈、文档、权限、通知。

AI 服务负责智能分析能力：

- 客户反馈分类、去重、严重程度判断。
- Bug 相似问题检索、复现步骤结构化、优先级建议。
- 项目进度风险、流程异常、版本质量风险分析。
- 文档问答、需求缺口检查、PRD 转任务/测试点。
- 自动生成日报、周报、发布说明和客户问题报告。

禅道扩展负责连接两者：

- 在禅道现有页面中增加 AI 按钮、AI 面板、AI 报告入口。
- 在登录后的业务页面增加固定于右下角的机器人入口，打开全局 AI 客服弹窗。
- 调用独立 AI 服务。
- 展示 AI 分析结果。
- 经人工确认后写回禅道评论、自定义字段、任务、Bug 或需求。

## 2. 背景与目标

### 2.1 背景

很多研发团队已经有项目管理工具，但仍存在这些问题：

- 项目经理难以及时发现项目延期、流程卡点和版本风险。
- 客户反馈分散在工单、聊天群、邮件、表单、客服系统中，难以归类和追踪。
- Bug 数量多时，重复问题、相似问题和优先级判断依赖人工经验。
- 文档沉淀在 Wiki、PRD、测试报告、会议纪要中，但检索和复用效率低。
- 管理者需要周报、风险报告、版本质量报告，但人工整理成本高。

传统项目管理软件解决了“记录”和“流转”，但没有很好解决“理解”和“分析”。

### 2.2 项目目标

建设一个基于禅道的 AI 分析系统，让团队可以：

- 通过 AI 把控项目进度和流程风险。
- 通过 AI 分析客户反馈，自动归类、去重、转需求或 Bug。
- 通过 AI 理解项目文档，支持问答、总结、缺口检查。
- 通过 AI 分析 Bug，识别相似问题、建议优先级和负责人。
- 通过 AI 输出项目日报、周报、版本风险报告和客户声音报告。

### 2.3 不做什么

第一阶段不做：

- 不从零开发项目管理系统。
- 不深度魔改禅道核心源码。
- 不把大模型、向量库、任务队列直接塞进禅道 PHP 应用。
- 不让 AI 自动关闭 Bug、自动改状态、自动发布版本。
- 不做全量效能平台和复杂 BI。
- 不做客户反馈 AI 分类、项目风险周报和项目范围知识问答，这些能力进入 V1.1。
- 不做模型管理后台，模型参数先通过 AI 服务端配置注入。
- 不写回自定义字段，用户确认后仅允许添加评论。

第一阶段 AI 只做“Bug 与单文档辅助分析 + 全局 AI 客服 + 对象级权限校验 + 人工确认”。

## 3. 底座选型

### 3.1 为什么选择禅道

禅道适合作为本项目底座，原因是：

- 中文友好，适合国内团队和客户。
- 开源版可私有化部署。
- 研发管理模型完整，覆盖产品、项目、需求、任务、Bug、测试、反馈、文档。
- Bug 和测试管理能力较成熟，适合质量分析。
- 支持扩展机制，可以在不改核心源码的情况下增加页面、模型、控制器、配置、语言包和前端资源。
- 源码中已经存在 `module/ai`，包含模型配置、Prompt、数据源、表单注入等 AI 基础能力，可作为参考或复用。

### 3.2 竞品/备选对比

| 平台 | 中文友好度 | 开源情况 | 适合程度 | 结论 |
| --- | --- | --- | --- | --- |
| 禅道开源版 | 高 | 开源版可用，支持私有部署 | 很适合研发项目、Bug、测试、反馈、文档 | 首选 |
| Redmine | 中 | GPL，成熟开源 | 项目和 Issue 稳定，但 UI 传统，中文体验一般 | 备选 |
| MantisBT | 中 | 开源 | Bug 跟踪强，项目/文档/流程弱 | 只适合缺陷系统 |
| GitLab CE | 中 | 开源核心 | 代码、Issue、CI 强，项目管理广度不足 | 适合代码研发团队 |
| OpenProject | 低到中 | 开源核心 | 项目管理强，但中文体验不理想 | 不作为首选 |
| Plane/Taiga | 中到低 | 开源 | UI 现代或敏捷友好，但中文和企业流程弱 | 不作为首选 |

### 3.3 禅道技术架构简述

禅道是 PHP 单体 Web 应用，技术结构大致如下：

```text
浏览器
  ↓
Apache/Nginx + PHP
  ↓
www/index.php
  ↓
zentaoPHP Router
  ↓
module/{模块}/control.php
  ↓
module/{模块}/model.php
  ↓
DAO + MySQL
  ↓
module/{模块}/view 或 ui
```

核心目录：

```text
framework/    zentaoPHP 框架核心
module/       业务模块，例如 bug、task、story、doc、feedback、project、ai
extension/    扩展目录
config/       配置
db/           数据库脚本
www/          Web 入口和静态资源
```

## 4. 总体产品定位

### 4.1 一句话定位

基于禅道的 AI 项目管理助手，帮助研发团队自动分析项目风险、文档内容、客户反馈和 Bug 问题。

### 4.2 目标用户

- 10-200 人的软件研发团队。
- 使用禅道或计划私有化部署项目管理系统的团队。
- 有大量客户反馈、Bug、文档和项目跟踪需求的团队。
- 外包交付团队、企业软件团队、SaaS 团队、智能硬件研发团队。

### 4.3 核心角色

- 项目经理：关注进度、风险、阻塞、资源和交付。
- 产品经理：关注需求、客户反馈、优先级和版本规划。
- 测试负责人：关注 Bug、测试用例、版本质量和缺陷趋势。
- 开发负责人：关注任务分派、Bug 根因、重复问题和技术风险。
- 客服/客户成功：关注客户反馈归类、响应状态和问题闭环。
- 管理者：关注项目整体健康度、客户声音和质量趋势。

## 5. 总体架构

### 5.1 架构图

```text
                  ┌────────────────────┐
                  │      用户浏览器      │
                  └─────────┬──────────┘
                            │
                            ▼
┌────────────────────────────────────────────────┐
│                    禅道                         │
│  项目 / 需求 / 任务 / Bug / 测试 / 文档 / 反馈   │
│                                                │
│  extension/custom                              │
│  - AI 按钮                                     │
│  - AI 面板                                     │
│  - AI 报告入口                                 │
│  - 调用 AI 服务                                │
└───────────────┬────────────────────────────────┘
                │ API / Webhook / DB 只读同步
                ▼
┌────────────────────────────────────────────────┐
│                  AI 服务                        │
│  ai-api      对外接口                           │
│  ai-worker   异步任务                           │
│  scheduler   定时分析                           │
│  parser      文档/附件解析                       │
│  llm-gateway 模型网关                           │
│  rules       规则引擎                           │
└───────┬─────────────┬─────────────┬────────────┘
        │             │             │
        ▼             ▼             ▼
┌────────────┐ ┌────────────┐ ┌─────────────────┐
│ 业务数据库  │ │  向量数据库 │ │ 大模型/私有模型  │
│ PostgreSQL │ │ Qdrant 等   │ │ OpenAI/通义等   │
└────────────┘ └────────────┘ └─────────────────┘
```

### 5.2 部署建议

MVP 部署：

```text
Docker Compose
├── zentao        # 禅道
├── ai-api        # AI 服务 API
├── ai-worker     # 后台任务
├── postgres      # AI 服务业务库
├── redis         # 队列/缓存
└── qdrant        # 向量数据库
```

企业部署：

```text
Kubernetes / Docker Compose
├── 禅道应用
├── AI API 多实例
├── AI Worker 多实例
├── PostgreSQL/MySQL
├── Redis
├── Qdrant/Milvus
├── MinIO
├── 模型网关
└── 审计与监控
```

## 6. 禅道扩展设计

### 6.1 什么是禅道扩展页面

禅道扩展页面是指：不直接修改禅道原始源码，而是通过 `extension/custom/...` 扩展目录，在原有页面中插入自己的 AI 按钮、面板、弹窗、菜单或报告入口。

例如 Bug 详情页原本展示：

```text
Bug 标题
复现步骤
严重程度
负责人
评论
```

通过扩展后可增加：

```text
AI 分析
- 相似 Bug
- 建议优先级
- 建议负责人
- 可能影响模块
- 复现步骤整理
```

### 6.2 可扩展能力

| 扩展类型 | 能力 | AI 用途 |
| --- | --- | --- |
| `control` | 扩展控制器方法 | 增加 AI 分析接口、按钮回调 |
| `model` | 扩展业务模型逻辑 | 获取 Bug/任务/文档数据，调用 AI 服务 |
| `ui` / `view` | 扩展页面展示 | 插入 AI 面板、按钮、弹窗 |
| `lang` | 扩展语言包 | 增加中文菜单和提示 |
| `config` | 扩展配置 | AI 服务地址、开关、菜单配置 |
| `css` / `js` | 扩展前端资源 | AI 面板交互、异步请求 |
| `hook` | 在已有逻辑中插入代码 | 企业版或支持工作流 hook 的场景可用于保存后触发同步 |
| `.api` | API 扩展 | 对外暴露 AI 相关接口 |

### 6.3 扩展目录示例

```text
extension/custom/common/ext/config/ai.php
extension/custom/common/ext/lang/zh-cn/ai.php

extension/custom/bug/ext/ui/view.ai.html.hook.php
extension/custom/bug/ext/control/analyze.php
extension/custom/bug/ext/model/ai.php

extension/custom/doc/ext/ui/view.ai.html.hook.php
extension/custom/doc/ext/control/ask.php
extension/custom/doc/ext/model/ai.php

extension/custom/project/ext/ui/view.ai.html.hook.php
extension/custom/project/ext/control/risk.php

extension/custom/feedback/ext/ui/view.ai.html.hook.php
extension/custom/feedback/ext/control/classify.php
```

### 6.4 禅道扩展应该做什么

禅道扩展只做轻量集成：

- 展示 AI 入口。
- 获取当前页面对象 ID。
- 调用 AI 服务接口。
- 展示 AI 分析结果。
- 支持用户确认后写回禅道。
- 记录调用日志和错误提示。

### 6.5 不建议放进禅道扩展的内容

这些能力应放在独立 AI 服务：

- 向量数据库和 RAG 检索。
- 文档解析、OCR、附件解析。
- 大模型网关。
- 客户反馈批量聚类。
- 项目风险定时分析。
- 异步任务队列。
- 复杂报表和指标预计算。
- 多租户模型权限和审计。

### 6.6 Bug 和文档同步触发约束

当前禅道代码支持通过 `extension/custom` 增加控制器方法和页面 hook，项目中已经有 Bug AI 扩展示例：

```text
extension/custom/bug/ext/control/aianalyze.php
extension/custom/bug/ext/ui/view.ai.html.hook.php
```

因此可以新增 `bug-aiSync`、`doc-aiSync` 等扩展方法，用于同步 Bug 和文档到 AI 服务。

需要注意：

- Bug 创建、编辑、删除等流程中存在 `executeHooks` 调用，但 Open 版框架层会直接返回空结果，不能把 MVP 的自动同步依赖在该机制上。
- 文档模块创建、编辑、删除流程没有同样稳定的 after hook 入口。
- 覆盖 Bug/Doc 的 `create`、`edit`、`delete` 方法可以实现实时触发，但会增加禅道升级维护成本。

MVP 推荐策略：

- Bug 和文档详情页提供手动同步按钮。
- AI 服务或禅道扩展提供定时增量补偿扫描。
- 禅道只调用 AI 服务同步接口，不直接写向量数据库。
- 实时自动触发作为第二阶段增强，在手动同步和补偿机制稳定后再实现。

## 7. AI 服务设计

### 7.1 AI 服务职责

AI 服务负责所有智能分析和数据处理：

- 同步禅道数据。
- 清洗和标准化数据。
- 文档切片和向量化。
- 语义搜索和相似度匹配。
- 调用大模型。
- 执行规则判断。
- 生成分析报告。
- 存储 AI 结果。
- 向禅道扩展提供 API。

### 7.2 AI 服务模块

| 模块 | 说明 |
| --- | --- |
| `ai-api` | 提供 HTTP API，供禅道扩展和外部系统调用 |
| `ai-worker` | 执行异步任务，例如向量化、批量分析、报告生成 |
| `scheduler` | 定时生成日报、周报、版本风险报告 |
| `sync-service` | 从禅道同步项目、需求、任务、Bug、文档、反馈 |
| `parser-service` | 解析 Markdown、HTML、Word、PDF、日志、附件文本 |
| `embedding-service` | 生成向量并写入向量数据库 |
| `retrieval-service` | RAG 检索、相似 Bug、相似反馈 |
| `llm-gateway` | 统一接入 OpenAI、Azure OpenAI、通义、文心、DeepSeek、私有模型 |
| `rule-engine` | 项目流程规则、SLA 规则、版本发布规则 |
| `report-service` | 生成项目报告、客户反馈报告、质量报告 |
| `audit-service` | 记录 AI 调用、数据来源、用户确认动作 |

### 7.3 AI 服务 API 示例

```text
POST /api/bug/{id}/analyze
POST /api/bug/{id}/similar
POST /api/doc/{id}/summary
POST /api/doc/{id}/ask
POST /api/feedback/{id}/classify
POST /api/project/{id}/risk-report
POST /api/release/{id}/quality-report
POST /api/sync/object
POST /api/webhook/zentao
```

## 8. 数据同步设计

### 8.1 数据来源

从禅道同步的数据包括：

- 项目：项目名称、计划、里程碑、状态、成员。
- 需求：标题、描述、验收标准、优先级、状态、版本。
- 任务：标题、描述、负责人、状态、工时、截止时间。
- Bug：标题、复现步骤、严重程度、优先级、影响版本、修复版本、状态。
- 测试用例：标题、步骤、预期结果、执行结果。
- 文档：标题、正文、附件、目录、关联项目。
- 客户反馈：反馈内容、客户、来源、模块、处理状态。
- 评论：关键讨论、结论、变更说明。
- 操作记录：创建、修改、状态流转、指派、关闭。

### 8.2 同步方式

优先级：

1. 禅道扩展控制器主动调用 AI 服务。
2. 禅道 API。
3. 禅道 Webhook。
4. 数据库只读同步。
5. 定时全量/增量扫描。

不建议 AI 服务直接写禅道核心表。写回禅道时优先通过 API 或禅道扩展控制器。

MVP 同步方式：

- Bug：`bug-aiSync` 手动同步 + 按 `openedDate`、`lastEditedDate`、`deleted` 定时补偿。
- 文档：`doc-aiSync` 手动同步 + 按 `addedDate`、`editedDate`、`deleted` 定时补偿。
- 保存后实时同步暂不作为强依赖，避免依赖 Open 版不可用的 `executeHooks` 或覆盖复杂控制器流程。

### 8.3 同步策略

```text
创建数据
  ↓
手动同步或增量扫描同步到 AI 服务

文本内容更新
  ↓
重新抽取文本并重新向量化

结构化字段更新
  ↓
只更新 metadata

Webhook 丢失
  ↓
定时任务补偿同步
```

第二阶段如需要更实时的体验，再在禅道扩展中覆盖 Bug/Doc 的保存流程，在保存成功后调用 AI 服务同步接口，并保留定时补偿兜底。

## 9. 向量数据库设计

### 9.1 什么数据需要存向量库

向量库只存需要语义搜索、相似匹配、问答和聚类的文本数据。

| 数据 | 是否入向量库 | 用途 |
| --- | --- | --- |
| 需求标题、描述、验收标准 | 是 | 相似需求、PRD 问答、任务拆解 |
| Bug 标题、复现步骤、实际结果、期望结果 | 是 | 相似 Bug、重复问题、根因分析 |
| Bug 截图 OCR 文本和图片描述 | 是 | 错误截图检索、相似异常识别 |
| 客户反馈原文 | 是 | 反馈分类、去重、客户声音聚类 |
| 工单/客服记录 | 是 | 相似案例、问题归因 |
| 项目文档/Wiki | 是 | 文档问答、知识检索 |
| 文档图片 OCR 文本和图片描述 | 是 | 流程图、原型图、截图问答 |
| 会议纪要 | 是 | 待办提取、决策追踪 |
| 测试用例步骤和预期结果 | 是 | 推荐测试点、失败用例转 Bug |
| 发布说明/变更记录 | 是 | 版本问答、发布风险分析 |
| 评论/讨论记录 | 部分入库 | 用于上下文，但要过滤噪音 |
| 附件原文件、图片原文件 | 否 | 存对象存储或禅道附件，向量库只存提取文本和引用 |
| 项目状态、负责人、时间、优先级 | 否 | 放关系型数据库 |
| 操作日志 | 通常否 | 只做审计或流程统计 |

### 9.2 什么时候入向量库

创建时入库：

- 新需求。
- 新 Bug。
- 新文档。
- 新客户反馈。
- 新测试用例。
- 新会议纪要。

文本更新时重新向量化：

- 标题变更。
- 描述变更。
- 复现步骤变更。
- 文档正文变更。
- 客户反馈正文变更。
- 测试步骤变更。
- 图片重新上传或 OCR/图片描述变更。

结构化字段更新时只更新 metadata：

- 状态。
- 负责人。
- 优先级。
- 所属项目。
- 版本。
- 模块。

删除、关闭、归档时：

- 不一定物理删除向量。
- 优先更新 metadata，例如 `status=closed`、`deleted=true`、`archived=true`。
- 检索时通过 metadata 过滤。

### 9.3 向量数据结构

```json
{
  "id": "bug:123:chunk:1",
  "tenant_id": "company_001",
  "source": "zentao",
  "object_type": "bug",
  "object_id": 123,
  "chunk_index": 1,
  "project_id": 8,
  "product_id": 3,
  "module_name": "登录",
  "title": "登录后跳转失败",
  "content": "复现步骤：...",
  "source_url": "/bug-view-123.html",
  "created_at": "2026-07-09T10:00:00+08:00",
  "updated_at": "2026-07-09T10:00:00+08:00",
  "status": "active",
  "permission_scope": ["project:8", "product:3"]
}
```

向量库一条记录对应一个 chunk。业务关系、分析结果、采纳状态、模型调用日志放关系型数据库，不放向量库。

推荐 payload 字段：

| 字段 | 说明 |
| --- | --- |
| `tenant_id` | 租户 ID |
| `object_type` | 对象类型，例如 bug、doc、feedback、story、case |
| `object_id` | 禅道对象 ID |
| `chunk_type` | text、image、comment、attachment |
| `chunk_index` | 同一对象下的切片序号 |
| `title` | 对象标题 |
| `content` | 用于向量化和 RAG 的文本 |
| `heading_path` | 文档章节路径，文档类 chunk 必填 |
| `file_id` | 图片或附件在禅道中的文件 ID |
| `source_url` | 原始对象或附件跳转链接 |
| `project_id` | 项目 ID，用于过滤 |
| `product_id` | 产品 ID，用于过滤 |
| `module_id` | 模块 ID，用于过滤 |
| `status` | 对象状态 |
| `permission_scope` | 权限范围 |
| `deleted` | 是否删除 |

图片 chunk 示例：

```json
{
  "id": "bug:123:image:1",
  "tenant_id": "company_001",
  "source": "zentao",
  "object_type": "bug",
  "object_id": 123,
  "chunk_type": "image",
  "file_id": 889,
  "title": "登录后无法进入首页 - 附图1",
  "content": "OCR文本：403 Forbidden。图片描述：登录后页面空白，浏览器控制台显示权限错误。",
  "source_url": "/file-read-889.html",
  "project_id": 8,
  "product_id": 3,
  "status": "active",
  "permission_scope": ["project:8", "product:3"],
  "deleted": false
}
```

### 9.4 权限过滤

RAG 检索必须做权限过滤：

- 用户只能检索自己在禅道有权限访问的项目、产品、文档、Bug。
- 向量库 metadata 中必须保存租户、项目、产品、权限范围。
- 查询时先传入用户身份和权限范围，再检索。
- 不允许 AI 回答用户无权限访问的内容。

### 9.5 分段策略

总体原则：

- 短对象不强行切分，整条对象作为一个 chunk。
- 长文档按标题层级和段落切分。
- 图片不直接入向量库，先 OCR 和生成图片描述，再把文本作为 image chunk 入库。
- 每个 chunk 必须带来源对象、权限范围、标题和跳转链接。

| 类型 | chunk 策略 | 建议大小 |
| --- | --- | --- |
| Bug | 整条 Bug 为主，关键评论可单独切片 | 300-1200 中文字 |
| 客户反馈 | 整条反馈为主 | 100-800 中文字 |
| 需求 | 单条需求为主，长 PRD 按章节切片 | 300-1200 中文字 |
| 测试用例 | 单条测试用例为主 | 200-800 中文字 |
| 文档 | 按 H1/H2/H3 和段落切片 | 500-1000 中文字 |
| 会议纪要 | 按议题、决策、待办切片 | 500-1000 中文字 |
| 图片 | OCR 文本 + 图片描述 + 所属对象上下文 | 100-800 中文字 |
| 日志附件 | 先摘要，再入摘要 chunk | 不建议原始大日志全量入库 |

文档切片规则：

- 优先按标题层级切分。
- 章节超过 1000 中文字时按段落继续切分。
- 段落仍过长时按句子切分。
- chunk 之间保留 100-150 字 overlap。
- 每个 chunk 前补充文档标题、章节路径、所属项目或产品。

Bug chunk 示例：

```text
标题：登录后无法进入首页
产品：CRM
模块：登录
严重程度：严重
优先级：P1

复现步骤：
1. 打开登录页
2. 输入账号密码
3. 点击登录

实际结果：
页面空白，控制台报 403。

期望结果：
登录成功后进入首页。

关键评论：
开发确认与权限回调有关。
```

文档 chunk 示例：

```text
文档：登录模块 PRD
章节：登录模块 PRD > 异常流程 > Token 过期
内容：
当用户 Token 过期时，系统应尝试刷新 Token。刷新失败后跳转登录页，并提示用户重新登录。
```

### 9.6 图片和附件处理

Bug 和文档中的图片需要作为独立 image chunk 处理。

处理流程：

```text
图片上传或文档更新
  ↓
提取图片文件
  ↓
OCR 识别图片文字
  ↓
可选：多模态模型生成图片描述
  ↓
拼接图片文字、图片描述、所属对象上下文
  ↓
生成 embedding
  ↓
写入向量库
```

V1.1 图片能力：

- OCR 识别截图中的错误码、异常文本、按钮文案、接口路径。
- 图片 chunk 保留 `file_id`、`source_url`、所属对象和权限范围。
- 文档图片保留章节路径。
- AI 回答引用图片时，可跳回原 Bug 附件或文档图片。

增强版图片能力：

- UI 截图识别页面异常、弹窗、按钮状态。
- 流程图总结流程节点。
- 架构图总结系统组件关系。
- 表格截图转成结构化表格。
- 报错截图提取错误栈和关键异常。

安全要求：

- 图片原文件不存向量库。
- 图片 OCR 和图片描述继承原对象权限。
- OCR 结果保留置信度。
- 含手机号、邮箱、Token、Cookie、身份证等敏感信息的截图必须先脱敏。
- OCR 置信度低时，AI 回答应提示“图片文字识别可能不准确”。

## 10. 核心 AI 功能

### 10.1 客户反馈 AI

目标：把分散客户反馈转为可管理、可追踪、可分析的需求和 Bug。

功能：

- 自动分类：Bug、需求、咨询、投诉、环境问题、使用问题。
- 自动摘要：提取客户问题、影响范围、期望结果。
- 严重程度判断：结合客户等级、关键词、影响范围、历史问题。
- 相似反馈合并：识别重复反馈。
- 关联已有 Bug/需求：推荐可关联事项。
- 转 Bug/需求建议：生成标题、描述、复现步骤、优先级。
- 客户声音报告：按客户、版本、模块、行业聚类。

落地入口：

- 反馈详情页 AI 面板。
- 反馈列表批量分析按钮。
- 客户反馈周报。

### 10.2 Bug AI

目标：提升缺陷处理效率和版本质量判断能力。

功能：

- 相似 Bug 检索。
- 重复 Bug 识别。
- 复现步骤结构化。
- 严重程度和优先级建议。
- 可能负责人建议。
- 影响模块判断。
- 日志/错误栈摘要。
- 重开 Bug 原因分析。
- 版本质量风险分析。

落地入口：

- Bug 创建页：AI 辅助整理标题、步骤、优先级。
- Bug 详情页：相似 Bug、处理建议、风险说明。
- Bug 列表页：批量相似合并和风险筛选。

### 10.3 文档 AI

目标：让项目文档可问答、可检查、可转化。

功能：

- 文档摘要。
- 文档问答。
- 需求缺口检查：背景、目标、边界、异常流程、验收标准。
- 文档与任务一致性检查。
- PRD 转需求/任务/测试点。
- 会议纪要转待办。
- 发布说明生成。

落地入口：

- 文档详情页 AI 总结/问答。
- 文档编辑页 AI 检查。
- 项目文档库全局问答。

### 10.4 项目流程 AI

目标：帮助项目经理提前发现风险。

功能：

- 延期风险识别。
- 长时间未更新任务识别。
- 阻塞事项识别。
- 需求无验收标准识别。
- Bug 无复现步骤识别。
- 版本发布前 P0/P1 Bug 检查。
- 跨团队依赖风险。
- 项目日报/周报生成。

落地入口：

- 项目概览 AI 风险面板。
- 每日/每周自动报告。
- 版本发布前质量检查。

### 10.5 版本发布 AI

目标：降低带病发布风险。

功能：

- 未关闭严重 Bug 检查。
- 本版本高频客户反馈分析。
- 最近重开 Bug 分析。
- 测试覆盖情况总结。
- 变更范围总结。
- 发布说明生成。
- 发布风险等级建议。

落地入口：

- 版本详情页。
- 发布检查清单。
- 发布前自动报告。

## 11. 首批 MVP 范围

第一版只做 2 个对象分析闭环、1 个全局 AI 客服入口和一组必需的基础能力。

### 11.1 Bug 智能分析

能力：

- 相似 Bug 检索。
- 复现步骤整理。
- 优先级/严重程度建议。
- 分析结果展示在 AI 面板；用户确认后可添加评论。

### 11.2 单文档摘要与问答

能力：

- Bug 与文档索引。
- 单文档摘要。
- 单文档问答。
- 引用来源和无证据提示。

### 11.3 必需基础能力

能力：

- HMAC 服务鉴权、Nonce 防重放、租户隔离和幂等。
- 禅道批量对象级权限回调；`permission_scope` 只做候选预过滤。
- PostgreSQL、Redis/Celery、Qdrant 和基础模型网关。
- 同步状态、任务查询、失败重试、调用审计和基础运维日志。
- AI 结果查询、采纳、驳回和仅添加评论的写回。

### 11.4 全局 AI 客服

能力：

- 登录后的业务页面右下角统一展示机器人图标，点击后打开聊天弹窗或右侧抽屉。
- Bug 详情页围绕当前 Bug 问答，文档详情页围绕当前文档问答，反馈详情页围绕当前反馈问答，其他页面仅回答经过审核的平台使用手册和 FAQ。
- MVP 的反馈会话只读取当前反馈字段和已关联的授权资料；反馈自动分类和生成 Bug/需求草稿仍在 V1.1 实现。
- 支持流式回答、连续追问、引用来源、最近会话恢复、停止生成和清空会话。
- 页面范围由禅道服务端根据当前路由生成并签名，前端不能自行扩大到其他项目或对象。
- AI 服务不可用时只提示客服暂不可用，不影响当前页面原有功能。

客户反馈分类、项目风险周报、项目范围及跨对象问答、通用推荐和模型管理后台进入 V1.1。

## 12. 用户流程

### 12.1 Bug 分析流程

```text
用户打开 Bug 详情页
  ↓
点击 AI 分析
  ↓
禅道扩展调用 AI 服务
  ↓
AI 服务读取 Bug 内容和相似向量
  ↓
大模型生成分析结论
  ↓
禅道页面展示结果
  ↓
用户确认后写入评论/字段
```

### 12.2 客户反馈转 Bug 流程

```text
客户反馈进入禅道
  ↓
AI 服务同步反馈文本
  ↓
自动分类和相似检索
  ↓
生成 Bug/需求建议
  ↓
产品/测试人员确认
  ↓
创建 Bug/需求
  ↓
保留原反馈关联关系
```

### 12.3 文档问答流程

```text
文档创建或更新
  ↓
AI 服务解析文本
  ↓
切片并写入向量库
  ↓
用户在文档/项目页面提问
  ↓
按权限检索相关内容
  ↓
大模型生成回答
  ↓
展示引用来源
```

### 12.4 项目周报流程

```text
每周定时任务
  ↓
同步项目数据
  ↓
计算延期、阻塞、Bug、反馈指标
  ↓
检索关键文档和讨论
  ↓
生成 AI 周报
  ↓
推送到禅道项目页/邮件/飞书/企微
```

## 13. 数据模型草案

AI 服务业务库建议保存这些表：

| 表 | 用途 |
| --- | --- |
| `sync_objects` | 禅道对象同步状态 |
| `documents` | 解析后的文档主记录 |
| `document_chunks` | 文档切片记录 |
| `ai_analysis_results` | AI 分析结果 |
| `ai_reports` | 周报、风险报告、质量报告 |
| `ai_tasks` | 异步任务 |
| `feedback_clusters` | 客户反馈聚类 |
| `similarity_links` | 相似 Bug/反馈/需求关系 |
| `llm_calls` | 模型调用审计 |
| `user_permissions_cache` | 权限缓存 |
| `ai_confirmations` | 用户确认/采纳/驳回记录 |

分析结果示例：

```json
{
  "object_type": "bug",
  "object_id": 123,
  "analysis_type": "bug_triage",
  "summary": "该问题影响登录后跳转，疑似与权限回调有关。",
  "risk_level": "high",
  "suggested_priority": "P1",
  "similar_objects": [
    {"type": "bug", "id": 98, "score": 0.91}
  ],
  "evidence": [
    {"type": "bug", "id": 98, "field": "steps"},
    {"type": "doc", "id": 21, "field": "content"}
  ],
  "status": "pending_confirmation"
}
```

## 14. 权限与安全

### 14.1 权限原则

- AI 不能突破禅道权限。
- 用户只能分析自己有权限访问的数据。
- AI 回答必须带引用来源。
- 敏感数据调用外部模型前需要脱敏。
- 企业私有化场景支持内网模型或私有模型。

### 14.2 安全措施

- API Token/JWT 鉴权。
- 禅道到 AI 服务的服务间签名。
- 租户隔离。
- 向量库 metadata 权限过滤。
- 模型调用日志。
- 敏感字段脱敏。
- 可配置是否允许调用公网模型。
- 所有 AI 建议需人工确认后才能写回核心业务数据。

## 15. 技术选型建议

### 15.1 禅道侧

- 禅道开源版。
- 使用 `extension/custom` 扩展。
- 尽量复用 `module/ai` 的模型配置、Prompt 和表单注入思路。
- 不改 `framework/` 和核心 `module/`。

### 15.2 AI 服务侧

推荐技术栈：

| 层 | 建议 |
| --- | --- |
| API 服务 | Python FastAPI 或 Node.js NestJS |
| Worker | Celery/RQ/Sidekiq 类任务队列，或 BullMQ |
| 数据库 | PostgreSQL |
| 队列 | Redis |
| 向量库 | Qdrant，后续可选 Milvus |
| 文档解析 | unstructured、pandoc、pdfplumber、OCR 服务 |
| 模型网关 | OpenAI Compatible API 适配层 |
| 部署 | Docker Compose 起步，后续 Kubernetes |

如果团队偏 PHP，也不建议 AI 服务继续用 PHP。AI 服务更适合 Python 或 Node.js。

## 16. AI 服务技术方案与开发规范

### 16.1 技术目标

AI 服务是一个独立部署的后端系统，不依赖禅道 PHP 运行时。它负责数据同步、文本解析、向量化、语义检索、大模型调用、规则分析、报告生成和结果存储。

技术目标：

- 与禅道低耦合，通过 API/Webhook/只读同步交互。
- 支持私有化部署。
- 支持多模型接入，优先兼容 OpenAI API 格式。
- 支持异步任务，避免长耗时分析阻塞禅道页面。
- 支持租户隔离、权限过滤、审计追踪。
- 所有 AI 结论必须可回溯到来源数据。

### 16.2 推荐技术栈

| 层 | 推荐技术 | 说明 |
| --- | --- | --- |
| API 服务 | Python FastAPI | 生态适合 AI、RAG、文档解析，开发效率高 |
| 异步任务 | Celery + Redis | 文档解析、向量化、批量报告、定时分析 |
| 定时任务 | Celery Beat 或 APScheduler | 周报、日报、增量同步、补偿任务 |
| 业务数据库 | PostgreSQL | 存同步状态、AI 结果、报告、审计日志 |
| 向量数据库 | Qdrant | 部署简单、metadata filter 好用，适合 MVP |
| 缓存/队列 | Redis | 队列、锁、缓存、幂等控制 |
| 对象存储 | MinIO，可选 | 存附件解析中间文件、导出报告 |
| 文档解析 | unstructured、pypdf、python-docx、BeautifulSoup | 解析 PDF、Word、HTML、Markdown |
| 模型调用 | OpenAI SDK 或 HTTP Client | 通过 OpenAI Compatible 协议接多模型 |
| 监控 | Prometheus + Grafana，可选 | API、Worker、模型调用、任务失败率 |
| 日志 | JSON 日志 + Loki/ELK，可选 | 调试、审计、故障追踪 |

MVP 最小组合：

```text
FastAPI
PostgreSQL
Redis
Celery
Qdrant
OpenAI Compatible LLM
Docker Compose
```

### 16.3 服务模块划分

```text
ai-service/
├── apps/
│   ├── api/             # FastAPI HTTP 接口
│   ├── worker/          # Celery Worker
│   └── scheduler/       # 定时任务
├── core/
│   ├── config.py        # 配置
│   ├── security.py      # 鉴权、签名、脱敏
│   ├── logging.py       # 日志
│   └── errors.py        # 错误码
├── modules/
│   ├── zentao_sync/     # 禅道数据同步
│   ├── parser/          # 文档/附件解析
│   ├── embedding/       # 向量化
│   ├── retrieval/       # 检索和重排
│   ├── llm/             # 模型网关
│   ├── bug_ai/          # Bug 分析
│   ├── feedback_ai/     # 客户反馈分析
│   ├── doc_ai/          # 文档问答
│   ├── project_ai/      # 项目风险分析
│   └── report/          # 报告生成
├── repositories/        # 数据库访问
├── schemas/             # Pydantic 请求/响应模型
├── migrations/          # Alembic 迁移
└── tests/
```

### 16.4 API 设计规范

API 使用 REST 风格，统一 JSON 返回。

路径规范：

```text
/api/v1/{resource}/{id}/{action}
```

示例：

```text
POST /api/v1/bugs/123/analyze
POST /api/v1/bugs/123/similar
POST /api/v1/docs/88/ask
POST /api/v1/feedback/56/classify
POST /api/v1/projects/9/risk-reports
POST /api/v1/sync/zentao/object
GET  /api/v1/tasks/{task_id}
```

统一响应：

```json
{
  "code": 0,
  "msg": "success",
  "data": {}
}
```

错误响应：

```json
{
  "code": 40302,
  "msg": "PERMISSION_DENIED",
  "data": {
    "request_id": "req_20260709_002",
    "details": {}
  }
}
```

长耗时接口不直接同步返回结果，返回任务 ID：

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

### 16.5 鉴权与服务间调用规范

禅道扩展调用 AI 服务时必须带鉴权信息：

```http
X-Client-Id: zentao_company_001
X-Tenant-Id: company_001
X-Actor-Type: user
X-Zentao-User: zhangsan
X-Timestamp: 1783580000
X-Nonce: 5f7e3198-0a92-4dee-96fa-c12bb34fd509
X-Request-Id: req_xxx
X-Body-SHA256: 51e7f...
X-Signature: sha256=8d80a...
```

要求：

- `X-Client-Id` 对应 AI 服务保存的 Client 和轮换密钥。
- `X-Zentao-User` 标识当前禅道用户。
- `X-Tenant-Id` 标识租户。
- `X-Signature` 按《AI服务接口文档》的 canonical request 使用 HMAC-SHA256 计算。
- `X-Timestamp + X-Nonce` 防止重放攻击，修改类接口还必须携带 `X-Idempotency-Key`。
- AI 服务根据用户权限过滤数据，不信任前端传入的对象内容。

请求头、签名、响应结构和错误码均以《AI服务接口文档》为唯一契约。

### 16.6 数据同步规范

同步对象统一抽象为 `source_object`：

```json
{
  "source": "zentao",
  "tenant_id": "company_001",
  "object_type": "bug",
  "object_id": "123",
  "version": "2026-07-09T10:30:00+08:00",
  "event": "created|updated|deleted|closed",
  "payload": {}
}
```

同步方式：

- 实时：Webhook 或禅道扩展主动调用。
- 准实时：定时拉取最近更新时间大于 `last_sync_time` 的数据。
- 补偿：每日校验最近 7 天变更数据。

幂等规则：

- 同一个 `tenant_id + object_type + object_id + version` 只能处理一次。
- Worker 失败可重试。
- 重试不能重复创建向量和分析结果。

### 16.7 文本抽取与切片规范

统一文本格式：

```text
标题：登录后跳转失败
类型：Bug
项目：CRM 系统
模块：登录
正文：
复现步骤...
期望结果...
实际结果...
```

切片规则：

- 短文本，如 Bug、反馈、测试用例：通常整条作为一个 chunk。
- 长文档：按标题层级切片。
- 每个 chunk 建议 500-1000 中文字。
- chunk 之间保留 100-150 字 overlap。
- chunk 必须保留标题、对象类型、项目、模块等上下文。

不入库内容：

- 空文本。
- 纯格式文本。
- 无意义评论，如“收到”“已处理”。
- 权限不可确认的数据。

### 16.8 向量库规范

集合命名：

```text
tenant_{tenant_id}_knowledge
```

或多租户统一集合：

```text
zentao_knowledge
```

MVP 建议使用统一集合，通过 metadata 过滤租户：

```json
{
  "tenant_id": "company_001",
  "object_type": "bug",
  "object_id": "123",
  "chunk_id": "bug:123:chunk:1",
  "project_id": "8",
  "product_id": "3",
  "module_name": "登录",
  "status": "active",
  "permission_scope": ["project:8", "product:3"],
  "updated_at": "2026-07-09T10:30:00+08:00"
}
```

检索规范：

- 必须带 `tenant_id` 过滤。
- 必须带用户权限范围过滤。
- 相似 Bug 默认只查 `object_type=bug`。
- 文档问答可查 `doc|story|task|bug|case`，但必须按场景限制。
- TopK 初始取 20，重排后给模型 5-8 条。

删除规范：

- 默认软删除：`deleted=true`。
- 查询时过滤删除和归档数据。
- 确认需要物理删除时，再删除向量点。

### 16.9 模型网关规范

所有模型调用经过 `llm-gateway`，业务模块不得直接调用模型供应商。

业务模块禁止直接调用 OpenAI、DeepSeek、通义、Azure 等供应商 SDK 或 HTTP API。`bug_ai`、`feedback_ai`、`doc_ai`、`project_ai` 只能调用 AI 服务内部统一接口。

推荐调用链：

```text
业务模块
  ↓
modules/llm/gateway.py
modules/embedding/gateway.py
  ↓
模型适配层
  ↓
OpenAI / DeepSeek / 通义 / Azure / 私有模型
```

MVP 推荐使用 `LiteLLM` 作为多模型适配层，业务层仍只依赖项目内部 gateway。后续如果替换为自研 adapter、LlamaIndex、LangChain 或私有模型网关，不影响业务模块。

暂不建议第一版重度使用 LangChain。第一版优先保持链路可控：Qdrant 检索、权限过滤、Prompt 组装、模型调用和人工确认写回都由项目内 service 明确实现。

模型配置归属：

- 模型配置和基础调用放在独立 AI 服务。
- 禅道扩展只保存 AI 服务地址、服务 Token 和功能开关。
- 禅道扩展不直接保存模型 API Key，不直接调用模型供应商。
- AI 服务负责聊天模型、Embedding 模型、视觉模型、重试、限流、超时和审计。

模型配置：

```json
{
  "provider": "openai_compatible",
  "base_url": "https://model.example.com/v1",
  "model": "deepseek-chat",
  "api_key": "******",
  "timeout_seconds": 60,
  "max_retries": 2
}
```

模型配置字段：

| 字段 | 说明 |
| --- | --- |
| `provider` | 模型供应商，例如 openai_compatible、azure、qwen、deepseek、local |
| `base_url` | 模型 API 地址 |
| `api_key` | 加密保存的模型密钥 |
| `chat_model` | 默认聊天模型 |
| `embedding_model` | 默认向量模型 |
| `vision_model` | 可选，多模态图片理解模型 |
| `timeout_seconds` | 调用超时时间 |
| `max_retries` | 最大重试次数 |
| `rate_limit` | 每分钟最大调用数 |
| `enabled` | 是否启用 |
| `public_model_allowed` | 是否允许调用公网模型 |

统一调用接口：

```python
llm.chat(
    messages=[...],
    model="default_chat",
    temperature=0.2,
    response_format="json",
    timeout=60
)
```

Embedding 统一调用接口：

```python
embedding.embed(
    text=chunk.content,
    model="default_embedding"
)
```

图片理解统一调用接口：

```python
vision.describe(
    image_bytes=image,
    prompt="请描述这张 Bug 截图中的异常信息",
    model="default_vision"
)
```

V1.1 可以先不启用视觉模型，只使用 OCR 生成 image chunk。

要求：

- 分类、优先级建议使用低温度：`temperature=0-0.3`。
- 文案生成可使用中温度：`temperature=0.5-0.7`。
- 结构化结果必须要求 JSON Schema。
- 模型调用必须记录输入摘要、输出、token、耗时、费用估算。
- 失败时支持重试和降级。
- API Key 必须加密存储，日志不得输出明文密钥。
- Embedding 模型和聊天模型分开配置，避免用聊天模型承担向量任务。
- 图片理解能力可选，V1.1 可先只做 OCR。
- 模型切换只能影响模型配置或 adapter，不应要求修改业务模块代码。
- 模型调用失败必须写入调用日志和任务错误状态。

### 16.10 Prompt 规范

Prompt 必须版本化管理：

```text
prompt_code: bug_triage
version: 1.0.0
scenario: Bug 分析
owner: ai-team
```

Prompt 输出必须结构化：

```json
{
  "summary": "问题摘要",
  "risk_level": "low|medium|high|critical",
  "suggested_priority": "P0|P1|P2|P3",
  "suggested_owner_reason": "建议负责人理由",
  "similarity_reason": "相似问题依据",
  "evidence": [
    {
      "object_type": "bug",
      "object_id": "98",
      "quote": "相关证据摘要"
    }
  ],
  "confidence": 0.82
}
```

Prompt 要求：

- 明确角色、任务、输入、输出格式。
- 禁止编造没有证据的结论。
- 证据不足时返回 `insufficient_evidence=true`。
- 所有建议必须带理由。
- 不允许直接输出“已修改状态”“已关闭 Bug”等执行结果。

### 16.11 AI 结果存储规范

AI 结果不直接覆盖禅道核心字段，先存 AI 服务库。

状态流：

```text
generated
  ↓
pending_confirmation
  ↓
accepted / rejected / expired
  ↓
written_back
```

结果结构：

```json
{
  "tenant_id": "company_001",
  "object_type": "bug",
  "object_id": "123",
  "analysis_type": "bug_triage",
  "result": {},
  "evidence": [],
  "confidence": 0.82,
  "status": "pending_confirmation",
  "created_by": "ai",
  "confirmed_by": null,
  "created_at": "2026-07-09T10:30:00+08:00"
}
```

建议表字段：

| 字段 | 说明 |
| --- | --- |
| `id` | 分析结果 ID |
| `tenant_id` | 租户 ID |
| `object_type` | 原始对象类型 |
| `object_id` | 原始对象 ID |
| `analysis_type` | 分析类型，例如 bug_triage、feedback_classify、doc_qa、project_weekly |
| `task_id` | 异步任务 ID |
| `request_id` | 请求 ID |
| `prompt_code` | Prompt 编码 |
| `prompt_version` | Prompt 版本 |
| `model` | 使用的模型 |
| `input_snapshot` | 输入摘要或脱敏后的输入快照 |
| `result` | AI 结构化结果 |
| `evidence` | 证据来源 |
| `confidence` | 置信度 |
| `status` | generated、pending_confirmation、accepted、rejected、written_back、failed |
| `error_code` | 失败错误码 |
| `error_message` | 失败错误信息 |
| `token_input` | 输入 token |
| `token_output` | 输出 token |
| `duration_ms` | 耗时 |
| `created_by` | 发起人 |
| `created_at` | 发起时间 |
| `confirmed_by` | 采纳或驳回人 |
| `confirmed_at` | 采纳或驳回时间 |
| `written_back_at` | 写回时间 |

写回禅道规则：

- 用户点击确认后才写回。
- 优先写评论或 AI 分析记录。
- 改优先级、负责人、状态等核心字段必须二次确认。
- 所有写回记录保留审计日志。
- 创建 Bug、需求时先生成草稿，由用户确认后调用禅道接口创建。
- 关联已有 Bug、需求时必须展示关联对象和理由。
- 写回失败时保留 AI 结果，状态标记为 `write_back_failed`，允许用户重试。
- 用户修改 AI 草稿后写回，必须记录 AI 原始结果和用户最终提交结果。

写回动作分类：

| 动作 | 是否允许自动执行 | 要求 |
| --- | --- | --- |
| 写入评论 | 否 | 用户确认后执行 |
| 创建 Bug 草稿 | 否 | 只生成草稿，不直接创建正式 Bug |
| 创建需求草稿 | 否 | 只生成草稿，不直接创建正式需求 |
| 修改优先级 | 否 | 二次确认并记录审计 |
| 修改严重程度 | 否 | 二次确认并记录审计 |
| 关联已有对象 | 否 | 展示候选对象和理由 |
| 发布周报 | 否 | 项目经理确认后发布 |

### 16.12 权限与数据安全规范

必须遵守：

- AI 服务不能绕过禅道权限。
- 向量检索必须按 `tenant_id + permission_scope` 做候选预过滤，但 `permission_scope` 不是授权依据。
- 候选内容进入 Prompt 前必须回调禅道批量权限接口逐个校验对象读取权限。
- 权限接口不可用、结果不明确或 ACL 版本落后时默认拒绝访问。
- 外部模型调用前可配置脱敏。
- API Key 加密存储。
- 日志中不输出完整敏感文本和密钥。
- 客户可配置是否允许公网模型。
- 企业版支持只使用内网模型。

敏感信息脱敏：

```text
手机号 -> [PHONE]
邮箱 -> [EMAIL]
身份证 -> [ID_CARD]
Token/API Key -> [SECRET]
客户名称 -> 可配置是否脱敏
```

### 16.13 日志与审计规范

每次 AI 分析记录：

- `request_id`
- 调用用户
- 租户
- 对象类型和 ID
- 分析类型
- 检索到的证据 ID
- 模型名称
- token 消耗
- 耗时
- 是否成功
- 用户是否采纳

日志要求：

- API 日志使用 JSON。
- 错误日志包含堆栈，但不包含密钥。
- 模型输入输出可配置保存级别。
- 企业部署默认保存调用摘要，不保存完整 Prompt。

### 16.14 测试规范

必须覆盖：

- API 单元测试。
- 数据同步幂等测试。
- 权限过滤测试。
- 向量检索 metadata 过滤测试。
- Prompt 输出 JSON Schema 校验。
- Worker 重试测试。
- 禅道扩展调用 AI 服务的集成测试。

AI 质量测试集：

- 50 条历史 Bug。
- 50 条客户反馈。
- 20 篇项目文档。
- 10 个真实项目周报样例。

评价指标：

- 分类准确率。
- 相似 Bug Top5 命中率。
- 文档问答引用准确率。
- 项目风险人工认可率。
- AI 建议采纳率。

### 16.15 部署与配置规范

环境变量示例：

```env
APP_ENV=production
DATABASE_URL=postgresql://user:pass@postgres:5432/ai_service
REDIS_URL=redis://redis:6379/0
QDRANT_URL=http://qdrant:6333
ZENTAO_BASE_URL=http://zentao
ZENTAO_SERVICE_TOKEN=******
LLM_PROVIDER=openai_compatible
LLM_BASE_URL=https://model.example.com/v1
LLM_API_KEY=******
DEFAULT_CHAT_MODEL=deepseek-chat
DEFAULT_EMBEDDING_MODEL=bge-m3
```

部署要求：

- API 和 Worker 分开进程。
- 数据库迁移使用 Alembic。
- 健康检查接口：`GET /healthz`。
- 就绪检查接口：`GET /readyz`。
- Docker 镜像固定版本号。
- 所有配置通过环境变量或配置文件注入。

### 16.16 MVP 技术交付清单

第一版技术交付：

- FastAPI 服务骨架。
- Celery Worker。
- PostgreSQL 表结构。
- Qdrant 向量集合。
- 禅道同步模块。
- Bug 向量化。
- 文档向量化。
- Bug 相似检索 API。
- Bug AI 分析 API。
- 单文档摘要和问答 API。
- 全局 AI 客服悬浮入口与聊天弹窗。
- `help`、`bug`、`doc`、`feedback` 范围的会话、消息、历史消息和流式回答 API。
- 会话、消息、引用和反馈数据表。
- 与业务向量数据隔离的平台帮助知识库。
- AI 分析结果表。
- AI 异步任务表。
- 通过服务端配置注入的基础模型网关。
- 索引状态查询 API。
- 失败任务重试 API。
- HMAC 鉴权、Nonce 防重放、幂等和租户隔离。
- 禅道批量对象级权限校验接口联调。
- AI 结果采纳、驳回和仅添加评论的写回。
- 统一日志。
- 调用审计和 token 统计。
- Docker Compose 部署。

图片 OCR、客户反馈向量化与分类、项目周报、项目范围及跨对象问答、通用推荐和模型管理后台不属于首批 MVP 技术交付。

## 17. 实施路线

### 17.1 阶段 0：技术验证，1-2 周

目标：

- 部署禅道开源版。
- 验证 `extension/custom` 能否在 Bug、文档、反馈页面插入 AI 入口或面板。
- 验证禅道 API 或数据库只读同步。
- 验证 AI 服务调用大模型和向量库。

交付：

- 禅道测试环境。
- AI 服务 Demo。
- Bug 详情页 AI 按钮。
- 文档向量问答 Demo。
- 全局机器人入口和 `help` 范围聊天 Demo。

### 17.2 阶段 1：数据同步与向量化，2-3 周

目标：

- 同步 Bug 和文档。
- 建立增量同步机制。
- 建立向量库入库和更新策略。
- 写入权限 metadata，并完成禅道批量对象级权限回调。

交付：

- 数据同步服务。
- 向量化 Worker。
- Qdrant 集合和 metadata 规范。
- 同步监控页面或日志。

### 17.3 阶段 2：AI MVP，4-6 周

目标：

- Bug 相似检索和优先级建议。
- 单文档摘要和问答。
- 上线全局 AI 客服，支持 `help`、`bug`、`doc`、`feedback` 范围、流式回答和历史恢复。
- AI 结果采纳、驳回和仅添加评论的写回。

交付：

- 禅道 AI 扩展入口。
- AI 服务 API。
- AI 分析结果表。
- Bug 与文档 AI 面板。
- 所有登录后业务页面的机器人入口与聊天弹窗。

### 17.4 阶段 3：试点运行，4 周

目标：

- 找 1-3 个真实项目试用。
- 观察 AI 建议采纳率。
- 优化 Prompt、规则和权限。
- 补齐异常处理和审计。

交付：

- 试点报告。
- 问题清单。
- 下一阶段路线图。

### 17.5 阶段 4：商业化增强，6-8 周

目标：

- 多租户支持。
- 私有模型支持。
- 飞书/企微/钉钉推送。
- 客户反馈多渠道接入。
- 管理驾驶舱。

交付：

- 可对外演示版本。
- 私有化部署包。
- 企业版功能清单。

## 18. 项目指标

### 18.1 产品指标

- AI 分析按钮点击率。
- AI 建议采纳率。
- 相似 Bug 命中率。
- 客户反馈自动分类准确率。
- 文档问答满意率。
- 周报人工修改比例。

### 18.2 业务指标

- Bug 平均处理时长是否下降。
- 重复 Bug 创建数量是否下降。
- 客户反馈转需求/Bug 的时间是否下降。
- 项目风险提前发现数量。
- 版本发布遗留严重 Bug 数量。

### 18.3 技术指标

- 数据同步延迟。
- 向量入库成功率。
- AI 接口响应时间。
- Worker 任务失败率。
- 模型调用成本。
- 权限过滤错误数。

## 19. 风险与应对

| 风险 | 表现 | 应对 |
| --- | --- | --- |
| 禅道升级冲突 | 扩展依赖页面 DOM 或内部方法 | 少改核心，扩展隔离，升级前跑兼容测试 |
| AI 幻觉 | 回答不准确或编造结论 | 强制引用来源，结论分置信度，人工确认 |
| 权限泄露 | 用户看到无权限文档内容 | 查询前权限过滤，向量 metadata 隔离 |
| 数据同步不完整 | 分析结果缺上下文 | Webhook + 定时补偿 + 同步状态监控 |
| 向量库噪音大 | 检索出无关内容 | 文本清洗、chunk 策略、metadata 过滤、重排 |
| 模型成本高 | 大量调用 LLM | 缓存、批处理、小模型分类、大模型只做复杂分析 |
| 客户数据敏感 | 不能发公网模型 | 脱敏、私有模型、内网部署 |
| 用户不信任 AI | 建议无法落地 | 展示证据链和理由，只做建议不自动执行 |

## 20. 商业化思路

### 20.1 产品形态

- 禅道 AI 插件/扩展包。
- 独立 AI 服务私有化部署包。
- SaaS 版 AI 分析服务。
- 企业定制版，支持私有模型和专属数据接入。

### 20.2 收费方式

| 版本 | 目标用户 | 收费建议 |
| --- | --- | --- |
| 社区体验版 | 小团队试用 | 免费或低价，限制项目数/调用量 |
| 团队版 | 10-50 人研发团队 | 按用户或按项目收费 |
| 专业版 | 50-200 人团队 | 增加客户反馈、文档问答、风险报告 |
| 企业版 | 私有化客户 | 部署费 + 年服务费 + 模型适配费 |

### 20.3 差异化

- 中文禅道生态友好。
- 面向研发项目和 Bug 闭环，不是泛聊天机器人。
- 能结合项目流程、文档、客户反馈和版本质量。
- 可私有化、可接国产模型、可保留数据在内网。

## 21. 推荐下一步

1. 部署禅道开源版测试环境。
2. 写一个最小 `extension/custom/bug` 扩展，在 Bug 详情页插入“AI 分析”按钮。
3. 搭建 AI 服务 Demo：`ai-api + postgres + redis + qdrant`。
4. 先同步 Bug、文档、反馈三类数据。
5. 完成两个 Demo：
   - Bug 相似问题检索。
   - 文档问答。
6. 再扩展到客户反馈分类和项目风险周报。

## 22. 参考来源

- 禅道开源版：https://www.zentao.net/zentao-os.html
- 禅道 GitHub 源码：https://github.com/easysoft/zentaopms
- 禅道 Gitee 源码：https://gitee.com/easysoft/zentaopms
- Redmine：https://www.redmine.org/
- MantisBT：https://www.mantisbt.org/
- OpenProject：https://www.openproject.org/
- Plane GitHub：https://github.com/makeplane/plane
- Taiga：https://taiga.io/
