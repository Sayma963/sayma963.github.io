---
title: "Taming Git Merge Conflicts: What They Are, Why They Happen, and How I Actually Fix Them"
date: 2026-07-03 10:00:00 +0600
categories: [Software Engineering, Version Control]
tags: [git, merge-conflicts, version-control, best-practices]
pin: false
---

## The Panic Moment

Every developer remembers their first merge conflict. You run `git merge` or `git pull`, and instead of a clean "Already up to date," your terminal throws this at you:

```
Auto-merging src/app.js
CONFLICT (content): Merge conflict in src/app.js
Automatic merge failed; fix conflicts and then commit the result.
```

It looks scary. It isn't. A merge conflict just means Git found two changes to the same piece of a file and refuses to guess which one you actually want. That's it — Git being cautious, not broken.

This post is my personal take on merge conflicts: what causes them, and — more importantly — how to fix them in a way that actually matches *why* they happened, instead of just making the red text go away.

## What Is a Merge Conflict, Really?

Git tracks history as a graph of commits. When you merge two branches, Git tries to automatically combine changes using a three-way merge: it looks at the common ancestor commit, and the two divergent versions, and stitches them together line by line.

A conflict occurs when **both branches changed the same lines (or nearby lines) in a way Git can't reconcile automatically**. Git marks the disputed region directly in the file:

```
<<<<<<< HEAD
const taxRate = 0.15;
=======
const taxRate = 0.18;
>>>>>>> feature/update-tax
```

`HEAD` is your current branch's version; the bottom section is the incoming branch's version. Git is handing the decision back to a human because it has no way to know which value is correct.

## Root Causes (Not Just Symptoms)

Treating every conflict the same way is the mistake most beginners make. In my experience, conflicts trace back to a handful of distinct root causes, and each deserves a different fix.

### 1. Parallel edits to the same line(s)

**Cause:** Two people edit the same logic independently — classic when a team member changes a config value or fixes a bug on their branch while you fix the same spot differently on yours.

**Example:** Branch A changes `taxRate = 0.15`, branch B changes it to `0.18`. Neither is "wrong" syntactically — Git just can't pick a winner.

### 2. Structural/formatting divergence

**Cause:** One branch reformats a file (e.g., a linter or Prettier run) while another branch makes a substantive change nearby. Every line looks "changed" even though only whitespace moved.

**Example:** Teammate runs `prettier --write` across the repo on `main`, changing indentation on 200 files. Meanwhile you've been working on a long-lived feature branch touching 3 of those same files. Merging now shows conflicts on nearly every line, even though your actual logic change is tiny.

### 3. Renamed or moved files

**Cause:** One branch renames/moves a file while another branch edits the file's original path. Git tries to detect renames heuristically, but it can misfire, especially with big changes alongside the rename.

**Example:** You rename `utils.js` → `helpers/stringUtils.js` on your branch; a teammate edits `utils.js` on theirs. Git may not link the edit to the new path.

### 4. Long-lived, divergent branches

**Cause:** This is the *meta* root cause behind most conflicts — a branch that isn't rebased/merged with `main` for weeks accumulates so much drift that even unrelated changes start colliding.

**Example:** A feature branch sits untouched for a month while five other PRs land on `main`. By the time you merge, you're reconciling a month of team history in one sitting.

## Fixing Conflicts to Match the Root Cause

This is the part most tutorials skip. Resolving `<<<<<<<` markers is mechanical — the real skill is choosing the *right* resolution strategy for *why* the conflict happened.

**For parallel logic edits (#1):** Don't blindly pick "yours" or "theirs." Understand what each branch was trying to achieve and write the version that satisfies both intents — often that means talking to the other author for five minutes rather than guessing.

```bash
git status                # see conflicted files
# open file, resolve manually, keeping correct logic
git add src/app.js
git commit
```

**For formatting divergence (#2):** Don't resolve line-by-line — that's a losing battle. Instead, align formatting *before* merging: rebase your branch, run the same formatter, then merge so only real logic differs.

```bash
git checkout feature-branch
git rebase main           # or merge main in first
npx prettier --write .    # match main's formatting
git add -A && git commit -m "chore: reformat to match main"
```

**For renamed files (#3):** Use `git log --follow` and `git diff --find-renames` to confirm Git tracked the rename correctly, then manually re-apply the teammate's edit to the new path if Git split it into a delete+add instead of a move.

**For long-lived branch drift (#4):** The fix isn't a Git command — it's a workflow change. Merge or rebase from `main` frequently (daily, if the team is active) so conflicts stay small and contextual instead of turning into an archaeological dig once a month.

```bash
git fetch origin
git rebase origin/main     # do this often, not once at the end
```

## What NOT to Do (and Why)

- **Don't run `git checkout --ours` or `--theirs` as a default habit.** It silently discards one side's work. Fine for auto-generated files (lockfiles, build artifacts); dangerous for actual logic, because you might delete a bug fix without realizing it.
- **Don't resolve conflicts without reading both sides.** Skimming past `<<<<<<<` markers and keeping "whatever compiles" often reintroduces bugs the other branch had already fixed.
- **Don't let branches go stale to "avoid conflicts now."** Delaying a merge doesn't prevent the conflict — it just moves it later and makes it bigger (see root cause #4). Small, frequent merges are strictly cheaper than one giant one.
- **Don't force-push over a shared branch to "make conflicts disappear."** This rewrites history teammates rely on and can destroy their work. Use `git rebase` locally, and only force-push (`--force-with-lease`, never plain `--force`) on branches you alone own.
- **Don't treat every conflict as a coin flip.** As shown above, formatting conflicts and logic conflicts need entirely different fixes. Applying a formatting fix (reformat before merge) to a logic conflict won't help, and vice versa.

## My Personal Workflow

When I hit a conflict, I ask one question first: *what actually diverged — code, formatting, or file structure?* That answer decides my next move, not the conflict markers themselves. In practice:

1. `git fetch` + rebase early and often — most conflicts I see today are small because I never let branches drift for weeks.
2. When a conflict does appear, I read both versions in full before touching anything.
3. I only use `--ours`/`--theirs` for generated files I don't hand-edit (e.g., `package-lock.json`), and I regenerate them afterward rather than trusting either side blindly.
4. I test after every resolution — a conflict that "looks" resolved can still be logically broken.

Merge conflicts aren't a sign something went wrong. They're Git doing exactly its job: refusing to guess when humans disagree. Match your fix to the cause, and they stop being scary.
