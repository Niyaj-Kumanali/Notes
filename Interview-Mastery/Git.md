# Git

## Overview

- **Definition** — Git is a distributed version control system that tracks source code changes, enables branching and merging, and supports collaboration through local and remote repositories.
- **Why It Exists** — Solves the problems of coordinating concurrent work on shared code, maintaining a history of changes, reverting to previous states, and enabling non-linear development through branching, all without requiring a central server for basic operations.
- **Historical Context** — Created in 2005 by Linus Torvalds to manage Linux kernel development, replacing BitKeeper. Git emphasized performance, distributed architecture, data integrity (SHA-1 hashing), and support for non-linear workflows. It became the dominant VCS through platforms like GitHub, GitLab, and Bitbucket.
- **Key Concepts** — **Repository** (local or remote copy of the codebase), **Commit** (snapshot of changes), **Staging area** (intermediate index for preparing commits), **Branch** (movable pointer to a commit), **Remote** (hosted repository for collaboration), **Merge** (combining branches), **Rebase** (rewriting commit base), **Stash** (temporary shelving), **Working tree** (checked-out files on disk).

## Core Concepts

- Local vs remote: Every Git repository is a full copy of the project history. Local repositories enable offline operations (commits, diffs, logs). Remote repositories are hosted copies (GitHub, GitLab, Bitbucket) that facilitate collaboration through fetch, pull, and push operations.
- The staging area (index) is an intermediate layer between the working tree and the repository. Files are added to the staging area with `git add` and committed from the staging area with `git commit`. This allows selective staging of changes and crafting atomic commits with partial file modifications.
- Branching is Git's core mechanism for parallel development. A branch is a lightweight movable pointer to a commit. Feature branches isolate work until it is ready to merge. GitFlow uses a structured model with main, develop, feature, release, and hotfix branches. Trunk-based development keeps branches short-lived and merges frequently into a single main line.
- Merging strategies:
  - Fast-forward: When the target branch has not diverged, Git simply moves the branch pointer forward. No merge commit is created.
  - 3-way merge: When branches have diverged, Git creates a merge commit that combines changes from both branches using three snapshots (common ancestor, source, target).
  - Squash merge: Combines all commits from a feature branch into a single commit on the target branch, discarding individual commit history.
  - Rebase merge: Rewrites the feature branch commits onto the tip of the target branch, creating a linear history before fast-forward merging.

- Rebase vs merge tradeoffs: Rebase produces a linear, clean history but rewrites commit hashes, making it dangerous for shared branches. Merge preserves history and commit authors but introduces merge commits that can clutter the log. The rule of thumb: rebase before pushing, merge after pushing.
- Conflict resolution occurs when the same part of a file is modified in two branches being merged or rebased. Git marks the conflicting region in the file with conflict markers. The developer edits the file to resolve the conflict, stages it, and commits or continues the rebase.
- git log filtering options: `--author=<pattern>` filters by author, `--grep=<pattern>` filters commit messages, `--since`/`--until` filters by date, `--oneline` shows compact output, `--graph` shows branch topology, `--decorate` shows branch and tag names, `--format="%h %an %s"` customizes output with placeholders.
- Interactive rebase (`git rebase -i`) opens a todo list of commits that can be reworded (edit message), squashed (combine with previous commit and merge messages), fixup (combine with previous commit discarding the message), reordered, or dropped. Essential for cleaning up a feature branch before merging.
- Stashing (`git stash`) temporarily shelves uncommitted changes (tracked files by default). Use `git stash push` to stash including untracked files, `git stash pop` to apply and drop the latest stash, `git stash apply` to keep the stash, and `git stash list` to view all stashes.
- Cherry-pick (`git cherry-pick <hash>`) applies a specific commit from another branch onto the current branch. Useful for selectively porting bug fixes between release branches without merging all intervening commits.
- Revert (`git revert <hash>`) creates a new commit that undoes a previous commit's changes. It is safe for shared history because it does not rewrite existing commits. Reset (`git reset --soft/mixed/hard <ref>`) moves the branch pointer and optionally modifies the staging area or working tree. Reset rewrites history and should not be used on published branches.
- Remote management: A repository can have multiple remotes. origin is typically the fork or default remote. upstream is often the original repository in a forking workflow. `git fetch` downloads remote objects and refs without merging. `git pull` runs fetch then merge (or rebase with `--rebase`). `git push` uploads local commits to a remote and updates remote refs.
- Pull request workflow: Fork the upstream repository, create a feature branch, commit changes, push the branch to the fork, open a pull request, review and discuss changes, merge the PR, delete the feature branch both locally and on the remote.
- Git hooks are scripts that execute automatically on Git events. Pre-commit hooks run before each commit and are commonly used for linting, formatting checks, or running a subset of tests. Hooks are stored in `.git/hooks/` and are not pushed to the remote; teams use tools like husky to manage shared hooks.
- .gitignore is a pattern file that tells Git which files to ignore. Common entries include build artifacts (target/, build/, .gradle/), IDE files (.idea/, *.iml, .vscode/), dependency directories (node_modules/, vendor/), environment files (.env, .venv), and OS files (.DS_Store).

## Common Mistakes

- **Force-pushing to shared branches**
  - A developer modifies the history of a shared branch (e.g., main, develop) by running `git push --force` after rebasing or amending commits. This destroys commits that teammates may have based their own work on, causing divergent histories and difficult recoveries.
  - **Why it looks correct:** The developer's local branch is clean and the force push succeeds. The impact on teammates is not immediately apparent to the pusher.
  - Never use `git push --force` on branches that others might have fetched. Use `git push --force-with-lease` instead, which rejects the push if the remote has refs the local branch does not know about. Prefer creating a new branch over rewriting shared history.

- **Rebasing published commits**
  - A developer runs `git rebase` on a branch that has already been pushed to a shared remote and other developers have based work on. The rewritten commits have new hashes, breaking the shared understanding of the branch history.
  - **Why it looks correct:** The rebase produces a cleaner, linear history on the developer's local branch. The immediate build succeeds and the developer may not realize others depend on the branch.
  - Apply the rule: never rebase commits that have been pushed to a shared remote. Use merge to integrate changes on shared branches. If a rebase is absolutely necessary, coordinate with the team and communicate a force-push window.

- **Merge conflicts due to poor commit discipline**
  - Developers work on large features for days without pulling upstream changes, then face massive merge conflicts when merging back to main. Conflicts span multiple files and involve complex interdependencies, making resolution error-prone.
  - **Why it looks correct:** Working in isolation avoids interruptions and the branch tracks only the feature work. Frequent merging feels like overhead during development.
  - Pull or rebase from the target branch daily to stay current. Make atomic, single-purpose commits that are easy to understand and revert. Use short-lived feature branches merged within a day or two. Configure CI to run against the target branch to detect integration issues early.

## Real-World Scenarios

### Handling an urgent hotfix on a release branch

- A critical production bug is discovered in version 2.3 of a product. The team creates a hotfix branch from the release/v2.3 tag. A developer commits the fix on the hotfix branch and pushes it. The fix is merged into main and the release/v2.3 branch. The release branch is re-tagged and deployed. The same fix is cherry-picked onto the develop branch to keep it in sync. This workflow ensures the hotfix reaches production without deploying incomplete features from develop.

### Recovering from a force-push disaster

- A junior developer force-pushes a rebased branch to the shared develop branch, wiping out commits from three teammates. The recovery uses `git reflog` on a teammate's local repository to find the original commit hashes. The teammate creates a new branch from the last known good commit and pushes it to develop. The team re-applies the lost commits using cherry-pick. The developer is instructed to use `--force-with-lease` going forward.

## Scenario-Based Questions

**Q: You accidentally committed a change to the main branch instead of a feature branch. How do you undo this mistake?**

- Use `git log` to find the commit hash. Run `git revert <hash>` to create a new commit that undoes the change on main. Then create a feature branch from the commit before the revert and cherry-pick the original commit onto it. Alternatively, if you have not pushed yet, use `git reset HEAD~1` and then `git checkout -b feature-branch` to move the commit to a new branch.

- **Interview follow-up:** How would you handle this if the commit has already been pushed and other developers may have pulled from main?

**Q: Your team's feature branches frequently diverge from main, causing complex merge conflicts. What process changes do you recommend?**

- Adopt trunk-based development with short-lived feature branches merged within one or two days. Encourage developers to pull or rebase from main at least once daily. Use CI to run builds against the merge target so integration issues surface early. Educate the team on making atomic commits focused on a single concern, which are easier to merge and resolve.

- **Interview follow-up:** How would you handle a long-running feature that genuinely cannot be split into shorter increments, such as a months-long database migration?

## Interview Questions

- **What is the difference between git revert and git reset?**
  - git revert creates a new commit that undoes the changes of a specified commit. It is safe for shared history because it does not rewrite existing commits. git reset moves the branch pointer to a specified commit and optionally modifies the staging area (--mixed) or working tree (--hard). Reset rewrites history and should only be used on unshared local branches.

- **Explain the tradeoffs between rebase and merge.**
  - Rebase produces a linear, clean commit history by rewriting commits onto the target branch tip. It avoids merge commits but changes commit hashes, making it dangerous for shared branches. Merge preserves commit history and hashes but introduces merge commits that can clutter the log. Merge is safer for shared branches; rebase is preferred for local feature branch cleanup before pushing.

- **How does git stash work and when should you use it?**
  - git stash temporarily shelves uncommitted changes (tracked files by default) so you can switch branches or apply a hotfix without committing incomplete work. Changes are stored on a stack. Use `git stash pop` to reapply and remove the latest stash, or `git stash apply` to reapply while keeping it. Use `git stash push -u` to include untracked files. Stash is ideal for short interruptions but should not replace proper branching.

- **What is the staging area and why is it useful?**
  - The staging area (index) is an intermediate layer between the working tree and the repository. Files are added to the staging area with `git add` and committed with `git commit`. The staging area enables selective staging of file changes, crafting atomic commits that group related changes, and reviewing staged changes with `git diff --cached` before committing.

## Developer Recommendations

- **Rebase local branches before merging to main**
  - Produces a clean, linear history that is easier to review, bisect, and revert. Eliminates unnecessary merge commits in the main branch.
  - Before merging a feature branch, run `git rebase main` to replay commits on top of the latest main. Resolve any conflicts locally. Then fast-forward merge with `git checkout main && git merge feature-branch`.
  - **Production story:** A team's main branch had 50 merge commits in a week, making git bisect nearly impossible. Switching to rebase-before-merge reduced the main branch log to a clean timeline where each commit was traceable to a single feature or fix.

- **Write atomic commits with descriptive messages**
  - Atomic commits (one concern per commit) make the history easier to understand, review, revert, and cherry-pick. Descriptive messages serve as documentation.
  - Use the staging area to stage only related changes. Write commit messages in imperative tense with a concise summary line and optional body. Use `git add -p` to stage specific hunks within a file.
  - **Production story:** When a production bug was traced to a specific change, an atomic commit structure allowed the team to revert exactly that change without losing other work in the same file, resolving the incident in minutes instead of hours.

- **Use feature branches with descriptive naming conventions**
  - Clearly named branches communicate purpose and ownership. Naming conventions make automated cleanup and CI integration straightforward.
  - Adopt a convention like `prefix/description` (e.g., `feature/user-auth`, `bugfix/null-pointer`, `hotfix/payment-timeout`). Delete branches after merging both locally (`git branch -d`) and remotely (`git push origin --delete`).

- **Configure pre-commit hooks for code quality**
  - Catches formatting issues, linting errors, and missing tests before they enter the repository. Reduces review noise and enforces team standards automatically.
  - Use a tool like husky or lefthook to manage shared hooks in the repository. Configure hooks for linting, formatting, and running a fast subset of tests. Keep hooks fast so developers do not disable them.
