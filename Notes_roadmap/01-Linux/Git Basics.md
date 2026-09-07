## 1. Concept

**Git** is a distributed version-control system used to track changes in files and manage code/configuration history.

For SREs, Git is especially important because infrastructure code, Kubernetes manifests, Terraform, CI/CD configuration, monitoring rules, and application configuration are commonly stored in Git.

---

## 2. Definition

Git tracks changes through four main areas:

```text
Working Directory
       ↓ git add
Staging Area
       ↓ git commit
Local Repository
       ↓ git push
Remote Repository
```

### Working Directory

Files you are currently editing.

### Staging Area

Changes selected for the next commit.

### Local Repository

Committed history stored on your machine.

### Remote Repository

Shared repository such as GitHub/GitLab/Bitbucket.

---

# 3. Basic Commands

## Check repository status

```bash
git status
```

Shows:

- modified files
    
- untracked files
    
- staged files
    
- current branch
    

**SRE habit:** When you're confused about Git, run `git status` first.

---

## See changes

```bash
git diff
```

Shows unstaged changes.

For one file:

```bash
git diff deployment.yaml
```

---

## Stage changes

```bash
git add file.txt
```

Stage a specific file.

```bash
git add .
```

Stage all changes in the current directory.

**Production habit:** Prefer staging specific files when possible instead of blindly using `git add .`.

---

## Commit changes

```bash
git commit -m "Update application configuration"
```

Creates a commit containing staged changes.

---

## View history

```bash
git log
```

Compact history:

```bash
git log --oneline
```

Example:

```text
5dccaee Increase application replicas
770ff05 Add application config
```

---

## View a commit

```bash
git show <commit-id>
```

Example:

```bash
git show 5dccaee
```

Shows:

- commit information
    
- author
    
- commit message
    
- exact changes
    

---

# 4. Branches

A branch is an independent line of development.

Create and switch to a branch:

```bash
git switch -c feature-x
```

Switch to an existing branch:

```bash
git switch main
```

List branches:

```bash
git branch
```

Example:

```text
* main
  feature-x
  testing
```

`*` indicates the current branch.

---

# 5. Merge

Merge brings changes from one branch into another.

Example:

```bash
git switch main
git merge feature-x
```

If there are no conflicting changes, Git may perform a **fast-forward merge**.

```text
main
  ↓
A → B → C

feature-x
        ↓
        D
```

After merging:

```text
A → B → C → D
```

---

# 6. Merge Conflicts

A conflict occurs when Git cannot automatically determine which change should be kept.

Example:

```yaml
replicas: 5
<<<<<<< HEAD
timeout: 40s
=======
timeout: 60s
>>>>>>> feature-x
```

Meaning:

```text
<<<<<<< HEAD
Current branch's version
=======
Incoming branch's version
>>>>>>> feature-x
```

### Resolve a conflict

1. Open the file.
    
2. Decide the correct configuration.
    
3. Remove the conflict markers.
    
4. Check the result.
    
5. Stage the resolved file.
    
6. Commit.
    

Example:

```yaml
replicas: 5
timeout: 60s
```

Then:

```bash
git diff
git add app-config.yaml
git commit -m "Resolve configuration merge conflict"
```

**Important:** Never blindly choose "ours" or "theirs" for production infrastructure. Validate the correct configuration first.

---

# 7. Remote Repository

## Clone

Create a local copy of a remote repository:

```bash
git clone <repository-url>
```

---

## Fetch

Download changes from the remote repository without integrating them into your current branch.

```bash
git fetch origin
```

Think:

```text
Remote
  ↓
Local knowledge of remote changes
```

Your current branch is not automatically changed.

---

## Pull

Fetch remote changes and integrate them into your current branch.

```bash
git pull
```

Think:

```text
git pull ≈ git fetch + integration
```

---

## Push

Send local commits to the remote repository.

```bash
git push
```

Example:

```text
Local repository
      ↓ git push
Remote repository
```

---

# 8. Revert vs Reset

This is important for SRE work.

## git revert

Creates a new commit that reverses an earlier commit.

```bash
git revert <commit-id>
```

Example:

```bash
git revert abc123
```

History remains intact:

```text
A → B → C → D
          ↓
         revert
          ↓
A → B → C → D → R
```

### Use when:

- Commit is already pushed.
    
- Branch is shared.
    
- Production was affected.
    
- You need a safe rollback.
    

**Preferred approach for shared/production branches.**

---

## git reset

Moves the branch pointer backward.

Example:

```bash
git reset --hard <commit-id>
```

Potentially removes commits from the current branch history.

### Reset modes

```bash
git reset --soft <commit>
```

Move HEAD but keep changes staged.

```bash
git reset <commit>
```

Move HEAD and keep changes unstaged.

```bash
git reset --hard <commit>
```

Move HEAD and discard changes.

**Be careful with `--hard`.**

For shared production branches, don't use reset casually.

---

# 9. .gitignore

`.gitignore` tells Git which files/directories should not be tracked.

Example:

```text
.env
*.log
*.tmp
.terraform/
terraform.tfstate
terraform.tfstate.backup
*.tfvars
.DS_Store
```

Check why a file is ignored:

```bash
git check-ignore -v .env
```

Example:

```text
.gitignore:1:.env    .env
```

### Important

`.gitignore` is **not a security mechanism**.

If a secret has already been committed:

```text
.gitignore
    ↓
does NOT remove the secret from Git history
```

You need to rotate the credential and take appropriate history-cleanup measures.

---

# 10. Git Commands for SRE Troubleshooting

These are the commands I would consider particularly important for an SRE.

### Find recent changes

```bash
git log --oneline -10
```

### Inspect a suspicious commit

```bash
git show <commit-id>
```

### Compare two versions

```bash
git diff <old-commit> <new-commit>
```

### Check current changes

```bash
git status
git diff
```

### Safely undo a shared commit

```bash
git revert <commit-id>
```

### Update knowledge of remote changes

```bash
git fetch origin
```

---

# 11. SRE Relevance

Git is not just a developer tool for an SRE.

It is commonly used for:

- Kubernetes manifests
    
- Terraform
    
- Ansible
    
- Helm charts
    
- CI/CD pipelines
    
- Monitoring configuration
    
- Alerting rules
    
- Application configuration
    
- Infrastructure documentation
    

A typical production workflow:

```text
Clone
  ↓
Create branch
  ↓
Modify infrastructure/config
  ↓
git diff
  ↓
git status
  ↓
git add
  ↓
git commit
  ↓
git push
  ↓
Pull Request
  ↓
CI/CD validation
  ↓
Code review
  ↓
Merge
  ↓
Deployment
```

---

# 12. Real-World SRE Example

Imagine production suddenly starts returning:

```text
503 errors
0.1% → 15%
```

Five minutes earlier, a deployment changed the application configuration.

First:

```bash
git log --oneline -10
```

Find the recent deployment/configuration commit.

Then:

```bash
git show <commit-id>
```

Check exactly what changed.

Compare versions if necessary:

```bash
git diff <previous-commit> <new-commit>
```

Then correlate the Git change with:

- Kubernetes pod health
    
- CPU/memory
    
- application logs
    
- request latency
    
- load-balancer errors
    
- dependency health
    

If the change is confirmed as the cause:

```bash
git revert <bad-commit>
```

Then allow the normal CI/CD pipeline to deploy the rollback and monitor recovery.

### SRE principle

**Don't revert a commit merely because it happened shortly before an incident. Correlate the change with the failure first.**

---

# 13. Quick Revision

```text
git status
→ What is happening?

git diff
→ What exactly changed?

git add
→ Put changes into staging.

git commit
→ Save staged changes to local history.

git log
→ View commit history.

git show
→ Inspect a specific commit.

git switch
→ Change branches.

git merge
→ Combine branch changes.

git fetch
→ Download remote changes without integrating.

git pull
→ Fetch + integrate.

git push
→ Send commits to remote.

git revert
→ Safely undo a shared commit.

git reset
→ Move branch history backward; use carefully.

.gitignore
→ Prevent matching untracked files from being added.

git check-ignore -v
→ Find why a file is ignored.
```

---

# 14. Must-Remember Commands

```bash
git status
git diff
git add <file>
git commit -m "message"
git log --oneline
git show <commit>
git branch
git switch <branch>
git switch -c <new-branch>
git merge <branch>
git fetch origin
git pull
git push
git revert <commit>
git reset --hard <commit>   # DANGEROUS
git check-ignore -v <file>
```

---

# 15. One-Minute Memory Model

```text
EDIT
 ↓
git status
 ↓
git diff
 ↓
git add
 ↓
git commit
 ↓
git push
 ↓
PR / CI
 ↓
MERGE
 ↓
DEPLOY
```

For an incident:

```text
INCIDENT
 ↓
git log
 ↓
git show
 ↓
git diff
 ↓
Check metrics + logs
 ↓
Confirm cause
 ↓
git revert
 ↓
Deploy rollback
 ↓
Monitor recovery
```