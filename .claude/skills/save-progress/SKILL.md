---
name: save-progress
description: Commit learning progress and quiz files, then push to main
tools: Read, Bash
---

# Save Progress Skill

This skill commits your learning progress (`progress.json` and quiz files) to git and pushes changes to the main branch on GitHub.

## Usage

```
/save-progress
```

No arguments needed. The skill will automatically:
- Commit `progress.json` (progress tracker)
- Commit all quiz files in `.claude/quizzes/`
- Generate a detailed commit message showing the most recent topic and score
- Push changes to `origin/main`

## Instructions

When the user invokes `/save-progress`:

### 1. Validate Environment

**Check if progress.json exists**:
- Use Read tool to check for `progress.json` in project root
- If not found, display error and exit:
  ```
  ✗ Error: progress.json not found
    Suggestion: Complete a quiz first using /generate-questions
  ```

### 2. Check for Changes

Use Bash tool to check git status for both progress.json and quiz files:

```bash
git status --porcelain progress.json .claude/quizzes/
```

**Parse output**:
- ` M progress.json` = modified progress file
- `?? progress.json` = untracked progress file
- `?? .claude/quizzes/quiz-*.md` = new quiz files
- Empty output = no changes

**If no changes to either progress.json or .claude/quizzes/**:
```
No changes to progress or quizzes. Nothing to save.
```
Exit without committing.

### 3. Read Progress Data for Commit Message

**Objective**: Generate detailed commit message showing most recent topic and score.

**Algorithm**:
1. Read `progress.json` using Read tool
2. Parse JSON structure: `modules[moduleNum].subsections[subsectionNum]`
3. Create array to collect recent updates: `recentUpdates = []`
4. Iterate through all modules and subsections:
   - For each subsection where `lastUpdated` is not null:
     - Extract: `topic`, `currentScore`, `lastUpdated`, `name`
     - Add to `recentUpdates` array
5. Sort `recentUpdates` by `lastUpdated` timestamp (descending, most recent first)
6. If `recentUpdates` is not empty:
   - Take first item (most recent)
   - Generate message: `"update progress: {topic} ({score}/100)"`
   - Example: `"update progress: 1.1 (85/100)"`
7. If `recentUpdates` is empty or parsing fails:
   - Use fallback: `"update learning progress"`

**Expected progress.json structure**:
```json
{
  "metadata": { ... },
  "modules": {
    "01": {
      "subsections": {
        "1.1": {
          "name": "ASCSP Mission",
          "currentScore": 85,
          "lastUpdated": "2026-02-01T14:30:22Z",
          "quizHistory": [...]
        }
      }
    }
  }
}
```

### 4. Stage Progress Files

Use Bash tool to stage files explicitly:

```bash
git add progress.json .claude/quizzes/
```

This stages:
- `progress.json` (progress tracker)
- All files in `.claude/quizzes/` directory (all quiz history)

**IMPORTANT**: Never use `git add .` or `git add -A` - only stage progress-related files.

### 5. Create Commit

Use the commit message generated in step 3.

**Bash command with heredoc for proper escaping**:
```bash
git commit -m "$(cat <<'EOF'
update progress: 1.1 (85/100)
EOF
)"
```

**If commit fails**:
- Check if there are actually staged changes
- Display git error message to user
- Exit without pushing

### 6. Push to Main

**Check current branch first**:
```bash
git branch --show-current
```

**If not on main**:
```
✗ Error: You're on branch [branch-name], not main
  Suggestion: Switch to main first with: git checkout main
```
Exit without pushing.

**If on main, push to remote**:
```bash
git push origin main
```

**Handle push errors**:
- Network errors: Display git error message
- No remote configured: Display "No remote repository configured"
- Authentication errors: Display git error message
- Merge conflicts: Display "Push rejected. Pull latest changes first: git pull origin main"

### 7. Report Success

**Count quiz files in the commit**:
Use Bash to count files:
```bash
git diff-tree --no-commit-id --name-only -r HEAD | grep -c ".claude/quizzes/"
```

**Display success message**:
```
✓ Progress saved and pushed to main
  Committed: progress.json + [N] quiz file(s)
  Message: update progress: 1.1 (85/100)
  Branch: main → origin/main
```

Where:
- `[N]` = count of quiz files in commit (e.g., 0, 1, 3)
- Message = the actual commit message used
- Format is clean and informative

## Error Handling

### Scenarios and Responses

**1. progress.json doesn't exist**:
```
✗ Error: progress.json not found
  Suggestion: Complete a quiz first using /generate-questions
```

**2. No changes to commit**:
```
No changes to progress or quizzes. Nothing to save.
```

**3. Not a git repository**:
```
✗ Error: This directory is not a git repository
  Suggestion: Initialize git with: git init
```

**4. Not on main branch**:
```
✗ Error: You're on branch [branch-name], not main
  Suggestion: Switch to main first with: git checkout main
```

**5. Uncommitted changes to other files**:
Continue normally - we only commit progress.json and quizzes.

**6. Push fails (network/auth)**:
```
✗ Error: Push failed
  Git says: [git error message]
  Suggestion: Check network connection and authentication
```

**7. Push rejected (needs pull)**:
```
✗ Error: Push rejected
  Suggestion: Pull latest changes first: git pull origin main
```

**8. Can't parse progress.json**:
Use fallback commit message "update learning progress" and continue.

## Implementation Notes

**Git Commands**:
- Always use explicit paths: `progress.json .claude/quizzes/`
- Never use wildcards in `git add` that could match unintended files
- Use `--porcelain` for parseable git status output
- Use heredoc for commit messages to handle special characters

**Path Considerations**:
- All paths are relative from project root
- `progress.json` is in project root
- Quiz files are in `.claude/quizzes/` subdirectory
- Cross-platform compatible (CLI, web, desktop)

**Error Handling Strategy**:
- Fail fast with clear messages
- Always suggest next action to user
- Never leave repository in inconsistent state
- Display git error messages when helpful

**Commit Message Format**:
- Lowercase, brief, consistent with repository style
- Format: `"update progress: X.X (N/100)"`
- Example: `"update progress: 1.1 (85/100)"`
- Fallback: `"update learning progress"`

## Success Criteria

✓ Validates progress.json exists before attempting commit
✓ Checks for changes to avoid empty commits
✓ Stages only progress.json and quiz files (never other files)
✓ Generates detailed commit message with topic and score
✓ Creates commit with proper message formatting
✓ Verifies on main branch before pushing
✓ Pushes to origin/main successfully
✓ Displays clear success message with file count
✓ Handles all error scenarios gracefully
✓ Provides actionable suggestions for errors
✓ Works cross-platform (CLI, web, desktop)
