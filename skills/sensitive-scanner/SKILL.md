---
name: "sensitive-scanner"
description: "Scans project for sensitive information (API keys, passwords, secrets). Invoke when user wants to check for leaked credentials or before committing/pushing code."
---

# Sensitive Information Scanner

This skill scans a project for sensitive information such as API keys, passwords, tokens, and other secrets that should not be committed to version control.

## When to Use

- Before committing code to check for accidentally included secrets
- When auditing a codebase for security issues
- Before pushing to a public repository
- When reviewing pull requests for sensitive data leaks
- Periodic security audits of the codebase

## Working Modes

**Default Mode:** Scan Local Changes - Covers both staged and unstaged changes, the most common use case before committing.

### Mode 1: Scan Local Changes (Default)

Scans all local changes including both staged and unstaged files. This is the default mode when no specific mode is requested.

```
User request: "scan for sensitive info"
User request: "check my changes for secrets"
User request: "scan local changes"
User request: "scan staged files" (staged only)
User request: "scan unstaged files" (unstaged only)
```

**Execution:**
1. Get list of staged files: `git --no-pager diff --cached --name-only`
2. Get list of unstaged files: `git --no-pager diff --name-only`
3. Combine and scan all local changes
4. Report findings grouped by staged/unstaged status

**Note:** User can specify "staged only" or "unstaged only" to limit the scope.

### Mode 2: Scan All Git-Tracked Files

Scans all files currently managed by git (excluding files in .gitignore).

```
User request: "scan all files for sensitive info"
User request: "check the entire project for secrets"
User request: "scan everything"
```

**Execution:**
1. Get list of all tracked files: `git --no-pager ls-files`
2. Scan each file for sensitive patterns
3. Report all findings with file paths and line numbers

### Mode 3: Scan Last N Commits

Scans changes in the last N commits to find newly introduced sensitive information.

```
User request: "scan last 5 commits for secrets"
User request: "check recent changes for sensitive info"
User request: "scan last 3 commits"
```

**Execution:**
1. Get diff for the specified commits: `git --no-pager diff HEAD~N HEAD`
2. Scan only the changed/added lines
3. Report findings with commit info and file locations

## Sensitive Information Patterns

Scan for these categories of sensitive data:

### API Keys & Tokens
- AWS Access Key ID: `AKIA[0-9A-Z]{16}`
- AWS Secret Access Key: `[A-Za-z0-9/+=]{40}`
- GitHub Token: `ghp_[a-zA-Z0-9]{36}`
- GitHub OAuth: `gho_[a-zA-Z0-9]{36}`
- GitLab Token: `glpat-[a-zA-Z0-9_-]{20}`
- Slack Token: `xox[baprs]-[0-9]{10,13}-[a-zA-Z0-9]{24}`
- Stripe Key: `sk_live_[a-zA-Z0-9]{24,}`
- Stripe Publishable Key: `pk_live_[a-zA-Z0-9]{24,}`
- Google API Key: `AIza[a-zA-Z0-9_-]{35}`
- OpenAI API Key: `sk-[a-zA-Z0-9]{20,}T3BlbkFJ[a-zA-Z0-9]{20,}`
- Generic API Key patterns

### Passwords & Secrets
- Password in assignment: `password\s*=\s*['"][^'"]+['"]`
- Password in URL: `://[^:]+:[^@]+@`
- Secret key patterns: `secret[_-]?key\s*=\s*['"][^'"]+['"]`
- Private key headers: `-----BEGIN (RSA |DSA |EC |OPENSSH )?PRIVATE KEY-----`
- Bearer tokens: `Bearer\s+[a-zA-Z0-9._-]+`

### Database & Connection Strings
- Connection strings: `(mysql|postgres|mongodb|redis)://[^\s]+`
- Database URLs with credentials
- JDBC connection strings with passwords

### Local Paths
- Windows absolute paths: `[A-Za-z]:\\[^\s'"<>|*?\n]+` (e.g., `C:\Users\username\...`)
- Windows UNC paths: `\\\\[^\s'"<>|*?\n]+` (e.g., `\\server\share\...`)
- Unix absolute paths: `/(?:home|Users|root|var|etc|opt|srv)/[^\s'"<>|*?\n]+`
- User home paths: `~/(?:Documents|Desktop|Downloads|\.ssh|\.config|\.aws)/[^\s'"<>|*?\n]+`
- Paths containing usernames: `/(?:Users|home)/[a-zA-Z0-9_]+/`
- SSH config paths: `\.ssh/[^\s'"<>|*?\n]+`
- AWS credentials paths: `\.aws/[^\s'"<>|*?\n]+`
- Config file paths: `\.(?:config|conf|cfg)/[^\s'"<>|*?\n]+`

**Note:** Local paths may reveal:
- User's real name (from username in path)
- System structure and organization
- Installed software locations
- Project locations on developer's machine

### Other Sensitive Data
- Email addresses (when in config/context suggests credentials)
- IP addresses in sensitive contexts
- Credit card numbers: `\b\d{4}[-\s]?\d{4}[-\s]?\d{4}[-\s]?\d{4}\b`
- Social Security Numbers: `\b\d{3}[-\s]?\d{2}[-\s]?\d{4}\b`
- JWT tokens: `eyJ[a-zA-Z0-9_-]*\.eyJ[a-zA-Z0-9_-]*\.[a-zA-Z0-9_-]*`

## Exclusions

Do NOT flag:
- Files in `.gitignore`
- Files in `.env.example` or similar example templates
- Placeholder values like `YOUR_API_KEY_HERE`, `<API_KEY>`, `xxx`, `****`
- Test fixtures explicitly marked as mock data
- Documentation examples with obviously fake values

## Output Format

Present findings in a clear, actionable format:

```
## Sensitive Information Scan Results

**Mode:** [All Files / Last N Commits / Staged Files]
**Files Scanned:** X files
**Findings:** Y potential issues

### Findings

| File | Line | Type | Match | Severity |
|------|------|------|-------|----------|
| path/to/file.js | 42 | API Key | `AKIA...` | High |
| config.py | 15 | Password | `password = "..."` | High |

### Recommendations

1. [Specific recommendation for each finding]
2. Consider using environment variables for: [list]
3. Add to .gitignore: [files that should be ignored]

### Quick Fix Commands

```bash
# Remove sensitive file from git history
git filter-branch --force --index-filter \
  'git rm --cached --ignore-unmatch path/to/sensitive/file' \
  --prune-empty --tag-name-filter cat -- --all

# Add to .gitignore
echo "path/to/file" >> .gitignore
```
```

## Best Practices to Suggest

1. Use environment variables for secrets
2. Use `.env` files (add to .gitignore)
3. Use secret management tools (Vault, AWS Secrets Manager, etc.)
4. Add pre-commit hooks with tools like `git-secrets` or `gitleaks`
5. Never commit `.env` files with real credentials
6. Use placeholder values in example files

## Tools Integration

Suggest using these tools for automated scanning:
- `gitleaks` - Fast static analysis for git repos
- `git-secrets` - Prevents committing secrets
- `truffleHog` - Searches through git history
- `detect-secrets` - Baseline-aware secret detection
