---
description: 智能 Git 提交，自动生成 Conventional Commits 中文提交信息并处理 Changeset。仅在用户显式要求提交时触发。
---
# 智能 Git 提交

在保证提交安全边界的前提下，优先降低主会话上下文占用：大改动走子代理分析，小改动本地直出。

> **触发条件**：仅在用户显式要求提交代码时执行。

## 完整流程

### Step 1: 检查工作区状态

```bash
git rev-parse --abbrev-ref HEAD
git status --porcelain
```

如果没有任何变更，直接告知用户无需提交并结束。

### Step 2: 阈值判定（决定是否委派）

先用轻量命令评估变更规模：

```bash
git diff --shortstat
git diff --cached --shortstat
git status --porcelain | wc -l
```

满足任一条件即委派到 `git-commit-analyzer` 子代理（`model: fast`）：

- 变更文件数 > 10
- `insertions + deletions` > 300 行
- 出现多模块混合改动（如 `src/` + `packages/` + `scripts/`）

未达到阈值则由当前 Skill 直接分析。

> 子代理失败时，必须自动降级回当前 Skill 本地分析，不得中断提交流程。

### Step 3: 筛选并暂存当前会话相关文件

只提交当前会话实际修改/创建的文件，忽略无关变更。

```bash
git add <本次会话相关文件>
```

若无法可靠识别会话文件，可暂存全部变更文件。

### Step 4: 生成结构化分析结果

无论是本地分析还是子代理分析，必须产出以下结构化字段：

```yaml
type: feat|fix|refactor|style|docs|test|chore|perf
scope: <必填，英文模块名>
subject: <中文主题，<=50字>
body: <中文正文，说明为什么>
changesetLevel: none|patch|minor|major
changesetReason: <中文说明>
riskNotes: <可选，中文风险提示>
```

#### 类型与 changeset 映射

- `feat` -> `minor`
- `fix|perf|style|refactor|chore|docs|test` -> `patch`
- 含 `BREAKING CHANGE` 或 `!:` -> `major`
- 若暂存区无业务代码（非 `src/`、`packages/` 业务源码）-> `none`

### Step 5: 组装提交信息

格式固定：

```text
<type>(<scope>): <subject>

<body>
```

规则：

- `type` 英文
- `scope` 必填英文，不可省略
- `subject/body` 必须中文

### Step 6: 创建 Changeset（在 git add 之后、git commit 之前）

**判定 `changesetLevel`**：回顾 Step 4 的分析结果。

- 若 `changesetLevel == none`（暂存区无 `src/`、`packages/` 业务源码），跳过本步骤。
- 若 `changesetLevel != none`，**必须**执行以下三条命令（不可省略、不可合并到其他步骤）：

```bash
# 6-1. 生成随机文件名
CHANGESET_NAME=$(node -e "console.log(require('crypto').randomBytes(8).toString('hex'))")

# 6-2. 写入 changeset 文件
cat > ".changeset/${CHANGESET_NAME}.md" <<'CHANGESET_EOF'
---
'项目名称': <patch|minor|major>
---

<中文变更摘要，与 Step 4 的 changesetReason 一致>
CHANGESET_EOF

# 6-3. 加入暂存区
git add ".changeset/${CHANGESET_NAME}.md"
```

**为什么必须在此步骤完成**：Step 8 的 `git commit` 会触发 pre-commit hook（`auto-changeset.sh`）。
该脚本在非交互式环境（Agent / CI）中无法弹出交互提示，会自动跳过。
如果 Step 6 没有提前把 changeset 放入暂存区，changeset 就永远不会被创建。
当暂存区已包含 `.changeset/*.md` 文件时，hook 会直接 `exit 0`，不触发任何提示。

### Step 7: 预览与确认

输出以下预览并等待用户 `ok / edit / cancel`：

```text
提交类型: <type>
影响范围: <scope>
Changeset: <none|patch|minor|major>
提交信息:
────────────────────────────
<完整提交信息>
────────────────────────────
涉及文件 (N 个):
  - file1
  - file2
```

如果 `changesetLevel != none`，额外显示：

```text
Changeset 文件: .changeset/<name>.md（已暂存）
```

### Step 8: 执行提交

```bash
git commit -m "$(cat <<'EOF'
<完整提交信息>
EOF
)"
git log -1 --oneline
```

> **不要使用 `--no-verify`**。pre-commit hook 会执行 `format:staged`、`type-check:staged` 和 `auto-changeset.sh`，
> 这些检查应当正常运行。只要 Step 6 已将 changeset 加入暂存区，`auto-changeset.sh` 会检测到并直接通过。
>
> 如果 hook 因非交互式环境失败（极少见），在报错日志中确认失败原因后，可降级为 `git commit --no-verify`，
> 但必须在响应中向用户说明跳过了哪些检查。

## 注意事项

- 不提交敏感信息（`.env`、密钥、凭证）
- 一次提交尽量只做一件事
- 不主动询问是否推送远端
- 不主动汇报无关未提交文件
