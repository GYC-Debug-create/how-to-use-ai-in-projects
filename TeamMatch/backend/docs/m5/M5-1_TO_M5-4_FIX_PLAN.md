# M5-1 到 M5-4 规则层修复计划

审查基准：

- `docs/design/system-design-v2.1.md`
- `docs/m5/M5_CONTEXT.md`
- `docs/m5/M5_TEST_PLAN.md`
- `docs/m5/M5-1_REVIEW_ISSUES.md`
- `docs/m5/M5-2_REVIEW_ISSUES.md`
- `docs/m5/M5-3_REVIEW_ISSUES.md`
- `docs/m5/M5-4_REVIEW_ISSUES.md`

本计划用于后续直接执行代码修改。目标是修正 M5-1 到 M5-4 现有规则层实现和单元测试，使其对齐 V2.1，同时避免影响 M3 / M4 / M6 既有业务。

## 1. 本轮边界

本轮做：

- 修 M5-1 资格判断、稳定 `reasonCode`、互评窗口懒关闭。
- 修 M5-2 低分说明长度、正负向标签白名单。
- 修 M5-3 项目封顶计算的双重读取问题。
- 重写 M5-4 异常评价检测为覆盖全部队友后的批量语义。
- 补齐对应单元测试和防回归测试。
- 追加 `IMPLEMENT_LOG.md` 测试摘要。

本轮不做：

- 不写 Controller。
- 不实现完整提交编排服务。
- 不插入 `evaluation` / `evaluation_tag` / `skill_endorse`。
- 不写 `credit_change`。
- 不更新 `user.credit_score`。
- 不实现异常命中后的事务内批量回滚。
- 不做大规模包迁移。
- 不改变 M4 退出投票语义。
- 不实现 M6 处罚、申诉、复核流程。

本轮标题建议为：

```text
M5-1~M5-4 规则层修复
```

不要称为完整提交全链路。完整提交链路留到下一轮：

```text
validation -> insert evaluation -> insert tags/endorsements -> credit calculation -> anomaly detection -> batch status/effective/score update
```

该链路后续必须放在同一 `@Transactional` 中完成。

## 2. 最终裁决

### 2.1 M5-4 异常检测算法

采用“每条评价均分都满足”的算法，不采用所有评价混在一起的总平均 `grand_average`。

规则 A，全低分：

```text
评价者对同项目所有应评价队友的每条评价四维均分都 <= 2
```

规则 B，全满分：

```text
评价者对同项目所有应评价队友的每条评价四维均分都 = 5
```

示例：

```text
队友 A: 1,1,1,1 avg=1.0
队友 B: 3,3,3,3 avg=3.0
```

结果：不触发异常。因为 B 的评价均分不满足 `<= 2`。

### 2.2 `EVALUATOR_NOT_MEMBER`

不保留独立对外码。评价人不是项目成员统一返回：

```text
NOT_PROJECT_MEMBER
```

目标用户不是项目成员返回：

```text
TARGET_NOT_PROJECT_MEMBER
```

### 2.3 M5-3 封顶计算

新增纯函数，不改旧签名。

新增：

```java
int calculateCappedDeltaFromExisting(int existingProjectDelta, int proposedDelta);
```

保留：

```java
int calculateCappedDelta(Long userId, Long projectId, int proposedDelta);
```

旧方法内部委托新方法，保证 M4 现有调用不受影响。

### 2.4 实体和 Mapper

本轮不迁移以下共享对象：

- `com.teammatch.entity.Evaluation`
- `com.teammatch.entity.CreditChange`
- `com.teammatch.mapper.EvaluationMapper`
- `com.teammatch.mapper.CreditChangeMapper`

理由：

- `CreditChange` 会被 M4 退出扣分、M5 互评计分、M6 处罚/申诉恢复共用。
- `Evaluation` 后续会被 M6 评价复核和评价申诉读取或更新。
- 现在迁到 `m5.*` 会制造跨模块依赖方向问题。

## 3. Phase 1：ReasonCode 对齐

预计时间：5 到 10 分钟。

修改文件：

- `src/main/java/com/teammatch/common/ReasonCode.java`
- `src/main/java/com/teammatch/service/EvaluationEligibilityService.java`
- 相关测试文件

资格类稳定码：

```text
NOT_PROJECT_MEMBER
TARGET_NOT_PROJECT_MEMBER
PROJECT_NOT_ENDED
EVAL_WINDOW_CLOSED
ALREADY_EVALUATED
SELF_EVALUATION
```

调整：

```text
SELF_EVALUATION_NOT_ALLOWED -> SELF_EVALUATION
DUPLICATE_EVALUATION -> ALREADY_EVALUATED
TARGET_NOT_MEMBER -> TARGET_NOT_PROJECT_MEMBER
EVALUATOR_NOT_MEMBER -> 删除或不再使用，对外统一 NOT_PROJECT_MEMBER
```

内容校验类建议稳定码：

```text
SCORE_FIELD_MISSING
SCORE_OUT_OF_RANGE
COMMENT_CONTAINS_VIOLATION
HIGH_SCORE_MISSING_POSITIVE_TAG
LOW_SCORE_MISSING_NEGATIVE_TAG
LOW_SCORE_COMMENT_TOO_SHORT
```

说明：

- 若当前 `Result.code` 已作为前端业务判断字段，则 M5 相关失败响应应返回上述稳定字符串。
- 不迁移 `Result`，不改公共响应结构字段名。

测试要求：

- 所有 M5 测试不再断言 `M5001` / `M5101` 这类数字码。
- 改为断言稳定字符串，如 `NOT_PROJECT_MEMBER`。

## 4. Phase 2：M5-1 资格链路和 eval_closed 懒关闭

预计时间：20 到 30 分钟。

修改文件：

- `src/main/java/com/teammatch/service/EvaluationEligibilityService.java`
- `src/main/java/com/teammatch/mapper/ProjectMapper.java`
- `src/test/java/com/teammatch/service/EvaluationEligibilityServiceTest.java`

### 4.1 统一资格失败码

`validateSubmission` 中按以下规则返回：

```text
evaluatorId == targetId -> SELF_EVALUATION
已评价过 -> ALREADY_EVALUATED
评价人不是 active 成员 -> NOT_PROJECT_MEMBER
目标用户不是 active 成员 -> TARGET_NOT_PROJECT_MEMBER
项目未 ended -> PROJECT_NOT_ENDED
互评窗口关闭 -> EVAL_WINDOW_CLOSED
```

### 4.2 懒关闭

只在以下条件同时满足时执行懒关闭：

```text
project.status == ended
evalDeadline != null
now > evalDeadline
```

动作：

```text
ended -> eval_closed
```

建议优先使用条件更新，避免并发覆盖：

```text
where id = ? and status = 'ended'
set status = 'eval_closed'
```

`ProjectMapper` 明确新增方法：

```java
@Update("UPDATE project SET status = 'eval_closed' WHERE id = #{id} AND status = 'ended'")
int updateStatusToEvalClosed(@Param("id") Long id);
```

`EvaluationEligibilityService` 中窗口过期分支调用该方法，不使用 `updateById` 做懒关闭。

注意：

- `evalDeadline == null` 不等价于窗口过期。
- `evalDeadline == null` 可以返回 `EVAL_WINDOW_CLOSED`，但不要把项目静默改成 `eval_closed`。

测试新增：

- `checkProjectEligibility` 窗口过期时触发 `eval_closed`。
- `getEvaluatableMembers` 通过项目资格检查路径触发懒关闭。
- `validateSubmission` 通过提交前兜底路径触发懒关闭。
- `evalDeadline == null` 返回 `EVAL_WINDOW_CLOSED`，但不调用状态更新。
- 非 `ended` 状态不触发懒关闭。

## 5. Phase 3：M5-2 内容校验

预计时间：30 到 45 分钟。

修改文件：

- `src/main/java/com/teammatch/service/EvaluationValidationService.java`
- `src/main/java/com/teammatch/config/ValidationConfig.java`
- `src/test/java/com/teammatch/service/EvaluationValidationServiceTest.java`

### 5.1 低分说明不少于 20 字

规则：

```text
任一维度为 1 或 2 时，comment.trim().length() >= 20
```

失败码：

```text
LOW_SCORE_COMMENT_TOO_SHORT
```

测试：

- 低分说明 19 字失败。
- 低分说明 20 字通过。
- 低分说明 21 字通过。
- 原有低分通过测试里的短说明必须替换为不少于 20 字。

### 5.2 标签白名单

正向标签：

```text
沟通积极
按时交付
技术可靠
责任心强
```

负向标签：

```text
沟通差
延期
质量低
失联
```

校验规则：

- 有任一维度为 5 时，`positiveTags` 必须非空。
- 有任一维度为 5 时，`positiveTags` 至少包含一个合法正向标签。
- 有任一维度为 5 时，`positiveTags` 中全部标签都必须属于正向白名单。
- 有任一维度为 1 或 2 时，`negativeTags` 必须非空。
- 有任一维度为 1 或 2 时，`negativeTags` 至少包含一个合法负向标签。
- 有任一维度为 1 或 2 时，`negativeTags` 中全部标签都必须属于负向白名单。

测试：

- 未知正向标签失败。
- 未知负向标签失败。
- 混入未知正向标签失败。
- 混入未知负向标签失败。
- 定稿正向标签通过。
- 定稿负向标签通过。

## 6. Phase 4：M5-3 信誉计算

预计时间：30 到 45 分钟。

修改文件：

- `src/main/java/com/teammatch/m4/service/CreditRuleService.java`
- `src/main/java/com/teammatch/m4/service/impl/CreditRuleServiceImpl.java`
- `src/main/java/com/teammatch/service/impl/CreditCalculationServiceImpl.java`
- `src/test/java/com/teammatch/m4/service/CreditRuleServiceImplTest.java`
- `src/test/java/com/teammatch/service/CreditCalculationServiceTest.java`

### 6.1 新增纯函数

接口新增：

```java
int calculateCappedDeltaFromExisting(int existingProjectDelta, int proposedDelta);
```

实现：

```java
public int calculateCappedDeltaFromExisting(int existingProjectDelta, int proposedDelta) {
    if (proposedDelta == 0) {
        return 0;
    }

    int targetTotal = existingProjectDelta + proposedDelta;
    if (targetTotal > 10) {
        return Math.max(0, 10 - existingProjectDelta);
    }
    if (targetTotal < -10) {
        return Math.min(0, -10 - existingProjectDelta);
    }
    return proposedDelta;
}
```

旧方法保留：

```java
public int calculateCappedDelta(Long userId, Long projectId, int proposedDelta) {
    int existing = getExistingProjectDelta(userId, projectId);
    return calculateCappedDeltaFromExisting(existing, proposedDelta);
}
```

### 6.2 M5 编排改用单次快照

`CreditCalculationServiceImpl` 中：

```text
existingProjectDelta = getExistingProjectDelta(targetId, projectId)
rawDelta = mapAverageScoreToDelta(avg)
cappedDelta = calculateCappedDeltaFromExisting(existingProjectDelta, rawDelta)
projectTotalAfterApplied = existingProjectDelta + cappedDelta
```

禁止：

```text
先读 existingProjectDelta，再调用内部又读一次 DB 的 calculateCappedDelta(userId, projectId, rawDelta)
```

测试：

- 新增 `calculateCappedDeltaFromExisting` 纯函数测试。
- 保留旧 `calculateCappedDelta` 兼容测试。
- 补 `getExistingProjectDelta` 查询条件防回归测试。
- 查询条件至少验证：
  - `userId`
  - `projectId`
  - `changeType = evaluation`
  - `effective = true`
- 修正测试命名与实际输入不一致的问题。

## 7. Phase 5：M5-4 异常检测

预计时间：60 到 90 分钟。

修改文件：

- `src/main/java/com/teammatch/service/AnomalyDetectionService.java`
- `src/main/java/com/teammatch/dto/AnomalyDetectionResult.java`
- `src/main/java/com/teammatch/mapper/EvaluationMapper.java`
- `src/test/java/com/teammatch/service/AnomalyDetectionServiceTest.java`

### 7.1 服务接口

废弃或删除单条检测接口：

```java
detect(EvaluationSubmitDTO dto)
detect(int communicationScore, int taskScore, int skillScore, int responsibilityScore)
```

新增批量语义接口：

```java
AnomalyDetectionResult detect(Long evaluatorId, Long projectId);
```

服务注入：

- `EvaluationMapper`
- `TeamMemberMapper`

### 7.2 逻辑

流程：

```text
1. 查询项目 active 成员列表。
2. 排除 evaluator 自己，得到应评价人数 expectedTargetCount。
3. 查询 evaluator 在 project 下已提交的评价列表 submittedEvaluations。
4. 如果 submittedEvaluations.size < expectedTargetCount，返回 INCOMPLETE_COVERAGE。
5. 如果 expectedTargetCount == 0，返回 NORMAL。
6. 计算每条评价的四维均分。
7. 所有评价均分都 <= 2，返回 EXTREME_LOW。
8. 所有评价均分都 = 5，返回 EXTREME_PERFECT。
9. 否则返回 NORMAL。
```

命中异常时：

```text
affectedEvaluationIds = submittedEvaluations 的全部 id
triggered = true
coverageComplete = true
```

未覆盖时：

```text
coverageComplete = false
triggered = false
reason = INCOMPLETE_COVERAGE
affectedEvaluationIds = empty
```

正常时：

```text
coverageComplete = true
triggered = false
reason = NONE
affectedEvaluationIds = empty
```

### 7.3 DTO

`AnomalyDetectionResult` 字段建议：

```java
private boolean coverageComplete;
private boolean triggered;
private String reason;
private List<Long> affectedEvaluationIds;
```

reason 取值：

```text
NONE
INCOMPLETE_COVERAGE
EXTREME_LOW
EXTREME_PERFECT
```

### 7.4 Mapper

`EvaluationMapper` 需要新增查询：

```java
List<Evaluation> findByEvaluatorAndProject(Long evaluatorId, Long projectId);
```

建议查询条件：

```sql
WHERE evaluator_id = #{evaluatorId}
  AND project_id = #{projectId}
```

是否排除 `voided` 留到提交编排/复核阶段。本轮 M5-4 只处理 M5-4 自己产生前的提交集合，不扩展复核语义。

### 7.5 测试

必须覆盖：

- 未覆盖全部队友，返回 `INCOMPLETE_COVERAGE`。
- 覆盖全部队友，每条评价均分都 `<= 2`，返回 `EXTREME_LOW`，并返回全部 evaluation id。
- 覆盖全部队友，每条评价均分都 `= 5`，返回 `EXTREME_PERFECT`，并返回全部 evaluation id。
- 覆盖全部队友，部分低分、部分正常，不触发。
- 覆盖全部队友，均分恰好 `2.0`，触发全低分。
- 覆盖全部队友，均分恰好 `5.0`，触发全满分。
- 仅 2 名项目成员时，评价 1 名队友即完成覆盖。

## 8. Phase 6：注入、断言和测试整理

预计时间：20 到 30 分钟。

该 Phase 不单独形成 Gerrit 变更，应分散到 M5-1 到 M5-4 四个对应变更中，避免产生纯风格提交和重复返工。

修改范围：

- `EvaluationEligibilityService`
- `EvaluationValidationService`
- `CreditCalculationServiceImpl`
- `AnomalyDetectionService`
- M5 相关测试

### 8.1 注入方式

将字段注入改为构造器注入：

```java
@RequiredArgsConstructor
public class EvaluationEligibilityService {
    private final ProjectMapper projectMapper;
    private final TeamMemberMapper teamMemberMapper;
    private final EvaluationMapper evaluationMapper;
    private final UserMapper userMapper;
}
```

`EvaluationValidationService` 同理。

`CreditCalculationServiceImpl` 同理，使用 `@RequiredArgsConstructor` + `private final CreditRuleService creditRuleService`。

`AnomalyDetectionService` 重写时直接使用构造器注入。

### 8.2 断言

M5 相关测试统一使用 AssertJ：

```java
assertThat(actual).isEqualTo(expected);
```

不要混用大量 JUnit 原生 `assertEquals`。

### 8.3 测试命名

修正命名与实际输入不一致的测试，例如：

```text
upper_bound_4_49 -> avg_4_25
lower_bound_2_99 -> avg_2_75
```

如果要保留原命名，则补真实 `4.49` / `2.99` 输入测试。

### 8.4 Phase 6 归属

执行时按以下方式归属，不创建单独 Gerrit：

| 归属 | Phase 6 内容 |
|---|---|
| M5-1 Gerrit | `EvaluationEligibilityService` 构造器注入 |
| M5-2 Gerrit | `EvaluationValidationService` 构造器注入 |
| M5-3 Gerrit | `CreditCalculationServiceImpl` 构造器注入、AssertJ 断言统一、测试命名修正 |
| M5-4 Gerrit | `AnomalyDetectionService` 构造器注入 |

## 9. 预计波及文件

预计修改 15 到 18 个文件。

必改：

```text
src/main/java/com/teammatch/common/ReasonCode.java
src/main/java/com/teammatch/service/EvaluationEligibilityService.java
src/main/java/com/teammatch/service/EvaluationValidationService.java
src/main/java/com/teammatch/service/AnomalyDetectionService.java
src/main/java/com/teammatch/service/impl/CreditCalculationServiceImpl.java
src/main/java/com/teammatch/m4/service/CreditRuleService.java
src/main/java/com/teammatch/m4/service/impl/CreditRuleServiceImpl.java
src/main/java/com/teammatch/config/ValidationConfig.java
src/main/java/com/teammatch/dto/AnomalyDetectionResult.java
src/main/java/com/teammatch/mapper/EvaluationMapper.java
src/test/java/com/teammatch/service/EvaluationEligibilityServiceTest.java
src/test/java/com/teammatch/service/EvaluationValidationServiceTest.java
src/test/java/com/teammatch/service/AnomalyDetectionServiceTest.java
src/test/java/com/teammatch/m4/service/CreditRuleServiceImplTest.java
src/test/java/com/teammatch/service/CreditCalculationServiceTest.java
IMPLEMENT_LOG.md
```

可能改：

```text
src/main/java/com/teammatch/mapper/ProjectMapper.java
```

不改：

```text
src/main/java/com/teammatch/common/Result.java
src/main/java/com/teammatch/entity/Evaluation.java
src/main/java/com/teammatch/entity/CreditChange.java
src/main/java/com/teammatch/entity/User.java
src/main/java/com/teammatch/entity/Project.java
src/main/java/com/teammatch/entity/TeamMember.java
src/main/java/com/teammatch/m4/service/impl/ExitVoteServiceImpl.java
M3 业务代码
M6 业务代码
Controller
```

## 10. 验证命令

分项验证：

```bash
mvn -Dtest=EvaluationEligibilityServiceTest test
mvn -Dtest=EvaluationValidationServiceTest test
mvn "-Dtest=CreditRuleServiceImplTest,CreditCalculationServiceTest" test
mvn -Dtest=AnomalyDetectionServiceTest test
```

全量验证：

```bash
mvn test
```

预计时间：

```text
代码修改 + 单测修复：3 到 4 小时
全量验证 + fallout 修复 + IMPLEMENT_LOG：4 到 5 小时
```

## 11. 执行方案和进度记录

执行方式：

- 按 Gerrit 1 到 Gerrit 4 严格线性处理。
- 每完成一个 Gerrit 的代码修改，先跑对应分项测试。
- 4 个 Gerrit 都完成后，在栈顶执行全量 `mvn test`。
- 每个 Gerrit 的 Phase 6 整理项随对应业务修改一起做，不单独提交。
- 如需中途暂停，更新下表状态和最后一次测试结果，避免下次重复排查。

进度状态取值：

```text
未开始 / 修改中 / 测试中 / 已通过 / 阻塞
```

| 顺序 | Gerrit | 范围 | Phase 6 随手项 | 状态 | 分项测试 | 备注 |
|---:|---|---|---|---|---|---|
| 1 | M5-1 | ReasonCode 资格码、资格链路、`eval_closed` 懒关闭、`ProjectMapper.updateStatusToEvalClosed` | `EvaluationEligibilityService` 构造器注入 | 已通过 | `mvn -Dtest=EvaluationEligibilityServiceTest test` | 已合入 origin/master (388cd38)；19/19 通过 |
| 2 | M5-2 | 低分 20 字、标签白名单、拒绝未知标签 | `EvaluationValidationService` 构造器注入 | 已通过 | `mvn -Dtest=EvaluationValidationServiceTest test` | 40/40 通过；全量 126/126；本地已提交；Change 2379 closed，待重新推送 |
| 3 | M5-3 | `calculateCappedDeltaFromExisting`、单次快照、查询条件防回归 | `CreditCalculationServiceImpl` 构造器注入、AssertJ、测试命名 | 已通过 | `mvn "-Dtest=CreditRuleServiceImplTest,CreditCalculationServiceTest" test` | 43/43 通过；全量 127/127；已推送 Change 2721 |
| 4 | M5-4 | 批量异常检测、`AnomalyDetectionResult`、`EvaluationMapper` 查询、异常测试重写 | `AnomalyDetectionService` 构造器注入 | 已通过 | `mvn -Dtest=AnomalyDetectionServiceTest test` | 已修复：`detect(evaluatorId, projectId)` 批量语义 + 覆盖全部队友后整体判断；代码已对齐 V2.1 |
| 5 | Final | 全量测试与 `IMPLEMENT_LOG.md` 摘要 | 无 | 未开始 | `mvn test` | M5-4 完成后执行 |

预计执行时间：

```text
M5-1：30 到 45 分钟
M5-2：40 到 60 分钟
M5-3：40 到 60 分钟
M5-4：60 到 90 分钟
全量验证和 IMPLEMENT_LOG：30 到 60 分钟
总计：4 到 5 小时
```

### 11.1 M5-1 执行记录

执行时间：2026-05-11

实际修改：

- `ReasonCode`：互评资格相关对外码改为 V2.1 稳定字符串；评价人非成员统一走 `NOT_PROJECT_MEMBER`；目标人非成员走 `TARGET_NOT_PROJECT_MEMBER`。
- `ProjectMapper`：新增 `updateStatusToEvalClosed(id)` 条件更新，只允许 `ended -> eval_closed`。
- `EvaluationEligibilityService`：改为构造器注入；窗口过期时触发懒关闭；`validateSubmission` 返回 V2.1 资格码。
- `EvaluationEligibilityServiceTest`：M5-1 单测从 17 个扩展到 19 个，覆盖入口、待评价列表、提交前兜底三条路径上的过期懒关闭，并覆盖 `evalDeadline == null` 不懒关闭。

验证结果：

- `mvn -Dtest=EvaluationEligibilityServiceTest test`：19/19 通过。
- `mvn test`：119/119 通过。

提交状态：

- 原 M5-1 Gerrit Change `2228` 已 closed，不能继续向原 `Change-Id: I0b1240bf74f2ccc73660d8d48b01d3de670fc246` 上传 patch set。
- 当前正确做法是基于 `origin/master` 新建修复 Change。
- 当前本地修复分支：`m5-1-fix-v21`。
- 首次本地修复提交：`a23621180eb30f3aea38e466becb77f1b5155b11`。
- rebase 后当前提交：`388cd382e63e3d0549cceb2b46dab08d3e6d2120`。
- 当前新 `Change-Id`：`Ib6a19c30f7c00c7c72aff6d2a7703b7e3fd8643a`。
- 提交文件只有 4 个：
  - `src/main/java/com/teammatch/common/ReasonCode.java`
  - `src/main/java/com/teammatch/mapper/ProjectMapper.java`
  - `src/main/java/com/teammatch/service/EvaluationEligibilityService.java`
  - `src/test/java/com/teammatch/service/EvaluationEligibilityServiceTest.java`
- `docs/` 为未跟踪审查/计划文件，不属于 M5-1 Gerrit 提交内容。
- 2026-05-11 已基于最新 `origin/master` rebase，解决 `ReasonCode.java` 冲突；冲突解法是保留 master 中 M3 `M30xx` reasonCode，同时保留 M5-1 的 V2.1 互评资格 reasonCode。
- 若尚未看到 Gerrit `SUCCESS` 输出，推送命令为：`git push origin HEAD:refs/for/master`。

本轮踩坑记录：

- 曾在 `m5-dev` 栈顶 M5-4 上误执行 `git commit --amend`，导致 M5-1 修复被塞进 M5-4。已用 `b2c0c99` 恢复 M5-4，并重新 amend/push 为 `6d17c48` 覆盖 Gerrit 2627 的错误 patch set。
- 曾从 `backup/m5-wrong-amend-20260511` 执行 `git checkout ... -- src/main/java/com/` 和 `git restore ... -- src/main/java/com/`，导致 M3/M4/M6/M5-4 文件被整目录带入。后续禁止用目录级 restore/checkout 取回局部修复。
- PowerShell 中长路径被换行拆开后，后续片段会被当成独立命令执行。后续不要手动拆长命令。
- `git add src/main/java/` 会把不属于本轮的 M3/M4/M6/M5-4 文件一起暂存。后续优先用 `git add -u`，或每条命令只 add 一个明确文件。
- Gerrit 提示 `Merge Conflict` 时，不代表测试失败；需要 `git fetch origin` 后 `git rebase origin/master`，解决冲突并上传同一个 `Change-Id` 的新 patch set。
- 本轮 rebase 冲突只发生在 `ReasonCode.java`。不要简单选择 ours/theirs；应合并两边：保留 M3 新增枚举，替换 M5 资格枚举为 V2.1 稳定字符串。
- rebase 后提交 hash 会变化，`Change-Id` 不应变化。本轮从 `a236211` 变为 `388cd38`，`Change-Id` 保持 `Ib6a19c30f7c00c7c72aff6d2a7703b7e3fd8643a`。
- rebase 后必须重新跑分项测试和全量测试。本轮 rebase 后结果：`EvaluationEligibilityServiceTest` 19/19 通过，`mvn test` 119/119 通过。

下一轮开始前必须检查：

```powershell
git status --short --branch
git log -1 --pretty=fuller
git diff --name-status origin/master..HEAD
```

安全提交原则：

- 若只改已跟踪文件，用 `git add -u`，不要 `git add .`，不要 `git add docs/`。
- 若需要从备份分支取文件，每条命令只取一个完整文件路径，例如：

```powershell
git restore --source=<branch> -- src/main/java/com/teammatch/service/EvaluationValidationService.java
```

- 提交前必须确认 `git diff --name-status origin/master..HEAD` 只包含本轮目标文件。
- 如果 Gerrit 提示某个 Change closed，不再复用旧 `Change-Id`，改为基于 `origin/master` 新建修复 Change。

Gerrit reply 可用文本：

```text
Done.

Rebased on latest origin/master and resolved the ReasonCode.java conflict by keeping the newly added M3 reason codes while preserving the V2.1-aligned M5-1 reasonCode changes.

Scope remains limited to M5-1:
- ReasonCode alignment
- eval_closed lazy close
- constructor injection
- eligibility unit test updates

Verified:
- mvn -Dtest=EvaluationEligibilityServiceTest test: 19/19 passed
- mvn test: 119/119 passed
```

## 12. Gerrit 拆分建议

推荐拆成 4 个 Gerrit 变更，对应原 M5-1 到 M5-4 四个提交。

理由：

- 当前审查问题本来就是按 M5-1 / M5-2 / M5-3 / M5-4 分开记录。
- 每个变更的测试范围更清楚。
- Gerrit reviewer 可以逐个确认规则修复。
- 单个 patch set 失败时不影响其他已修好的部分。

建议拆分：

### Gerrit 1：M5-1 资格和 reasonCode 对齐

包含：

- `ReasonCode` 中资格类稳定码调整。
- M5-1 资格失败码调整。
- `eval_closed` 懒关闭。
- `ProjectMapper.updateStatusToEvalClosed` 条件更新。
- `EvaluationEligibilityService` 构造器注入。
- `EvaluationEligibilityServiceTest`。

风险：

- `ReasonCode` 是公共文件，但本次只改 M5 使用的枚举值。

验证：

```bash
mvn -Dtest=EvaluationEligibilityServiceTest test
```

### Gerrit 2：M5-2 内容校验

包含：

- 低分说明不少于 20 字。
- 正负向标签白名单。
- 拒绝未知标签。
- `EvaluationValidationService` 构造器注入。
- `EvaluationValidationServiceTest`。

验证：

```bash
mvn -Dtest=EvaluationValidationServiceTest test
```

### Gerrit 3：M5-3 信誉计算一致性

包含：

- 新增 `calculateCappedDeltaFromExisting`。
- M5 计算编排改为单次 `existingProjectDelta` 快照。
- 查询条件防回归测试。
- 测试命名整理。
- `CreditCalculationServiceImpl` 构造器注入。
- AssertJ 断言统一。

验证：

```bash
mvn "-Dtest=CreditRuleServiceImplTest,CreditCalculationServiceTest" test
```

### Gerrit 4：M5-4 异常检测批量语义

包含：

- `AnomalyDetectionService` 重写为 `detect(evaluatorId, projectId)`。
- `AnomalyDetectionResult` 重写。
- `EvaluationMapper` 新增查询方法。
- `AnomalyDetectionService` 构造器注入。
- `AnomalyDetectionServiceTest` 重写。

验证：

```bash
mvn -Dtest=AnomalyDetectionServiceTest test
```

### 最终验证

4 个变更都准备好后，在栈顶执行：

```bash
mvn test
```

如果 Gerrit 不方便堆叠 4 个变更，也可以合成 1 个变更，但不推荐。合成一个变更会让 `ReasonCode`、资格、内容、信誉、异常检测全混在一起，review 负担更大。

结论：

```text
优先 4 个 Gerrit 变更。
如果时间很紧，至少拆成 2 个：
1. M5-1 + M5-2 契约与校验
2. M5-3 + M5-4 计算与异常检测
```

## 13. IMPLEMENT_LOG 追加内容

完成后追加摘要，不粘贴完整日志。

建议记录：

```text
日期：
范围：M5-1 到 M5-4 规则层修复
变更：
- M5-1 reasonCode 对齐与 eval_closed 懒关闭
- M5-2 低分 20 字说明与标签白名单
- M5-3 单项目封顶计算改为单次快照
- M5-4 异常检测改为覆盖全部队友后的批量语义
测试：
- mvn -Dtest=EvaluationEligibilityServiceTest test
- mvn -Dtest=EvaluationValidationServiceTest test
- mvn "-Dtest=CreditRuleServiceImplTest,CreditCalculationServiceTest" test
- mvn -Dtest=AnomalyDetectionServiceTest test
- mvn test
 后续：
- 实现 M5 提交编排服务，同一事务内完成 evaluation、credit_change、user.credit_score 与异常挂起联动
```

## 14. M5-1 到 M5-3 返工根因分析

### 一句话

不是代码能力问题。是 V2.1 条款没在写代码前变成 checklist，测试跟了实现走，Gerrit 兜底成了第一个真正对照 V2.1 的环节。

### 根因

| 编号 | 根因 | 出在哪个模块 |
|---|---|---|
| R1 | V2.1 没有前置拆成 checklist——开发凭记忆，遗漏离散条款（≥20字、标签白名单、懒关闭触发点） | M5-1, M5-2 |
| R2 | 测试验证的是实现，不是 V2.1——代码写错一句，测试就跟着错一句；17/33/25 全绿不代表合规 | 全部 |
| R3 | 审查发生在 Gerrit push 之后——Codex + opencode 都发现所有问题，但发现时代码已经在 Gerrit 上了 | 全部 |
| R4 | 跨章节条款漏读——eval_closed 在 9.1（事务/懒检查边界），标签白名单在 6.9（表格后面的文字块），不标小节号容易被跳过 | M5-1, M5-2 |
| R5 | 公共契约没有独立冻结——ReasonCode 在各模块独立命名，M5-1 叫 `DUPLICATE_EVALUATION` 而 V2.1 叫 `ALREADY_EVALUATED`，没有全项目 grep 对齐 | M5-1, M5-3 |
| R6 | 链路被拆碎——M5-1~M5-4 是同一互评链路，各自独立写时没考虑入口、状态迁移、检测时机的前后依赖。单独看都对，串起来漏了 eval_closed | M5-1 |
| R7 | Gerrit 操作纪律松散——已关闭 change 仍复用 Change-Id、误 amend 到 M5-4、stage docs/、命令行分行写错——不产生业务 bug 但增加返工轮次 | 全部 |

### M5 必读 V2.1 条款

```
4.3/4.4  项目状态机 + 懒检查边界
6.3      互评资格
6.4      四维结构
6.5      信誉分映射 + 单项目封顶
6.6      异常检测（P0 冻结）
6.8      关键规则（≥20字、标签要求）
6.9      P0 已定稿标签
8.4      reasonCode 统一枚举
9.1      事务边界
```

### 后续 M5 开发约束

| 编号 | 约束 | 执行时机 |
|---|---|---|
| C1 | V2.1 条款前置 checklist——每个 change 列出：涉及 V2.1 条款行号、代码文件、测试点、明确不做什么 | 写代码前 |
| C2 | 测试对 V2.1 不对实现——测试名体现 V2.1 规则，断言引用 V2.1 规格值，不引用当前实现值 | 写测试时 |
| C3 | 公共契约独立审查——涉及 ReasonCode/状态枚举/credit_change.change_type/事务边界时，`rg` 全项目引用，提交说明单独写影响范围 | 涉及时 |
| C4 | Push 前四步——① `git status` ② `git diff --name-status origin/master..HEAD` ③ 对应单测 + `mvn test` ④ `git log -1` 确认只有一个 Change-Id | Push 前 |
| C5 | Gerrit 规则——已关闭 change 不复用 Change-Id；业务提交不夹带 docs/ 审查文件；不在错误 HEAD 上 commit --amend | 提交时 |
| C6 | M5-4 前置——等 M5-1/M5-2/M5-3 合入 origin/master 并同步后再开 M5-4，按 C1 建 checklist 再编码 | M5-4 开始前 |

