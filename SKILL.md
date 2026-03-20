---
name: genebox-code-pr
description: |
  Automatically commit code and create Bitbucket pull requests for GeneBox projects.

  **When to use this skill:**
  - User types /genebox-code-pr command
  - User asks to "提交代码并创建PR" or "code commit and create PR" for GeneBox
  - User mentions creating pull request after making code changes
  - User needs to push local branch and create PR on Bitbucket

  This skill handles the complete workflow: git add → generate commit message → git commit → git push → create Bitbucket PR.
compatibility: |
  - Git repository (Bitbucket Server)
  - mcp0_gitkraken MCP server for git operations
  - mcp0_pull_request_create MCP tool for PR creation
---

**方案：环境变量配置（推荐）**

Skill 依赖两个环境变量：
- `BITBUCKET_USERNAME` - Bitbucket 用户名
- `BITBUCKET_APP_PASSWORD` - Bitbucket App Password

**首次使用配置步骤：**

1. **创建 App Password**（只需一次）：
   - 登录 Bitbucket → 右上角头像 → Personal settings → App passwords → Create app password
   - 权限勾选：Pull requests: Write, Projects: Read
   - 保存生成的密码

2. **配置环境变量**，选择以下任一方式：

   **方式 A：临时配置（当前终端有效）**
   ```bash
   export BITBUCKET_USERNAME="your_username"
   export BITBUCKET_APP_PASSWORD="your_app_password"
   ```

   **方式 B：永久配置（推荐，写入 ~/.zshrc）**
   ```bash
   echo 'export BITBUCKET_USERNAME="your_username"' >> ~/.zshrc
   echo 'export BITBUCKET_APP_PASSWORD="your_app_password"' >> ~/.zshrc
   source ~/.zshrc
   ```

3. **验证配置**：
   ```bash
   echo $BITBUCKET_USERNAME  # 应显示你的用户名
   ```

**Skill 工作流程：**

当执行 `/genebox-code-pr` 时，Skill 会自动：
1. 检测环境变量是否已配置
2. **如果未配置**：显示上述配置步骤，并提示用户完成后重试
3. **如果已配置**：继续执行 commit → push → 创建 PR
---

# GeneBox Code PR Skill

## Overview

This skill automates the complete code submission workflow for GeneBox projects:
1. Stage changes with `git add`
2. Generate Conventional Commits format message based on changes
3. Commit changes
4. Push to remote
5. Create Bitbucket pull request

## Workflow

### Step 0: Verify credentials

**Before starting, verify Bitbucket credentials are configured:**

```bash
# 检查环境变量
if [ -z "$BITBUCKET_USERNAME" ] || [ -z "$BITBUCKET_APP_PASSWORD" ]; then
  echo "❌ Bitbucket credentials not configured"
  echo ""
  echo "Please configure first:"
  echo "1. Create App Password at: Bitbucket → Personal settings → App passwords"
  echo "2. Run: export BITBUCKET_USERNAME=\"your_username\""
  echo "3. Run: export BITBUCKET_APP_PASSWORD=\"your_app_password\""
  echo "4. (Optional) Save to ~/.zshrc for persistence"
  exit 1
fi
```

**If credentials missing:**
- Display configuration instructions (shown above)
- Stop workflow and wait for user to configure

**If credentials present:** Continue to Step 1

### Step 1: Check repository status

Get current git status and branch information:
- Current branch name (e.g., `dev/song`)
- Changed files
- Whether there are staged/unstaged changes

### Step 2: 生成提交信息

分析变更并生成符合 Conventional Commits 格式的中文提交信息：

**格式：** `<类型>(<作用域>): <描述>`

**类型：**
- `feat`: 新功能
- `fix`: Bug 修复
- `docs`: 文档变更
- `style`: 代码样式变更（格式化、分号等）
- `refactor`: 代码重构
- `test`: 添加或更新测试
- `chore`: 构建过程、依赖等

**示例：**
- `feat(report): 添加聚合卡片组件`
- `fix(auth): 解决登录令牌过期问题`

**流程：**
1. 查看变更的文件来确定作用域（例如 `Report/`、`Auth/`、`Common/`）
2. 根据文件变更模式确定类型
3. 生成简明的描述（中文）
4. 展示给用户确认或编辑

### Step 3: Git operations

```
git add .
git commit -m "<message>"
git push origin <current-branch>
```

### Step 4: Create pull request (using curl)

**Target branch pattern:**
GeneBox uses version-based feature branches like `7.5.0-feature`, `7.6.0-feature`.

**List available feature branches:**
```bash
# Get remote branches matching *.*.*-feature pattern
git branch -r --list '*/*.*.*-feature' | sed 's/origin\///' | sort -V
```

**Ask user to select target branch** from the dynamically listed remote branches.

**Example branches output:**
- 7.5.0-feature
- 7.6.0-feature
- 7.7.0-feature

If no matching branches found, allow manual input.

**PR Title:** Same as commit message

**PR Description template:**
```markdown
## Summary
<commit message description>

## Changes
- List of changed files with brief descriptions

## Related
- JIRA/Linear issue: <if mentioned in commit or detected from branch>
```

**Create PR using curl:**
```bash
# Detect repository from git remote
REPO_URL=$(git remote get-url origin)
REPO_NAME=$(basename -s .git $REPO_URL)

# Parse reviewers (comma-separated usernames)
REVIEWERS_JSON=""
if [ -n "$PR_REVIEWERS" ]; then
  # Convert comma-separated to JSON array: [{"user":{"name":"user1"}},{"user":{"name":"user2"}}]
  REVIEWERS_JSON=$(echo "$PR_REVIEWERS" | sed 's/[^,]*/{\"user\":{\"name\":\"&\"}}/g' | sed 's/,/,/g')
  REVIEWERS_JSON=",\"reviewers\":[$REVIEWERS_JSON]"
fi

# Create PR using Bitbucket REST API
curl -X POST \
  -u "$BITBUCKET_USERNAME:$BITBUCKET_APP_PASSWORD" \
  -H "Content-Type: application/json" \
  -d "{
    \"title\": \"$COMMIT_MESSAGE\",
    \"description\": \"$PR_DESCRIPTION\",
    \"fromRef\": {\"id\": \"refs/heads/$CURRENT_BRANCH\"},
    \"toRef\": {\"id\": \"refs/heads/$TARGET_BRANCH\"}$REVIEWERS_JSON
  }" \
  "http://git.dev.genebox.cn/rest/api/1.0/projects/APP/repos/$REPO_NAME/pull-requests"
```

**Environment variables used:**
- `$BITBUCKET_USERNAME` - from ~/.zshrc export
- `$BITBUCKET_APP_PASSWORD` - from ~/.zshrc export
- `$COMMIT_MESSAGE` - generated in Step 2
- `$CURRENT_BRANCH` - detected in Step 1
- `$TARGET_BRANCH` - user selected
- `$PR_DESCRIPTION` - generated from template
- `$PR_REVIEWERS` - optional, comma-separated usernames (e.g., "user1,user2,user3")

**Reviewer configuration options:**

**Option A: Default reviewers (set in environment)**
```bash
# Add to ~/.zshrc for default reviewers
export PR_REVIEWERS="reviewer1,reviewer2"
```

**Option B: Interactive reviewer selection (recommended)**
```
Claude: "Select reviewers for this PR:"
Options:
A. 杨必万 (yangbiwan)
B. 马乐 (male)
C. Both reviewers
D. Skip reviewers

User selects option:
- If A: PR_REVIEWERS="yangbiwan"
- If B: PR_REVIEWERS="male"
- If C: PR_REVIEWERS="yangbiwan,male"
- If D: No reviewers
```

**Option C: Branch-based reviewers**
```bash
# Auto-select reviewers based on branch pattern
if [[ $CURRENT_BRANCH == *"report"* ]]; then
  export PR_REVIEWERS="yangxiangbo"
elif [[ $CURRENT_BRANCH == *"auth"* ]]; then
  export PR_REVIEWERS="yangbiwan"
fi
```

**Success response:**
```json
{
  "id": 5456,
  "title": "feat(report): add report aggregation feature",
  "state": "OPEN",
  "links": {
    "self": [
      {
        "href": "http://git.dev.genebox.cn/projects/APP/repos/rnapp/pull-requests/5456"
      }
    ]
  }
}
```

**Display result to user:**
```
✅ PR created successfully!
📝 Title: feat(report): add report aggregation feature
🔗 Link: http://git.dev.genebox.cn/projects/APP/repos/rnapp/pull-requests/5456
```

## Edge Cases

**No changes to commit:**
- Inform user and exit gracefully

**Already committed but not pushed:**
- Skip commit step, just push

**Push fails (conflict):**
- Suggest user pull and resolve conflicts first

**PR creation fails:**
- Check if PR already exists for this branch
- Report specific error to user

**Authentication issues:**
- If `BITBUCKET_USERNAME` or `BITBUCKET_APP_PASSWORD` is not set, go to **Step 0** and display configuration instructions
- If curl returns 401/403 error: "Authentication failed. Please check your credentials and try again."
- If PR already exists: curl returns 409 with "A pull request already exists for this source and target"
- If repository not found: curl returns 404, verify repository name and permissions

## Example Interaction

**User:** /genebox-code-pr

**Claude:** 
> 我来帮你提交代码并创建 PR。先查看当前状态...
> 
> 当前分支：`dev/song`
> 变更文件：
> - App/Pages/Report/Widgets/ReportAggregation/ReportAncestryCard.js
> - App/Pages/Report/Widgets/ReportAggregation/ReportDetailCard.js
> 
> 生成的提交信息：`feat(report): 添加报告聚合卡片`
> 
> 请确认或编辑：[输入框]

**User:** (confirms or edits)

**Claude:**
> 选择目标分支：
> - 7.5.0-feature
> - 7.6.0-feature
> - 7.7.0-feature
> - 其他：[输入分支名]

**User:** Selects 7.5.0-feature

**Claude:**
> 选择此 PR 的审核人：
> A. 杨必万 (yangbiwan)
> B. 马乐 (male)
> C. 两位审核人
> D. 跳过审核人

**User:** Selects C

**Claude:**
> 正在提交并推送...
> 正在创建 PR，将 `dev/song` → `7.5.0-feature` 合并并添加审核人...
> 
> ✓ PR 已创建：http://git.dev.genebox.cn/projects/APP/repos/rnapp/pull-requests/XXXX/overview
> 📋 审核人：yangbiwan, male
