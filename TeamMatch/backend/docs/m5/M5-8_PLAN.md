# M5-8 用户端互评/信誉接口层 — 实施规划（修正版 V1.2）

**版本：** V1.3（Codex 审查修正版，第二轮）  
**适用范围：** TeamMatch 后端 M5-8A 用户端 API/Controller 层（仅互评，不含信誉查询）  
**前置依赖：** M5-1~M5-7 Service 层已全部完成（资格校验、内容校验、信誉计算、异常检测、提交编排、评价复核、申诉恢复）  
**最高业务依据：** `docs/design/system-design-v2.1.md`

---

## A. M5-8A 任务边界

M5-8A 是 M5 模块用户端互评 API 层，只做互评相关接口，不做信誉查询。

| 在范围内 | 不在范围内 |
|---|---|
| 5 个用户端 Controller 端点（项目级资格 + 目标级资格 + 可评价成员 + 提交互评 + 查看收到的互评） | 管理端接口（评价复核、申诉处理） |
| `AuthService.getUserIdFromToken(token)` 鉴权注入 | 业务状态机重写 |
| DTO 转换 + 统一响应 `Result<T>` | `schema.sql` 修改 |
| 前台互评匿名展示（不暴露 evaluatorId） | 绕开 `credit_change` 直接改 `user.credit_score` |
| Controller 层单元测试 (MockMvc) | report / penalty / Jenkins / 部署 |
| `EvaluationMapper` 最小只读查询扩展（妥协方案） | 新增业务 Service / 修改已有 Service |
| `SubmitEvaluationRequest` — 不含 evaluatorId 的新请求 DTO | 修改已有 `EvaluationSubmitDTO` / `EvaluationSubmitService` |
| — | **信誉查询接口（B6/B7）— 拆分到 M5-8B** |
| — | **CreditController / CreditScoreVO / CreditChangeVO / CreditChangePageVO** |

**Controller 层职责边界（硬约束）：**

```
AuthService.getUserIdFromToken(token) → userId
  → 参数接收 → 入参校验
  → 调用已有 Service → DTO/VO 转换 → 统一响应 Result<T>
```

所有业务判断（状态机、资格校验、信誉计算、事务边界）继续留在已有 Service 中，Controller 不重写、不扩张。

---

## B. 推荐接口列表

### B1. 检查互评资格 — 项目级

判断当前用户能否进入该项目的互评页面（不判断具体评价谁）。

| 项 | 内容 |
|---|---|
| **Method** | `GET` |
| **Path** | `/api/m5/projects/{projectId}/evaluation-eligibility` |
| **Auth** | `Authorization: Bearer <token>` |
| **请求参数** | Path: `projectId` (Long) |
| **成功响应** | `{ "code": "00000", "message": "成功", "data": true }` |
| **失败响应示例** | `{ "code": "NOT_PROJECT_MEMBER", "message": "当前用户不是该项目成员", "data": null }` |
| **复用 Service** | `EvaluationEligibilityService.checkProjectEligibility(userId, projectId)` |

**可能的 reasonCode：**

- `NOT_PROJECT_MEMBER` — 当前用户不是该项目成员
- `PROJECT_NOT_ENDED` — 项目尚未进入互评阶段
- `EVAL_WINDOW_CLOSED` — 互评窗口已关闭
- `M5004` — 项目不存在

### B2. 检查互评资格 — 目标级

判断当前用户能否评价某个具体成员。覆盖：不能自评、target 必须是 active 成员、不能重复评价、项目 ended、互评窗口未关闭。

| 项 | 内容 |
|---|---|
| **Method** | `GET` |
| **Path** | `/api/m5/projects/{projectId}/members/{targetId}/evaluation-eligibility` |
| **Auth** | `Authorization: Bearer <token>` |
| **请求参数** | Path: `projectId` (Long), `targetId` (Long) |
| **成功响应** | `{ "code": "00000", "message": "成功", "data": true }` |
| **失败响应** | 见下表 |
| **复用 Service** | `EvaluationEligibilityService.validateSubmission(userId, targetId, projectId)` |

**可能的 reasonCode（比项目级更全）：**

| reasonCode | 说明 |
|---|---|
| `SELF_EVALUATION` | 不能评价自己 |
| `ALREADY_EVALUATED` | 已评价过该成员 |
| `NOT_PROJECT_MEMBER` | 评价人不是项目成员 |
| `TARGET_NOT_PROJECT_MEMBER` | 被评价人不是项目成员 |
| `PROJECT_NOT_ENDED` | 项目尚未进入互评阶段 |
| `EVAL_WINDOW_CLOSED` | 互评窗口已关闭 |

### B3. 查询某项目可评价成员

| 项 | 内容 |
|---|---|
| **Method** | `GET` |
| **Path** | `/api/m5/projects/{projectId}/evaluatable-members` |
| **Auth** | `Authorization: Bearer <token>` |
| **请求参数** | Path: `projectId` (Long) |
| **成功响应** | `{ "code": "00000", "data": [ { "userId": 1, "nickname": "张三", "avatarUrl": "...", "evaluated": false } ] }` |
| **失败响应** | 同 B1 资格检查失败码 |
| **复用 Service** | `EvaluationEligibilityService.getEvaluatableMembers(userId, projectId)` |

**返回字段说明：**

| 字段 | 类型 | 说明 |
|---|---|---|
| `userId` | Long | 成员用户 ID |
| `nickname` | String | 成员昵称 |
| `avatarUrl` | String | 成员头像 URL |
| `evaluated` | Boolean | `true` = 当前用户已评价过该成员；`false` = 尚未评价 |

复用已有 DTO: `EvaluatableMemberDTO`（无需新增）。

### B4. 提交互评

| 项 | 内容 |
|---|---|
| **Method** | `POST` |
| **Path** | `/api/m5/evaluations` |
| **Auth** | `Authorization: Bearer <token>` |
| **Content-Type** | `application/json` |
| **请求体** | `SubmitEvaluationRequest`（**新 DTO，不含 evaluatorId**） |
| **成功响应** | `{ "code": "00000", "data": EvaluationSubmitResult }` |
| **失败响应** | 资格/校验/重复/冲突等 reasonCode |
| **复用 Service** | `EvaluationSubmitService.submit(dto)` |

**请求体字段（新 DTO `SubmitEvaluationRequest`）：**

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `projectId` | Long | 是 | 项目 ID |
| `targetId` | Long | 是 | 被评价人 ID |
| `communicationScore` | Integer | 是 | 沟通评分 1-5 |
| `taskScore` | Integer | 是 | 任务完成评分 1-5 |
| `skillScore` | Integer | 是 | 技术能力评分 1-5 |
| `responsibilityScore` | Integer | 是 | 责任心评分 1-5 |
| `comment` | String | 否 | 评价评论（低分时 ≥20 字） |
| `positiveTags` | List\<String\> | 否 | 正向标签列表 |
| `negativeTags` | List\<String\> | 否 | 负向标签列表 |

**关键设计：`evaluatorId` 不在请求体中。**

Controller 通过 `authService.getUserIdFromToken(token)` 获取 userId，在调用 Service 前注入 `dto.setEvaluatorId(userId)`。

> **评价人身份只能来自 token，不能来自前端 body。** 这是 P0 安全约束。

**Controller 处理流程：**

```text
1. authService.getUserIdFromToken(token) → userId
2. SubmitEvaluationRequest → EvaluationSubmitDTO
3. dto.setEvaluatorId(userId)   // 强制注入，不信任前端
4. submitService.submit(dto)     // 调已有 Service
5. 透传 Result<EvaluationSubmitResult>
```

**响应字段（已有 `EvaluationSubmitResult`）：**

| 字段 | 类型 | 说明 |
|---|---|---|
| `evaluationId` | Long | 创建的互评记录 ID |
| `evaluationStatus` | String | `normal` / `pending_review` |
| `effective` | Boolean | 是否已计入信誉分 |
| `creditResult` | CreditCalculationResult | 信誉计算详情（均分、delta、封顶信息） |
| `anomalyResult` | AnomalyDetectionResult | 异常检测结果 |

### B5. 查看我收到的互评

| 项 | 内容 |
|---|---|
| **Method** | `GET` |
| **Path** | `/api/m5/evaluations/received` |
| **Auth** | `Authorization: Bearer <token>` |
| **请求参数** | Query: `projectId` (Long, **可选**) |
| **成功响应** | `{ "code": "00000", "data": [ ReceivedEvaluationVO ] }` |
| **复用** | `EvaluationMapper`（需新增 `findByTargetId` 查询 — **只读查询妥协方案**） |

**projectId 过滤语义（已落实）：**

| projectId | 行为 |
|---|---|
| 不传 / null | 返回当前用户收到的**全部**非 voided 评价（跨项目） |
| 传入具体值 | 只返回当前用户在该 `projectId` 下收到的非 voided 评价 |

**返回字段（新 VO `ReceivedEvaluationVO`）：**

| 字段 | 类型 | 说明 |
|---|---|---|
| `evaluationId` | Long | 互评记录 ID |
| `projectId` | Long | 项目 ID |
| `communicationScore` | Integer | 沟通评分 |
| `taskScore` | Integer | 任务完成评分 |
| `skillScore` | Integer | 技术能力评分 |
| `responsibilityScore` | Integer | 责任心评分 |
| `averageScore` | BigDecimal | 四维均分 |
| `comment` | String | 评价评论 |
| `status` | String | `normal` / `pending_review` / `kept_no_credit` |
| `createdAt` | LocalDateTime | 评价时间 |

**关键约束：**

1. **匿名** — 不包含 `evaluatorId`，不暴露评价者身份（V2.1 §6.8）
2. **过滤 voided** — `status` 等于 `Evaluation.STATUS_VOIDED` 的评价不展示给被评价者（V2.1 §6.8）。优先使用 `Evaluation` 实体已有常量进行过滤，不硬编码字符串字面量
3. **待复核/保留不计分展示** — `status` 为 `Evaluation.STATUS_PENDING_REVIEW` 和 `Evaluation.STATUS_KEPT_NO_CREDIT` 的评价正常展示，前端通过 `status` 字段区分展示样式
4. **projectId 过滤在 Controller 层完成** — 详见 D1

**只读查询妥协方案（在计划中明确标注）：**

> `EvaluationController` 对 B5 直接调用 `EvaluationMapper.findByTargetId`，仅做：查数据库 → 过滤 voided → 按 projectId 过滤（如有）→ 转 VO。**不得**加入业务状态机、信誉分计算、复核逻辑。如后续查询需求增多，抽取 `EvaluationQueryService`。

**已确认：** 项目中不存在已有的 `EvaluationQueryService` 或类似只读查询 Service。M5 现有 Service 全为写链路（资格校验、提交编排、复核、申诉恢复），因此当前最小侵入方案是 Controller 直接调 Mapper。

---

## C. 拟新增/修改文件清单

| # | 文件 | 操作 | 类型 |
|---|---|---|---|
| 1 | `src/main/java/com/teammatch/controller/EvaluationController.java` | **新增** | Controller |
| 2 | `src/main/java/com/teammatch/dto/SubmitEvaluationRequest.java` | **新增** | 请求 DTO |
| 3 | `src/main/java/com/teammatch/dto/ReceivedEvaluationVO.java` | **新增** | 响应 VO |
| 4 | `src/main/java/com/teammatch/mapper/EvaluationMapper.java` | **修改** | Mapper（新增 findByTargetId） |
| 5 | `src/test/java/com/teammatch/controller/EvaluationControllerTest.java` | **新增** | 测试 |

**不修改的文件：** 所有已有 Service（`EvaluationEligibilityService`、`EvaluationSubmitService`、`CreditCalculationService`、`EvaluationValidationService`、`AnomalyDetectionService`、`EvaluationReviewService`、`AppealRestoreService`、`AuthService`）、已有 Entity、ReasonCode、schema.sql、已有 DTO（`EvaluationSubmitDTO`、`EvaluationSubmitResult`、`EvaluatableMemberDTO`）。

---

## D. 每个文件的职责

### D1. EvaluationController.java

- 注解：`@RestController` + `@RequestMapping("/m5")`
- 依赖注入：`AuthService`、`EvaluationEligibilityService`、`EvaluationSubmitService`、`EvaluationMapper`
- Token 解析：**优先注入 `AuthService`，调用 `authService.getUserIdFromToken(token)`**（与 M6 的 `PenaltyController`、`ReportController` 统一模式）

**鉴权失败处理（写死）：**

所有需要 token 的端点方法签名必须使用：
```java
@RequestHeader(value = "Authorization", required = false) String authHeader
```
`required = false` 是 P0 硬约束——缺失 header 时不能让 Spring 在框架层拦截（如 400 Bad Request），必须让请求进入 Controller 方法体内，由统一鉴权私有方法处理并返回 `Result.fail(ReasonCode.UNAUTHORIZED)`。

鉴权入口封装为私有方法（所有端点统一调用）：
```java
private Long getCurrentUserId(String authHeader) {
    if (authHeader == null || authHeader.isBlank()) {
        throw new AuthenticationException(ReasonCode.UNAUTHORIZED);
    }
    return authService.getUserIdFromToken(authHeader);
}
```

Controller 端点方法入口调用 `getCurrentUserId(authHeader)`。若：
- 缺少 `Authorization` header（null / blank）→ 私有方法直接抛 `AuthenticationException`
- token 无效 / 过期 → `authService.getUserIdFromToken` 内部抛异常

Controller **必须** catch 异常并统一返回 `Result.fail(ReasonCode.UNAUTHORIZED)`，不得让异常逃逸为 500 或不受控响应。

- 5 个端点：

| 端点 | 路径 | 核心逻辑 |
|---|---|---|
| B1 项目级资格 | `GET /m5/projects/{projectId}/evaluation-eligibility` | `eligibilityService.checkProjectEligibility(userId, projectId)` → 直接透传 Result |
| B2 目标级资格 | `GET /m5/projects/{projectId}/members/{targetId}/evaluation-eligibility` | `eligibilityService.validateSubmission(userId, targetId, projectId)` → 成功则 `Result.success(true)`，失败透传 reasonCode |
| B3 可评价成员 | `GET /m5/projects/{projectId}/evaluatable-members` | `eligibilityService.getEvaluatableMembers(userId, projectId)` → 直接透传 Result |
| B4 提交互评 | `POST /m5/evaluations` | ① `SubmitEvaluationRequest` → `EvaluationSubmitDTO`；② `dto.setEvaluatorId(userId)` 强制注入；③ `submitService.submit(dto)` → 透传 Result |
| B5 收到的互评 | `GET /m5/evaluations/received` | `evaluationMapper.findByTargetId(userId)` → 过滤 voided → 若 `projectId` 非空则按 `projectId` 二次过滤 → 转换为 `ReceivedEvaluationVO` 列表 → `Result.success(list)`（只读查询妥协方案） |

**B5 处理流程（已落实 projectId 过滤）：**

```text
1. getCurrentUserId(authHeader) → userId
2. evaluationMapper.findByTargetId(userId) → List<Evaluation>
3. 过滤：排除 status == Evaluation.STATUS_VOIDED 的记录
4. 若 projectId != null：二次过滤，
   使用 Objects.equals(evaluation.getProjectId(), projectId) 比较
   （避免 Long 拆箱 NPE，projectId 和 evaluation.getProjectId() 均可能为 null）
5. 转换为 List<ReceivedEvaluationVO>
6. Result.success(list)
```

### D2. SubmitEvaluationRequest.java

```text
包路径: com.teammatch.dto
字段（不含 evaluatorId）:
  - Long projectId
  - Long targetId
  - Integer communicationScore
  - Integer taskScore
  - Integer skillScore
  - Integer responsibilityScore
  - String comment
  - List<String> positiveTags
  - List<String> negativeTags
```

与 `EvaluationSubmitDTO` 的区别：去掉了 `evaluatorId` 字段。Controller 负责从 `authService.getUserIdFromToken(token)` 解析 userId 后注入。

### D3. ReceivedEvaluationVO.java

```text
包路径: com.teammatch.dto
字段（不暴露 evaluatorId）:
  - Long evaluationId
  - Long projectId
  - Integer communicationScore
  - Integer taskScore
  - Integer skillScore
  - Integer responsibilityScore
  - BigDecimal averageScore
  - String comment
  - String status          // normal / pending_review / kept_no_credit
  - LocalDateTime createdAt  // 统一使用 LocalDateTime，与项目 Entity 风格一致
```

**时间字段类型说明：** 项目现有 Entity（`Evaluation`、`CreditChange`）全部使用 `LocalDateTime`，现有 DTO（`EvaluatableMemberDTO`、`EvaluationSubmitResult`、`CreditCalculationResult`）不包含时间字段。`ReceivedEvaluationVO` 新增时间字段统一使用 `LocalDateTime`，与项目整体风格一致。

### D4. EvaluationMapper.java（最小修改）

新增一个只读查询方法：

```java
/**
 * 查询用户收到的所有互评（按 target_id）
 * M5-8A B5 使用，前台匿名展示。
 * 过滤 voided 和 projectId 二次过滤在 Controller 层完成。
 * 如后续需按 projectId 过滤，Controller 层做 Java 过滤即可（数据量可控）。
 */
@Select("SELECT * FROM evaluation WHERE target_id = #{targetId} ORDER BY created_at DESC")
List<Evaluation> findByTargetId(@Param("targetId") Long targetId);
```

**过滤策略说明：**

- Mapper 层：只按 `target_id` 查询全部记录，不做 projectId 和 status 过滤
- Controller 层：过滤 `status == Evaluation.STATUS_VOIDED`，再按 `projectId`（若传入）过滤
- 理由：用户收到的互评数量有限（项目规模上限约束），Java 内存过滤开销可忽略；避免 Mapper 方法膨胀

---

## E. 需要复用的已有 Service

| 接口 | Service 类 | 复用位置 |
|---|---|---|
| **鉴权** | `AuthService.getUserIdFromToken(token)` | `EvaluationController` 所有端点入口 |
| **B1** 项目级资格 | `EvaluationEligibilityService.checkProjectEligibility()` | `EvaluationController` |
| **B2** 目标级资格 | `EvaluationEligibilityService.validateSubmission()` | `EvaluationController` |
| **B3** 可评价成员列表 | `EvaluationEligibilityService.getEvaluatableMembers()` | `EvaluationController` |
| **B4** 提交互评 | `EvaluationSubmitService.submit()` | `EvaluationController` |
| B4 兜底资格校验 | `EvaluationEligibilityService.validateSubmission()` | 已在 `submit()` 链路内调用 |
| B4 内容校验 | `EvaluationValidationService.validateSubmission()` | 已在 `submit()` 链路内调用 |
| B4 信誉计算 | `CreditCalculationService.calculate()` | 已在 `submit()` 链路内调用 |
| B4 异常检测 | `AnomalyDetectionService.detect()` | 已在 `submit()` 链路内调用 |

**不复用的 Service（M5-8A 是用户端，不做管理端）：**

- `EvaluationReviewService` — 管理端评价复核
- `AppealRestoreService` — 管理端申诉恢复
- M4 的 `ExitVoteService` / `ProjectService` — M5 不跨模块

---

## F. 需要新增的 DTO/VO

| DTO/VO | 用途 | 关键约束 |
|---|---|---|
| `SubmitEvaluationRequest` | B4 请求体 | **不含 `evaluatorId`**，由 Controller 从 `authService.getUserIdFromToken()` 注入 |
| `ReceivedEvaluationVO` | B5 响应 | **不含 `evaluatorId`**（匿名）；`status` 原样透传（`voided` 已被过滤不展示）；`createdAt` 使用 `LocalDateTime` |

**无需新增的 DTO（已有可直接用）：**

- `EvaluationSubmitDTO`（B4 内部转换用）— 已存在，Controller 注入 evaluatorId 后使用
- `EvaluationSubmitResult`（B4 出参）— 已存在
- `EvaluatableMemberDTO`（B3 出参）— 已存在

---

## G. 测试计划

### G1. 测试框架与模式

- **框架：** `@WebMvcTest` + `@MockBean` + JUnit 5 + MockMvc + AssertJ JSONPath
- **模式：** 参照已有 `ExitVoteControllerTest` 的 `@WebMvcTest` 写法
- **Mock：** Mock 所有 Service/Mapper 依赖（`AuthService`、`EvaluationEligibilityService`、`EvaluationSubmitService`、`EvaluationMapper`），Controller 只测自己的行为
- **每端点至少覆盖：** 成功路径 + 2 种主要失败路径
- **测试文件存放：** `src/test/java/com/teammatch/controller/`

### G2. EvaluationControllerTest

| # | 端点 | 测试场景 | 预期 `$.code` | 关键断言 |
|---|---|---|---|---|
| **鉴权（至少覆盖 B1 和 B5，验证缺 header 不会被 Spring 框架层拦截）** |
| A1 | B1, B5 | 缺少 Authorization header（`required=false` 保证请求进入 Controller） | `"M3000"` (UNAUTHORIZED) | Controller 返回统一鉴权失败，不抛 400/500 |
| A2 | B1, B5 | token 无效/过期 | `"M3000"` (UNAUTHORIZED) | 同上 |
| **B1 项目级资格** |
| E1 | B1 | 正常互评资格 → `eligible=true` | `"00000"` | `$.data = true` |
| E2 | B1 | 非项目成员 → 拒绝 | `"NOT_PROJECT_MEMBER"` | `$.data = null` |
| E3 | B1 | 项目未结束 → 拒绝 | `"PROJECT_NOT_ENDED"` | |
| E4 | B1 | 互评窗口关闭 → 拒绝 | `"EVAL_WINDOW_CLOSED"` | |
| **B2 目标级资格** |
| E5 | B2 | 正常可评价该 target | `"00000"` | `$.data = true` |
| E6 | B2 | 自评 → 拒绝 | `"SELF_EVALUATION"` | |
| E7 | B2 | 重复评价 → 拒绝 | `"ALREADY_EVALUATED"` | |
| E8 | B2 | target 非项目成员 → 拒绝 | `"TARGET_NOT_PROJECT_MEMBER"` | |
| **B3 可评价成员** |
| E9 | B3 | 正常返回成员列表 | `"00000"` | `$.data` 为数组，含 `userId`/`nickname`/`evaluated` |
| E10 | B3 | 资格不通过 → 透传失败码 | 同 B1 | |
| **B4 提交互评** |
| E11 | B4 | 正常提交 → 返回结果 | `"00000"` | `$.data.evaluationId` 非空；verify evaluatorId 被正确注入 |
| E12 | B4 | 请求体不含 evaluatorId，Controller 注入成功 | `"00000"` | Service 收到的 dto.evaluatorId == mockUserId |
| E13 | B4 | 自评 → 拒绝 | `"SELF_EVALUATION"` | Service 返回，Controller 透传 |
| E14 | B4 | 重复评价 → 拒绝 | `"ALREADY_EVALUATED"` | |
| **B5 收到的互评** |
| E15 | B5 | 正常返回收到的评价列表（不传 projectId） | `"00000"` | `$.data` 为数组，不含 `evaluatorId` 字段 |
| E16 | B5 | voided 评价被过滤 | `"00000"` | 列表不含 `status=voided` 的记录 |
| E17 | B5 | pending_review 评价正常展示 | `"00000"` | 列表含 `status=pending_review` 的记录 |
| E18 | B5 | **传入 projectId，过滤生效** | `"00000"` | 所有返回记录的 `projectId` 均等于传入值 |
| E19 | B5 | **传入 projectId，该 projectId 下无评价 → 空数组** | `"00000"` | `$.data` 为空数组 `[]` |
| E20 | B5 | **不传 projectId，返回全部 projectId 的评价** | `"00000"` | `$.data` 包含不同 projectId 的记录 |

### G3. 测试运行命令

```bash
# 仅运行 M5-8A 测试
mvn -Dtest="EvaluationControllerTest" test

# 运行全部 M5 测试（确认不回归）
mvn test -pl .
```

---

## H. 明确不做什么

1. **不写管理端接口** — 不暴露 `EvaluationReviewController` 或 `AppealRestoreController`
2. **不写评价复核管理入口** — `EvaluationReviewService` 和 `AppealRestoreService` 已有 Service 但不包 Controller
3. **不写 report / penalty** — M6 职责
4. **不写 Jenkins / 部署** — M6 职责
5. **不修改 `schema.sql`** — 不新增字段、不改约束、不改 DDL
6. **不在 Controller 重写业务状态机** — 所有判断调用已有 Service
7. **不绕开 `credit_change` 直接修改 `user.credit_score`** — B4 走 `submit()` 事务链路
8. **不暴露 `evaluatorId` 给被评价者** — B5 用 `ReceivedEvaluationVO` 剥离身份字段
9. **不写前端代码** — 只出后端 API
10. **不做 `skill_endorse` 写入** — 当前 `submit` 链路暂未写入（已知限制），不在 M5-8A 补。请求体中也不出现 `skillTagIds` 字段
11. **不做互评申诉提交接口** — `appeal` 提交属于 M6 治理链路，不在 M5-8A 用户端范围
12. **不做未读角标聚合接口** (`GET /api/me/badges`) — 属于 M3/M4 通知体系，不在 M5-8A 范围
13. **不修改已有 `EvaluationSubmitDTO` / `EvaluationSubmitService`** — 通过新 `SubmitEvaluationRequest` DTO 隔离，不碰已有 Service 契约和测试
14. **不做信誉查询接口（B6/B7）** — 拆分到 M5-8B，不在 M5-8A 范围
15. **不新增 Mapper 方法做 projectId 过滤** — B5 的 projectId 过滤在 Controller 层 Java 内存完成

---

## I. 主要风险点

| # | 风险 | 影响 | 缓解措施 |
|---|---|---|---|
| **R1** | **B4 身份伪造** — 若后续有人在请求体中加 `evaluatorId` | 越权评价 | `SubmitEvaluationRequest` 不含该字段；Controller 通过 `authService.getUserIdFromToken()` 获取 userId 并注入。P0 安全约束 |
| **R2** | **B5 缺少 Mapper 查询** — `EvaluationMapper.findByTargetId` 不存在 | B5 无法实现 | 新增一个简单 `@Select`（只读，无业务逻辑），标注为「只读查询妥协方案」 |
| **R3** | **B5 匿名不彻底** — comment 文本中可能含评价者身份信息 | 匿名承诺在极端情况下失效 | VO 字段级白名单确保不泄露 `evaluatorId`，但 comment 文本由用户填写，无法完全剥离；文档标注已知限制 |
| **R4** | **Token 解析异常** — `authService.getUserIdFromToken` 异常未捕获 | 返回 500 或不受控响应 | Controller 必须 catch 异常并统一返回 `Result.fail(ReasonCode.UNAUTHORIZED)`，测试覆盖 A1/A2 |
| **R5** | **B5 voided 过滤后列表为空** — 用户所有收到的评价都被 voided | 返回空数组 `[]`，用户可能认为数据丢失 | 返回空数组而非错误；前端展示"暂无有效评价" |
| **R6** | **B4 新 DTO 增加维护成本** — `SubmitEvaluationRequest` 与 `EvaluationSubmitDTO` 字段高度重合 | 字段变更需同步两处 | 两处 DTO 职责不同：一个面向前端（无 evaluatorId），一个面向 Service（有 evaluatorId）。后续可改为 MapStruct 自动映射 |

---

## J. M5-8A 执行边界

**本次只做 M5-8A（互评用户端 API），不做 M5-8B（信誉查询）。**

| Change | 内容 | 文件 |
|---|---|---|
| **M5-8A** | 互评读写闭环 | `EvaluationController` + `SubmitEvaluationRequest` + `ReceivedEvaluationVO` + `EvaluationMapper` 扩展 + `EvaluationControllerTest` |

**合入后 M5-8B 单独规划：** 信誉分值（B6）、信誉分变化流水（B7）拆到 M5-8B，不在本文档范围。

### M5-8A 详细拆解

| 文件 | 行数估算 | 说明 |
|---|---|---|
| `EvaluationController.java` | ~160 行 | 5 个端点 + AuthService 鉴权注入 + 异常捕获 + evaluatorId 注入 + B5 projectId 过滤 |
| `SubmitEvaluationRequest.java` | ~55 行 | 9 个字段（不含 evaluatorId） |
| `ReceivedEvaluationVO.java` | ~50 行 | 10 个字段（createdAt 为 LocalDateTime） |
| `EvaluationMapper.java`（修改） | +8 行 | 新增 `findByTargetId` |
| `EvaluationControllerTest.java` | ~300 行 | 20 个测试用例（含鉴权 2 + 业务 18） |

---

## K. 执行前必须只读检查的现有结构（已确认，仅供参考）

以下检查项已在制定本计划前完成：

| 检查项 | 结果 | 证据 |
|---|---|---|
| 全局 context-path | `server.servlet.context-path: /api` | `application.yml` L28 |
| 已有 Controller 路径风格 | `@RequestMapping("/auth")` 等，不写 `/api` 前缀 | `AuthController`、`ProfileController`、`ExitVoteController` |
| **`AuthService.getUserIdFromToken(String)`** | ✅ **存在**，M6 Controller 已统一使用 | `AuthService.java:87`；`PenaltyController`、`ReportController` 调用方 |
| Result 成功 code | `"00000"` | `ReasonCode.SUCCESS` |
| reasonCode 返回字段 | `code` / `message` / `data` | `Result.java` |
| `Evaluation` 状态常量 | ✅ `STATUS_NORMAL`、`STATUS_PENDING_REVIEW`、`STATUS_VOIDED`、`STATUS_KEPT_NO_CREDIT` | `Evaluation.java:15-18` |
| 项目 VO 时间字段风格 | Entity 全部使用 `LocalDateTime`；现有 DTO 不含时间字段 | `Evaluation.java`、`CreditChange.java`、`EvaluatableMemberDTO` 等 |
| 已有 M5 Mapper 方法 | `EvaluationMapper.findByTargetId` **不存在** | 搜索确认 |
| 已有 M5 查询 Service | **不存在** `EvaluationQueryService` | 搜索确认 |

---

## L. 仍需执行前只读确认的方法签名和类

以下内容在执行前需通过只读工具最终确认，避免因分支差异导致实现偏差：

### L-A. 鉴权相关

| # | 需确认项 | 确认方式 | 影响 |
|---|---|---|---|
| L1 | `AuthService.getUserIdFromToken` 方法签名（参数类型、返回值、异常声明） | `read_file AuthService.java:87` | Controller 注入和调用方式 |
| L2 | `AuthServiceImpl.getUserIdFromToken` 对 `Bearer ` 前缀是否自动 strip | `read_file AuthServiceImpl.java:334` | Controller 传参时是否需要手动 strip |
| L3 | `ReasonCode.UNAUTHORIZED` 的 code 值 | `read_file ReasonCode.java` | 鉴权失败响应的 `$.code` 断言值（当前预期 `"M3000"`） |
| L4 | `AuthenticationException` 构造方法是否接受 `ReasonCode` 参数 | `read_file exception/AuthenticationException.java` | 私有鉴权方法 `getCurrentUserId` 的异常构造方式 |

### L-B. Service 层签名

| # | 需确认项 | 确认方式 | 影响 |
|---|---|---|---|
| L5 | `EvaluationEligibilityService.checkProjectEligibility(Long userId, Long projectId)` 返回值类型 | `read_file EvaluationEligibilityService.java` | B1 Controller 调用和 Result 透传 |
| L6 | `EvaluationEligibilityService.validateSubmission(Long evaluatorId, Long targetId, Long projectId)` 返回值类型 | 同上 | B2 Controller 调用和 Result 透传 |
| L7 | `EvaluationEligibilityService.getEvaluatableMembers(Long userId, Long projectId)` 返回值类型 | 同上 | B3 Controller 调用和 Result 透传 |
| L8 | `EvaluationSubmitService.submit(EvaluationSubmitDTO dto)` 方法签名 | `read_file EvaluationSubmitService.java` | B4 Controller 调用 |

### L-C. DTO / Entity 字段

| # | 需确认项 | 确认方式 | 影响 |
|---|---|---|---|
| L9 | `EvaluationSubmitDTO` 的 `setEvaluatorId(Long)` setter 是否存在 | `read_file EvaluationSubmitDTO.java` | B4 注入 evaluatorId 的方式 |
| L10 | `EvaluationSubmitDTO` 全部字段与 `SubmitEvaluationRequest` 的对应关系 | 同上 | DTO 转换代码 |
| L11 | `Evaluation.STATUS_VOIDED`、`STATUS_PENDING_REVIEW`、`STATUS_KEPT_NO_CREDIT`、`STATUS_NORMAL` 常量值 | `read_file Evaluation.java:15-18` | B5 状态过滤字符串（确认常量值为 `"voided"` / `"pending_review"` / `"kept_no_credit"` / `"normal"`） |

### L-D. Mapper

| # | 需确认项 | 确认方式 | 影响 |
|---|---|---|---|
| L12 | `EvaluationMapper` 当前已有的 `@Select` 方法列表和命名风格 | `read_file EvaluationMapper.java` | 避免 `findByTargetId` 命名冲突 |

---

## M. 开发顺序建议

```
1. 执行前只读确认（L1-L12）
2. 读本文档确认理解
3. 创建 M5-8A 文件：
   - SubmitEvaluationRequest.java
   - ReceivedEvaluationVO.java
   - EvaluationMapper.java（新增 findByTargetId）
   - EvaluationController.java
   - EvaluationControllerTest.java
4. 运行 EvaluationControllerTest 确认全部 20 个测试通过
5. 本地启动验证 B1/B2/B3/B4/B5 端点
6. Gerrit 提交 M5-8A
7. 追加 IMPLEMENT_LOG.md
```
