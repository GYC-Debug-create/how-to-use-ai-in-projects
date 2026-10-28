# M5-2 互评提交内容基础校验提交审查问题记录

审查对象：

- 提交标题：`feat(m5): 添加互评提交内容基础校验`
- 提交哈希：`48906df35b4b15e8ea03d98cf39575d644c4c7c3`
- Change-Id：`Ie98fc2e346c62791d34b4101fb54218422c3f61a`
- 审查日期：2026-05-11
- 审查范围：仅检查 M5-2 互评提交 DTO、评分范围、评论违规词、高低分标签与低分评论校验，以及 `EvaluationValidationServiceTest`。

## 结论

该提交编译和测试均可通过，但按 `docs/design/system-design-v2.1.md` 与 `docs/m5/M5_CONTEXT.md` 严格验收，不建议放行。

主要问题是两个冻结业务约束没有被实现，现有测试反而把过宽的校验口径固化为通过场景。

## P0 阻断问题

### 1. 低分说明未校验不少于 20 字

问题位置：

- `src/main/java/com/teammatch/service/EvaluationValidationService.java`
- `src/test/java/com/teammatch/service/EvaluationValidationServiceTest.java`

冻结规则要求：

- 任一维度为 1 / 2 分时，必须附负向标签和说明。
- 关键规则进一步明确：说明需要 `≥ 20 字`。

当前实现：

- `checkLowScoreComment` 只判断 `comment == null || comment.trim().isEmpty()`。
- 只要非空就通过，没有检查长度。

当前测试中被错误放行的示例：

- `需要改进`
- `需要提升技术能力`
- `责任心需要提升`

这些说明都明显不足 20 字，但测试断言为通过。

影响：

- 低分评价可以只写极短理由，削弱互评可信度。
- 后续 M5-3 信誉分计算与异常检测会基于不合规评价继续沉淀。

修复要求：

- 对低分场景使用 `comment.trim().length() >= 20` 作为最低要求。
- 补充低于 20 字失败、刚好 20 字通过、超过 20 字通过的测试。
- 现有低分通过用例需要改成不少于 20 字的说明。

### 2. 高低分标签只检查非空，未限定为 P0 定稿标签

问题位置：

- `src/main/java/com/teammatch/service/EvaluationValidationService.java`
- `src/test/java/com/teammatch/service/EvaluationValidationServiceTest.java`

冻结规则给出的 P0 已定稿标签：

- 正向标签：`沟通积极`、`按时交付`、`技术可靠`、`责任心强`
- 负向标签：`沟通差`、`延期`、`质量低`、`失联`

当前实现：

- 高分只要求 `positiveTags` 非空。
- 低分只要求 `negativeTags` 非空。
- 没有校验标签是否属于定稿集合。

当前测试中被错误放行的示例：

- 正向标签：`技术强`、`非常优秀`
- 负向标签：`沟通不畅`、`任务拖延`、`技术不足`、`责任心不足`

这些都不在 V2.1 的 P0 定稿标签集合中。

影响：

- M5 会写入或传递非契约标签，前端展示、统计、复核、后续治理都可能产生不一致。
- P0 标签字典的冻结意义被绕开。

修复要求：

- 在 M5-2 校验层维护或注入定稿标签白名单。
- 高分标签必须至少包含一个合法正向标签，且建议拒绝未知正向标签。
- 低分标签必须至少包含一个合法负向标签，且建议拒绝未知负向标签。
- 补充未知正向标签失败、未知负向标签失败、定稿标签通过测试。

## P1 风险

### 内容校验错误码尚未进入 V2.1 reasonCode 表

问题位置：

- `src/main/java/com/teammatch/common/ReasonCode.java`

当前新增：

- `SCORE_FIELD_MISSING`
- `SCORE_OUT_OF_RANGE`
- `COMMENT_CONTAINS_VIOLATION`
- `HIGH_SCORE_MISSING_POSITIVE_TAG`
- `LOW_SCORE_MISSING_NEGATIVE_TAG`
- `LOW_SCORE_COMMENT_EMPTY`

这些内容校验错误码在当前 V2.1 的 `reasonCode` 表中没有明确定稿。若后续接口直接对外暴露，需要先补齐接口契约或确认这些错误码属于内部实现返回。

## 已验证测试

在临时 worktree 固定到 `48906df35b4b15e8ea03d98cf39575d644c4c7c3` 后执行：

```bash
mvn -Dtest=EvaluationValidationServiceTest test
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
Tests run: 67, Failures: 0, Errors: 0, Skipped: 0
BUILD SUCCESS
```

说明：

- 编译和现有测试通过。
- 通过结果不能覆盖上述业务契约问题，因为现有测试缺少 20 字说明和定稿标签白名单断言。

## 建议修复顺序

1. 先补低分说明 `≥ 20 字` 校验与边界测试。
2. 再补 P0 定稿正向 / 负向标签白名单校验。
3. 将现有测试中非定稿标签替换为 V2.1 定稿标签。
4. 与 M5-1 的 `reasonCode` 契约问题一起统一整理，避免 M5-3 / M5-4 继续依赖错误响应口径。
