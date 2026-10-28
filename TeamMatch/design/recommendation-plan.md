# 推荐算法改造方案（终版）

## 架构：三阶段级联过滤

```
请求进来
    │
    ▼
阶段一：硬约束过滤（SQL层）
    credit_score < 80     → 直接过滤
    Jaccard < 0.15        → 直接过滤
    user == projectOwner  → 直接过滤（队长不推荐给自己）
    已在项目中 / 有pending邀请 → 直接过滤
    │
    ▼
阶段二：技能匹配粗排
    通过硬约束的人，按 Jaccard 相似度排序
    │
    ▼
阶段三：可信度精排
    FinalScore = Jaccard × 技术能力可信度 × 合作可信度

    技术能力可信度 = 队友互评技术维度分 × 评价者信誉分 / 总权重
                     × 贝叶斯混合 GitHub 先验
    合作可信度     = credit_score / 100
```

---

## 改动范围

| 类型 | 文件 | 路径 |
|------|------|------|
| 新增 | `SkillAuthorityService.java` | `src/main/java/com/teammatch/service/` |
| 新增 | `EndorsementPoint.java` | `src/main/java/com/teammatch/dto/` |
| 新增 | `RecommendationItem.java` | `src/main/java/com/teammatch/dto/` |
| 新增 | `RecommendationController.java` | `src/main/java/com/teammatch/m4/controller/` |
| 新增 | `RecommendationService.java` | `src/main/java/com/teammatch/m4/service/` |
| 新增 | `RecommendationServiceImpl.java` | `src/main/java/com/teammatch/m4/service/impl/` |
| 修改 | `EvaluationMapper.java` | `src/main/java/com/teammatch/mapper/` |

不修改的文件：schema.sql、CreditRuleServiceImpl、AnomalyDetectionService、ExitVoteServiceImpl、AppealRestoreServiceImpl

---

## 文件1：EndorsementPoint.java（新增）

```java
package com.teammatch.dto;

import lombok.Data;

@Data
public class EndorsementPoint {
    private int skillScore;
    private int evaluatorCreditScore;
    private Long evaluatorId;
    private Long targetUserId; // 被评价人，用于批量查询后分组
}
```

---

## 文件2：RecommendationItem.java（新增）

```java
package com.teammatch.dto;

import lombok.Data;
import java.util.List;

@Data
public class RecommendationItem {
    private Long userId;
    private String nickname;
    private String avatarUrl;
    private int creditScore;
    private int techScore;
    private double jaccardSimilarity;
    private double techAuthority;
    private double trustFactor;
    private double finalScore;
    private List<String> matchedSkills;
    private boolean passedHardFilter;
    private String authorityBreakdown; // "3次互评，加权信誉94"
}
```

---

## 文件3：SkillAuthorityService.java（新增）

```java
package com.teammatch.service;

import com.teammatch.dto.EndorsementPoint;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;

import java.util.Collections;
import java.util.List;

@Service
@RequiredArgsConstructor
public class SkillAuthorityService {

    private static final double PRIOR_EQUIVALENT_WEIGHT = 5.0;
    private static final int MIN_EVALUATOR_CREDIT = 30;
    private static final double DEFAULT_PRIOR = 0.3;

    /**
     * 计算用户的技术能力可信度。
     * 对所有重叠技能返回同一份互评数据的加权平均。
     * skill_score 是四维评价中的整体技术能力评分，不分具体技能标签。
     *
     * @param points              该用户收到的所有互评技术维度打分（调用方批量查好传入，避免N+1）
     * @param overlappingSkillIds 与项目需求重叠的技能ID列表
     * @param techPriorNormalized techScore归一化值 [0,1]，null表示无GitHub数据
     */
    public double computeAuthority(List<EndorsementPoint> points,
                                    List<Long> overlappingSkillIds,
                                    Double techPriorNormalized) {
        if (overlappingSkillIds == null || overlappingSkillIds.isEmpty()) {
            return DEFAULT_PRIOR;
        }
        if (points == null) points = Collections.emptyList();

        double singleAuth = computeSingle(points, techPriorNormalized);

        // 每个重叠技能挂同一份可信度（skill_score是整体评价）
        return singleAuth;
    }

    /**
     * 公开此方法供外部复用（如 buildBreakdown）
     */
    public double computeSingle(List<EndorsementPoint> points, Double techPrior) {
        double weightedSum = 0;
        double totalWeight = 0;

        for (EndorsementPoint p : points) {
            if (p.getEvaluatorCreditScore() < MIN_EVALUATOR_CREDIT) continue;
            double weight = p.getEvaluatorCreditScore() / 100.0;
            weightedSum += weight * (p.getSkillScore() / 5.0);
            totalWeight += weight;
        }

        if (totalWeight == 0) {
            return techPrior != null ? techPrior : DEFAULT_PRIOR;
        }

        double evaluationTrust = weightedSum / totalWeight;

        if (techPrior != null) {
            return (weightedSum + techPrior * PRIOR_EQUIVALENT_WEIGHT)
                 / (totalWeight + PRIOR_EQUIVALENT_WEIGHT);
        }
        return evaluationTrust;
    }
}
```

---

## 文件4：EvaluationMapper.java（修改：新增方法）

新增方法：

```java
/**
 * 批量查多个用户的互评技术维度打分（避免N+1）
 */
@Select("SELECT e.skill_score AS skillScore, " +
        "       u.credit_score AS evaluatorCreditScore, " +
        "       e.evaluator_id AS evaluatorId, " +
        "       e.target_id AS targetUserId " +
        "FROM evaluation e " +
        "JOIN user u ON e.evaluator_id = u.id " +
        "WHERE e.target_id IN (" +
        "  <foreach collection='userIds' item='id' separator=','>#{id}</foreach>" +
        ") " +
        "  AND e.status = 'normal'")
List<EndorsementPoint> findSkillScoresByTargets(@Param("userIds") List<Long> userIds);
```

---

## 文件5：RecommendationService.java（新增）

```java
package com.teammatch.m4.service;

import com.teammatch.dto.RecommendationItem;
import java.util.List;

public interface RecommendationService {
    List<RecommendationItem> recommend(Long projectId, int limit);
}
```

---

## 文件6：RecommendationServiceImpl.java（新增，核心逻辑）

```java
package com.teammatch.m4.service.impl;

import com.teammatch.dto.EndorsementPoint;
import com.teammatch.dto.RecommendationItem;
import com.teammatch.entity.*;
import com.teammatch.m4.service.RecommendationService;
import com.teammatch.mapper.*;
import com.teammatch.service.SkillAuthorityService;
import com.baomidou.mybatisplus.core.conditions.query.LambdaQueryWrapper;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;

import java.util.*;
import java.util.stream.Collectors;

@Service
@RequiredArgsConstructor
public class RecommendationServiceImpl implements RecommendationService {

    private static final int CREDIT_HARD_MIN = 80;
    private static final double JACCARD_HARD_MIN = 0.15;
    private static final double MAX_TECH_SCORE = 2000.0;

    private final UserMapper userMapper;
    private final ProjectMapper projectMapper;
    private final TechProfileMapper techProfileMapper;
    private final ProjectSkillMapper projectSkillMapper;
    private final UserSkillMapper userSkillMapper;
    private final TeamMemberMapper teamMemberMapper;
    private final TeamRequestMapper teamRequestMapper;
    private final SkillTagMapper skillTagMapper;
    private final EvaluationMapper evaluationMapper;
    private final SkillAuthorityService skillAuthorityService;

    @Override
    public List<RecommendationItem> recommend(Long projectId, int limit) {
        Project project = projectMapper.selectById(projectId);
        if (project == null) return Collections.emptyList();

        // 项目所需技能ID列表
        Set<Long> projectSkillIds = projectSkillMapper.selectByProjectId(projectId).stream()
                .map(ProjectSkill::getSkillTagId).collect(Collectors.toSet());

        // 阶段一：硬约束 SQL 过滤
        List<User> candidates = userMapper.selectList(new LambdaQueryWrapper<User>()
                .ge(User::getCreditScore, CREDIT_HARD_MIN)
                .eq(User::getStatus, "active")
                .eq(User::getFormalProfileCompleted, 1)
                .ne(User::getId, project.getCreatorId())                        // 排除队长自己
                .notInSql("id", "SELECT user_id FROM team_member " +
                         "WHERE project_id = " + projectId + " AND status = 'active'")
                .notInSql("id", "SELECT to_user_id FROM team_request " +
                         "WHERE project_id = " + projectId +
                         " AND request_type = 'invite' AND status = 'pending'")
        );

        if (candidates.isEmpty()) return Collections.emptyList();

        // === 批量预取，避免N+1 ===

        List<Long> candidateIds = candidates.stream().map(User::getId).toList();

        // tech_profile 批量取
        Map<Long, TechProfile> techProfileMap = techProfileMapper.selectBatchIds(candidateIds).stream()
                .collect(Collectors.toMap(TechProfile::getClaimedByUserId, tp -> tp, (a, b) -> a));

        // user_skill 批量取
        Map<Long, Set<Long>> skillsByUser = userSkillMapper.selectByUserIds(candidateIds).stream()
                .collect(Collectors.groupingBy(
                        UserSkill::getUserId,
                        Collectors.mapping(UserSkill::getSkillTagId, Collectors.toSet())));

        // evaluation 互评技术分批量取
        Map<Long, List<EndorsementPoint>> pointsByUser = evaluationMapper
                .findSkillScoresByTargets(candidateIds).stream()
                .collect(Collectors.groupingBy(EndorsementPoint::getTargetUserId));

        // skillTag 名称批量取
        Set<Long> allSkillIds = new HashSet<>(projectSkillIds);
        skillsByUser.values().forEach(allSkillIds::addAll);
        Map<Long, String> skillNameMap = skillTagMapper.selectBatchIds(allSkillIds).stream()
                .collect(Collectors.toMap(SkillTag::getId, SkillTag::getName));

        // === 阶段二 + 阶段三：遍历候选 ===

        List<RecommendationItem> items = new ArrayList<>();
        for (User u : candidates) {
            Set<Long> userSkillIds = skillsByUser.getOrDefault(u.getId(), Collections.emptySet());

            // 一次计算 Jaccard 和重叠技能
            Set<Long> overlapSet = new HashSet<>(projectSkillIds);
            overlapSet.retainAll(userSkillIds);
            double jaccard = computeJaccard(projectSkillIds, userSkillIds, overlapSet);
            List<Long> overlapping = new ArrayList<>(overlapSet);

            // 阶段一补充：Jaccard 硬约束
            if (jaccard < JACCARD_HARD_MIN) continue;

            // 阶段三：精排
            TechProfile tp = techProfileMap.get(u.getId());
            Double techPrior = (tp != null && tp.getTechScore() > 0)
                    ? Math.min(tp.getTechScore() / MAX_TECH_SCORE, 1.0) : null;

            List<EndorsementPoint> points = pointsByUser.getOrDefault(u.getId(), Collections.emptyList());
            double authority = skillAuthorityService.computeAuthority(points, overlapping, techPrior);
            double trustFactor = Math.min(u.getCreditScore() / 100.0, 1.0);
            double finalScore = jaccard * authority * trustFactor;

            // 诊断文案
            String breakdown = buildBreakdown(points);

            // 技能名称
            List<String> matchedNames = overlapping.stream()
                    .map(skillNameMap::get).filter(Objects::nonNull).toList();

            RecommendationItem item = new RecommendationItem();
            item.setUserId(u.getId());
            item.setNickname(u.getNickname());
            item.setAvatarUrl(u.getAvatarUrl());
            item.setCreditScore(u.getCreditScore());
            item.setTechScore(tp != null ? tp.getTechScore() : 0);
            item.setJaccardSimilarity(round2(jaccard));
            item.setTechAuthority(round2(authority));
            item.setTrustFactor(round2(trustFactor));
            item.setFinalScore(round2(finalScore));
            item.setMatchedSkills(matchedNames);
            item.setPassedHardFilter(true);
            item.setAuthorityBreakdown(breakdown);
            items.add(item);
        }

        items.sort((a, b) -> Double.compare(b.getFinalScore(), a.getFinalScore()));
        return items.subList(0, Math.min(limit, items.size()));
    }

    // === 工具方法 ===

    private double computeJaccard(Set<Long> projectSkills, Set<Long> userSkills, Set<Long> overlap) {
        if (projectSkills.isEmpty()) return 1.0;
        Set<Long> union = new HashSet<>(projectSkills);
        union.addAll(userSkills);
        if (union.isEmpty()) return 0;
        return (double) overlap.size() / union.size();
    }

    private String buildBreakdown(List<EndorsementPoint> points) {
        if (points == null || points.isEmpty()) return "暂无互评数据";
        long validCount = points.stream()
                .filter(p -> p.getEvaluatorCreditScore() >= 30).count();
        double avgCredit = points.stream()
                .filter(p -> p.getEvaluatorCreditScore() >= 30)
                .mapToInt(EndorsementPoint::getEvaluatorCreditScore)
                .average().orElse(0);
        return validCount + "次互评认证，加权信誉" + Math.round(avgCredit);
    }

    private double round2(double v) {
        return Math.round(v * 100.0) / 100.0;
    }
}
```

---

## 文件7：RecommendationController.java（新增）

```java
package com.teammatch.m4.controller;

import com.teammatch.common.Result;
import com.teammatch.dto.RecommendationItem;
import com.teammatch.m4.service.RecommendationService;
import lombok.RequiredArgsConstructor;
import org.springframework.web.bind.annotation.*;

import java.util.List;

@RestController
@RequestMapping("/api/m4/projects")
@RequiredArgsConstructor
public class RecommendationController {

    private final RecommendationService recommendationService;

    @GetMapping("/{projectId}/recommendations")
    public Result<List<RecommendationItem>> recommend(
            @PathVariable Long projectId,
            @RequestParam(defaultValue = "20") int limit) {
        return Result.success(recommendationService.recommend(projectId, limit));
    }
}
```

---

## 需要注意的依赖前提（AI 实现前必须验证）

1. **UserSkillMapper 是否有 `selectByUserIds` 批量方法**——如果没有，使用 MyBatis-Plus 的 `selectList(new LambdaQueryWrapper<UserSkill>().in(UserSkill::getUserId, userIds))`
2. **ProjectSkillMapper 是否有 `selectByProjectId`**——如果没有，用 LambdaQueryWrapper 等效
3. **TechProfile 的 claimedByUserId 字段名是否正确**——读实体类确认
4. **ProjectMapper.selectById** 是否返回 creatorId —— 确认字段映射
5. **TeamMemberMapper / TeamRequestMapper** 表名和字段名是否和硬约束SQL匹配
6. **SkillTag 的 name 字段**用于诊断面板展示

---

## 设计决策说明

| 决策 | 原因 |
|------|------|
| Jaccard硬约束 0.15 | 太低无意义，太高误拦"匹配少但愿意学"的人 |
| 信誉硬约束 80 | 跑路过的人不进推荐 |
| 排除队长自己 | 队长不需要被推荐给自己 |
| skill_score不分技能维度 | 四维评价的skill_score是整体技术能力，不是分Java/Python的打分 |
| PRIOR_WEIGHT=5 | 3-5次互评后真人评价接管，不为100是因为互评权重大约是credit/100≈1 |
| trustFactor只用credit_score | tech_score已在authority先验里出现，不重复计算 |
| FinalScore用乘法 | 每个维度是闸门不是加分项 |
| 批量预取5张表 | 一次推荐只发5条SQL，不随候选人数增长 |

---

## 接口

```
GET /api/m4/projects/{projectId}/recommendations?limit=20

Response:
{
  "code": 200,
  "data": [
    {
      "userId": 1,
      "nickname": "张三",
      "creditScore": 92,
      "techScore": 850,
      "jaccardSimilarity": 0.85,
      "techAuthority": 0.72,
      "trustFactor": 0.92,
      "finalScore": 0.56,
      "matchedSkills": ["Java", "Vue"],
      "passedHardFilter": true,
      "authorityBreakdown": "3次互评认证，加权信誉94"
    }
  ]
}
```

---

## 前端诊断面板样式

每个推荐卡片下方展示：

```
[✓硬约束]  [技能匹配：85% (Java, Vue)]  [技术认证：3次互评 | 加权信誉94]  → 综合 0.56
```

---

## 答辩话术（2分钟）

> "推荐做了三层。第一层硬约束：信誉低于80或技能匹配低于15%或队长自己不出现。
> 第二层Jaccard粗排。第三层是核心——自填标签不完全可信，
> 我们用互评里队友给的技术评分做置信度加权。
> 同样是'会Java'，被高信誉队友认证过的排序完全不一样。
> 新用户没有互评数据就用GitHub画像做先验，互评多了先验自动退场——
> 推荐从'他说他会什么'切换到'别人认证他技术行不行'。"
