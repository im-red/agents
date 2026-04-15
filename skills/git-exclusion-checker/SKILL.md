---
name: "git-exclusion-checker"
description: "Checks for files that should not be managed by git. Invoke when user wants to verify .gitignore coverage or before committing to catch unwanted files."
---

# Git Exclusion Checker

This skill checks whether files that should not be managed by git have been added to the repository. It helps maintain a clean repository by identifying files that should be in `.gitignore`.

## When to Use

- Before committing to verify no unwanted files are staged
- When auditing the repository for files that shouldn't be tracked
- When setting up or reviewing `.gitignore`
- After cloning a project to verify proper exclusions
- When `.gitignore` seems incomplete or missing files

## Working Modes

**Default Mode:** Check Local Changes - Covers both staged and unstaged changes, the most common use case before committing.

### Mode 1: Check Local Changes (Default)

Scans all local changes including both staged and unstaged files. This is the default mode when no specific mode is requested.

```
User request: "check for exclusions"
User request: "check my changes"
User request: "check local changes"
User request: "check staged files" (staged only)
User request: "check unstaged files" (unstaged only)
```

**Execution:**
1. Get list of staged files: `git --no-pager diff --cached --name-only`
2. Get list of unstaged files: `git --no-pager diff --name-only`
3. Combine and check all local changes
4. Report findings grouped by staged/unstaged status

**Note:** User can specify "staged only" or "unstaged only" to limit the scope.

### Mode 2: Check All Tracked Files

Scans all files currently tracked by git to find files that should typically be excluded.

```
User request: "check all tracked files for exclusions"
User request: "what files shouldn't be in git"
User request: "audit my git repo for unwanted files"
User request: "check if any wrong files are tracked"
```

**Execution:**
1. Get list of all tracked files: `git --no-pager ls-files`
2. Check each file against exclusion patterns
3. Report all problematic files with recommendations

## File Categories That Should Be Excluded

### Dependencies & Package Managers

| Pattern | Reason |
|---------|--------|
| `node_modules/` | NPM dependencies - should be installed via `npm install` |
| `vendor/` | Composer/PHP dependencies |
| `packages/` | Package directories |
| `*.lock` (selective) | Some lock files should be committed, others not |
| `yarn.lock` | Should be committed (tracks exact versions) |
| `package-lock.json` | Should be committed (tracks exact versions) |
| `Pods/` | iOS CocoaPods dependencies |
| `.venv/`, `venv/`, `env/` | Python virtual environments |

### Build Outputs & Artifacts

| Pattern | Reason |
|---------|--------|
| `dist/` | Build output directory |
| `build/` | Build output directory |
| `out/` | Build output directory |
| `target/` | Maven/Gradle build output |
| `*.o`, `*.obj` | Object files |
| `*.exe`, `*.dll`, `*.so` | Compiled binaries |
| `*.class` | Java compiled classes |
| `*.pyc`, `__pycache__/` | Python compiled files |
| `.next/` | Next.js build output |
| `.nuxt/` | Nuxt.js build output |

### Environment & Configuration Files

| Pattern | Reason |
|---------|--------|
| `.env` | Environment variables - often contains secrets |
| `.env.local` | Local environment overrides |
| `.env.*.local` | Local environment files |
| `*.pem`, `*.key` | SSL certificates and private keys |
| `secrets.yaml`, `secrets.json` | Secret configuration files |
| `credentials.json` | Credential files |

### IDE & Editor Settings

| Pattern | Reason |
|---------|--------|
| `.idea/` | JetBrains IDE settings |
| `.vscode/` | VS Code settings (debatable - some teams share) |
| `*.swp`, `*.swo` | Vim swap files |
| `.DS_Store` | macOS folder metadata |
| `Thumbs.db` | Windows thumbnail cache |
| `*.sublime-*` | Sublime Text settings |

### Logs & Temporary Files

| Pattern | Reason |
|---------|--------|
| `*.log` | Log files |
| `logs/` | Log directories |
| `*.tmp`, `*.temp` | Temporary files |
| `*.bak` | Backup files |
| `*~` | Backup files (Unix) |
| `.cache/` | Cache directories |

### Database Files

| Pattern | Reason |
|---------|--------|
| `*.db` | SQLite database files |
| `*.sqlite`, `*.sqlite3` | SQLite database files |
| `*.sql` (sometimes) | Database dumps - may contain sensitive data |

### OS Generated Files

| Pattern | Reason |
|---------|--------|
| `.DS_Store` | macOS folder metadata |
| `Thumbs.db` | Windows thumbnail cache |
| `Desktop.ini` | Windows folder settings |
| `$RECYCLE.BIN/` | Windows recycle bin |
| `*.stackdump` | Windows stack dumps |

### Large Binary Files

| Pattern | Reason |
|---------|--------|
| `*.zip`, `*.tar`, `*.gz` | Archives - use releases instead |
| `*.rar`, `*.7z` | Archives |
| `*.iso`, `*.img` | Disk images |
| `*.mp4`, `*.mov`, `*.avi` | Video files (use Git LFS if needed) |
| `*.mp3`, `*.wav` | Audio files (use Git LFS if needed) |
| `*.psd`, `*.ai` | Design files (use Git LFS if needed) |

### Cache & Session Files

| Pattern | Reason |
|---------|--------|
| `.cache/` | Cache directories |
| `*.cache` | Cache files |
| `sessions/` | Session data |
| `*.session` | Session files |

## Files That SHOULD Be Committed

These files are often questioned but should typically be tracked:

| File | Reason |
|------|--------|
| `package-lock.json` | Ensures consistent dependency versions |
| `yarn.lock` | Ensures consistent dependency versions |
| `.gitignore` | Repository exclusion rules |
| `README.md` | Project documentation |
| `LICENSE` | Legal information |
| `.editorconfig` | Shared editor settings |
| `.eslintrc*`, `.prettierrc*` | Code style configuration |
| `tsconfig.json` | TypeScript configuration |
| `docker-compose.yml` | Docker configuration |
| `Dockerfile` | Docker build instructions |

## Output Format

Present findings in a clear, actionable format:

```
## Git Exclusion Check Results

**Mode:** [All Tracked Files / Staged Files]
**Files Checked:** X files
**Issues Found:** Y files that should likely be excluded

### Files That Should Be Excluded

| File | Category | Reason | Severity |
|------|----------|--------|----------|
| node_modules/... | Dependencies | Should be installed via npm | High |
| .env | Environment | May contain secrets | Critical |
| dist/bundle.js | Build Output | Generated file | Medium |

### Recommended .gitignore Additions

```gitignore
# Add these patterns to .gitignore:
node_modules/
.env
dist/
```

### Quick Fix Commands

```bash
# Remove file from git tracking (keep local)
git rm --cached path/to/file

# Remove directory from git tracking
git rm -r --cached path/to/directory/

# Remove from git history (use with caution)
git filter-branch --force --index-filter \
  'git rm --cached --ignore-unmatch path/to/file' \
  --prune-empty --tag-name-filter cat -- --all
```

### Severity Levels

| Level | Description | Action |
|-------|-------------|--------|
| Critical | Contains secrets or sensitive data | Remove immediately, rotate credentials |
| High | Should never be in repo | Remove from tracking |
| Medium | Usually excluded, may have exceptions | Review and decide |
| Low | Often excluded, team preference | Consider adding to .gitignore |
```

## Best Practices

1. **Review .gitignore regularly** - Update as project evolves
2. **Use git rm --cached** - Remove tracked files without deleting locally
3. **Commit .gitignore early** - Set up exclusions at project start
4. **Use templates** - Start with .gitignore templates for your stack
5. **Consider Git LFS** - For large binary files that must be tracked
6. **Never commit secrets** - Use environment variables or secret managers

## Language/Stack Specific Patterns

### Node.js / JavaScript
```
node_modules/
npm-debug.log*
yarn-debug.log*
yarn-error.log*
.npm
.yarn-integrity
```

### Python
```
__pycache__/
*.py[cod]
*$py.class
*.so
.Python
venv/
.venv/
ENV/
*.egg-info/
dist/
build/
```

### Java / JVM
```
target/
*.class
*.jar
*.war
*.ear
.gradle/
build/
```

### .NET / C#
```
bin/
obj/
*.suo
*.user
*.userosscache
*.sln.docstates
packages/
```

### Go
```
*.exe
*.exe~
*.dll
*.so
*.dylib
*.test
*.out
vendor/
```

### Rust
```
target/
Cargo.lock (for libraries)
**/*.rs.bk
```

### Ruby
```
*.gem
*.rbc
.bundle/
.config/
coverage/
InstalledFiles
lib/bundler/man
pkg/
rdoc/
spec/reports/
test/tmp/
test/version_tmp/
tmp/
```

## Tools Integration

Suggest using these tools:
- `git check-ignore -v <file>` - Check why a file is ignored
- `git ls-files -i --exclude-from=.gitignore` - List ignored files
- `gibo` - Generate .gitignore from templates
- `gitignore.io` - Online .gitignore generator
