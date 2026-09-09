---
description: Feature gate — tests, commit, two-axis review with a fix loop. Leaves the branch ready; never merges, pushes or opens a PR
argument-hint: [max rounds, default 3]
---

Run after `/speckit-implement`, when the code and tasks are done.

This command takes the feature from "the code is written" to "the branch is
tested, committed and reviewed". **It does not merge, does not push and does
not open a pull request.** What happens to the branch afterwards is decided and
done outside this command — see step 8 for what that means if you are asked to
integrate while it is running.

> **Assumes the integration branch is `main`.** If yours is named
> differently, replace `main` throughout this file — it appears in the
> branch guard, in the migration scan and in the diff base.

> **About commits.** Neither Spec Kit nor the review skill runs `git commit` —
> in Spec Kit it is an optional `after_implement` hook from the git extension,
> and review sub-agents are read-only by design. **This command commits**, at
> step 3. Without it the review would inspect an empty diff.

## The round budget

`N` = the first argument — `$ARGUMENTS` — or **3** if none was given.

One round is one attempt at the sequence **2 → 3 → 4**. A round that never
reaches step 4 is still a round: if step 2 leaves the suite red, that round was
spent on the tests, and what remains of the budget is what the review gets. The
budget is **shared** between tests and review, not counted separately for each.

Announce `round <k> of N` at the start of every round. When round N ends without
both axes clean — because the tests are still failing, or because the review
still has findings — stop as step 7 describes.

---

0. **Branch check.** `git rev-parse --abbrev-ref HEAD`.

   - `fatal: not a git repository` — STOP. There is no repository here, so
     there is no branch, no diff base and nothing to review. Say so, and check
     this is the directory you meant before anyone creates one.
   - We are on `main` — STOP and say the branch is missing, `/start-feature`
     was never run, and the work is sitting on `main`.
   - Any other name that is not `NNN-name` — STOP and report it. The review
     locates requirements by branch name, so `/review-feature` would stop at
     its own step 1 anyway. Do not rename the branch yourself.

   Then check the diff base exists, because steps 1 and 4 both depend on it:

   ```
   git rev-parse --verify main
   ```

   If that fails, `main` is not a local branch — a fresh clone has only the
   branch it checked out, with the rest under `origin/`. STOP and say so. Do
   not let it slide: `git diff main...HEAD` then dies with
   `fatal: ambiguous argument 'main...HEAD'`, and a fatal error mistaken for an
   empty result reads exactly like "this feature touches no migrations" at step
   1 and like "nothing to review" at step 4. Either fetch the branch
   (`git fetch origin main:main`) or use the name this repository actually
   integrates into.

1. **Destructive migration check.** First enumerate the migrations this feature
   touches — both sides, because at this point the code is usually still
   uncommitted:

   ```
   git diff main...HEAD --name-only
   git status --porcelain -uall
   ```

   `-uall` matters: without it `git status` collapses a whole untracked
   directory into one line (`?? migrations/`), and a feature's first migration
   is very often the file that creates that directory. Listing files instead of
   directories is the difference between scanning the migration and never
   seeing it.

   From those two lists keep the migration files. Where they live depends on
   the stack — `Migrations/*.cs` (EF Core), `alembic/versions/*.py`,
   `prisma/migrations/*/migration.sql`, `db/migrate/*.rb` (Rails),
   `migrations/*.sql`, `*.up.sql` / `*.down.sql` (golang-migrate),
   `**/changelog*.xml` (Liquibase). If this repository keeps them elsewhere,
   find that directory before concluding there are none: "no migrations" is a
   claim about the repository's layout, so check it rather than assume it.

   Read every file you kept, and look for a drop of an object that already
   exists — in raw SQL (`DROP TABLE`, `DROP COLUMN`, `DROP TYPE`, and
   `DROP INDEX` / `DROP CONSTRAINT` on a unique index) and in the ORM wrappers
   for the same thing (EF `DropColumn` / `DropTable`, Alembic
   `op.drop_column` / `op.drop_table`, Rails `remove_column` / `drop_table`,
   a Django or Prisma migration whose generated SQL drops a column).

   If any of them drops a column, a table, a type, or a unique index — STOP.
   Begin your answer with "This feature contains a destructive migration — it
   needs a human decision" and list exactly what is dropped, file by file. An
   automated gate cannot restore data it has already deleted.

   **This stop has no override, deliberately** — there is no flag for it and
   asking again does not unlock it. So say what the two ways forward are, or
   the feature looks permanently stuck:

   - make the migration non-destructive and run this command again. The usual
     shape is expand/contract: add the new object, migrate the data, ship it,
     and drop the old object in a separate later migration once nothing reads
     it any more;
   - or keep the drop and take this feature through commit and review by hand,
     outside this command, having decided about the data yourself — backup,
     retention, and whether anything still depends on what is being dropped.

   Do not edit the migration to make the drop harder to spot, and do not carry
   on through the remaining steps assuming someone will look at it later.

2. **Tests.** Run the project's test suite. Detect the stack from the files
   present, and run every stack the repository actually contains:

   | Marker | Command |
   |---|---|
   | `*.sln`, `*.csproj` | `dotnet test` |
   | `package.json` with a real `test` script | `npm test` |
   | `pytest.ini`, `tox.ini`/`setup.cfg`/`pyproject.toml` configuring pytest, or `tests/` holding `test_*.py` | `pytest` |
   | `go.mod` | `go test ./...` |
   | `Cargo.toml` | `cargo test` |
   | `pom.xml` | `mvn -q test` |
   | `Gemfile` with rspec | `bundle exec rspec` |

   These markers are weaker than they look, so read the file before trusting
   one:

   - a `pyproject.toml` says "Python", not "pytest". Look for a
     `[tool.pytest.ini_options]` section, or pytest among the dev
     dependencies, and for test files that actually exist;
   - a bare `tests/` directory is not a Python marker in a repository with no
     Python in it;
   - `"test": "echo \"Error: no test specified\" && exit 1"` is the npm
     scaffold placeholder, not a suite. It fails by design — treat it as **no
     suite** under the rule below, and do not start "fixing the code" to
     satisfy it.

   If nothing matches, say which command you would run and ask before running
   it.

   If the suite is empty — no test files, or only a placeholder script — say so
   plainly and STOP: an empty suite is not a pass. Mention where it comes from,
   because it is nearly always the same cause: test tasks are optional in Spec
   Kit's task template and have to be asked for at `/speckit-tasks`. Committing
   untested code behind a green checkmark is the failure this command exists to
   prevent.

   If tests fail, fix the code. If the test itself is wrong rather than the
   code, explain why before changing it. Never silently rewrite a test to match
   current behaviour.

   **Fixing tests costs a round.** Once you have made a fix here, that round is
   over: start the next one back at this step and re-run the suite. Never carry
   a red suite into step 3, and never keep fixing past the budget — if the
   suite is still red when round N ends, stop exactly as step 7 says, with the
   failing tests named.

3. **Commit.** The only commit point in this command — on the first round and
   on every later one. Without it the next step is meaningless:
   `/review-feature` builds its diff with `git diff main...HEAD`.

   **Look before touching the index.** Staging is not history and a mis-stage
   is undone with `git restore --staged`, but the point of this check is to
   happen before anything is committed, so make it before anything is staged:

   ```
   git add -A --dry-run
   ```

   Read that list. It should contain only this feature's files:

   - source and tests;
   - the Spec Kit artefacts under `specs/<branch>/` — `plan.md` and `tasks.md`,
     plus whatever else `/speckit-plan` and `/speckit-checklist` generated
     there: `research.md`, `data-model.md`, `quickstart.md`, `contracts/`,
     `checklists/`. `spec.md` was committed by `/start-feature` and will not
     appear again unless it changed.

   Anything else: stop and find out why, do not commit it just in case.

   **Secrets.** Scan the same list for `.env` and `.env.*`, `*.pem`, `*.key`,
   `*.pfx`, `*.p12`, `id_rsa`, `*.keystore`, local settings files
   (`appsettings.*.local.json`, `secrets.json`, `local.settings.json`) and any
   filename containing `secret`, `password`, `credential` or `token`. If one is
   there — STOP, and say so. A secret that reaches history is not fixed by
   deleting the commit; it has to be rotated.

   This scan sees only what would be staged, which makes it a filename check
   over untracked, non-ignored files. Two things it cannot see — say so rather
   than imply coverage you do not have:

   - a secret already covered by `.gitignore` never shows up here. That is the
     intended outcome, not a miss; `git check-ignore -v <path>` confirms a
     particular file is ignored on purpose rather than missing by accident;
   - a secret pasted *inside* an ordinary file has an ordinary filename. If the
     diff touches configuration, connection strings or CI definitions, read
     those hunks.

   If the list is clean, stage it and confirm what you got matches what you
   just read:

   ```
   git add -A
   git status --short
   ```

   If `git add` fails with `does not have a commit checked out`, the repository
   contains a nested git repository. It has to be added to `.gitignore`; report
   it, do not delete anything.

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
   something, and step 3 will commit the fix. That is the next round, drawn
   from the same budget as any round already spent on tests (see **The round
   budget** above).

   If round N ends without both axes clean, STOP. Leave the branch as it is,
   begin your answer with "This feature is stuck after N rounds" and list what
   is still failing — failing tests by name, remaining findings by axis. Do not
   spend an extra round because the next one would probably converge.

8. **Stop here.** The branch is committed, tested and reviewed — that is the
   whole job of this command.

   Do not merge. Do not push. Do not open a pull request. Do not switch to
   `main` and do not touch any branch other than the feature branch you are
   on. Integrating the work is a separate decision, made outside this command,
   because doing it automatically would put code on the integration branch —
   and possibly into a deploy — without anyone choosing that.

   If you are asked to integrate while this command is running, it still ends
   here: finish the report, say that integrating is outside this gate, and let
   that decision be made on its own with the report in hand. This is a scope
   boundary, not a refusal — nothing stops the same person from merging the
   branch a moment later, deliberately.

9. **Report at the end**, in this shape:

   - the branch name, and that it is committed, tested and reviewed;
   - how many rounds it took out of the budget, and what the review found and
     fixed;
   - anything the reader has to decide or do themselves: new environment
     variables or secrets the feature needs, and the fact that the branch is
     not integrated anywhere.

   State plainly that nothing was merged or pushed. Do not suggest what to run
   next — what happens to the branch is not this command's business.
