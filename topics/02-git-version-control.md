# 2. Version Control with Git

## Table of Contents
- [Git Installation & Setup](#git-installation--setup)
- [Git Basics](#git-basics)
- [Branching & Merging](#branching--merging)
- [Branching Strategies](#branching-strategies)
- [Merging vs Rebasing](#merging-vs-rebasing)
- [Remote Repositories](#remote-repositories)
- [Undoing Changes](#undoing-changes)
- [Git Best Practices](#git-best-practices)
- [Projects](#projects)

---

## Git Installation & Setup

```bash
# Install Git
sudo apt install -y git       # Ubuntu/Debian
sudo yum install -y git       # CentOS/RHEL
brew install git               # macOS

# Verify installation
git --version
# Output: git version 2.43.0

# Configure Git (first-time setup)
git config --global user.name "Your Name"
git config --global user.email "your.email@example.com"
git config --global init.defaultBranch main
git config --global core.editor "nano"     # or "vim" or "code --wait"

# View configuration
git config --list

# Set up SSH for GitHub
ssh-keygen -t ed25519 -C "your.email@example.com"
cat ~/.ssh/id_ed25519.pub
# Copy this output and add to GitHub → Settings → SSH Keys

# Test connection
ssh -T git@github.com
# Output: Hi username! You've successfully authenticated
```

---

## Git Basics

### Initialize & First Commit

```bash
# Create a new project
mkdir my-devops-project && cd my-devops-project

# Initialize Git repository
git init
# Output: Initialized empty Git repository in /home/student/my-devops-project/.git/

# Check status
git status
# Output: On branch main, No commits yet

# Create a file
echo "# My DevOps Project" > README.md

# Stage the file (add to staging area)
git add README.md

# Check status again
git status
# Output: Changes to be committed: new file: README.md

# Commit
git commit -m "Initial commit: add README"
# Output: [main (root-commit) a1b2c3d] Initial commit: add README

# View commit history
git log
git log --oneline          # Compact view
git log --oneline --graph  # With branch graph
```

### The Three Areas of Git

```
┌──────────────┐    git add     ┌──────────────┐   git commit   ┌──────────────┐
│  Working      │ ──────────── │  Staging      │ ─────────────│  Repository  │
│  Directory    │              │  Area         │              │  (.git)      │
│              │              │  (Index)      │              │              │
│  - Edit files │              │  - Prepared   │              │  - Committed │
│  - Untracked  │ ◄─────────── │    for commit │              │    history   │
└──────────────┘  git restore  └──────────────┘              └──────────────┘
```

### Staging & Committing

```bash
# Create multiple files
echo "console.log('hello');" > app.js
echo "body { margin: 0; }" > style.css
echo "node_modules/" > .gitignore

# Stage specific files
git add app.js style.css

# Stage all changes
git add .
git add -A

# Unstage a file
git restore --staged style.css

# Check what's staged vs unstaged
git diff              # Unstaged changes
git diff --staged     # Staged changes (about to be committed)

# Commit with message
git commit -m "Add application files and gitignore"

# Stage and commit in one step (only tracked files)
git commit -am "Update app.js with new feature"
```

### Viewing History

```bash
# Full log
git log

# Compact log
git log --oneline
# Output:
# a1b2c3d (HEAD -> main) Add application files
# e4f5g6h Initial commit: add README

# Log with graph (useful for branches)
git log --oneline --graph --all

# Show specific commit details
git show a1b2c3d

# Show changes in last 3 commits
git log -3 -p

# Search commits by message
git log --grep="fix"

# Show who changed each line
git blame app.js

# Show file at a specific commit
git show a1b2c3d:app.js
```

---

## Branching & Merging

### Branch Basics

```bash
# List branches
git branch            # Local branches
git branch -a         # All branches (including remote)

# Create a new branch
git branch feature-login

# Switch to branch
git checkout feature-login
# or (modern way)
git switch feature-login

# Create and switch in one command
git checkout -b feature-signup
# or
git switch -c feature-signup

# Rename a branch
git branch -m old-name new-name

# Delete a branch
git branch -d feature-login       # Safe delete (must be merged)
git branch -D feature-login       # Force delete
```

### Merging

```bash
# Example workflow:
# 1. Create and switch to feature branch
git switch -c feature-navbar

# 2. Make changes and commit
echo "<nav>Home | About | Contact</nav>" > navbar.html
git add navbar.html
git commit -m "Add navigation bar"

# 3. Switch back to main
git switch main

# 4. Merge feature branch into main
git merge feature-navbar
# Output: Fast-forward merge (if no new commits on main)

# 5. Delete the feature branch
git branch -d feature-navbar
```

### Handling Merge Conflicts

```bash
# Scenario: Two branches modify the same file

# On main branch
echo "Hello from main" > greeting.txt
git add . && git commit -m "Add greeting on main"

# Create and switch to feature branch
git switch -c feature-greeting

# Modify the same file on feature branch
echo "Hello from feature branch" > greeting.txt
git add . && git commit -m "Update greeting on feature"

# Switch back to main and make another change
git switch main
echo "Hello from main - updated" > greeting.txt
git add . && git commit -m "Update greeting on main"

# Try to merge - CONFLICT!
git merge feature-greeting
# Output: CONFLICT (content): Merge conflict in greeting.txt

# View conflict markers in file
cat greeting.txt
# <<<<<<< HEAD
# Hello from main - updated
# =======
# Hello from feature branch
# >>>>>>> feature-greeting

# Resolve: Edit the file, keep what you want
echo "Hello from both main and feature" > greeting.txt

# Mark as resolved and commit
git add greeting.txt
git commit -m "Resolve merge conflict in greeting.txt"
```

---

## Branching Strategies

### GitFlow

```
main (production-ready)
  │
  ├── develop (integration branch)
  │     │
  │     ├── feature/login ──────── merge back to develop
  │     ├── feature/signup ─────── merge back to develop
  │     │
  │     └── release/v1.0 ──┬────── merge to main (tag v1.0)
  │                         └────── merge to develop
  │
  └── hotfix/critical-bug ──┬────── merge to main (tag v1.0.1)
                             └────── merge to develop

Rules:
- main: always production-ready, only merges from release/hotfix
- develop: integration branch, features merge here
- feature/*: branch from develop, merge back to develop
- release/*: branch from develop, merge to main + develop
- hotfix/*: branch from main, merge to main + develop
```

### GitHub Flow (Simpler)

```
main (always deployable)
  │
  ├── feature-login ──── PR ──── Code Review ──── Merge to main ──── Deploy
  │
  ├── fix-navbar ──────── PR ──── Code Review ──── Merge to main ──── Deploy
  │
  └── feature-api ─────── PR ──── Code Review ──── Merge to main ──── Deploy

Rules:
- main is always deployable
- Create feature branches from main
- Open Pull Request for review
- Merge to main after approval
- Deploy immediately after merge
```

---

## Merging vs Rebasing

```bash
# MERGE: Creates a merge commit, preserves history
git switch main
git merge feature-branch
# Result: A new merge commit connecting both histories

# REBASE: Replays commits on top of another branch, linear history
git switch feature-branch
git rebase main
# Result: Feature commits moved to tip of main (no merge commit)

# Interactive rebase (clean up commits before merging)
git rebase -i HEAD~3
# Opens editor to squash, reorder, or edit last 3 commits
# pick   a1b2c3d Add login form
# squash e4f5g6h Fix typo in login
# squash h7i8j9k Add validation
# → Combines into single clean commit
```

```
MERGE:                          REBASE:
  main: A─B─C───M              main: A─B─C
              │ /                          │
  feat: D─E─F─┘               feat:       D'─E'─F'

  (M = merge commit)           (D', E', F' = replayed commits)
```

> **Rule of thumb:** Use merge for shared branches, rebase for local cleanup before merging.

---

## Remote Repositories

### Setting Up Remotes

```bash
# Clone an existing repository
git clone https://github.com/username/repo.git
git clone git@github.com:username/repo.git     # SSH (preferred)

# Add a remote to existing local repo
git remote add origin git@github.com:username/my-project.git

# View remotes
git remote -v
# Output:
# origin  git@github.com:username/my-project.git (fetch)
# origin  git@github.com:username/my-project.git (push)

# Change remote URL
git remote set-url origin git@github.com:username/new-repo.git
```

### Push, Pull, Fetch

```bash
# Push local branch to remote
git push origin main

# Push and set upstream (first time)
git push -u origin main
# After this, you can just use:
git push

# Pull changes from remote (fetch + merge)
git pull origin main

# Fetch only (download without merging)
git fetch origin
git fetch --all       # Fetch from all remotes

# See what changed on remote
git log origin/main --oneline

# Push a new branch to remote
git switch -c feature-api
# ... make commits ...
git push -u origin feature-api
```

### Pull Requests (GitHub Workflow)

```bash
# 1. Fork the repository on GitHub (web UI)

# 2. Clone your fork
git clone git@github.com:YOUR-USERNAME/project.git
cd project

# 3. Add upstream remote (original repo)
git remote add upstream git@github.com:ORIGINAL-OWNER/project.git

# 4. Create a feature branch
git switch -c feature-awesome

# 5. Make changes and commit
echo "new feature" > feature.txt
git add . && git commit -m "Add awesome feature"

# 6. Push to your fork
git push -u origin feature-awesome

# 7. Create Pull Request on GitHub (web UI)
#    base: ORIGINAL-OWNER/project:main ← head: YOUR-USERNAME/project:feature-awesome

# 8. Keep your fork updated
git fetch upstream
git switch main
git merge upstream/main
git push origin main
```

---

## Undoing Changes

```bash
# Discard changes in working directory (not staged)
git restore file.txt
git checkout -- file.txt     # Older syntax

# Unstage a file (keep changes in working directory)
git restore --staged file.txt
git reset HEAD file.txt      # Older syntax

# Undo last commit (keep changes staged)
git reset --soft HEAD~1

# Undo last commit (keep changes in working directory)
git reset HEAD~1

# Undo last commit (discard changes completely)
git reset --hard HEAD~1      # DANGEROUS: changes are lost!

# Revert a commit (creates a new commit that undoes changes)
git revert a1b2c3d           # Safe: doesn't rewrite history

# Recover deleted branch or lost commits
git reflog                   # Shows all HEAD movements
git checkout -b recovered-branch a1b2c3d

# Stash changes temporarily
git stash                    # Save changes
git stash list               # View stashed changes
git stash pop                # Apply and remove latest stash
git stash apply              # Apply but keep in stash
git stash drop               # Remove latest stash
```

---

## Git Best Practices

### Commit Messages

```bash
# Good commit message format:
git commit -m "Add user authentication with JWT tokens

- Implement login/register endpoints
- Add JWT token generation and validation
- Create auth middleware for protected routes
- Add unit tests for auth service"

# Convention: Imperative mood, present tense
# Good:  "Add feature", "Fix bug", "Update docs"
# Bad:   "Added feature", "Fixed bug", "Updating docs"
```

### .gitignore

```bash
# Create .gitignore in project root
cat > .gitignore << 'EOF'
# Dependencies
node_modules/
vendor/
venv/
__pycache__/

# Build outputs
dist/
build/
*.o
*.class

# Environment files
.env
.env.local
*.secret

# IDE files
.vscode/
.idea/
*.swp
*.swo

# OS files
.DS_Store
Thumbs.db

# Logs
*.log
logs/

# Docker
docker-compose.override.yml

# Terraform
.terraform/
*.tfstate
*.tfstate.backup
*.tfvars
EOF
```

### Git Aliases

```bash
# Add useful aliases
git config --global alias.st "status"
git config --global alias.co "checkout"
git config --global alias.br "branch"
git config --global alias.ci "commit"
git config --global alias.lg "log --oneline --graph --all"
git config --global alias.last "log -1 HEAD"

# Usage
git st       # Same as git status
git lg       # Pretty log with graph
git last     # Show last commit
```

---

## Projects

### Project 1: Hello World Git Workflow

```bash
# Step 1: Initialize
mkdir hello-git && cd hello-git
git init

# Step 2: Create initial file
cat > app.py << 'EOF'
def greet(name):
    return f"Hello, {name}!"

if __name__ == "__main__":
    print(greet("World"))
EOF

git add app.py
git commit -m "Initial commit: add greeting app"

# Step 3: Create feature branch
git switch -c feature-farewell

cat >> app.py << 'EOF'

def farewell(name):
    return f"Goodbye, {name}!"

if __name__ == "__main__":
    print(farewell("World"))
EOF

git add app.py
git commit -m "Add farewell function"

# Step 4: Create another feature branch from main
git switch main
git switch -c feature-spanish

cat >> app.py << 'EOF'

def greet_spanish(name):
    return f"Hola, {name}!"
EOF

git add app.py
git commit -m "Add Spanish greeting"

# Step 5: Merge both features
git switch main
git merge feature-farewell
git merge feature-spanish    # This may cause a conflict → resolve it!

# Step 6: View final history
git log --oneline --graph --all
```

### Project 2: Collaborative Git Workflow (Simulated)

```bash
# Simulate two developers working on the same project

# Developer 1 setup
mkdir team-project && cd team-project
git init
echo "# Team Project" > README.md
git add . && git commit -m "Initial commit"

# Developer 1 works on feature A
git switch -c dev1/feature-a
cat > feature_a.py << 'EOF'
class UserAuth:
    def login(self, username, password):
        return {"status": "logged_in", "user": username}

    def logout(self, username):
        return {"status": "logged_out", "user": username}
EOF
git add . && git commit -m "Add user authentication module"

# Developer 2 works on feature B (from main)
git switch main
git switch -c dev2/feature-b
cat > feature_b.py << 'EOF'
class Database:
    def connect(self):
        return "Connected to database"

    def query(self, sql):
        return f"Executing: {sql}"
EOF
git add . && git commit -m "Add database module"

# Both modify README
echo "## Features\n- Auth Module" >> README.md
git add . && git commit -m "Update README with database info"

git switch main
git merge dev1/feature-a

# Developer 2's branch also modified README
git merge dev2/feature-b
# CONFLICT in README.md → resolve it

# After resolving
git add README.md
git commit -m "Merge feature-b: resolve README conflict"

# View the history
git log --oneline --graph --all
```

### Project 3: .gitignore Setup

```bash
mkdir sample-project && cd sample-project
git init

# Create project structure
mkdir -p src config logs node_modules
touch src/app.js config/settings.json .env .env.example
touch logs/app.log node_modules/package.json
echo '{"secret": "abc123"}' > config/secrets.json

# Create comprehensive .gitignore
cat > .gitignore << 'EOF'
# Dependencies
node_modules/

# Environment (secrets)
.env
config/secrets.json

# Logs
logs/
*.log

# OS files
.DS_Store
EOF

# Stage everything
git add .

# Check what will be committed (secrets should NOT be here)
git status
# Output should show:
#   .env.example (good - this is a template)
#   .gitignore
#   src/app.js
#   config/settings.json
# Output should NOT show:
#   .env
#   config/secrets.json
#   node_modules/
#   logs/

git commit -m "Initial project setup with proper .gitignore"

# Verify ignored files
git status --ignored
```

---

## Quick Reference

| Command | Description |
|---------|-------------|
| `git init` | Initialize new repository |
| `git clone url` | Clone a repository |
| `git add .` | Stage all changes |
| `git commit -m "msg"` | Commit staged changes |
| `git push -u origin branch` | Push branch to remote |
| `git pull` | Fetch and merge from remote |
| `git switch -c branch` | Create and switch branch |
| `git merge branch` | Merge branch into current |
| `git log --oneline --graph` | Visual commit history |
| `git stash` | Temporarily save changes |
| `git revert commit` | Undo a commit safely |
| `git diff` | Show unstaged changes |
