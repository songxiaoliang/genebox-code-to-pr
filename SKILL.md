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
  
  **Required credentials for Bitbucket PR creation:**
  - `BITBUCKET_USERNAME`: Bitbucket username
  - `BITBUCKET_APP_PASSWORD`: Bitbucket app password
  
  **First-time setup:** If credentials are not set, Claude will prompt the user to configure them.
  
  **GeneBox defaults:**
  - Workspace: APP
  - Repository: Auto-detected from git remote
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

### Step 0: Check credentials (first-time setup)

Before starting, check if Bitbucket credentials are configured:
- `BITBUCKET_USERNAME` environment variable
- `BITBUCKET_APP_PASSWORD` environment variable

**If not configured:**

1. **Prompt user for credentials:**
   ```
   It looks like this is your first time using the GeneBox PR feature.
   
   To create pull requests, I need your Bitbucket credentials:
   
   1. Bitbucket Username: [input field]
   2. App Password: [input field - masked]
   
   How to create an App Password:
   - Go to Bitbucket → Personal settings → App passwords → Create app password
   - Grant permissions: Pull requests: Write, Projects: Read
   
   Would you like to save these to your shell profile (~/.zshrc)? [Y/n]
   ```

2. **After user provides credentials:**
   ```bash
   export BITBUCKET_USERNAME="<provided-username>"
   export BITBUCKET_APP_PASSWORD="<provided-password>"
   ```
   
   If user chooses to save to profile:
   ```bash
   echo 'export BITBUCKET_USERNAME="<provided-username>"' >> ~/.zshrc
   echo 'export BITBUCKET_APP_PASSWORD="<provided-password>"' >> ~/.zshrc
   source ~/.zshrc
   ```

3. **Continue with the workflow** now that credentials are available.

### Step 1: Check repository status

Get current git status and branch information:
- Current branch name (e.g., `dev/song`)
- Changed files
- Whether there are staged/unstaged changes

### Step 2: Generate commit message

Analyze the changes and generate a commit message in Conventional Commits format:

**Format:** `<type>(<scope>): <description>`

**Types:**
- `feat`: New feature
- `fix`: Bug fix
- `docs`: Documentation changes
- `style`: Code style changes (formatting, semicolons, etc.)
- `refactor`: Code refactoring
- `test`: Adding or updating tests
- `chore`: Build process, dependencies, etc.

**Example:**
- `feat(report): add aggregation card component`
- `fix(auth): resolve login token expiration issue`

**Process:**
1. Look at changed files to determine scope (e.g., `Report/`, `Auth/`, `Common/`)
2. Determine type based on file changes patterns
3. Generate concise description
4. Present to user for confirmation or edit

### Step 3: Git operations

```
git add .
git commit -m "<message>"
git push origin <current-branch>
```

### Step 4: Create pull request

**Target branch pattern:**
GeneBox uses version-based feature branches like `7.5.0-feature`, `7.6.0-feature`.

**Ask user to select target branch** from available remote branches matching the pattern `*.*.*-feature`.

**PR Title:** Same as commit message (without type prefix if desired)

**PR Description template:**
```markdown
## Summary
<commit message description>

## Changes
- List of changed files with brief descriptions

## Related
- JIRA/Linear issue: <if mentioned in commit or detected from branch>
```

**Bitbucket configuration:**
- Provider: `bitbucket`
- Organization: `APP`
- Repository: Detect from git remote (e.g., `rnapp`)
- Source branch: Current branch
- Target branch: User-selected version feature branch

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

**Authentication issues (first-time setup):**
- If `BITBUCKET_USERNAME` or `BITBUCKET_APP_PASSWORD` is not set, go to **Step 0** and prompt for credentials
- If authentication fails with provided credentials, show the error and offer to reconfigure
- Save instructions for creating App Password in Bitbucket

## Example Interaction

**User:** /genebox-code-pr

**Claude:** 
> I'll help you commit and create a PR. Let me check the current status...
> 
> Current branch: `dev/song`
> Changed files:
> - App/Pages/Report/Widgets/ReportAggregation/ReportAncestryCard.js
> - App/Pages/Report/Widgets/ReportAggregation/ReportDetailCard.js
> 
> Generated commit message: `feat(report): add report aggregation cards`
> 
> Please confirm or edit: [input field]

**User:** (confirms or edits)

**Claude:**
> Which target branch?
> - 7.5.0-feature
> - 7.6.0-feature
> - Other: [input]

**User:** Selects 7.5.0-feature

**Claude:**
> Committing and pushing...
> Creating PR to merge `dev/song` → `7.5.0-feature`...
> 
> ✓ PR created: http://git.dev.genebox.cn/projects/APP/repos/rnapp/pull-requests/XXXX/overview
