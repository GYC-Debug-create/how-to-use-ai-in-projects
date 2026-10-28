# 未来优化记录

## 待优化项

### 1. Token 存储方案升级
- **当前**：localStorage
- **目标**：httpOnly cookie
- **原因**：localStorage 存在 XSS 风险，同域名下任意 JS 可读取
- **影响范围**：仅 `utils/token.js` 内部实现，调用方不受影响
- **依赖**：需要后端配合设置 Set-Cookie Header

### 2. 响应拦截器统一提取 response.data
- **当前**：`(response) => response` 原样透传
- **目标**：`(response) => response.data`，api 层直接拿到业务数据
- **原因**：联调确认后端返回格式 `{ code, message, data }` 后，统一提取，api 层不再重复 `.data`
- **影响范围**：仅 `utils/request.js` 响应拦截器成功回调，api 层调用方需相应调整
- **时机**：Phase 6 联调时确认后端返回体格式后改动
