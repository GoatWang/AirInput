# Write Debug Prompt for External LLM Discussion

Generate a structured debug prompt to discuss complex bugs or architectural issues with another LLM.

## Usage

```
/debug_prompt
```

### Recommended Workflow

1. **First, describe the problem in conversation** - paste errors, mention files, explain what you tried
2. **Then call `/debug_prompt`** - the command gathers all context from the conversation

## When to Use

- Complex bugs that require deep investigation
- Architectural issues needing external perspective
- Problems where you've tried multiple solutions without success
- Issues requiring analysis of multiple interconnected files

## Output Location

Save to `prompts/YYYYMMDD_ID_debug_description.md`

## Instructions

When this command is invoked:

### Step 1: Gather Context

Collect the following information from the conversation and codebase:

1. **Problem Summary**: What is failing? Observable symptoms?
2. **Error Messages**: Complete error output with stack traces
3. **Relevant Files**: Which files are involved?
4. **What Was Tried**: Previous attempts and why they failed

### Step 2: Build the Debug Prompt

Generate a markdown file with this structure.

**IMPORTANT**: When generating the prompt, you MUST:
1. Determine the actual filename first (e.g., `20260308_1_debug_whisper_crash.md`)
2. Pre-fill this filename in the "Prompt Being Reviewed" field of the response header

```markdown
# Debug: [Brief Problem Description]

## Response Header (MANDATORY)

**Your response MUST start with this header block:**

```
================================================================
DEBUG RESPONSE METADATA
----------------------------------------------------------------
1. Responder LLM: [Your model name]
2. Prompt Being Reviewed: YYYYMMDD_N_debug_<description>.md
3. Response Generated At: [YYYY-MM-DD HH:MM UTC]
================================================================
```

## Problem Summary
[What is failing, observable symptoms]

## Error Message
```
[Complete error output]
```

## Folder Structure
```
[Relevant directory tree]
```

## Environment Info
- OS: [...]
- Python: [...]
- Virtual Env: [...]

## Current Code
**File**: `path/to/file.py` (lines X-Y)
```python
[Relevant code snippet]
```

## What Happens
1. [Step 1]
2. [Step 2]
3. [Failure occurs]

## What I've Tried
### Attempt 1: [Description]
```python
[Code tried]
```
**Why this doesn't work**: [Factual observation, not speculation]

## Questions
1. [Specific question about behavior]
2. [Question about architectural pattern]

## Expected Behavior
[What should happen instead]

## Additional Context
[Logs, dependencies, system info]
```

### Step 3: Report

Tell the user:
1. The debug prompt file path
2. Summary of what's included
3. Suggest which LLM to use

---

## CRITICAL RULES

1. **DO NOT include guesses or assumptions about the root cause**
2. **DO NOT propose solutions or fixes in the debug prompt**
3. **DO NOT add speculative explanations**
4. **ONLY provide factual information**: code, errors, observations, and questions
5. The goal is to gather complete context for another LLM to analyze independently

## Content Requirements Checklist

- [ ] **Problem Summary**: Clear, concise description of the issue
- [ ] **Error Messages**: Complete error output with stack traces if applicable
- [ ] **Folder Structure**: Visual representation of relevant project directories
- [ ] **Environment Info**: OS, Python version, package manager, virtual environment paths
- [ ] **Current Code**: Relevant code snippets with file paths and line numbers
- [ ] **What Happens**: Step-by-step breakdown of the failure scenario
- [ ] **What I've Tried**: List attempted solutions with explanations of why they failed
- [ ] **Questions**: Specific questions to guide the discussion
- [ ] **Expected Behavior**: Clear success criteria
- [ ] **Additional Context**: Any other relevant information
