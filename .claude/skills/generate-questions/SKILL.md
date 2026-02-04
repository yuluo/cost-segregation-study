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
- **Default query format**: "ASCSP cost segregation [topic descriptor] professional standards"
- **If enhanced_search_query exists from weakness analysis**: Use that query instead to target weak concepts
- Example: For `1.1-ascsp-mission` → "ASCSP cost segregation mission professional standards"

Combine the study material content with web search results to inform question generation.

### 3.5. Analyze Quiz History for Weakness Targeting (Optional)

**Only execute this step if quizHistory exists for the current topic in progress.json.**

**Steps**:

1. **Read progress.json** from project root
2. **Parse the topic** to locate the subsection (e.g., topic `2.4` → `modules["02"].subsections["2.4"]`)
3. **Check quizHistory length**:
   - If `quizHistory` is empty or has 0 entries: Skip this step entirely and proceed to Step 4 with standard question generation
   - If `quizHistory` has 1+ entries: Continue with weakness analysis

4. **Read the most recent quiz file** from quizHistory:
   - Get the latest entry's `quizFile` path (the last item in the quizHistory array)
   - Use Read tool to load the quiz markdown file
   - **Note**: Always analyze only the most recent quiz, not trends across multiple attempts

5. **Parse missed questions** from the quiz file:
   - Search for lines containing `**Correct Answer**: [A-D] ✗` (incorrect answers)
   - Extract the question number from the `## Question [N]` header above each incorrect answer
   - Categorize by difficulty:
     - Questions 1-3: Recall weaknesses
     - Questions 4-7: Application weaknesses
     - Questions 8-10: Synthesis weaknesses
   - Count incorrect answers in each category

6. **Extract weak concepts**:
   - Find the Summary section at the end of the quiz file
   - Locate the line starting with `- **Topics Covered**:`
   - Parse all concepts listed after the colon (comma-separated)
   - Cross-reference missed question numbers with concepts from the Topics Covered list
   - Build a list of 3-5 key weak concepts

7. **Calculate weakness profile**:
   - Recall weakness: `recall_incorrect_count / 3`
   - Application weakness: `application_incorrect_count / 4`
   - Synthesis weakness: `synthesis_incorrect_count / 3`
   - Identify the weakest category (highest percentage incorrect)

8. **Build focused generation instructions**:

   **If overall score >= 70%** (decent performance):
   - Keep standard 3-4-3 distribution
   - Add instruction: "Prioritize these weak concepts in synthesis questions: [weak_concepts_list]"
   - Enhance web search with: "ASCSP cost segregation [topic] [weak_concept_1] [weak_concept_2]"

   **If overall score 40-69%** (moderate weaknesses):
   - Adjust distribution based on weakest category:
     - If recall is weakest: 5 recall, 3 application, 2 synthesis
     - If application is weakest: 2 recall, 5 application, 3 synthesis
     - If synthesis is weakest: 2 recall, 3 application, 5 synthesis
   - Add instruction: "Focus heavily on these weak concepts: [weak_concepts_list]. Create questions that test understanding from multiple angles."
   - Enhance web search with: "ASCSP cost segregation [topic] [weak_concept_1] [weak_concept_2] detailed explanation examples"

   **If overall score < 40%** (significant weaknesses):
   - Moderate focus on weakest category (5 questions)
   - Distribution:
     - If recall is weakest: 5 recall, 3 application, 2 synthesis
     - If application is weakest: 2 recall, 5 application, 3 synthesis
     - If synthesis is weakest: 2 recall, 3 application, 5 synthesis
   - Add instruction: "Generate questions that reinforce fundamental understanding of: [weak_concepts_list]. Use clear, educational distractors that address common misconceptions."
   - Enhance web search with: "ASCSP cost segregation [topic] [weak_concept_1] fundamentals basics examples practice"

   **If overall score = 100%** (perfect performance):
   - Heavy synthesis focus for advanced practice
   - Distribution: 1 recall, 2 application, 7 synthesis
   - Add instruction: "Generate advanced synthesis questions that combine multiple concepts and test edge cases. Challenge the learner with complex scenarios."
   - Enhance web search with: "ASCSP cost segregation [topic] advanced scenarios edge cases complex examples"

9. **Store the focused generation profile**:
   - Create variables to pass to Step 4:
     - `difficulty_distribution`: Array like [3, 4, 3] or [5, 3, 2]
     - `focus_concepts`: List of weak concepts to emphasize
     - `generation_instruction`: Custom instruction text to add
     - `enhanced_search_query`: Modified web search query

**Error Handling**:
- If progress.json doesn't exist: Skip weakness analysis, use standard generation
- If quizFile path is invalid or file doesn't exist: Skip weakness analysis, use standard generation
- If quiz file format is unexpected: Log warning, skip weakness analysis, use standard generation
- Always have fallback to standard question generation

**User Communication**:
If weakness analysis succeeds, display:
```
📊 Analyzing previous quiz performance...
✓ Found [N] quiz(es) for this topic
✓ Focusing on weak areas: [list 2-3 key weak concepts]
✓ Adjusting difficulty distribution: [recall_count] recall, [app_count] application, [synth_count] synthesis
```

If no history found:
```
📚 No previous quiz history found - generating standard quiz
```

### 4. Generate 10 Multiple-Choice Questions

**Use the difficulty distribution determined in Step 3.5** (or default [3, 4, 3] if no weakness analysis):

- **Recall questions** (default: Questions 1-3): Concept recall (definitions, key facts, terminology)
- **Application questions** (default: Questions 4-7): Application/analysis (scenarios, "what would you do", practical situations)
- **Synthesis questions** (default: Questions 8-10): Synthesis (combining concepts, edge cases, critical thinking)

**If focus_concepts were identified in Step 3.5**: Prioritize those concepts in questions. Ensure weak concepts appear in:
- Question content (directly testing the weak concept)
- Distractors (addressing misconceptions about the weak concept)
- Explanations (providing deeper understanding of the weak concept)

**If generation_instruction was created in Step 3.5**: Follow that custom instruction for question generation approach.

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
5. **Analyze Quiz Performance and Generate Notes**:
   - Read the quiz file that was just saved
   - Parse the quiz results to extract:
     - Questions 1-3 (recall): Count correct/incorrect
     - Questions 4-7 (application): Count correct/incorrect
     - Questions 8-10 (synthesis): Count correct/incorrect
     - Topics covered from Summary section (line starting with "- **Topics Covered**:")
     - Map incorrect questions to their concepts from Topics Covered list
   - Check quizHistory length to determine if this is first quiz or has previous attempts
   - Generate notes string using this format:

   **If first quiz (quizHistory has only 1 entry after adding current quiz)**:
   ```
   [Performance level] ([score]/100). Strengths: [difficulty breakdown + strong concepts]. Weaknesses: [difficulty breakdown + missed concepts].
   ```

   **If previous quiz exists (quizHistory has 2+ entries)**:
   ```
   [Trend phrase] from [prev_score]→[current_score]. Strengths: [difficulty breakdown + strong concepts]. Weaknesses: [difficulty breakdown + missed concepts].
   ```

   **Performance level mapping** (90-100: "Strong performance", 80-89: "Good performance", 70-79: "Moderate performance", 60-69: "Fair performance", 0-59: "Needs improvement")

   **Trend phrase mapping**:
   - If score increased: "Improved +[diff]"
   - If score decreased: "Declined -[diff]"
   - If score same: "Maintained"

   **Difficulty breakdown format**:
   - 3/3 or 4/4 correct: "Excellent [area]"
   - 2/3 or 3/4 correct: "Good/Solid [area]"
   - 1/3 or 2/4 correct: "Weak [area]"
   - 0/3 or 0-1/4 correct: "Poor [area]"

   **Concept selection**:
   - Parse "Topics Covered" from Summary section by splitting on commas
   - For strengths: Select 2-4 key concepts from correctly answered questions
   - For weaknesses: Select 2-4 key concepts from incorrectly answered questions
   - Map questions to concepts based on question number and Topics Covered order
   - Keep total note length to ~200-250 characters

   **Error handling**:
   - If quiz file cannot be read: Set notes to "Score: [X]/100"
   - If parsing fails: Set notes to "Score: [X]/100 - analysis unavailable"
   - If Topics Covered missing: Use difficulty breakdown only
   - Always populate notes with at least minimal information

6. **Update the notes field**:
   - Set `notes` to the generated analysis string from previous step
   - This replaces any existing notes content

7. **Update metadata**: Set `metadata.lastModified` to current timestamp
8. **Write updated progress.json** back to file

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
✓ Performance notes: [First 80 chars of notes]...
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
