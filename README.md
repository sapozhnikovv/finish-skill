# finish-skill

A closing gate for the [Spec Kit](https://github.com/github/spec-kit) feature
cycle: run the tests, commit the work, review it on two axes, fix what the
review found — and stop with the branch ready. It never merges, never pushes
and never opens a pull request; integrating the branch is a separate decision,
made outside this package.

The review is [Matt Pocock's `code-review` skill](https://github.com/mattpocock/skills),
which reports on two independent axes:

- **Standards** — repository conventions plus a baseline of code smells;
- **Spec** — whether the code matches the written requirements: unimplemented,
  partially implemented, or unrequested behaviour.

This package is the wiring that makes that review run *inside* the Spec Kit
cycle: against the right spec file, on a diff that actually exists, with a fix
loop around it.

Three markdown files. No daemon, no config, no generated state.

---

## Why this exists

Spec Kit generates specs, plans and tasks. It deliberately does **not** touch
git: branch creation lives in a `before_specify` hook, committing in an
`after_implement` hook, and both come from an optional git extension. If that
extension is not installed, `.specify/extensions.yml` does not exist, the hooks
never fire, and nothing in the cycle creates a branch or a commit.

Nothing warns you. The failure is silent and it looks exactly like success:

| What is missing | What you see |
|---|---|
| The branch | Work lands directly on `main`, with no message |
| The commit | `git diff main...HEAD` is empty, so any review reports "no issues found" |
| The commit | A later `git merge` answers `Already up to date` |

A review that inspects an empty diff passes every single time. For an automated
gate that is the worst possible outcome: a green light that means nothing. So
this package owns the git it needs, and every action has exactly one owner:

| Action | Command | When |
|---|---|---|
| Create the branch | `/start-feature` | right after `/speckit-specify` |
| Commit the spec | `/start-feature` | same run |
| Commit code and tests | `/finish-feature` | after tests, **before** the review |
| Commit review fixes | `/finish-feature` | next loop round, as its own commit |
| Merge, push, pull request | **nobody, ever** | outside this package — you do it, when you decide to |

If you *do* install Spec Kit's git extension, drop `/start-feature` — but first
check that the extension names the branch exactly like the spec directory. Spec
Kit states plainly that the two names are independent, and the review locates
requirements by branch name.

---

## What you gain over plain Spec Kit

Plain Spec Kit takes you from an idea to a spec, a plan, tasks and an
implementation, and stops there. Everything after "the code is written" is left
to you — which is exactly where a solo developer, or a non-technical owner
driving an agent, is weakest.

| | Plain Spec Kit | With this gate |
|---|---|---|
| **Review** | none in the cycle | two-axis review on every feature |
| **Requirements drift** | nothing compares code to spec | the Spec axis reads `specs/<branch>/spec.md` and reports what is missing or unrequested |
| **Findings** | you read a report and fix by hand | fixed automatically, re-tested, up to N rounds |
| **Tests** | optional in the task template, easy to skip | run as a gate before anything is committed |
| **Git** | untouched: no branch, no commit | one branch per feature, commits in a fixed order |
| **Empty diff** | reviewed as "clean" | refused with an explicit error |
| **Destructive migrations** | invisible | the gate stops and asks a human |
| **Secrets** | whatever `git add -A` picks up | the index is inspected before every commit |
| **History** | flat, if it exists at all | spec → implementation → review fixes |
| **Stuck feature** | silent partial work | explicit stop after N rounds, branch intact |
| **Integration** | your call | still your call — the gate stops before it |

Three properties matter more than any single row.

**The failure modes are loud.** Every way this cycle can go wrong — no branch,
no commit, empty diff, missing Spec axis, destructive migration, no
convergence — ends in a stop with an explanation instead of a green checkmark.
That is what makes the cycle usable by someone who cannot audit the result
themselves.

**Nothing reaches the integration branch by itself.** The gate prepares; you
integrate. An agent that merges on its own can put code — and a deploy — in
front of users while you are still reading its summary.

**Nothing is coupled.** Spec Kit writes `specs/` and `.specify/feature.json`.
The review skill is read-only. Only these three files touch git. Delete them
and you are back to plain Spec Kit, with nothing to unwind.

---

## Requirements

- [Spec Kit](https://github.com/github/spec-kit) initialised in the repository
  (`specify init --here`), with any integration that provides `/speckit-*`
  commands.
- [Matt Pocock's skill set](https://github.com/mattpocock/skills), specifically
  the **two-axis** `code-review` skill:
  ```bash
  npx skills add mattpocock/skills --agent claude --skill code-review
  ```
  Claude Code ships a built-in skill with a similar name — they are different
  tools. The one you want spawns two parallel sub-agents and reports under two
  headings.
- `git`, and a repository with at least one commit. Nothing else — no runtime,
  no package manager, no build step.

---

## Install

### Option 1 — copy the files

The whole package is three markdown files in `commands/`. Copy them into your
repository's `.claude/commands/`:

```bash
# from the root of your Spec Kit repository
mkdir -p .claude/commands
cp /path/to/finish-skill/commands/*.md .claude/commands/
```

```powershell
New-Item -ItemType Directory -Force .claude\commands | Out-Null
Copy-Item C:\path\to\finish-skill\commands\*.md .claude\commands\ -Force
```

Dragging the three files into `.claude/commands/` in a file manager works just
as well; there is nothing magic about the copy.

### Option 2 — pull it from GitHub in one command

```bash
git clone --depth 1 https://github.com/sapozhnikovv/finish-skill /tmp/finish-skill \
  && mkdir -p .claude/commands \
  && cp /tmp/finish-skill/commands/*.md .claude/commands/ \
  && rm -rf /tmp/finish-skill
```

```powershell
git clone --depth 1 https://github.com/sapozhnikovv/finish-skill "$env:TEMP\finish-skill"
New-Item -ItemType Directory -Force .claude\commands | Out-Null
Copy-Item "$env:TEMP\finish-skill\commands\*.md" .claude\commands\ -Force
Remove-Item -Recurse -Force "$env:TEMP\finish-skill"
```

A single file, without cloning:

```bash
curl -fsSL https://raw.githubusercontent.com/sapozhnikovv/finish-skill/main/commands/finish-feature.md \
  -o .claude/commands/finish-feature.md
```

### Check that it took

Start your agent in the repository and run `/finish-feature` with no feature in
progress. It should stop at step 0 and say the branch is missing — the correct
answer, and proof the file was picked up.

### Re-installing is safe

The three files are self-contained. Installing again overwrites them with the
current version and does nothing else: nothing is appended to another file,
nothing is registered, no state is generated, no cache is written. Running any
install command twice, or ten times, leaves exactly the same three files.

One thing to know: **an overwrite replaces your local edits.** If you changed a
file after installing — pinned the test command, renamed the main branch — copy
it aside first, or check afterwards:

```bash
git diff .claude/commands/
```

To uninstall, delete the three files. Nothing else remains.

---

## Use

```
/speckit-specify <what you want to build>
/start-feature            ← branch + commit of the spec
/speckit-clarify
/speckit-checklist        ← then resolve the checklist items, see below
/speckit-plan
/speckit-tasks            ← ask for tests explicitly; they are optional in the template
/speckit-analyze
/speckit-implement
/finish-feature           ← tests → commit → review → fixes → stop, branch ready
```

After that the branch is yours: integrating it — merge, push, pull request —
happens outside this package, the way your project already does it.

`/finish-feature` takes one optional argument: the maximum number of
fix-and-retry rounds, default `3`. For example, `/finish-feature 5`.

Two details surprise everyone, and neither is caused by this package:

- **Tests are optional in Spec Kit's task template.** If you do not ask for
  them at `/speckit-tasks`, no test tasks are generated — and a gate that runs
  an empty suite passes.
- **Checklists from `/speckit-checklist` start unchecked on purpose.** Marking
  them is the reviewer's call, so `/speckit-implement` reports `FAIL` until a
  human resolves them. Walk through them before you reach the code.

---

## Worked example

A small feature, from an empty branch to a branch that is ready to integrate.

### 1. Spec, then branch

```
> /speckit-specify An HTTP endpoint that returns a random city from a fixed list

  Created specs/001-random-city/spec.md

> /start-feature

  Branch 001-random-city created, spec committed.
  Next: /speckit-clarify
```

```console
$ git log --oneline
c3f001a (HEAD -> 001-random-city) 001-random-city: spec
c206e9a initial commit
```

### 2. Run the cycle, then the gate

```
> /finish-feature

  Step 0  branch 001-random-city — ok
  Step 1  no migrations in this feature
  Step 2  dotnet test — 7 passed
  Step 3  staged: src/CityEndpoint.cs, tests/CityEndpointTests.cs,
          specs/001-random-city/{plan,tasks}.md
          committed "001-random-city: random city endpoint"
  Step 4  running /review-feature

    Standards — 1 finding
      Random is constructed inside the request handler, so selection
      cannot be tested. Inject the randomness source.
    Spec — 1 finding
      FR-007 requires GET /city/all to return all three cities;
      the endpoint is not implemented.

  Step 6  fixing both findings
  Step 2  dotnet test — 9 passed
  Step 3  committed "001-random-city: review fixes"
  Step 4  running /review-feature
    Standards — clean
    Spec — clean

  Branch 001-random-city is committed, tested and reviewed. Two rounds:
  the review found two issues, both fixed and re-tested.
  Nothing was merged or pushed — the branch is not integrated anywhere.
  No new environment variables are required.
```

### 3. The history it leaves

```console
$ git log --oneline --graph --all
* 331883b (HEAD -> 001-random-city) 001-random-city: review fixes
* 67e6331 001-random-city: random city endpoint
* c3f001a 001-random-city: spec
* c206e9a (main) initial commit
```

The same three commits, in the same order, on every feature: what was asked
for, what was built, what the review caught. A month later that history still
answers "why is this line here" — and `main` is untouched until you say so.

### When it refuses

Forgot `/start-feature`:

```
> /finish-feature

  Stopped at step 0: we are on main, so no feature branch exists.
  Run /start-feature — the current work is sitting on main.
```

A migration that drops a column:

```
> /finish-feature

  This feature contains a destructive migration — it needs a human decision.
  Migration 20260913_RemoveLegacyFlag drops column users.legacy_flag.
  The branch is untouched.
```

Three rounds without convergence:

```
> /finish-feature

  This feature is stuck after 3 rounds.
  Still failing: Spec axis — FR-004 (email is not normalised before lookup).
  The branch is left as it is.
```

In every one of these the branch is intact and nothing was integrated. That is
the design: an explicit stop beats a silent pass.

---

## What `/finish-feature` does

```
0  refuse to run on main — the branch is missing
1  stop if a migration drops a column, table, type or unique index
2  run the test suite
3  commit — the only commit point, and it happens before the review
4  run /review-feature — the two-axis review
5  clean on both axes → go to 8
6  otherwise fix the findings (no commit here)
7  back to 2, up to N rounds, then stop and report
8  stop: no merge, no push, no pull request, no switching branches
9  report the branch, the rounds, the findings, any new env vars
```

The single commit point is deliberate. The loop returns to the test step, which
sits *before* the commit step, so review fixes are committed by step 3 on the
next round — as their own commit, which is what keeps the history readable.
Committing inside step 6 as well would leave step 3 facing an empty index on
the second round, and the cycle would stop with a bogus "nothing to commit".

---

## FAQ

**Does it merge, push or open a pull request?**
No, and there is no flag that makes it. That is the point: a gate that
integrates by itself can put code — and a deploy — in front of users while you
are still reading its summary. It does exactly two things to your repository:
commits the feature branch, and commits the review fixes. Everything after that
is yours.

If you want the merge automated, do it outside this command — a shell alias or
a CI job is the right place, because then it is visibly your decision, not a
side effect of a review.

**Can I run `/finish-feature` twice in a row?**
Yes. It is re-entrant and will not redo work:

- Back on `main` → stops at step 0 instead of committing to `main`.
- Still on the feature branch with nothing new → stops at step 3, because an
  empty index means either the implementation did nothing or the findings were
  never addressed.

Re-running it after fixing something by hand is a normal way to work.

**Does it conflict with Spec Kit?**
No. Spec Kit writes to `specs/` and `.specify/feature.json` (excluded by its
own `.gitignore`). The review skill is read-only. Only these commands touch
git, and they register no hooks, so a future Spec Kit git extension would not
collide — though if you install one, drop `/start-feature` so that a single
owner creates the branch.

**Which test command does it run?**
It detects the stack — `dotnet test`, `npm test`, `pytest`, `go test ./...`,
`cargo test`, `mvn test`, `bundle exec rspec` — and runs each one it finds in a
mixed repository. To pin the command, edit step 2 of
`.claude/commands/finish-feature.md`; after installation that file is yours.

**My main branch is not called `main`.**
Replace the name in all three command files; each one says so at the top. It
appears in the step-0 guard and in the diff base.

**Can I run `/start-feature` twice?**
Yes. On a second run it finds the branch already checked out, or the spec
already committed, and says so instead of duplicating anything. The same is
true after an interrupted attempt.

**Do I have to use `/start-feature`?**
Only if nothing else creates the branch. Its second job matters just as much:
the first commit. A branch without commits is indistinguishable from `main`,
and the review then has nothing to look at.

**Does it work outside Spec Kit?**
`/review-feature` needs `specs/<branch>/spec.md` — that is the Spec axis's
source of truth. Any workflow that puts a spec there and names branches to
match will work.

## License

MIT — see [LICENSE](LICENSE).

Spec Kit and Matt Pocock's skill set are separate projects under their own
licenses. This package neither vendors nor modifies them; it only calls them.
