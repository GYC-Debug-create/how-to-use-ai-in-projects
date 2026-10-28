# recommendation-plan.md 编译审查与修正版

## 结论

当前后端基线可编译：`mvn -q -DskipTests compile` 通过。

原方案不能直接照抄实现，主要编译失败点集中在 `RecommendationServiceImpl` 和 `EvaluationMapper`：

| 文件 | 结论 | 问题 | 替代写法 |
|---|---|---|---|
| `EndorsementPoint.java` | 可新增 | 新文件无冲突，字段可被 SQL alias 映射 | 保持即可 |
| `RecommendationItem.java` | 可新增 | 新文件无冲突，Lombok 可生成 setter/getter | 保持即可 |
| `SkillAuthorityService.java` | 可新增 | 无 Bean 冲突；`overlappingSkillIds` 只用于空判断，语义上不是分技能计算 | 可保留，或改名为 `TechnicalTrustService` |
| `EvaluationMapper.java` | 原写法运行失败 | 方案里的 `<foreach>` 没有包在 `<script>` 中；缺少 `EndorsementPoint` import；`JOIN user` 建议加反引号 | 使用下方修正版方法 |
| `RecommendationService.java` | 可新增 | 无冲突 | 保持即可 |
| `RecommendationServiceImpl.java` | 编译失败/逻辑错误 | `ProjectSkillMapper`、`TeamRequestMapper` 不存在；应使用 M4 mapper；`selectByProjectId`、`selectByUserIds` 不存在；`techProfileMapper.selectBatchIds(candidateIds)` 用用户 id 查画像主键，数据错误 | 使用下方修正版 |
| `RecommendationController.java` | 路径错误 | 项目已有 `server.servlet.context-path=/api`，Controller 不能再写 `/api`，否则实际路径变 `/api/api/...` | `@RequestMapping("/m4/projects")` |

## 路径和 Bean 冲突

- 新增 `RecommendationService` / `RecommendationServiceImpl`：当前项目没有同名 Service，Bean 不冲突。
- 新增 `SkillAuthorityService`：当前项目没有同名 Service，Bean 不冲突。
- 新增 `RecommendationController`：和 `ProjectController` 共用 `/m4/projects` 前缀，但新增 `GET /{projectId}/recommendations`，当前没有同路径方法，不冲突。
- 不应注入不存在的 `TeamRequestMapper`；当前只有 `com.teammatch.m4.mapper.M4TeamRequestMapper`。
- M4 推荐逻辑应使用 `com.teammatch.m4.mapper.M4ProjectMapper/M4ProjectSkillMapper` 和 `com.teammatch.m4.entity.Project/ProjectSkill`，不要混用 M5 根包 `ProjectMapper/Project`。

## M3/M5/M6 影响

- M3：只读 `user`、`user_skill`、`tech_profile`，不写认证/档案数据，不影响现有 M3。
- M5：只给 `EvaluationMapper` 增加一个只读方法，不改现有方法签名；互评、信誉流水、异常检测不受影响。
- M6：不改治理 Controller/Service/Mapper；只读取正常互评 `evaluation.status='normal'`，不影响申诉/复核/处罚。

## N+1 审查

修正后推荐链路没有候选人循环内查询：

| 预取 | 原方案状态 | 修正 |
|---|---|---|
| 项目技能 | `selectByProjectId` 不存在 | `M4ProjectSkillMapper.selectList(projectId wrapper)` 一次查询 |
| 候选用户 | 一次 `user` 查询，含子查询 | 保留 |
| 技术画像 | `selectBatchIds(candidateIds)` 查错主键 | 用候选人的 `techProfileId` 批量查 `tech_profile.id` |
| 用户技能 | `selectByUserIds` 不存在 | `UserSkillMapper.selectList(userId IN candidateIds)` 一次查询 |
| 互评技术分 | SQL 动态脚本错误 | `EvaluationMapper.findSkillScoresByTargets(candidateIds)` 一次查询 |
| 技能名 | 一次 `selectBatchIds(allSkillIds)` | 保留，并对空集合做 guard |

循环内只读内存 Map：`techProfileByUser`、`skillsByUser`、`pointsByUser`、`skillNameMap`。

## 修正版：EvaluationMapper.java 新增内容

需要新增 import：

```java
import com.teammatch.dto.EndorsementPoint;
```

新增方法：

```java
@Select("<script>" +
        "SELECT e.skill_score AS skillScore, " +
        "       u.credit_score AS evaluatorCreditScore, " +
        "       e.evaluator_id AS evaluatorId, " +
        "       e.target_id AS targetUserId " +
        "FROM evaluation e " +
        "JOIN `user` u ON e.evaluator_id = u.id " +
        "WHERE e.target_id IN " +
        "<foreach collection='userIds' item='id' open='(' separator=',' close=')'>" +
        "#{id}" +
        "</foreach> " +
        "AND e.status = 'normal'" +
        "</script>")
List<EndorsementPoint> findSkillScoresByTargets(@Param("userIds") List<Long> userIds);
```

## 修正版：RecommendationController.java

```java
package com.teammatch.m4.controller;

import com.teammatch.common.Result;
import com.teammatch.dto.RecommendationItem;
import com.teammatch.m4.service.RecommendationService;
import lombok.RequiredArgsConstructor;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

import java.util.List;

@RestController
@RequestMapping("/m4/projects")
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

## 修正版：RecommendationServiceImpl.java

```java
package com.teammatch.m4.service.impl;

import com.baomidou.mybatisplus.core.conditions.query.LambdaQueryWrapper;
import com.teammatch.dto.EndorsementPoint;
import com.teammatch.dto.RecommendationItem;
import com.teammatch.entity.SkillTag;
import com.teammatch.entity.TechProfile;
import com.teammatch.entity.User;
import com.teammatch.entity.UserSkill;
import com.teammatch.m4.entity.Project;
import com.teammatch.m4.entity.ProjectSkill;
import com.teammatch.m4.mapper.M4ProjectMapper;
import com.teammatch.m4.mapper.M4ProjectSkillMapper;
import com.teammatch.m4.service.RecommendationService;
import com.teammatch.mapper.EvaluationMapper;
import com.teammatch.mapper.SkillTagMapper;
import com.teammatch.mapper.TechProfileMapper;
import com.teammatch.mapper.UserMapper;
import com.teammatch.mapper.UserSkillMapper;
import com.teammatch.service.SkillAuthorityService;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;

import java.util.ArrayList;
import java.util.Collections;
import java.util.HashMap;
import java.util.HashSet;
import java.util.List;
import java.util.Map;
import java.util.Objects;
import java.util.Set;
import java.util.function.Function;
import java.util.stream.Collectors;

@Service
@RequiredArgsConstructor
public class RecommendationServiceImpl implements RecommendationService {

    private static final int CREDIT_HARD_MIN = 80;
    private static final double JACCARD_HARD_MIN = 0.15;
    private static final double MAX_TECH_SCORE = 2000.0;

    private final UserMapper userMapper;
    private final M4ProjectMapper projectMapper;
    private final M4ProjectSkillMapper projectSkillMapper;
    private final TechProfileMapper techProfileMapper;
    private final UserSkillMapper userSkillMapper;
    private final SkillTagMapper skillTagMapper;
    private final EvaluationMapper evaluationMapper;
    private final SkillAuthorityService skillAuthorityService;

    @Override
    public List<RecommendationItem> recommend(Long projectId, int limit) {
        if (projectId == null || limit <= 0) {
            return Collections.emptyList();
        }

        Project project = projectMapper.selectById(projectId);
        if (project == null) {
            return Collections.emptyList();
        }

        Set<Long> projectSkillIds = projectSkillMapper.selectList(
                new LambdaQueryWrapper<ProjectSkill>()
                        .eq(ProjectSkill::getProjectId, projectId))
                .stream()
                .map(ProjectSkill::getSkillTagId)
                .collect(Collectors.toSet());

        List<User> candidates = userMapper.selectList(new LambdaQueryWrapper<User>()
                .ge(User::getCreditScore, CREDIT_HARD_MIN)
                .eq(User::getStatus, "active")
                .eq(User::getFormalProfileCompleted, Boolean.TRUE)
                .ne(User::getId, project.getCreatorId())
                .notInSql("id", "SELECT user_id FROM team_member " +
                        "WHERE project_id = " + projectId + " AND status = 'active'")
                .notInSql("id", "SELECT to_user_id FROM team_request " +
                        "WHERE project_id = " + projectId +
                        " AND request_type = 'invite' AND status = 'pending'"));

        if (candidates.isEmpty()) {
            return Collections.emptyList();
        }

        List<Long> candidateIds = candidates.stream()
                .map(User::getId)
                .collect(Collectors.toList());

        Map<Long, TechProfile> techProfileByUser = fetchTechProfilesByUser(candidates);

        Map<Long, Set<Long>> skillsByUser = userSkillMapper.selectList(
                new LambdaQueryWrapper<UserSkill>()
                        .in(UserSkill::getUserId, candidateIds))
                .stream()
                .collect(Collectors.groupingBy(
                        UserSkill::getUserId,
                        Collectors.mapping(UserSkill::getSkillTagId, Collectors.toSet())));

        Map<Long, List<EndorsementPoint>> pointsByUser = evaluationMapper
                .findSkillScoresByTargets(candidateIds)
                .stream()
                .collect(Collectors.groupingBy(EndorsementPoint::getTargetUserId));

        Set<Long> allSkillIds = new HashSet<>(projectSkillIds);
        skillsByUser.values().forEach(allSkillIds::addAll);
        Map<Long, String> skillNameMap = allSkillIds.isEmpty()
                ? Collections.emptyMap()
                : skillTagMapper.selectBatchIds(allSkillIds).stream()
                        .collect(Collectors.toMap(SkillTag::getId, SkillTag::getName));

        List<RecommendationItem> items = new ArrayList<>();
        for (User user : candidates) {
            Set<Long> userSkillIds = skillsByUser.getOrDefault(user.getId(), Collections.emptySet());

            Set<Long> overlapSet = new HashSet<>(projectSkillIds);
            overlapSet.retainAll(userSkillIds);
            double jaccard = computeJaccard(projectSkillIds, userSkillIds, overlapSet);
            if (jaccard < JACCARD_HARD_MIN) {
                continue;
            }

            TechProfile techProfile = techProfileByUser.get(user.getId());
            Double techPrior = normalizeTechPrior(techProfile);
            List<EndorsementPoint> points = pointsByUser.getOrDefault(user.getId(), Collections.emptyList());

            double techAuthority = skillAuthorityService.computeAuthority(
                    points, new ArrayList<>(overlapSet), techPrior);
            double trustFactor = Math.min(Math.max(user.getCreditScore() / 100.0, 0.0), 1.0);
            double finalScore = jaccard * techAuthority * trustFactor;

            RecommendationItem item = new RecommendationItem();
            item.setUserId(user.getId());
            item.setNickname(user.getNickname());
            item.setAvatarUrl(user.getAvatarUrl());
            item.setCreditScore(user.getCreditScore());
            item.setTechScore(techProfile != null && techProfile.getTechScore() != null
                    ? techProfile.getTechScore() : 0);
            item.setJaccardSimilarity(round2(jaccard));
            item.setTechAuthority(round2(techAuthority));
            item.setTrustFactor(round2(trustFactor));
            item.setFinalScore(round2(finalScore));
            item.setMatchedSkills(overlapSet.stream()
                    .map(skillNameMap::get)
                    .filter(Objects::nonNull)
                    .collect(Collectors.toList()));
            item.setPassedHardFilter(true);
            item.setAuthorityBreakdown(buildBreakdown(points));
            items.add(item);
        }

        items.sort((a, b) -> Double.compare(b.getFinalScore(), a.getFinalScore()));
        return items.subList(0, Math.min(limit, items.size()));
    }

    private Map<Long, TechProfile> fetchTechProfilesByUser(List<User> candidates) {
        Map<Long, Long> profileIdByUser = candidates.stream()
                .filter(user -> user.getTechProfileId() != null)
                .collect(Collectors.toMap(User::getId, User::getTechProfileId, (a, b) -> a));

        if (profileIdByUser.isEmpty()) {
            return Collections.emptyMap();
        }

        Map<Long, TechProfile> profileById = techProfileMapper
                .selectBatchIds(new HashSet<>(profileIdByUser.values()))
                .stream()
                .collect(Collectors.toMap(TechProfile::getId, Function.identity(), (a, b) -> a));

        Map<Long, TechProfile> result = new HashMap<>();
        profileIdByUser.forEach((userId, profileId) -> {
            TechProfile profile = profileById.get(profileId);
            if (profile != null) {
                result.put(userId, profile);
            }
        });
        return result;
    }

    private Double normalizeTechPrior(TechProfile profile) {
        if (profile == null || profile.getTechScore() == null || profile.getTechScore() <= 0) {
            return null;
        }
        return Math.min(Math.max(profile.getTechScore() / MAX_TECH_SCORE, 0.0), 1.0);
    }

    private double computeJaccard(Set<Long> projectSkills, Set<Long> userSkills, Set<Long> overlap) {
        if (projectSkills.isEmpty()) {
            return 1.0;
        }
        Set<Long> union = new HashSet<>(projectSkills);
        union.addAll(userSkills);
        return union.isEmpty() ? 0.0 : (double) overlap.size() / union.size();
    }

    private String buildBreakdown(List<EndorsementPoint> points) {
        if (points == null || points.isEmpty()) {
            return "暂无互评数据";
        }
        List<EndorsementPoint> valid = points.stream()
                .filter(point -> point.getEvaluatorCreditScore() >= 30)
                .collect(Collectors.toList());
        if (valid.isEmpty()) {
            return "暂无有效互评数据";
        }
        double avgCredit = valid.stream()
                .mapToInt(EndorsementPoint::getEvaluatorCreditScore)
                .average()
                .orElse(0);
        return valid.size() + "次互评认证，加权信誉" + Math.round(avgCredit);
    }

    private double round2(double value) {
        return Math.round(value * 100.0) / 100.0;
    }
}
```
