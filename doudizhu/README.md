# doudizhu

[项目仓库：家庭斗地主](https://github.com/GYC-Debug-create/doudizhu)

这是我独立完成的本地 H5 斗地主 / 掼蛋小游戏，采用 Vibe Coding 方式开发。项目仓库 README 描述了可运行的游戏模式、规则、AI 对手、交互、音效和测试。项目不联网，也没有公开在线试玩地址。

## AI Coding 过程记录

当前项目仓库没有保存 Prompt、AI 对话、任务卡或审查报告原文。因此本页只整理可从公开 README、源码目录和 GitHub 提交历史核实的事实；没有把提交标题扩写成具体对话，也不推断未记录的模型分工。

### 可核实的开发轨迹

GitHub 提交历史显示，项目按相对独立的功能阶段推进。多个功能提交由 `GYC-Debug-create` 与 `claude` 共同署名；这是 AI 参与编码的仓库记录。提交列表不能单独证明每个实现细节由谁提出或撰写。

| 阶段 | 仓库记录 | 可核对位置 |
|---|---|---|
| 工程起步 | 搭建 uni-app + Vue 3 工程骨架 | [提交 f92463d](https://github.com/GYC-Debug-create/doudizhu/commit/f92463d55bdb1fe4c0854c1a5a24961c56ffbfd5) |
| 斗地主核心规则 | 54 张牌表示、牌型识别与比较、叫抢地主/出牌/结算状态机 | [牌组提交](https://github.com/GYC-Debug-create/doudizhu/commit/9e7fee3f26127a60548d03ffb8d54ee118056339)、[牌型提交](https://github.com/GYC-Debug-create/doudizhu/commit/84a248432c88fd5ba05fa61b0a5c0500661778d6)、[规则提交](https://github.com/GYC-Debug-create/doudizhu/commit/048e10718cd7fef4b79010abac0b267c6846acc6) |
| AI 对手与应用调度 | AI 着法枚举、评估和决策拆分；控制器事件化与回合调度 | [AI 提交](https://github.com/GYC-Debug-create/doudizhu/commit/a6e6efb05ca5087f6d5b0119cb1a913bc149e004)、[调度提交](https://github.com/GYC-Debug-create/doudizhu/commit/a5d2fa349ce77d33dddb4090feecd5acfb823a07) |
| 掼蛋扩展 | 增加 4 人 108 张 2v2 掼蛋引擎与 AI，并补规则测试 | [提交 2f43861](https://github.com/GYC-Debug-create/doudizhu/commit/2f43861750fbdc82afbb3431d64283a99d6d090d) |
| 界面与交互 | 牌桌组件、页面装配、档案、快捷键、缩放与交互 | [应用层提交](https://github.com/GYC-Debug-create/doudizhu/commit/d17a66ca2c31197d1a05954ee9f4571c7e3ec9ab)、[界面组件提交](https://github.com/GYC-Debug-create/doudizhu/commit/bbaf3265d05d9bd32aed41f5d2f0180a84beff91)、[页面提交](https://github.com/GYC-Debug-create/doudizhu/commit/921b9899615e07b18081d5e0d298d4490be1cc02) |
| 音效与语音 | WebAudio 音效、预录神经网络语音、按场景和情绪选择台词并与回合节奏同步 | [声音提交](https://github.com/GYC-Debug-create/doudizhu/commit/149de60b51576f63b3b64903f23908f0569709c6)、[语音提交](https://github.com/GYC-Debug-create/doudizhu/commit/220c5608cbf46c0378bef929861dde73f23b6585)、[播放节奏提交](https://github.com/GYC-Debug-create/doudizhu/commit/764cc842d1210a87d665f784d51094fe946ef502) |

### 项目事实与 AI 概念边界

- 项目是我独立完成的 Vibe Coding 作品；GitHub 提交记录可见 Claude 参与提交。没有留存的 Prompt 原文和具体对话不能由本页代替。
- 游戏中的“AI”是本地规则驱动的牌局对手，源码按着法生成、评估、策略决策组织；不要将它描述成接入大语言模型的对话式 AI 产品。
- 语音素材由微软 `edge-tts` 预生成，游戏运行时离线播放。这属于语音合成素材生成，不代表游戏运行时调用生成式 AI。
- 测试目录包含牌型、规则、AI 策略、应用层、拖拽选择等测试；可通过项目 README 的 `npm test` 执行。

## 后续可补的原始材料

如要让 AI 使用过程更完整，应从本人仍保留的历史会话或笔记补入真实的需求 Prompt、关键方案取舍、失败/修正案例和测试验收记录，并标注日期与对应提交。没有原始依据的部分保持未记录，不补造。
