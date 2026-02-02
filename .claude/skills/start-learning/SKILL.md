---
name: start-learning
description: Start learning session by pulling latest progress and launching quiz for next topic under 85
tools: Read, Bash
---

You are the start-learning skill. Your job is to automate the learning workflow by:
1. Syncing latest progress from GitHub
2. Finding the next topic to study (first one under 85)
3. Automatically launching the quiz for that topic

# Step 1: Pull Latest Progress from Remote

First, sync progress from GitHub to ensure we have the latest scores:

```bash
git pull origin main
```

Handle the output:
- If successful (already up-to-date or changes pulled): Continue to next step
  - If changes were pulled, inform user: "Synced latest progress from remote"
- If merge conflicts detected: Display error and exit
  - Error: "✗ Error: Git pull failed - merge conflicts detected"
  - Suggestion: "Resolve conflicts manually with: git status, then run /start-learning again"
- If network error: Warn but continue
  - Warning: "⚠ Warning: Could not sync from remote (network issue). Using local progress.json"
- If not on main branch: Warn but continue
  - Warning: "⚠ Warning: Not on main branch (currently on [branch-name]). Progress sync may be incomplete."

# Step 2: Validate Environment

Check if progress.json exists in the project root:

```
Read: progress.json
```

If progress.json doesn't exist:
- Display: "No progress.json found. Starting with topic 1.1"
- Set selectedTopic = "1.1"
- Skip to Step 5 (invoke generate-questions)

If progress.json exists but is invalid JSON:
- Display: "✗ Error: Could not parse progress.json (invalid JSON)"
- Suggestion: "Check file format or regenerate it"
- Exit without launching quiz

If progress.json is empty (no modules):
- Default to topic 1.1 and launch quiz

# Step 3: Parse Progress and Find Next Topic

Parse the progress.json file and find the next topic to study.

Algorithm:
1. Load and parse progress.json
2. Create ordered array of all topics by iterating modules in order: "01", "02", "03", "04", "05", "06", "07"
3. For each module, iterate subsections in numerical order (1.1, 1.2, ..., 2.1, 2.2, ...)
4. Build array: [{topic, score, name}, ...]
5. Filter topics where score < 85 → incompleteTopics
6. If incompleteTopics is not empty:
   - Select first item (earliest in sequential order)
   - selectedTopic = incompleteTopics[0]
   - retryMode = false
7. Else (all topics >= 85):
   - Find topic with minimum score from allTopics
   - selectedTopic = topicWithLowestScore
   - retryMode = true

Expected progress.json structure:
```json
{
  "modules": {
    "01": {
      "subsections": {
        "1.1": {"name": "...", "currentScore": 0, ...},
        "1.2": {"name": "...", "currentScore": 90, ...}
      }
    },
    "02": { "subsections": {...} }
  }
}
```

Important:
- Preserve sequential order when iterating (not random)
- Handle missing fields gracefully (default score = 0)
- Topic format: "X.Y" (e.g., "1.1", "2.3", "5.10")

# Step 4: Display Learning Status

Before launching the quiz, inform the user what's happening.

**If in normal mode (found topic < 85)**:
```
📚 Starting learning session...
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Next topic to study: [topic] - [topic name]
Current score: [score]/100

Launching quiz...
```

**If in retry mode (all topics >= 85)**:
```
🎉 All topics completed with score >= 85!
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Retrying topic with lowest score for mastery:
Topic: [topic] - [topic name]
Current score: [score]/100

Launching quiz...
```

# Step 5: Invoke Generate Questions Skill

Launch the quiz for the selected topic using the Skill tool.

**Method**: Use the Skill tool (NOT Bash) to invoke generate-questions

**Parameters**:
- skill: "generate-questions"
- args: The topic number as a string (e.g., "1.1", "2.3", "5.10")

**Example**:
```
If selectedTopic = "1.1":
  Use Skill tool with:
    skill = "generate-questions"
    args = "1.1"

If selectedTopic = "5.10":
  Use Skill tool with:
    skill = "generate-questions"
    args = "5.10"
```

**Why use Skill tool instead of Bash?**
- Maintains interactive context (can prompt questions and receive answers)
- Keeps the quiz session within the same Claude conversation
- Allows user to see and respond to quiz questions

**Error handling**:
- If skill invocation fails, display the error
- Show: "✗ Error: Could not invoke generate-questions skill"
- Suggestion: "Ensure generate-questions skill exists"

# Step 6: Post-Invocation Reminder

After the generate-questions skill completes, display:

```
✓ Quiz complete! Don't forget to save your progress:
  Run: /save-progress
```

# Error Scenarios Summary

1. **progress.json doesn't exist**: Default to 1.1, launch quiz
2. **progress.json invalid JSON**: Show error, exit
3. **Git pull merge conflicts**: Show error, exit
4. **Git pull network error**: Warn, continue with local
5. **Not on main branch**: Warn, continue
6. **Skill invocation fails**: Show error with suggestion
7. **Empty progress.json**: Default to 1.1, launch quiz

# Implementation Notes

- All paths relative from project root
- Use `git branch --show-current` to check current branch
- Parse git pull output to detect conflicts vs success
- Topic argument is passed directly to generate-questions skill
- Cross-platform compatible (CLI, web, desktop)
