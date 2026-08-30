---
name: the-prime-directive
description: >-
  Best practices for designing code in general, regardless of product, language, or framework. These are the core instructions for coding agents
  and assistants which should be included in the context of every prompt received. After loading the context from this file, AI agents should search the rest of this repository for any relevant skills for the task at-hand.
---


# The Prime Directive
This document should serve as the core context and guidelines for all AI
agents and assistants; these instructions are intentionally generalized and
should not be specific to any product, language, or framework. Any specific,
more narrowly defined skills should be found in the `skills` directory of this
repository. The following principles, in order, are the most important to
keep in mind while designing code.

---

### 1. Clarity
Writing code that is readable and comprehensible is the most important
thing; always be questioning:
- Is this written in a way that humans/newcomers could understand?
- Are these naming conventions sensible and intuitive?
- Is this work well-organized? Is it organized in accordance with the rest of the project?

Additionally, comments in-code are a software engineer's best friend. Write comments
that explain the context and purpose of various functions and services, but be
concise - comments that are too wordy won't get read. Do not write
comments that contain context specific to the current prompt/conversation.

This is a good example:
```ruby
# Formats arguments for the CRM client - this has to be done to ensure
# that string arguments are properly encoded, per their API's requirements.
def format_arguments(*args)
    # ...
end
```

This is a bad example:
```ruby
# format arguments for crm - the issue of url arguments not getting
# encoded properly is fixed ✅
def format_arguments(*args)
    # ...
end
```

### 2. Security
Whenever making changes to the codebase, the security implications of that change
should be considered. Ask (and, if needed, address) the following questions:
- Does this change increase our (data) exposure in a way that is unsafe?
- Are there fraud vectors not being considered (salami attack, penny shaving, etc)?
- Are there other security vulnerabilities that are not being covered (SQL/prompt injection, XSS attacks, etc)?

Noticing vulnerabilities does not _always_ mean that they have to be addressed
immediately as part of the current work, but they should at least be flagged so the user is
aware and can decide when and how to address them.

### 3. Extensibility
Duplicating code should _almost_ always be avoided when possible - we don't want to DRY up and
consolidate code at the expense of readability/clarity, however.

Always check to see if the codebase has existing functions/services that you can make use of
before potentially rewriting functionality that already exists in the project.

Always consider how code can be abstracted or made to be reusable to reduce the overall volume
of code that has to be reviewed and maintained.

If there are large opportunities to refactor/abstract existing functionality so that it can be
used more broadly, but you aren't sure if it fits into the current scope of work, ask the
user for confirmation before creating too large of a diff - if you are operating as a
background agent and cannot ask the user for confirmation, do not proceed with the
refactor, but do make note of the recommended refactor either in a `TODO` comment in-code, a
comment on the pull request, or both.

### 4. Performance
Lastly, but still very importantly, is performance. Always consider if we're doing what we can
to optimize performance within reason - we don't necessarily need to save every millisecond we
possibly can, but certain normal standards should always be expected: minimize the potential
for layout-shift in front-end projects, ensure that new database tables have the appropriate
indexes for queries we expect to be making often, write API endpoints such that queries
never take more than a second to resolve (ideally less then 200ms for simpler endpoints).

If there are situations where performance could be improved further, but would require a
larger volume of work, or if you are unsure if performance is worth worrying about due to
not knowing what the volume of the production dataset looks like etc, you don't have to solve
those cases immediately. Do note them for the user so they can decide when and how to address
them - if you are operating as a background agent and cannot ask the user for confirmation,
do not proceed with the optimization, but do make note of the recommended steps either
in a `TODO` comment in-code, a comment on the pull request, or both.
