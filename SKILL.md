---
name: generate-init-script
description: >
  Generate a Windows init.bat initialization script for open-source projects.
  Use this skill when the user asks to create a one-click setup/init/deploy script
  for a project that has a Python backend and a static HTML+JS+CSS frontend.
  The skill covers dependency checking, automated installation, and credential configuration.
---

# Generate Windows Init Script

## Scope & Applicability

This skill targets **frontend-backend separated projects** where:

- **Frontend**: pure HTML + JS + CSS (no build toolchain required beyond npm)
- **Backend**: Python

Other architectures may differ in specifics but can reference this skill as a baseline.

## Output Requirements

- Produce a **single, complete, runnable** `init.bat` file (Windows Batch).
- **No** PowerShell, WSL, or third-party shell dependencies.
- **No** explanatory prose, pseudocode, or TODO placeholders in the output.
- All user-facing output uses `echo`; all user input uses `set /p`.
- The script **must be idempotent** — safe to run repeatedly without side effects.
- Include clear stage banners and error messages so the user always knows what is happening.

---

## Three-Phase Architecture

The script **must** execute three phases **in strict order**. No phase may be skipped.

### Phase 1 — Baseline Environment Check (Blocking)

**Goal**: Verify that the system has the bare-minimum tools needed to pull and install dependencies.

#### What to check (derive from the project; common examples)

| Tool       | Detection command          |
|------------|----------------------------|
| Python     | `python --version`         |
| Node.js    | `node --version`           |
| npm        | `npm --version`            |
| pip        | `pip --version`            |
| git        | `git --version`            |

> Only include checks that the project actually requires. Read the project's dependency files (e.g. `requirements.txt`, `package.json`) to decide.

#### Behavioral rules

1. Check each dependency **one at a time**, in sequence.
2. If a dependency is **missing**:
   - Print a clear message: `[ERROR] <tool> not detected. Please install <tool> before continuing.`
   - **Block** with: `set /p _="After installing, press ENTER to re-check..."`
   - After ENTER, **re-check the same dependency**.
   - If still missing → loop back (continue blocking).
   - Only advance to the next check after the current one passes.
3. After **all** checks pass, proceed to Phase 2.

#### Constraints

- **Never** exit on first failure.
- **Never** skip a check.
- **Must** use a loop construct (`goto`-based label loop) for the retry mechanism.

#### Reference implementation pattern

```bat
:CHECK_PYTHON
python --version >nul 2>&1
if errorlevel 1 (
    echo [ERROR] Python is not detected. Please install Python and ensure it is in PATH.
    set /p _="After installing, press ENTER to re-check..."
    goto CHECK_PYTHON
)
echo [OK] Python detected.
```

---

### Phase 2 — Automated Installation (Non-Interactive)

**Goal**: Perform all setup steps that can be automated without user input.

#### Typical steps (adapt to the actual project)

**Python backend**:
1. Create virtual environment if it does not exist (`python -m venv venv`).
2. Activate virtual environment (`call venv\Scripts\activate.bat`).
3. Install dependencies (`pip install -r requirements.txt`).

**Node.js frontend**:
1. Run `npm install` in the frontend directory (skip if `node_modules` exists and `package.json` has not changed).

**Other** (examples):
- Database migrations
- Static asset generation
- Config file copying from templates

#### Constraints

- **Zero** user interaction in this phase — no `set /p`, no `pause`.
- Already-completed steps must be **skipped gracefully** (idempotent).
- On failure: print a clear error message and **exit** the script (`exit /b 1`).

#### Reference implementation pattern

```bat
echo.
echo "============================================================"
echo "  Phase 2: Automated Installation"
echo "============================================================"

:: Use goto to avoid nested if/else ( ... ) blocks
if exist "venv" goto SKIP_VENV
echo "[INFO] Creating Python virtual environment..."
python -m venv venv
if errorlevel 1 (
    echo "[ERROR] Failed to create virtual environment."
    exit /b 1
)
echo "[OK] Virtual environment created."
goto AFTER_VENV
:SKIP_VENV
echo "[SKIP] Virtual environment already exists."
:AFTER_VENV

call venv\Scripts\activate.bat

echo "[INFO] Installing Python dependencies..."
pip install -r requirements.txt
if errorlevel 1 (
    echo "[ERROR] Failed to install Python dependencies."
    exit /b 1
)
```

---

### Phase 3 — Credential Entry (Interactive)

**Goal**: Walk the user through entering required credentials, then generate production credential files from example templates.

#### Source of truth

The project may contain one or more **example credential files**, e.g.:

- `example_credentials.py`
- `example_credentials.json`
- `example_config.env`

Each file marks certain fields as **required**. Scan these files to determine what to ask.

#### Behavioral rules

1. Process each example credential file **one at a time**.
2. For each file:
   a. Parse / identify all **required** fields.
   b. For **each** field, before prompting input, print a brief contextual explanation containing:
      - **用途说明**: This key/token is used by which feature or module (e.g., "Used by the LLM chat module to call the OpenAI API").
      - **缺失影响**: What will happen or degrade if the user does not configure it (e.g., "If left empty, the AI conversation feature will be unavailable").
   c. Then prompt the user for that field using `set /p`.
   d. After **all** fields for that file are collected, **copy** (or generate) the example file into the production credential file with the user's values substituted in.
   e. Only then move to the next example credential file.
3. After all credential files are processed, print a success banner and exit normally.

> To write accurate contextual explanations, you **must** trace how each credential is consumed in the codebase — find the `import` / `require` / `load` statements that reference the credential file and identify which module or feature depends on each field. Do not guess; base explanations on actual code references.

#### Constraints

- **Never** batch multiple credential prompts into one input.
- **Never** skip a required field.
- **Never** rename/create the production file before all its fields are collected.
- The credential count is **not hard-coded** — derive it dynamically from the example files.

#### Reference implementation pattern

```bat
echo.
echo "============================================================"
echo "  Phase 3: Credential Configuration"
echo "============================================================"

echo "[INFO] Configuring credentials from example_credentials.py ..."
echo.

echo "  [1/2] OPENAI_API_KEY"
echo "         Used by: LLM chat module (backend/chat_service.py)"
echo "         If not set: AI conversation feature will be unavailable."
echo.
set /p OPENAI_API_KEY="  Enter your OPENAI_API_KEY: "
echo.

echo "  [2/2] AWS_SECRET_KEY"
echo "         Used by: File upload module (backend/storage.py)"
echo "         If not set: File upload and cloud storage will not work."
echo.
set /p AWS_SECRET_KEY="  Enter your AWS_SECRET_KEY: "

echo.
echo "[INFO] Writing credentials.py ..."
echo OPENAI_API_KEY = '!OPENAI_API_KEY!'> credentials.py
echo AWS_SECRET_KEY = '!AWS_SECRET_KEY!'>> credentials.py

echo "[OK] credentials.py created."
```

> The above is illustrative. The actual fields and file format **must** be derived by reading the project's example credential files.

---

## Pre-Generation Checklist

Before writing the script, you **must** inspect the project to answer:

1. **Which system-level tools does the project require?** → Read `requirements.txt`, `package.json`, `README.md`, etc.
2. **What backend setup is needed?** → Virtual environment? Specific Python version? Migrations?
3. **What frontend setup is needed?** → `npm install`? Anything else?
4. **Which example credential files exist, and what fields are required?** → Scan for files matching `example_*credential*`, `example_*config*`, etc.
5. **How is each credential used in the codebase?** → Trace imports/references to credential files; identify the module and feature that consumes each field.
6. **Are there any extra setup steps?** → `.env` files, database init, directory creation, etc.

Only after gathering this information should you generate the script.

---

## Style Guide

- Use `@echo off` and `setlocal enabledelayedexpansion` at the top.
- Use `chcp 65001 >nul` if the project contains non-ASCII content.
- Group code with clear comment banners for each phase.
- Use uppercase labels (e.g., `:CHECK_PYTHON`, `:PHASE_TWO`).
- End the script with a summary banner indicating success.
- Keep the script in a single file — do not split into multiple `.bat` files.
- **Wrap all `echo` string arguments in double quotes** (e.g., `echo "[OK] Done."`). Without quotes, special characters like `(`, `)`, `&`, `|`, `>`, `;`, and non-ASCII text can cause cmd.exe to misinterpret parts of the string as commands.
- **Ensure the `.bat` file uses CRLF line endings.** Files with Unix-style LF endings cause cmd.exe to merge or split lines unpredictably, leading to seemingly random parse errors.

### Batch Syntax Pitfalls

#### Never use `( ... ) > file` blocks for multi-line file writing

The cmd.exe parser pre-parses the entire `( ... )` block before execution. Any special characters inside — including `;`, `://`, `(`, `)`, Chinese/Unicode text, or `"` — will break the parser in unpredictable ways (fragments of words get executed as commands).

**Wrong:**
```bat
(
    echo # Config file
    echo API_KEY = "%API_KEY%"
    echo BASE_URL = "https://api.example.com/v1"
) > config.py
```

**Correct — use sequential `>` (create) and `>>` (append) redirects:**
```bat
echo # Config file> config.py
echo API_KEY = '!API_KEY!'>> config.py
echo BASE_URL = '!BASE_URL!'>> config.py
```

#### Prefer `goto` over `if ( ... ) else ( ... )` for non-trivial blocks

Nested or multi-statement `if/else` blocks are fragile in batch. Use `goto`-based flow instead:

**Fragile:**
```bat
if not exist "venv" (
    echo "[INFO] Creating venv..."
    python -m venv venv
    echo "[OK] Created."
) else (
    echo "[SKIP] Already exists."
)
```

**Robust:**
```bat
if exist "venv" goto SKIP_VENV
echo "[INFO] Creating venv..."
python -m venv venv
echo "[OK] Created."
goto AFTER_VENV
:SKIP_VENV
echo "[SKIP] Already exists."
:AFTER_VENV
```

---

## Common Mistakes to Avoid

| Mistake | Correct behavior |
|---------|-----------------|
| Exit immediately when a Phase 1 check fails | Block and retry in a loop |
| Ask for user input during Phase 2 | Phase 2 must be fully automated |
| Prompt all credentials at once | One field at a time, one file at a time |
| Rename example file before all fields are entered | Rename only after all fields are collected |
| Hard-code the number of credential fields | Derive dynamically from example files |
| Produce pseudocode or partial scripts | Output must be complete and runnable |
| Prompt for a credential without explaining its purpose | Each prompt must include usage context and impact of not configuring |
| Use `( ... ) > file` to write multi-line file content | Use sequential `>` and `>>` redirects — parenthesized blocks break on special chars and Unicode |
| Use `if (...) else (...)` with multiple statements | Prefer `goto`-based flow for non-trivial branching |
| Leave `echo` arguments unquoted | Always wrap `echo` text in double quotes to prevent misparse |
| Save `.bat` with LF line endings | Must use CRLF — LF causes cmd.exe to merge/split lines randomly |
