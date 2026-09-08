---
description: Review the current feature's diff with the two-axis code-review skill, bound to the Spec Kit spec file
argument-hint: [fixed point, default main]
---

The bridge between two independent tools — Spec Kit (which owns the feature
cycle) and Matt Pocock's two-axis `code-review` skill (which owns the review).
Nothing in either tool is reconfigured; this command only passes a file path
between them.

> **Assumes the integration branch is `main`.** If yours is named
> differently, replace `main` — here it is the default diff base.

1. Determine the current branch: `git rev-parse --abbrev-ref HEAD`. It must
   follow the Spec Kit convention `NNN-feature-name`. If it does not, say so
   and stop — do not guess.

2. Locate the spec: `specs/<branch>/spec.md`. If there is no such file, stop
   and report it. Never run the review blind, without a source of requirements.

3. Verify the diff is not empty: `git diff <fixed-point>...HEAD --stat`, where
   fixed-point is $ARGUMENTS (default `main`). An empty diff means there is
   nothing to review: stop and say so, rather than reporting a clean result.

4. Run the **two-axis `code-review` skill from Matt Pocock's set**.

   Do not confuse it with the similarly named skill built into Claude Code —
   they are different tools. The one you want spawns **two parallel sub-agents**
   and reports under two headings: **Standards** (repository conventions plus a
   baseline of code smells) and **Spec** (unimplemented, partially implemented
   or unrequested behaviour).

   Skill names and prefixes change between versions of the set. Find the actual
   name in the list of available skills. If no such skill is present, stop and
   report that the set is not installed — do not substitute the built-in
   `/code-review`, it is a different kind of review.

   Pass explicitly to the skill:
   - fixed point = $ARGUMENTS (default `main`);
   - the requirements source for the Spec axis — the `specs/<branch>/spec.md`
     found in step 2;
   - the standards sources — whatever this repository uses for conventions
     (for example `CLAUDE.md`, a constitution file under `.specify/memory/`,
     contributing guidelines, architecture notes referenced by the plan).

5. Print the combined report as-is, both axes, unabridged. If either axis found
   a problem, say so **at the start** of your answer; do not bury it.

6. Check the report itself: if there is **no** Spec-axis section, or it says the
   spec was unavailable, that is a broken setup, not a clean review. Report it
   as a problem rather than as a pass.

This command only reports. Fixing what it finds, and committing the fixes, is
the job of `/finish-feature`.
