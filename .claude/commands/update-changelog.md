# dynamic-changelog-updater

Automatic changelog manager that reads git commits, consolidates changes in English, and proposes a semantic version increment following strict SemVer rules. Maintains `releases/changelog.md` as the single source of truth for release history. The canonical version for this project is the `version` field inside `pyproject.toml` under `[project]`.

## Trigger

Activate this skill when:
- The user indicates that new commits were made or a feature/fix is ready to release.
- The user asks "what version should I bump to?", "update the changelog", "prepare the release", or similar.
- The user asks about the current version of the project.
- The user invokes `/update-changelog` manually.

## Process

### Step 1 — Read or Initialize `releases/changelog.md`

Check if `releases/changelog.md` exists.

- **If it exists**: read its current content to determine the latest version entry.
- **If it does NOT exist**: create the directory and file with this initial header:

```markdown
# Changelog

All notable changes to this project will be documented in this file.

This project adheres to [Semantic Versioning](https://semver.org/).

---
```

Extract the **current version** from the latest entry in the changelog (e.g., `## [1.2.0] - 2026-04-20`). If no version exists yet, start from `0.1.0`.

### Step 2 — Identify Branch and Read Commits

Run the following commands to gather context:

```bash
git branch --show-current
git log --oneline --since="$(git log --format=%aI -1 $(git describe --tags --abbrev=0 2>/dev/null || echo HEAD~20) 2>/dev/null || date -v-30d +%Y-%m-%d)" 2>/dev/null || git log --oneline -20
```

If tags are not available, fall back to the last 20 commits or commits since the last changelog entry date.

Parse each commit message. Group them by Conventional Commits type if present:
- `feat:`, `fix:`, `docs:`, `style:`, `refactor:`, `perf:`, `test:`, `chore:`
- `BREAKING CHANGE:` in footer or `!` after type (e.g., `feat!:`)

### Step 3 — Consolidate Changes in English

From the parsed commits, produce a clean, human-readable summary in **English** grouped by category:

```
Breaking Changes  → commits with BREAKING CHANGE or feat!/fix!
New Features      → feat: commits
Bug Fixes         → fix: commits
Performance       → perf: commits
Refactors         → refactor: commits (if externally visible)
Internal/Chores   → chore:, docs:, test:, style: (grouped together, brief)
```

Write each entry as a past-tense action sentence. Examples:
- `feat: add phone number to user profile` → "Added phone number field to user profile"
- `fix: return float in exchange rate service` → "Fixed exchange rate service returning string instead of float"
- `chore: extract HTTP string constants` → "Extracted HTTP message strings to shared constants file"

Omit entries that have no user-facing impact (pure internal style fixes) unless the user asks to include them.

### Step 4 — Propose Version Increment

Using the following **strict SemVer rules**, determine the correct version bump:

#### MAJOR (X.0.0) — Breaking Changes
Increment MAJOR when ANY of the following is true:
- An existing endpoint was **removed or renamed**
- A response field was **removed or renamed**
- A request field that was optional became **required**
- A return type changed in an **incompatible** way (e.g., `float` → `string`, object shape changed)
- A previously deprecated feature was **fully removed**
- A **deep architectural change** that breaks existing integrations (e.g., auth mechanism replaced)

#### MINOR (x.Y.0) — Backward-Compatible Features
Increment MINOR when ANY of the following is true (and no MAJOR trigger applies):
- A **new endpoint** was added
- A new **optional field** was added to request or response
- An existing feature was **deprecated** (but still works)
- A **performance improvement** with visible behavior difference (e.g., new pagination defaults)
- New **configuration option** added

#### PATCH (x.y.Z) — Bug Fixes and Invisible Changes
Increment PATCH when ALL changes are:
- **Bug fixes** that restore originally intended behavior
- **Security patches** that don't change the API surface
- **Internal refactors** invisible to API consumers (e.g., extracting constants, renaming files, Clean Architecture alignment)
- **Documentation** updates only
- **Test** additions or fixes only

Present the proposal to the user in this format:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
CHANGELOG UPDATE PROPOSAL
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Current version : X.Y.Z
Proposed bump   : PATCH | MINOR | MAJOR
New version     : X.Y.Z+1

Reason: <one sentence explaining why this bump level was chosen>

Changes to add:
  Breaking Changes:
    - <entry>
  New Features:
    - <entry>
  Bug Fixes:
    - <entry>
  Internal:
    - <entry>
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Confirm? (yes / adjust / cancel)
```

Wait for user confirmation before writing anything.

### Step 5 — Write the Changelog Entry

If the user confirms (or adjusts the version/entries), prepend the new entry to `releases/changelog.md` immediately after the header block:

```markdown
## [X.Y.Z] - YYYY-MM-DD

### Breaking Changes
- Entry here

### New Features
- Entry here

### Bug Fixes
- Entry here

### Internal
- Entry here

---
```

Only include sections that have at least one entry. Omit empty sections.

After writing, confirm to the user:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
CHANGELOG UPDATED
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
File    : releases/changelog.md
Version : X.Y.Z
Date    : YYYY-MM-DD
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

## Important Rules

- Always write changelog entries in **English**, regardless of the user's language.
- Never overwrite an existing version entry. If the version already exists, append a note or ask the user.
- If commits use no Conventional Commits format, infer the type from the message content (fix, add, remove, update, refactor, etc.).
- If there is ambiguity about the bump level (e.g., a refactor that may or may not break clients), **ask the user** before proposing.
- Do not create a git tag automatically — only update the file. Let the user decide when to tag.
- The canonical project version lives in `pyproject.toml` under `[project] version = "X.Y.Z"`. After confirming the new version, update this field as well.
