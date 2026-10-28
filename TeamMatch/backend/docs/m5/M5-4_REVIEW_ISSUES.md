# M5-4 异常评价判断提交审查问题记录

审查对象：

- 提交标题：`feat(m5): 添加异常评价判断`
- 提交哈希：`b2c0c99a1db42e62b9f11b51df539b51ed601a3e`
- Change-Id：`Ib182c3a8339a82c71d3cc1c9602917d6f6756db4`
- 审查日期：2026-05-11
- 审查范围：仅检查 M5-4 异常评价判断、`AnomalyDetectionResult` 和 `AnomalyDetectionServiceTest`。

## 结论

该提交编译和测试均可通过，但按 `docs/design/system-design-v2.1.md` 与 `docs/m5/M5_CONTEXT.md` 严格验收，不建议放行。

本提交最大问题是将 V2.1 冻结的“覆盖全部队友后的跨被评价者整体判断”改成了“单条评价评分模式判断”。这不是实现细节差异，而是业务规则口径变更。

## P0 阻断问题

### 1. 异常评价规则口径错误：单条评价不应直接判定 `pending_review`

问题位置：

- `src/main/java/com/teammatch/service/AnomalyDetectionService.java`
- `src/main/java/com/teammatch/dto/AnomalyDetectionResult.java`
- `src/test/java/com/teammatch/service/AnomalyDetectionServiceTest.java`

V2.1 冻结规则：

- P0 只保留两条基础规则。
- 检测时机：评价者覆盖全部队友后同步触发。
- 规则 A：评价者对同项目所有被评价者的 4 维度均分 `≤ 2`，进入 `pending_review`。
- 规则 B：评价者对同项目所有被评价者的 4 维度均分 `= 5`，进入 `pending_review`。
- 命中后，将该评价者在该项目中的全部评价批量标记为 `pending_review`。

当前实现：

- 对单条评价直接分类为 `normal` / `pending_review`。
- 规则 A 被实现为“四个维度中有 ≥2 个维度 ≤2”。
- 规则 B 被实现为“单条评价四个维度全部 =5”。

这两条都不是 V2.1 的冻结规则。

影响：

- 一个用户只评价了一个队友时，单条全低分或全满分就会被判为 `pending_review`，但 V2.1 要求只有覆盖全部队友后才触发。
- “两个维度低分”会被错误挂起，但 V2.1 的全低分规则看的是“对所有被评价者的 4 维度均分 ≤ 2”，不是单条评价中低分维度个数。
- 异常命中后的动作应批量影响该评价者在该项目中的全部评价，而不是给单条评价返回一个最终状态。

### 2. 测试把错误规则固化为通过预期

问题位置：

- `src/test/java/com/teammatch/service/AnomalyDetectionServiceTest.java`

当前测试中错误通过的场景：

- `1/1/4/4` 被断言为 `pending_review`
- `2/2/3/3` 被断言为 `pending_review`
- `2/2/5/5` 被断言为 `pending_review`
- 单条 `5/5/5/5` 被断言为 `pending_review`

这些测试没有覆盖 M5 测试计划要求的关键场景：

- 未覆盖全部队友时不检测。
- 覆盖全部队友且全低分后，全部评价进入 `pending_review`。
- 覆盖全部队友且全满分后，全部评价进入 `pending_review`。

影响：

- 当前 15 个测试通过不能作为 M5-4 合规证据。
- 后续如果提交互评链路接入该服务，会把错误异常状态写入评价状态和信誉流水有效性流程。

## 修复建议

建议不要继续沿用当前“单条评价状态分类”作为最终异常判断服务。

更符合 V2.1 的拆法：

1. `AnomalyDetectionService` 输入应包含评价者、项目、已提交评价集合、项目应评价队友数，或通过 mapper 查询这些信息。
2. 先判断评价者是否已覆盖全部队友；未覆盖时不触发异常，当前评价仍按正常提交链路继续。
3. 覆盖后再计算该评价者对所有被评价者的四维均分集合。
4. 全部评价均分 `≤ 2` 时，批量标记该评价者在该项目中的全部评价为 `pending_review`。
5. 全部评价均分 `= 5` 时，批量标记该评价者在该项目中的全部评价为 `pending_review`。
6. 命中后联动 `credit_change.effective=0` 和 `user.credit_score` 回退，应放到后续提交 / 异常挂起事务中处理。

如果当前阶段只想做纯计算辅助函数，也应调整语义：

- 不返回 `normal` / `pending_review` 这种最终评价状态。
- 只返回单条评价的均分和分类辅助信息。
- 明确不能由单条辅助结果直接决定 `evaluation.status`。

## 已验证测试

在临时 worktree 固定到 `b2c0c99a1db42e62b9f11b51df539b51ed601a3e` 后执行：

```bash
mvn -Dtest=AnomalyDetectionServiceTest test
```

结果：

```text
Tests run: 15, Failures: 0, Errors: 0, Skipped: 0
BUILD SUCCESS
```

继续执行：

```bash
mvn test
```

结果：

```text
Tests run: 132, Failures: 0, Errors: 0, Skipped: 0
BUILD SUCCESS
```

说明：

- 编译和现有测试通过。
- 通过结果不能覆盖上述业务契约问题，因为现有测试验证的是错误的单条异常口径。

## 建议处理顺序

1. 先废弃或降级当前单条 `pending_review` 分类语义。
2. 按“覆盖全部队友后整体判断”重写异常检测服务接口。
3. 补齐“未覆盖全部队友不检测”的测试。
4. 补齐“覆盖全部队友且全低分 / 全满分时批量挂起”的测试。
5. 与 M5-3 提交链路集成时，确保异常命中后 `credit_change.effective=0` 与信誉缓存回退在同一事务中处理。
