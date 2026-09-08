---
description: Feature gate — tests, commit, two-axis review with a fix loop. Leaves the branch ready; never merges, pushes or opens a PR
argument-hint: [max rounds, default 3]
---

Run after `/speckit-implement`, when the code and tasks are done.

This command takes the feature from "the code is written" to "the branch is
tested, committed and reviewed". **It never merges, never pushes and never
opens a pull request** — not by default, not on request. What happens to the
branch afterwards is decided and done outside this command.

> **Assumes the integration branch is `main`.** If yours is named
> differently, replace `main` throughout this file — it appears in the
> branch guard and in the diff base.

> **About commits.** Neither Spec Kit nor the review skill runs `git commit` —
> in Spec Kit it is an optional `after_implement` hook from the git extension,
> and review sub-agents are read-only by design. **This command commits**, at
> step 3. Without it the review would inspect an empty diff.

---

0. **Branch check.** `git rev-parse --abbrev-ref HEAD`. It must be a feature
   branch like `NNN-name`. If we are on `main`, STOP and say the branch is
   missing, `/start-feature` was never run, and the work is sitting on `main`.

1. **Destructive migration check.** Look at any database migrations in this
   feature — both committed and still in the working tree (`git status`); at
   this point the code is usually uncommitted.

   If any of them drops a column, a table, a type, or a unique index — STOP.
   Begin your answer with "This feature contains a destructive migration — it
   needs a human decision" and list exactly what is dropped. An automated gate
   cannot restore data it has already deleted.

2. **Tests.** Run the project's test suite. Detect the stack from the files
   present, and run every stack the repository actually contains:

   | Marker | Command |
   |---|---|
   | `*.sln`, `*.csproj` | `dotnet test` |
   | `package.json` with a `test` script | `npm test` |
   | `pytest.ini`, `pyproject.toml`, `tests/` | `pytest` |
   | `go.mod` | `go test ./...` |
   | `Cargo.toml` | `cargo test` |
   | `pom.xml` | `mvn -q test` |
   | `Gemfile` with rspec | `bundle exec rspec` |

   If nothing matches, say which command you would run and ask before running
   it. If the suite is empty, say so plainly — an empty suite is not a pass.

   If tests fail, fix the code. If the test itself is wrong rather than the
   code, explain why before changing it. Never silently rewrite a test to match
   current behaviour.

3. **Commit.** The only commit point in this command — on the first round and
   on every later one. Without it the next step is meaningless:
   `/review-feature` builds its diff with `git diff main...HEAD`.

   ```
   git add -A
   git status --short
   ```

   **Look at what is staged.** It should contain only this feature's files —
   source, tests, and the plan and task files under `specs/<branch>/`. Anything
   else: stop and find out why, do not commit it just in case.

   If anything looks like a secret — `.env`, `*.pem`, `*.key`, a local settings
   file, a filename containing `secret` or `password` — STOP, and say so. A
   secret that reaches history is not fixed by deleting the commit; it has to
   be rotated.

   If `git add -A` fails with `does not have a commit checked out`, the
   repository contains a nested git repository. It has to be added to
   `.gitignore`; report it, do not delete anything.

   Then:
   ```
   git commit -m "<branch>: <what was done, one line>"
   ```
   On later rounds this commit carries the review fixes; name it accordingly,
   for example `"<branch>: review fixes"`.

   If there is nothing to commit, STOP and say so. On the first round it means
   `/speckit-implement` changed nothing; on later rounds it means the review
   findings were never addressed. Both are failures, not successes.

4. **Review.** Run `/review-feature`.

   If it stops instead of reviewing — the review skill is not installed, the
   spec file is missing, the branch name does not match the convention, the
   diff is empty — that is a stop for this command too. Do not treat a review
   that never ran as a review that found nothing.

5. If both axes (Standards + Spec) are clean, go to step 8. A report with no
   Spec axis, or one saying the spec was not found, is **not** a clean result:
   find out why and re-run the review.

6. **If there are findings, fix them.** Do not commit here — the fixes are
   committed by step 3 on the next round, as their own commit, so the history
   shows what the review caught.

   Fix the code. Do not redefine the spec and do not edit the project's rules
   to make a finding go away. If a finding points at a genuine conflict with a
   recorded decision rather than a small defect, stop and report the conflict
   instead of resolving it yourself.

7. Back to step 2 — the tests must run again, because a fix can break
   something, and step 3 will commit the fix. This is one of at most N rounds,
   where N is the first argument (default 3), shared between tests and review,
   not counted separately.

   If it still does not converge after that many rounds, STOP. Leave the branch
   as it is, begin your answer with "This feature is stuck after N rounds" and
   list what is still failing.

8. **Stop here.** The branch is committed, tested and reviewed — that is the
   whole job of this command.

   Do not merge. Do not push. Do not open a pull request. Do not switch to
   `main` and do not touch any branch other than the feature branch you are
   on. This holds even if asked mid-run: integrating the work is a separate
   decision, made outside this command, because doing it automatically would
   put code on the integration branch — and possibly into a deploy — without
   anyone choosing that.

9. **Report at the end**, in this shape:

   - the branch name, and that it is committed, tested and reviewed;
   - how many rounds it took, and what the review found and fixed;
   - anything the reader has to decide or do themselves: new environment
     variables or secrets the feature needs, and the fact that the branch is
     not integrated anywhere.

   State plainly that nothing was merged or pushed. Do not suggest what to run
   next — what happens to the branch is not this command's business.
