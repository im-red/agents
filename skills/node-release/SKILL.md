---
name: "node-release"
description: "Plans a Node.js project release, updates files, and outputs required git commands. Invoke when user asks to create or plan a release."
---

# Node Release Planner

This skill plans a release for a Node.js project by calculating the new version, generating a changelog, directly editing files, and providing a step-by-step list of git commands to execute the release.

## Inputs

1. **Release Version (Required)**: Can be `major`, `minor`, `patch`, `X.Y.Z`, or `vX.Y.Z`.
2. **Release Branch (Optional)**: Defaults to `master` if not specified.

## Workflow

1. **Verify Workspace**:
   - Ensure the current branch is clean using `git status`. If it's not clean, halt and ask the user to commit or stash changes.
   - Get the current branch name (e.g., `git branch --show-current`).

2. **Calculate Target Version**:
   - Read the current version from `package.json`.
   - If the input is `major`, `minor`, or `patch`, calculate the target version based on the current version.
   - Extract the target version in the format `vX.Y.Z` (referred to as `RELEASE_VERSION`).
   - Extract the target version in the format `X.Y.Z` (referred to as `VERSION_NO_V`) for updating `package.json`.

3. **Generate CHANGELOG**:
   - Use commands like `git log <release_branch>..HEAD` and `git diff <release_branch>..HEAD` to gather the context of changes.
   - Generate a brief, feature-focused changelog prepended to the existing file. Focus on user-facing improvements and ignore underlying technology, refactoring, or internal code structure.
   - **Format Rule**: 
     - Use `## [<RELEASE_VERSION>] - YYYY-MM-DD` as the version header.
     - Group items by categories like `### New Features`, `### Improvements`, `### Fixes`.
     - Format each item as `- **<Feature Name>** - <Description>`.

4. **Directly Edit Files**:
   - **Update `CHANGELOG.md`**: Directly edit the `CHANGELOG.md` file using your file editing tools to prepend the new changelog content.
   - **Update `package.json`**: Directly edit the `package.json` file using your file editing tools to update the `"version"` field to `<VERSION_NO_V>`.

5. **Output Release Plan**:
   - Output the generated CHANGELOG to the user.
   - Provide a single block of git commands that the user can run to execute the rest of the release process.

## Command Output Requirements

The command list must perform the following actions sequentially. **Do not output commands for updating `CHANGELOG.md` or `package.json` as you must do that directly.**

1. **Commit the release**: Command to create a git commit. The commit message **must** be formatted as `chore: update changelog for <RELEASE_VERSION>`.
2. **Create a git tag**: Command to tag the release. The tag message **must** be a simplified version of the changelog. Multiple `-m` arguments should be used to support multiline messages.
3. **Switch to the release branch**: Command to checkout the release branch (e.g., `git checkout <release_branch>`).
4. **Merge the current branch**: Command to merge the developing branch into the release branch (e.g., `git merge <current_branch>`).
5. **Push to remote**: Command(s) to push the release branch and the new tag to the remote repository.
6. **Switch back**: Command to switch back to the original current branch.

### Example Output

```bash
# 1. Commit the release
git add CHANGELOG.md package.json package-lock.json
git commit -m "chore: update changelog for vX.Y.Z"

# 2. Create a git tag
git tag -a vX.Y.Z -m "Release vX.Y.Z" -m "- Feature 1" -m "- Fix 1"

# 3. Switch to release branch
git checkout master

# 4. Merge current branch
git merge feature-branch

# 5. Push branch and tag
git push origin master
git push origin vX.Y.Z

# 6. Switch back
git checkout feature-branch
```