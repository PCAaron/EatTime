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
| B | 同实例 A1 → 过期 → A2 新 Session | EXP-02 |
| C | 过期租约 Reclaim + B 新 Session / 更高 version | EXP-03 |

**原子条件（须由实验最终状态验证）**：状态允许领取、租约空或过期、新 Session ID、version 绑定、字段同事务/同 CAS 更新。

**通过标准**：任一时刻不存在两个有效 `(leaseOwner, claimVersion)` 执行权对同一 Job 生效。

### 7.3 ARC-05.3 — Fencing 与过期 Worker 防护

**须验证**：旧 Session 不得 Renew / Reclaim / 写 Task / Complete / Fail / Retry；不得通过读 Job 刷新 token；同 `workerInstanceId` 下 A1≠A2。

**实验**：EXP-02、EXP-03、EXP-04（操作矩阵扩展至 §7.3 列表）。

**凭证规则**：校验必须同时使用 **Claim Session ID + `claimVersion`**（不可拆）。

### 7.4 ARC-05.4 — 执行权与下游写入原子性（P0）

**时序核心**（须 EXP-04 + EXP-05）：

```text
Worker A1: 事务内 get Job 并校验旧 Session → 人为屏障暂停
Worker B:   Reclaim + Claim 新 Session（事务外 CAS）
Worker A1: 恢复并尝试提交 Task 写
```

**预期**：A1 不得成功写入 `schedule_tasks`；B 的执行权不被 A1 覆盖。

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
| ARC-05-EXP-02 | 同实例新旧 Claim Session 隔离 | 05.2 / 05.3 | NOT STARTED | — |
| ARC-05-EXP-03 | 过期租约 Reclaim 与旧 Session 失效 | 05.2 / 05.3 | NOT STARTED | — |
| ARC-05-EXP-04 | 旧 Worker 恢复后 ScheduleTask 写入防护 | 05.3 / **05.4** | NOT STARTED | — |
| ARC-05-EXP-05 | 执行权验证与下游写入事务竞争 | **05.4** | NOT STARTED | — |
| ARC-05-EXP-06 | 事务提交冲突与自动重试 | 05.1 | NOT STARTED | — |
| ARC-05-EXP-07 | 事务超时、网络中断、提交结果未知 | 05.1 | NOT STARTED | — |
| ARC-05-EXP-08 | 重复投递、重复执行与幂等 | 05.2 / 05.3 | NOT STARTED | — |
| ARC-05-EXP-09 | 多 Job 并发相互影响 | 05.1 | NOT STARTED | — |

### 8.2 实验详细设计

#### ARC-05-EXP-01 — 单 Job 并发双 Worker Claim

| 项 | 内容 |
| --- | --- |
| 验证目标 | 并发 Claim 仅一个成功；失败者 `stats.updated=0` |
| 前置条件 | 一条 `pending` Job；`leaseOwner=null` |
| 测试数据 | 预置 `arc05_background_jobs` 文档 1 条 |
| 并发参与者 | 云函数 `claimWorker` ×2（不同 `workerInstanceId`，不同预生成 `claimSessionId`） |
| 执行步骤 | ① 屏障同步两路 `Promise` 同时 `where({_id,status:pending,leaseOwner:null}).update(...)` ② 采集双方 `stats.updated` 与 Job 终态 |
| 预期结果 | 恰好一方 `updated=1`；Job 上 `leaseOwner`/`claimVersion` 与胜利方 Session 一致 |
| 失败表现 | 双方 `updated=1` 或 Job 处于不一致字段组合 |
| 证据采集 | Claim 前后 Job JSON、双方日志、时间戳、`stats` |
| 通过标准 | 终态唯一有效 Session；`claimVersion` 仅递增 1 次 |

#### ARC-05-EXP-02 — 同实例新旧 Claim Session 隔离

| 项 | 内容 |
| --- | --- |
| 验证目标 | 同 `workerInstanceId` 下 A1 与 A2 Session 不可互认 |
| 前置条件 | A1 已成功 Claim `version=1` |
| 测试数据 | Job + 本地模拟 A1 持 `(sessionA1,1)` |
| 并发参与者 | 单云函数实例逻辑分岔模拟 A1/A2 |
| 执行步骤 | ① A1 Claim ② 人工将 `leaseExpiresAt` 设为过去（或等待）③ 回收 ④ **同 workerInstanceId** 生成 `sessionA2` 再 Claim ⑤ A1 用旧三元组尝试 Renew/写/Complete |
| 预期结果 | A2 `version=2`；A1 全部写操作 `updated=0` 或事务失败 |
| 失败表现 | A1 凭借相同 `workerInstanceId` 成功写 |
| 证据采集 | 每步 Job 快照、A1/A2 操作日志 |
| 通过标准 | 02 §7.4.1 规则 3 + 06 §14.4.1 步骤 7 |

#### ARC-05-EXP-03 — 过期租约 Reclaim 与旧 Session 失效

| 项 | 内容 |
| --- | --- |
| 验证目标 | Reclaim 后 B 新 Session + 更高 version；A 旧 Session 失效 |
| 前置条件 | A 持有 `processing` 且租约过期 |
| 执行步骤 | ① Reclaim CAS ② B Claim ③ A 用旧 Session 续租/写/完成 |
| 预期结果 | A 全部失败；B 持有唯一有效租约 |
| 失败表现 | `claimVersion` 未递增或 A 仍成功 |
| 证据采集 | Reclaim 前后 Job、`claimVersion` 序列 |
| 通过标准 | 06 §14.4.3 + 02 §7.4.1 规则 5—6 |

#### ARC-05-EXP-04 — 旧 Worker 恢复后 ScheduleTask 写入防护（可控时序）

| 项 | 内容 |
| --- | --- |
| 验证目标 | P0：A1 暂停恢复后不得写入 Task |
| 前置条件 | A1 已 Claim；`schedule_tasks` 测试 doc 存在 |
| 并发参与者 | A1 事务协程 + B Reclaim 协程 |
| 执行步骤 | ① A1 进入 `runTransaction`，`get` Job 校验通过后 **await 屏障**（Redis/DB 标志位/共享文件 — 验证环境自选）② B Reclaim + Claim ③ 释放屏障 ④ A1 继续写 Job touch + Task update ⑤ 查询 Task 终态 |
| 预期结果 | 事务 reject/rollback；Task 无 A1 写入或保持 B 写入前状态 |
| 失败表现 | Task 含 A1 会话对应写入且 B 已占有 Job |
| 证据采集 | 事务 Promise 结果、Task 终态、操作时间线 |
| 通过标准 | **数据库终态**不违反 06 §14.4.4 |

**实现说明**：屏障须真并发，禁止顺序 `await` 模拟。

#### ARC-05-EXP-05 — Job 执行权验证与下游写入事务竞争（P0）

| 项 | 内容 |
| --- | --- |
| 验证目标 | 机制 A 是否在官方事务下满足原子边界 |
| 前置条件 | 同 EXP-04；额外记录 B 成功写入路径对照组 |
| 执行步骤 | 矩阵：**{事务内是否 touch Job} × {B Reclaim 时点}**；至少 4 组 |
| 预期结果 | 旧 Session 路径 Task 不写；新 Session 路径在合法三元组下可写 |
| 失败表现 | Job 属 B 而 Task 被 A1 改 |
| 证据采集 | 每组 Job + Task + 事务错误码 |
| 通过标准 | ARC-05.4 表「通过标准」 |

#### ARC-05-EXP-06 — 事务提交冲突与自动重试

| 项 | 内容 |
| --- | --- |
| 验证目标 | `runTransaction` 冲突重试次数、回调重入语义 |
| 执行步骤 | 两事务争用同一 `doc`；记录回调执行次数与最终 `times` 参数 |
| 预期结果 | 与官方「默认 3 次」一致或可观测偏差（偏差须记录） |
| 通过标准 | 行为可预测且可日志化 |

#### ARC-05-EXP-07 — 事务超时、网络中断、提交结果未知

| 项 | 内容 |
| --- | --- |
| 验证目标 | 调用方能否判定 commit 成败；Fail-Closed 恢复策略输入 |
| 执行步骤 | ① 人为延长屏障 > `cloud.init.timeout` ② 模拟中断（kill / 抛错点对照）③ 只读重查 Job + Task |
| 预期结果 | 文档化「已知/未知」集合 |
| 通过标准 | 对 06 Fail-Closed「能力支持但本次事务失败」行给出可操作恢复表 |

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

- Claim 前后 Job 全字段（至少：`status`、`leaseOwner`、`claimVersion`、`workerInstanceId`、`leaseExpiresAt`）
- 每次事务 commit/rollback 结果与重试次数
- `schedule_tasks` 终态
- 结构化操作日志（操作者、Session、version、ISO 时间）
- **以数据库终态**判定是否违反安全约束

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
| **P0** | 快照隔离 + 无 `where` 事务 CAS 能否满足 §14.4.4 | 官方明确非串行化且不能避免 write skew | EXP-04、EXP-05 |
| **P0** | 旧 Worker 事务在 Reclaim 后是否仍可提交 Task | 官方未等价保证 | EXP-04、EXP-05 |
| **P1** | 事务超时与 commit 未知 | 官方未说明 | EXP-07 |
| **P1** | Claim CAS 与事务内校验的组合是否构成隐藏 check-then-write | 架构歧义 | EXP-05 评审记录 |
| **P2** | TransactionBusy 类错误频率（社区个案） | 非官方 SLA | EXP-09 |
| **P2** | 工程 ARC-02 环境/SDK 版本未锁定 | §4 待填 | 工程初始化 |

---

## 15. 文档修订记录

| 版本 | 日期 | 说明 |
| --- | --- | --- |
| 0.1 | 2026-11-09 | 初版任务书；状态 NOT STARTED；含官方桌面调研与实验设计 |

---

## 附录 A — 执行前上下文检查（任务书编写时）

| 检查项 | 结果 | 说明 |
| --- | --- | --- |
| 当前分支和最新提交 | main / `39bbd5b` | `docs(spec): F-04.3.3 终校 07 追踪...` |
| 02 数据模型阅读 | 完成 | §7.4.1 BackgroundJob、Claim Session、`claimVersion`、索引/CAS 要求 |
| 06 技术架构阅读 | 完成 | §14.3—§14.4.7、机制 A/B、Fail-Closed |
| 07 审查报告阅读 | 完成 | §15.7 F-04.3.2-02、§15.8 F-04.3.3、ARC-05 待验证清单 |
| PRD 阅读 | 完成（节选） | 技术方向：小程序 + 云开发 |
| `specs/00-project-charter.md` | 不存在 | 已记录 |
| `docs/agent/STATUS.md` | 不存在 | 已记录 |
| 是否存在 ARC-05 文档 | 否 → **本文件新建** | |
| 工作区未提交修改 | 无 | `git status` clean |
| 与 ARC-05 新 Spec 冲突 | 未发现 | 06 已预留 ARC-05 与 Fail-Closed；无矛盾 |
| `specs/phases/` 目录 | 不存在 → 随本文创建 | 与 07 NEEDS_TECH_VALIDATION 口径一致 |
