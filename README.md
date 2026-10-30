# how-to-use-ai-in-projects

这个仓库展示我在自己的项目里怎么用 AI 干活：怎么分工、怎么下任务、怎么审 AI 的产出。

每个项目一个文件夹。里面放的是那个项目里真实用过的 AI 文件：规则、prompt、任务书、计划、审查记录。原文保留，只做了脱敏。每个文件夹都有一份 README，说明流程，每一条都标了依据在哪个文件、哪一行。

## 项目目录

| 项目 | 一句话 | 文件夹 |
|---|---|---|
| douluo-dalu | 我独立完成的文字 RPG 网页游戏；以 Vibe Coding 方式开发，使用多个 AI 协作、交叉审查和迭代 | [douluo-dalu/](douluo-dalu/) |
| doudizhu | 我独立完成的斗地主 / 掼蛋 H5 游戏；以 Vibe Coding 方式迭代引擎、AI 对手、界面与音效 | [doudizhu/](doudizhu/) |
| TeamMatch | 校园组队与互评平台（团队项目，我负责 M5 后端）。按角色给多个 AI 分工，实施计划先审再做 | [TeamMatch/](TeamMatch/) |
| mini-tik-tok | 简易版抖音（团队项目，我负责前端）。一套让 AI 带我写前端的规则 | [mini-tik-tok/](mini-tik-tok/) |

斗地主仓库没有保留 Prompt 或对话原文；对应过程说明只依据公开 README、源码结构和 GitHub 提交历史整理，不补写未留档的对话或模型分工。

## 几个项目里共同的做法

只列文件里写了的。「—」表示这个项目没有这方面的文件。斗地主列的阶段来自提交历史，仅表示仓库可见的实现顺序，不推断未留存的对话内容。

| 做法 | douluo-dalu | doudizhu | TeamMatch | mini-tik-tok |
|---|---|---|---|---|
| 给 AI 写入口，规定先读什么、按什么顺序读 | `docs/prompts/AI激活提示词.md` 第 7–10 行 | —（未留存） | `backend/AGENTS.md` 第 5–70 行；`requirements/00_AI读取说明_最新口径优先级.md` 第 75–86 行 | `SKILL.md` 第 23–30 行 |
| 写死 AI 不许做的事，尤其不许 Git 提交 | `docs/prompts/AI激活提示词.md` 第 18–22 行 | —（未留存） | `backend/AI_CONTEXT_INDEX.md` 第 18 行 | `SKILL.md` 第 47 行 |
| 小步推进，一次只做一个能验证的闭环 | `docs/任务书-第一轮审查修复.md` 第 12–13 行 | 提交历史按牌组、牌型、规则引擎、AI、应用层、界面和音效等阶段记录；具体任务卡未留存 | `backend/docs/m5/M5_TODO.md` 第 5 行；`backend/M5_WORK_ORDER.md` 第 153–155 行 | `SKILL.md` 第 36–39 行 |
| 多个 AI 分工，划清边界 | `docs/任务书-KimiK3-前端界面美化.md` 第 3–9 行；`docs/reports/统一标尺审查报告-flash.md` 第 5–7 行 | —（提交可见 Claude 参与，但详细分工未留存） | `backend/AI_CONTEXT_INDEX.md` 第 9–13 行 | — |
| 让 AI 互相审查，分歧由我裁定 | `docs/reports/第一轮审查定稿.md` 第 4–5、109–116 行 | —（未留存） | `backend/AGENTS.md` 第 325–393 行 | — |
| 关键决定和最终提交由我来做 | `docs/任务书-第二批执行-GLM.md` 第 5–9 行 | 你确认项目由你独立完成；GitHub 提交历史显示 GYC-Debug-create 与 Claude 共同署名 | `backend/AI_CONTEXT_INDEX.md` 第 14 行；`backend/M5_WORK_ORDER.md` 第 347、360 行 | `FRONTEND_PROGRESS.md` 第 28 行（验证由我手动执行） |
| 要求 AI 先核对现状、不准伪造结果 | `docs/prompts/AI激活提示词.md` 第 9、22 行；`docs/prompts/prompt.md` 第 17 行 | —（未留存） | `design/recommendation-plan.md` 第 401 行；`backend/docs/m5/M5-8B_PLAN.md` 第 338 行 | `SKILL.md` 第 49–55 行 |
| 控制上下文和成本 | `docs/任务书-KimiK3-前端界面美化.md` 第 19–24 行 | —（未留存） | `backend/docs/m5/M5_CONTEXT.md` 第 3、43 行 | `FRONTEND_PROGRESS.md` 第 16 行 |

## 脱敏说明

- 删掉的内容都写成「（已脱敏）」，包括本地绝对路径、账号（存档 ID、小程序 AppID）、Gerrit 服务地址、学校、与开发无关的个人用途句子。
- `127.0.0.1` 和 `localhost` 是本机地址，保留了。
- 除了脱敏，只有一处删减：TeamMatch 的 `backend/AGENTS.md` 只收录了规则正文，原因写在 [TeamMatch/README.md](TeamMatch/README.md)。
