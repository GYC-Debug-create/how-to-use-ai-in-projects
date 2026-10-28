# 任务书：全端"无下滑"改造（GLM5.3 执行）

> 起草：flash（2026-08-15）· 执行：GLM5.3 · 部署：flash（改造完成后整体上线）
> 前置状态：第一轮修复 19 项 + N1 legacy 同步已全部完成（HEAD `d5de2a4`），测试 236/236 全绿。

## 一、目标（产品需求原文）

> "不管是哪一端，都不应该出现下滑这个操作，影响游戏体验。"

即：**任何设备（桌面 / 平板 / 手机，竖屏与横屏）上打开游戏的任何页面，都不允许出现"下滑"（页面级纵向滚动）操作**。内容放不下一屏时，只能由内容区域**内部滚动**（不带动整个页面滚动、不出现橡皮筋/下拉刷新），且关键操作元素必须首屏可达。

## 二、现状调研（flash 已查证，含行号）

### Vue 版（`frontend/`，生产主入口 `/`）

已做对的部分（**不要回退**）：
- `App.vue` #app、`GamePage.vue`、`CreatePage.vue`、`TitlePage.vue`、`MapPage.vue` 根容器均为 `height:100dvh; overflow:hidden`（含 100vh 回退）。
- `GamePage.vue:711-715` 移动端状态面板展开时 `max-height:42dvh; overflow-y:auto; overscroll-behavior:contain`（内部滚动已带 contain）。
- `TitlePage.vue:451-455` 存档弹窗 `overflow-y:auto`。

疑点（需要逐项核实并修复）：
1. **移动端橡皮筋/下拉刷新**：`global.css` 的 `html/body` 未见 `overscroll-behavior:none` 与 `touch-action:manipulation`——iOS Safari 顶部下拉橡皮筋、安卓 Chrome 下拉刷新在 `overflow:hidden` 壳上仍可能发生，需全端封死。
2. **CreatePage.vue:1094 `overflow: visible`**：某区域允许溢出，需确认不会撑出页面滚动。
3. 各弹窗/列表内部滚动容器（商店、背包、技能树、图鉴、任务、锻造、日记、称号等）需统一 `overscroll-behavior:contain`，缺的补上（可全局规则收敛，如 `#app * { overscroll-behavior: contain }` 或逐类补）。
4. 小屏手机（如 390×844）上主游戏页的事件选项区/战斗按钮是否首屏可达、不被状态栏/通知遮挡，需实测确认（重点：事件选项多时是否只能靠下滑）。

### legacy 版（`public/`，兼容入口 `/game.html` `/map.html` `/codex.html`）

问题主源，**必须改造**：
1. `game.html:15-18` body `min-height:100vh; overflow-x:hidden`；`:34` #app `min-height:100vh`；`:406` .main-area `min-height:100vh`——**允许整页纵向滚动**，需改为视口固定（`height:100dvh; overflow:hidden`）+ 场景区内部滚动（`.scene-area` 已有 `overflow-y:auto`，保留并加 `overscroll-behavior:contain`）。
2. `game.html:286/304` 觉醒选项滚动条、`:505` 战斗日志、`:817` 试炼日志等内部滚动容器：补 `overscroll-behavior:contain`。
3. `game.html` 多处面板 `max-height:60vh; overflow-y:auto`（技能树 1009、称号 1032、羁绊 1116、快速旅行 1160、商店 1165、背包 1188、日记 1417、统计 1509、塔 1588、锻造 1636、任务 1653、训练 1681 等）：统一补 `overscroll-behavior:contain`。
4. `game.html:325` `.status-panel{position:sticky; height:100vh; overflow-y:auto}`：sticky 侧栏本身滚动 OK，但需确认与主区不叠加成双滚动。
5. `map.html`、`codex.html`：同样检查 body/#app 是否允许页面级滚动，按同一标准改造（map.html 已有 768/480 断点与横屏适配，codex.html 已有 760/600/480 断点，改造时应保持）。

## 三、改造要求（统一标准）

1. **页面级滚动 = 0**：所有页面（Vue 全部路由页 + legacy 三页）在任何视口尺寸下，body/根容器不得产生纵向滚动；采用"根容器 100dvh/100vh 固定 + overflow:hidden + 内容区内部滚动"结构。
2. **内部滚动容器**：凡 `overflow-y:auto/scroll` 处一律 `overscroll-behavior:contain`；`-webkit-overflow-scrolling` 不再需要（现代浏览器），不要引入。
3. **移动端封死橡皮筋**：`html, body { overscroll-behavior: none; touch-action: manipulation; }`（legacy 页同样）；iOS 动态视口用 `100dvh`（已有回退 100vh 的保持）。
4. **首屏可达**：小屏手机上，各页主操作（事件选项、战斗指令、创建流程按钮、图鉴切换等）必须在首屏内，禁止"滑到底才能点按钮"。
5. **不做**：不改玩法逻辑、事件/数据内容、server.js、部署配置；不引入第三方库；不改变桌面端现有观感（桌面端当前无滚动问题的页面保持原样）。

## 四、验收清单（交付时逐项打勾）

| # | 页面 | 桌面 1920 | 笔记本 1366 | 平板 1024 | 平板 768 | 手机 390×844 竖 | 手机 844×390 横 |
|---|---|---|---|---|---|---|---|
| 1 | 标题页（含存档弹窗） | | | | | | |
| 2 | 创建页（出身/武魂/觉醒） | | | | | | |
| 3 | 主游戏页（事件/状态面板/移动端展开态） | | | | | | |
| 4 | 战斗（含日志） | | | | | | |
| 5 | 地图页（Vue） | | | | | | |
| 6 | 图鉴页（Vue CodexPage） | | | | | | |
| 7 | 弹窗类（商店/背包/技能树/任务/锻造/日记/称号/快速旅行） | | | | | | |
| 8 | legacy game.html | | | | | | |
| 9 | legacy map.html | | | | | | |
| 10 | legacy codex.html | | | | | | |

验收动作：每格做两件事——① 页面**不可下滑**（触屏下拉/滚轮下滑均不产生页面级滚动）；② 内容溢出时**容器内部滚动**正常、滚动到边界**无橡皮筋**；③ 主操作元素首屏可见。

## 五、交付物

1. 逐项改造的 diff（commit 到本仓库，message 前缀 `no-scroll:`）。
2. 验收清单填写结果（直接填在第四节的表里，注明每个格子"通过 / 已修"）。
3. 若某项无法在无真机条件下验证，明确标注"待真机验证"并说明风险。

## 六、验证命令（跑完贴结果）

```bash
npm run validate:data
npm test        # 236/236 预期
```
