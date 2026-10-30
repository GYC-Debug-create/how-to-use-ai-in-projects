# douluo-dalu

斗罗大陆主题的文字 RPG 网页游戏《斗罗大陆·武魂觉醒》。

仓库：https://github.com/GYC-Debug-create/douluo-dalu

## 在线体验

- 网页版：项目提供可直接游玩的在线版本。进入后可自行注册账号并创建独立存档；目前没有公开共用的测试账号。
- 本地运行：见下方「运行」说明。

## 项目与个人贡献

这是我独立完成的个人项目，使用 Vibe Coding 方式从想法推进到可运行产品。我负责产品方向、需求拆解、方案取舍、AI 任务设计、结果审查与验收；AI 参与代码实现、分析、测试和迭代。项目并非团队共同开发。

本仓库也记录了开发中真实使用的 AI 协作文件，原文保留并做了脱敏。下文列出的流程与分工均可在对应记录中核对。

## 我是怎么规划 AI 的

### 用了哪些 AI

| AI | 干什么 | 依据 |
|---|---|---|
| flash | 写立场稿、做审查、起草任务书、执行第二到四批修复、部署 | `docs/reports/统一标尺审查报告-flash.md` 第 5–7 行；`docs/reports/第一轮修复复核报告-GLM.md` 第 3 行；`docs/任务书-无下滑改造-GLM.md` 第 3 行 |
| GLM5.3 | 写立场稿、做审查、审定任务书、执行第一批修复、复核 | `docs/reports/玩家预期与剧情逻辑标准-GLM.md` 第 4–6 行；`docs/reports/第一轮修复复核报告-GLM.md` 第 3–4 行 |
| Kimi（k3） | 只做前端界面美化 | `docs/任务书-KimiK3-前端界面美化.md` 第 1–3 行；`docs/prompts/kimi_game_prompt.md`、`kimi_prompt.md` |
| Trae | 先出计划再动手 | 两份计划都在 `.trae/documents/` 下 |
| 主会话（文件里没写是哪个工具） | 核心逻辑、数据、后端、测试 | `docs/任务书-核心逻辑修复.md` 第 1、5 行 |

### 流程一：写 prompt 下任务

| 做法 | 依据 |
|---|---|
| 两层 prompt：一段短的「激活提示词」，让 AI 先完整读完长的主指令，再读主指令列出的资料 | `docs/prompts/AI激活提示词.md` 第 7–10 行 |
| 主指令写清角色、范围、优先级、禁止事项、验收标准、测试要求、文档回写 | `docs/prompts/AI激活提示词.md` 第 10 行；`docs/prompts/完整BUG修复提示词.md` 第 7–13 行 |
| 写死 AI 不许做的事：不开子代理、不转交别的 AI、不 Git 提交、不伪造测试结果 | `docs/prompts/AI激活提示词.md` 第 18–22 行 |
| 问题文档可能比代码旧，要求 AI 先核对问题是否还存在，再动手 | `docs/prompts/AI激活提示词.md` 第 9 行 |
| 先在浏览器实测确认问题，再交给 AI 直接改；没实际验证的不准写「已完成」 | `docs/prompts/prompt.md` 第 3、17 行 |
| 每条 BUG 都带验收条件（分辨率、浏览器、缩放比例） | `docs/prompts/完整BUG修复提示词.md` 第 70、76、88 行等 |
| 迁移 prompt 交给另一个 AI 之前，先让 AI 只读核对里面的行数和数据，找出 5 处偏差再修正 | `.trae/documents/迁移prompt文档生成计划.md` 第 5–9、80 行 |

### 流程二：两个 AI 并行干活，先划清边界

| 做法 | 依据 |
|---|---|
| 任务书 A 归主会话（核心逻辑），任务书 B 归 Kimi k3（界面） | `docs/任务书-核心逻辑修复.md` 第 5 行；`docs/任务书-KimiK3-前端界面美化.md` 第 3 行 |
| 给 Kimi 列出禁改文件和允许改的文件，防止和主会话冲突 | `docs/任务书-KimiK3-前端界面美化.md` 第 6–8 行 |
| Kimi 要加数据字段，就写进任务书 A，由主会话来加 | `docs/任务书-KimiK3-前端界面美化.md` 第 9 行 |
| 不让 Kimi 跑测试，避免读到主会话改了一半的代码 | `docs/任务书-KimiK3-前端界面美化.md` 第 11 行 |
| 控制积分：只授权第一批，做完立即报告，第二批要等授权 | `docs/任务书-KimiK3-前端界面美化.md` 第 19–24、28 行 |

### 流程三：两个模型互审，我来拍板

一轮产品审查走完了下面 8 步：

| 步骤 | 谁做 | 依据 |
|---|---|---|
| 1. 各写一份立场稿：不看代码，先从玩家角度定标准 | GLM5.3、flash 各一份 | `docs/reports/玩家预期与剧情逻辑标准-GLM.md` 第 4–6 行；`docs/reports/玩家预期与剧情逻辑标准-flash.md` 第 3 行 |
| 2. 两份合成一把「统一标尺」，我裁定以斗罗大陆 1 为唯一正典 | 合并后由 GLM5.3 审定，三方署名 | `docs/reports/玩家预期与剧情逻辑统一标尺.md` 第 3–5 行 |
| 3. 按标尺分工审查，互补不重复，证据写到「文件:行号」 | flash 管剧情文本，GLM5.3 管系统规则 | `docs/reports/统一标尺审查报告-flash.md` 第 4–7 行；`docs/reports/第一轮产品审查-判例验证-GLM.md` 第 5–7 行 |
| 4. 合并成定稿，拿不准的列为「待用户确认」，由我拍板 | 双方署名 | `docs/reports/第一轮审查定稿.md` 第 4–5、109–116 行 |
| 5. 写修复任务书：flash 起草，GLM5.3 审查，分批执行，每批独立 commit | flash → GLM5.3 | `docs/任务书-第一轮审查修复.md` 第 4、12–15 行 |
| 6. GLM5.3 改了任务书的执行顺序 | GLM5.3 | `docs/任务书-第一轮审查修复.md` 第 60、78 行 |
| 7. 按我拍板的 3 项执行第二批 | GLM5.3 | `docs/任务书-第二批执行-GLM.md` 第 3–9 行 |
| 8. 逐个 commit 复核 11 个修复，结论是无回滚项 | GLM5.3 复核 | `docs/reports/第一轮修复复核报告-GLM.md` 第 3–5、112–114 行 |

后面的需求接着用这套分工：任务书开头贴产品需求原文，flash 起草，GLM5.3 执行，flash 部署（`docs/任务书-无下滑改造-GLM.md` 第 3、6–8 行）。

## 文件清单

| 文件 | 用途 | 为什么有 |
|---|---|---|
| `docs/prompts/AI激活提示词.md` | 让 AI 先读完主指令再开工的短 prompt | （待补） |
| `docs/prompts/完整BUG修复提示词.md` | BUG 修复主指令：角色、范围、禁止事项、验收 | （待补） |
| `docs/prompts/prompt.md` | 浏览器实测后写的修复任务书 | （待补） |
| `docs/prompts/前端框架迁移任务.md` | 原生 JS 迁 Vue 的任务 prompt | （待补） |
| `docs/prompts/kimi_game_prompt.md` | 给 Kimi 的全前端重写 prompt | （待补） |
| `docs/prompts/kimi_prompt.md` | 给 Kimi 的图鉴页重写 prompt | （待补） |
| `.trae/documents/迁移prompt文档生成计划.md` | 核对并修正迁移 prompt 的计划 | （待补） |
| `.trae/documents/斗罗大陆全面提升计划.md` | 七大工作流的整体提升计划 | （待补） |
| `docs/任务书-核心逻辑修复.md` | 任务书 A，归主会话 | （待补） |
| `docs/任务书-KimiK3-前端界面美化.md` | 任务书 B，归 Kimi k3 | （待补） |
| `docs/reports/玩家预期与剧情逻辑标准-GLM.md` | GLM5.3 立场稿 | （待补） |
| `docs/reports/玩家预期与剧情逻辑标准-flash.md` | flash 立场稿 | （待补） |
| `docs/reports/玩家预期与剧情逻辑统一标尺.md` | 合并后的审查标准 | （待补） |
| `docs/reports/统一标尺审查报告-flash.md` | flash 的审查报告 | （待补） |
| `docs/reports/第一轮产品审查-判例验证-GLM.md` | GLM5.3 的审查报告 | （待补） |
| `docs/reports/第一轮审查定稿.md` | 双方署名的审查定稿 | （待补） |
| `docs/任务书-第一轮审查修复.md` | 分批修复任务书 | （待补） |
| `docs/任务书-第二批执行-GLM.md` | 交给 GLM5.3 的第二批执行任务 | （待补） |
| `docs/reports/第一轮修复复核报告-GLM.md` | 11 个修复 commit 的复核 | （待补） |
| `docs/任务书-无下滑改造-GLM.md` | 一条产品需求的起草、执行、部署分工 | （待补） |

## 来源与脱敏

| 文件 | 原始位置 | 脱敏 |
|---|---|---|
| `docs/prompts/` 下 6 个文件 | 提交 `7ec2163` 加入，`ff9909e` 删除 | `prompt.md`：本地绝对路径 1 处、玩家存档 ID 2 处 |
| `.trae/documents/` 下 2 个文件 | 提交 `7ec2163` 加入，`ff9909e` 删除 | `迁移prompt文档生成计划.md`：本地绝对路径 1 处 |
| `docs/任务书-KimiK3-前端界面美化.md` | 提交 `7ec2163` 加入，`ff9909e` 删除 | 本地绝对路径 1 处 |
| `docs/任务书-核心逻辑修复.md` | main 分支（首次提交 `7ec2163`） | 玩家存档 ID 1 处 |
| `docs/reports/玩家预期与剧情逻辑标准-GLM.md` | main 分支（首次提交 `98187db`） | 无 |
| `docs/reports/玩家预期与剧情逻辑标准-flash.md` | main 分支（首次提交 `316d62d`） | 无 |
| `docs/reports/玩家预期与剧情逻辑统一标尺.md` | main 分支（首次提交 `72a29c9`） | 无 |
| `docs/reports/统一标尺审查报告-flash.md` | main 分支（首次提交 `064b161`） | 无 |
| `docs/reports/第一轮产品审查-判例验证-GLM.md` | main 分支（首次提交 `61bcd25`） | 无 |
| `docs/reports/第一轮审查定稿.md` | main 分支（首次提交 `d627b02`） | 无 |
| `docs/任务书-第一轮审查修复.md` | main 分支（首次提交 `a1f019e`） | 无 |
| `docs/任务书-第二批执行-GLM.md` | main 分支（首次提交 `05bcce9`） | 无 |
| `docs/reports/第一轮修复复核报告-GLM.md` | main 分支（首次提交 `ffe0c54`） | 无 |
| `docs/任务书-无下滑改造-GLM.md` | main 分支（首次提交 `82aeec7`） | 无 |

脱敏的地方都写成了「（已脱敏）」。`127.0.0.1` 和 `localhost` 是本机地址，保留了。
