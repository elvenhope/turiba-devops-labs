# Lab 3 · Branch, review, release

<!-- Copy this file to lab3/README.md (Lab 3, step 1.4) and fill it in as you go.
     Keep the headings exactly as they are: the grader looks for them by name.
     Put command output in the ```text blocks. -->

- **Fork:** https://github.com/<you>/turiba-devops-labs
- **Release:** https://github.com/<you>/turiba-devops-labs/releases/tag/v0.1.0
- **Reviewer:** @<reviewer>

## Part 1 · Set up

Output of `ssh -T git@github.com` (or `gh auth status`):

```text
Hi elvenhope! You've successfully authenticated, but GitHub does not provide shell access.
```

Output of `git remote -v`:

```text
origin  git@github.com:elvenhope/turiba-devops-labs.git (fetch)
origin  git@github.com:elvenhope/turiba-devops-labs.git (push)
upstream        https://github.com/mleitass/turiba-devops-labs.git (fetch)
upstream        https://github.com/mleitass/turiba-devops-labs.git (push)
```

## Part 2 · Playground

The hash in `.git/refs/heads/main` (step 2.3):
17bd1eb275ef10351548055e1e5c455695fdc5a3

The first line of `git cat-file -p HEAD` (step 2.3):
tree 6246df39db4c0193d1ee086b4532c335d9cc4ad0

The conflict block you saw in `config/app.env` (step 2.6):

```text
<<<<<<< HEAD
REPLICAS=2
=======
REPLICAS=3
>>>>>>> feature/scale-up
```

Which value you kept, and why:
I kept Replicas 3, cause it was newer.

The final `git log --oneline --graph --all` (step 2.6):

```text

*   e00131d (HEAD -> main) Merge branch 'feature/scale-up'
|\  
| * 2b5a3d8 (feature/scale-up) feat: scale to 3 replicas
* | 724e990 feat: scale to 2 replicas
|/  
*   3490bbf Merge branch 'feature/logging'
|\  
| * 480d936 (feature/logging) feat: add log level
* | 895cb48 docs: add README
|/  
* 6be7db0 (feature/healthcheck) feat: add health message
* 17bd1eb feat: print done after deploy
* 70a327c feat: add app config
* 864e021 feat: add deploy script
```

## Part 3 · Rescue missions

### m1 (worked example)

- PASS line: `PASS m1 cfd6fe2d`
- Symptom: deploy.sh was changed and broken; nothing staged or committed
- Command: git restore deploy.sh
- Why this command, and not a more dangerous one: it resets only this one file to the last commit. git reset --hard would also throw away every other uncommitted change in the repo, and deleting the folder would lose the history too. Careful: the broken edit is gone for good, because it was never committed (slide 17).

### m2

- PASS line: PASS m2 c0ab8893
- Symptom: env was staged but not commited 
- Command: git restore --staged .env
- Why this command, and not a more dangerous one: it's the simplest one that smoothly unstages the commit, and then we can add .gitignore

### m3

- PASS line:PASS m3 7c2c86bb
- Symptom: the commit message was wrong and some file was not added to the commit
- Command: git add the forgotten file, and then git commit --amend -m 
- Why this command, and not a more dangerous one: reseting or soft may have lost other work

### m4

- PASS line: PASS m4 4401dd7b
- Symptom: commits were on main, but should have been in a feature/cache branch
- Command: git branch feature/cache to create the new branch with the current commits on there, and then we just hard reset the main one to the origin/head since nothing was pushed.
- Why this command, and not a more dangerous one: i dont know, it was the simplest approach i guess, maybe we could have just cherrypicked the commits but that sounds harder

### m5

- PASS line: bf4b2e76
- Symptom: A bad commit that got on the remote, and teammate already has it. 
- Command: git revert to fix the bad commit with a new commit so the teammate can just do a git pull and doesn't mess up the history
- Why this command, and not a more dangerous one: git reset --hard would have trouble with the teammates localhistory requiring merges and unrelated historis and all that.

### m6

- PASS line: PASS m6 2e3e540b
- Symptom: accidentally hard removed 3 commits
- Command: we could have used something like git reset -- hard HEAD@{1}, but i used the gentler way of git branch rescue, and then merging them.
- Why this command, and not a more dangerous one: git branch rescue is less risky than another hard reset i guess

### m7

- PASS line: PASS m7 0ceeffe3
- Symptom: A commit made on a tag didn't belong to anything
- Command: So keep the commit you have to assign it a branch or something. So git switch -c hotfix did it
- Why this command, and not a more dangerous one: not sure

### m8

- PASS line: PASS m8 f85b6048
- Symptom: there was a bad commit somewhere
- Command: to find it we used git bisect, marked the last known good one, and current bad one. and then git bisect searched all the commits in between to find the exact commit where the result turned from good to bad.
- Why this command, and not a more dangerous one: no clue.

<!-- Stretch: add ### m7 and ### m8 the same way if you solved them. -->

## Part 4 · Protected main and pull requests

The rejected push to `main` (step 4.2):

```text
To github.com:elvenhope/turiba-devops-labs.git
 ! [remote rejected] main -> main (push declined due to repository rule violations)
error: failed to push some refs to 'github.com:elvenhope/turiba-devops-labs.git'

```

- PR #… (`.gitignore`): https://github.com/elvenhope/turiba-devops-labs/pull/1
- PR #… (`/healthz` docs, with the conflict): https://github.com/elvenhope/turiba-devops-labs/pull/2

## Part 5 · Release

- Release: https://github.com/elvenhope/turiba-devops-labs/releases/tag/v0.1.0
- Why `0.1.0` and not `1.0.0`? 
its a minor feature?

## Answers

1. In mission 5, why `git revert` and not `git reset --hard` plus a force push?
git revert creates a new commit that undoes the previous commit. it adds to the history. git reset --hard resets the history.

2. Why couldn't Git resolve the `CHANGELOG.md` conflict by itself, and how did you decide what to keep?

Because the commits were both saying two different things and git didn't know which to keep timeline wise. but we needed business logic to keep both.

3. A colleague says "I'll make a PR to get the latest code." What do they mean, and which command is it?
they mean git fetch and then git merging it


4. After `1.0.0`: what is the next version for (a) a bug fix, (b) a new `/api/todos/search` endpoint, (c) renaming `/api/todos` to `/api/tasks`?

(A) 1.0.1
(B) 1.1.0
(C) 2.0.0