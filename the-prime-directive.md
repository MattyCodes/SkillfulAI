---
name: the-prime-directive
description: >-
  Core, product-/language-/framework-agnostic guidelines for coding agents and
  assistants, meant to be loaded into the context of every prompt. A project's
  agent-instructions file can simply point here; after reading this, the agent
  should search this repository's `skills/` directory for guidance specific to
  the task at hand.
---

# The Prime Directive

Core guidelines for any coding agent, in any project. They're deliberately
generic — anything specific to a language, framework, or product lives in a
skill.

## Find the specific guidance first

Before starting, search this repository's `skills/` directory for skills that
cover the task — by language (`skills/ruby`), by layer (`skills/frontend`), by
domain (`skills/graphql`), and so on — and read the ones that apply. A project's
agent-instructions file may do nothing more than point here; everything more
specific than the universals below lives in `skills/`.

## Ground rules

Not tradeoffs — hold to all of them:

- **Do what was asked, and no more.** Keep diffs minimal and reversible. Don't
  opportunistically refactor or reformat unrelated code. Confirm before any
  destructive or hard-to-reverse action.
- **Match the codebase you're in.** Read the surrounding code first and follow
  its conventions, structure, and naming. Consistency with the project beats
  personal preference or the "ideal" form.
- **Verify before calling it done.** Run the build, tests, and linter; add or
  update tests for any behavior change; never claim something works if you
  haven't checked. Say what you verified and what you assumed.
- **Flag, don't sprawl.** When you spot valuable work outside the current scope
  — a refactor, an optimization, a latent bug — note it in a `TODO` comment, a
  PR comment, or both, rather than widening the diff. Running unattended and
  can't ask? Never expand scope on your own.

## Design principles

When these pull in different directions, the earlier one wins.

### 1. Clarity

Readable, comprehensible code is the priority. Keep asking:

- Could a newcomer follow this? Are the names intuitive?
- Is it organized the way the rest of the project is?

Comment the *why*, not the *what* — context and intent that isn't obvious from
the code itself. Don't narrate what the code plainly does, keep it brief, and
never leave comments that only make sense in light of the current change.

```ruby
# good — explains a non-obvious external constraint
# Formats arguments for the CRM client so string values are encoded the way
# their API requires.
def format_arguments(*args)
  # ...
end

# bad — narrates the change instead of the code
# format args for crm - encoding issue is fixed now ✅
def format_arguments(*args)
  # ...
end
```

### 2. Security

Weigh the security implications of every change, and flag what you find even
when it's out of scope (see Ground rules):

- Does this widen data exposure or weaken an authorization check?
- Does it open an injection path (SQL, shell, prompt) or an XSS/CSRF hole?
- Are untrusted inputs validated? Are secrets kept out of logs, errors, and
  version control?
- Is there business-logic abuse to consider (rounding, rate-limit, retry gaps)?

### 3. Extensibility

Reuse before you rebuild: check whether the project already has a function or
service for the job. Avoid duplication — but not at the cost of clarity; a
little repetition beats a confusing abstraction.

Look for chances to make code reusable so there's less to review and maintain.
Refactors that reach beyond the task follow "flag, don't sprawl".

### 4. Performance

Meet normal standards without chasing every millisecond. What "normal" means is
context-specific; as illustrations: front-ends avoid needless layout shift,
frequently queried tables carry the right indexes, typical endpoints resolve
well under a second. If a real gain needs a larger change, or you can't judge it
without knowing production data volumes, flag it rather than guess.
