# M5-9 实现审查报告

审查日期：2025-07-17
审查对象：M5-9 管理端评价复核 Controller 实现（4 个新增文件）
审查依据：`docs/m5/M5-9_IMPLEMENT_PLAN_V1.1.md`

---

## A. 总结论：通过

4 个新增文件全部到位，零修改已有文件。Controller 严格遵循 M6 `PenaltyAdminController` 风格，状态机/action 校验不下沉，测试覆盖 14 个场景且不做不稳定断言。

---

## B. P0 必改问题

**无。**

---

## C. P1 建议问题

### P1-1：B1（`GET /pending`）缺少无效/过期 token → M3000 的显式测试用例

V1.1 测试计划 A1 要求"Mock `isAdmin` throw `AuthenticationException`，经 `AuthExceptionHandler` 拦截 → M3000"。当前 B1 鉴权测试仅覆盖了非管理员（M3009），未覆盖无效 token。`AuthExceptionHandler` 虽提供全局兜底，但 Controller 层缺少回归保护——未来若有人把 B1 的 `@RequestHeader` 改成 `required = false` 并自行处理 token，可能漏掉这个路径。

**修复：** 补一个测试用例，mock `isAdmin` 抛出 `AuthenticationException`。

```java
@Test
@DisplayName("B1: 无效 token 返回 M3000（经 AuthExceptionHandler）")
void getPendingEvaluations_invalidToken() throws Exception {
    when(authService.isAdmin("Bearer invalid")).thenThrow(
            new AuthenticationException(ReasonCode.UNAUTHORIZED));

    mockMvc.perform(get("/admin/evaluations/pending")
                    .header("Authorization", "Bearer invalid"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.code").value(ReasonCode.UNAUTHORIZED.getCode()));

    verify(evaluationMapper, never()).selectList(any());
}
```

需补 `import com.teammatch.exception.AuthenticationException;`。

### P1-2：B1 空列表测试断言 `$.data.length()` 依赖非标准 JSONPath 函数

`$.data.length()` 在部分 JSONPath 实现中不被支持（Jayway JSONPath 支持但非标准）。若未来换 JSONPath 实现可能断裂。

**修复：** 将 C1 测试中 `.andExpect(jsonPath("$.data.length()").value(0))` 改为 `.andExpect(jsonPath("$.data").isEmpty())`。

---

## D. 是否可以交给 Codex 终审

**可以。** P1 问题均为低风险补强项，不影响施工和合入。

---

## E. 最小修复建议

两个 P1 问题的具体 diff 见 C 节，合计新增约 10 行测试代码 + 1 行 import + 1 行断言修改。

---

## 审查清单逐项结果

| # | 审查项 | 结果 | 证据 |
|---|---|---|---|
| 1 | 只新增 4 个文件 | ✅ | `git status` 仅见 4 个预期 untracked 文件 |
| 2 | 未修改已有 Service / Mapper / Entity / ReasonCode / schema.sql | ✅ | `git diff --stat HEAD` 无输出 |
| 3 | Controller 在 `com.teammatch.m6.controller` | ✅ | `EvaluationAdminController.java:1` |
| 4 | DTO/VO 在 `com.teammatch.m6.dto` | ✅ | `EvaluationReviewRequest.java:1`、`PendingEvaluationVO.java:1` |
| 5 | GET 只查 `pending_review` | ✅ | `Controller:42` — `wrapper.eq(Evaluation::getStatus, Evaluation.STATUS_PENDING_REVIEW)` |
| 6 | GET 仅可选 `projectId`，无无关过滤 | ✅ | `Controller:43-45` — 仅 `projectId` 一个可选参数 |
| 7 | POST 仅组装 `EvaluationReviewCommand` + 调 Service | ✅ | `Controller:65-76` |
| 8 | Controller 不复刻状态机 | ✅ | 无 `switch`/`if` 分支判断 `action` 值 |
| 9 | action 合法性交给 Service | ✅ | `Controller:72` — 直接 `setAction(request.getAction())`，不校验 |
| 10 | reviewerId 只从 token 注入 | ✅ | `Controller:68-70` — `adminId = getUserIdFromToken(token)` → `setReviewerId(adminId)` |
| 11 | 请求体无 reviewerId | ✅ | `EvaluationReviewRequest.java` — 仅 `action` + `reviewNote` 两个字段 |
| 12 | 未直接改 evaluation / credit_change / credit_score | ✅ | 无相关代码 |
| 13 | 未实现 report / penalty / Jenkins / 部署 | ✅ | 无相关代码 |
| 14 | 测试覆盖全部要求场景 | ✅ | 见下方测试覆盖矩阵 |
| 15 | 未强行断言 LambdaQueryWrapper 内部 SQL | ✅ | `ArgumentCaptor` 仅用于 `evaluationReviewService.review()` 的 Command 参数 |
| 16 | `Result.success()` 码断言与实际一致 | ✅ | `"00000"` = `ReasonCode.SUCCESS.getCode()` |

---

## 测试覆盖矩阵

| 测试用例 | 对应要求 | 结果 |
|---|---|---|
| `getPendingEvaluations_nonAdmin_returnsAdminRequired` | M3009 (B1) | ✅ |
| `reviewEvaluation_nonAdmin_returnsAdminRequired` | M3009 (B2) | ✅ |
| `reviewEvaluation_nullAdminId_returnsUnauthorized` | M3000 (B2) | ✅ |
| `getPendingEvaluations_emptyList` | pending 空列表 | ✅ |
| `getPendingEvaluations_normalList` | pending 正常列表 | ✅ |
| `getPendingEvaluations_withProjectId` | projectId 过滤 | ✅ |
| `reviewEvaluation_approve_success` | approve | ✅ |
| `reviewEvaluation_void_success` | void | ✅ |
| `reviewEvaluation_keepNoCredit_success` | keep_no_credit | ✅ |
| `reviewEvaluation_notFound` | M5005 | ✅ |
| `reviewEvaluation_invalidAction` | M5007 | ✅ |
| `reviewEvaluation_statusConflict` | M1003 | ✅ |
| `reviewEvaluation_noteTooLong` | M5008 | ✅ |
| `reviewEvaluation_creditChangeNotFound` | M5006 | ✅ |
| `reviewEvaluation_reviewerIdFromToken` | reviewerId 从 token 注入 | ✅ |
