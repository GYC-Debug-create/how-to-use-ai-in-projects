# M5 任务拆解

> ⚠️ **当前状态 (2026-05)**：M5-1 到 M5-7 的 Service 层已全部完成（资格校验、内容校验、信誉计算、异常检测、提交编排、评价复核、申诉恢复）。**Controller 层待补。** 本文档保留原始任务拆解结构供参考。

本文件用于 M5 开发时按小任务推进。每次只选择一个小任务，不要一次实现整个 M5。

## 0. 开发前固定动作

**新会话首次处理 M5 前**，必须读取 `docs/design/system-design-v2.1.md` 的关键章节：

- 0.3 冻结规则
- 0.4 P0/P1 边界
- 1.6 DDL 顺序
- 6 板块五：互评与信誉
- 7 中 evaluation / appeal / review 相关内容
- 8 reasonCode
- 9.1 事务边界

**同一会话后续小任务**，默认读取：

- `CLAUDE.md`
- `docs/m5/M5_CONTEXT.md`
- `docs/m5/M5_TODO.md`
- 当前要改的代码文件
- `IMPLEMENT_LOG.md` 最后一段

**遇到冲突、歧义、跨模块、状态机、事务、reasonCode、credit_change 时**，必须回查 V2.1。

**列出本次要改的文件和测试点**。

## 1. 数据库

- 确认 `evaluation` 表字段与唯一约束
  - 四维字段：`communication_score`, `task_score`, `skill_score`, `responsibility_score`
  - 唯一约束：`UNIQUE KEY uk_project_evaluator_target (project_id, evaluator_id, target_id)`
- 确认 `evaluation_tag` 表关联关系
- 确认 `skill_endorse` 只展示、不影响信誉分
- 确认 `credit_change` 流水字段与索引需求
- 确认 `appeal` 支持 `target_type='evaluation'`
  - 关键字段：`target_type`, `target_id`
  - 唯一约束建议：`UNIQUE KEY uk_target (target_type, target_id, user_id)` 配合 `status='pending'` 业务校验
- 确认 `credit_change` 追加为主，仅 `effective` 可由异常检测 / 复核流程按规则修改
- 确认不新增 `is_deleted`

## 2. 实体

- 创建或补齐 `Evaluation` 实体
- 创建或补齐 `EvaluationTag` 实体
- 创建或补齐 `SkillEndorse` 实体
- 创建或补齐 `CreditChange` 实体
- 创建或补齐 M5 需要的 `Appeal` 字段映射
- 使用英文枚举值，不用数字状态

## 3. Mapper / Repository

- `evaluation` 按项目、评价者、被评价者查询
- `evaluation` 检查重复评价
- `evaluation` 批量标记 `pending_review`
- `credit_change` 追加流水
- `credit_change` 查询用户项目内评价类流水累计值
- `credit_change` 查询和恢复 `effective`
- `appeal_restore` 幂等查询
- `appeal` 查询同一评价是否已有待处理申诉

## 4. Service

### 4.1 互评资格

- 校验当前用户是项目正式成员
- 校验目标用户是项目正式成员
- 校验项目处于 `ended`
- 校验仍在 7 日互评窗口
- 校验不能重复评价
- 校验不能评价自己
- 输出稳定 `reasonCode`

### 4.2 提交互评

- 校验四维分数范围
- 校验 1 / 2 分必须附负向标签和不少于 20 字说明
- 校验 5 分必须附正向标签
- 保存 `evaluation`
- 保存 `evaluation_tag`
- 保存 `skill_endorse`
- 计算四维均值
- 计算 `change_value`
- 应用单项目 ±10 封顶截断
- 写入 `credit_change`
- 更新 `user.credit_score`
- 同步触发异常评价检测
- 必须在同一事务中完成

### 4.3 异常评价

- 判断评价者是否已覆盖全部队友
- 全低分命中：批量 `pending_review`
- 全满分命中：批量 `pending_review`
- 对应 `credit_change.effective=0`
- 回退已计入的 `user.credit_score`

### 4.4 评价复核

- `approve`：`pending_review → normal`，恢复 `effective=1`，加回信誉分
- `void`：`pending_review → voided`，保持不计分
- `keep_no_credit`：`pending_review → kept_no_credit`，保持不计分
- 非 `pending_review` 状态禁止复核
- 必须在同一事务中完成

### 4.5 申诉恢复

- 仅处理 `appeal(target_type='evaluation')` 的 M5 相关部分
- 申诉通过时不改原始事实流水
- 原流水 `effective=1` 时追加 `appeal_restore`
- 原流水 `effective=0` 时跳过恢复
- 同一 `(source_type, source_id)` 已有恢复流水时跳过
- 回写 `user.credit_score`
- 必须在同一事务中完成

## 5. Controller

- 互评资格检查接口
- 提交互评接口
- 项目互评列表接口
- 我收到的互评接口
- 我的信誉分接口
- 信誉分变化记录接口
- 提交评价申诉接口或对接申诉模块入口
- 管理端评价复核接口

接口路径开发前确认；不要在本文件中把未确认路径当作冻结契约。

## 6. 测试

- 互评资格单元测试
- 信誉分计算单元测试
- 项目级封顶截断单元测试
- 异常评价挂起测试
- `credit_change.effective` 测试
- 评价复核三分支测试
- `appeal_restore` 幂等测试
- 接口 `reasonCode` 测试
- 事务回滚测试

## 7. 联调

- 与 M2 对齐互评页面字段
- 与 M2 对齐信誉分展示与 `effective=false` 展示口径
- 与 M6 对齐评价复核管理端入口
- 与 M6 对齐 `appeal(target_type='evaluation')` 流程边界
- 保留接口请求截图、测试结果和 Review 证据

## 8. 预计代码落点清单

本清单仅供规划参考，不要在本阶段创建代码文件。

### Entity

- `com.teammatch.entity.Evaluation`
  - 关键字段：communication_score, task_score, skill_score, responsibility_score
- `com.teammatch.entity.EvaluationTag`
- `com.teammatch.entity.SkillEndorse`
- `com.teammatch.entity.CreditChange`
- `com.teammatch.entity.Appeal`（M5 只补充 `target_type='evaluation'` 相关字段映射）
  - 关键字段：target_type, target_id

### Repository / Mapper

- `com.teammatch.repository.EvaluationRepository`
- `com.teammatch.repository.EvaluationTagRepository`
- `com.teammatch.repository.SkillEndorseRepository`
- `com.teammatch.repository.CreditChangeRepository`
- `com.teammatch.repository.AppealRepository`（M5 只补充评价申诉查询方法）

### Service

- `com.teammatch.service.EvaluationService`
- `com.teammatch.service.CreditChangeService`
- `com.teammatch.service.EvaluationReviewService`
- `com.teammatch.service.AppealRestoreService`（或合并到 `AppealService` 中）

### Controller

- `com.teammatch.controller.EvaluationController`
- `com.teammatch.controller.CreditChangeController`
- `com.teammatch.controller.EvaluationReviewController`（管理端）

### Test

- `com.teammatch.service.EvaluationServiceTest`
- `com.teammatch.service.CreditChangeServiceTest`
- `com.teammatch.service.EvaluationReviewServiceTest`
- `com.teammatch.controller.EvaluationControllerTest`
- `com.teammatch.controller.CreditChangeControllerTest`

实际包名、类名、文件组织方式以项目约定为准，本清单只是最小参考。
