# M5-3 信誉分映射与单项目封顶 — 开发规划

版本：V1.0  
适用范围：TeamMatch 后端 M5-3 子任务  
前置完成：M5-1（互评资格校验）、M5-2（互评提交内容基础校验）

---

## 1. 业务目标

对一条已通过资格校验和内容校验的互评，根据四维评分计算 averageScore，映射 rawDelta，结合该用户在当前项目已有的 evaluation 类 credit_change 累计值，应用单项目 ±10 封顶，输出结构化计算结果。

**不落库、不写 credit_change、不更新 user.credit_score。**

---

## 2. 规则依据

| 规则 | 来源 | 内容 |
|------|------|------|
| 平均分计算 | schema.sql L214, Evaluation entity | `averageScore = (communication + task + skill + responsibility) / 4.0`，DECIMAL(3,2) |
| rawDelta 映射 | system-design-v2.1.md §6.5 L577-586 | 八区间映射表（见下方） |
| 单项目封顶 | system-design-v2.1.md §6.5 L588 | 同一用户在同一项目中的互评分变化总和不超过 ±10 |
| credit_change 与 user.credit_score | schema.sql L258-276, L28 | `user.credit_score = 100 + SUM(change_value WHERE effective=1)`，credit_score 是缓存字段 |
| M5-3 不写库 | M5_WORK_ORDER.md §4, AI_CONTEXT_INDEX.md §8 | 本轮只做纯计算+只读查询 |

### V2.1 八区间映射表

| 均分区间 | change_value (rawDelta) |
|----------|------------------------|
| [1.0, 1.5) | -5 |
| [1.5, 2.0) | -3 |
| [2.0, 2.5) | -2 |
| [2.5, 3.0) | -1 |
| [3.0, 3.5) | 0 |
| [3.5, 4.0) | +2 |
| [4.0, 4.5) | +3 |
| [4.5, 5.0] | +5 |

---

## 3. 当前代码现状

| 文件 | 路径 | 实际职责 |
|------|------|----------|
| CreditRuleService | `src/main/java/com/teammatch/m4/service/CreditRuleService.java` | 接口，定义 T-088 评分→delta 映射、T-089 单项目封顶 |
| CreditRuleServiceImpl | `src/main/java/com/teammatch/m4/service/impl/CreditRuleServiceImpl.java` | **映射只有 5 档简化阈值（BUG）**，封顶逻辑正确 |
| Evaluation | `src/main/java/com/teammatch/entity/Evaluation.java` | 四维评分 + averageScore(BigDecimal) + status |
| CreditChange | `src/main/java/com/teammatch/entity/CreditChange.java` | 信誉流水实体 |
| CreditChangeMapper | `src/main/java/com/teammatch/mapper/CreditChangeMapper.java` | 仅继承 BaseMapper，无自定义方法 |
| ReasonCode | `src/main/java/com/teammatch/common/ReasonCode.java` | M50xx/M51xx/M52xx，无 M53xx |
| Result\<T\> | `src/main/java/com/teammatch/common/Result.java` | 通用响应包装 |
| CreditRuleServiceImpl 测试 | **不存在** | 无任何测试覆盖 |

---

## 4. 复用性判断矩阵

| # | 能力 | 判定 | 证据 |
|---|------|------|------|
| 1 | 输入四维评分并计算 averageScore | 部分支持 | `CreditRuleServiceImpl:30-35` 计算了 avg 但未返回 BigDecimal，未暴露为独立方法 |
| 2 | averageScore 映射 rawDelta | 部分支持（BUG） | `CreditRuleServiceImpl:37-41` 只有 5 档，缺 [2.0,2.5)→-2、[2.5,3.0)→-1、[3.5,4.0)→+2 |
| 3 | existingProjectDelta + rawDelta 后应用 ±10 封顶 | 已支持 | `CreditRuleServiceImpl:45-73` 逻辑正确 |
| 4 | 输出结构化结果 | 不支持 | 当前只返回 int，无 DTO |
| 5 | 不落库、不写 credit_change | 已支持 | 现有方法只做 SELECT 和纯计算 |

---

## 5. 实施方案

### 5.1 必须新增

| 文件 | 职责 |
|------|------|
| `src/main/java/com/teammatch/dto/CreditCalculationResult.java` | 计算结果 DTO：averageScore / rawDelta / existingProjectDelta / cappedDelta / projectTotalAfterApplied / capped |
| `src/main/java/com/teammatch/service/CreditCalculationService.java` | M5-3 编排服务：调用 CreditRuleService，组装结构化结果 |
| `src/test/java/com/teammatch/service/CreditCalculationServiceTest.java` | 编排服务单元测试 |
| `src/test/java/com/teammatch/m4/service/CreditRuleServiceImplTest.java` | 底层映射+封顶逻辑单元测试 |

### 5.2 必须修改

| 文件 | 修改内容 |
|------|----------|
| `src/main/java/com/teammatch/m4/service/CreditRuleService.java` | 新增 `int mapAverageScoreToDelta(double avg)` 和 `int getExistingProjectDelta(Long userId, Long projectId)` |
| `src/main/java/com/teammatch/m4/service/impl/CreditRuleServiceImpl.java` | 修复映射为八区间；提取 mapAverageScoreToDelta 为独立方法；提取 getExistingProjectDelta 复用查询逻辑 |

### 5.3 不应触碰

| 文件 | 原因 |
|------|------|
| sql/schema.sql | 禁止修改数据库结构 |
| EvaluationValidationService | M5-2 已完成 |
| EvaluationEligibilityService | M5-1 已完成 |
| 任何 Controller | 本轮不写 |
| M4 ExitVoteService / TeamRequestController | 不跨模块 |

---

## 6. 命名与架构决策

- `CreditRuleService` 保留为跨 M4/M5 底层信誉规则服务（映射 + 封顶）
- 扩展方式：新增方法签名，不改变已有签名
- M4 退出扣分不受影响（grep 确认无其他代码调用 calculateDeltaFromEvaluation）
- M5-3 编排服务命名 `CreditCalculationService`，放 `com.teammatch.service`（与 M5-1/M5-2 一致）

---

## 7. 实施顺序

| 步骤 | 文件 | 类型 |
|------|------|------|
| 1 | `dto/CreditCalculationResult.java` | 新增 |
| 2 | `m4/service/CreditRuleService.java` | 修改接口 |
| 3 | `m4/service/impl/CreditRuleServiceImpl.java` | 修复映射 + 提取方法 |
| 4 | `service/CreditCalculationService.java` | 新增编排 |
| 5 | `m4/service/CreditRuleServiceImplTest.java` | 新增测试 |
| 6 | `service/CreditCalculationServiceTest.java` | 新增测试 |

每步完成后停止，向用户说明并等待确认。

---

## 8. 测试规划

### 8.1 映射区间边界测试（CreditRuleServiceImplTest）

| 用例 | 输入 (comm,task,skill,resp) | avg | 期望 rawDelta |
|------|---------------------------|-----|---------------|
| 全1分最低 | (1,1,1,1) | 1.00 | -5 |
| 边界1.5 | (1,2,1,2) | 1.50 | -3 |
| 边界2.0 | (2,2,2,2) | 2.00 | -2 |
| 边界2.5 | (2,3,2,3) | 2.50 | -1 |
| 边界3.0 | (3,3,3,3) | 3.00 | 0 |
| 边界3.5 | (3,4,3,4) | 3.50 | +2 |
| 边界4.0 | (4,4,4,4) | 4.00 | +3 |
| 边界4.5 | (4,5,4,5) | 4.50 | +5 |
| 全5分最高 | (5,5,5,5) | 5.00 | +5 |
| 上界不含4.49 | (4,5,4,4) | 4.25 | +3 |
| 下界不含2.99 | (3,3,3,2) | 2.75 | -1 |

### 8.2 封顶逻辑测试（CreditRuleServiceImplTest）

| 用例 | existingDelta | proposedDelta | 期望 cappedDelta |
|------|--------------|---------------|-----------------|
| 无需封顶 | +5 | +3 | +3 |
| 正向封顶 | +8 | +5 | +2 |
| 已达上限 | +10 | +3 | 0 |
| 负向封顶 | -8 | -5 | -2 |
| 已达下限 | -10 | -3 | 0 |
| 正负抵消 | +7 | -3 | -3 |
| 零delta直接返回 | +5 | 0 | 0 |

### 8.3 编排流程测试（CreditCalculationServiceTest）

| 用例 | 描述 |
|------|------|
| calculate_full_flow_no_cap | 完整流程，未触发封顶，验证全部 6 字段 |
| calculate_full_flow_with_cap | 触发正向封顶，capped=true |
| calculate_negative_cap | 负向封顶场景 |
| calculate_zero_existing | existingDelta=0 时正常计算 |
| calculate_already_at_cap | 已达上限时 cappedDelta=0 |
| calculate_boundary_4_49 | avg=4.25 映射到 +3 而非 +5 |

---

## 9. 禁止事项

本轮明确禁止：

1. 不写 Controller
2. 不写完整提交互评事务
3. 不插入 evaluation / evaluation_tag / skill_endorse / credit_change
4. 不更新 user.credit_score
5. 不处理 pending_review 状态
6. 不处理 appeal / report / penalty
7. 不修改 schema.sql
8. 不执行 git add / commit / push / reset / clean / rebase / merge

---

## 10. 风险点

1. **映射修复影响 M4**：grep 确认无其他代码调用 calculateDeltaFromEvaluation，修改安全
2. **重复 DB 查询**：calculate() 中 getExistingProjectDelta 和 calculateCappedDelta 各查一次。本轮保持简单，后续可优化为传参避免重复
3. **V2.1 文档位置（已校正）**：实际最高依据位于 `docs/design/system-design-v2.1.md`。旧口径“项目根目录 system-design-v2.1.md”已过期，不再使用。

---

## 11. 验证方式

1. `mvn test -Dtest=CreditRuleServiceImplTest` — 映射和封顶
2. `mvn test -Dtest=CreditCalculationServiceTest` — 编排
3. `mvn test` — 确认不破坏 M5-1/M5-2 已有测试
4. 检查新代码无 INSERT/UPDATE 语句

---

## 12. 最终结论

**只能部分复用，需要扩展方法。**

- 封顶逻辑（calculateCappedDelta）可直接复用
- 映射逻辑有 BUG（5 档 vs V2.1 八区间），需修复
- 缺少结构化 DTO 和编排服务，需新增
- 不需要全新独立服务，在 CreditRuleService 上扩展 + 新增 CreditCalculationService 编排层即可
