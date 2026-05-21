# Day 1: DevOps Overview & Git Basics

> **Duration:** 1 Hour
> **Audience:** Complete beginners — no prior DevOps knowledge
> **Goal:** Students understand what DevOps is and can use basic Git commands

---

## Lesson Plan (Time Breakdown)

| Time | Duration | Topic |
|------|----------|-------|
| 0:00 | 15 min | What is DevOps? (Theory + Real-World Example) |
| 0:15 | 5 min | Git — Why Version Control? |
| 0:20 | 10 min | Git Setup & First Repository (Live Demo) |
| 0:30 | 15 min | Core Git Commands — Hands-On Practice |
| 0:45 | 10 min | Branching & Merging (Demo) |
| 0:55 | 5 min | Recap + Homework |

---

## Part 1: What is DevOps? (15 min)

### Start with a Story (2 min)

> *"Imagine you are building a mobile app. You have:*
> - *Developers who WRITE the code*
> - *Operations team who DEPLOYS and RUNS the app on servers*
>
> *In the old world, developers throw their code over the wall to operations:*
> *'Here, deploy this.' Operations says: 'It doesn't work on our servers!'*
> *Developer says: 'But it works on my machine!'*
>
> ***DevOps solves this problem.***"

### What is DevOps? (3 min)

**Write on board:**

```
DevOps = Development + Operations

It is a CULTURE and SET OF PRACTICES that brings
developers and operations teams together.

Goal: Deliver software FASTER, more RELIABLY, and with FEWER bugs.
```

**Simple Definition for Students:**

> "DevOps means automating everything — from writing code to putting it on the internet —
> so that software is delivered quickly, tested automatically, and runs without problems."

### Without DevOps vs With DevOps (3 min)

**Draw on board:**

```
WITHOUT DevOps:                        WITH DevOps:
─────────────────                      ────────────────
Developer writes code                  Developer writes code
     ↓ (waits days)                         ↓ (automatic)
Manually test                          Auto test in seconds
     ↓ (waits days)                         ↓ (automatic)
Send to ops team                       Auto build & package
     ↓ (waits days)                         ↓ (automatic)
Ops manually deploys                   Auto deploy to server
     ↓                                      ↓
Something breaks → blame game          Monitoring catches issues
Fix takes days                         Fix in minutes

Release: Once a month                  Release: Multiple times a day
```

### The DevOps Lifecycle (5 min)

**Draw the infinity loop on board:**

```
    Plan → Code → Build → Test
     ↑                      ↓
  Monitor ← Operate ← Release ← Deploy
```

**Explain each step simply:**

| Step | What Happens | Tool Example |
|------|-------------|--------------|
| **Plan** | Decide what to build | Jira, Trello |
| **Code** | Write the application | VS Code, Git |
| **Build** | Compile / Package the app | Docker |
| **Test** | Check if code works | Automated tests |
| **Release** | Prepare for deployment | CI/CD Pipeline |
| **Deploy** | Put on server | Kubernetes, AWS |
| **Operate** | Keep it running | Cloud servers |
| **Monitor** | Watch for problems | Grafana, Prometheus |

### DevOps Tools Overview (2 min)

**Show this quick visual:**

```
 Version Control:  Git, GitHub
 Containers:       Docker
 CI/CD:            Jenkins, GitHub Actions
 Cloud:            AWS, Azure, GCP
 IaC:              Terraform, Ansible
 Orchestration:    Kubernetes
 Monitoring:       Prometheus, Grafana
```

> *"Don't worry about remembering all tools. We will learn each one step by step.
> Today we start with the MOST IMPORTANT one — Git."*

---

## Part 2: Why Git? (5 min)

### The Problem Without Version Control (2 min)

**Ask students:**

> *"How many of you have done this?"*

```
project/
├── app.py
├── app_v2.py
├── app_v2_final.py
├── app_v2_final_REAL.py
├── app_v2_final_REAL_FIXED.py
└── app_backup_do_not_delete.py
```

> *"This is chaos. You don't know which version works. If you break something,
> you can't go back. If two people edit the same file, someone's work is lost."*

### What Git Solves (3 min)

**Write on board:**

```
Git = Time Machine for your code

With Git you can:
✓ Save snapshots of your code (commits)
✓ Go back to any previous version
✓ Work on new features without breaking the main code (branches)
✓ Multiple people can work on the same project
✓ See WHO changed WHAT and WHEN
```

> *"Git is used by EVERY software company in the world. If you learn only one
> tool from this entire course — make it Git."*

---

## Part 3: Git Setup & First Repository (10 min)

### Step 1: Install Git (2 min)

> *Students should have this pre-installed. If not:*

```bash
# Check if installed
git --version

# Install (Ubuntu)
sudo apt install git

# Install (Mac)
brew install git

# Windows: Download from git-scm.com
```

### Step 2: Configure Git (2 min)

**Type along with students:**

```bash
# Tell Git who you are
git config --global user.name "Your Name"
git config --global user.email "your.email@example.com"

# Verify
git config --list
```

> *"This is a one-time setup. Git attaches your name to every change you make."*

### Step 3: Create First Repository (6 min)

**LIVE DEMO — students follow along:**

```bash
# Create a project folder
mkdir my-first-project
cd my-first-project

# Initialize Git (this makes it a Git repository)
git init
```

> *"See that message? Git created a hidden `.git` folder. This folder stores
> ALL the history of your project. Never delete it."*

```bash
# Create a file
echo "Hello DevOps!" > hello.txt

# Check status — Git tells you what changed
git status
```

**Explain the output:**

```
On branch master
Untracked files:
  hello.txt      ← Git sees a new file but is NOT tracking it yet
```

```bash
# Tell Git to track this file (staging)
git add hello.txt

# Check status again
git status
```

```
Changes to be committed:
  new file: hello.txt    ← Ready to be saved (committed)
```

```bash
# Save a snapshot (commit)
git commit -m "Add hello.txt — my first commit"
```

**Explain:**

```
Think of it like this:

  1. You EDIT files          → Working Directory
  2. You SELECT files        → git add (Staging Area)
  3. You SAVE a snapshot     → git commit (Repository)

  Edit → Stage → Commit
```

---

## Part 4: Core Git Commands — Hands-On (15 min)

### Exercise: Students Practice Together

> *"Now you try. Follow me step by step."*

#### Exercise 1: Make Changes and Commit (5 min)

```bash
# Edit the file (add a second line)
echo "I am learning Git today" >> hello.txt

# See what changed
git diff
```

**Explain git diff output:**

```
+I am learning Git today     ← Green = new line added
```

```bash
# Stage and commit
git add hello.txt
git commit -m "Add learning message to hello.txt"

# View history
git log
```

**Explain git log:**

```
commit abc1234 (HEAD -> master)
Author: Your Name
Date:   Wed May 21 2026

    Add learning message to hello.txt

commit def5678
Author: Your Name
Date:   Wed May 21 2026

    Add hello.txt — my first commit
```

> *"See? Git remembers EVERY change. You can always go back."*

```bash
# Compact history view
git log --oneline
```

```
abc1234 Add learning message to hello.txt
def5678 Add hello.txt — my first commit
```

#### Exercise 2: Add More Files (5 min)

```bash
# Create more files
echo "name = 'DevOps Student'" > app.py
echo "node_modules/" > .gitignore

# Stage ALL files at once
git add .

# Check what's staged
git status

# Commit
git commit -m "Add Python app and gitignore"

# View all commits
git log --oneline
```

**Output:**

```
hij7890 Add Python app and gitignore
abc1234 Add learning message to hello.txt
def5678 Add hello.txt — my first commit
```

#### Exercise 3: Undo a Mistake (5 min)

```bash
# Oops — let's say you accidentally deleted content
echo "WRONG CONTENT" > hello.txt
cat hello.txt
# Output: WRONG CONTENT

# Check what changed
git diff

# UNDO — restore the file to last committed version
git restore hello.txt
cat hello.txt
# Output: Hello DevOps!
#         I am learning Git today

# Crisis averted!
```

> *"This is the power of Git. You can ALWAYS go back to any saved version."*

---

## Part 5: Branching & Merging (10 min)

### Explain Branches (3 min)

**Draw on board:**

```
master:  ──A──B──C──
                │
feature:        └──D──E
                      │
                      └── merge back to master

A, B, C = existing commits
D, E    = new feature work (doesn't affect master)
```

> *"A branch is like a parallel universe for your code.
> You can experiment without breaking the main version.
> When your experiment works — merge it back."*

### Live Demo (7 min)

```bash
# See current branch
git branch
# Output: * master

# Create a new branch and switch to it
git checkout -b feature-greeting

# Verify
git branch
# Output:   master
#         * feature-greeting    ← you are here
```

```bash
# Make changes on the feature branch
echo "print('Welcome to DevOps!')" >> app.py

git add app.py
git commit -m "Add welcome greeting to app"
```

```bash
# Switch back to master
git checkout master

# Check app.py — the greeting is NOT here!
cat app.py
# Output: name = 'DevOps Student'
```

> *"See? Master is unchanged. Our new code is safely on the feature branch."*

```bash
# Now merge the feature into master
git merge feature-greeting

# Check app.py — now it has the greeting!
cat app.py
# Output: name = 'DevOps Student'
#         print('Welcome to DevOps!')

# View the history
git log --oneline --graph
```

```bash
# Clean up — delete the feature branch
git branch -d feature-greeting
```

> *"This is how real teams work:*
> 1. *Create a branch for your task*
> 2. *Do your work*
> 3. *Merge back when done*
> 4. *Delete the branch"*

---

## Part 6: Recap & Homework (5 min)

### Quick Recap — Ask Students

> *"Who can tell me..."*

| Question | Answer |
|----------|--------|
| What is DevOps? | Culture + practices to deliver software faster |
| What does `git init` do? | Creates a new Git repository |
| What does `git add` do? | Stages files for commit |
| What does `git commit` do? | Saves a snapshot of staged files |
| What does `git branch` do? | Creates a parallel version of code |
| How do you undo a change? | `git restore filename` |

### Git Commands Cheat Sheet (Give to Students)

```
┌────────────────────────────────────────────────────────┐
│              GIT CHEAT SHEET — Day 1                    │
├────────────────────────────────────────────────────────┤
│                                                         │
│  SETUP:                                                 │
│    git config --global user.name "Name"                │
│    git config --global user.email "email"              │
│                                                         │
│  CREATE:                                                │
│    git init                  → New repository           │
│                                                         │
│  DAILY WORKFLOW:                                        │
│    git status                → See what changed         │
│    git add <file>            → Stage a file             │
│    git add .                 → Stage everything         │
│    git commit -m "message"   → Save snapshot            │
│    git log --oneline         → View history             │
│    git diff                  → See unsaved changes      │
│                                                         │
│  UNDO:                                                  │
│    git restore <file>        → Undo file changes        │
│                                                         │
│  BRANCHES:                                              │
│    git branch                → List branches            │
│    git checkout -b <name>    → Create & switch branch   │
│    git checkout <name>       → Switch to branch         │
│    git merge <name>          → Merge branch             │
│    git branch -d <name>      → Delete branch            │
│                                                         │
│  REMEMBER:                                              │
│    Edit → git add → git commit                          │
│    (Change) → (Stage) → (Save)                          │
│                                                         │
└────────────────────────────────────────────────────────┘
```

### Homework Assignment

```
HOMEWORK — Due Next Class
═════════════════════════

1. Create a new folder called "homework-git"
2. Initialize a Git repository
3. Create 3 files:
   - about.txt    → Write your name and what you want to learn
   - skills.txt   → List 3 skills you already have
   - goals.txt    → List 3 goals for this DevOps course
4. Make 3 separate commits (one per file) with meaningful messages
5. Create a branch called "add-hobbies"
6. On that branch, add a file called hobbies.txt
7. Merge it back to master
8. Run "git log --oneline" and take a screenshot

Expected output should look like:
   abc1234 Merge branch 'add-hobbies'
   def5678 Add hobbies file
   ghi9012 Add goals for DevOps course
   jkl3456 Add current skills
   mno7890 Add about me information
```

---

## Teaching Tips

- **Go SLOW** with commands — students are typing for the first time
- **Repeat the workflow**: edit → add → commit (say it 5+ times)
- **Show git status after EVERY step** — students need to see what changed
- **Make mistakes on purpose** — show how Git helps fix them
- After each section ask: *"Is everyone with me? Show me your terminal."*
- Walk around and check student screens during exercises
