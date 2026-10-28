# AI_CONTEXT_INDEX.md

## 0. AI 协作角色约束

进入仓库后必须先读取本文件，再根据任务场景决定是否读取其他文件。

### AI 协作分工

1. **ChatGPT** 只负责业务理解、任务拆解、边界确认、流程规划和 prompt 设计，不直接修改仓库代码。
2. **Claude Code** 负责读取本地仓库、执行代码实现、编写测试、解释 diff，但必须先列出拟修改文件和修改理由，等待用户确认后再动手。
3. **Claude 主编会话** 负责主线编码，是长期主线施工会话，不是临时讲解会话。
4. **临时 Claude/Codex 对话** 只负责单文件讲解或只读审查，不负责主线编码。
5. **Codex** 负责只读审查，重点检查是否违反设计文档、schema.sql、模块边界、测试要求和 Gerrit 提交规范。
6. **用户本人** 负责最终决策、检查 git diff、运行测试、执行 git add / git commit / git push。

### 硬约束

5. Codex / Claude Code 不得执行任何 Git 写操作，包括 git add、git commit、git push、git reset、git clean、git checkout、git rebase、git merge、git restore、git fetch。
6. 后续开发必须小步推进，每次只做一个明确的小业务闭环。
7. 写业务代码时必须同步写 JUnit 测试。
8. 未经用户明确确认，不得创建 Entity、Mapper、Service、Controller、Test 等业务文件。
9. 未经用户明确确认，不得创建 CLAUDE.md、.claude/、docs/、AI workflow、Review Guide、任务 CSV 等个人工作流文件。
10. 不得修改 sql/schema.sql，除非用户明确说明当前任务就是修改数据库基线。
11. M5 相关开发不得越界实现 M3/M4/M6 职责。
12. M5-1 的范围只是互评资格校验，不等于完整互评提交流程。

## 1. 快速启动（必读）

你是 TeamMatch 后端 AI。

默认只读：

```text
AI_CONTEXT_INDEX.md
AGENTS.md / CLAUDE.md（按当前工具选择）
docs/design/system-design-v2.1.md（至少读取当前任务相关章节；M5 首次处理需读 0.3、0.4、6、8、9.1）
M5_WORK_ORDER.md (如果当前任务是 M5 开发)
docs/m5/M5_CONTEXT.md、docs/m5/M5_TODO.md (如果当前任务是 M5 开发)
```

不要默认读取 CSV、docs/archive、database 全量材料、REVIEW_GUIDE 或全部 docs。
需要时按本文件第 4 节路由读取。

## 1.1 当前 M5 接手状态

> ✅ M5-1 到 M5-7 的 Service 层已全部完成（资格校验、内容校验、信誉计算、异常检测、提交编排、评价复核、申诉恢复）。
> ⚠️ Controller 层待补。
> 详情见 `IMPLEMENT_LOG.md` 和 `docs/m5/M5_TODO.md`。
>
> 历史完成记录：

- M5-1：互评资格校验
  - 代码落点：EvaluationEligibilityService
  - 测试落点：EvaluationEligibilityServiceTest

- M5-2：互评提交内容基础校验
  - 代码落点：EvaluationValidationService
  - 测试落点：EvaluationValidationServiceTest

- M5-3：信誉分映射与单项目封顶计算
  - 代码落点：CreditRuleService / CreditCalculationService
  - 测试落点：CreditRuleServiceImplTest / CreditCalculationServiceTest

下一步只允许继续小步推进 M5-4。

M5-4 开始前，不允许直接生成代码。必须先向用户说明：

1. M5-4 的业务目标；
2. 与 V2.1 的对应规则；
3. 预计修改文件；
4. 测试点；
5. 是否涉及事务、reasonCode、credit_change、user.credit_score；
6. 然后再生成施工 prompt，等待用户确认。

## 1.2 本地上下文文件提交规则

默认不要提交以下本地工作台文件，除非用户明确要求：

- AI_CONTEXT_INDEX.md
- AGENTS.md
- CLAUDE.md
- M5_WORK_ORDER.md
- M5_3_PLAN.md
- docs/m5/*.md
- .claude/**
- .vscode/**

Codex / Claude 不得执行 Git 写操作，包括：

- git add
- git commit
- git push
- git reset
- git clean
- git checkout
- git rebase
- git merge
- git restore
- git fetch

Git 提交、暂存、推送、分支切换、合并、变基、工作区清理只能由用户本人执行。

## 2. 文件定位

本文件是 TeamMatch 后端仓库的低 token 上下文入口索引。

本文件只负责说明：

- 默认读取哪些最小文件
- 不同任务场景下按需读取哪些文件
- 哪些文件禁止默认读取
- M5 后端开发边界
- AI 修改代码时的工作规则

本文件不是业务权威文件。业务规则以 docs/design/system-design-v2.1.md 为准；字段落地以 sql/schema.sql 和实际代码校验。若三者冲突，停止实现并先对齐。

## 3. 默认读取文件

每次进入后端仓库新会话时，默认只读取：

```text
AI_CONTEXT_INDEX.md
AGENTS.md / CLAUDE.md（按当前工具选择）
docs/design/system-design-v2.1.md（至少读取当前任务相关章节）
```

如果当前任务是 M5 开发，再读取：

```text
M5_WORK_ORDER.md
docs/m5/M5_CONTEXT.md
docs/m5/M5_TODO.md
```

默认读取目标：

- 确认上下文读取规则
- 避免一次性加载全部文档
- 按任务场景再决定是否读取其他文件

## 4. 按场景读取路由

### 接口设计 / 接口联调

触发条件：

- 新增或修改 Controller
- 修改请求参数、响应结构、分页格式
- 修改 HTTP 状态码或业务错误码
- 与前端进行接口联调
- 处理 `reasonCode`

按需读取：

```text
README.md (查看接口文档说明)
sql/schema.sql (确认数据库字段)
```

### 数据库 / DDL / 字段确认

触发条件：

- 新增或修改表
- 新增或修改 Entity / Mapper
- 确认字段名、状态枚举、唯一约束、索引
- 确认数据库建表顺序
- 确认信誉流水、评价、申诉相关字段

按需读取：

```text
sql/schema.sql
```

### 互评 / 信誉分 / 信誉流水

触发条件：

- 互评资格检查
- 提交互评
- 四维评分计算
- 单项目信誉分封顶
- 信誉变化流水写入
- 用户信誉分缓存回写
- 异常评价挂起

按需读取：

```text
M5_WORK_ORDER.md
docs/m5/M5_CONTEXT.md
docs/m5/M5_TODO.md
sql/schema.sql
```

### 测试 / 质量保证 / 证据链

触发条件：

- 新增或修改核心业务逻辑
- 新增或修改测试
- 需要补测试证据
- 需要确认测试要求

按需读取：

```text
M5_WORK_ORDER.md (查看测试要求)
docs/m5/M5_TEST_PLAN.md (查看 M5 测试计划)
README.md (查看测试规范)
```

## 5. 禁止默认读取

以下路径禁止默认读取，也禁止纳入日常上下文：

```text
.git/**
.claude/**
.vscode/**
node_modules/**
dist/**
target/**
.env*
application-local.yml
*.log
```

## 6. AI 工作规则

后端 AI 开发必须遵守：

1. 先列计划，再改文件。
2. 先说明本次任务目标、涉及文件、测试点和不确定项。
3. 小步修改，只做当前任务直接相关内容。
4. 不跨模块扩张。
5. 不静默修改冻结规则。
6. 不执行任何 Git 写操作；Git 提交、暂存、推送、分支切换、合并、变基、工作区清理只能由用户本人执行。
7. 不读取或修改 `docs/archive/**`。
8. 不自行引入 P1 能力。
9. 不绕开信誉流水修改信誉分。
10. 不把自由文案当作前后端业务判断依据，必须使用稳定 `reasonCode`。
11. 修改后列出：
    - 影响文件
    - 行为变化
    - 测试建议
    - 证据链位置
    - 未验证风险
12. 信息不足时标注“信息不足”，不要自行假设。
13. 遇到跨模块依赖时，先说明依赖方和对齐点，不直接替其他模块实现。

## 7. 单文件注释 Prompt 机制

对于启用本机制后新建或实质修改的关键业务文件：

- 主编 Claude 每创建或实质修改一个关键业务文件后，可以在该文件末尾以代码注释形式追加“单文件讲解 / 审查 Prompt”。
- 该 Prompt 由主编 Claude 创建，用户复制该文件末尾的注释 Prompt 到新的临时 Claude/Codex 对话中使用。
- 临时 Claude/Codex 对话不得推进新任务，不得修改代码，不得扩展到其他模块，不得执行 Git 操作，只围绕该文件做讲解或只读审查。
- 文件末尾仍保留“单文件讲解 / 审查 Prompt”注释，表示该文件仍处于待讲解 / 待审查状态。
- 文件末尾没有该注释 Prompt，表示该文件已经完成临时讲解 / 审查，并且注释 Prompt 已被清理。
- 主编 Claude 不应对已经清理 Prompt 的文件重复生成讲解 Prompt，除非该文件后来又发生实质修改。

该判断只适用于启用本机制之后新建或实质修改的关键文件。对历史已有文件，不能简单根据“没有注释 Prompt”判断它已经被讲解 / 审查过。

整轮提交前仍必须确保所有 AI Prompt 注释残留都被删除。Codex 整轮审查时需要检查是否残留“单文件讲解 / 审查 Prompt”“AI Prompt”“Claude/Codex”等非业务注释；如果没有残留，且对应文件是本轮按机制处理过的文件，则视为该文件已经完成讲解 / 审查流程。

## 8. M5 特别边界

M5 负责：

```text
evaluation
evaluation_tag
skill_endorse
credit_change
evaluation 类 appeal
评价复核
申诉恢复
信誉分计算
互评资格
互评提交
异常评价挂起
M5 单元测试
M5 接口测试
M5 边界测试
```

M5 不负责：

```text
report
penalty
治理后台入口
Jenkins
部署
用户认证
Token 签发
学校邮箱认证
正式档案建立
项目创建
项目状态流转
team_request
team_member 建立
exit_vote 发起与关闭主流程
```

M5 可以读取其他模块输出或调用其他模块服务，但不得在 M5 任务中扩张修改 M3 / M4 / M6 代码。

涉及以下边界时必须先标注跨模块对齐：

- M4 退出扣分写入信誉流水
- M6 处罚扣分写入信誉流水
- M6 评价复核管理端入口调用 M5 复核逻辑
- M6 申诉处理入口调用 M5 申诉恢复逻辑
- M2 前端互评、信誉、申诉页面依赖 M5 接口
- M3 用户状态或封禁状态影响互评资格判断

