# Dev Instruction

## ⚠️ MOST IMPORTANT PRINCIPLE: FACE THE BUG, DON'T WORK AROUND IT ⚠️

**NEVER use workarounds to bypass bugs. This is the #1 development standard.**

When you encounter an error or unexpected behavior:
1. **INVESTIGATE THE ROOT CAUSE** - Don't just make the error go away
2. **FIX THE ACTUAL PROBLEM** - Not the symptom
3. **If a file is empty when it shouldn't be** - Find out WHY it's empty, don't write code to "handle" empty files
4. **If an API returns unexpected data** - Debug WHY, don't add fallbacks to hide it
5. **If a step fails** - Trace back to find the real bug, don't make downstream steps "tolerant"

**WORKAROUNDS ARE FORBIDDEN because they:**
- Hide the real problem, making future debugging impossible
- Accumulate technical debt exponentially
- Cause cascading failures that are untraceable
- Make the system unreliable in ways you can't predict

---

## Development Guidelines

1. Clarify: ask if not clear before start working
2. Correct me: if the design is not feasible
3. **Dictionary Key Access Rules (STRICT POLICY)**:
   - **DEFAULT: Use direct key access `dict["key"]`** - This is the standard approach for ALL keys
   - **EXCEPTION: Use `.get("key", default)` ONLY when**:
     - You have explicit confirmation the key is optional in the data schema
     - You are deliberately handling missing keys with a fallback value
     - You can justify why the key might not exist
   - **This makes code fail fast on missing required keys and helps catch data structure issues early**
   - **When in doubt, use `dict["key"]` - let it fail if the key is missing so you can fix the real problem**
   - Example:
     ```python
     # GOOD: Use direct access by default
     trigger_key = config["trigger_key"]
     model_path = config["model_path"]

     # GOOD: Only use .get() for truly optional keys with justification
     polish_prompt = config.get("polish_prompt", "")  # Optional: polish is disabled by default

     # BAD: Don't use .get() by default "just in case"
     trigger_key = config.get("trigger_key")  # Wrong! trigger_key should always exist
     ```
4. **Error Handling and Fail-Fast Policy**:
   - **Do NOT use tricks to avoid raising errors** - If expected data is missing, raise an appropriate exception
   - **Fail fast with clear error messages** - Better to crash early than silently continue with bad data
   - **If data is expected to exist and doesn't, RAISE** - Use appropriate exception types
   - **Choose compatible exceptions**:
     - `ValueError`: For invalid values or missing required data
     - `FileNotFoundError`: For missing files (Python will raise this automatically for file operations)
     - `KeyError`: For missing dictionary keys (Python will raise this automatically with `dict["key"]`)
   - **No Overprotection Policy (STRICT)**: Do NOT wrap external calls in try-except blocks just to keep the app running
     - **Let errors propagate** - Errors are visible in logs and can be debugged immediately
     - **Silent failures cause bad UX** - If whisper.cpp crashes silently, user gets no transcription and no error
     - **The correct pattern**: Call directly without try-except. If it fails, the app stops with a clear stack trace
       ```python
       # GOOD: Let errors propagate
       result = subprocess.run(whisper_cmd, capture_output=True, check=True)

       # BAD: try-except hides failures
       try:
           result = subprocess.run(whisper_cmd, capture_output=True)
       except Exception as e:
           logger.error(f"Failed: {e}")  # Wrong! Hides the real problem
       ```

   - **File Existence Check Policy (STRICT)**: Do NOT use `if os.path.exists()` patterns that silently continue when required input files are missing
     - **RAISE FileNotFoundError** when a required input file is missing - don't just log and continue
       ```python
       # BAD: Silent fallback on required file
       if os.path.exists(model_path): load_model(model_path)
       else: logger.warning("model not found")  # continues silently!

       # GOOD: Fail fast
       if not os.path.exists(model_path):
           raise FileNotFoundError(f"Required model file not found: {model_path}")
       ```

## Python Execution

Use `uv` to run all Python code:
```bash
cd /Users/wanghsuanchung/Projects/AirInput && uv run python script.py
```

## File Naming Convention

When creating ANY files in the `prompts/` folder (including .py, .md, .yaml, .json, etc.), use the naming convention `YYYYMMDD_ID_description.extension` where:
- `YYYYMMDD` is the current date (e.g., 20260308)
- `ID` is a sequential number starting from 0 for each day (0, 1, 2, ...)
- `description` is a brief task description using underscores
- `extension` is the appropriate file extension (.py, .md, .yaml, .json, etc.)
- **Auto-determining the ID**: Before creating a file, scan the `prompts/` folder for files with today's date prefix:
  ```bash
  ls prompts/ | grep "^YYYYMMDD" | cut -d'_' -f2 | sort -n | tail -1
  ```
  Then use (max_id + 1) as the new ID. If no files exist for today, start with 0.

## Temporary & Intermediate Files

All working files go in `prompts/`, which is gitignored.

## Git

Commit messages always in English.

## Special Commands

Custom slash commands are defined in `.claude/commands/`. See `.claude/commands/*.md` for full documentation.

**Skills location:** `.claude/skills/` (project-local, not `~/.claude/skills/`)

## AirInput — Project-Specific

Push-to-talk voice input for macOS (headless). Core stack: Breeze ASR 25 via whisper.cpp + optional llama.cpp polish.

See `prompts/20260308_0_airinput_project_proposal.md` for full architecture and implementation plan.
