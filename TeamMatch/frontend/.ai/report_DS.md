# TeamMatch 前端审查报告 — DeepSeek 第一轮

**审查日期**：2026-05-10  
**Change / Commit**：86dfe15 — `feat(T-033): 组队请求列表联调准备 + 修复审查意见`  
**Change-Id**：Ie54c32d5b34868f8d963d4d8d4158976654724a2  
**审查范围**：基于 `review-frontend`（骨架初始化）→ 当前提交的完整 diff  

---

## A. 是否存在阻断问题

**否**

---

## B. 阻断问题列表

无。

---

## C. 审查意见修复验证

| 修复项 | 状态 | 验证 |
|---|---|---|
| 移除 `.codebuddy/` 目录和 `figma.html` | ✅ 已修复 | diff 中不再出现 |
| 回退 `project.config.json` appid 变更 | ✅ 已修复 | diff 中不再出现，保持骨架原值 `（已脱敏）` |
| 删除未使用的 `pages/member-profile/` | ✅ 已修复 | diff 中不再出现 |
| 队长移除改为发起投票入口（T-032） | ✅ 已修复 | 成员使用 `exit_mode: self_exit/exit_vote`，队长无直接移除按钮，操作图标仅占位 |
| 修复 `FUNCTION_HUB_PAGE_URL` 未定义 | ✅ 已修复 | `project-members.js` 第 12 行已定义常量 |
| 统一响应码判断为 `code===200` | ✅ 已修复 | `team-service.js` 返回 `code: 200`，`team-requests.js` 判断 `res.code !== 200` |

6/6 全部修复。

---

## D. 本次提交范围内未修复的 P1 缺陷（非阻断）

以下缺陷不影响仓库安全、不污染仓库、不影响其他负责人、不违反主业务口径，属于非阻断问题，记录于此供后续迭代修复：

### D1. `team-requests.js` — 切 Tab 无并发序号校验
- 位置：`loadRequests()` 第 92-132 行
- 问题：快速切换 Tab 时旧请求的响应可能覆盖新 Tab 的列表
- 建议：加 `requestId` 自增序号，回调时校验

### D2. `team-requests.js` — 接受/拒绝/取消只 toast 不更新本地状态
- 位置：`handleAction()` 第 141-156 行
- 问题：点击接受/拒绝/取消只弹 toast，不更新 `this.data.requests` 中对应项的状态，按钮点完卡片仍显示 pending
- 建议：用 `requestId` 找到对应项并修改 `status` 后 `setData`

### D3. `team-requests.wxml` — 操作按钮覆盖不全
- 位置：第 82-88 行
- 问题：只覆盖"收到 invite（接受/拒绝）"和"发出 apply（取消）"，漏掉：
  - 收到 apply → 应显示接受/拒绝
  - 发出 invite → 应显示取消
- 建议：补全四种情况的 action 分支

### D4. `team-service.js` — `await wx.request()` 不可靠
- 位置：`realGetRequests()` 第 122-132 行
- 问题：微信小程序原生 `wx.request` 不返回标准 Promise，`await` 可能拿到 `undefined`
- 建议：手动 `new Promise((resolve, reject) => { wx.request({... success: resolve, fail: reject }) })`

### D5. `project-members.wxml` — 缺少未登录守卫
- 位置：根 `<view class="page">` 第 1 行
- 问题：未登录用户进入时先渲染成员数据再跳转，闪现数据泄露
- 建议：加 `wx:if="{{isAuthenticated}}"`

### D6. `project-members.wxml` — 引用不存在的图片
- 位置：第 61-65 行
- 问题：`src="/images/icon-more.png"` 仓库中不存在，将显示空白/裂图
- 建议：替换为已有资源或移除占位

---

## E. 本次正面改进

| 改进项 | 评价 |
|---|---|
| `utils/team-service.js` Service 层封装 | `getRequests(type)` 统一入口，`USE_MOCK` 开关，`mockGetRequests`/`realGetRequests` 分离，500ms 延迟模拟 |
| loading / error / empty 三态 | `team-requests.wxml` 完整实现：加载动画（`@keyframes spin`）、错误文字+重试按钮、空状态插图+文案 |
| 卡片状态 5 变体 | pending/accepted/rejected/cancelled/expired，each with CSS class + label + resultHint |
| 退出机制正确展示 | `project-members.wxml` 区分 "主动退出"（self_exit）和 "被投票移除"（exit_vote），附日期 |
| RequestType 区分 | invite/apply 对应 "邀请"/"申请" 标签，hint 文案按类型分化 |
| V2.1 基线对齐 | 所有 JS 文件添加 V2.1 约束注释，不使用旧名称 |
| `.gitignore` 新增 | 排除 `.codebuddy/`、`.figma/`、`*.html.tmp`、`node_modules/`，预防未来污染 |

---

## F. 是否影响其他负责人代码

**否**

- `app.json` 只新增路由，不破坏已有页面
- `project.config.json` 未修改（appid 保持骨架原值）
- `utils/team-service.js` 为新增公共工具，不影响已有代码

---

## G. 是否污染仓库

**否**

- `.codebuddy/` 已移除
- `figma.html` 已移除
- `.gitignore` 主动预防未来污染

---

## H. 是否存在敏感信息

**否**

- 无硬编码 token（mock 使用 `mock-token-${Date.now()}` 仅在 Storage）
- 无硬编码密码/账号
- 无个人路径泄露
- `project.config.json` appid 保持骨架原始值

---

## I. 是否存在旧口径回流

**否**

- 成员退出使用 `self_exit` / `exit_vote`
- 请求状态使用 `pending/accepted/rejected/cancelled/expired`
- 请求类型使用 `invite/apply`
- 无 `removed`、`remove_vote`、队长直接移除等旧名称

---

## J. 是否建议继续 Code Review

**建议继续 Code Review**

---

## K. 建议 Gerrit 投票

**+1**

理由：无阻断问题。仓库安全干净，无敏感信息，无旧口径回流，不影响其他负责人。6 项审查意见全部修复。存在 6 个 P1 非阻断行为缺陷（见 D 节），不影响当前通过，后续迭代修复即可。

---

## L. 需要写入 FRONTEND_REVIEW_FEED.md 的内容

D1-D6 共 6 项 P1 非阻断缺陷（切 Tab 并发、action 不回写状态、action 覆盖不全、wx.request Promise 包装、未登录守卫缺失、icon-more.png 引用缺失）。

---

## M. 给组员的简短 Review 留言

> 本次提交无阻断问题，六项审查意见全部修复：`.codebuddy/` 已移除、appid 已回退、`member-profile` 已删除、退出机制已改为 exit_vote、`FUNCTION_HUB_PAGE_URL` 已定义、响应码已对齐 200。Service 层封装、三态切换、卡片状态变体实现清晰。另有几个非阻断优化点（切 Tab 并发、action 本地状态更新、`wx.request` Promise 包装、登录守卫、图片引用）已记录，后续迭代补充即可。建议继续 Code Review。
