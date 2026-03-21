# 🌿 Module 3: Git & GitHub — Version Control

> All examples use the **ShopEase** e-commerce microservice project.

---

## 📑 Table of Contents

- [1. Why Version Control?](#1-why-version-control)
- [2. Git Architecture](#2-git-architecture)
- [3. Git Setup](#3-git-setup)
- [4. Essential Git Commands](#4-essential-git-commands)
- [5. Branching Strategy](#5-branching-strategy)
- [6. Pull Requests](#6-pull-requests)
- [7. Git Conflicts](#7-git-conflicts)
- [8. Advanced Concepts](#8-advanced-concepts)
- [9. Real-Time Workflow](#9-real-time-workflow)
- [10. Cheat Sheet](#10-cheat-sheet)

---

## 1. Why Version Control?

| Problem | Git's Solution |
|---|---|
| Multiple devs working remotely | Central repo integrates all code |
| Track who changed what & when | Full commit history with author |
| Accidentally broke something | Revert to any previous version |
| Multiple features in parallel | Branches isolate each work stream |

---

## 2. Git Architecture

```
📂 Working Tree ──(git add)──▶ 📋 Staging Area ──(git commit)──▶ 💾 Local Repo ──(git push)──▶ ☁️ GitHub
     ◀──────────────────────(git pull / git clone)──────────────────────────────────────────────┘
```

### ShopEase Example

```bash
git clone https://github.com/shopease/product-service.git
# Edit ProductController.java
git add src/main/java/com/shopease/product/controller/ProductController.java
git commit -m "feat: add search products by category endpoint"
git push origin develop
```

---

## 3. Git Setup

```bash
# Install: https://git-scm.com/download/win
git config --global user.name "Surya Pandey"
git config --global user.email "surya@shopease.com"
git config --list    # verify
```

---

## 4. Essential Git Commands

| Command | Purpose |
|---|---|
| `git init` | Initialize new repo |
| `git clone <url>` | Download remote repo |
| `git status` | Check staging status |
| `git add .` | Stage all changes |
| `git commit -m "msg"` | Commit staged changes |
| `git push origin <branch>` | Upload to remote |
| `git pull origin <branch>` | Download + merge latest |
| `git log --oneline -10` | Recent commit history |
| `git restore <file>` | Discard working changes |
| `git stash` / `git stash apply` | Save/restore work temporarily |

### `.gitignore` for ShopEase

```gitignore
target/
*.class
*.jar
.idea/
*.iml
.project
.settings/
*.log
.env
.DS_Store
```

---

## 5. Branching Strategy

### ShopEase Branch Model

```
main ─────────────────────────────────────▶ (production)
  │
  └──▶ develop ───────────────────────────▶ (integration)
           │
           ├──▶ feature/product-api ──(PR)──▶ develop
           ├──▶ feature/order-api ───(PR)──▶ develop
           └──▶ bugfix/cart-total ───(PR)──▶ develop
```

| Branch | Purpose |
|---|---|
| `main` | Production-ready, always stable |
| `develop` | Ongoing integration |
| `feature/*` | New features (`feature/product-search`) |
| `bugfix/*` | Bug fixes (`bugfix/cart-total-fix`) |
| `hotfix/*` | Urgent prod fixes (`hotfix/payment-crash`) |
| `release/*` | Pre-production prep (`release/v2.0`) |

### Branch Commands

```bash
git branch                                  # list branches
git checkout -b feature/product-search      # create + switch
git checkout develop                        # switch branch
git clone -b develop <repo-url>             # clone specific branch
```

---

## 6. Pull Requests

> A PR is a request to **merge code from one branch to another** with team review.

### ShopEase PR Flow

```
1. Dev creates feature/product-search branch
2. Writes code, commits, pushes
3. Creates PR: feature/product-search → develop
4. Assigns reviewer (Tech Lead)
5. Reviewer reviews, leaves comments
6. Dev addresses feedback, pushes fixes
7. Reviewer approves → PR merged into develop
```

> **⚠️ Important:** In real projects, you **never push directly to `main` or `develop`**. Always use Pull Requests.

---

## 7. Git Conflicts

Conflicts occur when **two developers modify the same line** in the same file:

```bash
git pull origin develop
# CONFLICT in ProductRepository.java

# File shows:
<<<<<<< HEAD
    List<Product> findByNameContaining(String name);
=======
    List<Product> findByCategoryId(Long categoryId);
>>>>>>> origin/develop

# Resolution: Keep BOTH, remove conflict markers
    List<Product> findByNameContaining(String name);
    List<Product> findByCategoryId(Long categoryId);

git add .
git commit -m "resolve: merge conflict in ProductRepository"
git push
```

---

## 8. Advanced Concepts

### `git fetch` vs `git pull`

| Command | What It Does |
|---|---|
| `git fetch` | Downloads to local repo only (safer) |
| `git pull` | Downloads AND merges into working tree (`git fetch + git merge`) |

### `git merge` vs `git rebase`

| | `git merge` | `git rebase` |
|---|---|---|
| History | Preserves all commits | Rewrites to linear history |
| Use | Merging feature → develop | Keeping feature up-to-date |
| Safety | Safe for shared branches | Avoid on public branches |

### `git stash` — Temporary Storage

```bash
# Working on product search, urgent bug comes in
git stash                        # save current work
git checkout hotfix/urgent-fix   # fix bug
# ... fix, commit, push ...
git checkout feature/product-search
git stash apply                  # restore saved work
```

### `git fork` — Forking means copying another user's repo to your GitHub account (used for open-source contributions).

---

## 9. Real-Time Workflow

```
 👨‍💻 Developer                    👨‍💼 Tech Lead           ⚙️ DevOps Team
     │                              │                      │
     │ ──── Request repo access ────────────────────────▶  │
     │                              │                      │
     │ ◀────── Grant access ────────────────────────────── │
     │                              │                      │
     │── Clone develop branch ─────▶│                      │
     │── Create feature branch ────▶│                      │
     │── Write code, commit, push ─▶│                      │
     │── Create Pull Request ──────▶│                      │
     │                              │── Review code        │
     │ ◀── Request changes ────────│                      │
     │── Fix & push ───────────────▶│                      │
     │                              │── Approve & merge    │
     │                              │                      │
     │              BEFORE PROD RELEASE:                   │
     │                              │                      │
     │                              │ ◀── Code Freeze ──── │
     │                              │    (20-30 days)      │
```

> **💡 Pro Tip:** **Code Freeze** = DevOps disables commit permissions 20-30 days before production release to prevent risky last-minute changes.

---

## 10. Cheat Sheet

```
┌──────────────────────────────────────────────────────────┐
│                   GIT CHEAT SHEET                         │
├──────────────────────────────────────────────────────────┤
│ SETUP                                                     │
│   git config --global user.name "Name"                    │
│   git config --global user.email "email"                  │
│                                                           │
│ BASICS                                                    │
│   git clone <url>          → Download repo                │
│   git add .                → Stage all changes            │
│   git commit -m "msg"      → Commit                       │
│   git push origin <branch> → Push to remote               │
│   git pull origin <branch> → Pull latest                  │
│                                                           │
│ BRANCHING                                                 │
│   git branch               → List branches                │
│   git checkout -b <name>   → Create + switch              │
│   git checkout <name>      → Switch branch                │
│                                                           │
│ UNDO / FIX                                                │
│   git restore <file>       → Discard changes              │
│   git stash / stash apply  → Save/restore temp            │
│   git revert <commit-id>   → Revert a commit              │
│                                                           │
│ INSPECT                                                   │
│   git status               → Check status                 │
│   git log --oneline -10    → Recent commits               │
│   git diff                 → View changes                 │
└──────────────────────────────────────────────────────────┘
```

---

*← [02 — Maven](./02-maven-build-tool.md) | [04 — Agile, SDLC & JIRA →](./04-agile-sdlc-jira.md)*
