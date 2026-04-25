# release-finalizer

Consolidates the working changelog into an official versioned release file. Reads the accumulated changes in `releases/changelog.md`, determines the correct SemVer bump, creates a permanent release snapshot at `releases/vX.X.X.md`, updates the `version` file, and resets the changelog for the next cycle.

## Trigger

Activate this skill when:
- The user creates or switches to a branch whose name starts with `release/` (e.g., `release/1.2.0`, `release/next`).
- The user asks to "finalize the release", "consolidate the changelog", "generate a new official version", "cut a release", or similar phrasing.
- The user invokes `/finalize-release` manually.

## Process

### Step 1 — Read Source Files

Read both files:

1. `releases/changelog.md` — the working changelog with accumulated changes since the last release.
2. `version` (root of the project) — the current version string (e.g., `1.2.0`). If this file does not exist, assume `0.0.0` and inform the user.

If `releases/changelog.md` is empty or contains only the header with no entries, **stop** and notify the user:

> "There are no pending changes in `releases/changelog.md`. Nothing to release."

### Step 2 — Identify the Highest Suggested Version Bump

Scan `releases/changelog.md` for any lines that indicate a suggested version bump. These may appear as:
- Comments like `<!-- Suggested Version: MAJOR -->`, `<!-- Suggested: MINOR -->`, `Suggested Version: PATCH`, or any similar annotation left by the `/update-changelog` skill.
- Section headers like `### Breaking Changes` (implies MAJOR), `### New Features` (implies MINOR), `### Bug Fixes` or `### Internal` (implies PATCH).

Apply **priority order** — the highest level found wins:

```
MAJOR > MINOR > PATCH
```

| Signal | Bump Level |
|--------|-----------|
| `Breaking Changes` section has entries, or explicit MAJOR annotation | MAJOR |
| `New Features` section has entries, or explicit MINOR annotation | MINOR |
| Only `Bug Fixes` / `Internal` sections, or explicit PATCH annotation | PATCH |

Display the detected bump level to the user before proceeding.

### Step 3 — Calculate the New Version

Parse the current version from the `version` file as `MAJOR.MINOR.PATCH`.

Apply the bump:

| Bump | Rule |
|------|------|
| MAJOR | `(MAJOR+1).0.0` |
| MINOR | `MAJOR.(MINOR+1).0` |
| PATCH | `MAJOR.MINOR.(PATCH+1)` |

Present the version calculation to the user:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
RELEASE FINALIZATION PLAN
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Current version : X.Y.Z
Detected bump   : MAJOR | MINOR | PATCH
New version     : X.Y.Z+1

Files to create/modify:
  + releases/vX.Y.Z+1.md  (new release snapshot)
  ~ version               (updated to X.Y.Z+1)
  ~ releases/changelog.md (reset to header only)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Confirm? (yes / adjust version / cancel)
```

Wait for user confirmation. If the user wants to adjust the version, accept their input and use it as the new version instead.

### Step 4 — Create the Release Snapshot

Create the file `releases/vX.Y.Z+1.md` with the following structure:

```markdown
# Release vX.Y.Z+1

**Date**: YYYY-MM-DD
**Type**: MAJOR | MINOR | PATCH

---

<content of releases/changelog.md, with the following transformations:>
```

Transformations to apply when copying from `changelog.md`:
- Remove the top-level `# Changelog` header (it belongs to the working file).
- Remove any lines that match `<!-- Suggested Version: ... -->` or similar annotations.
- Remove any lines that say `Suggested Version:` (these are internal notes, not release content).
- Keep all section headers (`### Breaking Changes`, `### New Features`, etc.) and their entries.
- Keep the `---` dividers.

### Step 5 — Update the Version

Update the version in **two places**:

1. The `version` file at the project root (plain text, single line): `X.Y.Z+1`
2. The `version` field in `pyproject.toml` under `[project]`:

```toml
[project]
version = "X.Y.Z+1"
```

Both must be in sync. If `pyproject.toml` does not have a `version` field, add it.

### Step 6 — Reset `releases/changelog.md`

Overwrite `releases/changelog.md` with only the header block, ready for the next cycle:

```markdown
# Changelog

All notable changes to this project will be documented in this file.

This project adheres to [Semantic Versioning](https://semver.org/).

---
```

### Step 7 — Confirm Completion

Display the final summary:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
RELEASE FINALIZED
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
New version     : X.Y.Z+1
Release file    : releases/vX.Y.Z+1.md
version updated : X.Y.Z → X.Y.Z+1
Changelog reset : releases/changelog.md

Next steps:
  1. Review releases/vX.Y.Z+1.md
  2. Commit: git add releases/ version pyproject.toml && git commit -m "chore: release vX.Y.Z+1"
  3. Tag:    git tag vX.Y.Z+1
  4. Merge release branch into main
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

## Important Rules

- Never overwrite an existing `releases/vX.Y.Z.md` file. If it already exists, warn the user and stop.
- Never create the release without user confirmation from Step 3.
- If `version` file does not exist, create it with the calculated version after confirmation.
- Do not run `git commit` or `git tag` automatically — only suggest the commands. Let the user control the git workflow.
- All release content must be in **English**, regardless of the user's language.
