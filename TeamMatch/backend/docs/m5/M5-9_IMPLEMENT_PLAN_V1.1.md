# M5-9 管理端评价复核 Controller — 实施规划 V1.1

> 修订自 V1.0。修订要点：鉴权流程对齐 M6 PenaltyAdminController 风格、测试策略去除非稳定断言、补充 context-path 确认项。

## A. 任务边界

M5-9 把 M5-6 已完成的 `EvaluationReviewService.review()` 包装成管理端 HTTP API，并提供待复核评价列表查询。

| 在范围内 | 不在范围内 |
|---|---|
| 2 个管理端 Controller 端点（待复核列表 + 执行复核） | 用户端接口 |
| `AuthService.isAdmin(token)` 管理员鉴权 | 业务状态机重写 |
| DTO 转换 + 统一响应 `Result<T>` | `schema.sql` 修改 |
| `LambdaQueryWrapper` 查询 `pending_review` 列表 | 新增 Mapper 方法 |
| Controller 层单元测试 (MockMvc) | report / penalty / Jenkins / 部署 |
| `reviewerId` 从 token 强制注入 | 修改已有 Service |

**Controller 层职责边界（硬约束）：**

```
token → isAdmin check → getUserIdFromToken → 参数接收 → 调用已有 Service → DTO/VO 转换 → Result<T>
```

所有业务判断（状态机、事务边界、信誉计算）继续留在已有 `EvaluationReviewServiceImpl` 中，Controller 不重写、不扩张。

## B. 接口列表

### B1. GET /api/admin/evaluations/pending（待复核列表）

| 项 | 内容 |
|---|---|
| Method | GET |
| Path | `/api/admin/evaluations/pending` |
| Auth | `Authorization: Bearer <token>` + `isAdmin` 校验 |
| 请求参数 | Query: `projectId` (Long, 可选) |
| 成功响应 | `{ "code": "00000", "data": [ PendingEvaluationVO ] }` |
| 复用 | `evaluationMapper.selectList(wrapper)` — `LambdaQueryWrapper` 按 `status = 'pending_review'` + 可选 `projectId` + `createdAt DESC` |

**返回字段（新 VO `PendingEvaluationVO`）：**

| 字段 | 类型 | 说明 |
|---|---|---|
| `evaluationId` | Long | 互评记录 ID |
| `projectId` | Long | 项目 ID |
| `evaluatorId` | Long | 评价人 ID（管理端可追溯） |
| `targetId` | Long | 被评价人 ID |
| `communicationScore` | Integer | 沟通评分 |
| `taskScore` | Integer | 任务完成评分 |
| `skillScore` | Integer | 技术能力评分 |
| `responsibilityScore` | Integer | 责任心评分 |
| `averageScore` | BigDecimal | 四维均分 |
| `comment` | String | 评价评论 |
| `status` | String | 始终为 `"pending_review"` |
| `createdAt` | LocalDateTime | 创建时间 |

**约束：**
- 仅返回 `status = 'pending_review'` 的评价
- `projectId` 为可选过滤
- 按 `createdAt DESC` 排序
- 管理端可暴露 `evaluatorId`（与用户端 B5 匿名展示不同）
- 不传分页参数，返回全量列表（与 M6 admin controller 风格一致；P0 简化，后续数据量增大再扩展 `page`/`pageSize`）

### B2. POST /api/admin/evaluations/{id}/review（执行复核）

| 项 | 内容 |
|---|---|
| Method | POST |
| Path | `/api/admin/evaluations/{id}/review` |
| Auth | `Authorization: Bearer <token>` + `isAdmin` 校验 |
| Content-Type | `application/json` |
| 请求体 | `EvaluationReviewRequest`（不含 `reviewerId`） |
| 成功响应 | `{ "code": "00000", "data": EvaluationReviewResult }` |
| 复用 Service | `evaluationReviewService.review(command)` |

**请求体字段（`EvaluationReviewRequest`）：**

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `action` | String | 是 | `approve` / `void` / `keep_no_credit` |
| `reviewNote` | String | 否 | 复核备注（最大 500 字符，Service 层已校验） |

**Controller 不对 `action` 做枚举校验**——合法性完全交给 `EvaluationReviewService.review()`，非法值由 Service 返回 M5007。

**处理流程（对齐 M6 PenaltyAdminController 写端点风格）：**

```
1. isAdmin(token) → false → Result.fail(ADMIN_REQUIRED)（M3009）
2. adminId = getUserIdFromToken(token) → null → Result.fail(UNAUTHORIZED)（M3000，防御性守卫）
3. command.setEvaluationId(id)          // Path variable
4. command.setReviewerId(adminId)        // Token 强制注入
5. command.setAction(request.getAction())
6. command.setReviewNote(request.getReviewNote())
7. return evaluationReviewService.review(command)  // 透传 Result
```

**`reviewerId` 不在请求体中**——管理员身份只能来自 token，这是 P0 安全约束。

**可能的 reasonCode（Service 返回，Controller 透传）：**

| Code | 枚举 | 含义 |
|---|---|---|
| M5005 | `EVALUATION_NOT_FOUND` | 评价记录不存在 |
| M5007 | `INVALID_REVIEW_ACTION` | 无效的复核操作 |
| M1003 | `STATUS_CONFLICT` | 评价状态不是 `pending_review` |
| M5008 | `REVIEW_NOTE_TOO_LONG` | 复核备注过长 |
| M5006 | `CREDIT_CHANGE_NOT_FOUND` | 信誉分流水不存在或状态异常 |

## C. 拟新增文件

| # | 文件 | 操作 | 类型 |
|---|---|---|---|
| 1 | `src/main/java/com/teammatch/m6/controller/EvaluationAdminController.java` | 新增 | Controller |
| 2 | `src/main/java/com/teammatch/m6/dto/EvaluationReviewRequest.java` | 新增 | 请求 DTO |
| 3 | `src/main/java/com/teammatch/m6/dto/PendingEvaluationVO.java` | 新增 | 响应 VO |
| 4 | `src/test/java/com/teammatch/m6/controller/EvaluationAdminControllerTest.java` | 新增 | 测试 |

**不修改的文件：** `EvaluationReviewService`、`EvaluationReviewServiceImpl`、`EvaluationMapper`、`Evaluation`、`ReviewAction`、`EvaluationReviewCommand`、`EvaluationReviewResult`、`ReasonCode`、`schema.sql`。

### 为什么放在 `m6.controller` / `m6.dto` 包？

M6 在本项目中承担"管理端 HTTP 入口层"角色。现有 `AppealAdminController`（申诉）、`PenaltyAdminController`（处罚）、`ReportAdminController`（举报）全部位于 `com.teammatch.m6.controller`，尽管它们调用的 Service 分属不同业务模块。`EvaluationAdminController` 的业务能力属于 M5（评价复核），但 HTTP 管理端入口归入 M6 controller 包，与既有分层约定完全一致。Controller 本身不实现 M6 report/penalty 业务，只暴露 M5 Service 的管理端入口。

## D. 各文件职责

### D1. EvaluationAdminController.java

- 包路径：`com.teammatch.m6.controller`
- 注解：`@RestController` + `@RequestMapping("/admin/evaluations")` + `@RequiredArgsConstructor`
- 依赖注入：`AuthService`、`EvaluationReviewService`、`EvaluationMapper`
- 外部路径前缀：`server.servlet.context-path=/api` → 完整路径 `/api/admin/evaluations/...`

**鉴权模式（对齐 PenaltyAdminController）：**

读端点（GET）：`isAdmin(token)` → false 返回 M3009。
写端点（POST）：先 `isAdmin(token)` → M3009，再 `getUserIdFromToken(token)` → null 时 M3000。

说明：
- `isAdmin(token)` 内部调 `getUserIdFromToken`，token 无效/过期抛 `AuthenticationException` → `AuthExceptionHandler` 全局拦截 → M3000。
- `@RequestHeader("Authorization")` 不设 `required = false`（与所有 M6 Controller 一致）。缺少 header → Spring `MissingRequestHeaderException` → 400。这是项目既有行为，M5-9 不改变。
- 不引入私有鉴权方法，不引入 `throw new AuthenticationException`。

**B1（GET /admin/evaluations/pending）逻辑：**

```text
1. isAdmin(token) → 失败返回 M3009
2. LambdaQueryWrapper<Evaluation> wrapper = new LambdaQueryWrapper<>();
3. wrapper.eq(Evaluation::getStatus, Evaluation.STATUS_PENDING_REVIEW);
4. if (projectId != null) { wrapper.eq(Evaluation::getProjectId, projectId); }
5. wrapper.orderByDesc(Evaluation::getCreatedAt);
6. List<Evaluation> list = evaluationMapper.selectList(wrapper);
7. List<PendingEvaluationVO> voList = list.stream().map(this::toPendingEvaluationVO).collect(toList());
8. return Result.success(voList);
```

**B2（POST /admin/evaluations/{id}/review）逻辑：**

```text
1. isAdmin(token) → 失败返回 M3009
2. adminId = getUserIdFromToken(token) → null 则返回 M3000
3. EvaluationReviewCommand command = new EvaluationReviewCommand();
4. command.setEvaluationId(id);
5. command.setReviewerId(adminId);
6. command.setAction(request.getAction());
7. command.setReviewNote(request.getReviewNote());
8. return evaluationReviewService.review(command);  // 透传
```

### D2. EvaluationReviewRequest.java

- 包路径：`com.teammatch.m6.dto`
- 字段（不含 `reviewerId`）：`String action`、`String reviewNote`
- 与 `EvaluationReviewCommand` 的区别：去掉 `evaluationId` 和 `reviewerId`——前者来自 Path Variable，后者来自 token。

### D3. PendingEvaluationVO.java

- 包路径：`com.teammatch.m6.dto`
- 字段：`evaluationId`、`projectId`、`evaluatorId`、`targetId`、`communicationScore`、`taskScore`、`skillScore`、`responsibilityScore`、`averageScore`、`comment`、`status`、`createdAt`
- 管理端专用，含 `evaluatorId`（区别于用户端匿名展示）。

### D4. EvaluationAdminControllerTest.java

- `@WebMvcTest(EvaluationAdminController.class)` + `@MockBean` + JUnit 5 + MockMvc + AssertJ JSONPath
- Mock 全部依赖（`AuthService`、`EvaluationReviewService`、`EvaluationMapper`）
- ≥14 个测试用例

## E. 测试计划

### 鉴权测试（4 个）

| # | 端点 | 场景 | 预期 `$.code` | 说明 |
|---|---|---|---|---|
| A1 | B1 | 无效/过期 token | `"M3000"` | Mock `isAdmin` throw `AuthenticationException`，经 `AuthExceptionHandler` 拦截 |
| A2 | B2 | 同上 | `"M3000"` | 同上 |
| A3 | B1 | 非管理员 | `"M3009"` | Mock `isAdmin` return `false` |
| A4 | B2 | 同上 | `"M3009"` | 同上 |

> 缺少 Authorization header → 400（Spring `MissingRequestHeaderException`），是 M6 所有 Controller 统一行为，M5-9 保持一致。如需改进为返回 M3000，应作为独立横切改动覆盖所有 Controller。

### B1 待复核列表（4 个）

| # | 场景 | 预期 `$.code` | 关键断言 |
|---|---|---|---|
| C1 | 空列表 | `"00000"` | `$.data` 为空数组 |
| C2 | 正常返回 | `"00000"` | `$.data[0]` 含 `evaluationId`/`projectId`/`evaluatorId`/scores/`status="pending_review"`/`createdAt` |
| C3 | `projectId` 过滤 | `"00000"` | Mock `selectList` 返回过滤后列表；不强行断言 `LambdaQueryWrapper` 内部状态 |
| C4 | `createdAt DESC` 排序 | `"00000"` | Mock 返回预排序列表，断言首条为最新；排序正确性走 code review |

> C3/C4 修订说明：V1.0 原计划用 `ArgumentCaptor` 断言 `LambdaQueryWrapper` 内部 `eq`/`orderByDesc` 状态——这在 MockMvc 中不可行（mock 不执行 wrapper 内部逻辑）。修订后 mock 返回预设数据，测试关注 HTTP 层响应；`LambdaQueryWrapper` 的正确性走源码审查（`Evaluation::getXxx` Lambda 方法引用编译期校验，风险可控）。

### B2 执行复核（6 个）

| # | 场景 | 预期 `$.code` | 关键断言 |
|---|---|---|---|
| C5 | approve 成功 | `"00000"` | `$.data.newStatus = "normal"`，`creditEffectiveChanged = true` |
| C6 | void 成功 | `"00000"` | `$.data.newStatus = "voided"` |
| C7 | keep_no_credit 成功 | `"00000"` | `$.data.newStatus = "kept_no_credit"` |
| C8 | 评价不存在 | `"M5005"` | Mock Service 返回 `Result.fail(EVALUATION_NOT_FOUND)` |
| C9 | 无效 action | `"M5007"` | Mock Service 返回 `Result.fail(INVALID_REVIEW_ACTION)` |
| C10 | 状态冲突 | `"M1003"` | Mock Service 返回 `Result.fail(STATUS_CONFLICT)` |
| C11 | reviewNote 过长 | `"M5008"` | Mock Service 返回 `Result.fail(REVIEW_NOTE_TOO_LONG)` |
| C12 | reviewerId 从 token 注入 | `"00000"` | `command.getReviewerId()` = mock adminId；请求体不含 `reviewerId` |
| C13 | CREDIT_CHANGE_NOT_FOUND | `"M5006"` | Mock Service 返回 `Result.fail(CREDIT_CHANGE_NOT_FOUND)` |
| C14 | 请求体含 reviewerId 被忽略 | `"00000"` | 请求体即使含 `reviewerId`，Controller 不使用 |

## F. 明确不做什么

1. 不修改 `schema.sql`
2. 不修改已有 Service / ServiceImpl
3. 不写用户端接口
4. 不新增 Mapper 方法（使用 `BaseMapper.selectList`）
5. 不新增 ReasonCode
6. 不修改 `credit_change` / `user.credit_score`
7. 不做分页（P0 简化，后续再扩展）
8. 不做批量复核
9. 不写前端代码
10. 不写 Jenkins / 部署
11. Controller 不对 `action` 做枚举校验

## G. 风险点

| # | 风险 | 缓解 |
|---|---|---|
| R1 | `LambdaQueryWrapper` 字段名写错 | Lambda 方法引用编译期校验 |
| R2 | `evaluatorId` 暴露需与用户端区分 | VO 注释标注管理端专用 |
| R3 | `reviewerId` 伪造 | 请求体 DTO 不含 `reviewerId`，Controller 从 token 注入 |
| R4 | M5-8A 可能已有同名接口 | 执行前搜索确认 |
| R5 | M5-8A/M5-8B 未合入 | M5-9 依赖 Service 层已合入 master，不依赖 M5-8A/B Controller |

## H. 开发顺序

1. `git fetch --all --prune`，确认基线为最新 `origin/master`
2. 执行前只读确认（见 I 节）
3. 创建 4 个新文件
4. `mvn test -Dtest=EvaluationAdminControllerTest` → 全部通过
5. `mvn test` → 全量无回归
6. 本地启动验证 B1/B2
7. Gerrit 提交（仅 4 个文件，不含 `IMPLEMENT_LOG.md`）

## I. 执行前只读确认

| # | 确认项 | 确认方式 |
|---|---|---|
| K1 | `AuthServiceImpl.isAdmin` / `getUserIdFromToken` 的异常行为 | `read_file AuthServiceImpl.java` |
| K2 | `EvaluationReviewService.review` 方法签名 | `read_file EvaluationReviewService.java` |
| K3 | MyBatis-Plus 驼峰映射 | `application.yml`（已确认 `map-underscore-to-camel-case: true`） |
| K4 | `Evaluation.STATUS_PENDING_REVIEW` 常量值 | `read_file Evaluation.java`（已确认 `"pending_review"`） |
| K5 | M5-8A 无同名 Controller | 搜索 `EvaluationAdminController` |
| K6 | HEAD = `origin/master` | `git log origin/master..HEAD --oneline` |
| K7 | `com.teammatch.m6.dto` 存在 | `ls src/main/java/com/teammatch/m6/dto/`（已确认存在） |
| K8 | `server.servlet.context-path = /api` | `grep context-path application.yml`（已确认 `/api`） |

## J. 验收标准

1. B1/B2 两个端点可用
2. 非管理员返回 M3009
3. 无效/过期 token 返回 M3000（经 `AuthExceptionHandler`）
4. `reviewerId` 强制从 token 注入，请求体无法传入
5. ≥14 个 Controller 单测全部通过
6. 全量测试无回归
7. 不修改已有 Service / Mapper / Entity
8. `git diff` 仅含 4 个新文件
