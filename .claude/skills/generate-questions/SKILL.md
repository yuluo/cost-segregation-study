---
name: generate-questions
description: Generate and conduct interactive quiz on cost segregation study materials. Usage: /generate-questions <topic> (e.g., /generate-questions 1.1)
tools: Read, Grep, Glob, WebSearch, Write, Bash
---

# Generate Interactive Quiz Skill

This skill creates and conducts an interactive 10-question multiple-choice quiz based on cost segregation study materials.

## Available Topics

```
Module 1: Professional Standards (1.1 - 1.4)
Module 2: Fundamentals (2.1 - 2.6)
Module 3: Tax Law (3.1 - 3.8)
Module 4: Tangible Property (4.1 - 4.7)
Module 5: Methodologies (5.1 - 5.10)
Module 6: Legal Framework (6.1 - 6.7)
Module 7: Quality Reports (7.1 - 7.7)
```

## Instructions

When the user invokes `/generate-questions <topic>` (e.g., `/generate-questions 1.1`):

### 1. Parse Arguments and Validate

Extract the topic number from `$ARGUMENTS`. The format should be `X.X` where:
- First digit = module number (1-7)
- Full number = sub-topic (e.g., 1.1, 2.3, 5.10)

**Error Handling**:
- If no arguments: "Usage: /generate-questions <topic> (e.g., /generate-questions 1.1)"
- If invalid format: "Please provide topic in format X.X (e.g., 1.1, 2.3, 7.5)"
- If topic not found: "Topic X.X not found. Please check available topics above."

### 2. Locate Study Material

Use the Glob tool to find the study material README.md file:

**Pattern**: `0{module}-*/{topic}-*/README.md`

Example:
- Topic `1.1` → Pattern: `01-*/1.1-*/README.md`
- Topic `2.3` → Pattern: `02-*/2.3-*/README.md`
- Topic `5.10` → Pattern: `05-*/5.10-*/README.md`

Use the Read tool to load the entire README.md content.

### 3. Enhance with Web Search

Extract the main topic from the folder name and perform a WebSearch:
- Query format: "ASCSP cost segregation [topic descriptor] professional standards"
- Example: For `1.1-ascsp-mission` → "ASCSP cost segregation mission professional standards"

Combine the study material content with web search results to inform question generation.

### 4. Generate 10 Multiple-Choice Questions

Create exactly 10 questions with this difficulty distribution:
- **Questions 1-3**: Concept recall (definitions, key facts, terminology)
- **Questions 4-7**: Application/analysis (scenarios, "what would you do", practical situations)
- **Questions 8-10**: Synthesis (combining concepts, edge cases, critical thinking)

**Question Format**:
- Each question has exactly 4 answer choices: A, B, C, D
- One correct answer
- Three plausible distractors based on common misconceptions
- Each question must include an explanation

### 5. Conduct Interactive Quiz

Present questions one at a time in this format:

```
📚 Quiz on: [Topic Name from folder]
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Question [N] of 10:

[Question text]

A) [Option A]
B) [Option B]
C) [Option C]
D) [Option D]

What is your answer? (Enter A, B, C, or D)
```

**After User Responds**:
- If correct: "✓ Correct!"
- If incorrect: "✗ Incorrect."
- Always show: "The correct answer is [X]."
- Always provide explanation
- **Immediately continue to the next question** (no additional user input required)
- Add a visual separator between questions for readability

**Track**:
- User's answer for each question
- Whether each answer was correct/incorrect
- Running score

### 6. Calculate Final Score

After all 10 questions are completed:
- Calculate score: (correct answers / 10) × 100
- Display results in this format:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Quiz Complete!

Your Score: [score]/100

Performance Breakdown:
[For each question: ✓/✗ Question N: Correct/Incorrect (answer details)]

Correct: X/10
Grade: X/100

Quiz saved to: .claude/quizzes/quiz-[topic]-[timestamp].md
```

### 7. Save Quiz Results

Create a timestamped quiz file at: `.claude/quizzes/quiz-{topic}-{timestamp}.md`

Use timestamp format: `YYYY-MM-DD-HHmmss` (e.g., `2026-02-01-143022`)

**File Format**:

```markdown
# Quiz: [Topic Name] (Module X.X)
**Date**: [Full timestamp]
**Score**: [score]/100

---

## Question 1
[Question text]

A) [Option A]
B) [Option B]
C) [Option C]
D) [Option D]

**Your Answer**: [User's answer]
**Correct Answer**: [Correct answer] [✓ or ✗]
**Explanation**: [Explanation text]

---

[Repeat for all 10 questions]

---

## Summary
- **Total Score**: [score]/100
- **Correct**: X/10
- **Incorrect**: Y/10
- **Percentage**: Z%
- **Topics Covered**: [List main topics from the questions]
```

Use the Write tool to create this file with relative path from project root.

Report the file location to the user after saving.

### 8. Update Progress Tracker

After saving the quiz results, automatically update `progress.json` to track learning progress:

**Steps**:

1. **Read progress.json** from project root
2. **Parse the topic** to extract module and subsection:
   - Topic `1.1` → Module `"01"`, Subsection `"1.1"`
   - Topic `2.3` → Module `"02"`, Subsection `"2.3"`
   - Topic `5.10` → Module `"05"`, Subsection `"5.10"`
3. **Navigate to the correct location**: `modules[moduleNum].subsections[subsectionNum]`
4. **Update the subsection data**:
   - `currentScore`: Set to the quiz score (0-100)
   - `lastUpdated`: Set to current ISO 8601 timestamp (e.g., "2026-02-01T14:30:22Z")
   - `quizHistory`: Add new entry to the array:
     ```json
     {
       "date": "2026-02-01T14:30:22Z",
       "score": 85,
       "quizFile": ".claude/quizzes/quiz-1.1-2026-02-01-143022.md"
     }
     ```
5. **Update metadata**: Set `metadata.lastModified` to current timestamp
6. **Write updated progress.json** back to file

**Important**:
- Use Read tool to load progress.json
- Parse JSON content
- Update the specific subsection data
- Preserve all other data unchanged
- Use Write tool to save updated progress.json with proper formatting (2-space indentation)

**Error Handling**:
- If progress.json doesn't exist, inform user and continue
- If JSON parsing fails, log error and continue
- If subsection not found in progress.json, log warning and continue

**User Communication**:
After updating progress.json, display:
```
✓ Progress updated: [Topic] score saved as [score]/100
```

## Implementation Notes

**Path Strategy**:
- Use ONLY relative paths from project root
- Never use absolute paths
- This ensures cross-platform compatibility (CLI, web, desktop)

**Question Quality**:
- Base questions on study material content
- Enhance with current information from web search
- Create realistic distractors that test understanding
- Provide educational explanations that reinforce learning
- Mix terminology, concepts, and practical application

**Interactive Flow**:
- Wait for user input after each question
- Provide immediate feedback to enhance learning
- Track progress throughout the quiz
- Present final results comprehensively

**Error Recovery**:
- Validate topic number before searching
- Check if study material exists before generating questions
- Provide helpful error messages with usage examples

## Success Criteria

✓ Correctly locates study material for any valid topic (1.1 - 7.7)
✓ Generates exactly 10 questions with mixed difficulty
✓ Each question has 4 choices (A, B, C, D)
✓ Presents questions interactively one at a time
✓ Provides immediate feedback with explanations
✓ Calculates score on 0-100 scale
✓ Saves formatted quiz results to file
✓ Uses only relative paths for cross-platform compatibility
✓ Handles errors gracefully with helpful messages
