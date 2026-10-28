# M5-1 互评资格判断提交审查问题记录

审查对象：

- 提交标题：`feat(m5): 添加互评资格判断`
- 提交哈希：`857c6fc840c1817a5bbe86d9e035f46b0ad2b5b5`
- Change-Id：`I0b1240bf74f2ccc73660d8d48b01d3de670fc246`
- 审查日期：2026-05-11
- 审查范围：仅检查 M5-1A 项目级互评入口判断、M5-1B 待评价成员列表判断、M5-1C 提交前资格兜底校验。

## 结论

该提交自己的单元测试可以通过，但按 `docs/design/system-design-v2.1.md` 与当前 M5 规则严格验收，不建议放行。

主要问题不是编译或单测失败，而是测试把不符合冻结契约的行为固化成了预期。

## P0 阻断问题

### 1. `reasonCode` 输出不符合冻结契约

问题位置：

- `src/main/java/com/teammatch/common/ReasonCode.java`
- `src/main/java/com/teammatch/service/EvaluationEligibilityService.java`
- `src/test/java/com/teammatch/service/EvaluationEligibilityServiceTest.java`

当前实现使用：

- `M5001`
- `M5101`
- `M5102`
- `M5103`
- `M5104`
- `SELF_EVALUATION_NOT_ALLOWED`
- `DUPLICATE_EVALUATION`
- `TARGET_NOT_MEMBER`
- `EVALUATOR_NOT_MEMBER`

冻结规则要求互评资格相关 `reasonCode` 稳定为：

- `NOT_PROJECT_MEMBER`
- `TARGET_NOT_PROJECT_MEMBER`
- `PROJECT_NOT_ENDED`
- `EVAL_WINDOW_CLOSED`
- `ALREADY_EVALUATED`
- `SELF_EVALUATION`

影响：

- 前端或接口联调按稳定 `reasonCode` 判断时会失败。
- 当前测试断言的是错误口径，不能作为通过证据。

修复要求：

- 调整 `ReasonCode` 枚举或响应字段，使接口返回冻结文档要求的稳定 `reasonCode`。
- 将重复评价从 `DUPLICATE_EVALUATION` 对齐为 `ALREADY_EVALUATED`。
- 将自评从 `SELF_EVALUATION_NOT_ALLOWED` 对齐为 `SELF_EVALUATION`。
- 将目标成员不存在从 `TARGET_NOT_MEMBER` 对齐为 `TARGET_NOT_PROJECT_MEMBER`。
- 不建议继续用 `M5001/M5101` 作为业务判断主契约。

### 2. 互评窗口过期没有触发项目懒关闭

问题位置：

- `src/main/java/com/teammatch/service/EvaluationEligibilityService.java`

当前行为：

- `checkProjectEligibility` 检查到 `now.isAfter(evalDeadline)` 后直接返回 `EVAL_WINDOW_CLOSED`。
- 没有把 `project.status` 从 `ended` 更新为 `eval_closed`。

冻结规则要求：

- 互评窗口固定为项目结束后 7 日。
- P0 采用懒检查关闭项目互评窗口。
- 懒检查触发点包括项目详情、互评资格检查、提交互评等入口。

影响：

- 项目状态可能长期停留在 `ended`。
- 后续项目详情、列表、治理、评价展示会读到不一致状态。

修复要求：

- 在互评资格检查与提交前兜底校验路径中，发现窗口过期时应执行 `ended -> eval_closed`。
- 该更新应有明确 mapper/service 方法，并补单元测试验证更新被调用。
- 若后续进入真实提交链路，需要考虑事务边界。

## P1 测试缺口

当前 17 个测试覆盖了 M5-1 三个方法的基础分支，但缺少以下关键断言：

- `reasonCode` 字符串必须是冻结契约枚举值，而不是 `M5001/M5101` 数字码。
- 互评窗口过期时必须触发 `eval_closed` 状态更新。
- 互评资格接口层应返回 `eligible=false + reasonCode` 的联调形态。
- 提交前兜底校验中，重复评价应返回 `ALREADY_EVALUATED`。
- 提交前兜底校验中，自评应返回 `SELF_EVALUATION`。
- 提交前兜底校验中，目标用户非项目成员应返回 `TARGET_NOT_PROJECT_MEMBER`。

## 已验证测试

在临时 worktree 固定到 `857c6fc840c1817a5bbe86d9e035f46b0ad2b5b5` 后执行：

```bash
mvn -Dtest=EvaluationEligibilityServiceTest test
```

结果：

```text
Tests run: 17, Failures: 0, Errors: 0, Skipped: 0
BUILD SUCCESS
```

继续执行：

```bash
mvn test
```

结果：

```text
Tests run: 17, Failures: 0, Errors: 0, Skipped: 0
BUILD SUCCESS
```

说明：

- 编译和该提交内单元测试通过。
- 通过结果不能覆盖上述业务契约问题。

## 建议修复顺序

1. 先统一 `reasonCode` 契约，避免后续 M5-2 / M5-3 / M5-4 继续沿用错误枚举。
2. 补 `ended -> eval_closed` 懒关闭逻辑。
3. 修改现有 17 个测试中错误的断言。
4. 新增窗口过期触发状态更新测试。
5. 再继续检查 M5-2 到 M5-4，避免后续提交建立在错误基础上。
