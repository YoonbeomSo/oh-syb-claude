---
name: worktree
description: "Git worktree automation for isolated feature development. Triggers on: '/worktree create', '/worktree list', '/worktree remove', '/worktree done', '/worktree switch'. Creates isolated working directories outside the project to avoid .git conflicts."
allowed-tools:
  - Bash(git worktree:*)
  - Bash(git branch:*)
  - Bash(git checkout:*)
  - Bash(git rev-parse:*)
  - Bash(git -C:*)
  - Bash(git log:*)
  - Bash(git status:*)
  - Bash(git add:*)
  - Bash(git commit:*)
  - Bash(git push:*)
  - Bash(pwd:*)
  - Bash(basename:*)
  - Bash(mkdir:*)
  - Bash(cp:*)
  - Bash(rm:*)
  - Bash(ls:*)
  - Bash(df:*)
  - Bash(npm:*)
  - Bash(yarn:*)
  - Bash(pnpm:*)
  - Bash(bun:*)
  - Bash(./gradlew:*)
  - Bash(./mvnw:*)
  - Bash(mvn:*)
  - Bash(pip:*)
  - Bash(poetry:*)
  - Read
  - Write
  - Glob
  - Grep
---

# Worktree Skill

Git worktree automation for isolated feature development. Worktrees are created **outside the project directory** as sibling directories to avoid `.git` conflicts, build tool interference, and `.gitignore` dependency.

## When This Skill Activates

| Category | Trigger Phrases |
|----------|-----------------|
| **Create worktree** | `/worktree create <name>`, `/worktree new <name>` |
| **List worktrees** | `/worktree list`, `/worktree ls` |
| **Remove worktree** | `/worktree remove <name>`, `/worktree rm <name>` |
| **Done (finish work)** | `/worktree done` |
| **Switch worktree** | `/worktree switch <name>` |
| **Status** | `/worktree status` |

## Directory Structure

Worktrees are created as **sibling directories** outside the project:

```
~/projects/
├── shopping-api/                        # Main project (git root)
├── worktrees-shopping-api/              # Worktrees for shopping-api
│   ├── auth-feature/
│   └── hot-deals-api/
├── shopping-curation/                   # Another project
└── worktrees-shopping-curation/
    └── slot-refactor/
```

**Path formula:** `../worktrees-{project-name}/<branch-name>`

Where `project-name` = `basename $(git rev-parse --show-toplevel)`

## Commands

### Create Worktree

```
/worktree create <branch-name>
```

Creates an isolated worktree for feature development.

**Workflow:**
1. Resolve project name: `basename $(git rev-parse --show-toplevel)`
2. Set worktree dir: `../worktrees-{project-name}/<branch-name>`
3. Create parent dir if needed: `mkdir -p ../worktrees-{project-name}`
4. Create worktree: `git worktree add ../worktrees-{project-name}/<branch-name> -b <branch-name>`
5. Copy environment files (if they exist):
   - `.env`, `.env.local`, `.env.development`
   - `.nvmrc`, `.node-version`, `.tool-versions`
   - `.npmrc`
6. Detect and run dependency install (see Build Tool Detection)
7. **Verify worktree creation:**
   - Run `git -C ../worktrees-{project-name}/<name> status` to confirm clean state
   - Run `git worktree list` to confirm worktree is registered
   - If verification fails, run cleanup and report error
8. **`cd` into the new worktree** so subsequent commands run in the worktree context
9. Report path and instructions

**Example:**
```bash
project_name=$(basename $(git rev-parse --show-toplevel))
wt_dir="../worktrees-${project_name}/auth-feature"
mkdir -p "../worktrees-${project_name}"
git worktree add "$wt_dir" -b auth-feature
cp .env "$wt_dir/" 2>/dev/null || true
cp .nvmrc "$wt_dir/" 2>/dev/null || true
# then run detected build tool install in $wt_dir
```

### List Worktrees

```
/worktree list
```

Shows all active worktrees:
```bash
git worktree list
```

### Remove Worktree

```
/worktree remove <name>
```

Removes a worktree and optionally its branch:

**Workflow:**
1. Check for uncommitted changes and unpushed commits — warn user if either exists
2. Confirm removal with user (do NOT force-remove without explicit consent)
3. **If currently inside the worktree being removed**, `cd` to the main worktree first
4. Remove worktree: `git worktree remove ../worktrees-{project-name}/<name>`
   - Only use `--force` if user explicitly confirms after seeing warnings
5. Ask if branch should be deleted
6. If yes: `git branch -D <name>`
7. Prune: `git worktree prune`

### Done (Finish Work)

```
/worktree done
```

Finishes work in the current worktree and returns to the main project.

**Workflow:**
1. Verify current directory is a worktree (not the main worktree)
2. Show `git status` — if uncommitted changes exist, ask user what to do:
   - Commit and push
   - Leave as-is (just go back)
3. Show `git log` of unpushed commits — if any, ask user if they want to push
4. Resolve the main worktree path: `git worktree list --porcelain | head -1 | sed 's/worktree //'`
5. `cd` to the main worktree
6. Ask user if they want to remove the worktree they just left

### Switch Worktree

```
/worktree switch <name>
```

Switches to another worktree.

**Workflow:**
1. Run `git worktree list` to find the path matching `<name>`
2. If not found, show available worktrees and ask user to pick
3. `cd` to the matched worktree path

### Status

```
/worktree status
```

Shows status of current worktree:
- Current branch
- Uncommitted changes
- Relationship to main worktree

## Build Tool Detection

Detect the project's build system and run the appropriate install command in the new worktree. Check in this order:

| File | Build Tool | Install Command |
|------|-----------|----------------|
| `build.gradle.kts` or `build.gradle` | Gradle | `./gradlew build` (or skip if slow; inform user) |
| `pom.xml` | Maven | `./mvnw install -DskipTests` or `mvn install -DskipTests` |
| `package-lock.json` | npm | `npm install` |
| `yarn.lock` | Yarn | `yarn install` |
| `pnpm-lock.yaml` | pnpm | `pnpm install` |
| `bun.lockb` | Bun | `bun install` |
| `requirements.txt` | pip | `pip install -r requirements.txt` |
| `pyproject.toml` | Poetry/pip | `poetry install` or `pip install -e .` |
| `Gemfile` | Bundler | `bundle install` |
| `go.mod` | Go | `go mod download` |

**Note:** For Gradle/Maven projects, dependency install may take significant time. Inform the user and ask if they want to run it now or skip.

## Environment Files

These files are copied to new worktrees if they exist:

| File | Purpose |
|------|---------|
| `.env` | Environment variables |
| `.env.local` | Local overrides |
| `.env.development` | Development config |
| `.nvmrc` | Node version |
| `.node-version` | Node version (alternative) |
| `.npmrc` | npm configuration |
| `.tool-versions` | asdf version manager |

## Behavior Rules

### MUST DO
- Create worktrees under `../worktrees-{project-name}/` (outside the project)
- Resolve project name via `basename $(git rev-parse --show-toplevel)`
- Copy environment files to new worktrees
- Detect and offer to run dependency install
- **`cd` into the new worktree after creation** so the session continues in the worktree context
- **`cd` back to the main worktree** on `done`, `remove` (if inside the target), or `switch`
- Resolve the main worktree path via: `git worktree list --porcelain | head -1 | sed 's/worktree //'`
- Confirm before removing worktrees
- Check for uncommitted changes and unpushed commits before removal
- Use the user-provided name directly as the branch name (no prefix transformation)

### MUST NOT
- Create worktrees inside the project directory
- Add anything to `.gitignore` (worktrees are outside the project, no need)
- Force delete without user confirmation
- Leave orphaned worktrees
- Add `feat/` or any other prefix to branch names unless the user includes it

## Safety

**Before removal, check:**
- No uncommitted changes in worktree
- No unpushed commits on branch
- Warn user if either condition exists
- Only use `--force` after explicit user confirmation

**Recovery:**
- Worktrees can be recovered with `git worktree add` if directory deleted manually
- Branches are not deleted unless explicitly requested
- Run `git worktree prune` after manual directory deletions

## Decision Matrix: Worktree vs Branch

| Situation | Use Worktree | Use Branch |
|-----------|--------------|------------|
| Working on two features simultaneously | YES | NO |
| Quick hotfix while mid-feature | YES | NO |
| Sequential feature development | NO | YES |
| Code review while continuing work | YES | NO |
| Simple branch for later work | NO | YES |
| Different dependencies needed | YES | NO |
| Testing against different deps | YES | NO |
| Just want to save work in progress | NO | YES (stash or branch) |

**Rule of thumb:**
- **Parallel work** = worktree (isolated directories, no context switching)
- **Sequential work** = branch (single directory, normal git workflow)
- **Different dependencies** = worktree (each has own node_modules / build output)

## Example Sessions

### Create Worktree - Complete Session

```
User: /worktree create auth-refactor

Claude: Creating worktree for auth-refactor...
```

```bash
# Step 1: Resolve project info
$ git rev-parse --is-inside-work-tree
true

$ project_name=$(basename $(git rev-parse --show-toplevel))
$ echo $project_name
myapp

$ git worktree list
/Users/dev/myapp  abc1234 [main]

# Step 2: Check if branch already exists
$ git branch --list 'auth-refactor'
(no output = branch doesn't exist, safe to proceed)

# Step 3: Create worktree directory and worktree
$ mkdir -p "../worktrees-${project_name}"
$ git worktree add "../worktrees-${project_name}/auth-refactor" -b auth-refactor
Preparing worktree (new branch 'auth-refactor')
HEAD is now at abc1234 feat: add user dashboard

# Step 4: Copy environment files
$ cp .env "../worktrees-${project_name}/auth-refactor/" 2>/dev/null && echo "Copied .env" || echo "No .env found"
Copied .env

$ cp .nvmrc "../worktrees-${project_name}/auth-refactor/" 2>/dev/null && echo "Copied .nvmrc" || echo "No .nvmrc found"
Copied .nvmrc

# Step 5: Detect build tool and install
$ ls "../worktrees-${project_name}/auth-refactor/package-lock.json"
../worktrees-myapp/auth-refactor/package-lock.json

$ cd "../worktrees-${project_name}/auth-refactor" && npm install
added 847 packages in 32s

# Step 6: Verify worktree is functional
$ git -C "../worktrees-${project_name}/auth-refactor" status
On branch auth-refactor
nothing to commit, working tree clean

$ git worktree list
/Users/dev/myapp                                    abc1234 [main]
/Users/dev/worktrees-myapp/auth-refactor            abc1234 [auth-refactor]

# Step 7: cd into the new worktree
$ cd "../worktrees-${project_name}/auth-refactor"
```

**Output to user:**
```
Worktree created successfully.

Location: ../worktrees-myapp/auth-refactor  (now working here)
Branch:   auth-refactor
Base:     main (abc1234)

Environment files copied:
  - .env
  - .nvmrc

Dependencies installed via npm.

Ready to work. Current directory is now the worktree.

To open in a separate editor:
  code ../worktrees-myapp/auth-refactor
```

### List Worktrees - Complete Session

```
User: /worktree list

Claude: Current worktrees:
```

```bash
$ git worktree list
/Users/dev/myapp                                abc1234 [main]
/Users/dev/worktrees-myapp/auth-refactor        def5678 [auth-refactor]
/Users/dev/worktrees-myapp/api-cache            ghi9012 [api-cache]
```

**Output to user:**
```
Active Worktrees:

1. main (primary)
   Path:   /Users/dev/myapp
   Commit: abc1234

2. auth-refactor
   Path:   ../worktrees-myapp/auth-refactor
   Branch: auth-refactor
   Commit: def5678

3. api-cache
   Path:   ../worktrees-myapp/api-cache
   Branch: api-cache
   Commit: ghi9012
```

### Remove Worktree - Complete Session (With Warning)

```
User: /worktree remove auth-refactor

Claude: Checking worktree status before removal...
```

```bash
# Step 1: Resolve paths
$ project_name=$(basename $(git rev-parse --show-toplevel))
$ wt_path="../worktrees-${project_name}/auth-refactor"

# Step 2: Check for uncommitted changes
$ git -C "$wt_path" status --porcelain
 M src/auth/login.ts
?? src/auth/temp.ts

# Step 3: Check for unpushed commits
$ git -C "$wt_path" log origin/auth-refactor..HEAD --oneline 2>/dev/null || git -C "$wt_path" log origin/main..HEAD --oneline
a1b2c3d feat(auth): add session validation
d4e5f6g feat(auth): refactor token storage
```

**Output to user:**
```
WARNING: This worktree has uncommitted changes:

Modified:  src/auth/login.ts
Untracked: src/auth/temp.ts

WARNING: This worktree has 2 unpushed commits:

a1b2c3d feat(auth): add session validation
d4e5f6g feat(auth): refactor token storage

Are you sure you want to remove this worktree?
This will DELETE all uncommitted changes permanently.

Options:
1. Commit and push changes first, then remove
2. Force remove (loses uncommitted changes)
3. Cancel
```

```
User: force remove, I don't need those changes

Claude: Removing worktree...
```

```bash
# Step 4: Force remove (user explicitly confirmed)
$ git worktree remove "$wt_path" --force
```

**Output to user:**
```
Worktree removed.

The branch 'auth-refactor' still exists with 2 commits.
Delete the branch as well? (y/n)
```

```
User: yes

Claude: Deleting branch...
```

```bash
$ git branch -D auth-refactor
Deleted branch auth-refactor (was a1b2c3d).

$ git worktree prune

$ git worktree list
/Users/dev/myapp                            abc1234 [main]
/Users/dev/worktrees-myapp/api-cache        ghi9012 [api-cache]
```

**Output to user:**
```
Cleanup complete.
- Worktree directory removed
- Branch auth-refactor deleted
- Worktree references pruned
```

### Done - Complete Session

```
User: /worktree done

Claude: Checking current worktree status...
```

```bash
# Step 1: Confirm we're in a worktree (not the main repo)
$ git rev-parse --is-inside-work-tree
true

$ git worktree list --porcelain | head -1 | sed 's/worktree //'
/Users/dev/myapp

$ pwd
/Users/dev/worktrees-myapp/auth-refactor
# pwd != main worktree → we're in a secondary worktree, good to proceed

# Step 2: Check uncommitted changes
$ git status --porcelain
 M src/auth/session.ts

# Step 3: Check unpushed commits
$ git log origin/auth-refactor..HEAD --oneline 2>/dev/null
a1b2c3d feat(auth): add session validation
```

**Output to user:**
```
Current worktree: auth-refactor

Uncommitted changes:
  Modified: src/auth/session.ts

Unpushed commits (1):
  a1b2c3d feat(auth): add session validation

What would you like to do?
1. Commit, push, then go back to main project
2. Push existing commits only, then go back
3. Just go back (leave changes as-is)
```

```
User: 1

Claude: Committing and pushing...
```

```bash
$ git add -A && git commit -m "feat(auth): finalize session logic"
$ git push -u origin auth-refactor

# Step 4: cd back to main worktree
$ cd /Users/dev/myapp
```

**Output to user:**
```
Pushed to origin/auth-refactor.
Back in main project: /Users/dev/myapp

Remove the worktree 'auth-refactor'? (y/n)
```

### Switch Worktree - Complete Session

```
User: /worktree switch api-cache

Claude: Switching to api-cache...
```

```bash
$ git worktree list
/Users/dev/myapp                                abc1234 [main]
/Users/dev/worktrees-myapp/auth-refactor        def5678 [auth-refactor]
/Users/dev/worktrees-myapp/api-cache            ghi9012 [api-cache]

$ cd /Users/dev/worktrees-myapp/api-cache
```

**Output to user:**
```
Switched to worktree: api-cache
Branch: api-cache
Path:   /Users/dev/worktrees-myapp/api-cache
```

### Remove Worktree While Inside It - Complete Session

```
User: /worktree remove auth-refactor
(user is currently inside ../worktrees-myapp/auth-refactor)

Claude: You're currently inside this worktree. Checking status...
```

```bash
# Step 1: Detect we're inside the target worktree
$ pwd
/Users/dev/worktrees-myapp/auth-refactor

# Step 2: Check for uncommitted changes (same as normal remove)
$ git status --porcelain
(clean)

# Step 3: cd to main worktree FIRST
$ main_wt=$(git worktree list --porcelain | head -1 | sed 's/worktree //')
$ cd "$main_wt"

# Step 4: Now remove
$ git worktree remove "../worktrees-myapp/auth-refactor"
$ git worktree prune
```

**Output to user:**
```
Moved to main project: /Users/dev/myapp
Worktree 'auth-refactor' removed.

Delete branch 'auth-refactor' as well? (y/n)
```

## Error Handling

### Error: Branch Already Exists

```bash
$ git worktree add "../worktrees-myapp/auth-feature" -b auth-feature
fatal: a branch named 'auth-feature' already exists
```

**Recovery:**
```bash
# Option 1: Use existing branch (if no worktree already uses it)
$ git worktree add "../worktrees-myapp/auth-feature" auth-feature

# Option 2: Choose a different name
$ git worktree add "../worktrees-myapp/auth-feature-v2" -b auth-feature-v2

# Option 3: Delete the old branch first (safe delete, fails if unmerged)
$ git branch -d auth-feature
$ git worktree add "../worktrees-myapp/auth-feature" -b auth-feature
```

**User message:** "Branch 'auth-feature' already exists. Would you like to: (1) use the existing branch, (2) choose a different name, or (3) delete the old branch first?"

### Error: Worktree Directory Already Exists

```bash
$ git worktree add "../worktrees-myapp/auth-feature" -b auth-feature
fatal: '../worktrees-myapp/auth-feature' already exists
```

**Recovery:**
```bash
# Check if it's a valid worktree
$ git worktree list | grep auth-feature

# If listed: worktree exists, inform user
# If not listed: orphaned directory — confirm with user before removing
```

**User message:** "Directory '../worktrees-myapp/auth-feature' already exists. It appears to be an orphaned directory (not a valid worktree). Remove it and retry?"

Only proceed with removal after user confirms.

### Error: Dependency Install Fails

```bash
$ cd ../worktrees-myapp/auth-feature && npm install
npm ERR! code ERESOLVE
```

**Recovery:** Worktree is created successfully — only the install failed.

**User message:** "Worktree created successfully, but dependency install failed. The worktree is ready at `../worktrees-myapp/auth-feature`. You may need to resolve the dependency issue manually."

### Error: Disk Space Insufficient

```bash
npm ERR! code ENOSPC
```

**Recovery:**
```bash
# Clean up the partial worktree — confirm with user first
$ git worktree remove "../worktrees-myapp/auth-feature" --force
$ git branch -D auth-feature
$ git worktree prune
$ df -h .
```

**User message:** "Disk space insufficient. Would you like to clean up the partial worktree?"

### Error: Invalid Worktree Name

**Prevention:** Validate names before attempting creation.

```bash
# Valid: alphanumeric, hyphens, underscores, slashes
# Invalid: spaces, special chars, starting with hyphen
name="my feature"
sanitized=$(echo "$name" | tr ' ' '-' | tr -cd 'a-zA-Z0-9-_/')
# Result: my-feature
```

**User message:** "Invalid name 'my feature'. Branch names cannot contain spaces. Using 'my-feature' instead."

### Error: Worktree Locked

```bash
$ git worktree remove "../worktrees-myapp/auth-feature"
fatal: '../worktrees-myapp/auth-feature' is locked
```

**Recovery:**
```bash
$ git worktree unlock "../worktrees-myapp/auth-feature"
$ git worktree remove "../worktrees-myapp/auth-feature"
```

**User message:** "Worktree is locked (possibly by another process or editor). Close any editors with files from this worktree open, then retry."

## Cleanup Instructions for Partial Failures

If worktree creation fails partway through, clean up in reverse order.
**Always confirm with the user before running cleanup.**

```bash
# 1. Remove the worktree via git (safest)
$ git worktree remove "../worktrees-{project-name}/<name>" --force 2>/dev/null

# 2. If git remove failed, remove directory manually (after user confirmation)
$ rm -rf "../worktrees-{project-name}/<name>"

# 3. Prune worktree references
$ git worktree prune

# 4. Delete the branch if it was created
$ git branch -D <name> 2>/dev/null

# 5. Verify clean state
$ git worktree list
$ git branch --list '<name>'
```

---

**Note:** This skill manages worktrees only. Use standard git commands for commits, pushes, and merges within worktrees.
