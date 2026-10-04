---
title: "Git Essentials & Workflows"
tags: [devops, devops/git]
aliases: [Git, Git Workflows, Git Internals, Rebase vs Merge, Trunk-Based Development, GitFlow, Conventional Commits, git reflog]
updated: 2026-09-27
---
# Git Essentials & Workflows

<Note>
**TL;DR**

Git is a **content-addressed object store** (blobs, trees, commits, tags) plus **refs** (branches, tags, HEAD) that point at commits. Branches are just movable pointers, which is why branching is cheap and why almost nothing is truly lost (see `git reflog`). Teams choose a workflow: **trunk-based development** (short-lived branches, feature flags, continuous delivery) is the high-performing default; **GitFlow** suits versioned, scheduled releases. Use **rebase** to tidy your own local history and **merge** (or squash-merge) to integrate shared work.
</Note>

## 1. Mental model: the three areas

{/* diagram source: /diagrams/diagram-167.mmd */}
![diagram](/diagrams/diagram-167.svg)

```bash
git status                     # what changed where
git diff                       # working dir vs index
git diff --staged              # index vs last commit
git add -p                     # stage hunks interactively (review what you commit)
git commit -m "feat: add order export"
git log --oneline --graph --decorate --all
git switch -c feature/export   # create + switch branch (modern replacement for checkout -b)
git restore file.txt           # discard working-dir change (modern replacement for checkout -- file)
git restore --staged file.txt  # unstage
git fetch && git pull --rebase # update, replaying your local commits on top
git push -u origin feature/export
```

## 2. Internals: objects and refs

Everything in `.git/objects` is addressed by the hash of its content (SHA-1 today, SHA-256 supported).

| Object | Contains |
|---|---|
| **blob** | File content (no name) |
| **tree** | Directory listing: names → blob/tree hashes + modes |
| **commit** | Root tree hash, parent commit(s), author, committer, message |
| **tag** (annotated) | Points at an object with tagger, message, optional signature |

{/* diagram source: /diagrams/diagram-168.mmd */}
![diagram](/diagrams/diagram-168.svg)

```bash
git cat-file -p HEAD            # show the commit object
git cat-file -p HEAD^{tree}     # show its tree
git rev-parse main              # hash a ref points to
cat .git/HEAD                   # "ref: refs/heads/main"
```

Consequences:
- A **branch** is a file containing a commit hash. Creating one is instant.
- **Commits are immutable.** "Rewriting history" (amend, rebase) creates *new* commits; the old ones remain until garbage-collected.
- Identical files are stored once. Git stores snapshots, and compresses them into **packfiles** with deltas.
- **Detached HEAD** = HEAD points at a commit instead of a branch; new commits there are reachable only via reflog unless you create a branch.

## 3. Merge vs rebase

```text
Before:            A---B---C  main
                        \
                         D---E  feature

git merge feature   A---B---C-------M  main      (M has two parents; history preserved)
                         \         /
                          D-------E

git rebase main     A---B---C---D'---E'  feature  (D, E replayed as new commits; linear)
(on feature)
```

| | Merge | Rebase |
|---|---|---|
| History | True, shows parallel work | Linear, easier to read and bisect |
| Rewrites commits | No | Yes (new hashes) |
| Safe on shared branches | Yes | **No**: never rebase commits others have pulled |
| Conflicts | Resolved once | Possibly once per replayed commit (`git rerere` helps) |

Common team policy: rebase your local feature branch on `main` to stay current (`git pull --rebase`), clean it with interactive rebase (`git rebase -i` to squash fixups), then integrate with a **squash merge** or **merge commit** through the PR. After rewriting a pushed branch, push with `git push --force-with-lease` (refuses if someone else pushed).

## 4. Branching strategies

### Trunk-based development
- Everyone integrates into `main` at least daily via **short-lived branches** (hours to a couple of days) or direct commits with pair review.
- Unfinished features hide behind **feature flags**.
- `main` is always releasable; CI runs on every commit; releases are tags or continuous deploys.
- Strongly associated with high DORA performance (see [CI-CD Pipelines](/Software%20Engineering/DevOps/CI-CD%20Pipelines)).

### GitFlow
- Long-lived `main` (released) and `develop` (integration), plus `feature/*`, `release/*`, `hotfix/*` branches.
- Fits software with explicit versions and support windows (mobile apps, on-prem products, libraries).
- Costs: long-lived branches, big merges, slower feedback. Its author added a note (2020) recommending simpler flows for continuously delivered web apps.

### GitHub Flow
- `main` + feature branches + PRs; deploy after merge (or from the branch before merge). A light version of trunk-based.

{/* diagram source: /diagrams/diagram-169.mmd */}
![diagram](/diagrams/diagram-169.svg)

## 5. Conventional Commits and semantic versioning

```text
<type>[optional scope][!]: <description>

feat(checkout): add Apple Pay
fix(api): return 404 for unknown order
chore(deps): bump express to 5.1.0
feat!: drop Node 18 support          # "!" or "BREAKING CHANGE:" footer → major version
```

Types: `feat` (→ minor), `fix` (→ patch), `docs`, `refactor`, `perf`, `test`, `build`, `ci`, `chore`. Tools like semantic-release, release-please or changesets generate versions and changelogs from them. Enforce with commitlint in a commit-msg hook or PR title check.

## 6. Recovering from mistakes

| Situation | Command |
|---|---|
| Undo last commit, keep changes staged | `git reset --soft HEAD~1` |
| Undo last commit, keep changes unstaged | `git reset HEAD~1` |
| Throw away last commit and changes | `git reset --hard HEAD~1` (dangerous) |
| Undo a pushed commit safely | `git revert <sha>` (new inverse commit) |
| Fix last commit message / add a file | `git commit --amend` |
| "I lost commits after a reset/rebase" | `git reflog` → `git reset --hard HEAD@{3}` or `git branch rescue <sha>` |
| Deleted a branch | `git reflog` / `git fsck --lost-found`, then `git branch name <sha>` |
| Find the commit that introduced a bug | `git bisect start; git bisect bad; git bisect good v1.3.0` (+ `git bisect run npm test`) |
| Park work temporarily | `git stash push -m "wip"` / `git stash pop` |
| Take one commit from another branch | `git cherry-pick <sha>` |

The **reflog** records every movement of HEAD and branch tips locally (kept ~90 days by default), so "lost" commits are almost always recoverable until garbage collection.

## 7. Worktrees

Work on several branches at once without stashing or cloning again. All worktrees share one object store.

```bash
git worktree add ../app-hotfix hotfix/1.4.1     # second checkout in another folder
git worktree add -b spike/cache ../app-spike    # new branch in a new worktree
git worktree list
git worktree remove ../app-hotfix
```

Great for reviewing a PR while your feature build runs, and for running several AI coding agents in parallel, each in its own worktree.

## 8. Other senior-level tools

- **Signed commits/tags** (GPG, SSH or Sigstore gitsign) + branch protection requiring signatures.
- **Branch protection / rulesets**: required reviews, status checks, linear history, no force pushes on `main`.
- **CODEOWNERS** for automatic reviewer assignment.
- **Monorepo scale**: `git sparse-checkout`, partial clone (`--filter=blob:none`), `git maintenance start`, Scalar. See [Monorepos](/Software%20Engineering/Frontend/14%20Tooling/Monorepos).
- **Large files**: Git LFS; never commit build artifacts or datasets.
- **Hooks**: pre-commit (lint/format), commit-msg (conventional commits). Client hooks are advisory; enforce in CI.
- **Secrets**: once pushed, a secret is compromised. Rotate first, then purge with `git filter-repo` if needed. Enable push protection / secret scanning.

<Info>
**🆕 2025–2026**

- **Git 3.0 is being prepared** (no release date announced as of 2026-09). Accepted breaking changes in the official BreakingChanges document: **SHA-256** as default hash for new repos, **reftable** as default ref storage, **`main`** as default initial branch, `safe.bareRepository` defaulting to explicit, and **Rust becoming mandatory** in phases (auto-detected from Git 2.52, enabled by default from 2.55, required in 3.0). Removals include `git whatchanged` and `git pack-redundant`. The last 2.x release will be a long-term support release. Existing SHA-1 repositories keep working.
- Worktrees became mainstream in 2025–2026 as the standard way to run multiple coding agents in parallel on one repo.
</Info>

## Pitfalls

<Warning>
**Common pitfalls**

- Rebasing or force-pushing a shared branch: teammates' history diverges. Use `--force-with-lease` and only on your own branches.
- Giant PRs from long-lived branches: slow reviews, painful conflicts. Keep PRs small.
- `git add .` committing `.env`, build output or huge files. Maintain `.gitignore`, use `git add -p`.
- `git reset --hard` with uncommitted work: that work is not in the reflog (only commits are).
- Merge commits from `git pull` cluttering history: set `git config --global pull.rebase true`.
</Warning>

## Senior perspective

- Branching strategy is a delivery decision, not a Git preference: long-lived branches delay integration and hide risk. Pair trunk-based development with feature flags, fast CI and good test coverage.
- Protect `main` with rulesets and required checks; make the fast path the safe path.
- History is documentation: good commit messages explain **why**. Squash noisy WIP commits; keep meaningful ones.
- Be ready for SHA-256/Git 3.0 in tooling that parses hashes (40 vs 64 hex characters).

## Interview questions

<Accordion title="What is a Git commit, internally?">
An immutable object containing a pointer to a root tree (the snapshot), parent commit hash(es), author/committer metadata and a message. Its ID is the hash of that content, so changing anything creates a new commit.
</Accordion>

<Accordion title="Merge vs rebase: when do you use each?">
Rebase to update and clean up your own unpublished branch (linear history). Merge (or squash-merge via PR) to integrate into shared branches. Never rebase commits others have based work on.
</Accordion>

<Accordion title="How do you recover a commit you &quot;lost&quot; after `git reset --hard`?">
`git reflog` shows previous positions of HEAD; find the commit and `git reset --hard <sha>` or create a branch at it. Works until the objects are garbage-collected (default ~90 days for reachable reflog entries, 30 for unreachable).
</Accordion>

<Accordion title="Trunk-based development vs GitFlow?">
Trunk-based: everyone integrates into main frequently via short-lived branches, feature flags hide incomplete work, main is always deployable, ideal for continuous delivery. GitFlow: develop/release/hotfix branches for scheduled versioned releases; more ceremony and merge overhead.
</Accordion>

<Accordion title="`git revert` vs `git reset`?">
`reset` moves the branch pointer (rewrites history; fine locally). `revert` adds a new commit that undoes a previous one (safe on shared branches).
</Accordion>

<Accordion title="What are Conventional Commits good for?">
A structured commit message format (`feat:`, `fix:`, `!` for breaking) that enables automated semantic versioning, changelog generation and readable history.
</Accordion>

<Accordion title="What is `git bisect`?">
A binary search over history to find the commit that introduced a bug; mark good and bad commits (or run a test script automatically) and Git halves the range each step.
</Accordion>

## Related
- [DevOps Index](/Software%20Engineering/DevOps/DevOps%20Index) · [CI-CD Pipelines](/Software%20Engineering/DevOps/CI-CD%20Pipelines) · [GitOps - Argo CD & Flux](/Software%20Engineering/DevOps/GitOps%20-%20Argo%20CD%20%26%20Flux) · [Monorepos](/Software%20Engineering/Frontend/14%20Tooling/Monorepos) · [Linux & Shell Essentials](/Software%20Engineering/DevOps/Linux%20%26%20Shell%20Essentials)

## Sources
- [Pro Git book (Scott Chacon, Ben Straub)](https://git-scm.com/book/en/v2), especially ch. 10 *Git Internals*
- [Git docs: git-reflog](https://git-scm.com/docs/git-reflog) · [git-worktree](https://git-scm.com/docs/git-worktree) · [git-bisect](https://git-scm.com/docs/git-bisect)
- [Git docs: BreakingChanges (Git 3.0)](https://git-scm.com/docs/BreakingChanges)
- [Conventional Commits 1.0.0](https://www.conventionalcommits.org/en/v1.0.0/) · [Semantic Versioning 2.0.0](https://semver.org/)
- [trunkbaseddevelopment.com](https://trunkbaseddevelopment.com/)
- [Vincent Driessen: A successful Git branching model (with 2020 note)](https://nvie.com/posts/a-successful-git-branching-model/)
- [DORA: Trunk-based development capability](https://dora.dev/capabilities/trunk-based-development/)
