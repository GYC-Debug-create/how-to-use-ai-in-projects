# M5 测试计划

本文件只覆盖 M5 测试。后续实现每个小任务时，优先补对应测试或说明无法运行原因。

## 单元测试

### 互评资格

覆盖：

- 当前用户不是项目成员 → `NOT_PROJECT_MEMBER`
- 目标用户不是项目成员 → `TARGET_NOT_PROJECT_MEMBER`
- 项目未结束 → `PROJECT_NOT_ENDED`
- 互评窗口关闭 → `EVAL_WINDOW_CLOSED`
- 已评价过 → `ALREADY_EVALUATED`
- 评价自己 → `SELF_EVALUATION`
- 全部满足 → eligible

### 信誉分映射

覆盖所有区间边界：

- [1.0, 1.5)
- [1.5, 2.0)
- [2.0, 2.5)
- [2.5, 3.0)
- [3.0, 3.5)
- [3.5, 4.0)
- [4.0, 4.5)
- [4.5, 5.0]

### 项目级封顶

覆盖：

- 正向未超过 +10
- 正向截断到 +10
- 负向未超过 -10
- 负向截断到 -10
- 已有正向流水后继续加分
- 已有负向流水后继续扣分
- `effective=0` 流水不参与当前有效分计算

### 提交互评

覆盖：

- 四维评分保存
- 1 / 2 分缺负向标签失败
- 1 / 2 分说明不足 20 字失败
- 5 分缺正向标签失败
- `skill_endorse` 保存但不影响信誉分
- 成功提交写入 `evaluation` 与 `credit_change`
- 重复提交被拒绝

### 异常评价

覆盖：

- 未覆盖全部队友时不检测
- 覆盖全部队友且全低分 → 全部 `pending_review`
- 覆盖全部队友且全满分 → 全部 `pending_review`
- 正常评价不挂起
- 挂起后 `credit_change.effective=0`
- 挂起后回退 `user.credit_score`

### `credit_change.effective`

覆盖：

- 正常评价 `effective=1`
- 异常挂起 `effective=0`
- `approve` 恢复为 `effective=1`
- `void` 保持 `effective=0`
- `keep_no_credit` 保持 `effective=0`
- `effective=false` 仍可查询展示

### 评价复核

覆盖：

- `approve`：`pending_review → normal`
- `void`：`pending_review → voided`
- `keep_no_credit`：`pending_review → kept_no_credit`
- 非 `pending_review` 不允许复核
- 重复复核失败或幂等处理符合实现约定
- 状态、流水、信誉缓存同事务一致

### `appeal_restore` 幂等

覆盖：

- 原流水 `effective=1` 时追加恢复流水
- 恢复值为原流水相反数
- 已存在同一 `(source_type, source_id)` 恢复流水时跳过
- 原流水 `effective=0` 时跳过
- 回写 `user.credit_score`
- 事务失败时不产生半截恢复

## 接口测试

覆盖：

- 互评资格检查接口返回 `eligible=false + reasonCode`
- 提交互评失败返回对应 `reasonCode`
- 重复评价返回 `409` 或约定冲突响应
- 评价列表不向被评价者展示 `voided`
- 信誉流水列表展示 `effective=false` 并标注挂起
- 管理端评价复核要求管理员鉴权
- 评价申诉重复提交返回 `DUPLICATE_APPEAL`
- 已无效评价申诉返回 `EVALUATION_ALREADY_INVALIDATED`

## 边界测试

覆盖：

- 互评窗口最后时刻
- 互评窗口过期后一刻
- 四维均值刚好落在区间边界
- 项目只有两名成员
- 全低分 / 全满分覆盖全部队友的临界提交
- 多条评价同时触发项目级封顶
- 复核与申诉恢复涉及同一评价时的数据一致性

## 事务测试

关键链路：

- 提交互评中任一步失败，`evaluation`、`credit_change`、`user.credit_score` 不应半更新
- 异常挂起中任一步失败，评价状态、流水 effective、信誉缓存不应撕裂
- 复核中任一步失败，状态与信誉不应不一致
- 申诉恢复中任一步失败，不应产生半截 `appeal_restore`

## 测试证据

每次 M5 任务完成后记录：

- 测试命令
- 测试范围
- 通过 / 失败结果
- 失败原因与修复结果
- 无法运行测试的原因
- 追加到 `IMPLEMENT_LOG.md`

## 测试证据交付口径

M5 测试证据统一放到：

```text
docs/test-evidence/m5/
```

目录结构建议：

```text
docs/test-evidence/m5/
├─ unit/
│  ├─ evaluation-eligibility.md
│  ├─ credit-calculation.md
│  ├─ project-cap.md
│  └─ ...
├─ integration/
│  ├─ evaluation-submit.md
│  ├─ abnormal-detection.md
│  ├─ review-flow.md
│  └─ ...
└─ api/
   ├─ evaluation-api.md
   ├─ credit-api.md
   └─ ...
```

每个证据文件包含：

- 测试目标
- 测试命令
- 测试输入
- 预期输出
- 实际输出
- 通过 / 失败判断
- 截图或日志片段（可选）

`IMPLEMENT_LOG.md` 只追加摘要：

- 测试范围
- 通过 / 失败结果
- 证据文件路径

不要在 `IMPLEMENT_LOG.md` 中粘贴完整测试日志。
