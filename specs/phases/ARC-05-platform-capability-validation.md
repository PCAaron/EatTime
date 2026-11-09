# ARC-05 — 微信云开发事务与 Worker 并发安全能力验证任务书

| 字段 | 值 |
| --- | --- |
| 任务编号 | ARC-05 |
| 文档类型 | **平台能力验证任务书**（非验证结果报告） |
| 整体状态 | **NOT STARTED** |
| 创建日期 | 2026-11-09 |
| 关联审查项 | F-04.3.2-02、F-04.3.3（含 F-04.3.3-05 P0）、06 §14.4.4 机制 A/B |
| 前置 Spec | 02 §7.4.1、06 §14.3—§14.4.7、07 §15.7—§15.8 |

---

## 1. 任务背景与目标

EatTime 一期采用**微信小程序原生框架 + 微信云开发**（见 `PRD.md` §0、06 §2、ADR-001）。过敏解除后的日程重校验通过 **Transactional Outbox**（`BackgroundJob`）异步执行，Worker 对 `ScheduleTask` 的写入必须与 Job **执行权（Claim Session + `claimVersion`）** 处于同一具备可靠并发隔离的原子边界（06 §14.4.4）。

F-04.3.3.1.1 已在 **Spec 层**完成 Claim Session ID、`claimVersion` fencing 与 §14.4 规则收敛；**不得**将 Spec 层 PASS 等同于平台或实现 PASS（07 §15.8）。

**本轮目标**：

1. 在**不修改**既有设计规则的前提下，验证微信云开发能否可靠实现这些规则。
2. 形成可独立审查、可执行、可留存证据的验证任务书（本文档）。
3. 实验设计先行；**本轮不执行实验、不实现生产 Worker**。

**本轮非目标**：

- 编写生产 Worker、修改数据模型、修改业务 Spec、改云函数业务代码、创建生产索引/集合、提交 Git。

---

## 2. 当前架构约束（引用，不重复定义）

实施与验证 **必须** 遵守下列已收敛约束；细节以源文档为准。

| 约束 | 权威出处 |
| --- | --- |
| `leaseOwner` = Claim Session ID；`workerInstanceId` 仅诊断；执行权 = `(jobId, claimSessionId, claimVersion)` 三元组 | 02 §7.4.1；06 §14.4.1 |
| `claimVersion` 为 Job 级 fencing token，与当次 Claim Session 绑定 | 02 §7.4.1；06 §14.4.1 |
| Claim / Renew / Reclaim / 下游写入 / Complete / Fail / Retry 均验证 Claim Session + `claimVersion` | 06 §14.4.1—§14.4.5 |
| **禁止** check-then-write（事务外先验权再写 `ScheduleTask`） | 06 §14.4.4 |
| 机制 A（跨集合事务）首选；机制 B 仅 ARC-05 证明 A 不可用且评审批准；否则 **Fail-Closed** | 06 §14.4.4 |
| Claim 为 **单次** `where` 条件更新（CAS），影响记录数 = 1 | 06 §14.4.1；02 §7.4.1 |

---

## 3. 验证范围（ARC-05.1 — ARC-05.4）

| 子任务 | 目标 | 任务书状态 | 验证结论状态 |
| --- | --- | --- | --- |
| ARC-05.1 | 微信云开发事务能力、隔离语义与限制 | 方案已定义 | **NOT STARTED** |
| ARC-05.2 | 原子 Claim 与 Claim Session 绑定（场景 A/B/C） | 实验已设计 | **NOT STARTED** |
| ARC-05.3 | Claim Session fencing 与过期 Worker 防护 | 实验已设计 | **NOT STARTED** |
| ARC-05.4 | Job 执行权验证与 `ScheduleTask` 下游写入原子性（**P0**） | 实验已设计 | **NOT STARTED** |

---

## 4. 平台环境信息（执行前必填）

> **待确认**：本节在首次实验前由执行人填写；任务书阶段仅列出检查项。

| 检查项 | 记录值 | 状态 |
| --- | --- | --- |
| 云开发环境 ID | | 待确认 |
| 环境类型（测试 / 生产专用验证环境） | | 待确认 |
| 微信开发者工具版本 | | 待确认 |
| 小程序基础库版本 | | 待确认 |
| 云函数运行时（Node 版本） | | 待确认 |
| `wx-server-sdk` 锁定版本 | | 待确认 |
| 云函数 `cloud.init({ timeout })` 配置 | 官方默认 15s（见 server.init） | 待实测 |
| 验证专用集合命名前缀 | 建议 `arc05_*` 隔离 | 待创建 |
| 与 06 §2.3 / ARC-02 的关系 | 对齐工程初始化记录 | 待对齐 |

---

## 5. 官方资料调研

### 5.1 调研原则

1. 仅以腾讯**微信开放文档**（云开发 / 云数据库）为能力依据；第三方博客不作为最终依据。
2. API 存在 ≠ 满足本项目并发安全要求；隔离级别须对照 06 §14.4.4。
3. 官方未说明项标注 **官方未说明**，不得标 PASS。
4. 文档歧义须通过 **ARC-05-EXP-*** 实验闭合。
5. 下列「桌面调研」完成于任务书编写时（2026-11-09）；**不构成 ARC-05 子任务 PASS**。

### 5.2 能力调研登记表

| 能力名称 | 官方文档 | 官方链接 | 文档版本/更新时间 | 文档原文或准确摘要 | 对 ARC-05 的意义 | 证据等级 |
| --- | --- | --- | --- | --- | --- | --- |
| 云数据库事务入口 | 事务（开发指引） | https://developers.weixin.qq.com/miniprogram/dev/wxcloudservice/wxcloud/guide/database/transaction.html | 2026-08-24 | 仅云函数端；`wx-server-sdk` ≥ 1.7.0；支持跨多记录、跨多集合原子操作 | ARC-05.1、ARC-05.4 机制 A 前提 | **官方明确** |
| `runTransaction` | Database.runTransaction | https://developers.weixin.qq.com/miniprogram/dev/wxcloudservice/wxcloud/reference-sdk-api/database/Database.runTransaction.html | 2026-04-15 | 仅云函数；冲突时自动重试，默认最多 3 次；回调正常返回后 SDK 自动提交 | ARC-05.1 Q7、EXP-06 | **官方明确** |
| `startTransaction` | Database.startTransaction | https://developers.weixin.qq.com/miniprogram/dev/wxcloudservice/wxcloud/reference-sdk-api/database/Database.startTransaction.html | 2026-04-15 | 需手动 `commit`/`rollback`；限制同 runTransaction | EXP-07 可控提交时机 | **官方明确** |
| 事务内操作形态 | 事务 / Transaction.collection | https://developers.weixin.qq.com/miniprogram/dev/wxcloudservice/wxcloud/guide/database/transaction.html | 2026-08-24 | 事务内**不支持** `where` 批量操作；仅 `doc` 单记录与 `add` | 机制 A 步骤 1 **不能**在事务内使用 `where` CAS；须 `doc.get`+校验+`doc.update` 或实验等价方案 | **官方明确** |
| 隔离等级 | 事务 — 隔离等级 | 同上 | 2026-08-24 | **快照隔离**；**无串行化**；**无法避免写偏（write skew）** | ARC-05.4 与 Spec「可靠并发隔离」差距评估的核心依据 | **官方明确** |
| 快照与事务锁 | 事务 — 快照隔离 | 同上 | 2026-08-24 | 写操作加事务锁；其他事务写入直接失败；普通更新阻塞至锁释放或超时 | 两事务竞争同一 `background_jobs` 文档时的预期行为（须 EXP-05 实测） | **官方明确** |
| 跨集合事务 | 事务 — 单记录操作说明 | 同上 | 2026-08-24 | 可在同一事务中对集合 A 的 `x`、`y` 与集合 B 的 `z` 操作 | `background_jobs` + `schedule_tasks` 跨集合可行性（ARC-05.4 Q4） | **官方明确** |
| 条件更新（CAS） | Collection.update | https://developers.weixin.qq.com/miniprogram/dev/wxcloudservice/wxcloud/reference-sdk-api/database/collection/Collection.update.html | 2026-07-04 | `where(...).update` 返回 `stats.updated`；0 条匹配时 API 仍可成功 | ARC-05.2 Claim/Reclaim **事务外** CAS 依据 | **官方明确** |
| 条件更新指引 | 更新（增删查改） | https://developers.weixin.qq.com/miniprogram/dev/wxcloudservice/wxcloud/guide/database/update.html | — | 云函数内 `where` + `update`；`_.inc` 等原子指令 | Claim `claimVersion` 递增语义 | **官方明确** |
| 事务内条件 CAS | — | — | — | 未描述事务 `doc.update` 是否支持服务器端条件谓词（仅应用层读快照后判断） | 机制 A 步骤 1「影响行数 = 0 则回滚」在事务内的落法 | **官方未说明** |
| 事务超时上限 | — | — | — | 事务指南未给出事务专属超时 | EXP-07 | **官方未说明** |
| 单事务文档数量上限 | — | — | — | 未在事务页列出 | EXP-09 | **官方未说明** |
| 提交结果未知 / 幂等 commit | — | — | — | 未描述网络中断后 commit 是否可查询 | EXP-07 | **官方未说明** |
| 云函数 API 默认超时 | server.init | https://developers.weixin.qq.com/miniprogram/dev/wxcloudservice/wxcloud/reference-sdk-api/init/server.init.html | — | `cloud.init` 的 `timeout` 默认 **15000 ms**（1.7.0+） | 长事务与暂停 Worker 实验的环境约束 | **官方明确** |
| 小程序端事务 | — | — | — | 事务 API 标注「仅云函数」 | Worker 须在云函数内执行写入验证 | **官方明确** |

### 5.3 ARC-05.1 调查问题清单与当前证据状态

| # | 问题 | 官方/实验状态 | 责任实验 |
| --- | --- | --- | --- |
| 1 | 是否支持事务？ | 官方：**是**（云函数） | — |
| 2 | 事务覆盖哪些操作？ | 官方：`doc` get/update/set/remove、`add`；无 `where`/聚合 | — |
| 3 | 同事务读写多文档？ | 官方：**支持** | ARC-05-EXP-05 |
| 4 | 跨 `background_jobs` + `schedule_tasks`？ | 官方：允许多集合多 `doc` | ARC-05-EXP-05 |
| 5 | 提交时可靠冲突检测？ | 官方：冲突时 runTransaction 重试；细节未等价于串行化 | ARC-05-EXP-05、EXP-06 |
| 6 | 两事务并发改同一 Job？ | 官方：事务锁 + 他事务写入失败 | ARC-05-EXP-05 |
| 7 | 重试责任方？ | 官方：`runTransaction` 平台重试（默认 3 次），其他异常不重试 | EXP-06 |
| 8 | 超时/中断后能否确定 commit？ | **官方未说明** | EXP-07 |
| 9 | 时间/数量限制？ | 事务页未说明；云函数默认 15s API 超时 | EXP-07、EXP-09 |
| 10 | SDK/环境差异？ | **待确认**（填 §4） | 全部实验 |

### 5.4 桌面调研对机制的初步含义（推断，非 PASS）

下列为任务书阶段的**工程含义分析**，须由实验否定或确认：

1. **Claim（ARC-05.2）**：宜使用云函数内 `where` + `update` + `stats.updated`（与 06 §14.4.1 一致），**在事务外**完成。
2. **机制 A（ARC-05.4）**：事务内无法 `where` CAS；步骤 1 可能形态为 `transaction.collection('background_jobs').doc(jobId).get()` → 应用层校验 `(leaseOwner, claimVersion, leaseExpiresAt)` → `doc.update` touch → 再写 `schedule_tasks`。该形态是否满足 06「旧 Worker 在新 Reclaim 后不得提交步骤 2」，**取决于**快照隔离 + 文档锁是否在 Reclaim 提交时使旧事务失败 — **须 EXP-04/05**。
3. **写偏风险**：官方明确快照隔离**不能**避免 write skew。若 Reclaim 与旧 Worker 的下游事务涉及**不同谓词集合**的读写，是否存在 Spec 禁止的旁路 — **须专门设计 EXP-05 时序**。
4. **不得**因存在 `runTransaction` 即认定 ARC-05.4 PASS（06 §14.4.4 已要求）。

---

## 6. 事务能力分析（验证方案，非结论）

### 6.1 机制 A 参考实现形态（仅供实验编码，非生产）

```text
// 云函数内 — 实验伪代码，生产 Worker 禁止在未 PASS ARC-05.4 前使用
await db.runTransaction(async (tx) => {
  const job = await tx.collection('arc05_background_jobs').doc(jobId).get()
  // 校验本地 claimSessionId + claimVersion + leaseExpiresAt（三元组）
  if (!sessionValid(job.data, localTriple)) await tx.rollback('SESSION_INVALID')
  // touch Job（或合法续期字段）— 意图：提交时与并发 Reclaim 竞争同一 doc
  await tx.collection('arc05_background_jobs').doc(jobId).update({ data: { updatedAt: db.serverDate() } })
  await tx.collection('arc05_schedule_tasks').doc(taskId).update({ data: { /* 业务字段 */ } })
})
```

**实验须验证**：Reclaim 在步骤之间插入后，旧事务是 **rollback**、**冲突重试后失败** 还是 **错误提交**。

### 6.2 机制 B 方向（仅当 A 失败且评审批准）

见 §10；本轮不实施、不选定镜像字段方案。

### 6.3 Fail-Closed 对齐（06 §14.4.4）

验证未 PASS 前，任何自动化 `ScheduleTask` 写入实验也须使用 **隔离集合** `arc05_schedule_tasks`，不得触碰生产集合。

---

## 7. 子任务验证方案

### 7.1 ARC-05.1 — 微信云开发事务能力

**通过标准**：事务范围、隔离语义、冲突处理对 06 §14.4.4 的覆盖度有**官方证据 + 实验结论**闭环；未知项已列入 P0/P1。

**活动**：

1. 补全 §4 环境信息。
2. 复核 §5.2 登记表（SDK 升级时重新抓取文档日期）。
3. 执行 EXP-05、EXP-06、EXP-07、EXP-09。
4. 输出「ARC-05.1 结论文档段」（填入 §12 模板）。

### 7.2 ARC-05.2 — 原子 Claim 与 Claim Session 绑定

**场景**：

| 场景 | 描述 | 实验 |
| --- | --- | --- |
| A | 双 Worker 并发 Claim 同一 Job | EXP-01 |
| B | 同实例 / 不同 Session 的身份隔离（**不**含 Reclaim 主路径） | EXP-02 |
| C | 过期租约 Reclaim、并发回收、重新 Claim（**不**重复 EXP-02 会话三元组场景） | EXP-03 |

**原子条件（须由实验最终状态验证）**：与 02 §7.4.1、06 §14.4.1 一致（见 EXP-01 条件字段清单）；新 Session ID 在 CAS **前**生成；`claimVersion` 仅在原子更新内递增；影响记录数 = 1 方视为成功。

**通过标准**：任一时刻不存在两个有效 `(leaseOwner, claimVersion)` 执行权对同一 Job 生效。

### 7.3 ARC-05.3 — Fencing 与过期 Worker 防护

**须验证**（与 EXP-02 / EXP-03 分工见 §8.2）：

- 旧 Session 在失效后不得 Renew、不得写 Task、不得 Complete / Fail / Retry（Reclaim **权限**属于回收 Worker，不属于失效旧 Session）；
- 不得通过读 Job 刷新本地 `(claimSessionId, claimVersion)`；
- 旧 Worker 暂停恢复场景下的下游写入拒绝（EXP-04、EXP-05）。

**实验**：EXP-04（可控时序 + 屏障）、EXP-05（事务阶段竞争矩阵 T1—T6）；EXP-02 仅覆盖「会话身份」侧 fencing，EXP-03 仅覆盖「租约生命周期」侧 fencing。

**凭证规则**：校验必须同时使用 **Claim Session ID + `claimVersion`**（不可拆）；`workerInstanceId` 不作为执行权依据。

### 7.5 实验时间源与租约边界（P1-01）

Spec 中租约比较以 **`serverNow`（服务端时钟为权威）** 为准（02 §7.4.1 `leaseExpiresAt` 说明；06 §14.4 过滤条件）。ARC-05 **不得**默认「云函数 `Date.now()` = 数据库服务端时间」。

| 项 | 任务书口径 | 证据责任 |
| --- | --- | --- |
| `serverNow` 定义 | 实验与 Worker 伪代码中的 `serverNow`：指 **判定租约是否过期、Claim/Reclaim 条件、机制 A 步骤 1 过滤** 所采用的**同一时间基准**；生产目标为与持久化 `leaseExpiresAt` 写入时一致的权威时钟 | EXP-01、EXP-03、EXP-05 每组须记录采用的 `serverNow` 来源 |
| 已确认能力（桌面调研） | 云函数内 `db.serverDate()` 可用于 **写入** `updatedAt` / `leaseExpiresAt`（见 06 示例与微信更新指引） | 实验日志须记录 `serverDate` 写入值与读回值 |
| **待验证**能力 | 微信云开发是否提供与 CAS/事务一致的 **可读服务端时间 API**，供应用层在 Claim 条件中比较 `nextRetryAt`、`leaseExpiresAt`，且与 `serverDate` 写入语义一致 | **若无**：记入能力缺口 + 设计 EXP-01 附加用例（双 Worker 时钟偏差注入）；**不得**因未实测而标 PASS |
| 仅云函数本地时钟时 | 若条件 CAS 只能使用云函数 `new Date()` 与库内已存 datetime 比较 | 须在同环境测量：多实例并发 Claim 时边界秒级误判率；记录 NTP/容器时钟偏差上界（**待实测**） |
| Claim 边界 | `nextRetryAt` 为空或 ≤ `serverNow`；可领取态 `leaseOwner=null` 且 `leaseExpiresAt=null`（06 §14.4.1） | EXP-01 |
| Renew 边界 | `leaseExpiresAt > serverNow` 方可续期；**禁止**对已过期租约 Renew「复活」 | EXP-03（到期前/后对照） |
| Reclaim 边界 | `status=processing` 且 `leaseExpiresAt <= serverNow` | EXP-03 |
| 事务执行中租约到期 | 机制 A 步骤 1 含 `leaseExpiresAt > serverNow`；**若**事务始于合法租约、提交前租约已过期 | EXP-05（T4/T5 子组）+ EXP-07：**不得**假设允许提交 Task；预期 Fail-Closed（步骤 1 影响 0 或事务失败） |
| 实验时间戳 | 屏障时间线、操作日志使用 **UTC ISO-8601**；同时记录各 Worker **本地** `Date.now()` 与所用 `serverNow` 来源字段 | §8.3 证据通用要求 |

**禁止**：为通过实验而临时改写业务 Job 上的租约字段作为屏障；屏障不得参与被测 Claim/Reclaim/机制 A 事务（见 EXP-04）。

### 7.4 ARC-05.4 — 执行权与下游写入原子性（P0）

**时序核心**（须 EXP-04 + EXP-05 T1—T4/T6）：

```text
Worker A1: 机制 A 事务 → 在预定阶段暂停（见 EXP-04/05 暂停点）
Worker B:   事务外 Reclaim + Claim 新 Session（独立调用）
Worker A1: 恢复至 commit；与 B 竞争同一 Job doc
```

**预期**：B 取得执行权后，A1 **不得**持久化 `schedule_tasks` 业务写入（**否则 ARC-05.4 = FAIL**）。T5 合法先后见 EXP-05。

**须回答的分析题**（写入 §12 结论时逐条引用实验证据）：

1. 同事务 get + 校验 + 写 Task 是否足够？
2. 是否必须在事务内对 Job **写**以使提交时冲突？
3. 两事务竞争同一 Job 时平台行为？
4. B 在 A1 提交前 Reclaim 的平台行为？
5. A1 已提交后 B 的 Reclaim 是否仍可能成功？
6. 租约过期后旧事务是否仍可提交？
7. 提交结果未知时如何避免重复写？

**未 PASS 前架构门禁**（06 §14.4.4）：禁止生产 Worker 自动写 `ScheduleTask`；禁止事务外二次校验降级。

---

## 8. 实验矩阵

### 8.1 总览

| 实验编号 | 名称 | 子任务 | 状态 | 结论 |
| --- | --- | --- | --- | --- |
| ARC-05-EXP-01 | 单 Job 并发双 Worker Claim | 05.2 | NOT STARTED | — |
| ARC-05-EXP-02 | Claim Session 身份隔离（同实例 A1/A2） | 05.2 / 05.3 | NOT STARTED | — |
| ARC-05-EXP-03 | Lease 生命周期、Reclaim 与重新 Claim | 05.2 / 05.3 | NOT STARTED | — |
| ARC-05-EXP-04 | 旧 Worker 恢复后 ScheduleTask 写入防护 | 05.3 / **05.4** | NOT STARTED | — |
| ARC-05-EXP-05 | 执行权验证与下游写入事务竞争 | **05.4** | NOT STARTED | — |
| ARC-05-EXP-06 | 事务提交冲突与自动重试 | 05.1 | NOT STARTED | — |
| ARC-05-EXP-07 | 事务超时、网络中断、提交结果未知 | 05.1 | NOT STARTED | — |
| ARC-05-EXP-08 | 重复投递、重复执行与幂等 | 05.2 / 05.3 | NOT STARTED | — |
| ARC-05-EXP-09 | 多 Job 并发相互影响 | 05.1 | NOT STARTED | — |

### 8.2 实验详细设计

#### ARC-05-EXP-01 — 单 Job 并发双 Worker Claim（P0-03）

| 项 | 内容 |
| --- | --- |
| 验证目标 | 并发 Claim 仅一个成功；失败者 `stats.updated=0` 且 **不得**继续 ScheduleTask 写入；SDK 对条件更新、`_.inc(claimVersion)`、影响记录数的支持情况可观测 |
| 前置条件 | 一条 `pending` Job：`claimVersion=0`，`nextRetryAt` 为空或 ≤ 实验用 `serverNow`（§7.5）；`leaseOwner=null`，`leaseExpiresAt=null` |
| 测试数据 | 预置 `arc05_background_jobs` 文档 1 条（隔离集合） |
| 并发参与者 | **两个独立云函数调用**（不同 `workerInstanceId`；各自在发起 CAS **前**生成不同 `claimSessionId`） |
| CAS 条件字段（与 06 §14.4.1、02 §7.4.1 对齐，**不得**省略） | `_id=jobId`；`status=pending`；`nextRetryAt` 不存在或 ≤ `serverNow`；`leaseOwner=null`；`leaseExpiresAt=null` |
| 原子更新字段（同一 `update`） | `status=processing`；`leaseOwner=<本次预生成 Claim Session ID>`；`workerInstanceId=<本实例 ID>`（仅诊断）；`leaseExpiresAt=serverNow+leaseDuration`；`claimVersion` **数据库原子** +1（如 `_.inc(1)`，**禁止**先 `get` 后在应用层 `+1` 并无条件写回）；`updatedAt=serverNow`（或 `db.serverDate()`，须在证据中注明与 §7.5 一致性） |
| 执行步骤 | ① 外部**启动屏障**仅同步两路 Claim 发起时刻（屏障 **不**写 Job，见 EXP-04 协调通道原则）② 双方同时执行上述 CAS ③ 采集 `stats.updated` ④ 失败方 **必须**尝试一次下游写（机制 A 或占位调用）以验证其 **未**持有执行权 ⑤ 读 Job 终态 |
| 预期结果 | 恰好一方 `updated=1`；胜方 `leaseOwner`/`claimVersion`/`workerInstanceId` 与其预生成 Session 一致；败方 `updated=0` 且下游写 **不得**成功 |
| 失败表现 | 双方 `updated=1`；`claimVersion` 递增 ≠1；字段组合不一致（如 `processing` 但 `leaseOwner=null`） |
| SDK 能力缺口 | 不支持 `_.inc`、不支持多条件 `where`、不返回 `stats.updated`、或无法在同一 API 完成条件+递增 → 记录为 **能力缺口**，**禁止**用应用层模拟原子 Claim 标 PASS |
| 证据采集 | Claim 前后 Job JSON、双方 Session 预生成值、`stats`、失败方下游写结果、§7.5 时间源说明 |
| 通过标准 | 02 §7.4.1 规则 3 + 06 §14.4.1 全文；终态唯一有效 Session；`claimVersion` 仅递增 1 次 |
| API 待核实 | 实验编码前须对照当前锁定 `wx-server-sdk` 文档核对 `where().update()`、`command.inc` 形态；任务书阶段 **不**编造可运行代码 |

#### ARC-05-EXP-02 — Claim Session 身份隔离（P1-02，**非** Reclaim 主路径）

| 项 | 内容 |
| --- | --- |
| 验证边界 | **仅**验证 Claim Session 身份与本地三元组不变性；**不**在此实验主路径验证「Lease 到期前能否 Reclaim」（属 EXP-03） |
| 验证目标 | 同一 `workerInstanceId` 可先后产生 **不同** Claim Session；A1 与 A2 的 Session ID **必须**不同；A1 恢复后 **禁止**用 Job 上最新 `leaseOwner`/`claimVersion` 覆盖本地 `(jobId, sessionA1, v1)`；A1 用旧三元组不得 Renew / 机制 A 写 Task / Complete / Fail |
| 前置条件 | A1 按 EXP-01 规则成功 Claim，`claimVersion=1`，`leaseOwner=sessionA1` |
| 测试数据 | Job + A1 进程持久化三元组（**故意不**在恢复时重读 Job 刷新 token） |
| 并发参与者 | 实例 A：协程/调用 A1（挂起）；实例 A 或 B：`sessionA2` 新 Claim（在 A1 租约已合法失效后，失效方式 **二选一**：等待 `leaseExpiresAt` 自然到期后由 **EXP-03 的 Reclaim** 置 `pending`，再 Claim；或实验编排仅调用 §14.4.1 Claim 于已 `pending` 的 Job） |
| 执行步骤 | ① A1 Claim 并持有三元组 ② A1 **暂停**（不调 Renew）③ 使租约失效并完成 **一次** Reclaim+新 Claim（步骤细节引用 EXP-03，本实验只消费其终态：`sessionA2`/`v2` 在 Job 上）④ A1 恢复 ⑤ A1 **不得**执行「按 `_id` 读 Job 并采用新版本」⑥ A1 用 `(sessionA1,v1)` 尝试 Renew、机制 A、Complete ⑦ A2 用合法三元组对照执行一条 Renew 或只读外的合法操作 |
| 预期结果 | Job 上为 `sessionA2`/`v2`；A1 全部 mutating 操作失败（`updated=0` 或事务 rollback）；A2 在合法三元组下 Renew 成功（若租约仍有效） |
| 失败表现 | A1 因 `workerInstanceId` 相同而成功写或 Renew；A1 读 Job 后改用 `v2` 继续写 |
| 证据采集 | A1 恢复前后 **本地**三元组日志、是否发生 Job 全字段重读、各操作 `stats`/事务结果 |
| 通过标准 | 06 §14.4.1 步骤 7、同实例双会话行；02 §7.4.1 规则 3—4 |
| 与 EXP-03 分界 | EXP-03 负责 Reclaim CAS 正确性、到期前后 Reclaim、并发回收；本实验 **不**重复验收 Reclaim 字段清理 |

#### ARC-05-EXP-03 — Lease 生命周期与 Reclaim（P1-02，**非**会话读覆盖场景）

| 项 | 内容 |
| --- | --- |
| 验证边界 | **仅**验证租约时效、Reclaim、重新 Claim 与并发回收；**不**重复 EXP-02 的「A1 禁止读 Job 刷新 token」主验收（EXP-02 已覆盖） |
| 验证目标 | 租约 **未**到期时 Reclaim **必须**失败；到期后 Reclaim 成功并清理 `leaseOwner`/`workerInstanceId`/`leaseExpiresAt`，**不**改 `claimVersion`；新 Claim 产生新 Session 且 `claimVersion+1`；多 Reclaim Worker 仅一个成功；Reclaim 与后续 Claim 之间 Job 不得长时间处于非法组合态 |
| 前置条件 | 子用例 3a：A 持有 `processing`，`leaseExpiresAt > serverNow`；3b：同 Job，`leaseExpiresAt <= serverNow` |
| Reclaim CAS（06 §14.4.3） | 条件：`status=processing`，`leaseExpiresAt <= serverNow`；更新：`pending`（可含 `nextRetryAt`），租约字段清空，**不**修改 `claimVersion` |
| 执行步骤 | ① **3a** 到期前发起 Reclaim → 预期 `updated=0` ② **3b** 到期后 Reclaim → `updated=1`，核验字段清理与 `claimVersion` 不变 ③ B 按 §14.4.1 Claim → `claimVersion` 递增、新 `leaseOwner` ④ 可选：两回收 Worker 并发 Reclaim（仅一处成功）⑤ 旧 Session A 对 Reclaim/Claim 路径做一次 mutating 尝试（预期失败，**轻量**，不断言读覆盖——详见 EXP-02） |
| 预期结果 | 3a 失败；3b 成功后 B 唯一有效租约；`claimVersion` 序列：回收前后不变，Claim 后 +1 |
| 失败表现 | 到期前回收成功；回收改动 `claimVersion`；双回收均成功；Claim 后仍存在双执行权 |
| 证据采集 | 各步 `serverNow` 与 `leaseExpiresAt` 对照、Job 快照、`stats.updated` |
| 通过标准 | 06 §14.4.3 + 02 §7.4.1 规则 5—6、规则 8 |

#### ARC-05-EXP-04 — 旧 Worker 恢复后 ScheduleTask 写入防护（P0-01，可控时序）

| 项 | 内容 |
| --- | --- |
| 验证目标 | P0：在 B 已完成 Reclaim + 新 Claim 后，A1 恢复并提交机制 A **不得**成功写入 `schedule_tasks`；屏障本身不得污染被测事务能力结论 |
| 前置条件 | A1 已 Claim 且本地三元组合法；`arc05_schedule_tasks` 测试 doc 存在；B 为独立 Worker 调用 |
| 并发参与者 | **真实并发**：云函数调用 #1（A1，`runTransaction` 或 `startTransaction`）+ 云函数调用 #2（B，事务外 CAS）+ **测试控制器**（第三类进程，见下） |

**屏障与协调（P0-01，必守）**

| 原则 | 要求 |
| --- | --- |
| 不参与被测事务 | 屏障 **仅**协调测试 Worker 时序；**禁止**在屏障通道或协调集合中执行 Claim、Reclaim、机制 A 步骤 1/2、或修改被测 Job 的业务字段 |
| 独立协调通道（优先序） | ① 专用 **HTTP 测试控制器**（内存/文件状态，不连被测库写 Job）② 独立 **Redis / 消息**（key 与 `arc05_*` Job 无关）③ **禁止**使用对被测 `arc05_background_jobs` 的「标志位 update」作为屏障（会干扰 CAS/事务锁） |
| A1 暂停点 | 机制 A 事务回调内：**已完成** `doc(jobId).get` 且应用层校验旧 Session 通过；**尚未** touch Job（步骤 1 写）；**尚未**写 Task |
| A1 → 控制器 | 暂停点后调用协调 API：`POST /barrier/reached {experimentId, role:A1, phase:TX_AFTER_READ}`；阻塞直至 `phase:RESUME_A1` 或超时 |
| B 路径 | 控制器在收到 `TX_AFTER_READ` 后 **才**触发 B：Reclaim（若需）+ §14.4.1 Claim；完成后 `POST /barrier/b_done {newSession,newVersion}` → 控制器下发 `RESUME_A1` |
| 时间线证据 | 须记录 ISO 时间：`t_a1_pause`、`t_b_reclaim_done`、`t_b_claim_done`、`t_a1_resume`、`t_tx_settle` |
| 超时与退出 | 屏障等待上限建议 `barrierTimeoutMs`（如 120000，< 云函数硬超时）；超时 → 本 **run 标为 INVALID**（非事务 PASS）；强制结束挂起事务/调用 |
| 真并发 | A1 与 B **必须**为并行调用；**禁止**单进程 `await B(); await A1继续()` 顺序模拟 |
| 屏障失败 | 协调 API 不可用、死锁、超时 → 实验 **INVALID 或 FAIL**；**不得**解释为「微信事务能力通过」 |

| 执行步骤 | ① 控制器 reset ② 并行启动 A1 事务与 B「待命」③ A1 到达暂停点并通知控制器 ④ B 执行 Reclaim+Claim 并通知 ⑤ 控制器恢复 A1 ⑥ A1 执行 touch Job + Task update 并 commit ⑦ 只读查询 Job + Task 终态 |
| 预期结果 | A1 事务 rollback/reject 或提交后 Task **无** A1 侧业务写入；Job 属 B 的新 Session |
| 失败表现 | B 已占有 Job 后 Task 仍出现 A1 写入（→ **ARC-05.4 必须 FAIL**） |
| 证据采集 | 屏障事件日志、事务结果（含 `runTransaction` 重试次数）、Job/Task 终态、§7.5 时间戳 |
| 通过标准 | **数据库终态**满足 06 §14.4.4；屏障 INVALID 的 run **不计入** PASS 样本 |

#### ARC-05-EXP-05 — 执行权与下游写入事务竞争矩阵（P0-02）

| 项 | 内容 |
| --- | --- |
| 验证目标 | 区分旧 Worker 在事务内 **四阶段**（仅读 Job → 条件语义校验 → touch Job 写 → 写 Task / 提交）与 B 的 Reclaim+Claim 竞态；判定机制 A 是否满足 06 §14.4.4 |
| 前置条件 | 每组独立 Job+Task；屏障协议同 EXP-04（可复用控制器） |
| SDK 重试（须与 EXP-06 交叉） | `runTransaction` 冲突时平台可能 **重试回调**；须记录回调执行次数。重入后 **是否**重新 `get` Job、是否仍用 **本地** `(sessionA1,v1)` 校验 — **须实测**，**不得**将「回调曾成功返回」等同于「事务已 commit」 |
| 禁止项 | 事务外二次校验作为 fencing；事务回调成功 ≠ 提交成功；心跳推断旧 Worker 停止 |

**竞争矩阵 T1—T6（每组须独立跑、独立证据 ID）**

| 场景 | A1 事务内阶段（暂停点） | B 执行 | 重点验证 | 预期事务行为（旧 Session A1） | Job 终态 | ScheduleTask 终态 | PASS 证据 | FAIL / PARTIAL |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| **T1** | **尚未** `get` Job | Reclaim+Claim 完成 | A1 起始前执行权已属 B | 回调内校验失败或事务 abort；**不得**写 Task | `processing`，`leaseOwner=B`，`claimVersion` 已递增 | 无 A1 写入 | Task 无 A1 字段；事务拒绝日志 | A1 仍写 Task → **ARC-05.4 FAIL** |
| **T2** | 已 `get`+校验，**未** touch | B 完成 | 旧快照是否仍提交 | 提交时应与 B 争用锁/版本；步骤 1 失败或整体 rollback | B 占有 | 无 A1 业务写 | 同左 | Job 属 B 且 Task 被 A1 改 → **FAIL** |
| **T3** | 已 touch Job，**未**写 Task | B 完成 | 冲突检测 | 整笔 rollback 或 commit 不含 Task 写 | B 占有 | 无 A1 写 | touch 未导致 A1 Task 落库 | 任 A1 Task 落库 → **FAIL** |
| **T4** | 已写 Task，**未** commit | B 完成 | 阻止旧 Task 落库 | **必须**整笔失败或 Task 变更不可见 | B 占有 | B 路径对照可写；无 A1 写 | 终态只读 | A1 Task 可见 → **FAIL** |
| **T5** | A1 **完整 commit** 成功 | **随后** B Reclaim+Claim | 合法先后：A1 先完成 | A1 commit 在 B 前时 Task **可**含 A1（合法）；B 之后不得覆盖为「越权」误判 | 视 B 是否回收：可能 `pending`→B 占有 | A1 写入若先于 B 领取则保留 | 时间线证明 `t_a1_commit < t_b_claim` | 将 T5 时间序颠倒解读为 FAIL |
| **T6** | A1 提交结果 **unknown**（断点见 EXP-07） | B 完成 | 能否识别已提交 Task | **不得**盲目重试写；只读 Job+Task 后 Fail-Closed | B 占有 | 以持久化为准 | 恢复决策表（EXP-07）一致 | 未知时重复写导致双写 → **FAIL** |

**每组通用输入记录**

- 初始 Job：`status`、`claimVersion`、租约字段
- A1 本地 `(jobId, claimSessionId, claimVersion)`
- B 的 Reclaim 条件是否满足、新 Session、`claimVersion` 递增后值
- 暂停点 phase 名（与 EXP-04 枚举对齐）
- 事务 settle 结果：**committed / rolled_back / unknown**（三者分列，禁止混称）

**执行步骤（每组）**

1. 按场景设置 A1 暂停点
2. B 完成 Reclaim+Claim
3. 恢复 A1 完成剩余阶段至 commit/unknown
4. 只读 Job + Task；对照上表判定
5. 记录 `runTransaction` 回调次数与最终错误码

**ARC-05.4 硬性 FAIL 门槛（P0-02-5）**：若 **B 已成功取得新执行权** 后，A1 仍能使 **ScheduleTask 业务写入持久化成功**，则 ARC-05.4 = **FAIL**，与 T 编号无关。

**对照组**：新 Session B 在合法三元组下机制 A 成功写 Task（证明非环境性假阴性）。

**通过标准**：T1—T4、T6 均满足「B 占有后无 A1 Task 写」；T5 时间序符合语义；未知提交走 Fail-Closed。

#### ARC-05-EXP-06 — 事务提交冲突与自动重试

| 项 | 内容 |
| --- | --- |
| 验证目标 | `runTransaction` 冲突重试次数、回调重入语义 |
| 执行步骤 | 两事务争用同一 `doc`；记录回调执行次数与最终 `times` 参数 |
| 预期结果 | 与官方「默认 3 次」一致或可观测偏差（偏差须记录） |
| 通过标准 | 行为可预测且可日志化 |

#### ARC-05-EXP-07 — 事务失败、超时与提交结果未知（P1-03）

| 项 | 内容 |
| --- | --- |
| 验证目标 | 调用方能否判定 commit 成败；为 06 §14.4.4 Fail-Closed「能力支持但本次事务失败」与 §14.4.7「事务提交结果未知」提供 **可执行** 恢复输入；与 EXP-05 **T6** 衔接 |

**须覆盖场景（独立 run，标注断点）**

| # | 场景 | 断点/手法 | 观测 |
| --- | --- | --- | --- |
| 7.1 | 事务开始前网络/API 不可用 | 未进入 `runTransaction` | 无 Job/Task 变更；Worker 不得假设已写 |
| 7.2 | 事务执行中网络中断 | `get` 后/touch 前/touch 后注入失败 | 事务状态 + 终态只读 |
| 7.3 | commit **前**客户端异常退出 | `startTransaction` + 未 `commit` 进程结束 | Job/Task 是否变更 |
| 7.4 | commit 已发出、响应前超时 | 代理延迟/丢弃响应 | **unknown** |
| 7.5 | 服务端已 commit、客户端未收到成功 | 对照 7.4 只读 | 可能 known-commit |
| 7.6 | 服务端失败、客户端无法确认 | 同上 | unknown vs fail |
| 7.7 | SDK 自动重试导致回调重复 | 与 EXP-06 同 run 记录 | 回调次数、是否双写 Task |
| 7.8 | Worker 恢复后只读 Job + ScheduleTask | 应用恢复决策表 | 是否 Fail-Closed |

**恢复决策表（实验结论须回填此表可操作性，不得留空臆断）**

| 观测结果（持久化 + 执行权） | 是否允许重试事务 | 是否允许继续 ScheduleTask 写入 | 恢复动作 |
| --- | --- | --- | --- |
| Task **无** 本次业务写；Job 仍属本 Session 且租约有效 | 是（新事务） | 是（须机制 A） | 在确认三元组后重试 |
| Task **已有** 本次幂等键对应写；Job 仍 processing | 否（避免双写） | 否 | 按幂等视为完成或进入完成 Job 路径 |
| Task 不确定；Job 仍属本 Session | **否** | **否** | **Fail-Closed**：只读、告警、人工或编排查询 |
| Task 不确定；Job 已属 **新** Session | **否** | **否** | 旧 Worker **退出**；禁止用新 token 写 |
| 事务 **unknown**；未确认 Task | **否** | **否** | **Fail-Closed**；禁止假设回滚 |
| 事务 **unknown**；Task 已存在业务写 | 否 | 否 | 以 Task 为准收敛；禁止盲目重试写 |
| 平台返回明确 rollback | 是（若仍持执行权） | 视执行权 | 记录错误码；重试前再 CAS/只读 |
| ARC-05 能力未 PASS | 否 | **否** | 06「能力尚未验证」 |

**必守原则（P1-03）**

- 结果 unknown 时 **不得**假设事务失败；**不得**直接重复非幂等业务写入
- **必须**先以 Job + Task **持久化终态**确认；无法确认 → Fail-Closed
- **不得**仅凭客户端异常、超时、本地日志断定已 rollback

| 执行步骤 | 按上表 # 逐场景执行；必要时 `startTransaction` 控制 commit 时点；每场景后只读双集合 |
| 预期结果 | 明确「平台可判定 / 不可判定」集合；决策表每行有实测依据或标 **官方未说明** |
| 证据采集 | 事务 API 返回值、云函数日志、**数据库终态**、是否发生 EXP-05 T6 型双写 |
| 通过标准 | 决策表可指导 Worker 实现且符合 06 Fail-Closed；**不**代表平台能力 PASS |

#### ARC-05-EXP-08 — 重复投递、重复执行、幂等

| 项 | 内容 |
| --- | --- |
| 验证目标 | 同 Job 重复执行不产生矛盾 Task 状态（在合法 fencing 下） |
| 执行步骤 | 同一有效 Session 重复提交相同 Task 条件更新；跨 Session 重复 |
| 通过标准 | 合法 Session 幂等；非法 Session 拒绝 |

#### ARC-05-EXP-09 — 多 Job 并发相互影响

| 项 | 内容 |
| --- | --- |
| 验证目标 | 多 `doc` 同事务 / 多事务并行是否引发非预期全局锁或 TransactionBusy |
| 执行步骤 | N 路 Job 并行 Claim + 写；监控错误率 |
| 通过标准 | 记录在案；若错误率影响生产 SLA，列入风险 |

### 8.3 证据采集通用要求

除接口返回值外，**必须**采集：

- Claim 前后 Job 全字段（至少：`status`、`leaseOwner`、`claimVersion`、`workerInstanceId`、`leaseExpiresAt`、`nextRetryAt`）
- 每次事务 **committed / rolled_back / unknown** 分列记录；`runTransaction` **回调执行次数**（非单次回调返回即 commit）
- `schedule_tasks` 终态
- 结构化操作日志（操作者、Session、version、**serverNow 来源**、UTC ISO 时间、屏障 phase）
- EXP-04/05：**屏障事件时间线**（与 §8.2 EXP-04 枚举一致）
- **以数据库终态**判定是否违反安全约束；屏障 INVALID 的 run 单独标注，不计入 PASS 统计

---

## 9. 验收标准

### 9.1 子任务

| 子任务 | 通过标准 | 当前 |
| --- | --- | --- |
| ARC-05.1 | 事务范围、隔离、冲突语义有官方 + 实验闭环 | **NOT STARTED** |
| ARC-05.2 | 并发 Claim 不产生双执行权 | **NOT STARTED** |
| ARC-05.3 | 旧 Session 不能续租/执行/改数据 | **NOT STARTED** |
| ARC-05.4 | 执行权验证与 Task 写入可靠原子性 | **NOT STARTED** |

### 9.2 整体状态规则

仅允许：**PASS** / **PARTIAL** / **FAIL**（整体验证完成后填写 §12）。

- 不得将「尚未测试」标 PASS。
- 不得将「官方未说明」标 PASS。
- 不得单次实验成功即标并发安全 PASS。
- **ARC-05.4 未 PASS 前**：保持 06 Fail-Closed「能力尚未验证」行为。

### 9.3 P0 安全门槛

ARC-05.4 为门禁项；未通过则：

- 禁止生产 Worker 自动写 `ScheduleTask`。
- 禁止事务外二次校验降级。
- 禁止仅以心跳推断旧 Worker 停止。

---

## 10. 风险与替代方案（本轮不实施）

若 ARC-05.2 或 ARC-05.4 **FAIL**，供架构决策参考（不得降低 Spec 安全要求）：

| 方向 | 一致性 | 并发安全 | 故障恢复 | 重复执行 | 运维复杂度 | 成本 | 改动范围 |
| --- | --- | --- | --- | --- | --- | --- | --- |
| 1. 单 Worker / 分区队列，避免同 Job 并行 | 中 | 依赖单点/分区 | 中 | 仍须幂等 | 低 | 低 | 06 §14 调度 |
| 2. 外部协调（Redis/etcd 租约） | 高 | 可做强 fencing | 中 | 中 | 高 | 中 | 新增基础设施 |
| 3. 换用支持串行化/条件事务的 DB | 高 | 高 | 中 | 中 | 高 | 高 | ADR 级 |
| 4. Task 写入独立幂等管道 + 异步对账 | 最终一致 | 须严格键 | 高 | 高 | 高 | 中 | 04/06 流程 |
| 5. 机制 B：Task 文档镜像 fencing 字段 | 取决于实测 | 须单 doc 原子 | 中 | 中 | 中 | 低 | 02 字段扩展（**须评审，非本轮**） |

---

## 11. 验证证据记录模板

```markdown
### 证据条目 ID: ARC-05-EV-____

| 字段 | 值 |
| --- | --- |
| 关联实验 | ARC-05-EXP-__ |
| 执行时间 | |
| 执行人 | |
| 环境 | §4 快照 |
| SDK 版本 | |
| 操作序列 | |
| Job 终态 | （JSON 或哈希） |
| Task 终态 | |
| 事务结果 | success / rollback / unknown |
| 是否违反 Spec | 是/否 + 引用条款 |
| 附件 | 日志路径 / 导出文件 |
```

---

## 12. 最终结论模板（验证完成后填写）

> **当前为空 — 整体 NOT STARTED**

| 子任务 | 结论 | 证据索引 | 备注 |
| --- | --- | --- | --- |
| ARC-05.1 | | | |
| ARC-05.2 | | | |
| ARC-05.3 | | | |
| ARC-05.4 | | | |

**ARC-05 整体**：`________`（PASS / PARTIAL / FAIL）

**对 06 §14.4.4 机制 A/B 的裁决**：`________`

**F-04.3.3-05 / F-04.3.2-02 是否可升格**：仅当 ARC-05.4 + 实现验证完成；**任务书阶段不得填写 PASS**。

---

## 13. 对 07-spec-consistency-review.md 的回填要求

验证执行方在 **ARC-05 实验完成** 后更新 07（**非本轮**），须遵守：

| 07 位置 | 回填内容 | 禁止 |
| --- | --- | --- |
| §15.8 F-04.3.3-05「实现验证」列 | 链接本文 §12 + 证据 ID | 在无 EXP-04/05 前标 PASS |
| §15.7 F-04.3.2-02 行 | ARC-05 整体结论摘要 | 提前改 PARTIAL→PASS |
| F-04.3.3 总判定 | 仍 **PARTIAL PASS（Spec 层）** 直至实现+平台双 PASS | 标成全绿 PASS |
| F-04 整体 | 保持 **PARTIAL** 直至 restrictionType 决策 + ARC-05 + 实现 | RESOLVED |

回填示例（验证后）：

```text
实现验证：ARC-05 §12 — ARC-05.4 PASS（2026-__-__），证据 ARC-05-EV-012..018
```

---

## 14. 未解决问题跟踪（任务书阶段）

| 优先级 | 问题 | 原因 | 下一步证据 |
| --- | --- | --- | --- |
| **P0** | 快照隔离 + 无 `where` 事务 CAS 能否满足 §14.4.4 | 官方明确非串行化且不能避免 write skew | EXP-04、EXP-05 T1—T4 |
| **P0** | 旧 Worker 事务在 Reclaim 后是否仍可提交 Task | 官方未等价保证；**若发生 → ARC-05.4 FAIL** | EXP-04、EXP-05 |
| **P1** | 可读 `serverNow` 与 `serverDate` 一致性 | 02 要求服务端时钟权威；微信能力 **待实测** | §7.5、EXP-01/03 |
| **P1** | 事务超时与 commit 未知 | 官方未说明 | EXP-07、EXP-05 T6 |
| **P1** | Claim CAS 与事务内校验的组合是否构成隐藏 check-then-write | 架构歧义；机制 A 步骤须在事务内 touch | EXP-05 评审记录 |
| **P1** | 实验屏障可靠性与真并发 | 屏障失败不得误判平台 PASS | EXP-04 协调规范 |
| **P2** | TransactionBusy 类错误频率（社区个案） | 非官方 SLA | EXP-09 |
| **P2** | 工程 ARC-02 环境/SDK 版本未锁定 | §4 待填 | 工程初始化 |

---

## 15. 文档修订记录

| 版本 | 日期 | 说明 |
| --- | --- | --- |
| 0.1 | 2026-11-09 | 初版任务书；状态 NOT STARTED；含官方桌面调研与实验设计 |
| 0.2 | 2026-11-09 | **ARC-05.1 定点修订**：强化 EXP-04 屏障独立性/真并发；EXP-05 T1—T6 矩阵；EXP-01 与 02/06 Claim 对齐；§7.5 时间源；拆分 EXP-02/03 边界；细化 EXP-07 恢复决策表。**未执行**平台实验；子任务结论仍为 NOT STARTED |

---

## 附录 A — 执行前上下文检查

| 检查项 | 结果 | 说明 |
| --- | --- | --- |
| 当前分支和最新提交 | main，与 `origin/main` 同步 | 2026-11-09 修订前 `git status` clean |
| 02 数据模型阅读 | 完成 | §7.4.1 Claim Session、`claimVersion`、租约 CAS |
| 06 技术架构阅读 | 完成 | §14.4.1—§14.4.7、机制 A/B、Fail-Closed、`serverNow` |
| 07 审查报告阅读 | 完成 | `specs/07-spec-consistency-review.md` §15.7—§15.8 调度/执行权/ARC-05 待验证 |
| `specs/00-project-charter.md` | **不存在** | 已记录，未编造 |
| `docs/agent/STATUS.md` | **不存在** | 已记录，未编造 |
| `package.json` / npm scripts | **不存在** | 仓库根目录无 `package.json`；不假设测试命令 |
| ARC-05 版本/状态 | 0.2 / 整体 **NOT STARTED** | EXP-01—EXP-09 结论栏仍为 NOT STARTED |
| 本轮授权修改 | `specs/phases/ARC-05-platform-capability-validation.md` | ARC-05.1 实验设计修订 |
| 与 02/06 冲突 | 未发现新增冲突 | 修订以 02 §7.4.1、06 §14.4 为准；Barrier 禁止写 Job |
