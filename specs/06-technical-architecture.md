# 食刻 — 一期技术架构 Spec

- **文档路径**：`specs/06-technical-architecture.md`
- **版本**：V1.0
- **状态**：Draft
- **对应 PRD**：V1.2
- **关联文档**：
  - `specs/01-business-flow.md`
  - `specs/02-data-model.md`
  - `specs/03-content-spec.md`
  - `specs/04-schedule-rules.md`
  - `specs/05-safety-rules.md`
- **一期目标场景**：
  - 孕 30 周左右的孕晚期用户
  - 2 岁左右幼儿的主要照护者

---

## 1. 文档目标

本文档定义食刻一期的整体技术架构、模块职责、数据访问方式、云函数划分、日程引擎、安全规则中心、内容管理、任务调度、权限控制、部署及质量保障要求。

目标是将前五份 Spec 中确定的产品与业务规则转换为可实施的技术设计。

本架构需要解决：

1. 如何组织微信小程序前端代码。
2. 如何划分孕期与婴幼儿两个业务域。
3. 如何设计共享业务能力。
4. 如何实现统一日程生成引擎。
5. 如何实现独立安全规则中心。
6. 如何管理食谱、食材和内容版本。
7. 如何实现未来 7 天日程自动补齐。
8. 如何保证日程生成和任务操作的幂等性。
9. 如何处理安全档案、食谱和规则更新后的日程重新校验。
10. 如何实现轻量内容维护后台。
11. 如何保证敏感数据访问安全。
12. 如何支持后续扩展到完整孕期及 0—36 月龄。

## 2. 一期技术选型

### 2.1 技术栈

| 层级     | 技术方案                         | 一期用途                         |
| -------- | -------------------------------- | -------------------------------- |
| 用户端   | 微信小程序原生框架               | 用户交互、日程、食谱、任务管理   |
| 前端语言 | JavaScript 或 TypeScript         | 页面与业务逻辑                   |
| 前端样式 | WXSS                             | 小程序界面                       |
| 后端平台 | 微信云开发                       | 云端基础设施                     |
| 后端业务 | 微信云函数                       | 业务接口及规则执行               |
| 数据库   | 微信云数据库                     | 用户、档案、食谱、日程等数据     |
| 文件存储 | 微信云存储                       | 食谱图片、内容资料等非结构化文件 |
| 定时调度 | 云开发定时触发器                 | 未来 7 天日程补齐、定时检查      |
| 内容管理 | 轻量受控管理工具                 | 食谱、食材、规则和审核维护       |
| 规则引擎 | TypeScript/JavaScript 纯函数模块 | 阶段匹配、候选筛选、安全校验     |
| 数据分析 | 事件埋点与基础统计               | 核心行为和业务质量分析           |

### 2.2 技术选型原则

一期优先采用微信生态原生能力，降低基础设施和运维成本。

不建议一期引入：

- Kubernetes。
- 微服务拆分。
- 独立消息队列集群。
- Drools 等通用规则引擎平台。
- 独立推荐算法服务。
- 独立 AI 日程生成服务。
- 复杂 CMS。
- 多套数据库。
- 不必要的第三方用户身份体系。

当业务规模和实际性能需求出现明确证据后，再评估是否拆分。

### 2.3 技术版本约束

具体 SDK、云函数运行时、数据库 API 和构建工具版本应在项目初始化时确认。

实施前必须检查：

- 微信开发者工具当前支持的基础库版本。
- 微信小程序项目配置。
- 云开发环境配置。
- 云函数运行时支持情况。
- 当前云数据库 SDK 及事务能力。
- 云开发定时触发器支持情况。

不得在没有检查项目环境的情况下直接假定某个 SDK API 或运行时版本可用。

## 3. 整体架构

### 3.1 逻辑架构

```text
┌─────────────────────────────────────────────┐
│                用户交互层                    │
│                                             │
│  微信小程序                                  │
│  ├── 用户与档案                              │
│  ├── 孕期业务                                │
│  ├── 幼儿业务                                │
│  ├── 日程与任务                              │
│  ├── 食谱详情                                │
│  └── 个人设置                                │
└──────────────────────┬──────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────┐
│                云端业务接口层                │
│                                             │
│  ├── User / Profile Service                 │
│  ├── Pregnancy Service                      │
│  ├── Child Service                          │
│  ├── Schedule Service                       │
│  ├── Task Service                           │
│  ├── Content Service                        │
│  ├── Safety Service                         │
│  └── Analytics Service                      │
└──────────────────────┬──────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────┐
│                共享业务能力层                │
│                                             │
│  ├── Profile Manager                        │
│  ├── Schedule Engine                        │
│  ├── Task Manager                            │
│  ├── Stage Resolver                         │
│  ├── Content Resolver                       │
│  ├── Safety Rule Engine                     │
│  ├── Rule Version Manager                   │
│  ├── Idempotency Manager                    │
│  └── Audit Manager                          │
└──────────────────────┬──────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────┐
│                数据与内容层                  │
│                                             │
│  ├── 用户与档案数据                          │
│  ├── 孕期业务数据                            │
│  ├── 幼儿业务数据                            │
│  ├── 食材与食谱内容                          │
│  ├── 安全规则与用户安全档案                  │
│  ├── 日程与任务数据                          │
│  ├── 内容及规则版本                          │
│  └── 审计及行为事件                          │
└──────────────────────┬──────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────┐
│                微信云开发基础设施            │
│                                             │
│  云函数 / 云数据库 / 云存储 / 定时触发器     │
└─────────────────────────────────────────────┘
```

### 3.2 核心架构约束

1. 小程序端只负责交互和展示，不承担最终安全判断。
2. 用户身份由可信服务端上下文确定，不接受客户端传入的用户 ID 作为授权依据。
3. 业务写操作必须通过云函数完成。
4. 日程生成逻辑不能直接放在页面中。
5. 安全规则必须通过统一服务执行。
6. 内容审核和发布必须在受控后台完成。
7. 孕期与幼儿业务域保持独立。
8. 共享能力通过明确的服务接口提供。
9. 历史日程必须保留生成时使用的内容及规则版本。
10. 所有涉及安全状态变化的操作必须保留审计记录。

## 4. 业务域与模块边界

### 4.1 用户与档案域

职责：

- 用户身份识别。
- 用户基础信息管理。
- 多档案管理。
- 当前档案选择。
- 档案状态管理。
- 档案与用户之间的归属关系。

一期支持一个账号拥有多个孕期或幼儿档案。

虽然一期界面可以简化档案切换，但底层数据必须具备多档案隔离能力。

### 4.2 孕期业务域

职责：

- 孕期档案。
- 孕周计算。
- 孕期阶段识别。
- 孕期饮食约束。
- 孕期食谱适用性。
- 孕期特殊情况记录。

孕期业务域独立维护专业字段，不与幼儿业务字段强行合并。

### 4.3 幼儿业务域

职责：

- 幼儿档案。
- 年龄阶段识别。
- 喂养与进食相关信息。
- 食物过敏信息。
- 食物观察记录。
- 幼儿进食安全约束。
- 幼儿食谱适用性。

幼儿业务域应能够扩展到 0—36 月龄，但一期只启用 2 岁左右幼儿相关规则和内容。

### 4.4 共享业务域

| 模块              | 职责                     |
| ----------------- | ------------------------ |
| Schedule Service  | 日程生成和查询           |
| Task Service      | 微任务状态与操作         |
| Content Service   | 食材、食谱和内容版本读取 |
| Safety Service    | 统一安全校验             |
| Stage Resolver    | 当前阶段识别             |
| Audit Service     | 操作审计                 |
| Analytics Service | 行为事件记录             |

共享的是基础业务能力，不是所有业务实体。

### 4.5 模块依赖规则

```text
Pregnancy Domain ─────┐
                      ├── Schedule Service
Child Domain ─────────┘

Schedule Service
  ├── Stage Resolver
  ├── Content Service
  ├── Safety Service
  └── Task Service

Safety Service
  ├── User Safety Profile
  ├── Food Safety Profile
  ├── Recipe Safety Profile
  └── Published Safety Rules
```

约束：

- Schedule Service 不直接修改安全规则。
- Pregnancy Domain 不直接访问 Child Domain 内部数据。
- Child Domain 不直接访问 Pregnancy Domain 内部数据。
- 业务域之间通过明确的服务接口协作。
- Safety Service 不依赖页面组件。
- 内容服务不能自行绕过安全校验直接写入日程。

## 5. 项目目录建议

建议采用模块化单体结构，而不是拆成多个独立服务。

```text
shike/
├── miniprogram/
│   ├── app.js
│   ├── app.json
│   ├── app.wxss
│   ├── pages/
│   │   ├── onboarding/
│   │   ├── profile/
│   │   ├── schedule/
│   │   ├── task/
│   │   ├── recipe/
│   │   └── settings/
│   ├── modules/
│   │   ├── pregnancy/
│   │   │   ├── pages/
│   │   │   ├── components/
│   │   │   └── services/
│   │   ├── child/
│   │   │   ├── pages/
│   │   │   ├── components/
│   │   │   └── services/
│   │   ├── schedule/
│   │   ├── task/
│   │   └── safety/
│   ├── components/
│   ├── services/
│   │   ├── cloud.js
│   │   ├── user.js
│   │   └── analytics.js
│   ├── utils/
│   ├── constants/
│   └── assets/
│
├── cloudfunctions/
│   ├── user/
│   ├── profile/
│   ├── pregnancy/
│   ├── child/
│   ├── schedule/
│   ├── task/
│   ├── content/
│   ├── safety/
│   ├── admin/
│   └── shared/
│
├── packages/
│   ├── domain/
│   │   ├── pregnancy/
│   │   ├── child/
│   │   ├── schedule/
│   │   ├── task/
│   │   └── safety/
│   ├── rules/
│   ├── content-models/
│   ├── validation/
│   └── shared/
│
├── admin/
│   ├── pages/
│   ├── components/
│   ├── services/
│   └── permissions/
│
├── specs/
│   ├── 00-project-overview.md
│   ├── 01-business-flow.md
│   ├── 02-data-model.md
│   ├── 03-content-spec.md
│   ├── 04-schedule-rules.md
│   ├── 05-safety-rules.md
│   └── 06-technical-architecture.md
│
├── tests/
│   ├── unit/
│   ├── integration/
│   ├── safety/
│   └── fixtures/
│
├── scripts/
├── docs/
├── project.config.json
└── README.md
```

说明：

- 以上为逻辑目录建议，不代表必须一次性创建全部目录。
- `packages/` 用于组织纯业务逻辑，不依赖微信运行环境的代码。
- 如果当前微信开发者工具或构建方式不适合直接引用外部 package，应先确认模块解析方式，再决定是否使用独立目录或构建复制。
- 不要为了目录结构引入不必要的构建系统。
- 云函数之间优先共享明确的公共模块，避免复制业务规则。
- `admin/` 的具体技术形态可在实施阶段选择，不能因此提前引入复杂 CMS。

## 6. 核心服务设计

### 6.1 User Service

主要能力：

- 获取当前登录用户。
- 初始化用户记录。
- 获取用户档案列表。
- 更新用户基础信息。
- 校验档案归属。

关键约束：

- 使用可信的微信身份上下文。
- 用户 ID 由服务端获取。
- 不允许客户端传入其他用户 ID 后直接读取或修改其数据。

### 6.2 Profile Service

主要能力：

- 创建孕期档案。
- 创建幼儿档案。
- 查询档案详情。
- 修改档案。
- 切换当前档案。
- 获取档案版本。

档案修改涉及阶段、安全或日程适用性时，应触发后续校验流程。

### 6.3 Pregnancy Service

主要能力：

- 计算当前孕周。
- 解析孕期阶段。
- 查询孕期适用规则。
- 读取孕期特殊限制。

孕周计算和日期边界必须统一实现，不能在多个页面中分别计算。

### 6.4 Child Service

主要能力：

- 计算幼儿年龄。
- 解析适用阶段。
- 读取幼儿进食能力相关信息。
- 管理食物观察记录。
- 管理过敏相关状态。

年龄计算必须统一处理生日、时区及日期边界问题。

### 6.5 Content Service

主要能力：

- 查询食材。
- 查询食谱。
- 查询食谱版本。
- 获取内容适用阶段。
- 获取食材及食谱安全属性。
- 查询审核状态。
- 查询内容来源。

只允许读取已发布且在有效期内的内容作为生产候选。

### 6.6 Schedule Service

主要能力：

- 生成未来 7 天日程。
- 查询日程。
- 查询日程任务。
- 补齐缺失日程。
- 重新生成日程。
- 处理档案阶段变化。
- 处理内容和规则变化。

### 6.7 Task Service

主要能力：

- 完成任务。
- 跳过任务。
- 替换食谱。
- 顺延任务。
- 查询任务状态。
- 记录任务执行历史。

所有会改变当前有效食谱的操作必须重新执行安全校验。

### 6.8 Safety Service

主要能力：

- 查询用户安全档案。
- 查询有效安全规则。
- 校验食材。
- 校验食谱。
- 校验复合食材。
- 校验交叉接触信息。
- 输出统一安全判断。
- 记录规则命中。
- 触发未来任务重新校验。

Safety Service 必须是唯一的生产安全判断入口。

## 7. 日程引擎技术设计

### 7.1 设计原则

日程引擎是规则驱动的确定性业务模块，不是 AI 内容生成器。

核心流程：

```text
读取档案
  ↓
识别业务域和当前阶段
  ↓
获取日程槽位
  ↓
查询已发布内容候选
  ↓
阶段过滤
  ↓
安全规则硬过滤
  ↓
营养结构匹配
  ↓
个性化排序
  ↓
结果完整性校验
  ↓
写入日程和任务
  ↓
记录生成版本与审计信息
```

### 7.2 规则引擎结构

建议将规则实现为纯函数：

```text
resolveStage(profile, date)
  → StageResult

buildScheduleSlots(stage, dateRange)
  → ScheduleSlot[]

getRecipeCandidates(slot, contentSnapshot)
  → RecipeCandidate[]

filterByStage(candidates, stage)
  → RecipeCandidate[]

filterBySafety(candidates, safetyContext)
  → SafetyFilterResult

matchNutrition(candidates, nutritionProfile)
  → RecipeCandidate[]

rankCandidates(candidates, preferences)
  → RecipeCandidate[]

validateSchedule(scheduleDraft)
  → ValidationResult
```

纯函数要求：

- 输入明确。
- 输出确定。
- 不直接访问数据库。
- 不依赖页面状态。
- 不依赖系统当前时间的隐式值。
- 不在函数内部直接写入数据库。
- 规则测试可独立执行。

数据库读取、事务和任务写入由应用服务层处理。

### 7.3 日程生成上下文

一次日程生成应构建明确的上下文：

```text
ScheduleGenerationContext
  - userId
  - profileId
  - domain
  - targetDateRange
  - profileSnapshot
  - stageSnapshot
  - safetyProfileSnapshot
  - contentVersionSnapshot
  - safetyRuleVersion
  - scheduleRuleVersion
  - preferenceSnapshot
  - requestId
```

生成过程中不得混用不同版本的档案和规则。

### 7.4 未来 7 天日程

一期采用每日执行、提前生成未来 7 天日程。

技术要求：

- 使用统一业务时区和日期计算方式。
- 按档案分别生成。
- 检查已有日程。
- 只补齐缺失日期或缺失槽位。
- 不重复生成已经存在的有效任务。
- 不覆盖已完成任务。
- 不擅自覆盖用户已跳过或已替换的任务。
- 生成失败时保留已有有效日程。
- 日程不足时返回可解释的结果。

### 7.5 候选不足

候选不足时：

- 不放宽安全规则。
- 不使用未经审核的内容。
- 不使用阶段不适用的食谱。
- 不使用安全属性不完整且必须审核的内容。
- 可以返回不完整日程。
- 应记录缺少候选的原因。
- 必要时向用户展示合理的结构建议或待补充提示。

### 7.6 日程结果版本快照

每次生成应记录：

- profileVersion
- stageRuleVersion
- scheduleRuleVersion
- safetyRuleVersion
- contentVersionId
- generatedAt
- generationRequestId

每个 ScheduleTask 应关联具体的食谱内容版本。

历史任务不能只引用食谱当前版本，否则内容更新后无法还原当时的推荐依据。

## 8. 安全规则中心技术设计

### 8.1 模块独立性

安全规则中心应作为独立业务模块存在，但一期不需要独立部署成微服务。

逻辑结构：

```text
Safety Service
  ├── Safety Profile Repository
  ├── Safety Rule Repository
  ├── Food Safety Resolver
  ├── Recipe Safety Resolver
  ├── Allergen Matcher
  ├── Cross Contact Checker
  ├── Safety Rule Evaluator
  ├── Safety Audit Writer
  └── Safety Revalidation Handler
```

### 8.2 统一校验入口

```text
checkRecipeSafety({
  profileId,
  recipeId,
  contentVersionId,
  scheduleDate,
  operationType
})
```

返回：

```text
SafetyCheckResult
  - result
  - matchedRules[]
  - blockedIngredients[]
  - warnings[]
  - reviewRequiredItems[]
  - safetyProfileVersion
  - safetyRuleVersion
  - checkedAt
```

### 8.3 Fail Closed

安全校验失败时必须默认拒绝自动放行。

以下情况必须阻止本次候选进入生产日程：

- 安全规则服务不可用。
- 用户安全档案读取失败。
- 关键食材映射无法完成。
- 规则版本不可用。
- 安全数据结构不符合要求。
- 关键安全属性缺失。
- 校验结果不完整。

不能将异常捕获后转换成 `allow`。

### 8.4 安全档案更新

**过敏禁忌解除与审计（F-04.3-06 / F-04.3-07 / F-04.3.1-02）**：

- `AllergyChangeLog` 写入集合 `allergy_change_logs`（02 §7.4）；分类：**用户安全数据审计**，与通用 `audit_logs` 并存、不重复建第二套禁忌审计集合。
- 仅 **`safety` / `profile` 云函数**（或等价应用服务）在业务校验通过后执行；小程序客户端 **只读** 或经云函数间接查询，**不得**直写。
- **`allergy_item_deactivated` 原子一致性（首选：单库多文档事务 + Transactional Outbox，F-04.3.2）**：
  1. **预检（事务外，非并发保证）**：校验 `actorId`、`userConfirmationId`、`userConfirmationAt`、`operationId`、`confirmationScope` 与目标 `AllergyItem`；校验用户确认凭证有效且未过期。**幂等快路径（须校验终态）**：若已存在同 `(profileId, allergyItemId, operationId)` 的 `outcome=success` 日志 **且** 目标 `AllergyItem.status=inactive` → 返回原成功结果（不重复更新、不重复 Outbox）。若仅有 success 日志但 `status` 仍为 `active` → **数据异常**，不得对外返回成功，须告警并按运维流程修复。
  2. **事务内（必须全部成功或全部回滚）**：
     - **条件更新** `allergy_items`：过滤 `_id` + `profileId` + **`status=active`**，更新为 `inactive`（可配合 `updatedAt` 乐观字段，但 **必须以影响记录数为准**，见 §8.4.1）；
     - 影响记录数 **必须为 1** 方可继续；为 **0** 时 **不得** 在本事务写入 `outcome=success` 或创建 Outbox Job（转入 §8.4.1 分支）；**> 1** 或平台异常 → 回滚并记录错误；
     - **同一事务内**追加 `AllergyChangeLog`：`changeType=allergy_item_deactivated`，`outcome=success`，`previousStatus=active`，`nextStatus=inactive`，`changedItems` 含变更快照；
     - **同一事务内**创建 `BackgroundJob`（02 §7.4.1）：`jobType=allergy_item_deactivated_schedule_revalidation`，`status=pending`，`idempotencyKey=profileId+allergyItemId+operationId`，并在 success 日志写入 `revalidationTriggered=true`、`revalidationReference=<job._id>`；
     - **禁止**在 `status` 条件更新未成功（影响记录数 ≠ 1）时写入 `outcome=success` 或创建 Outbox。
  3. **事务提交后**：由 `scheduler` / `safety` Worker 异步领取 §14.4 定义的 Job 执行日程重校验；**不得**依赖「提交后再创建 Job」作为唯一触发（Job 已在事务内持久化）。
  4. **失败路径**：预检失败或事务失败 → **不得**将 `status` 置为 `inactive`；可 **独立** 追加 `outcome=failed`/`rejected` 日志（不与成功解除同事务）；客户端可凭相同 `operationId` 在凭证仍有效时重试事务（§8.4.1）。
- **等效机制**：若运行环境无法使用可靠多文档事务，须采用两阶段状态（如 `deactivation_pending` → `inactive`）+ 可恢复 Saga，且 **`outcome=success` 与 Outbox Job 仅出现在 `inactive` 已持久化之后**；具体状态机须在实现设计评审中映射至 02 `AllergyItem.status` 枚举扩展或内部操作表，**不得**仅声明「最终一致」而无补偿步骤与 Outbox 等价物。

#### 8.4.1 解除幂等、并发与影响记录数（F-04.3.2）

**数据库约束（须与 02 §12 一致）**：

- `allergy_change_logs`：**唯一索引** `(profileId, allergyItemId, operationId)`，约束 `changeType=allergy_item_deactivated` 且 `outcome=success`（或实现层等效唯一性）。
- `background_jobs`：**唯一索引** `idempotencyKey`（过敏解除重校验 Job）。

**条件更新语义**：解除 API **必须**使用「仅当 `status=active` 时更新为 `inactive`」的原子操作；**禁止**将事务外的 `find` 结果作为唯一并发依据。

**影响记录数 = 0 时的分支**（事务内更新失败后 **不得** 写 success 日志）：

| 当前持久化状态 | 处理 |
| --- | --- |
| 已存在本 `operationId` 的 success 日志且 `status=inactive` | 幂等成功，返回原结果 |
| `status=inactive` 但无本 `operationId` 的 success 日志 | 视为已由其他 `operationId` 解除或数据不一致；返回「已解除」或冲突，**不得**补写 success |
| `status=active`，无 success 日志 | 并发失败或重复 `operationId` 首请求仍在进行；返回处理中或可重试，**不得**写 success |
| 其他 | 终止并告警 |

**相同 `operationId` 重试**：见 02 §7.4 表；失败后可重试事务，但 success 日志与 Outbox **至多一条**（唯一索引强制）。

**不同 `operationId` 并发**：仅首个影响记录数 = 1 的请求完成 success + Outbox；其余影响记录数 = 0，**不得**产生第二条 success 审计。

**异常场景保证**（规格层）：

| 异常场景 | 必须保证的结果 |
| --- | --- |
| AllergyItem 条件更新失败（影响 0，且无幂等 success） | 无 success 审计、无 Outbox Job |
| AllergyChangeLog 写入失败 | 整事务回滚，`status` 仍为 `active` |
| Outbox Job 创建失败 | 整事务回滚，不得单独留下 `inactive` + success |
| 事务提交后 Worker 未运行 | Job 仍为 `pending`，可扫描恢复 |
| Worker 执行中崩溃 | Job 租约到期后可回收并重新领取（§14.4）；旧 Worker 失去执行权后不得写入下游 |
| 重复消费同一 Job | 不重复产生错误任务副作用（§14.4 幂等 + `claimVersion` 隔离） |
| 重校验持续失败 | 保留 `lastError`、`retryCount`，达上限进 `failed` |
| 系统重启 | 扫描 `pending` / **租约过期**的 `processing` 回收（§14.4）；**不得**仅凭 `updatedAt` 代替租约判断 |

安全信息发生变化时（含 Worker 完成的过敏解除重校验）：

1. 保存新版本（含 `AllergyChangeLog` 追加，适用时）。
2. 消费已持久化的重校验 `BackgroundJob`（或同等 Outbox），**不得**假设仅内存/进程内触发。
3. 查询未来未执行日程。
4. 对相关食谱重新执行安全校验。
5. 阻断不再满足条件的任务。
6. 记录重新校验结果。
7. 根据业务策略处理替换。

重新校验任务应具备幂等能力，避免重复执行导致状态混乱。

### 8.5 安全规则版本

每次发布安全规则都必须生成新版本。

规则执行结果应记录：

- 规则版本。
- 用户安全档案版本。
- 食谱安全版本。
- 命中规则。
- 执行结果。
- 执行时间。

不能只保存一个当前规则版本，导致历史结果失去依据。

## 9. 云函数设计

### 9.1 划分原则

云函数按照业务能力划分，而不是按照小程序页面数量划分。

一期建议模块：

| 云函数      | 主要职责                   |
| ----------- | -------------------------- |
| `user`      | 用户初始化和身份相关操作   |
| `profile`   | 档案创建、修改和查询       |
| `pregnancy` | 孕期信息与阶段计算         |
| `child`     | 幼儿信息与阶段计算         |
| `schedule`  | 日程生成和查询             |
| `task`      | 任务完成、跳过、替换、顺延 |
| `content`   | 食材和食谱读取             |
| `safety`    | 安全档案及统一安全校验     |
| `admin`     | 后台内容和规则管理         |
| `scheduler` | 定时补齐和重新校验任务     |

实际部署时可以根据调用频率和维护复杂度合并部分云函数，但不得合并业务职责导致边界模糊。

### 9.2 云函数调用结构

```text
miniprogram
  ↓
cloud.callFunction
  ↓
cloud function entry
  ↓
authentication / authorization
  ↓
input validation
  ↓
application service
  ↓
domain logic
  ↓
repository
  ↓
cloud database
```

### 9.3 云函数返回结构

建议统一：

```text
{
  success: true,
  data: {},
  error: null,
  requestId: "..."
}
```

失败：

```text
{
  success: false,
  data: null,
  error: {
    code: "SCHEDULE_GENERATION_FAILED",
    message: "日程生成失败",
    retryable: true
  },
  requestId: "..."
}
```

约束：

- 不将数据库内部错误直接返回客户端。
- 不返回敏感规则内部配置。
- 错误码保持稳定。
- 错误信息应适合前端处理。
- 服务端日志保留完整错误上下文。

### 9.4 云函数超时和任务拆分

日程生成和安全重新校验不能假设一定能在一次云函数执行时间内完成。

需要：

- 控制单次处理的数据范围。
- 支持分批处理。
- 支持失败重试。
- 支持任务状态查询。
- 避免长时间占用单个云函数。
- 对已经完成的批次不重复写入。

当业务量尚小时可以同步处理；出现执行时长或并发压力后，再切换为分批异步处理。

## 10. 云数据库设计

### 10.1.0 集合与 02 持久化实体映射（权威）

| 集合 | 02 实体 | 说明 |
| --- | --- | --- |
| `schedules` | `ScheduleInstance` | 与 02 建议名 `schedule_instances` 同义，择一落地 |
| `schedule_tasks` | `ScheduleTask` | 含 `scheduleId`、`slotCode` |
| `user_safety_profiles` | `UserSafetyProfile` | 四种 `allergyStatus` 唯一存储归属 |
| `allergy_items` | `AllergyItem` | 用户已知过敏禁忌（05 逻辑名 AllergyRestriction） |
| `allergy_profiles` | — | **已废弃**；历史数据应迁移至 `user_safety_profiles` |
| `food_observations` | `FoodObservation` | 新食物观察 |
| `allergy_change_logs` | `AllergyChangeLog` | 过敏状态与禁忌变更审计（仅服务端追加） |
| `background_jobs` | `BackgroundJob` | 后台任务（含过敏解除日程重校验 Outbox，02 §7.4.1） |
| `food_allergens` | `FoodAllergen` | 过敏原字典 |
| `food_safety_profiles` | `FoodSafetyProfile` | 食材侧安全属性 |
| `recipe_safety_profiles` | `RecipeSafetyProfile` | 食谱侧安全属性 |
| `content_versions` | `ContentVersion` | 唯一内容版本集合；食谱即 `contentType=recipe` |
| `recipe_versions` | — | **已废弃**；与 `content_versions` 不得并存，新实现勿用 |
| `safety_rules` / `safety_rule_versions` | `SafetyRule` / `RuleVersion` | 与 02 规则版本对齐 |

`SchedulePlan`、`ScheduleDay`、`ScheduleSlot` 不单独建集合（02 §6.0、04 §3.1.1）。内容 `status` 枚举见 02 §5.1.1。

### 10.1 数据集合分类

以下为建议集合清单，最终必须与 `02-data-model.md` 合并，避免出现重复实体或字段定义冲突。

#### 用户及档案

| 集合                 | 用途         |
| -------------------- | ------------ |
| `users`              | 用户基本信息 |
| `profiles`           | 档案基础信息 |
| `pregnancy_profiles` | 孕期业务数据 |
| `child_profiles`     | 幼儿业务数据 |

#### 内容中心

| 集合                 | 用途         |
| -------------------- | ------------ |
| `food_items`         | 标准食材     |
| `food_categories`    | 食材分类     |
| `food_allergens`     | 过敏原字典   |
| `recipes`            | 食谱基础信息 |
| `content_versions`   | 食谱等内容版本（`ContentVersion`，含 `contentType=recipe`） |
| `recipe_ingredients` | 食谱食材关系 |
| `source_references`  | 内容依据来源 |

#### 安全规则

| 集合                     | 用途         |
| ------------------------ | ------------ |
| `user_safety_profiles`   | 用户安全档案（`UserSafetyProfile`） |
| `allergy_items`          | 用户已知过敏禁忌（`AllergyItem`）   |
| `allergy_change_logs`    | 过敏变更审计（`AllergyChangeLog`，追加写入） |
| `food_safety_profiles`   | 食材安全属性 |
| `recipe_safety_profiles` | 食谱安全属性 |
| `safety_rules`           | 安全规则     |
| `safety_rule_versions`   | 安全规则版本 |
| `food_observations`      | 食物观察记录 |

#### 日程和任务

| 集合                       | 用途         |
| -------------------------- | ------------ |
| `schedules`                | 日程实例（`ScheduleInstance`，同 02 `schedule_instances`） |
| `schedule_tasks`           | 日程任务     |
| `task_operation_logs`      | 任务操作记录 |
| `schedule_generation_logs` | 日程生成记录 |

#### 系统及审计

| 集合                  | 用途               |
| --------------------- | ------------------ |
| `audit_logs`          | 管理操作及安全审计 |
| `allergy_change_logs` | 用户过敏状态与禁忌变更审计（与 02 §7.4 一致；**非**通用 `audit_logs` 替代品） |
| `analytics_events`    | 用户行为事件       |
| `idempotency_records` | 幂等请求记录       |
| `background_jobs`     | 后台任务记录       |

### 10.2 数据库设计原则

- 每条业务数据必须具有明确归属。
- 用户数据必须通过服务端鉴权。
- 业务域字段分开管理。
- 内容基础数据与内容版本分开管理。
- 安全规则与用户安全档案分开管理。
- 日程实例与日程任务分开管理。
- 历史任务保留版本引用。
- 避免在一个集合中存储大量无边界嵌套数据。
- 不把所有业务数据集中存放在一个用户文档中。

### 10.3 数据归属

建议统一使用：

```text
userId
profileId
domain
createdAt
updatedAt
```

具体实体按业务需要增加字段。

`userId` 必须来自服务端可信身份上下文。

### 10.4 索引设计

至少评估以下查询：

- 根据 userId 查询档案。
- 根据 profileId 查询安全档案。
- 根据 profileId 和日期范围查询日程。
- 根据 scheduleId 查询任务。
- 根据 profileId 和任务状态查询待执行任务。
- 根据 recipeId 和版本查询内容。
- 根据规则状态及生效时间查询有效规则。
- 根据任务状态和重试时间查询后台任务。
- 根据 `profileId` + `allergyItemId` + `operationId` 查询 `allergy_change_logs`（解除幂等读路径，F-04.3.1-02 / F-04.3.2；02 §7.4）。
- **`allergy_change_logs` 唯一索引**：`profileId + allergyItemId + operationId`（约束成功解除审计；02 §12）。
- **`background_jobs` 唯一索引**：`idempotencyKey`（过敏解除重校验 Job 去重；02 §7.4.1）。
- **`background_jobs` Worker 扫描**：`jobType + status + nextRetryAt`（或 `status + nextRetryAt`）。

云数据库的索引限制、组合查询能力和实际执行计划必须在实施阶段验证。

不能只依据逻辑查询设计假定数据库一定支持对应查询。

## 11. 日程生成幂等与并发控制

### 11.1 幂等目标

同一个档案、同一个目标日期范围、同一版本上下文的重复请求，不应生成重复日程或重复任务。

### 11.2 幂等键

建议生成：

```text
generationKey =
  profileId
  + domain
  + targetStartDate
  + targetEndDate
  + generationVersion
```

具体实现可以使用稳定哈希。

幂等键必须包含能够影响生成结果的关键参数。

不能将请求时间戳作为唯一幂等依据。

### 11.3 并发控制

典型场景：

- 用户主动进入日程页面触发补齐。
- 定时任务同时触发补齐。
- 用户连续点击重新生成。
- 多个云函数同时操作同一档案。

建议：

1. 创建生成任务。
2. 检查相同幂等键是否已存在。
3. 使用事务或条件更新争取任务执行权。
4. 生成日程草稿。
5. 检查档案和规则版本是否发生变化。
6. 写入日程与任务。
7. 标记生成任务完成。
8. 失败时释放或过期处理执行锁。

具体锁机制必须结合云数据库实际支持能力设计。

不能假定数据库天然具备分布式锁。

### 11.4 生成失败处理

- 已经存在的有效任务不能被删除。
- 失败的生成任务需要保留错误信息。
- 可重试错误与不可重试错误必须区分。
- 重试不能创建重复日程。
- 安全校验失败不能通过重试绕过。
- 多次失败应进入可观察的异常状态。

## 12. 任务状态与事务设计

### 12.1 任务状态

建议沿用 `04-schedule-rules.md` 定义的任务状态。

实现前需要将任务状态统一成受控枚举，不能在页面和云函数中自行创造状态名称。

### 12.2 任务操作

以下操作必须由服务端完成：

- 完成。
- 跳过。
- 替换。
- 顺延。
- 阻断。
- 重新安排。

### 12.3 操作原子性

例如替换食谱：

```text
用户提交替换请求
  ↓
检查用户和档案归属
  ↓
检查当前任务状态
  ↓
读取目标食谱版本
  ↓
执行安全校验
  ↓
执行阶段和业务约束校验
  ↓
检查任务版本是否变化
  ↓
更新任务
  ↓
写入操作记录
```

任务更新和操作记录应尽可能通过事务保证一致性。

如果云数据库事务无法覆盖全部操作，应采用状态机和可恢复任务设计，不能出现任务已更新但审计完全丢失的静默失败。

### 12.4 并发操作

对于同一个任务：

- 同时完成和跳过时，只允许一个合法状态转换成功。
- 同时替换时，需要检测任务版本。
- 已完成任务不能被普通替换操作覆盖。
- 已阻断任务不能直接恢复原食谱。
- 顺延后必须重新校验目标日期适用性。

建议采用乐观锁字段，例如 `version`。

更新时要求：

```text
update where
  taskId = expectedTaskId
  and version = expectedVersion
```

只有一个并发请求能够成功更新。

具体查询和条件更新方式应按云数据库实际 API 验证。

## 13. 内容版本管理

### 13.1 内容基础信息与版本分离

建议：

```text
Recipe
  └── RecipeVersion
        ├── ingredients
        ├── instructions
        ├── nutrition
        ├── applicability
        ├── safetyProfile
        ├── sourceReferences
        ├── reviewStatus
        └── version
```

基础食谱用于稳定标识。

版本用于保存具体内容。

### 13.2 日程引用

每条任务必须关联：

- recipeId
- contentVersionId
- contentSnapshotVersion
- safetyRuleVersion

历史日程展示时应能还原当时的食谱内容。

### 13.3 内容更新

食谱发生变化时：

1. 创建新版本。
2. 完成内容审核。
3. 完成安全审核。
4. 发布新版本。
5. 新生成日程使用新版本。
6. 未来未执行任务重新检查是否受影响。
7. 历史完成任务保留原版本。

### 13.4 内容撤回

食谱撤回后：

- 不再进入新的候选池。
- 查询未来未执行任务。
- 对关联任务重新校验。
- 不直接删除历史任务。
- 不覆盖用户已经完成的记录。

## 14. 定时任务与后台任务

### 14.1 一期定时任务

| 任务              | 触发方式   | 目的               |
| ----------------- | ---------- | ------------------ |
| 未来 7 天日程补齐 | 定时触发器 | 检查需要补齐的日程 |
| 日程重新校验      | 后台任务   | 处理规则或档案变化 |
| 食谱撤回处理      | 后台任务   | 检查关联任务       |
| 过期规则检查      | 定时任务   | 检查失效规则       |
| 异常任务重试      | 定时任务   | 重试可恢复任务     |
| 数据质量检查      | 定时任务   | 发现缺失或异常数据 |

### 14.2 定时任务原则

- 定时任务必须幂等。
- 支持分批执行。
- 支持失败重试。
- 记录执行开始、结束及结果。
- 不能假设定时触发器绝对准时。
- 不依赖定时任务完成所有关键安全检查。

关键安全校验必须在实际业务操作时同步执行。

定时重新校验是补充机制，不是安全校验的唯一入口。

### 14.3 后台任务结构

建议：

```text
BackgroundJob
  - jobId / _id
  - jobType
  - profileId
  - allergyItemId（过敏解除重校验等类型必填）
  - operationId（与 AllergyChangeLog 关联）
  - targetId（可选冗余，如 profileId）
  - idempotencyKey
  - status
  - retryCount / attempts
  - nextRetryAt
  - lastError / lastErrorCode
  - leaseOwner（Claim Session ID）/ workerInstanceId（诊断）/ leaseExpiresAt / claimVersion（fencing token，02 §7.4.1、§14.4）
  - createdAt
  - updatedAt
  - completedAt
```

任务状态至少需要支持：

- pending
- processing
- completed（06 其它 Job 类型沿用时可用 `succeeded` 作同义，但过敏解除重校验 Job **须**使用 `completed` 与 02 §7.4.1 一致）
- failed
- retry_wait（可选；与 `pending` + `nextRetryAt` 组合表达退避时二选一落地）

具体字段需与实际任务执行方案保持一致；**过敏解除日程重校验**的字段语义以 02 §7.4.1 为准。

### 14.4 过敏解除日程重校验 Worker（F-04.3.2 / F-04.3.3）

**权威时间**：租约是否有效、是否可回收、续期后的到期时间，均以**服务端当前时间**为准；Worker 本地时钟 **不得**作为租约判断依据。

**可配置参数（实施阶段配置并文档化；下列为待验证的建议默认值，非业务事实）**：

| 参数 | 含义 | 建议默认（待实现验证） |
| --- | --- | --- |
| `leaseDuration` | 单次租约时长 | 如 5 分钟 |
| `leaseRenewInterval` | 长任务续期间隔（须 `< leaseDuration`） | 如 1 分钟 |
| `leaseReclaimScanInterval` | 过期租约扫描周期 | 如 1 分钟 |

#### 14.4.1 原子领取（Claim）

**领取会话（F-04.3.3.1.1）**：每次发起新的 claim 尝试，Worker **必须**先生成新的 **Claim Session ID**（全局唯一、不可复用；**禁止**直接使用稳定的 Worker 运行实例 ID 作为 Claim Session ID）。**仅当**后续单次条件更新影响记录数 = 1 时，该 ID 才成为本次有效领取会话，并与本次原子递增得到的 `claimVersion` **绑定**；CAS 失败、影响 0 或无法确认原子成功的请求 **不得**将已生成的 ID 当作有效执行身份。

Worker（`scheduler` 或 `safety`）**禁止**「先读 Job 再无条件 update」。领取 **必须**为**单次条件更新**（Compare-And-Set），过滤条件 **同时**包含：

1. `_id = jobId`
2. `status = pending`
3. `nextRetryAt` 为空或 ≤ 服务端当前时间
4. **不存在有效租约**：`leaseOwner = null` 且 `leaseExpiresAt = null`（或与 02 §7.4.1 未领取初始态一致）

更新字段 **在同一原子操作中**：

1. `status = processing`
2. `leaseOwner = <本次 claim 在 CAS 前已生成的 Claim Session ID>`
3. `workerInstanceId = <本 Worker 运行实例 ID>`（可选写入，**仅**诊断；**不得**替代步骤 2）
4. `leaseExpiresAt = serverNow + leaseDuration`
5. `claimVersion = claimVersion + 1`（读-改-写须在数据库原子语义内完成）
6. `updatedAt = serverNow`

**仅当**影响记录数 = 1 时领取成功；影响记录数 = 0 时 **不得**执行该 Job 的任何业务逻辑，且 **不得**将本次预生成的 Claim Session ID 用于续期或下游写入。

**同一 Worker 运行实例再次领取同一 Job**：**必须**生成 **新的** Claim Session ID（与上次不同），走完整 §14.4.1；**不得**沿用旧会话 ID。**旧执行流程**恢复时 **必须**仍持有其**原始** `(claimSessionId, claimVersion)`，**禁止**读取 Job 当前 `leaseOwner` / `claimVersion` 并替换本地会话。

**`claimVersion` 与 Claim Session 归属（F-04.3.3.1 / F-04.3.3.1.1，禁止竞态认领）**：

1. Worker **只能**使用**由本次成功原子领取操作产生并绑定**的 `(claimSessionId, claimVersion)` 作为执行权。**禁止**将「按 `_id` 读取到的 Job 文档当前 `claimVersion`」直接当作本执行流程的 fencing token，除非该读取 **同时**证明文档仍归属**本次**领取会话（见下）。
2. **优先**：原子领取 API **必须**在影响记录数 = 1 时返回本次写入后的 Job 快照（至少含 `claimVersion`、`leaseOwner`、`leaseExpiresAt`；可含 `workerInstanceId`）。Worker **必须**自该返回值持久化 `(jobId, claimSessionId, claimVersion)`（`claimSessionId` = 返回值 `leaseOwner`），供 §14.4.2—§14.4.5 使用。
3. **补充读取（仅当平台无法在领取 API 返回更新后文档时）**：在 **已确认** CAS 成功且影响记录数 = 1 后，允许 **一次**按 `_id` 读取；Worker **必须**在采用读到的字段前 **全部**满足：
   - `_id` 与本次领取目标 Job 一致
   - `status = processing`
   - `leaseOwner` **等于**本次 claim **在 CAS 中写入的 Claim Session ID**（与 Worker 本地 `claimSessionId` 一致；**不得**与 `workerInstanceId` 混用）
   - `claimVersion` **等于**本次成功 claim **已绑定**的版本（来自 CAS 返回值、或平台在 CAS 成功时给出的递增结果）；**禁止**因读到的版本号更大而采用 Job 上最新 `claimVersion`
   - `leaseExpiresAt > serverNow`
   - 本次 claim 已确认原子成功（影响记录数 = 1）
4. **不充分条件（必须拒绝）**：仅 `_id` 相同、`status = processing`、`leaseOwner` 等于 Worker 运行实例 ID、租约仍有效 — **不足以**证明属于本次领取会话；**不得**继续业务写入。
5. 任一条件不满足 → **视为领取未完成**，Worker **必须**放弃执行：不得写入 `ScheduleTask`、不得续期、不得完成/失败 Job；仅允许只读诊断。**禁止**通过重试读取「最新 `claimVersion`」或「最新 `leaseOwner`」覆盖或替换步骤 2 / 步骤 3 已合法取得的会话三元组。
6. **典型竞态（必须拒绝）**：Worker A 会话 A1 CAS 成功但未及时获得版本 → A1 租约过期 → Worker B 会话 B1 重新领取递增版本并写入新 `leaseOwner` → A1 再按 `_id` 读取若见 `leaseOwner ≠ A1` 或 `claimVersion` 与 A1 绑定版本不一致，**不得**采用读到的值，**不得**继续业务写入。
7. **同实例双会话（F-04.3.3.1.1，必须拒绝）**：Worker 运行实例 A 会话 A1 领取 `claimVersion = 1` 后暂停 → 租约过期回收 → **同一**运行实例 A 会话 A2 领取 `claimVersion = 2` → A1 恢复后 **不得**因 `workerInstanceId` 相同而通过校验；A1 的续租、下游写入、完成/失败 **必须**失败；A2 在合法会话 + 版本下方可执行。
8. 后续续期、§14.4.4 下游写入、完成/失败/退避 **均须**使用步骤 2 或步骤 3 合法得到的 **同一份** `(claimSessionId, claimVersion)`；**不得**在运行中改用 Job 上更新的会话或版本号，直至本执行流程确认执行权失效并退出。

并发领取：多个 Worker 对同一 Job 仅 **一个** 条件更新成功。

#### 14.4.2 租约续期（Renew）

长任务 **必须**在 `leaseRenewInterval` 周期内尝试续期。续期 **必须**为条件更新，过滤条件 **同时**包含：

- `_id = jobId`
- `status = processing`
- `leaseOwner = §14.4.1 本次领取会话的 Claim Session ID`（**不得**仅用 `workerInstanceId`）
- `claimVersion = §14.4.1 本次领取会话合法取得的版本`
- `leaseExpiresAt > serverNow`（**禁止**对已过期租约续期「复活」）

更新：`leaseExpiresAt = serverNow + leaseDuration`，`updatedAt = serverNow`（**不得**变更 `leaseOwner` / `claimVersion`）。

- 续期成功：可继续业务写入（仍须遵守 §14.4.4 **原子执行权边界**；续期本身 **不得**替代 §14.4.4 与 `ScheduleTask` 写入的原子绑定）。
- 续期失败（影响 0）或网络结果未知：**不得**假设仍持有租约；须重新读取 Job 或再次条件校验（仍须匹配 **本地** Claim Session + `claimVersion`）；**立即停止**新的 `ScheduleTask` 及重校验结果写入；仅允许只读诊断。

#### 14.4.3 过期回收与重新领取

**回收（Reclaim）** — 针对僵死或超时 Worker，**不能**假设旧 Worker 已停止：

条件更新过滤：

- `status = processing`
- `leaseExpiresAt <= serverNow`（租约 **已** 过期）

更新：

- `status = pending`（若同时触发重试退避，可写 `nextRetryAt` 且保持 `pending`）
- `leaseOwner = null`，`workerInstanceId = null`，`leaseExpiresAt = null`
- 按策略 `retryCount` 递增或仅 `updatedAt` 更新（**不得**在此步写 `completed` / `failed` 业务结论）
- **不修改** `claimVersion`（旧 Worker 仍可能持有旧版本号，但 Job 已非其租约下的 `processing` 且版本将在下次领取时被递增淘汰）

**仅当**影响记录数 = 1 时回收成功；多回收 Worker 并发时只有一个成功。

**重新领取**：回收后的 Job 走 §14.4.1 领取流程，**必须**再次递增 `claimVersion`、生成 **新** Claim Session ID 并写入 `leaseOwner` / `leaseExpiresAt`。旧领取会话持有的 `(claimSessionId, claimVersion)` **立即**失去对 Job 状态与下游写入的有效性（**与**旧会话是否来自同一 `workerInstanceId` **无关**）。

**禁止**：在未递增 `claimVersion`、未设置新租约的情况下，允许第二个 Worker 并行执行同一 Job 的业务副作用。

**与「processing → pending 退避」的区别**：可恢复失败时，**当前有效租约持有者**在 §14.4.5 中主动释放租约并置 `pending`；过期回收 **无** 有效租约，由扫描器发起。

#### 14.4.4 执行权隔离与下游写入（F-04.3.3 / F-04.3.3.1 核心）

**原则**：失去租约或 `claimVersion` 已落后的 Worker，**不得**产生任何持久化业务副作用（含 `ScheduleTask` 更新、重校验结果、`TaskActionLog` 追加等）。**禁止**「BackgroundJob 终态更新失败但 ScheduleTask 已被旧 Worker 写入」。

**原子执行权边界（F-04.3.3.1）**：

对 **每一条** 会改变 `ScheduleTask` 或重校验结论的持久化写入，**执行权校验与业务写入必须处于同一个具备并发隔离能力的原子边界内**（同一数据库事务，或经架构评审确认等价的持久化协调原子操作）。**禁止**以下做法作为「隔离」：

- 事务外「先读 Job / 先条件更新 Job，再单独更新 `ScheduleTask`」的普通 check-then-write；
- 将「同一次 Cloud Function 调用内的顺序执行」等同于跨 Worker / 跨调用的互斥锁；
- 在 `ScheduleTask` 已写入后，指望 BackgroundJob 完成/失败 CAS 失败来「撤销」下游副作用。

**本 Worker 持有的执行权前提**（与 §14.4.1 绑定，**不得**在原子边界外单独当作已生效）：

| 校验项 | 要求 |
| --- | --- |
| Job 状态 | `status = processing` |
| 领取会话 | `leaseOwner` = §14.4.1 **本次**合法取得的 Claim Session ID |
| Fencing | `claimVersion` = 与上述 Claim Session **同一次**成功 claim 绑定的版本 |
| 租约时效 | `leaseExpiresAt > serverNow` |
| 诊断（非充分） | `workerInstanceId` 可与本地实例一致，**不得**单独替代上三项 |

**BackgroundJob 状态变更**（完成、失败、退避、错误信息、`retryCount`、`nextRetryAt`）：**必须**使用条件字段（`_id` + `status=processing` + `leaseOwner`（Claim Session ID）+ `claimVersion` + 租约未过期）。旧领取会话延迟提交 **必须**因 Claim Session 不匹配、`claimVersion` 不匹配或租约失效而拒绝（影响 0）。**禁止**通过更新审计日志、重读 Job 最新 token 等方式绕过 fencing。**说明**：Job 终态 CAS 与 `ScheduleTask` 写入 **不是**同一原子边界时，终态 CAS 失败 **不能**修复已发生的 Task 写入；故 Task 写入 **必须先**满足下述原子机制。

**ScheduleTask 重校验写入 — 批准机制（二选一，须先完成 ARC-05 能力验证）**：

**机制 A：跨集合数据库事务（首选，待 ARC-05 验证）**

在同一**已确认具备所需隔离语义**的数据库事务内 **顺序或原子提交**：

1. 对 `background_jobs` 做 **执行权条件更新**（过滤：`status=processing`、`leaseOwner`（Claim Session ID）、§14.4.1 绑定的 `claimVersion`、`leaseExpiresAt > serverNow`；更新可为 touch `updatedAt` 或合法续期字段）；
2. 对目标 `schedule_tasks` 做 **业务条件更新**（`profileId` 匹配、`status` 为未执行态如 `pending`、计划时间在未来等，见 04 §9.5 / 05 §5.3.1）。

**必须**满足：旧 Worker 在新 Worker 已取得执行权后，其事务 **不得**成功提交步骤 2（步骤 1 影响行数为 0 则整事务回滚，**不得**留下 Task 变更）。**不得**因平台提供 `runTransaction` 类 API 即默认满足；须在 ARC-05 中记录：隔离级别、冲突检测、跨集合支持范围、与并发 Worker 的实测结论。

**机制 B：持久化执行协调原子操作（仅当 ARC-05 确认机制 A 不可用且评审批准）**

须设计 **单一原子写入边界**，使「执行权 CAS」与「单条 `schedule_tasks` 条件更新」**不可被旧 Worker 在新 Worker 取得执行权后单独提交**。示例方向（实施时择一落地并回填 ADR/02 字段，**不得**静默引入未文档化字段）：

- 经确认的跨文档事务 batch（与机制 A 等价）；或
- 在 `schedule_tasks`（或专用协调集合）上增加 **与 Job 执行权绑定的镜像条件字段**，且镜像更新与 Task 业务更新 **在同一事务/同一文档原子更新**内与 `background_jobs` 执行权 CAS 联动。

**未选定机制 B 具体方案前，不得**对 `ScheduleTask` 做自动化写入。

**Fail-Closed（F-04.3.3.1，三种状态须区分记录）**：

| 状态 | 含义 | Worker 行为 |
| --- | --- | --- |
| **能力尚未验证** | ARC-05 未完成；跨集合/协调机制是否满足上表 **未知** | **禁止**自动化修改 `ScheduleTask`；Job 保持 `processing` 或进入 **待人工恢复** / 显式失败路径（写入 `lastError` 须含原因码）；须告警 |
| **能力已确认不支持** | 评审结论：当前微信云开发环境 **无法**提供合格原子边界 | 同上；不得降级为 check-then-write |
| **能力支持但本次事务失败** | 提交冲突、超时、结果未知 | **不得**假设 Task 未写入；须按幂等与只读重查恢复；在重新确认 §14.4.1 执行权前 **不得**重复 blindly 写入 |

**租约过期 / 重新领取**：新领取会话取得执行权后 `claimVersion` 递增；旧领取会话任何机制 A/B 提交 **必须**在步骤 1 失败并整笔回滚。**旧执行流程从暂停中恢复**且本地 `(claimSessionId, claimVersion)` 已失效：**必须**停止写入并退出，**不得**提交缓存的 Task 变更，**不得**用 Job 上最新会话/版本续命。

**幂等**：同一 Job 对同一 `ScheduleTask` 的重复重校验写入须保持业务幂等（§14.4.6）；执行权隔离 **补充** 幂等，**不替代** `idempotencyKey`。

**读操作**：扫描未来任务可为只读；**任何**改变 `ScheduleTask` 或写入重校验结论的 operation 均受本节约束。

#### 14.4.5 完成、失败与重试

**成功完成**：全部目标 `ScheduleTask` 写入均已通过 §14.4.4 **机制 A/B 原子边界**（或无需写入的只读结论）后，再对 Job 做条件更新 → `completed`，写入 `completedAt`，释放租约（`leaseOwner=null`，`workerInstanceId=null`，`leaseExpiresAt=null`）；可 emit `safety_revalidation_completed`（05 事件表）。**禁止**在仍有未通过原子边界的 Task 写入计划时标记完成。

**可恢复失败**：在 **未** 通过 §14.4.4 原子边界写入的前提下，若仍持有有效执行权，可对 Job 做条件更新：`retryCount` 递增，计算 `nextRetryAt`（指数退避，最大次数等见下），`status=pending`，**释放租约**。若失败原因属于 §14.4.4 Fail-Closed（能力未验证/不支持/事务结果未知且无法确认 Task 状态），**不得**用普通退避替代 Fail-Closed 记录。

**超过最大重试或不可恢复错误**：条件更新 → `status=failed`，写入 `lastError` / `retryCount`，释放租约；**禁止**静默删除。

**人工恢复**：管理端或运维 API 须支持对 `failed` Job 重新置 `pending`（新 `operationId` **不**要求；仍关联原解除 `operationId`）或创建补偿 Job；须审计；清空租约字段，**不重置** `claimVersion`。

**退避参数（可配置，建议默认待实现验证）**：最大重试 **5** 次、`baseDelay` 1 分钟、指数退避；`retry_wait` 与 `pending` + `nextRetryAt` 二选一落地。
#### 14.4.6 执行范围（业务，F-04.3.2 保持不变）

1. 仅重校验该 `profileId` 下 **未来尚未执行** 的 `ScheduleTask`（与 04 §9.5、05 §5.3.1 一致）。
2. **不得**修改已完成历史 `ScheduleTask`。
3. **不得**绕过 S1 过敏过滤（05 §5.3.2）。
4. 执行逻辑须 **幂等**：同一 Job 重复消费不得重复错误阻断或重复写入矛盾状态（在 **§14.4.1 合法** `claimVersion` 持有者下，且 §14.4.4 原子边界内）。

#### 14.4.7 异常场景（租约与执行权）

| 场景 | 触发条件 | Job / 租约变化 | 是否允许业务写入 | 恢复 |
| --- | --- | --- | --- | --- |
| Worker 正常完成 | 重校验结束 | 条件 → `completed`，释放租约 | 每条 Task 写入须 §14.4.4 机制 A/B | — |
| Worker 崩溃 | 进程退出 | 租约到期后 §14.4.3 回收 | 崩溃后旧 Worker 无写入权 | 扫描回收 + 重新领取 |
| 执行超过租约 | 未续期 | 过期后回收；或续期成功则延长 | 续期失败则停止写入 | 续期 / 回收 |
| A 过期，B 重新领取 | `leaseExpiresAt` 到期 | B 领取 `claimVersion++` | A **禁止**（机制 A/B 步骤 1 失败）；B 在验证 §14.4.1 版本后允许 | 自动 |
| A 失租约后尝试写入 | A 版本过期或租约失效 | 无合法 Job 终态 | **拒绝**；Task 写入须在原子边界内失败 | A 应退出 |
| A 暂停后恢复 | 租约已过期或被新领取会话取代 | 新会话持有新 `(leaseOwner, claimVersion)` | 旧会话 **禁止**；不得用 `_id` 读取的最新会话/版本继续写；**不得**因同一 `workerInstanceId` 放行 | 旧会话退出；新会话或回收 |
| 同实例 A1 暂停后 A2 领取（F-04.3.3.1.1） | 实例 A 会话 A1 租约过期；A 会话 A2 领取 `claimVersion++` | Job 上 `leaseOwner`=A2；A1 本地仍为 A1 | A1 **禁止**（会话/版本不匹配）；A2 在 §14.4.1 合法三元组下允许 | A1 退出 |
| 领取后补充读失败 | §14.4.1 步骤 3 不满足 | 不视为有效领取 | **禁止**一切业务写入 | 放弃本次执行 |
| 两 Worker 同时领取 | 并发 CAS | 一者成功递增版本 | 失败者 **禁止** | — |
| 两 Worker 同时回收 | 并发 CAS | 一者成功置 `pending` | 回收过程无业务写入 | — |
| A 延迟提交完成 | B 已领取或已完成 | CAS 失败 | **拒绝**（**不能**撤销已发生的 Task 写入，故 A 不应再产生 Task 写） | 人工查 Job / Task 状态 |
| 续期结果未知 | 网络超时 | 不得假设成功 | **暂停写入**直至重新确认 §14.4.1 执行权 | 重读 Job 或重试续期 |
| ARC-05 未验证 | 机制 A/B 未确认 | Job 可保持 `processing` 或 Fail-Closed | **禁止** Task 自动化写入 | 人工 / 能力验证后恢复 |
| 事务提交结果未知 | 超时/网络 | 不得假设回滚或成功 | **暂停**；幂等重查 Task 与 Job 后再决策 | 运维流程 |
| 重复处理同一 Job | 重试 / 重新领取 | 版本递增 | 仅当前 §14.4.1 合法版本持有者；业务幂等仍生效 | §14.4.1 / §14.4.3 |

**系统重启**：启动时扫描 `pending` 及 **租约过期**的 `processing`（§14.4.3），**不得**仅按 `updatedAt` 阈值回收而忽略 `leaseExpiresAt`。

**与 AllergyChangeLog 的边界**：Worker **不得**更新 append-only `AllergyChangeLog` 以反映进度；进度仅存在于 `background_jobs`。

## 15. 轻量内容管理后台

### 15.1 一期定位

一期需要轻量、受控的内容维护能力，不建设复杂 CMS。

后台主要服务于：

- 食材维护。
- 过敏原字典维护。
- 食谱维护。
- 食谱审核。
- 安全属性维护。
- 安全规则维护。
- 内容发布与撤回。
- 规则版本管理。
- 审计记录查看。

### 15.2 权限角色

建议：

| 角色       | 权限                 |
| ---------- | -------------------- |
| 内容编辑   | 创建和编辑草稿       |
| 内容审核   | 审核食谱和内容       |
| 专业审核   | 审核安全及专业适用性 |
| 发布管理员 | 发布审核通过的版本   |
| 系统管理员 | 管理角色和系统配置   |

普通用户不能访问管理接口。

### 15.3 后台技术形态

一期可以选择：

- 受控的简单 Web 管理页面。
- 微信生态支持的管理工具。
- 基于现有项目能力实现的内部维护页面。

选型原则：

- 不把后台权限暴露给普通小程序用户。
- 所有管理操作在云端重新鉴权。
- 敏感规则变更保留审计记录。
- 不允许前端直接修改已发布规则。
- 不允许未审核内容进入生产推荐池。

具体后台技术方案在实施前根据团队能力和部署条件确认。

## 16. 权限与数据安全

### 16.1 身份认证

用户端使用微信登录体系。

服务端应通过可信调用上下文识别当前用户。

不能相信客户端传入的：

- userId
- openId
- role
- isAdmin
- permission

这些字段如果存在，必须由服务端重新获取或校验。

### 16.2 数据访问控制

用户只能访问自己拥有或被明确授权访问的档案。

每次查询和写入都应校验：

1. 当前用户身份。
2. 档案归属。
3. 当前操作权限。
4. 目标数据归属。
5. 当前业务状态是否允许操作。

### 16.3 数据库权限

客户端不应直接拥有修改核心业务数据的权限。

尤其是：

- 过敏禁忌。
- 安全档案。
- 过敏变更审计（`allergy_change_logs`）。
- 日程任务。
- 内容发布状态。
- 安全规则。
- 管理员权限。

这些操作必须经过云函数的权限检查和业务校验。

数据库安全规则需要按实际集合逐一配置，并在测试环境验证越权访问。

### 16.4 敏感数据

过敏信息、孕期信息和幼儿信息应按照适用法律法规进行敏感个人信息管理。

要求：

- 最小必要采集。
- 明确处理目的。
- 提供必要告知。
- 按要求取得单独同意或履行其他适用法律要求。
- 严格限制后台访问。
- 敏感操作记录审计。
- 不默认用于广告画像。
- 不在普通日志中输出完整敏感信息。

具体法律合规要求应由项目法务或隐私负责人确认。

## 17. 日志、监控与埋点

### 17.1 技术日志

至少记录：

- requestId
- cloudFunction
- operation
- executionStatus
- duration
- errorCode
- retryCount
- createdAt

禁止在普通日志中记录完整过敏信息、医疗资料及其他不必要的敏感数据。

### 17.2 业务日志

建议关注：

- 日程生成成功率。
- 日程生成失败原因。
- 候选食谱不足情况。
- 食谱安全拦截数量。
- 手动替换成功率。
- 任务操作失败率。
- 日程重新校验结果。
- 内容审核及发布状态。

### 17.3 行为埋点

沿用 PRD 和前序 Spec 已定义的事件：

- profile_created
- schedule_generated
- schedule_viewed
- task_completed
- task_skipped
- recipe_replaced
- recipe_viewed
- schedule_deferred
- stage_changed
- allergy_status_updated
- food_observation_created
- safety_rule_triggered

埋点不得默认收集不必要的医疗敏感详情。

## 18. 异常处理

### 18.1 异常分类

| 异常           | 处理方式                   |
| -------------- | -------------------------- |
| 用户未登录     | 引导登录                   |
| 档案不存在     | 提示创建或重新选择档案     |
| 档案无权限     | 拒绝访问                   |
| 食谱不存在     | 返回内容不可用             |
| 食谱未发布     | 不进入候选池               |
| 安全校验失败   | 阻断候选                   |
| 安全服务不可用 | 停止本次生成               |
| 日程重复生成   | 返回已有有效结果或任务状态 |
| 任务状态冲突   | 返回冲突错误，要求刷新     |
| 定时任务失败   | 记录并按策略重试           |
| 数据库写入失败 | 不返回成功，保留恢复依据   |
| 内容版本失效   | 重新读取有效版本并校验     |

### 18.2 错误码建议

| 错误码                       | 说明             |
| ---------------------------- | ---------------- |
| `UNAUTHORIZED`               | 未授权           |
| `FORBIDDEN`                  | 无操作权限       |
| `PROFILE_NOT_FOUND`          | 档案不存在       |
| `PROFILE_VERSION_CONFLICT`   | 档案版本冲突     |
| `CONTENT_NOT_AVAILABLE`      | 内容不可用       |
| `SAFETY_CHECK_BLOCKED`       | 安全规则阻断     |
| `SAFETY_CHECK_UNAVAILABLE`   | 安全服务不可用   |
| `SCHEDULE_GENERATION_FAILED` | 日程生成失败     |
| `SCHEDULE_CONFLICT`          | 日程冲突         |
| `TASK_VERSION_CONFLICT`      | 任务版本冲突     |
| `TASK_INVALID_TRANSITION`    | 非法任务状态转换 |
| `IDEMPOTENCY_CONFLICT`       | 幂等请求冲突     |
| `VALIDATION_FAILED`          | 参数校验失败     |
| `INTERNAL_ERROR`             | 内部错误         |

错误码属于技术接口定义，不应直接作为用户可见文案。

## 19. 测试策略

### 19.1 单元测试

重点测试：

- 孕周计算。
- 幼儿年龄计算。
- 阶段解析。
- 日程槽位生成。
- 食谱候选筛选。
- 营养结构匹配。
- 安全规则判断。
- 过敏原映射。
- 任务状态转换。
- 幂等键生成。

### 19.2 集成测试

重点测试：

- 用户建档。
- 档案读取和权限隔离。
- 日程生成。
- 食谱替换。
- 任务跳过。
- 任务顺延。
- 安全档案更新。
- 未来日程重新校验。
- 食谱版本发布。
- 食谱撤回。
- 定时任务补齐。

### 19.3 安全测试

必须覆盖：

- 用户 A 不能访问用户 B 的档案。
- 用户不能通过传入其他 profileId 修改他人数据。
- 手动替换不能绕过过敏禁忌。
- 顺延不能绕过安全校验。
- 安全服务异常不能默认放行。
- 未审核食谱不能进入候选池。
- 未审核安全规则不能发布。
- 普通用户不能调用管理接口。
- 规则版本更新后未来任务能够重新校验。

### 19.4 并发测试

至少覆盖：

- 同一档案并发生成日程。
- 同一任务并发替换。
- 完成和跳过同时发生。
- 用户操作与定时任务同时发生。
- 安全档案更新与日程生成同时发生。
- 内容撤回与任务补齐同时发生。

### 19.5 质量门禁

每次代码提交应执行：

- 静态检查。
- 业务逻辑单元测试。
- 安全规则测试。
- 关键云函数集成测试。
- 构建或微信开发者工具编译检查。

具体脚本名称以项目实际 package 配置为准，不得假定项目一定使用某个测试框架。

## 20. 环境与部署

### 20.1 环境划分

建议至少：

- Development
- Production

如团队需要稳定验收环境，可增加：

- Staging

每个环境应使用独立云开发环境和独立数据。

### 20.2 配置管理

环境配置应包含：

- 云开发环境 ID。
- 云函数配置。
- 文件存储配置。
- 定时任务配置。
- 管理后台配置。
- 内容版本配置。

敏感配置不能硬编码到公开前端代码中。

### 20.3 发布流程

```text
开发
  ↓
本地检查
  ↓
单元测试
  ↓
云开发测试环境部署
  ↓
集成测试
  ↓
安全规则测试
  ↓
产品验收
  ↓
生产发布审批
  ↓
生产部署
  ↓
监控和异常检查
```

### 20.4 数据变更

数据库集合、索引、初始化数据及规则版本需要有可追踪的变更记录。

不要通过直接修改生产数据库的方式替代正常发布流程。

生产数据修复必须具备：

- 操作记录。
- 影响范围。
- 操作前后数据。
- 回滚或恢复方案。
- 执行人及审批人。

## 21. 一期实施优先级

建议按照以下顺序实施。

### Phase 1：工程基础

- 微信小程序项目初始化。
- 云开发环境配置。
- 用户身份识别。
- 基础云函数调用封装。
- 数据库集合和权限配置。
- 统一错误码和日志结构。

### Phase 2：用户与档案

- 用户初始化。
- 多档案模型。
- 孕期档案。
- 幼儿档案。
- 阶段计算。
- 档案版本管理。

### Phase 3：内容中心

- 食材数据模型。
- 食谱数据模型。
- 食谱版本。
- 内容审核状态。
- 来源依据管理。
- 轻量维护工具。

### Phase 4：安全规则中心

- 用户安全档案。
- 过敏原字典。
- 食材安全属性。
- 食谱安全属性。
- 安全规则执行器。
- 统一安全校验接口。
- 安全审计。
- 规则版本管理。

### Phase 5：日程引擎

- 日程槽位生成。
- 候选食谱读取。
- 阶段过滤。
- 安全过滤。
- 营养结构匹配。
- 个性化排序。
- 日程写入。
- 幂等控制。

### Phase 6：任务系统

- 任务查询。
- 完成任务。
- 跳过任务。
- 替换食谱。
- 顺延任务。
- 任务状态机。
- 并发冲突处理。

### Phase 7：定时任务和闭环

- 未来 7 天日程补齐。
- 安全信息变更后的重新校验。
- 内容撤回后的任务处理。
- 规则更新后的任务处理。
- 行为埋点。
- 异常监控。

### Phase 8：验收与发布

- 业务流程测试。
- 数据权限测试。
- 安全规则测试。
- 并发测试。
- 微信小程序体验验收。
- 生产发布。

## 22. 一期架构验收标准

### 22.1 架构边界

- [ ] 孕期与幼儿业务域独立。
- [ ] 共享能力通过明确接口调用。
- [ ] 页面不直接实现核心业务规则。
- [ ] 日程生成逻辑独立于页面。
- [ ] 安全规则统一执行。
- [ ] 内容、规则和日程版本可追溯。

### 22.2 数据与权限

- [ ] 支持一个用户多个档案。
- [ ] 用户数据按档案隔离。
- [ ] 服务端鉴权有效。
- [ ] 普通用户无法修改内容和规则。
- [ ] 敏感信息访问受控。
- [ ] 关键操作有审计记录。

### 22.3 日程与任务

- [ ] 支持未来 7 天日程。
- [ ] 重复请求不产生重复任务。
- [ ] 日程候选不足时不放宽安全约束。
- [ ] 替换、跳过、顺延符合状态机。
- [ ] 并发操作有冲突检测。
- [ ] 失败任务能够恢复或重试。

### 22.4 安全规则

- [ ] 安全规则中心独立。
- [ ] 安全校验覆盖自动生成和所有用户操作入口。
- [ ] 安全服务异常时停止自动放行。
- [ ] 过敏禁忌能够硬过滤。
- [ ] 食材与食谱安全属性可追溯。
- [ ] 安全档案变化能够触发未来任务重新校验。
- [ ] 未审核规则不能发布。

### 22.5 可维护性

- [ ] 业务规则可独立测试。
- [ ] 规则版本可管理。
- [ ] 内容版本可管理。
- [ ] 云函数职责清晰。
- [ ] 错误码统一。
- [ ] 日志和埋点可追踪。
- [ ] 定时任务可监控。
- [ ] 项目具备基本部署和回滚流程。

## 23. 待确认事项

| 编号   | 待确认事项                              | 建议处理阶段   |
| ------ | --------------------------------------- | -------------- |
| ARC-01 | 项目最终采用 JavaScript 还是 TypeScript | 工程初始化     |
| ARC-02 | 微信云开发具体环境及 SDK 版本           | 工程初始化     |
| ARC-03 | 是否采用独立构建工具管理共享 package    | 工程初始化     |
| ARC-04 | 轻量管理后台具体实现方式                | 内容中心开发   |
| ARC-05 | 云数据库事务能力和限制（**须**覆盖 §14.4.4 机制 A/B 与 08 §8.4 多文档事务；未完成前 Worker **不得**自动化写入 `ScheduleTask`） | 日程引擎开发（**阻塞**过敏解除重校验 Worker 下游写入） |
| ARC-06 | 云函数运行时和超时限制                  | 后端实施       |
| ARC-07 | 定时任务批处理和重试策略                | 后台任务开发   |
| ARC-08 | 一期用户数据及内容数据规模预估          | 数据库设计     |
| ARC-09 | 日程生成同步与异步处理边界              | 日程引擎开发   |
| ARC-10 | 数据保留、删除及审计期限                | 上线前合规评审 |
| ARC-11 | 微信小程序订阅消息是否纳入一期          | 产品范围确认   |
| ARC-12 | 生产发布与数据库变更审批流程            | 上线准备       |

这些事项不应阻止整体架构设计，但必须在对应实施阶段确认。

## 24. 架构决策记录（ADR）

### ADR-001：采用微信云开发

**决策**：一期采用微信小程序原生框架与微信云开发。

**原因**：

- 降低后端基础设施维护成本。
- 充分利用微信生态能力。
- 适合一期轻量业务规模。
- 减少额外部署和运维复杂度。

### ADR-002：采用模块化单体

**决策**：业务模块在同一后端工程内保持清晰边界，不提前拆分微服务。

**原因**：

- 一期团队规模和业务复杂度尚不足以支撑微服务运维成本。
- 模块化边界能够为后续拆分提供基础。
- 更容易保持业务规则一致性。

### ADR-003：规则引擎采用纯函数模块

**决策**：使用 TypeScript/JavaScript 实现可测试的确定性规则模块。

**原因**：

- 规则行为透明。
- 易于单元测试。
- 不需要额外规则引擎基础设施。
- 适合一期阶段和内容规模。

### ADR-004：安全规则独立

**决策**：Safety Service 是独立业务模块，统一提供安全校验。

**原因**：

- 安全逻辑不能散落在页面和各业务入口。
- 自动生成、替换、顺延和重新生成必须遵循同一套规则。
- 便于审计、测试、版本管理及后续扩展。

### ADR-005：内容与业务规则分离

**决策**：食材和食谱作为内容数据维护，业务约束通过规则配置执行。

**原因**：

- 内容可以独立维护和审核。
- 规则可以独立升级。
- 避免修改一个食谱就需要修改程序逻辑。
- 支持未来增加其他孕期和幼儿阶段。

### ADR-006：历史任务保留版本快照

**决策**：日程任务关联食谱内容版本、规则版本及必要的生成上下文。

**原因**：

- 保留历史推荐依据。
- 支持内容撤回后的影响排查。
- 支持安全规则升级后的重新校验。
- 避免历史记录随当前内容更新而变化。

---

## 25. 下一阶段建议

完成 `06-technical-architecture.md` 后，建议进入工程落地前的 Spec 一致性审查。

建议建立：

`specs/07-spec-consistency-review.md`

重点审查：

1. PRD 与业务流程是否一致。
2. 业务流程与数据模型是否一致。
3. 数据模型与内容规范是否一致。
4. 内容规范与日程规则是否一致。
5. 日程规则与安全规则是否一致。
6. 安全规则与技术架构是否一致。
7. 数据实体、字段、状态和枚举是否存在重复定义。
8. 一期范围是否出现未经批准的扩展。
9. 待确认事项是否有明确责任人和决策阶段。
10. 是否存在无法通过现有技术架构实现的业务要求。

在完成一致性审查并解决阻塞问题后，再进入项目初始化和实际编码。

**工程实施原则：先保证数据模型、内容审核、安全校验和日程生成闭环正确，再建设完整 UI 和体验优化。**
