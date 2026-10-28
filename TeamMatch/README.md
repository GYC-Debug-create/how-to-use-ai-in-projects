# TeamMatch

面向校园场景的组队与互评平台，有微信小程序用户端和 Web 管理后台。

仓库：
- 后端：https://github.com/GYC-Debug-create/teammatch-backend
- 前端：https://github.com/GYC-Debug-create/teammatch-frontend

这是团队项目。我负责 M5 模块（互评与信誉）的后端开发（`backend/M5_WORK_ORDER.md` 第 8 行）。这里收录的 AI 文件都是我写的，组员写的部分已单独标出。

这些 AI 文件大多只放在我本地，没有提交到团队仓库。比如 `frontend/.ai/FRONTEND_REVIEW_CONTEXT.md` 第 7 行写明「不提交 Gerrit」；AI 也不能自己建这类工作流文件，要等我确认（`backend/AI_CONTEXT_INDEX.md` 第 22 行）。

## 我是怎么规划 AI 的

### 分工

| 谁 | 干什么 | 不许干什么 | 依据 |
|---|---|---|---|
| ChatGPT | 理解业务、拆任务、确认边界、规划流程、设计 prompt | 不直接改仓库代码 | `backend/AI_CONTEXT_INDEX.md` 第 9 行 |
| Claude Code（主编会话） | 读仓库、写代码、写测试、解释 diff；长期负责主线编码 | 动手前必须先列出要改的文件和理由，等我确认 | 同上，第 10–11 行 |
| 临时 Claude / Codex 对话 | 讲解单个文件，或只读审查 | 不负责主线编码 | 同上，第 12 行 |
| Codex | 只读审查：查有没有违反设计文档、schema、模块边界、测试要求和 Gerrit 规范 | 不写代码 | 同上，第 13 行；`backend/M5_WORK_ORDER.md` 第 7 行 |
| DeepSeek-V4-Pro | 审查前端提交 | — | `frontend/.ai/FRONTEND_REVIEW_CONTEXT.md` 第 5 行；`frontend/.ai/report_DS.md` 标题 |
| 我 | 最终决策、看 git diff、跑测试、git add / commit / push | — | `backend/AI_CONTEXT_INDEX.md` 第 14 行 |

所有 AI 都不准执行任何 Git 写操作（`backend/AI_CONTEXT_INDEX.md` 第 18 行）。

入口文件也兼容 Cursor 和 Kiro（`backend/AGENTS.md` 第 3 行），但文件里没有写它们具体分到了什么活。

### 流程

| 环节 | 我怎么安排 AI | 依据 |
|---|---|---|
| 1. 统一口径 | 资料有新有旧，规定所有 AI 先读「读取说明」，冲突一律以设计文档 V2.1 为准，不许自己折中 | `requirements/00_AI读取说明_最新口径优先级.md` 第 3、65–73 行 |
| 2. 进门路由 | 一个入口文件按任务类型写死读取路径：开发、审查、其他任务各走各的；Claude Code 另有轻量入口 | `backend/AGENTS.md` 第 5–70 行；`backend/CLAUDE.md` |
| 3. 工作令约束执行 | 不越界、不黑箱、不一次性大生成；每写完一个文件要解释，要等我确认才继续 | `backend/M5_WORK_ORDER.md` 第 9、153–155、175、347、360 行 |
| 4. 拆任务、省上下文 | 每次只做一个小任务；给 M5 单独做一个低 token 的默认入口，规定哪些资料默认不读 | `backend/docs/m5/M5_TODO.md` 第 5 行；`backend/docs/m5/M5_CONTEXT.md` 第 3、43 行 |
| 5. 计划先审再做 | 实施计划过两轮 Codex 审查才动手；执行前先做只读确认，按确认结果改方案 | `backend/docs/m5/M5-8_PLAN.md` 第 3 行；`backend/docs/m5/M5-8B_PLAN.md` 第 8、338 行 |
| 6. 方案先核对代码 | 推荐算法方案列出「AI 实现前必须验证」的前提，再出一份编译审查修正版 | `design/recommendation-plan.md` 第 401 行；`design/recommendation-plan-audit.md` 第 5–7 行 |
| 7. 审查后返工 | 每个提交都留审查记录，再合并成一份修复计划，按计划改 | `backend/docs/m5/M5-1_TO_M5-4_FIX_PLAN.md` 第 3–13 行 |
| 8. 终审交给 Codex | 先做一轮实现审查，判断能不能交给 Codex 终审 | `backend/docs/m5/M5-9_REVIEW_REPORT.md` 第 55–57 行 |
| 9. 审查标准 | 只卡公共契约、安全和冻结规则，其余记录后补；回复固定三段式；Review 用组长口吻 | `backend/AGENTS.md` 第 185–323、403–425 行 |
| 10. 多 AI 交叉审查 | 第一轮各审各的，互不告知结论；第二轮只抛具体分歧点；最后由我按裁量规则表裁定 | `backend/AGENTS.md` 第 325–393 行 |
| 11. 审查 SOP | 把 AI 审 Gerrit 时踩过的坑写成固定步骤，省时间和 token | `backend/docs/review/gerrit-review-rules.md` 第 3 行；`backend/AGENTS.md` 第 44–64 行 |
| 12. 前端审查 | 给审前端的 AI 单独写入口文件；DeepSeek 出第一轮报告，并验证上一轮审查意见改没改 | `frontend/.ai/FRONTEND_REVIEW_CONTEXT.md` 第 5–7 行；`frontend/.ai/report_DS.md` 第 22 行 |
| 13. 自主验收 | 一句话启动：AI 先把前后端复制到临时目录，原项目只读，自己理解功能、生成并执行测试，最后交覆盖矩阵和 Bug 证据 | `one_click_qa/START_PROMPT.txt`；`one_click_qa/CLAUDE.md` 第 16–29 行 |
| 14. 留开发日志 | 规定 AI 做完后追加开发日志 | `backend/AGENTS.md` 第 172 行；`backend/IMPLEMENT_LOG.md` |

## 文件清单

| 文件 | 用途 | 为什么有 |
|---|---|---|
| `backend/AGENTS.md` | 各家 AI 的统一入口：读取路由、冻结规则、审查标准、交叉审查协议 | （待补） |
| `backend/CLAUDE.md` | Claude Code 的轻量入口 | （待补） |
| `backend/AI_CONTEXT_INDEX.md` | AI 分工和硬约束 | （待补） |
| `backend/M5_WORK_ORDER.md` | M5 工作令：约束 Claude Code 怎么写代码 | （待补） |
| `backend/docs/m5/M5_CONTEXT.md` | M5 低 token 上下文入口 | （待补） |
| `backend/docs/m5/M5_TODO.md` | M5 任务拆解 | （待补） |
| `backend/docs/m5/M5_TEST_PLAN.md` | M5 测试计划 | （待补） |
| `backend/docs/m5/M5-3_PLAN.md` | M5-3 开发规划 | （待补） |
| `backend/docs/m5/M5-8_PLAN.md` | M5-8 实施规划（Codex 审查修正版） | （待补） |
| `backend/docs/m5/M5-8B_PLAN.md` | M5-8B 实施规划（含执行前只读确认） | （待补） |
| `backend/docs/m5/M5-9_IMPLEMENT_PLAN_V1.1.md` | M5-9 实施规划 | （待补） |
| `backend/docs/m5/M5-1_REVIEW_ISSUES.md` | M5-1 提交审查记录 | （待补） |
| `backend/docs/m5/M5-2_REVIEW_ISSUES.md` | M5-2 提交审查记录 | （待补） |
| `backend/docs/m5/M5-3_REVIEW_ISSUES.md` | M5-3 提交审查记录 | （待补） |
| `backend/docs/m5/M5-4_REVIEW_ISSUES.md` | M5-4 提交审查记录 | （待补） |
| `backend/docs/m5/M5-1_TO_M5-4_FIX_PLAN.md` | 四份审查记录合并成的修复计划 | （待补） |
| `backend/docs/m5/M5-9_REVIEW_REPORT.md` | M5-9 实现审查报告 | （待补） |
| `backend/docs/review/gerrit-review-rules.md` | AI 审 Gerrit 的操作规则 | （待补） |
| `backend/IMPLEMENT_LOG.md` | 后端开发日志 | （待补） |
| `frontend/.ai/FRONTEND_REVIEW_CONTEXT.md` | 前端 AI 审查入口 | （待补） |
| `frontend/.ai/report_DS.md` | DeepSeek 第一轮前端审查报告 | （待补） |
| `frontend/.ai/T33_REMEDIATION.md` | T-33 前端整改清单 | （待补） |
| `one_click_qa/CLAUDE.md` | 自主验收的启动规则 | （待补） |
| `one_click_qa/QA_AUTOTEST.md` | 自主验收的完整要求 | （待补） |
| `one_click_qa/START_PROMPT.txt` | 自主验收的启动语 | （待补） |
| `requirements/00_AI读取说明_最新口径优先级.md` | 告诉 AI 新旧资料怎么取舍 | （待补） |
| `design/recommendation-plan.md` | 推荐算法方案 | （待补） |
| `design/recommendation-plan-audit.md` | 推荐算法方案的编译审查修正版 | （待补） |

## 来源与脱敏

| 文件 | 原始位置 | 脱敏 |
|---|---|---|
| `backend/AGENTS.md` | 本地后端开发副本 `shixun12_backend/AGENTS.md`（没进团队仓库）。只收录前 464 行规则正文，第 465 行起是误粘贴的 3 份重复审查输出，没有收录 | Gerrit 服务地址 1 处 |
| `backend/CLAUDE.md`、`AI_CONTEXT_INDEX.md`、`M5_WORK_ORDER.md` | 本地后端开发副本根目录（没进团队仓库） | 无 |
| `backend/docs/m5/` 下 13 个文件 | 本地后端开发副本 `docs/m5/`（没进团队仓库）。M5-1～4 的审查记录对应公开仓库里的修复提交 `49a00a3`、`e0bdc52`、`bf4556c`、`cf86fa0` | 无 |
| `backend/docs/review/gerrit-review-rules.md` | 本地后端开发副本 `docs/review/`（没进团队仓库） | Gerrit 服务地址 1 处 |
| `backend/IMPLEMENT_LOG.md` | teammatch-backend 仓库根目录。第 1–2、148–252、254–277 行是我写的；**第 3–147 行和第 253 行组员负责**（按 git blame） | Gerrit 服务地址 1 处 |
| `frontend/.ai/` 下 3 个文件 | 本地前端开发副本 `shixun12_frontend/.ai/`（没进团队仓库） | `FRONTEND_REVIEW_CONTEXT.md`：学校 1 处、与开发无关的个人用途句子 1 句；`report_DS.md`：小程序 AppID 1 处；`T33_REMEDIATION.md`：本地绝对路径 1 处、小程序 AppID 2 处 |
| `one_click_qa/` 下 3 个文件 | 本地项目资料 `06_测试与用户手册/one_click_qa/` | 无 |
| `requirements/00_AI读取说明_最新口径优先级.md` | 本地项目资料 `02_需求/转换稿_md/` | 无 |
| `design/` 下 2 个文件 | 本地项目资料 `03_详细设计/推荐方案/`。对应公开仓库提交 `3901601` | 无 |

脱敏的地方都写成了「（已脱敏）」。`127.0.0.1` 和 `localhost` 是本机地址，保留了。
