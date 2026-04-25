---
command: /commit-pro
description: >
  Professional git commit assistant. Triggers automatically when the user asks to make a commit,
  create a commit, do a commit, or save changes to git. Asks which project to commit, stages all
  changes, analyzes them, and proposes a well-formatted conventional commit message for review
  before executing.
---

# git-commit-pro

A professional git commit assistant that follows the Conventional Commits specification.

## Trigger

Activate this skill when the user:
- Asks to "make a commit", "do a commit", "commit changes", "commit this", or similar phrasing
- Invokes `/commit-pro` manually

## Process

### Step 1 — Ask for the project path

Before doing anything, ask the user:

> "In which project or path do you want to make the commit?"

Wait for the user's answer. Do not proceed until they provide a path or project name.

### Step 2 — Stage all changes

Navigate to the provided path and run:

```bash
git -C <path> add .
```

Confirm that staging was successful before continuing.

### Step 3 — Analyze staged changes

Run the following command to inspect what is staged:

```bash
git -C <path> diff --cached
```

If there are no staged changes, inform the user:
> "There are no staged changes in `<path>`. Nothing to commit."

Stop here if nothing is staged.

### Step 4 — Generate commit message

Based on the diff output, generate a commit message following the **Conventional Commits** format:

```
<type>: <short description in lowercase>
```

**Allowed types:**
| Type | When to use |
|------|-------------|
| `feat` | A new feature |
| `fix` | A bug fix |
| `docs` | Documentation only changes |
| `style` | Formatting, missing semicolons, etc. (no logic change) |
| `refactor` | Code restructuring without behavior change |
| `perf` | Performance improvements |
| `test` | Adding or updating tests |
| `chore` | Build process, tooling, or dependency updates |

**Rules:**
- Description must be in **English**
- Description must be **lowercase**
- Description must be **max 100 characters**
- Be specific and descriptive — avoid vague messages like "fix bug" or "update file"

### Step 5 — Present proposal and ask for confirmation

Show the proposed message to the user in this exact format:

---
**Proposed commit message:**
```
<type>: <description>
```
Do you want to commit with this message? (yes / no)

---

### Step 6a — If user accepts

Execute:
```bash
git -C <path> commit -m "<type>: <description>"
```

Confirm success to the user.

### Step 6b — If user rejects

Generate **2 alternative** commit messages using different types or phrasings that are equally valid. Present all three options (original + 2 alternatives) numbered:

---
**Here are 3 options for your commit message:**

1. `<original type>: <original description>`
2. `<alt type>: <alt description>`
3. `<alt type>: <alt description>`

Which one would you like to use? (1, 2, 3, or write your own)

---

Wait for user selection. If the user picks a number, use that message and execute `git -C <path> commit -m "..."`. If the user writes their own message, use it exactly as provided. Do not commit until the user explicitly confirms.
