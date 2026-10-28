# T-33 前端整改清单

适用提交：`T-33 新增 Service 层封装、加载/错误/空状态、完整的卡片状态变体，同时保持已有视觉风格不变`

Change-Id：`I84cd07c8a4aa99e5df8637b4a0ef77dad1a19d6f`

目标：在不大改现有视觉风格的前提下，修复组队请求页和成员页的行为缺口，并清理或说明仓库级遗留问题。

## 1. 审查结论口径

如果只审 T-33 单提交：

- 建议 Gerrit 投票：`+2`
- 理由：方向正确，Service 层、loading/error/empty、状态变体都已补上；剩余主要是 Mock 阶段行为不完整和局部边界问题，不影响其他人开发，按当前开发过渡阶段标准不应降低通过票。

如果审 `review-frontend -> 当前提交` 全量 diff：

- 需要先处理仓库级问题后再通过。
- `.codebuddy/figma`、个人路径、`project.config.json` appid、未注册 `member-profile` 属于基线/全量范围问题，不应全部算到 T-33 单提交头上，但提交包最终仍需要清理或说明。

## 2. T-33 需要整改的问题

### 2.1 快速切 tab 时旧请求可能覆盖新列表

位置：

- `pages/team-requests/team-requests.js`
- `switchTab()`
- `loadRequests()`

问题：

当前 `loadRequests()` 没有请求序号或 activeTab 校验。用户快速切换“我收到的 / 我发出的”时，旧请求后返回，可能覆盖当前 tab 的列表。

建议修复：

- 增加 `requestSeq` 或 `loadingRequestId`。
- 每次 `loadRequests()` 自增序号。
- Promise 返回后先判断序号是否仍是最新，或判断 `this.data.activeTab === activeTab`。
- 不是最新请求则直接丢弃返回结果。

验收标准：

- 快速连续点击两个 tab，最终列表必须和当前高亮 tab 一致。

### 2.2 接受 / 拒绝 / 取消只 toast，不更新状态

位置：

- `pages/team-requests/team-requests.js`
- `handleAction()`

问题：

当前点击“接受 / 拒绝 / 取消”只显示 toast，不更新本地 `requests`，也没有调用 service。用户点完后卡片仍是 `pending`，按钮仍然存在。

建议修复：

- Mock 阶段可先在前端本地更新当前卡片状态。
- `accept` -> `accepted`
- `reject` -> `rejected`
- `cancel` -> `cancelled`
- 更新后重新生成 label/class/hint，或复用统一的 normalize 方法。

验收标准：

- 点击接受后，卡片状态变为 `ACCEPTED`，按钮消失，显示结果提示。
- 点击拒绝后，卡片状态变为 `REJECTED`。
- 点击取消后，卡片状态变为 `CANCELLED`。

### 2.3 操作按钮覆盖不完整

位置：

- `pages/team-requests/team-requests.wxml`

问题：

当前只覆盖：

- 收到 `invite`：接受 / 拒绝
- 发出 `apply`：取消

漏掉：

- 收到 `apply`：队长应能接受 / 拒绝
- 发出 `invite`：邀请方应能取消

建议修复：

- pending 且 `isReceived === true` 时显示接受 / 拒绝。
- pending 且 `isReceived === false` 时显示取消。
- 不要把操作条件写死到单个 requestType 上。

验收标准：

- pending received apply 有接受 / 拒绝按钮。
- pending received invite 有接受 / 拒绝按钮。
- pending sent apply 有取消按钮。
- pending sent invite 有取消按钮。

### 2.4 `await wx.request(...)` 真实接口预留不可靠

位置：

- `utils/team-service.js`
- `realGetRequests()`

问题：

微信小程序 `wx.request` 通常不是原生 Promise。当前写法 `const res = await wx.request(...)` 后续切到真实接口时可能拿不到 `res.data`。

建议修复：

- 手动包装 Promise。
- 保持返回格式统一：`{ code, message, data: { list } }`。

参考写法：

```js
function request(options) {
  return new Promise((resolve, reject) => {
    wx.request({
      ...options,
      success: resolve,
      fail: reject
    })
  })
}
```

验收标准：

- `USE_MOCK = false` 时 `realGetRequests()` 能正常返回后端响应。
- 网络失败时返回错误对象或抛出后被页面 catch。

### 2.5 成员页未登录时会先渲染 mock 数据

位置：

- `pages/project-members/project-members.wxml`
- `pages/project-members/project-members.js`

问题：

JS 里有 `isAuthenticated`，但 WXML 根节点没有 `wx:if="{{isAuthenticated}}"`。未登录时页面可能先显示成员 mock 数据，再跳转登录页。

建议修复：

- 根节点加 `wx:if="{{isAuthenticated}}"`。
- 和 `team-requests.wxml` 保持一致。

验收标准：

- 未登录进入成员页时，不应闪现成员列表。

### 2.6 操作图标资源不存在

位置：

- `pages/project-members/project-members.wxml`
- `src="/images/icon-more.png"`

问题：

仓库没有 `images/icon-more.png`，队长可见的操作图标会加载失败。

建议修复：

- 改用仓库已有资源，例如 `assets/CodeBuddyAssets/...` 中合适的图标；或
- 新增正确图标资源；或
- 暂时用文本 / CSS 圆点实现，不引用不存在文件。

验收标准：

- 页面不再引用不存在的 `/images/icon-more.png`。
- 队长视角下操作图标正常显示。

## 3. 仓库级问题

以下问题按审查范围区分处理。

### 3.1 T-33 单提交直接相关

位置：

- `.codebuddy/rules/m2.mdc`

问题：

T-33 相比父提交新增 / 修改了 `.codebuddy/rules/m2.mdc`。这是本地 AI / IDE 规则文件，通常不应进入 Gerrit 业务提交。

建议修复：

- 从提交中移除 `.codebuddy/rules/m2.mdc`。
- 如果团队确实要保留，需要在 Review 中说明它是团队共享规则，不是个人工具产物。

### 3.2 全量 diff / 提交包范围相关

这些问题不是 T-33 单提交独有，但如果最终审整个提交包，仍建议处理：

- `.codebuddy/figma/1_18/figma.html` 含 `（已脱敏）` 个人路径。
- `project.config.json` appid 从 `（已脱敏）` 改为 `（已脱敏）`，需要说明是否团队共享测试号。
- `pages/member-profile/` 存在但未在 `app.json` 注册，需删除或注册。

## 4. 修复时不要做的事

- 不要大改现有视觉风格。
- 不要重构成新架构。
- 不要引入 npm / Vue / React / Taro。
- 不要把 Mock 阶段问题扩大成完整后端联调。
- 不要改无关页面。
- 不要把旧口径 `removed`、`remove_vote`、队长直接移除成员带回来。

## 5. 建议提交说明

可以使用类似说明：

```text
T-33 修复组队请求页交互与仓库清理问题

- 防止快速切换 tab 时旧请求覆盖当前列表
- Mock 阶段补齐接受、拒绝、取消后的本地状态更新
- 补齐 pending 请求的按钮展示规则
- 包装 wx.request，保留真实接口预留
- 成员页增加登录态渲染保护
- 修复不存在的操作图标路径
- 移除或说明 .codebuddy 规则文件
```
