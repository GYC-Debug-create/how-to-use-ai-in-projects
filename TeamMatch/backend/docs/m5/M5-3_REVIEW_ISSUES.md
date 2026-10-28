# M5-3 信誉分映射与单项目封顶提交审查问题记录

审查对象：

- 提交标题：`feat(m5): 实现 M5-3 信誉分映射与单项目封顶`
- 提交哈希：`f65d2efb830e8b7895dac1b7259d0765998798a3`
- Change-Id：`Id8656d3d0bb54fa48d26d1c62ac1eb69aa2d2830`
- 审查日期：2026-05-11
- 审查范围：仅检查 M5-3 互评信誉分八区间映射、单项目 ±10 封顶、`CreditCalculationResult`、`CreditCalculationService` 和相关单元测试。

## 结论

限定只看 M5-3 自身，本提交未发现阻断八区间映射或单项目 ±10 封顶的 P0 业务错误。

`change_type='evaluation'`、`effective=true`、`user_id + project_id` 的累计查询方向与 V2.1 和 M5 规则一致。新增 33 个 M5-3 相关单元测试通过，全量测试也通过。

但本提交仍有两个需要在继续 M5-4 或接入提交链路前处理的 P1 风险，以及一个 P2 测试质量问题。M5-1 / M5-2 已记录的 P0 问题不在本文件重复展开，但如果按提交栈整体验收，它们仍会影响 M5 后续链路。

## P1 风险

### 1. 编排层重复读取项目累计值，结果字段可能不是同一快照

问题位置：

- `src/main/java/com/teammatch/service/impl/CreditCalculationServiceImpl.java`
- `src/main/java/com/teammatch/m4/service/impl/CreditRuleServiceImpl.java`

当前流程：

1. `CreditCalculationServiceImpl.calculate` 先调用 `getExistingProjectDelta(...)` 得到 `existingProjectDelta`。
2. 随后调用 `calculateCappedDelta(...)`。
3. `calculateCappedDelta(...)` 内部再次调用 `getExistingProjectDelta(...)`。
4. `projectTotalAfterApplied` 使用第一次读取的 `existingProjectDelta + cappedDelta`。

风险：

- 若后续接入提交事务时两个读取之间出现并发写入，`cappedDelta` 可能基于第二次累计值计算，而 `projectTotalAfterApplied` 却基于第一次累计值展示，结构化结果内部不再同源。
- 即使当前 M5-3 不落库，后续 M5-4 / 提交互评链路会把该结果作为写 `credit_change` 的依据，最好现在收紧。

建议：

- 将封顶计算拆成纯函数，例如 `calculateCappedDeltaFromExisting(int existingProjectDelta, int proposedDelta)`。
- 编排层只读取一次 `existingProjectDelta`，再用同一个快照计算 `cappedDelta` 和 `projectTotalAfterApplied`。
- 如果后续提交链路落库，应在提交事务内完成读取、计算和写入，并配合唯一约束 / 行锁策略。

### 2. M5 信誉计算逻辑继续放在 `com.teammatch.m4.service` 下，模块边界不清

问题位置：

- `src/main/java/com/teammatch/m4/service/CreditRuleService.java`
- `src/main/java/com/teammatch/m4/service/impl/CreditRuleServiceImpl.java`
- `src/main/java/com/teammatch/service/impl/CreditCalculationServiceImpl.java`

当前现象：

- M5 的互评信誉分映射、evaluation 类 `credit_change` 查询和封顶逻辑落在 M4 包下。
- M5 的 `CreditCalculationServiceImpl` 反向依赖 M4 的 `CreditRuleService`。

风险：

- 后续 M4 退出扣分、M5 评价计分、M6 处罚 / 申诉恢复都会围绕 `credit_change` 发展，继续把 M5 规则放在 M4 包下会让职责边界越来越混乱。
- 课程协作时容易误判文件归属，导致 M4/M5/M6 提交相互踩改。

建议：

- 短期至少在 M5 文档 / IMPLEMENT_LOG 中说明这是复用旧类的临时方案。
- 更稳妥的做法是把互评信誉分映射与项目封顶逻辑移动到 M5 或 common credit 包，由 M4/M6 通过明确接口复用。

## P2 测试质量问题

### 1. 查询条件没有被单测锁住

问题位置：

- `src/test/java/com/teammatch/m4/service/CreditRuleServiceImplTest.java`

当前测试对 mapper 使用：

```java
when(creditChangeMapper.selectList(any(LambdaQueryWrapper.class)))
```

风险：

- 提交说明中特别强调“修正查询条件：使用 change_type 而非 source_type”，但测试没有验证 wrapper 中确实包含 `change_type='evaluation'`、`effective=true`、`user_id`、`project_id`。
- 未来如果有人把查询改回 `source_type`，这些测试仍可能通过。

建议：

- 至少补一个针对 `getExistingProjectDelta` 的测试，捕获传入 mapper 的 `LambdaQueryWrapper`，或改成自定义 mapper 方法并对方法参数做精确验证。

### 2. 个别边界测试命名与实际输入不一致

问题位置：

- `src/test/java/com/teammatch/m4/service/CreditRuleServiceImplTest.java`

当前示例：

- `mapAverageScoreToDelta_upper_bound_4_49` 实际传入 `4.25`
- `mapAverageScoreToDelta_lower_bound_2_99` 实际传入 `2.75`

影响：

- 不影响当前结果正确性。
- 但测试名会误导 reviewer，以为已经覆盖了靠近区间上界的浮点输入。

建议：

- 改名为 `avg_4_25`、`avg_2_75`，或真的补 `4.49`、`2.99` 这类直接调用 `mapAverageScoreToDelta(double)` 的边界测试。

## 已验证测试

在临时 worktree 固定到 `f65d2efb830e8b7895dac1b7259d0765998798a3` 后执行：

```bash
mvn "-Dtest=CreditRuleServiceImplTest,CreditCalculationServiceTest" test
```

结果：

```text
Tests run: 33, Failures: 0, Errors: 0, Skipped: 0
BUILD SUCCESS
```

继续执行：

```bash
mvn test
```

结果：

```text
Tests run: 100, Failures: 0, Errors: 0, Skipped: 0
BUILD SUCCESS
```

说明：

- M5-3 自身的计算测试通过。
- 全量测试通过。
- 上述 P1/P2 属于后续集成一致性和测试防回归问题，不是当前单测失败。

## 建议处理顺序

1. 保留当前八区间映射和 ±10 截断算法。
2. 优先把封顶计算改成基于一次 `existingProjectDelta` 快照的纯函数，避免结果字段不一致。
3. 决定 `CreditRuleService` 的最终模块归属，避免 M5 继续扩大 M4 包职责。
4. 补强 `getExistingProjectDelta` 查询条件测试。
5. 继续检查 M5-4 时，重点看它是否依赖 M5-1 / M5-2 / M5-3 的未修问题。
