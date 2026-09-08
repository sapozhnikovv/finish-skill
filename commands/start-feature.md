---
description: Create the feature branch named after the spec directory, and commit the spec. Run right after /speckit-specify
---

Run this **immediately after `/speckit-specify`**, before anything else.

> **Assumes the integration branch is `main`.** If yours is named
> differently, replace `main` in the state check below.

## Why this command exists

Spec Kit does not create a git branch. It creates `specs/NNN-name/` and records
the path in `.specify/feature.json`. Branch creation lives in the optional git
extension's `before_specify` hook; if `.specify/extensions.yml` does not exist,
nothing runs and no branch appears — silently.

Without a branch, everything built on "one feature, one branch" breaks:
`/review-feature` cannot locate the spec, `/finish-feature` has nothing to
review, and the work lands directly on `main` with no gate.

This command closes that gap and guarantees the important part: **the branch
name matches the spec directory name exactly.** That is how the review finds
the requirements.

## Steps

1. Read `.specify/feature.json` and take `feature_directory`. If the file or the
   key is missing, stop: the spec does not exist yet, run `/speckit-specify`
   first.

2. The branch name is the last path segment — `specs/003-user-auth` → branch
   `003-user-auth`. Check that it looks like `NNN-short-name`; if it does not,
   report it and stop. Do not rename anything yourself.

3. Check the repository state:
   - `git rev-parse --abbrev-ref HEAD` — we should be on `main`. Already on a
     branch with this exact name: say so and stop. On a *different* feature
     branch: stop — the previous feature is still open. Finish it with
     `/finish-feature` and integrate it before starting a new one.
   - `git status --porcelain` — there must be no changes to **tracked** files.
     The new spec files are untracked, which is expected and fine. Uncommitted
     code changes belong to earlier work: stop and show them.

4. Create the branch:
   ```
   git checkout -b <branch-name>
   ```
   If it already exists, `git checkout <branch-name>` instead.

5. **Commit the spec**, so the branch has a first commit of its own:
   ```
   git add specs/<branch-name>
   git status --short
   git commit -m "<branch-name>: spec"
   ```
   Check the index before committing: only files under `specs/` belong here.
   Anything else — stop and find out why.

   A branch with no commits is indistinguishable from `main`, and
   `git diff main...HEAD` on it is empty. An empty branch leaves the review
   with nothing to look at.

   If there is nothing to stage because the spec is already committed, say so
   and carry on — this command is safe to run again after an interrupted
   attempt, and it never creates a second branch or a second commit.

6. Verify: the current branch matches the spec directory, the spec is
   committed, `git status` is clean.

7. Report in one line: branch `<name>` created, spec committed, next is
   `/speckit-clarify`.

## Do not

- Do not rename the spec directory to match a branch. Spec Kit picks the name;
  the branch follows it, never the other way around.
- Do not create the branch before `/speckit-specify` — the number is not known
  until then.
- Do not commit anything but `specs/`. Code, plan and tasks are committed by
  `/finish-feature`, together with the implementation.
- Do not merge anything here, and do not delete the previous feature branch —
  neither this command nor `/finish-feature` integrates branches by default.
