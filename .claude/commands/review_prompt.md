# Review Prompt Generator

Generate a structured prompt to ask an external LLM to review code implementation for logical errors, missing updates, and feasibility issues.

## Usage

```
/review_prompt <description_of_work>
```

## Instructions

When this command is invoked:

### Step 1: Gather Context

1. **Identify the work to review** from the user's description
2. **Find relevant files**:
   - Check git diff for recent changes related to the work
   - Identify modified Python scripts
   - Find any related plan files in `prompts/` folder

### Step 2: Collect Code Diffs

For each modified file:
1. Get the git diff showing the changes
2. Extract key code snippets that are critical to the implementation
3. Note the data structures and file I/O patterns

### Step 3: Generate the Review Prompt

Create a markdown file in `prompts/` folder with naming convention:
```
prompts/YYYYMMDD_N_review_<description>.md
```

**IMPORTANT**: When generating the prompt, you MUST:
1. Determine the actual filename first (e.g., `20260308_1_review_key_listener.md`)
2. Pre-fill this filename in the "Prompt Being Reviewed" field of the response header
3. Do NOT leave it as a placeholder

#### Required Sections:

```markdown
# Review Request: [Title]

## Task Objective
[Clear description of what was implemented and the goal]

## Files Modified/Created
| File | Type | Purpose |
|------|------|---------|
| ... | ... | ... |

## Code Diffs
[Include all relevant code diffs with explanations]

## Review Questions
[Specific checklist of things to verify]

### 1. Logical Errors
- [ ] Question 1

### 2. Data Flow Verification
- [ ] Question 1

### 3. Potential Runtime Errors
- [ ] Question 1

## Important Instructions

### DO NOT Modify Code

**You must NOT modify any code files.** This is a review-only task.

### Response Header (MANDATORY)

**Your response MUST start with this header block:**

```
================================================================
REVIEW RESPONSE METADATA
----------------------------------------------------------------
1. Reviewer LLM: [Your model name]
2. Prompt Being Reviewed: YYYYMMDD_N_review_<description>.md
3. Response Generated At: [YYYY-MM-DD HH:MM UTC]
================================================================
```

### Report Language

**整份 Review 報告必須使用繁體中文撰寫。** 包括：
- Summary（摘要）
- Issues Found（發現的問題）
- Recommendations（建議）
- 所有說明文字與分析

唯一例外：程式碼片段、檔案路徑、變數名稱、英文專有名詞保留原文。

### Output Your Review

Write your review results to a new file in the `prompts/` folder following this naming convention:
```
prompts/YYYYMMDD_ID_description.md
```
```

### Step 4: Report to User

Tell the user:
1. The review prompt file path created
2. Summary of what's included
3. Suggest which LLM to use for the review

---

## Code Review Checklist

1. **Logical Errors** — Function parameters, return types, data structure alignment
2. **Data Flow** — Input/output file consistency, field pass-through
3. **Filename Consistency** — Correct file reads/writes
4. **Runtime Safety** — KeyError risks, optional field handling, file existence checks

## Key Rules

### MUST Include:
- [ ] Complete code diffs for all modified files
- [ ] Specific review questions as checklist
- [ ] "DO NOT modify code" instruction

### MUST NOT:
- [ ] Ask reviewer to write fixed code
- [ ] Include subjective/creative review criteria
