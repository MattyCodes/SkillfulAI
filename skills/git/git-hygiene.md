---
name: git-hygiene
description: >-
  House git hygiene: keep shared/main-branch history clean by favoring
  squash-and-merge over rebasing shared branches, and writing commit messages
  — especially PR titles, which become the squash commit — as Conventional
  Commits (https://www.conventionalcommits.org/en/v1.0.0/). Use this whenever
  writing a commit message or PR title, deciding how to merge a pull request,
  deciding whether to rebase a branch, or setting up branch-protection /
  merge-strategy / PR-template defaults for a repo. Apply it even if the user
  doesn't say "conventional commits" or "git hygiene" out loud.
---

# Git hygiene

> **Scope — how commits land on shared/main branches.** This covers merge
> strategy and commit-message format for history that other people read and
> build on. It has nothing to say about branching models, release/tag
> automation, or day-to-day git commands beyond these two concerns. If the task
> falls outside "Use this skill for" below, this skill doesn't apply.

**Use this skill for**

- Writing a commit message or a pull request title.
- Deciding how to merge a PR (squash / rebase / merge commit).
- Deciding whether it's safe to rebase a given branch.
- Configuring a repo's default merge-strategy, branch protection, or PR title
  template.
- Reviewing any of the above.

**Do not use this skill for**

- Choosing a branching model (trunk-based, git-flow, etc.) — this assumes
  whatever trunk-based setup is in place and only governs how work lands on it.
- Release tagging, changelog generation, or version-bump automation — those
  often *consume* Conventional Commits but are a separate concern.
- Ordinary git usage unrelated to history hygiene.

---

## 1. Avoid rebasing shared branches; squash-and-merge into main

The goal is a **trunk history that's linear and legible** — `git log --oneline
main` reads as one line per unit of work. There are two ways people usually try
to get there, and both have a real cost:

- **Rebase-and-merge** linearizes history, but it does it by rewriting commits.
  Rebase a branch anyone else has pulled, or that a reviewer has already
  checked out, and their next pull conflicts or silently diverges. It also
  needs a force-push, which is unforgiving if it goes wrong.
- **A plain merge commit** avoids rewriting anything, but trunk ends up with
  every intermediate "wip", "fix typo", "address review comments" commit from
  the branch, forever.

**Squash-and-merge gets both properties at once**: trunk gets exactly one
commit per merged PR — clean and linear — and nothing that's already shared
ever gets rewritten. That's why it's the default here.

**In practice:**

- **On `main`/trunk: never rebase it, never force-push to it.** Merge every PR
  with squash-and-merge.
- **On your own branch, before it's shared:** rebasing onto latest trunk to
  stay current, or an interactive rebase to tidy your own commits before
  opening the PR, is fine — nobody else has based work on it yet, and the
  squash on merge will clean it up regardless.
- **Once a branch is shared** — someone else has pulled it, or it's an open PR
  reviewers have already checked out — stop rebasing it. Keep adding commits
  and let the eventual squash-merge handle the cleanup. A reviewer re-reviewing
  a force-pushed branch has to start over; don't make them.
- **Never rebase or force-push a branch someone else is actively working on or
  has branched from.** This is the one hard rule — everything else is judgment.

"Avoid rebasing where possible" means exactly that: reach for squash-and-merge
first, and only rebase your own not-yet-shared work when you have a specific
reason to.

## 2. Conventional Commits on main/shared branches

Because merges into main are **squashed**, the commit message that actually
matters is the squash commit — and both GitHub and GitLab default that
commit's subject to the **PR title**. So: write the PR title as a Conventional
Commit line; the WIP commits inside the branch don't need to follow the format,
since they're squashed away.

**Format:**

```
<type>[optional scope][!]: <description>

[optional body]

[optional footer(s)]
```

- **`feat`** — a new feature. **`fix`** — a bug fix. These are the two types
  the spec gives special meaning (SemVer MINOR / PATCH); other types are
  explicitly permitted and commonly include `build`, `chore`, `ci`, `docs`,
  `style`, `refactor`, `perf`, `test`.
- **Scope** is an optional parenthesized noun for what part of the codebase is
  affected: `feat(auth): ...`, `fix(worker): ...`.
- **Description** follows the colon and a space, immediately — no line break.
- **Body** (optional) starts after a blank line and is free-form prose —
  motivation, contrast with previous behavior, whatever the reviewer needs.
- **Breaking changes** are marked with `!` right before the colon
  (`refactor(api)!: ...`), a footer `BREAKING CHANGE: <description>`, or both.
  Footer tokens use `-` instead of spaces (`Refs:`, `Reviewed-by:`) except
  `BREAKING CHANGE`, which is always that exact, all-caps two-word token
  (`BREAKING-CHANGE` is an accepted synonym).

**Examples:**

```
feat(auth): add passwordless login via magic link

fix(worker): retry transient Redis timeouts instead of failing the job

refactor(api)!: rename `user_id` to `userId` in response payloads

BREAKING CHANGE: clients must switch to the camelCase field name.
```

**"Wherever possible" is deliberate** — don't force a bad fit. An automated
dependency bump or a revert rarely maps cleanly to `feat`/`fix`; reach for
`chore`/`build`, or fall back to a short, clear, non-conventional message
rather than mislabeling it. The goal is a readable, semantically useful trunk
history, not 100% format compliance.

---

## Quick reference

| Situation | Do | Don't |
| --- | --- | --- |
| Your own branch, not yet shared | rebase onto trunk to stay current; tidy with interactive rebase before opening the PR | — |
| Branch others have pulled, or an open PR under review | keep pushing commits; let the squash-merge clean it up | rebase or force-push it |
| Merging a PR into `main` | squash-and-merge | rebase-and-merge; a merge commit that keeps every WIP commit |
| PR title | a Conventional Commit line — it becomes the squash commit subject | "fixes stuff", "wip", "final final v2" |
| Breaking API/behavior change | `type(scope)!: ...` and/or a `BREAKING CHANGE:` footer | breaking something without flagging it in the message |
| A change that doesn't fit `feat`/`fix` | `chore` / `build` / `refactor` / etc., or a plain clear message | force an inaccurate type onto it |
