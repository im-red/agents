---
name: "changelog-generator"
description: "Generates a feature-focused changelog from git diffs and commits between two branches. Invoke when user asks to generate a changelog or release notes."
---

# Changelog Generator

This skill generates a concise, feature-focused changelog based on the differences between a developing branch and a releasing branch.

## Instructions

1. **Identify Branches**:
   - Determine the **developing branch** (first input) and the **releasing branch** (second input).
   - If no branch names are provided by the user, default to the **current branch** as the developing branch, and the **`master`** branch as the releasing branch.

2. **Gather Context**:
   - Use terminal commands (e.g., `git log <releasing_branch>..<developing_branch>` and `git diff <releasing_branch>..<developing_branch>`) to get the commit messages and code differences.

3. **Generate Changelog**:
   - Analyze the diff and commit messages to identify the changes.
   - The changelog **must be brief**.
   - Focus strictly on **features** and user-facing improvements, ignoring the underlying technology, refactoring details, or internal code structure.

4. **Output Format**:
   - Save all generated changelog content into a **single file** (e.g., `CHANGELOG.md` or as requested by the user).
   - Additionally, output a commit message for this changelog update. Use **`vX.Y.Z`** as the version placeholder in the commit message.

5. **Git Tag Message**:
   - Also generate a **tag message** for the release tag (e.g., `vX.Y.Z`).
   - The tag message should contain the version number on the first line, followed by a summary of the changes (same categories as the changelog: new features, improvements, bug fixes, etc.).
   - Output the tag message along with the commit message so the user can create the tag with `git tag -a vX.Y.Z -m "<tag message>"`.