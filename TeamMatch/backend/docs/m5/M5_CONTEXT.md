# M5 精简上下文

本文件是 TeamMatch M5 开发的低 token 默认入口。

## 最高依据

唯一最高业务依据仍是：

```text
docs/design/system-design-v2.1.md
```

## 分层阅读协议

**每个新会话首次处理 M5 前**，必须读取 `docs/design/system-design-v2.1.md` 的关键章节：

- 0.3 冻结规则
- 0.4 P0/P1 边界
- 1.6 DDL 顺序
- 6 板块五：互评与信誉
- 7 中 evaluation / appeal / review 相关内容
- 8 reasonCode
- 9.1 事务边界

**同一会话后续小任务**，默认使用 `docs/m5/` 精简上下文：

1. `CLAUDE.md`
2. `docs/m5/M5_CONTEXT.md`
3. `docs/m5/M5_TODO.md`
4. 当前要改的代码文件
5. `IMPLEMENT_LOG.md` 最后一段

**遇到以下情况必须回查 V2.1**：

- 冲突
- 歧义
- 跨模块
- 状态机
- 事务
- reasonCode
- credit_change

## 禁止默认读取

- `docs/archive/`
- PDF
- Excel
- 旧功能设计
- 旧立项书

## M5/M6 边界

**M5 负责**：

- `evaluation`
- `evaluation_tag`
- `skill_endorse`
- `credit_change`
- `evaluation` 类 `appeal`
- 评价复核
- 申诉恢复
- 信誉分计算
- 互评资格
- 互评提交
- 异常评价挂起
- M5 单元测试、接口测试、边界测试

**M6 负责**：

- `report`
- `penalty`
- 治理后台入口 / 工作台
- Jenkins
- 部署

**M5 不负责但可能读取**：

- 微信登录、Token、邮箱认证、档案、技能标签基础维护（M3）
- 项目发布、组队请求、`team_member`、`exit_vote` 状态机实现（M4）

M5 可以调用这些模块的查询结果或服务能力，但不要在 M5 任务中扩张修改它们。

## 核心业务规则

- 当前用户必须是项目正式成员才可互评
- 目标用户必须是项目正式成员才可被评
- 项目必须处于 `ended`
- 必须在项目结束后 7 日互评窗口内
- 同一项目内同一对象只能评价一次
- 不能评价自己
- 每条评价包含四维：沟通协作（communication_score）、任务完成（task_score）、技术能力（skill_score）、责任心（responsibility_score）
- 系统保存四维明细分，再按四维均值计算信誉变化
- 任一维度为 1 / 2 分时，必须附负向标签和不少于 20 字说明
- 任一维度为 5 分时，必须附正向标签
- P0 不等待全员提交

## 信誉分规则

初始分为 100。

| 均分区间 | `change_value` |
|---|---:|
| [1.0, 1.5) | -5 |
| [1.5, 2.0) | -3 |
| [2.0, 2.5) | -2 |
| [2.5, 3.0) | -1 |
| [3.0, 3.5) | 0 |
| [3.5, 4.0) | +2 |
| [4.0, 4.5) | +3 |
| [4.5, 5.0] | +5 |

同一用户在同一项目中的互评分变化总和不超过 ±10。

`user.credit_score` 是缓存：

```text
100 + SUM(credit_change.change_value WHERE effective=1)
```

必须通过 `credit_change` 追加流水，不得绕开流水直接改信誉分。

## 异常评价规则

P0 只保留两条基础异常检测：

- 全低分：评价者对同项目所有被评价者的 4 维度均分 ≤ 2
- 全满分：评价者对同项目所有被评价者的 4 维度均分 = 5

检测时机：评价者覆盖全部队友后同步触发。

命中后：将该评价者在该项目中的全部评价批量标记为 `pending_review`，对应 `credit_change.effective=0`，异常评价不能直接计分。

## 申诉与复核

评价争议：

- 用户侧走 `appeal(target_type='evaluation')`
- 管理员侧走 `evaluation review`
- 不走 `report`

评价复核结果：

- `approve`：恢复为 `normal`
- `void`：作废
- `keep_no_credit`：保留展示但不计分

申诉通过后：

- 不篡改原 `credit_change.effective`
- 新增 `change_type='appeal_restore'` 正向流水
- `change_value = 原流水相反数`
- 回写 `user.credit_score`
- 同一 `(source_type, source_id)` 已存在 `appeal_restore` 时跳过，保证幂等
- 原流水 `effective=0` 时跳过恢复

## M5 reasonCode

- `NOT_PROJECT_MEMBER`
- `TARGET_NOT_PROJECT_MEMBER`
- `PROJECT_NOT_ENDED`
- `EVAL_WINDOW_CLOSED`
- `ALREADY_EVALUATED`
- `SELF_EVALUATION`
- `DUPLICATE_APPEAL`
- `EVALUATION_ALREADY_INVALIDATED`

## 事务边界

以下 M5 链路必须在同一事务中完成：

- 提交互评
- 复核评价
- 通过申诉

涉及状态、`credit_change.effective`、`user.credit_score`、恢复流水时，任一步失败必须整体回滚。

## 默认工作方式

- 每次只做一个小任务
- 实现前先列出本次要改的文件和测试点
- 只改当前任务直接相关文件
- 不跨模块扩张
- 开发完成必须追加 `IMPLEMENT_LOG.md`

## 禁止旧口径

- 不改 7 日互评窗口
- 不绕开 `credit_change`
- 不把异常评价直接计分
- 不让评价争议走 `report`
- 不新增旧资料里的业务口径
