# IMPLEMENT_LOG

## 2026-05-26 工程化：部署文档 / 集成测试 / JaCoCo 门禁

**文档与脚本**
- `docs/DEPLOY.md`：阿里云轻量服务器部署（MySQL/Redis/jar/deploy.sh）
- `docs/TESTING.md`：三层测试策略；不推荐 H2 全库替代 MySQL
- `scripts/deploy.sh`、`application-prod.yml`、`application-local.yml.example`
- `README.md` 同步 health、schema 路径、测试/部署链接

**代码小改**
- `AppealAdminController`：UNKNOWN_ERROR 固定文案 + error 日志（对齐 ReportAdmin）
- `HealthControllerTest`、`AppealAdminControllerTest` 补测

**测试基础设施**
- JaCoCo：`mvn verify` 出报告；`-Pcoverage-check` 启用行/分支门禁（默认 45%/35%）
- 集成测试：`ApplicationIntegrationIT`（Testcontainers MySQL+Redis），`RUN_INTEGRATION_TESTS=true mvn verify -Pintegration`

---

## 2026-05-24 M5/M6 对齐：pending_review 不开放申诉

**M5 结论**：`pending_review` 评价不开放用户申诉入口；代码逻辑无冲突；更新 V2.1 文档。

**文档**：
- `TeamMatch 系统详细设计文档` §6.7、§6.8、§7.4、§8.8.2：冻结申诉入口规则；D-11/D-13 P0 不实现
- `M6_API_Documentation.md`：申诉规则与已知缺口同步

**代码**（与文档一致）：
- `AppealServiceImpl`：仅 `status=normal` 可申诉；按 `uk_appeal_target_user` 拦截重复申诉
- `ReportAdminController`：区分 NOT_FOUND / PARAM_ERROR / STATUS_CONFLICT
- `AppealMapper.selectByTargetAndUser` + 相关单测

---

## 2026-05-24 M6 联调收尾（master 对齐）

**背景：** master 已合入 M5-9 评价复核（`6b32911`）及 M6 治理闭环（`44969c1`）。Jenkins 远程已配置，部署脚本暂不做。

### 本轮交付
1. **`M6_API_Documentation.md`**：联调版全量重写——25 接口速查表、M3 登录路径、统一错误码、实体字段、Apifox 测试指南、M2 页面映射。
2. **`EvaluationAdminController`**：鉴权从 `AuthService` 迁移至 `AuthUtil`，与其余 M6 Controller 一致。
3. **`EvaluationAdminControllerTest`**：对齐 AuthUtil + `AuthExceptionHandler` 测试模式。

### M6 模块当前状态

| 功能 | 接口数 | 状态 | 测试 |
|------|--------|------|------|
| 板块管理 (Board) | 6 | 完成 | BoardServiceTest + Controller |
| 举报管理 (Report) | 5 | 完成 | ReportServiceTest + Controller |
| 处罚管理 (Penalty) | 6 | 完成 | PenaltyServiceTest + Controller |
| 申诉管理 (Appeal) | 6 | 完成 | AppealServiceTest + Controller |
| 评价复核 (Review) | 2 | 完成（M5-9 已合入） | EvaluationAdminControllerTest |
| **M6 治理接口合计** | **25** | 见 `M6_API_Documentation.md` | 8 个 Controller 测试 |

### 待协调（发群/当面确认）

| 对象 | 事项 |
|------|------|
| **M2** | 拿更新后的 `M6_API_Documentation.md` 联调；举报处理页需支持 `createPenalty` 等字段 |
| **M5** | ~~D-11/D-13~~ 已对齐：pending_review 不开放申诉，P0 不实现级联 |
| **M3** | M6 写 `user.credit_score` / `user.status` 跨模块是否 OK；预置 admin 账号可用 |
| **产品/全组** | 信誉分下限规则 |

### 仍待（非阻塞联调）
- D-11/D-13 级联实现（P0 不实现）
- 管理端信誉流水审计 API（可选）
- 治理闭环 Apifox 证据（部署环境上手工）
- 联调服务器实际部署（见 `docs/DEPLOY.md`）

---

## 2026-05-19 M6 Controller 统一 AuthUtil 认证

**改动：** M6 全部需鉴权 Controller 从直接调用 `AuthService` 迁移为 `AuthUtil`（`requireUserId` / `requireAdmin`），认证异常交由 `AuthExceptionHandler` 统一返回 M3000/M3009。

**涉及文件：**
- Controller：`PenaltyController`、`PenaltyAdminController`、`ReportController`、`ReportAdminController`、`BoardController`、`AppealController`、`AppealAdminController`
- 测试：M6 全部 7 个 Controller 均有 `*ControllerTest`（含 `AppealControllerTest`、`AppealAdminControllerTest`）；`AppealServiceTest` 覆盖 Service 层

**约定：**
- 用户端：`authUtil.requireUserId(token)`
- 管理端：`authUtil.requireAdmin(token)`；需操作人 ID 时再 `authUtil.requireUserId(token)`

---

## 2026-05-23 M6 治理闭环补全 & 与 M5-9 对齐

### 与 M5 Gerrit（b05cea7）
- 评价复核 HTTP 由 **M5-9** 提供（`EvaluationAdminController`），**已于 master 合入**（`6b32911`）。
- 此前误在本地复制 M5 的 Controller/DTO/测试，已全部删除（见下）。

### M6 本轮交付
1. **evaluation 申诉批准 → 信誉恢复**：`AppealServiceImpl` 调用 `AppealRestoreService.restore(appealId)`，失败 `BusinessException` 同事务回滚。
2. **举报→处罚联动**：`ReportHandleDTO.createPenalty` 等字段；`ReportServiceImpl` 校验后同事务创建处罚。
3. **文档**：`M6_API_Documentation.md` 更新举报/申诉/复核说明。

### 仍待（后续迭代）
- D-11/D-13 复核↔申诉级联（与 M5 协商）
- 信誉流水审计 API（可选）
- health / Jenkins 网站配置 / 部署文档（不在本次范围）

---

## 2026-05-22 M6 模块 CRUD & Code Review（历史）

### 新增功能
1. **申诉管理**（M6-4）：支持 evaluation/penalty 两种目标的申诉提交、审批，penalty 类型批准后自动撤销处罚。
2. **举报-处罚联动**：`ReportServiceImpl.handleReport()` 新增 `createPenalty` 参数，处理举报时可联动创建处罚。

### 重构
- **Controller 鉴权迁移**：全部 Controller 从 `AuthService` 迁移为 `AuthUtil`，统一认证异常处理。
- **新增常量类**：`PenaltyType`（处罚类型）、`CreditChangeType`（流水变更类型）。

### Code Review 发现并修复的问题
| 问题 | 状态 |
|------|------|
| Controller 鉴权重复代码 | 已迁移至 AuthUtil |
| ReportHandleDTO 缺少处罚联动参数 | 已新增 createPenalty/penaltyType/creditDeductValue/penaltyReason |
| ReportServiceImpl 处理举报后无处罚联动 | 已实现 createPenaltyFromReport |
| 缺少 FORBIDDEN 错误码 | 已新增 ReasonCode.FORBIDDEN(M1004) |
| function_limit 撤销时未检查其他生效处罚 | 已实现 restoreUserStatusIfNoActiveFunctionLimit |
| 测试覆盖率不足 | 已补充边界/异常分支测试 |

### M6 模块最终状态

| 功能 | 接口数 | 状态 | 测试 |
|------|--------|------|------|
| 板块管理 (Board) | 6 | 完成 | BoardServiceTest + Controller |
| 举报管理 (Report) | 5 | 完成 | ReportServiceTest + Controller |
| 处罚管理 (Penalty) | 6 | 完成 | PenaltyServiceTest + Controller |
| 申诉管理 (Appeal) | 6 | 完成（含 evaluation 恢复接线） | AppealServiceTest + Controller |
| 评价复核 (Review) | 2 | 完成（M5-9 已合入 master） | EvaluationAdminControllerTest |
| **M6 治理接口** | **25** | 见 `M6_API_Documentation.md` | 8 个 Controller 测试 |

### API 文档
- M6_API_Documentation.md：涵盖全部 23 个接口的请求/响应/权限说明

### 需要队友配合的事项

1. **M3 确认**：`user.credit_score` 和 `user.status` 的跨模块写入是否 OK。
2. **M5 确认**：`credit_change` 表的 penalty / penalty_restore 流水写入是否 OK。
3. **前端确认**：举报处理页面是否展示处罚联动参数（`createPenalty` 等）。
4. **产品确认**：信誉分下限规则（目前可扣至负值，需统一设计后落地）。

---

## 2026-05-18 M4 退出扣分并发边界补测

**问题：** `selfExit` 已通过 `memberRows != 1` 拦截 team_member CAS 异常结果，但原单测只覆盖 `memberRows == 0` 的并发抢占场景，未显式覆盖异常多行边界。

**修复：** `ExitVoteServiceTest` 补充 `memberRows > 1` 分支，断言返回 `TEAM_VOTE_CONFLICT`，且不写 `credit_change`、不更新 `user.credit_score`。

**测试证据：** `mvn -Dtest=ExitVoteServiceTest test`，38 测试全部通过。

---

## 2026-05-15 M5 Service 层收口风险修复

本轮修复了 M5-1 到 M5-7 Service 层的 2 个 P0 阻塞问题和 1 个 P1 问题。

### P0-1: credit_change 混查修复

**问题：** `CreditChangeMapper.findBySourceTypeAndSourceIds` 只限定 `source_type`/`source_id`/`effective=1`，未限定 `change_type`。异常挂起（`handleAnomalyTriggered`）可能把同一 `evaluationId` 下的 `appeal_restore` 流水也查出来并置为 `effective=false`，污染账本，并可能导致后续 `appeal_restore` 幂等检查误判 `alreadyRestored`。

**修复：** `CreditChangeMapper.findBySourceTypeAndSourceIds` 增加 `AND change_type = 'evaluation'` 条件，仅查询原始互评流水。

**变更文件：** `CreditChangeMapper.java`

### P0-2: approve 重复计分修复

**问题：**
1. `EvaluationMapper.updateReview` 的 WHERE 条件缺少 `status='pending_review'`，并发 approve 可重复更新状态。
2. `CreditChangeMapper.batchUpdateEffective` 的 WHERE 条件缺少 `effective=0` 守卫，并发 approve 可重复生效同一流水。
3. `void` / `keep_no_credit` 同样缺少条件更新守卫。

**修复：**
1. 新增 `EvaluationMapper.updateReviewIfPending`：WHERE 增加 `AND status = 'pending_review'`。
2. 新增 `CreditChangeMapper.updateEffectiveToTrue`：原子性翻转 effective 0→1，限定完整条件（id + source_type + source_id + change_type + user_id + effective=0）。
3. `EvaluationReviewServiceImpl.executeApprove` / `executeVoid` / `executeKeepNoCredit` 改用条件更新方法，rows != 1 时抛 `STATUS_CONFLICT`。
4. `executeApprove` 中仅当 effective 0→1 更新 rows == 1 时才调用 `updateCreditScore`。

**变更文件：** `EvaluationMapper.java`、`CreditChangeMapper.java`、`EvaluationReviewServiceImpl.java`

### P1: 重复提交唯一键冲突映射 ALREADY_EVALUATED

**问题：** 并发重复提交互评时，第二个 insert 可能命中 `uk_project_evaluator_target` 唯一键冲突，当前未显式转换为 `ALREADY_EVALUATED`，可能暴露为未知异常。

**修复：** `EvaluationSubmitServiceImpl.submit` 中 `evaluationMapper.insert` 捕获 `DuplicateKeyException`，返回 `Result.fail(ReasonCode.ALREADY_EVALUATED)`。不引入锁。

**变更文件：** `EvaluationSubmitServiceImpl.java`

### skill_endorse 当前限制

`skill_endorse` 目前仅 DDL / 实体到位，提交互评链路暂不写入。该能力不影响信誉分账本，后续由 M5-8 / M5-9 或专门任务承接。

### 测试证据

- `EvaluationSubmitServiceTest`: 24 测试（含新增 `anomalyShouldNotSuspendAppealRestoreChanges`、`duplicateEvaluationInsertShouldReturnAlreadyEvaluated`）
- `EvaluationReviewServiceTest`: 28 测试（含新增 `approveDuplicateShouldUpdateCreditScoreOnlyOnce`，已更新现有测试适配新 Mapper 方法签名）
- `AppealRestoreServiceTest`: 19 测试（无变更）
- `CreditCalculationServiceTest`: 8 测试（无变更）
- `CreditRuleServiceImplTest`: 35 测试（无变更）
- 合计 **114 测试，全部通过**

---

## 2026-05-12 阶段进度总览

| 模块 | 负责 | 进度 | 关键产出 |
|---|---|---|---|
| M3 认证/档案/技能 | 后端一 | ~95% | Auth + Profile Controller/Service，权限基础，技能标签 CRUD |
| M4 项目/组队/退出 | 后端二 | ~90% | Project/TeamRequest/ExitVote Controller + Service，并发关闭协议，35+17 测试 |
| M5 互评/信誉/申诉 | 后端三 | ~80% | 7/7 Service 完成（M5-1~M5-7），143 测试，**Controller 待补** |
| M6 治理/部署 | 治理 | ~30% | Board 板块管理完成，Report/Penalty/申诉审批/Jenkins 未开始 |
| M1 前端一 | 前端 | ~60-70% | 认证/档案/冷启动/项目浏览（7 页面） |
| M2 前端二+后台 | 前端 | ~50-60% | 组队/退出/互评/信誉展示/管理端（6+5 页面） |

**后端总体 ~74%，全项目总体 ~65-70%。** Phase 2 收尾阶段。

**M5 待办：**
- Controller 接口（互评资格/提交/列表/信誉分/评价复核管理端）
- 与 M2 对齐互评页面字段、信誉分展示
- 与 M6 对齐评价复核管理端入口、申诉流程边界
- M5-7 Gerrit Change: https://（已脱敏）/c/shixun12_backend/+/2792

---

## 2026-05-12 M5-7 申诉恢复 Service

- 新增 `AppealRestoreService` + `AppealRestoreServiceImpl`：对已批准的 `evaluation` 类申诉执行信誉恢复，追加 `appeal_restore` 流水并回写 `user.credit_score`，全链路 `@Transactional`。
- 新增 DTO：`AppealRestoreCommand`（入参，仅 appealId）、`AppealRestoreResult`（出参，含 skipped/alreadyRestored/restoreValue）。
- 新增实体：`Appeal`（最小化字段：id / userId / targetType / targetId / status），M6 可扩展完整字段映射。
- 新增 Mapper：`AppealMapper`（继承 BaseMapper，selectById 自动可用）。
- 新增 Mapper 方法：
  - `CreditChangeMapper.findBySourceTypeAndSourceIdAndChangeType`：按精确四维键查原评价流水（不预判 effective，由业务代码判断）。
  - `CreditChangeMapper.findAppealRestoreExists`：幂等检查，查是否已有 `appeal_restore` 流水。
  - `EvaluationMapper.selectByIdForUpdate`：SELECT FOR UPDATE 锁定 evaluation 行（V2.1 8.7 D-11/D-13）。
- 新增 ReasonCode：`M5009`(APPEAL_NOT_FOUND)、`M5010`(APPEAL_NOT_APPROVED)、`M5011`(INVALID_APPEAL_TARGET_TYPE)、`EVALUATION_ALREADY_INVALIDATED`（字符串常量名）。
- 删除 ReasonCode：`M5012`(APPEAL_RESTORE_ALREADY_EXECUTED) —— 幂等分支改为 `success(alreadyRestored=true)`，该常量不会被任何业务路径返回。
- 关键契约（V2.1 6.7 冻结口径）：
  - 仅处理 `appeal.target_type='evaluation'`，其他类型抛 `INVALID_APPEAL_TARGET_TYPE`。`function_limit` 申诉恢复由 M6 单独处理，不在本服务范围。
  - 入口 null 防护：`command` 或 `appealId` 为 null 返回 `PARAM_ERROR`。
  - userId 一致性校验：`appeal.userId` 必须等于 `evaluation.targetId`（使用 `Objects.equals`），不一致返回 `PARAM_ERROR` + `log.warn`。
  - voided 评价已判无效，直接拒绝返回 `EVALUATION_ALREADY_INVALIDATED`。
  - 原流水 `effective=0`（pending_review / kept_no_credit）→ `success(skipped=true)`，不视为失败。
  - 幂等：同一 `(source_type, source_id, change_type, user_id)` 已有 `appeal_restore` 流水 → `success(alreadyRestored=true)`，不视为失败。
  - 恢复流水 `change_value = -original.changeValue`，`effective=true`，`source_type='evaluation'`。
  - 恢复值以已锁定的 `evaluation.getId()` 作为 `sourceId`（而非 `appeal.getTargetId()`）。
  - 代码中所有字符串字面量（`"evaluation"` / `"approved"` / `"appeal_restore"`）已提取为 `static final` 常量。
- 测试：19 单测全量通过，覆盖正常恢复（负分/正分/零分）、幂等跳过、effective=0 跳过、7 种异常分支（appeal 不存在/未批准/非 evaluation 类型/userId 不一致/evaluation 不存在/voided/原流水缺失）、2 种事务回滚（insert 失败/updateCreditScore 失败）。
- 已知限制：
  - ~~无 Controller 接口入口~~（已由 M6 `AppealServiceImpl` 在批准申诉时调用 `AppealRestoreService`）。
  - `function_limit` 申诉恢复由 M6 单独实现，不在 M5-7 范围。
  - SELECT FOR UPDATE 锁竞争场景标注为集成测试/手工覆盖，不在 JUnit 单测中强制要求。

## 2026-05-12 M5-6 评价复核 Service

- 新增 `EvaluationReviewService` + `EvaluationReviewServiceImpl`：对 `pending_review` 评价执行 `approve` / `void` / `keep_no_credit` 三种复核操作，`@Transactional` 保证 evaluation.status + credit_change.effective + user.credit_score 原子更新。
- 新增 DTO：`EvaluationReviewCommand`（入参）、`EvaluationReviewResult`（返回值，含 oldStatus / newStatus / creditDelta / creditEffectiveChanged）。
- 新增 `ReviewAction` 枚举（`approve` / `void` / `keep_no_credit`），入口校验无效 action 返回 `INVALID_REVIEW_ACTION`。
- 新增 ReasonCode：`M5005`(EVALUATION_NOT_FOUND)、`M5006`(CREDIT_CHANGE_NOT_FOUND)、`M5007`(INVALID_REVIEW_ACTION)、`M5008`(REVIEW_NOTE_TOO_LONG)。
- 新增 Mapper 方法：`CreditChangeMapper.findSuspendedOne`（限定 source_type + source_id + change_type + user_id + effective=0 查询单条挂起流水）、`EvaluationMapper.updateReview`（更新 status + reviewer_id + review_note + reviewed_at + updated_at）。
- 关键契约：
  - `approve`：evaluation.status → normal，credit_change.effective 0→1，user.credit_score 增量加回（不绕开 credit_change）。
  - `void`：evaluation.status → voided，credit_change.effective 保持 0，不更新 credit_score。
  - `keep_no_credit`：evaluation.status → kept_no_credit，credit_change.effective 保持 0，不更新 credit_score。
  - 重复 approve 或其他状态提交 → `STATUS_CONFLICT`，不重复加分。
- 测试：26 单测全量通过，覆盖三种成功路径、5 种异常分支（不存在/状态冲突/流水缺失/已生效/备注过长）、3 种落库失败回滚。
- 已知限制：尚无 Controller 暴露复核 API（M5 专注底层能力），集成测试待后续补充。
- Gerrit Change: I7431dab4df3b7348cdf3b2e0e76740b9556c0ca3

## 2026-05-12 M5-5 补充提交

- `Evaluation` 新增互评状态常量：`normal`、`pending_review`、`voided`、`kept_no_credit`。
- `EvaluationSubmitServiceImpl` 将互评提交链路中的 `normal` / `pending_review` 状态字符串替换为常量引用，并补充精简日志：提交入口、异常挂起触发、信誉分更新失败。
- 已知限制：V2.1 当前未定义全局信誉分下限，`user.credit_score` 理论上可被有效 `credit_change` 扣至负值。由于 `user.credit_score = 100 + SUM(change_value WHERE effective=1)` 是由 `credit_change` 推导出的缓存，直接在 SQL 中截断会破坏异常撤销、申诉恢复和审计时的流水一致性。本轮不引入 `GREATEST` 下限，待全组确认下限规则后统一设计落地。
