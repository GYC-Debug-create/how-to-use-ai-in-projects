# Gerrit Review 操作规则

> 记录审查中踩过的坑，减少后续审查的时间和 token 消耗。

## 项目结构约定

审查前先确认以下路径，不要猜测：

```text
项目根目录：仓库根（不是 backend/ 子目录）
Java 源码：  src/main/java/com/teammatch/
测试源码：  src/test/java/com/teammatch/
子模块：    com.teammatch.m4.* / com.teammatch.m6.*（独立包）
```

**用 `git diff --name-only FETCH_HEAD^1..FETCH_HEAD` 直接拿到变更文件列表，再用 `git show FETCH_HEAD:<路径>` 读文件。不要猜 `org/example/backend/` 或任何其他包路径。**

## 8 条核心规则

### 1. `ls-remote` 第一步

**问题**：在未 fetch 的 ref 上浪费多轮搜索。

**规则**：任何 Gerrit 审查操作前，先用 `git ls-remote` 确认目标 ref 是否存在于远程仓库，再决定是否 fetch。

```bash
git ls-remote origin | grep <关键词>
```

### 2. 对比远程/本地 refs

**问题**：盲目逐个尝试 ref，效率极低。

**规则**：先用 `ls-remote` 列出远程 refs，与本地 refs 对比，精准 fetch 仅缺失的 ref，不逐个盲目尝试。

```bash
git ls-remote origin refs/changes/* > remote_refs.txt
git show-ref | grep changes > local_refs.txt
# 对比差异后精准 fetch
```

### 3. Change-Id ≠ ref 路径

**问题**：被 `I22` 前缀误导，错误尝试去查 `changes/22/` 路径。用 `git ls-remote | grep Change-Id` 永远找不到——因为 ref 路径里不含 Change-Id 哈希。

**规则**：Change-Id（如 `I22abc...`）只是变更标识符，**不是** ref 路径组成部分。正确做法是**先用 Gerrit API 查出 change number**，再构造 ref 路径。

```bash
# 第 0 步：用 Gerrit API 通过 Change-Id 查 change number
curl -s "https://<user>:<pass>@（已脱敏）/a/changes/?q=<Change-Id>"
# 返回 JSON，取 "_number" 字段，如 3233

# 构造 ref 路径：refs/changes/<末两位>/<change_number>/<patchset>
# 例：change_number=3233 → refs/changes/33/3233/2

# 错误做法
git fetch origin refs/changes/22/<change-id>
git ls-remote | grep <Change-Id>   # 永远找不到

# 正确做法
git fetch origin refs/changes/<nn>/<change_number>/<patchset>
```

### 4. merge commit 用 `git diff`

**问题**：merge commit 使用 `diff-tree` 输出为空，被误导认为无变更。

**规则**：对于 merge commit，使用 `git diff <merge-commit>^1 <merge-commit>` 或 `git diff <merge-commit>^1..<merge-commit>^2` 来查看实际合并内容变更，不用 `diff-tree`。

```bash
# 查看 merge commit 引入的变更
git diff <merge-commit>^1 <merge-commit>
```

### 5. 审查流程 SOP

**问题**：每次审查步骤不统一，重复探索。

**规则**：按标准化流程执行每次审查：

0. Gerrit API 查询 Change-Id → 获取 change number → 构造 ref 路径
1. `git ls-remote` 确认目标 ref 存在
2. `git fetch` 拉取缺失 ref
3. `git diff` 获取变更内容
4. 按 `AGENTS.md` 审查标准逐项检查
5. 输出三段式 Review 结论（阶段性成果 / 必须现在处理 / 记录后补）

### 6. Gerrit Change 审查先走最短正确路径

**问题**：收到明确的 Gerrit Change 审查请求时，提前做 memory 检索、本地分支审计、worktree 跑测试或并行大范围读取，会偏离审查主线；在 Windows sandbox 下还容易触发 helper 启动失败，浪费审查时间。

**规则**：用户给出 Change-Id / Gerrit change 后，先严格完成第 5 条 SOP，拿到真实 `FETCH_HEAD` diff 后再判断是否需要补充动作。

不要在 SOP 完成前做这些事：

- 不要先审本地当前分支、未跟踪文件或当前 HEAD；本地状态最多只用于提醒“审查以 Gerrit ref 为准”。
- 不要先做 memory / 历史记录扩展检索，除非 diff 中出现已知争议点且需要校准阈值。
- 不要先创建 worktree 或跑测试；只有静态审查出现编译/行为争议，或用户明确要求验证时再跑。
- 不要用并行命令大范围读文件；本仓库在 Windows sandbox 下优先使用简单单命令读取，尤其是 `git show`、`git diff`、`Get-Content -Encoding UTF8`、`rg`。
- 不要把“补充证据”放在“定位 Gerrit ref 和读取 diff”之前。

正确顺序固定为：

```text
读本文件 -> 读 V2.1 相关冻结段 -> Gerrit API 查 change number -> ls-remote -> fetch -> diff -> 按需补读依赖文件 -> 输出 Review
```

### 7. Windows sandbox 失败规避

**问题**：本仓库在 Windows sandbox 下，某些并行读文件、PowerShell 管道、复杂 git plumbing 命令会反复触发 `windows sandbox: spawn setup refresh` 或 `orchestrator_helper_launch_canceled`。这类失败不提供审查价值，只会消耗时间和额度。

**规则**：在本仓库执行 Gerrit 审查、同步、只读定位时，优先使用已经稳定验证过的简单单命令。不要为了节省几秒使用并行工具或复杂管道。

优先使用这些稳定命令形态：

```bash
git status --short --branch
git branch -vv
git log --oneline --decorate -5
git fetch origin
git fetch origin refs/changes/<nn>/<change_number>/<patchset>
git diff --name-status FETCH_HEAD^1..FETCH_HEAD
git diff --unified=80 FETCH_HEAD^1..FETCH_HEAD -- <path>
git show FETCH_HEAD:<path>
Get-Content -LiteralPath <path> -Encoding UTF8
rg -n "<pattern>" <path>
```

同步远端时，在确认没有已跟踪文件改动后，优先使用：

```bash
git fetch origin
git rebase origin/master
git branch -f master origin/master
```

不要反复尝试这些不稳定形态：

- 不要用 `multi_tool_use.parallel` 并行跑多个 shell 读取命令。
- 不要把 `git show`、`Get-Content`、`git ls-tree`、`git merge-base --is-ancestor` 再接 PowerShell 管道做行号或过滤。
- 不要为了拿行号反复尝试复杂命令；优先用 `rg -n`，或直接读完整文件后只给总 Reply。
- `git grep` 如在普通 sandbox 下失败，可用已批准的 `git grep` 前缀重试一次；不要在普通 sandbox 中反复撞同一个失败。
- 如果简单单命令也偶发失败，换同等简单命令继续；不要升级成更复杂的组合命令。

### 8. 反模式速查表

**问题**：同类错误反复出现，缺乏事前自查机制。

**规则**：每次审查开始前，快速自查以下反模式：

| 反模式 | 正确做法 |
|--------|----------|
| 直接用 ref 猜测路径 fetch | 先 `ls-remote` 确认 |
| 把 Change-Id 当作路径片段 | Change-Id 仅用于标识；先 Gerrit API 查 change number |
| 用 `git ls-remote \| grep Change-Id` 找 ref | Change-Id 不在 ref 路径里，永远找不到 |
| 对 merge commit 用 `diff-tree` | 用 `git diff` 查看合并变更 |
| 逐个盲目尝试所有可能 ref | 对比远程/本地后精准 fetch |
| 跳过标准流程自行探索 | 按 SOP 固定步骤执行 |
| 先查本地分支 / status 再审 Gerrit Change | 先定位并 fetch Gerrit ref；本地状态不代表被审内容 |
| 先创建 worktree 或跑测试 | 先完成静态 diff 审查；测试只作为必要补充证据 |
| 并行大范围读取导致 sandbox helper 失败 | 优先简单单命令、UTF-8 读取和 `git show FETCH_HEAD:<路径>` |
| 普通 sandbox 反复尝试同一失败命令 | 换稳定单命令，或对已批准前缀只重试一次 |

---

**每次审查前读一遍本文件，跳过无效查找步骤。**
