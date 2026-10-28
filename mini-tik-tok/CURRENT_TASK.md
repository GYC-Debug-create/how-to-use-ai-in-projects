# CURRENT_TASK.md

## 当前阶段

Phase 5：视频管理（发布 + 我的视频 + 删除）。

## 当前目标

完成 PublishView.vue 和 MyVideosView.vue 两个页面，实现视频发布、分页列表、删除功能。

## Phase 4 已完成内容（供参考）

- `FeedView.vue`：currentVideo 状态机 + previous/forward/next 流转 + 点赞（likedMap 去重 + 红心）
- mock 数据模式，后端就绪后替换 API 调用即可
- 路由守卫临时放行（Phase 4 开发期间）

## 已存在的 API 函数（api/video.js）

```
publishVideo(data)      → POST /api/videos          (body: { title, videoUrl })
getMyVideos(params)     → GET /api/users/me/videos   (query: page, pageSize)
deleteVideo(videoId)    → DELETE /api/videos/{videoId}
```

## Phase 5 涉及文件

| 文件 | 当前状态 | Phase 5 目标 |
|------|---------|-------------|
| `views/PublishView.vue` | Phase 1 空壳 | 表单：标题 + videoUrl + 发布按钮 |
| `views/MyVideosView.vue` | Phase 1 空壳 | 分页列表 + 删除按钮 + 跳转发布页 |
| `router/index.js` | Phase 3 路由守卫 + Phase 4 临时放行 | 确认 /publish 和 /my-videos 路由正常 |

## 推荐推进顺序

### PublishView.vue

1. 静态骨架：标题输入框 + videoUrl 输入框 + 发布按钮
2. 定义 publishForm（reactive）+ v-model 双向绑定
3. 非空校验 + errorMessage
4. 调用 publishVideo(data) + 成功后跳转 /my-videos
5. mock 模式：跳过 API 调用，直接模拟发布成功

### MyVideosView.vue

1. 静态骨架：视频列表区域 + 分页按钮 + "去发布"按钮
2. 定义 myVideos/page/pageSize/total 状态
3. 调用 getMyVideos({ page, pageSize }) 加载列表
4. 分页：上一页/下一页
5. 删除按钮：调用 deleteVideo(videoId)，成功后刷新列表
6. mock 模式

## 核心状态

```
PublishView:
  publishForm = { title: '', videoUrl: '' }   // 响应式表单数据
  publishLoading = false                       // 发布中
  errorMessage = ''                            // 校验/接口错误提示

MyVideosView:
  myVideos = []        // 当前页视频列表
  page = 1             // 当前页码
  pageSize = 10        // 每页条数
  total = 0            // 总条数
  listLoading = false  // 列表加载中
```

## 概念边界

- `page` / `pageSize` / `total` 三个参数一起决定分页
- 总页数 = Math.ceil(total / pageSize)
- 删除后应重新请求当前页（因为列表变了）
- 发布成功后跳转 /my-videos，新视频应出现在列表中
- 新发布的视频会不会出现在自己的推荐流？由后端推荐算法决定，前端不处理

## 当前验收标准

- /publish 页面：填写标题和 videoUrl，点发布
- /my-videos 页面：显示自己的视频列表，支持翻页
- 删除按钮可用，删除后列表刷新
- 发布成功后跳转 /my-videos

## 当前禁止事项

- 不美化 UI，功能跑通优先
- 不实现视频上传/转码（videoUrl 手动填入）
- 不实现编辑功能
- 不要把 UI、API、Router 混在一次修改里
- 先做 PublishView，再做 MyVideosView
- mock 模式优先，后端就绪后切换
