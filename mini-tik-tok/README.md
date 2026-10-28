# mini-tik-tok

课程项目：视频流推荐系统，一个简易版抖音。

仓库：https://github.com/GYC-Debug-create/mini-tik-tok

这是团队项目。我是前端负责人（`FRONTEND_PROGRESS.md` 第 6 行）。这里的 5 个文件都是我写的，用来让 AI 带我写前端。

## 我是怎么规划 AI 的

### 出发点

我后端基础好，前端和 JS 弱。所以先把「我是谁、哪些东西不能默认我懂」写给 AI（`SKILL.md` 第 8–21 行）。

### 每轮对话怎么接上

| 做法 | 依据 |
|---|---|
| 每次开新对话，AI 按固定顺序读 4 个文件：规则 → 进度 → 当前任务 → 讲解规则 | `SKILL.md` 第 23–30 行；`FRONTEND_PROGRESS.md` 第 9–14 行 |
| 优化记录默认不读，只在阶段收尾、重构或我要求时才读 | `FRONTEND_PROGRESS.md` 第 16 行 |
| 当前任务单独一张卡：目标、涉及文件、验收标准、禁止事项 | `CURRENT_TASK.md` 第 3–9、25、76、83–90 行 |
| 每个阶段用同一个输出模板 | `FRONTEND_PROGRESS.md` 第 47–58 行 |

### AI 和我怎么分工

| 轨道 | 谁主导 | 依据 |
|---|---|---|
| 写代码 | AI 辅助 | `FRONTEND_PROGRESS.md` 第 26 行 |
| 讲解 | AI 讲，我理解 | 同上，第 27 行 |
| 验证 | 我手动执行（浏览器 Network、控制台、页面操作） | 同上，第 28 行 |
| 边界 | AI 标出哪些是前端做的、哪些不能夸大 | 同上，第 30 行 |

### 给 AI 的硬规则

| 规则 | 依据 |
|---|---|
| 小步推进，每一步页面都要能运行、能验证；不准提交会白屏的中间态 | `SKILL.md` 第 36–38 行 |
| 一轮只改一层，不准同时改 UI、API、Router、状态管理 | `SKILL.md` 第 39 行；`CURRENT_TASK.md` 第 88 行 |
| 改之前说清：改哪个文件、为什么、只做什么、浏览器应该看到什么 | `SKILL.md` 第 49–55 行 |
| 改之后说清：改了什么、代码从哪一行流到哪一行、怎么验证 | `SKILL.md` 第 57–63 行 |
| 讲代码先逐行讲当前代码，再讲概念，不先讲大段抽象概念 | `TEACHING_RULES.md` 第 3–15 行；`SKILL.md` 第 44 行 |
| 不执行任何 git add / commit / push | `SKILL.md` 第 47 行 |
| 不一次性生成全部代码 | `FRONTEND_PROGRESS.md` 第 374 行 |

### 用的是什么工具

这 5 个文件里没有写具体工具名。`SKILL.md` 用的是 skill 格式（第 1–4 行有 name 和 description）。

## 文件清单

| 文件 | 用途 | 为什么有 |
|---|---|---|
| `SKILL.md` | 前端 AI 协作规则 | （待补） |
| `TEACHING_RULES.md` | AI 怎么给我讲代码 | （待补） |
| `FRONTEND_PROGRESS.md` | 跨对话的进度记录和分工 | （待补） |
| `CURRENT_TASK.md` | 当前阶段的任务卡 | （待补） |
| `FUTURE_OPTIMIZATIONS.md` | 延后的优化项，AI 默认不读 | （待补） |

## 来源与脱敏

| 文件 | 原始位置 | 脱敏 |
|---|---|---|
| `SKILL.md` | 仓库根目录，提交 `bac570d` | 个人用途的句子 1 句（与开发无关） |
| `TEACHING_RULES.md` | 仓库根目录，提交 `bac570d` | 个人用途的句子 2 句（与开发无关） |
| `FRONTEND_PROGRESS.md` | 仓库根目录，提交 `bac570d` | 个人用途的句子 13 句（与开发无关，含五轨道表里的 1 行） |
| `CURRENT_TASK.md` | 仓库根目录，提交 `bac570d` | 无 |
| `FUTURE_OPTIMIZATIONS.md` | 仓库根目录，提交 `bac570d` | 无 |

脱敏的地方都写成了「（已脱敏）」。
