# M5-8B 用户端信誉查询接口层 — 实施规划

**版本：** V1.4（K5/K7 收口版）  
**适用范围：** TeamMatch 后端 M5-8B 用户端信誉查询 API/Controller 层  
**前置依赖：**
- M5-8A 已提交 Gerrit Change，但**不一定已 submit/merge 到 master**
- M5-1~M5-7 Service 层已完成
- K1-K12 只读确认显示：当前项目未配置 MyBatis-Plus 分页插件，因此 M5-8B **不依赖 `selectPage` 物理分页**，B7 改用 `selectList + Java 内存分页`，不修改 MyBatis-Plus 配置
- **若 M5-8A 尚未合入 master，M5-8B 不得直接基于包含 M5-8A commit 的本地 HEAD 提交**，避免形成 stacked change
- 执行前必须检查以下命令，确认提交基线为最新 `origin/master` 而非本地未合入分支：

```bash
git fetch origin
git status --short
git branch -vv
git rev-parse HEAD                          # 当前 HEAD commit
git merge-base HEAD origin/master           # 与 master 的共同祖先
git log origin/master..HEAD --oneline       # 本地领先 master 的 commit（若输出含 M5-8A 则暂停）
git diff --name-status origin/master...HEAD # 本地独有文件变更
```

> **判定规则：** 若 `git log origin/master..HEAD --oneline` 包含 M5-8A 相关 commit，或 `git merge-base HEAD origin/master` 不等于 `origin/master`，说明当前基线不干净或不是最新主干，**M5-8B 不得直接在当前 HEAD 施工**。安全选择只有三种：等 M5-8A submit 后从 `origin/master` 干净分支施工；从最新 `origin/master` 新建 8B 独立分支；或仅搬运 8B 的 5 个目标文件，确保 Gerrit Change 不依赖 8A commit。

**最高业务依据：** `docs/design/system-design-v2.1.md`

---

## A. M5-8B 任务边界

M5-8B 把信誉查询能力包装成 2 个用户端 HTTP API。与 M5-8A 无文件交叉，完全独立。

| 在范围内 | 不在范围内 |
|---|---|
| 2 个用户端 Controller 端点（信誉分 + 流水） | 管理端接口 |
| `AuthService.getUserIdFromToken(token)` 鉴权注入 | 业务状态机重写 |
| DTO 转换 + 统一响应 `Result<T>` | `schema.sql` 修改 |
| `effective=false` 流水标注 `suspended=true` | 绕开 `credit_change` 直接改 `user.credit_score` |
| Controller 层单元测试 (MockMvc) | 新增 Mapper 方法（复用 BaseMapper `selectList`） |
| — | M5-8A 文件修改 |
| — | report / penalty / Jenkins / 部署 |

**Controller 层职责边界（硬约束）：**

```
token → userId → 查询 UserMapper / CreditChangeMapper → DTO/VO 转换 → Result<T>
```

只读查询，不写任何数据。

---

## B. 推荐接口列表

### B6. 查看我的信誉分

| 项 | 内容 |
|---|---|
| **Method** | `GET` |
| **Path** | `/api/m5/credit/score` |
| **Auth** | `Authorization: Bearer <token>` |
| **请求参数** | 无 |
| **成功响应** | `{ "code": "00000", "data": { "userId": 1, "creditScore": 95 } }` |
| **复用** | `UserMapper.selectById(userId)` → 读 `credit_score` 缓存字段 |

**返回字段（新 VO `CreditScoreVO`）：**

| 字段 | 类型 | 说明 |
|---|---|---|
| `userId` | Long | 当前用户 ID |
| `creditScore` | Integer | 当前信誉分（`user.credit_score` 缓存值） |

**注意：** 纯读接口。`user.credit_score = 100 + SUM(credit_change.change_value WHERE effective=1)`，由 M5-5/M5-6/M5-7 Service 事务链路维护。Controller 只读不写。

### B7. 查看我的信誉分变化记录

| 项 | 内容 |
|---|---|
| **Method** | `GET` |
| **Path** | `/api/m5/credit/changes` |
| **Auth** | `Authorization: Bearer <token>` |
| **请求参数** | Query: `projectId` (Long, 可选), `changeType` (String, 可选), `page` (int, 默认 1), `pageSize` (int, 默认 20) |

**参数校验规则：**

| 参数 | 规则 |
|---|---|
| `page` | 默认 1，若 < 1 按 1 处理 |
| `pageSize` | 默认 20，若 < 1 按 20 处理，**最大 100**（超过截断为 100） |
| `changeType` | 若非空，必须在白名单内。白名单在施工时对齐 `schema.sql` 与 `CreditChange` 实体 `changeType` 注释（当前为 `evaluation` / `exit_vote` / `self_exit` / `penalty` / `penalty_restore` / `appeal_restore`），**不得自行裁剪**。白名单在 Controller 中定义为 `private static final Set<String> ALLOWED_CHANGE_TYPES`，不得散落字符串字面量。非法值返回 `PARAM_ERROR`（`ReasonCode.PARAM_ERROR`，code=`"M1001"`） |

**排序规则：** 默认按 `created_at DESC` 排序，最近的信誉变化记录排在最前面。

**查询安全约束：** 所有查询条件必须以 `userId = currentUserId` 为第一条件。**不允许通过请求参数传入 userId 查询他人流水。**

**成功响应：** `{ "code": "00000", "data": { "list": [ CreditChangeVO ], "total": 15, "page": 1, "pageSize": 20 } }`

**复用：** `CreditChangeMapper` (MyBatis-Plus `BaseMapper.selectList`)

**返回字段（新 VO `CreditChangeVO`）：**

| 字段 | 类型 | 说明 |
|---|---|---|
| `id` | Long | 流水 ID |
| `userId` | Long | 用户 ID |
| `projectId` | Long | 关联项目 ID（可为 null） |
| `changeType` | String | 变化类型 |
| `changeValue` | Integer | 变化值（正=加分，负=扣分） |
| `effective` | Boolean | 是否计入信誉分 |
| `suspended` | Boolean | 是否挂起（`!effective`，前端展示标记用） |
| `description` | String | 人类可读描述 |
| `createdAt` | LocalDateTime | 创建时间 |

**分页格式约定：** `list` / `total` / `page` / `pageSize`。
当前项目未配置 MyBatis-Plus 分页插件，B7 不使用 `selectPage`，而是先 `selectList(wrapper)`，再在 Controller 内做 Java 内存分页并转换为统一分页格式。

**关键约束：**
- `effective=false` 的流水也展示，`suspended=true` 标注"已挂起"（V2.1 §6.8）
- `changeType='appeal_restore'` 的恢复流水正常展示
- `changeType='penalty'` / `changeType='penalty_restore'` 仅作为 `credit_change` 只读流水展示，不代表 M5-8B 处理 M6 处罚或申诉逻辑
- 用户端响应不暴露 `sourceType` / `sourceId`，避免前台反推出 evaluationId、penaltyId 等内部记录 ID
- 内存分页只服务 P0 用户端个人信誉流水查询；若未来数据量扩大，再单独引入分页插件或 Mapper 级物理分页，不混入 M5-8B

---

## C. 拟新增文件清单

| # | 文件 | 操作 | 类型 |
|---|---|---|---|
| 1 | `src/main/java/com/teammatch/controller/CreditController.java` | **新增** | Controller |
| 2 | `src/main/java/com/teammatch/dto/CreditScoreVO.java` | **新增** | 响应 VO |
| 3 | `src/main/java/com/teammatch/dto/CreditChangeVO.java` | **新增** | 响应 VO |
| 4 | `src/main/java/com/teammatch/dto/CreditChangePageVO.java` | **新增** | 分页包装 VO |
| 5 | `src/test/java/com/teammatch/controller/CreditControllerTest.java` | **新增** | 测试 |

**不修改的文件：** `UserMapper`、`CreditChangeMapper`、`CreditChange`、`ReasonCode`、`schema.sql`、所有已有 Service、M5-8A 的 5 个文件。

---

## D. 每个文件的职责

### D1. CreditController.java

- 注解：`@RestController` + `@RequestMapping("/m5")`
- **路径口径：Controller 只写 `/m5`，完整外部路径 `/api/m5/...` 来自 `server.servlet.context-path=/api`（`application.yml` 已配置）。严禁在 Controller 中写 `/api/m5`，否则外部路径变为 `/api/api/m5/...`**
- 依赖注入：`AuthService`、`UserMapper`、`CreditChangeMapper`
- Token 解析：复用 M5-8A 模式——私有方法 `getCurrentUserId`，所有端点参数使用 `@RequestHeader(value = "Authorization", required = false) String authHeader`。**不能简写为 `@RequestHeader(required=false)`**，必须写死 `value = "Authorization"`。null/blank → `AuthenticationException(UNAUTHORIZED)`，token 解析失败 → 包装为 `AuthenticationException(UNAUTHORIZED)`
- catch 收窄为 `AuthenticationException`，Service/Mapper 调用在 try 外

| 端点 | 路径 | 核心逻辑 |
|---|---|---|
| B6 | `GET /m5/credit/score` | `userMapper.selectById(userId)` → 用户不存在时返回 `Result.fail(ReasonCode.NOT_FOUND)`，code 值以 K4 只读确认为准（当前预期为 `"M1002"`） → 构建 `CreditScoreVO` → `Result.success(vo)` |
| B7 | `GET /m5/credit/changes` | `LambdaQueryWrapper<CreditChange>` 按 `userId` + 可选 `projectId`/`changeType` + `createdAt DESC` → `creditChangeMapper.selectList(wrapper)` → 转换 `CreditChangeVO` → Java 内存分页 → `CreditChangePageVO`（每条 `suspended = !effective`，统一返回 `list/total/page/pageSize`） |

**B7 查询与内存分页逻辑：**
```text
1. LambdaQueryWrapper 必须始终包含 eq(CreditChange::getUserId, userId)
2. 可选 projectId / changeType 过滤
3. wrapper.orderByDesc(CreditChange::getCreatedAt)  // 默认排序
4. List<CreditChange> all = creditChangeMapper.selectList(wrapper)
5. List<CreditChangeVO> allVo = all.stream()
       .map(this::toCreditChangeVO)
       .collect()
6. total = allVo.size()
7. 对 allVo 做 Java 内存分页：
   - page 默认 1，page < 1 按 1
   - pageSize 默认 20，pageSize < 1 按 20，pageSize > 100 截断为 100
   - fromIndex = min((page - 1) * pageSize, total)
   - toIndex = min(fromIndex + pageSize, total)
   - list = allVo.subList(fromIndex, toIndex)
8. CreditChangePageVO result = { list, total, page, pageSize }
9. Result.success(result)
```

**effective → suspended 转换（防空）：**
```java
private CreditChangeVO toCreditChangeVO(CreditChange cc) {
    CreditChangeVO vo = new CreditChangeVO();
    // ... 其他字段映射 ...
    boolean effective = Boolean.TRUE.equals(cc.getEffective());  // 防空
    vo.setEffective(effective);
    vo.setSuspended(!effective);
    // ...
    return vo;
}
```
不使用简单的 `suspended = !cc.getEffective()`，避免 `Boolean` 包装类型拆箱 NPE。

### D2. CreditScoreVO.java

```text
包路径: com.teammatch.dto
字段:
  - Long userId
  - Integer creditScore
```

### D3. CreditChangeVO.java

```text
包路径: com.teammatch.dto
字段:
  - Long id
  - Long userId
  - Long projectId         // 可为 null
  - String changeType
  - Integer changeValue
  - Boolean effective
  - Boolean suspended      // !effective
  - String description
  - LocalDateTime createdAt
```

### D4. CreditChangePageVO.java

```text
包路径: com.teammatch.dto
字段（统一分页格式）:
  - List<CreditChangeVO> list
  - long total
  - int page
  - int pageSize
```

### D5. CreditControllerTest.java

- `@WebMvcTest(CreditController.class)` + `@MockBean`（AuthService、UserMapper、CreditChangeMapper）
- Mock 模式与 M5-8A 的 `EvaluationControllerTest` 一致
- 至少 16 个测试用例（详见 G 节）

---

## E. 需要复用的已有资源

| 资源 | 类型 | 复用位置 |
|---|---|---|
| `AuthService.getUserIdFromToken(token)` | Service | `CreditController` 鉴权 |
| `UserMapper.selectById(userId)` | Mapper (BaseMapper) | B6 查询用户信誉分 |
| `CreditChangeMapper.selectList(wrapper)` | Mapper (BaseMapper) | B7 查询当前用户信誉流水 |
| `AuthenticationException` | Exception | 鉴权失败 |
| `ReasonCode.UNAUTHORIZED` | Enum | 鉴权失败响应 |

---

## F. 需要新增的 DTO/VO

| DTO/VO | 用途 | 关键约束 |
|---|---|---|
| `CreditScoreVO` | B6 响应 | `userId` + `creditScore` |
| `CreditChangeVO` | B7 单条流水 | `suspended = !effective` |
| `CreditChangePageVO` | B7 分页包装 | `list / total / page / pageSize` |

---

## G. 测试计划

### G1. 测试框架

- `@WebMvcTest(CreditController.class)` + `@MockBean` + JUnit 5 + MockMvc + AssertJ JSONPath
- Mock 全部依赖（AuthService、UserMapper、CreditChangeMapper）

### G2. CreditControllerTest

| # | 端点 | 测试场景 | 预期 `$.code` | 关键断言 |
|---|---|---|---|---|
| A1 | B6 | 缺少 Authorization header → UNAUTHORIZED | `"M3000"` | `@RequestHeader(value="Authorization", required=false)` 保证进入 Controller |
| A2 | B7 | 缺少 Authorization header → UNAUTHORIZED | `"M3000"` | 同上 |
| A3 | B6 | 无效 token → UNAUTHORIZED | `"M3000"` | `authService.getUserIdFromToken` 抛异常 → Controller catch → M3000 |
| A4 | B7 | 无效 token → UNAUTHORIZED | `"M3000"` | 同上，不允许逃逸为 500 |
| A5 | B6 | 过期 token → UNAUTHORIZED | `"M3000"` | `authService.getUserIdFromToken` 抛 `AuthenticationException` 或 `RuntimeException` 均统一为 M3000 |
| A6 | B7 | 过期 token → UNAUTHORIZED | `"M3000"` | 同上，不允许逃逸为 500 |
| C1 | B6 | 正常返回信誉分 | `"00000"` | `$.data.userId` 非空，`$.data.creditScore` = 100 |
| C2 | B6 | 用户不存在 → `NOT_FOUND` | `"M1002"` | `userMapper.selectById` 返回 null |
| C3 | B7 | 正常返回分页流水（默认 page=1） | `"00000"` | `$.data.list` 数组，`$.data.total` ≥ 0，`$.data.page` = 1 |
| C4 | B7 | `effective=false` → `suspended=true` | `"00000"` | `$.data.list[?].suspended = true`，`$.data.list[?].effective = false` |
| C5 | B7 | `effective=true` → `suspended=false` | `"00000"` | `$.data.list[?].suspended = false` |
| C6 | B7 | 传 `changeType=evaluation` 过滤生效 | `"00000"` | 所有 `list[?].changeType` = `"evaluation"` |
| C7 | B7 | 传 `projectId` 过滤生效 | `"00000"` | 所有 `list[?].projectId` 匹配 |
| C8 | B7 | 分页字段为 `list/page` 而非 `records/current` | `"00000"` | `$.data.list` 存在，`$.data.records` 不存在 |
| C9 | B7 | `pageSize=999` → 截断，响应 pageSize=100 | `"00000"` | mock `selectList` 返回多条数据，断言 `$.data.pageSize=100` 且 `list.length() <= 100` |
| C10 | B7 | `changeType=invalid` → `PARAM_ERROR` | `"M1001"` | 非法值被拦截 |
| C11 | B7 | `page=0` / `pageSize=0` 纠正为默认值 | `"00000"` | 断言响应 `page=1`、`pageSize=20`，不抛异常 |
| C12 | B7 | B7 查询强制绑定当前 token userId | `"00000"` | 优先通过 ArgumentCaptor 捕获 wrapper 断言包含 `eq(userId)`；若 MyBatis-Plus wrapper 内部结构难以稳定断言，则降级为代码审查必看点，不能为了测试脆弱解析内部字段 |
| C13 | B7 | `changeType=penalty_restore` 过滤生效 | `"00000"` | 白名单包含 schema 已存在类型，仅做只读展示 |
| C14 | B7 | 响应不暴露 `sourceType` / `sourceId` | `"00000"` | `$.data.list[0].sourceType` 和 `$.data.list[0].sourceId` 不存在 |
| C15 | B7 | 返回结果按 createdAt DESC 排序 | `"00000"` | 优先通过 ArgumentCaptor 检查 wrapper 包含 `orderByDesc(CreditChange::getCreatedAt)`；若无法稳定断言 wrapper 内部结构，则作为代码审查必看点，不能只靠 mock 返回顺序证明 |
| C16 | B7 | 请求携带 `userId` 参数也不能查询他人流水 | `"00000"` | `/m5/credit/changes?userId=999` 仍使用 token userId 构造查询 |
| C17 | B7 | Java 内存分页 `page=2` 返回第二页 | `"00000"` | mock `selectList` 返回超过 `pageSize` 的数据，断言第二页首条不是第一页首条 |
| C18 | B7 | `total` 等于过滤后完整结果数 | `"00000"` | `total` 使用 `selectList` 返回完整列表大小，不等于当前页 `list.length()` |

---

## H. 明确不做什么

1. 不修改 `schema.sql`
2. 不修改已有 Service / Mapper（`CreditChangeMapper` 不新增方法，只用 `BaseMapper.selectList`）
3. 不写管理端接口
4. 不新增 `ReasonCode`
5. 不修改 `user.credit_score`（只读）
6. 不做 M5-8A 文件修改
7. 不写 report / penalty / Jenkins / 部署
8. **B7 不允许通过请求参数传 userId 查询他人流水**；userId 强制取自 token

---

## I. 主要风险点

| # | 风险 | 缓解 |
|---|---|---|
| R1 | `UserMapper.selectById` 返回 null 时未处理 → NPE | Controller 判空，返回 `Result.fail(ReasonCode.NOT_FOUND)` |
| R2 | Java 内存分页容易把 `total` 写成当前页数量 | `total` 必须使用过滤后完整列表大小，测试 C18 验证 |
| R3 | `CreditChange.projectId` 为 null 时 `LambdaQueryWrapper.eq` 行为 | 仅当 `projectId` 参数非空时才添加条件 |
| R4 | 当前项目无 MyBatis-Plus 分页插件，`selectPage` 物理分页不可作为依据 | B7 明确改用 `selectList + Java 内存分页`，不修改配置、不新增 Mapper 方法 |
| R5 | M5-8A 未合入时基于本地 HEAD 提交 → stacked change | 执行前 K7 检查 git 状态；必要时从 `origin/master` 新建分支 |

---

## J. 开发顺序

```
1. 执行前检查 git 状态（fetch / status / branch -vv / rev-parse / merge-base / log origin/master..HEAD / diff origin/master...HEAD），确认基线为最新 origin/master
2. 读本文档 + M5-8_PLAN.md（已熟悉则跳过）
3. 执行 K1-K12 只读确认；K5 已确认当前无分页插件，B7 按 `selectList + Java 内存分页` 执行，不得扩范围改 Mapper / SQL / 配置
4. 创建文件：
   - CreditScoreVO.java
   - CreditChangeVO.java
   - CreditChangePageVO.java
   - CreditController.java
   - CreditControllerTest.java
5. 运行 mvn -Dtest=CreditControllerTest test 确认全部通过
6. 本地启动验证 B6/B7 端点
7. Gerrit 提交 M5-8B（仅上述 5 个文件，不含 IMPLEMENT_LOG.md）
```

> **IMPLEMENT_LOG.md 不纳入本次 M5-8B Gerrit Change。** M5-8B 合入后再单独整理文档，保持本次 Change 干净。

---

## K. 执行前只读确认

| # | 需确认项 | 确认方式 | 影响 |
|---|---|---|---|
| K1 | `UserMapper.selectById` 是否由 MyBatis-Plus `BaseMapper` 自动提供 | 确认 `UserMapper extends BaseMapper<User>` | B6 无需新增 Mapper 方法 |
| K2 | `CreditChangeMapper` 继承 `BaseMapper<CreditChange>`，`selectList` 可用 | 确认类声明 | B7 只读查询 |
| K3 | `CreditChange` 实体字段名与数据库列名映射（`map-underscore-to-camel-case: true`） | `application.yml` | `LambdaQueryWrapper` 字段名风格 |
| K4 | `ReasonCode.NOT_FOUND` / `ReasonCode.PARAM_ERROR` 是否存在及其 code 值 | `read_file ReasonCode.java` | B6 用户不存在固定返回 `M1002`，B7 非法 `changeType` 固定返回 `M1001` |
| K5 | MyBatis-Plus 分页插件是否已配置（`PaginationInnerInterceptor`） | 搜索 `PaginationInnerInterceptor` 或 `MybatisPlusInterceptor` 配置类 | 当前确认无分页插件；M5-8B 采用 `selectList + Java 内存分页`，不修改配置、不新增 Mapper 方法 |
| K6 | `CreditChange.effective` 的 Java 类型是 `Boolean` 还是 `Integer` | `read_file CreditChange.java` | `effective → suspended` 转换方式（`Boolean.TRUE.equals()` vs `== 1`） |
| K7 | git 状态：当前 HEAD 是否为 origin/master（不含未合入的 M5-8A commit） | `git rev-parse HEAD`、`git merge-base HEAD origin/master`、`git log origin/master..HEAD --oneline`、`git diff --name-status origin/master...HEAD` | 避免形成 stacked change |
| K8 | `AuthService.getUserIdFromToken` 当前抛出的异常类型（`AuthenticationException` / `RuntimeException`）以及是否 strip `Bearer ` 前缀 | `read_file AuthServiceImpl.java` 的 `getUserIdFromToken` 方法 | Controller `getCurrentUserId` 必须把 null/blank/无效/过期统一转换为 `UNAUTHORIZED` |
| K9 | `User` 实体 `getCreditScore()` getter 是否存在 | `read_file User.java` | B6 读 `credit_score` |
| K10 | `CreditChange` 实体所有 getter 方法名（`getUserId` / `getProjectId` / `getChangeType` / `getChangeValue` / `getEffective` / `getDescription` / `getCreatedAt`） | `read_file CreditChange.java` | `LambdaQueryWrapper` 方法引用和 VO 映射；用户端 VO 不映射 `sourceType/sourceId` |
| K11 | docs/、.ai/ 未跟踪文件存在，确认 `git add` 时不被误带入 | `git status --short` | Gerrit Change 只包含 5 个目标文件 |
| K12 | `credit_change.change_type` 当前枚举是否包含 `penalty_restore` | 对照 `schema.sql`、`CreditChange.java` 注释、`CreditChangeType` 常量 | B7 白名单必须以 K12 的真实确认结果为准；若真实代码/DDL 不含 `penalty_restore`，不得在 M5-8B 中自行新增或假定存在 |
