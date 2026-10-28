# Douyin Lite Frontend Progress

## 项目定位

课程项目：视频流推荐系统（简易版抖音）
当前职责：成员 E，前端负责人
目标：（已脱敏）

本文件是后续 AI 对话的进度记录。每次开始新对话时，先阅读：

1. `SKILL.md`
2. `FRONTEND_PROGRESS.md`
3. `CURRENT_TASK.md`
4. `TEACHING_RULES.md`

默认不要读取 `FUTURE_OPTIMIZATIONS.md`，只在阶段收尾、重构或用户明确要求优化时读取。

---

## 五轨道并行模型

每个阶段不是"写完代码就完"，而是五条轨道同时推进：

| 轨道 | 内容 | 谁主导 |
|------|------|--------|
| **Coding** | AI 帮你写什么文件、完成什么功能 | AI 辅助 |
| **讲解** | 为什么这么做、文件职责、数据怎么流 | AI 讲解，用户理解 |
| **验证** | 浏览器 Network / 控制台 / 页面操作确认 | 用户手动执行 |
| （已脱敏） | （已脱敏） | （已脱敏） |
| **边界** | 哪些是前端做的、哪些是后端做的、哪些不能夸大 | AI 标注 |

---

## 学习主线：每个阶段必须回答的 8 个问题

1. 这个阶段解决什么用户问题？
2. 涉及哪些前端文件？
3. 页面层、api 层、utils 层分别负责什么？
4. 用户点击后数据怎么流？
5. 请求从哪里发出，响应回到哪里？
6. 我应该用 Network 看什么？
7. （已脱敏）
8. 哪些不能夸大？

---

## 每阶段固定输出模板

```
阶段名称：
本阶段目标：
先讲概念：
再看文件：
再写最小代码：
再手动验证：
（已脱敏）
我必须能独立复述的链路：
```

---

## 阶段总览

| 阶段 | 内容 | 状态 |
|------|------|------|
| Phase 1 | 前端基础概念 + 项目骨架 | 已完成 |
| Phase 2 | 前后端通信基础（token/request/api） | 已完成 |
| Phase 3 | 登录注册（完整链路） | 已完成 |
| Phase 4 | 推荐流核心（previous/forward/next + 点赞） | 已完成 |
| Phase 5 | 视频管理（发布 + 我的视频 + 删除） | 未开始 |
| Phase 6 | （已脱敏） | 未开始 |

---

## Phase 1：前端基础概念 + 项目骨架

### 状态：Phase 1 完成，进入 Phase 2

### 1A 设计讲解（已完成）

- [x] SPA 是什么
- [x] 为什么选 Vue3 + Vite
- [x] 前端项目目录结构 — 为什么这样分
- [x] Vue 单文件组件（.vue）是什么
- [x] 数据流初步

### 1B 编码（已完成）

- [x] `npm create vite@latest` 创建项目
- [x] 建立目录结构（views/api/utils/router/components）
- [x] 创建 `router/index.js`，配好 4 条路由
- [x] 创建 4 个空 `.vue` 页面
- [x] `main.js` 挂载 router，`App.vue` 放 `<router-view>`

### 1C 验证（已完成）

- [x] 4 个路由文件创建完毕，结构正确

### （已脱敏）

- [x] 能讲清楚：SPA 是什么、目录为什么这样分、.vue 文件三段分别干什么
- [x] 理解链路：浏览器输入 /login → Vue Router 匹配路由 → 加载 LoginView.vue → 显示在 `<router-view>` 区域

### 我必须能独立复述的链路

```
浏览器输入 /login → Vue Router 匹配路由 → 加载 LoginView.vue → 显示在 <router-view> 区域
全程没有向服务器发 HTML 请求
```

---

## Phase 2：前后端通信基础

### 状态：Phase 2 完成，进入 Phase 3

### 2A 设计讲解（已完成）

- [x] fetch vs axios，为什么选 Axios
- [x] Axios 实例是什么，为什么封装 request.js
- [x] 请求拦截器 — 解决什么问题
- [x] 响应拦截器 — 解决什么问题
- [x] Token 存哪里
- [x] API 模块为什么按业务域拆分

### 2B 编码（已完成）

- [x] `utils/token.js`
- [x] `utils/request.js`
- [x] `api/auth.js`
- [x] `api/feed.js`
- [x] `api/video.js`

### 2C 验证（已完成）

- [x] 所有导出函数存在，参数正确

### （已脱敏）

- [x] 能讲清楚：拦截器是什么、token 怎么注入、为什么分 api 文件

---

## Phase 3：登录注册（完整链路）

### 当前重点

当前重点文件是 `LoginView.vue`。必须先完成并讲清楚登录注册页面，再进入 `router/index.js` 路由守卫。

### 3A 设计讲解

- [x] JWT 登录全链路（7 步，每步谁做什么）
- [x] 前后端安全分工
- [x] 路由守卫 — 为什么只是体验优化
- [x] 401 响应拦截
- [x] 状态码含义：400/401/403/404/409/500

### 3B 编码

- [x] `LoginView.vue`
- [x] `router/index.js` beforeEach 路由守卫
- [x] `utils/request.js` 补充 401 跳转

### 3C 验证

- [x] 未登录访问 /feed → 自动跳 /login
- [x] 登录成功 → 跳 /feed（需后端配合）
- [x] 手动清 Token → 刷新后跳 /login

### （已脱敏）

- [x] 能讲清楚：登录全链路、安全边界、401 处理

---

## Phase 4：推荐流核心

### 4A 设计讲解

- [x] 为什么只要 currentVideo 一个状态
- [x] previous / forward / next 三个接口的区别
- [x] forward → next 降级逻辑
- [x] 点赞本地更新，为什么不能直接 +1
- [x] API 函数 vs 页面代码的职责边界

### 4B 编码

- [x] `FeedView.vue`（mock 数据版本）
- [x] previous / forward / next 状态流转
- [x] 点赞 + likeCount 更新 + likedMap 去重 + 红心样式

### 4C 验证

- [x] 上一条/下一条切换视频（mock 循环）
- [x] 点赞后 likeCount 变化，按钮变红，重复点赞弹窗
- [x] 退出清 Token 跳登录

### （已脱敏）

- [x] 能讲清楚：three-feed-api 设计、currentVideo 状态机、点赞防重复

---

## Phase 5：视频管理

### 5A 设计讲解

- [ ] 分页 page/pageSize/total 三个参数
- [ ] 发布视频数据流
- [ ] 删除后为什么刷新列表
- [ ] 新发布视频会不会出现在自己推荐流

### 5B 编码

- [ ] `PublishView.vue`
- [ ] `MyVideosView.vue`（分页 + 删除）

### 5C 验证

- [ ] 发布后在"我的视频"可见
- [ ] 删除后列表刷新
- [ ] 分页参数在 Network 可见

### （已脱敏）

- [ ] 能讲清楚：分页设计、发布链路、删除刷新策略

---

## Phase 6：（已脱敏）

### 6A 设计讲解

- [ ] mock vs 真实接口 — 各验证什么
- [ ] CORS 是什么
- [ ] 浏览器 Network 面板各看什么

### 6B 操作

- [ ] baseURL 切真实后端
- [ ] 逐页面联调
- [ ] （已脱敏）
- [ ] 整理 README/演示链路

### 6C 验证

- [ ] Network 11 项检查全部通过

---

## 浏览器联调检查清单

每次切真实接口时检查：

- [ ] URL 是否正确
- [ ] Method 是否正确
- [ ] Query / Body 是否正确
- [ ] Request Header 是否带 `Authorization`
- [ ] Response Body 是否是 `code/message/data`
- [ ] 401 是否清 Token 并跳登录
- [ ] 403 是否理解为权限不足
- [ ] 404 是否理解为资源不存在或已删除
- [ ] 500 是否先看后端日志
- [ ] CORS 是否单独判断，不误判成接口没写
- [ ] mock 切真实接口时，不改页面业务逻辑，只改 request/baseURL/api 层

---

## Token / 401 学习链路

必须看懂这条链：

```text
LoginView 登录成功
-> login(data)
-> request.post('/api/auth/login')
-> 后端返回 token
-> token.js 保存 token
-> request.js 请求拦截器注入 Authorization
-> 访问受保护接口
-> 如果响应 401，响应拦截器清 token 并跳 /login
```

（已脱敏）

前端做 Token 保存、请求注入和体验层跳转；真正的鉴权由后端 `JwtAuthFilter` 完成。

---

## 推荐页核心逻辑记录

`currentVideo` 表示当前正在展示的视频。

点击上一条：

```text
getPreviousVideo(currentVideo.id)
-> 如果 data != null，更新 currentVideo
-> 如果 data == null，提示已是第一个视频
```

点击下一条：

```text
getForwardVideo(currentVideo.id)
-> 如果 data != null，更新 currentVideo
-> 如果 data == null，再调用 getNextVideo()
-> getNextVideo 返回新推荐后更新 currentVideo
```

点击点赞：

```text
likeVideo(currentVideo.id)
-> 成功后更新当前视频 likeCount
-> 重复点赞后端返回 200，但不应重复增加计数
```

不要混淆：

- `next`：拉新推荐，后端会写 `visit_records`
- `previous`：访问历史中更早的一条
- `forward`：访问历史中更新的一条

---

## 页面路由表

| 路由 | 页面 | 功能 |
|------|------|------|
| `/login` | `LoginView.vue` | 登录 / 注册 |
| `/feed` | `FeedView.vue` | 推荐视频主页、previous/forward/next、点赞、退出 |
| `/my-videos` | `MyVideosView.vue` | 我的视频分页、删除、跳转发布 |
| `/publish` | `PublishView.vue` | 填写标题和 videoUrl 发布视频 |

## API 调用表

| 模块 | 函数 | 后端接口 |
|------|------|---------|
| auth | `register(data)` | `POST /api/auth/register` |
| auth | `login(data)` | `POST /api/auth/login` |
| auth | `logout()` | `POST /api/auth/logout` |
| feed | `getNextVideo()` | `GET /api/feed/videos/next` |
| feed | `getPreviousVideo(currentVideoId)` | `GET /api/feed/videos/previous?currentVideoId={id}` |
| feed | `getForwardVideo(currentVideoId)` | `GET /api/feed/videos/forward?currentVideoId={id}` |
| video | `publishVideo(data)` | `POST /api/videos` |
| video | `getMyVideos(params)` | `GET /api/users/me/videos?page=1&pageSize=10` |
| video | `deleteVideo(videoId)` | `DELETE /api/videos/{videoId}` |
| video | `likeVideo(videoId)` | `POST /api/videos/{videoId}/like` |

## 前端状态表

| 状态 | 所属页面/模块 | 用途 |
|------|-------------|------|
| `token` | `utils/token.js` | 登录态凭据 |
| `currentUser` | 登录后页面 | 当前用户基本信息 |
| `currentVideo` | `FeedView.vue` | 当前展示的视频 |
| `feedLoading` | `FeedView.vue` | 推荐页请求中状态 |
| `myVideos` | `MyVideosView.vue` | 我的视频列表 |
| `page` / `pageSize` | `MyVideosView.vue` | 分页参数 |
| `total` | `MyVideosView.vue` | 视频总数 |
| `publishForm` | `PublishView.vue` | `title` 和 `videoUrl` |
| `errorMessage` | 页面级 | 普通错误提示 |

---

## 不做的事情（重申）

- 不美化 UI，功能跑通优先
- 不实现评论、关注、私信等
- 不实现视频上传/转码
- 不碰后端代码
- 不 Element Plus 强依赖
- 不一次性生成全部代码

---

## 当前下一步

Phase 5：视频管理（发布 + 我的视频 + 删除）
