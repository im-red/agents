# Skill Creation Rules

The following rules must be followed when creating or modifying skills in this project.

## Rule 1: Use `--no-pager` Flag with Git Commands

When using git commands in skills, always specify the `--no-pager` flag when suitable to avoid being blocked by git's pager functionality.

**Example:**
```bash
git --no-pager log -1 --pretty=format:"%H"
git --no-pager diff HEAD~1 HEAD
git --no-pager branch -a
```

**Reason:** Git's default pager can cause interactive prompts that block automated execution in skill workflows. Using `--no-pager` ensures commands complete without requiring user interaction.
