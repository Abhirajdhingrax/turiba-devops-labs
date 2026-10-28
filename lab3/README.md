# Lab 3 · Branch, review, release

- **Fork:** https://github.com/Abhirajdhingrax/turiba-devops-labs
- **Release:** https://github.com/Abhirajdhingrax/turiba-devops-labs/releases/tag/v0.1.0
- **Reviewer:** @tbd

## Part 1 · Set up

Output of `ssh -T git@github.com` (or `gh auth status`):

```text
Hi Abhirajdhingrax! You've successfully authenticated, but GitHub does not provide shell access.
```

Output of `git remote -v`:

```text
origin	git@github.com:Abhirajdhingrax/turiba-devops-labs.git (fetch)
origin	git@github.com:Abhirajdhingrax/turiba-devops-labs.git (push)
upstream	https://github.com/mleitass/turiba-devops-labs.git (fetch)
upstream	https://github.com/mleitass/turiba-devops-labs.git (push)
```

## Part 2 · Playground

The hash in `.git/refs/heads/main` (step 2.3): `9bbc9d7e2cfdb541854232c1201e846ae551cd14`

The first line of `git cat-file -p HEAD` (step 2.3): `tree 6246df39db4c0193d1ee086b4532c335d9cc4ad0`

The conflict block you saw in `config/app.env` (step 2.6):

```text
<<<<<<< HEAD
REPLICAS=2
=======
REPLICAS=3
>>>>>>> feature/scale-up
LOG_LEVEL=info
```

Which value you kept, and why: I kept `REPLICAS=3`, because the `feature/scale-up` branch was the deliberate change to handle more load, and three replicas still leave two running if one fails.

The final `git log --oneline --graph --all` (step 2.6):

```text
*   a889ecb Merge branch 'feature/scale-up'
|\  
| * a42e7c5 feat: scale to 3 replicas
* | e6c48a4 feat: scale to 2 replicas
|/  
*   a283d21 Merge branch 'feature/logging'
|\  
| * ed742e6 feat: add log level
* | 697bc4d docs: add README
|/  
* 7a3e7e3 feat: add health message
* 9bbc9d7 feat: print done after deploy
* 6fc44e4 feat: add app config
* ec21e4f feat: add deploy script
```

## Part 3 · Rescue missions

### m1 (worked example)

- PASS line: `PASS m1 756d3a9e`
- Symptom: `deploy.sh` was changed and broken; nothing staged or committed
- Command: `git restore deploy.sh`
- Why this command, and not a more dangerous one: It resets only this one file to the last commit. `git reset --hard` would also throw away every other uncommitted change, and deleting the folder would lose the history. The broken edit is gone for good, because it was never committed.

### m2

- PASS line: `PASS m2 2b80ce6b`
- Symptom: `.env` with a database password was staged together with a real change in `app.js`
- Command: `git restore --staged .env`, then `echo .env > .gitignore`, `git commit -m "feat: read the port from PORT"` (only `app.js` was staged), then `git add .gitignore` and `git commit -m "chore: ignore .env"`
- Why this command, and not a more dangerous one: `git restore --staged` only takes `.env` out of the index and keeps both the file and the staged `app.js` change. `git reset --hard` would throw the `app.js` change away, and committing first and deleting `.env` later would leave the password in the history forever.

### m3

- PASS line: `PASS m3 0692ebbe`
- Symptom: The last commit was called `fxi typo` and was missing `CONTRIBUTING.md`; nothing pushed
- Command: `git add CONTRIBUTING.md` then `git commit --amend -m "docs: fix typos in README and add CONTRIBUTING.md"`
- Why this command, and not a more dangerous one: `--amend` replaces the last commit with a corrected one, so there is still one commit with a proper message. That is safe only because nothing was pushed. A second commit would keep the bad message, and `git reset --hard` would lose the README fix.

### m4

- PASS line: `PASS m4 711d2926`
- Symptom: Two commits were made on `main` but belong on `feature/cache`; nothing pushed
- Command: `git branch feature/cache` then `git reset --hard origin/main`
- Why this command, and not a more dangerous one: Creating the branch first keeps both commits safe on `feature/cache`; only then is `main` moved back to `origin/main`. The working tree was clean, so `reset --hard` lost nothing. Resetting first, or redoing the work by hand, could lose the commits, and nothing had to be pushed.

### m5

- PASS line: `PASS m5 18d16176`
- Symptom: `feat: set replicas to 0` was already pushed, and the teammate's clone has it
- Command: `git revert --no-edit HEAD` then `git push`
- Why this command, and not a more dangerous one: `git revert` adds a new commit that undoes the bad one, so the shared history is not rewritten and the teammate can simply pull (fast-forward). `git reset --hard` plus a force push would rewrite origin, so the teammate's clone would no longer match and the bad commit could come back.

### m6

- PASS line: `PASS m6 72b9bca8`
- Symptom: After `git reset --hard HEAD~3`, three commits were missing from `main`
- Command: `git reflog` (shows `HEAD@{1}: commit: feat: add request logging`), then `git reset --hard HEAD@{1}`
- Why this command, and not a more dangerous one: The commits still exist in `.git`; the reflog remembers where HEAD was before the reset, so `main` can be moved back to it. The working tree was clean, so nothing was lost. Deleting and recloning would throw the reflog away, and the commits would be lost for good. (`git branch rescue HEAD@{1}` would be an even safer option.)

### m7 (stretch)

- PASS line: `PASS m7 5d44bf98`
- Symptom: A fix was committed on a detached HEAD after checking out tag `v1.0.0`
- Command: `git switch -c hotfix/1.0.1`
- Why this command, and not a more dangerous one: Creating a branch right where HEAD is keeps the fix reachable. Switching to `main` first would leave the commit on no branch, findable only through the reflog until Git cleans it up.

### m8 (stretch)

- PASS line: `PASS m8 867f31ce`
- Symptom: One of 20 commits broke `test.sh`
- Command: `git bisect start`, `git bisect bad`, `git bisect good <first commit>`, `git bisect run ./test.sh`, hash into `ANSWER`, `git bisect reset`
- Why this command, and not a more dangerous one: Bisect is a binary search: about 5 test runs instead of checking 20 commits by hand, and it never changes any commit.

## Part 4 · Protected main and pull requests

The rejected push to `main` (step 4.2):

```text
remote: error: GH013: Repository rule violations found for refs/heads/main.        
remote: Review all repository rules at https://github.com/Abhirajdhingrax/turiba-devops-labs/rules?ref=refs%2Fheads%2Fmain        
remote: 
remote: - Changes must be made through a pull request.        
remote: 
To github.com:Abhirajdhingrax/turiba-devops-labs.git
 ! [remote rejected] main -> main (push declined due to repository rule violations)
error: failed to push some refs to 'github.com:Abhirajdhingrax/turiba-devops-labs.git'
```

- PR #1 (`.gitignore`): https://github.com/Abhirajdhingrax/turiba-devops-labs/pull/1
- PR #2 (`/healthz` docs, with the conflict): https://github.com/Abhirajdhingrax/turiba-devops-labs/pull/2

## Part 5 · Release

- Release: https://github.com/Abhirajdhingrax/turiba-devops-labs/releases/tag/v0.1.0
- Why `0.1.0` and not `1.0.0`? In Semantic Versioning, `0.y.z` means initial development: the API is not declared stable and may still change. `1.0.0` would promise a stable public API, which this course project does not have yet.

## Answers

1. In mission 5, why `git revert` and not `git reset --hard` plus a force push?

   `git revert` adds a new commit that undoes the bad one, so the history everyone already has stays the same and the teammate can just `git pull`. `git reset --hard` plus `git push --force` rewrites the shared history: the teammate's clone no longer matches origin, their next pull or push gets messy, and they might even push the bad commit back. On a protected `main`, force pushes are blocked anyway.

2. Why couldn't Git resolve the `CHANGELOG.md` conflict by itself, and how did you decide what to keep?

   Both branches added a different line at the same place: the end of the `### Added` list. Git merges line by line and cannot know whether one line should replace the other, or in which order both belong, so it stops and asks. Both lines describe real changes that are now on `main`, so I kept both and deleted the three marker lines.

3. A colleague says "I'll make a PR to get the latest code." What do they mean, and which command is it?

   They mean updating their own copy with the newest commits from GitHub. That is not a pull request: it is `git pull` (a `git fetch` followed by a merge). A pull request goes the other way: it asks for your branch's changes to be merged into someone else's branch, usually `main`.

4. After `1.0.0`: what is the next version for (a) a bug fix, (b) a new `/api/todos/search` endpoint, (c) renaming `/api/todos` to `/api/tasks`?

   (a) `1.0.1`: a bug fix is a PATCH. (b) `1.1.0`: a new endpoint adds a feature without breaking existing clients, a MINOR change. (c) `2.0.0`: renaming the endpoint breaks every client that calls `/api/todos`, a MAJOR change.
