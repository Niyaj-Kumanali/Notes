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

## Use Cases

Reach for Git whenever you need to track changes, collaborate on code, or manage multiple release streams.

- **Feature branch collaboration** — Developers create branches from main, commit incrementally, and merge via pull requests after code review.
  - Keeps main stable while multiple features are developed in parallel.
  - **Avoid when:** deploying directly from trunk (trunk-based development) matches your release cadence better.

- **Hotfix patching in production** — Branch from a release tag, fix the bug, merge back to both main and the release branch.
  - Cherry-pick the fix to other active branches without merging unrelated changes.
  - **Avoid when:** the fix can wait for the next regular release cycle.

- **Commit history cleanup before merging** — Interactive rebase squashes, rewords, and reorders commits so the target branch receives a clean, reviewable history.
  - Essential for maintaining readable project history.
  - **Avoid when:** the branch is shared with other developers — rebasing rewrites commit hashes.

- **Recovering from mistakes** — `git reflog` locates lost commits, `git reset` undoes staged or committed changes, and `git revert` safely undoes public history.
  - Every Git user should know these recovery workflows.
  - **Avoid when:** the commits exist only on a remote with no local clones — reflog is local-only.

- **Open-source contribution** — Fork, feature branch, commit, push, open a PR, then sync from upstream via fetch and rebase.
  - The standard model for contributing to projects on GitHub, GitLab, and Bitbucket.
  - **Avoid when:** you have direct push access to the shared branch.

---

## Scenario-Based Questions

**Q: You accidentally committed a change to the main branch instead of a feature branch. How do you undo this mistake?**

- Use `git log` to find the commit hash. Run `git revert <hash>` to create a new commit that undoes the change on main. Then create a feature branch from the commit before the revert and cherry-pick the original commit onto it. Alternatively, if you have not pushed yet, use `git reset HEAD~1` and then `git checkout -b feature-branch` to move the commit to a new branch.

- **Interview follow-up:** How would you handle this if the commit has already been pushed and other developers may have pulled from main?

**Q: Your team's feature branches frequently diverge from main, causing complex merge conflicts. What process changes do you recommend?**

- Adopt trunk-based development with short-lived feature branches merged within one or two days. Encourage developers to pull or rebase from main at least once daily. Use CI to run builds against the merge target so integration issues surface early. Educate the team on making atomic commits focused on a single concern, which are easier to merge and resolve.

- **Interview follow-up:** How would you handle a long-running feature that genuinely cannot be split into shorter increments, such as a months-long database migration?


**Q: A teammate accidentally pushed a commit containing a database password to a public GitHub repository. How do you remediate this?**

- Immediately rotate the exposed password. Then run `git filter-branch --tree-filter` or use the BFG Repo-Cleaner tool to remove the file from all commits in the repository history. Force-push the rewritten history to all branches. Notify GitHub Support to invalidate any cached copies. Finally, instruct the team to rebase any local branches on the cleaned history. The password must still be considered compromised even after removal.

- **Interview follow-up:** What are the limitations of git filter-branch, and when would you use BFG instead?


**Q: You run git merge and get a conflict on every line of a file that only one developer modified. What is happening and how do you fix it?**

- The file likely has different line endings (CRLF vs LF) or whitespace differences between the two branches. Git treats the entire file as changed even though only whitespace differs. Run `git merge --strategy-option=ignore-space-change` or `git merge -Xignore-all-space` to merge ignoring whitespace. After merging, normalize line endings by configuring `.gitattributes` with `* text=auto` and committing a line-ending normalization commit.

- **Interview follow-up:** How would you configure a repository so that whitespace conflicts never happen again?


**Q: You need to split a monorepo into multiple separate repositories while preserving the full commit history for each directory. How do you do it?**

- Use `git filter-branch --subdirectory-filter` or `git subtree split` to extract a subdirectory into its own repository with complete history. The newer `git filter-repo` tool (recommended over filter-branch) can do this more efficiently. After extraction, push the new repository to a new remote. The original monorepo remains unchanged. Each extracted repo retains all commits that touched files in its directory.

- **Interview follow-up:** How would you set up the new repositories so that future changes to shared code can be synchronized across them?


**Q: A developer ran git reset --hard and lost an hour of unstaged work. Can you recover it?**

- Yes, if the files were previously staged with `git add`, they can be recovered from the Git object store using `git fsck --lost-found`, which finds dangling blobs. If the files were never staged, recovery depends on the editor's local history or IDE local history feature. For future prevention, recommend `git stash` before risky operations, or use `git reset --soft` instead of `--hard` to preserve working tree changes.

- **Interview follow-up:** What is the difference between dangling blobs and unreachable objects, and how does git gc affect recovery?


**Q: Your CI pipeline runs tests on every push, but you want to skip CI for documentation-only commits. How do you configure this?**

- Include `[skip ci]` or `[ci skip]` in the commit message to tell most CI systems (GitHub Actions, GitLab CI, Jenkins) to skip the pipeline. For more granular control, configure CI workflow rules to only trigger when certain file paths change (e.g., `paths-ignore: ['docs/**', '*.md']` in GitHub Actions). This reduces CI costs and speeds up documentation updates.

- **Interview follow-up:** When is skipping CI dangerous, and what alternative approach would you recommend for ensuring documentation changes are still reviewed?


**Q: You need to apply a hotfix from the main branch to a release branch, but the release branch has diverged significantly. Cherry-pick the commits or merge the branch?**

- Use `git cherry-pick` for selective commit transfer. Identify the hotfix commits on main using `git log --oneline main --fixes=<ticket-id>` and cherry-pick them onto the release branch. Merging the entire main branch would pull in unrelated features not ready for release. After cherry-picking, run tests on the release branch to verify the fix works in that context. Cherry-pick creates new commit hashes, which is acceptable for release branches.

- **Interview follow-up:** How would you handle a situation where the cherry-picked commit conflicts with changes already on the release branch?


**Q: Multiple developers are working on the same file and merging frequently causes the same conflicts to appear repeatedly. What strategy reduces this pain?**

- Use `git rerere` (Reuse Recorded Resolution) to automatically reapply previously resolved conflicts. Enable it with `git config --global rerere.enabled true`. Rerere records conflict resolutions during merge or rebase and replays them automatically when the same conflict pattern appears again. Additionally, encourage smaller, more frequent merges and using `git diff` and `git log` communication to coordinate edits to shared files.

- **Interview follow-up:** How does git rerere store conflict resolutions, and can it be shared across the team?


**Q: Your team wants to enforce a policy that every commit message follows a specific format (e.g., Conventional Commits). How do you implement this?**

- Use a `commit-msg` Git hook that validates the commit message against a regex pattern. For team-wide enforcement without relying on each developer's hooks, use a CI check (e.g., GitHub Actions with a commit-lint action) that fails the build if the commit message format is invalid. For local enforcement, use tools like commitlint installed via husky. This ensures the commit history remains consistent and parseable for changelog generation.

- **Interview follow-up:** How would you retroactively fix commit messages in a feature branch that did not follow the convention, before merging to main?

## Interview Questions

- **What is the difference between git revert and git reset?**
  - git revert creates a new commit that undoes the changes of a specified commit. It is safe for shared history because it does not rewrite existing commits. git reset moves the branch pointer to a specified commit and optionally modifies the staging area (--mixed) or working tree (--hard). Reset rewrites history and should only be used on unshared local branches.

- **Explain the tradeoffs between rebase and merge.**
  - Rebase produces a linear, clean commit history by rewriting commits onto the target branch tip. It avoids merge commits but changes commit hashes, making it dangerous for shared branches. Merge preserves commit history and hashes but introduces merge commits that can clutter the log. Merge is safer for shared branches; rebase is preferred for local feature branch cleanup before pushing.

- **How does git stash work and when should you use it?**
  - git stash temporarily shelves uncommitted changes (tracked files by default) so you can switch branches or apply a hotfix without committing incomplete work. Changes are stored on a stack. Use `git stash pop` to reapply and remove the latest stash, or `git stash apply` to reapply while keeping it. Use `git stash push -u` to include untracked files. Stash is ideal for short interruptions but should not replace proper branching.

- **What is the staging area and why is it useful?**
  - The staging area (index) is an intermediate layer between the working tree and the repository. Files are added to the staging area with `git add` and committed with `git commit`. The staging area enables selective staging of file changes, crafting atomic commits that group related changes, and reviewing staged changes with `git diff --cached` before committing.

- **What is the difference between git fetch and git pull?**
  - `git fetch` downloads commits, objects, and refs from a remote repository without merging them into your local branch. It safely updates remote-tracking branches. `git pull` runs fetch then immediately merges the fetched changes into your current branch (or rebases with `--rebase`). Use fetch to inspect incoming changes before integrating; use pull only when you are ready to merge.

- **How do you undo a commit that has already been pushed to a shared branch?**
  - Use `git revert <hash>` to create a new commit that undoes the changes. This is safe because it does not rewrite history. If the commit introduced a security issue, revert it and notify the team. Never use `git reset` on pushed commits in shared branches. After reverting, push the revert commit normally and communicate with the team about the change.

- **What is a detached HEAD state and how do you recover from it?**
  - A detached HEAD occurs when you check out a specific commit instead of a branch (e.g., `git checkout <hash>` or `git checkout origin/main`). You are no longer on a named branch. Create a new branch from the current position with `git checkout -b new-branch-name`. If you made commits in detached HEAD, they can be found via `git reflog` and attached to a branch before they are garbage collected.

- **Explain the purpose of git bisect and how you use it.**
  - `git bisect` performs a binary search through the commit history to find the commit that introduced a bug. Start with `git bisect start`, mark the current commit as bad (`git bisect bad`) and a known good commit as good (`git bisect good <ref>`). Git checks out a midpoint commit. Test it, mark it good or bad, and repeat. When the single culprit commit is found, exit with `git bisect reset`. Automate with `git bisect run <script>`.

- **What is the difference between a soft, mixed, and hard reset?**
  - `git reset --soft HEAD~1` moves the branch pointer back one commit but keeps all changes in the staging area and working tree. `git reset --mixed HEAD~1` (default) moves the pointer and unstages changes but keeps them in the working tree. `git reset --hard HEAD~1` moves the pointer and discards all changes in both staging and working tree. Use soft to re-commit, mixed to re-stage, and hard only when you are certain.

- **How does git handle binary files and how can you optimize repository size?**
  - Git stores binary files as full blobs for each version since it cannot diff them efficiently. This bloats the repository. Store large binaries with Git LFS (Large File Storage), which replaces binaries in the repo with text pointers and stores the actual file on a remote server. Use `.gitattributes` patterns like `*.psd filter=lfs diff=lfs merge=lfs -text` to manage binaries. Regularly run `git gc` to optimize the local repository.

- **What is a git submodule and what are its drawbacks?**
  - A submodule is a reference to another Git repository embedded at a specific commit. It allows a project to include an external dependency pinned to an exact version. Drawbacks include complex update workflows (`git submodule update --init --recursive`), the need to commit the submodule pointer change separately, and the risk of the submodule URL becoming unavailable. Consider subtrees or package managers as alternatives.

- **How do you squash multiple commits into one in Git?**
  - Use `git rebase -i HEAD~N` to interactively rebase the last N commits. In the todo list, change `pick` to `squash` (or `s`) for all commits you want to merge into the previous commit. Save and close. Git will prompt for a combined commit message. Alternatively, use `git merge --squash` on a feature branch to produce a single commit. Squashing should only be done on local branches before pushing.

- **What is git reflog and how do you use it for recovery?**
  - `git reflog` shows a local log of every commit HEAD has pointed to, including commits lost after resets, rebases, or branch deletions. Each entry shows a HEAD@{N} reference. Recover lost commits by checking out the reflog reference or resetting to it. Reflog entries expire after 90 days by default. Reflog is local only — it cannot recover commits that were never fetched from a remote.

- **Explain the difference between a bare repository and a non-bare repository.**
  - A bare repository has no working tree and only stores Git data (objects, refs, HEAD). It is used as a server-side remote — you cannot directly edit files in it. A non-bare repository has a working tree where files are checked out for editing. `git init --bare` creates a bare repo. Central repositories and remote mirrors are typically bare. Developers work in non-bare repos.

- **How do you set up a Git hook to run tests before every commit?**
  - Create a `pre-commit` script in `.git/hooks/pre-commit` (executable) that runs the test suite. If the tests fail, the script exits with a non-zero status and the commit is aborted. For team-wide hooks, use tools like husky, lefthook, or `core.hooksPath` to point to a version-controlled hooks directory. Keep the hook fast to avoid developer frustration — run only a focused subset of tests.

- **What is the difference between git tag and git branch?**
  - A tag is a named reference to a specific commit that does not move. Tags are used for releases (v1.0, v2.3.1) and are lightweight (just a pointer) or annotated (with metadata and GPG signature). A branch is a movable pointer that advances with new commits. Tags are read-only once created; branches evolve. Push tags with `git push --tags` and check them out for production deployments.

- **How do you resolve a merge conflict manually?**
  - Git marks conflicts in the affected file with `<<<<<<< HEAD`, the current branch content, `=======`, the incoming branch content, and `>>>>>>> branch-name`. Edit the file to decide which content to keep or combine both. Remove the conflict markers. Stage the resolved file with `git add`. If merging, commit the merge. If rebasing, run `git rebase --continue`. Always test the resolved file before continuing.

- **What is the purpose of git worktree and when would you use it?**
  - `git worktree` allows you to check out multiple branches simultaneously in separate directories from a single repository. Use it when you need to work on a hotfix while keeping your current branch checked out, or to run tests on different branches without stashing. Add a worktree with `git worktree add ../path branch-name`. Each worktree shares the same Git objects but has its own working tree and index.

- **How does git blame work and how can you use it for debugging?**
  - `git blame <file>` annotates each line of a file with the commit hash, author, date, and commit message of the last change to that line. Use it to identify when a specific change was introduced, who made it, and in what context. Combine with `git log --oneline` on the blame commit to understand the full change. Blame is essential for tracking down regressions and understanding code evolution.

- **What is the difference between git archive and git bundle, and when would you use each?**
  - `git archive` creates a tar or zip archive of the repository at a specific commit without history. Useful for distributing source code releases. `git bundle` creates a binary file containing Git objects and refs that can be used as a remote repository. Useful for offline transfer — you can pull or fetch from a bundle file. Archive is for distribution; bundle is for synchronization between disconnected repositories.

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
