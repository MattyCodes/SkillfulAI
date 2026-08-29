---
name: ruby-style
description: >-
  House Ruby conventions, based on Shopify's Ruby Style Guide: layout and
  indentation, syntax idioms, naming, class/module structure, exception
  handling, collections, strings, regular expressions, and Minitest testing —
  plus a base "autocorrect-only" RuboCop config to drop into new projects. Use
  this whenever writing, refactoring, or reviewing Ruby code (`.rb` files),
  choosing between Ruby idioms, structuring a class or module, handling
  exceptions, or setting up / tuning `.rubocop.yml`. NOT for Rails-specific
  conventions (model/controller/migration patterns — use a Rails style guide),
  RSpec (this guide standardizes on Minitest), or gem dependency and Bundler
  management. Apply it even if the user doesn't say "style guide" or "RuboCop"
  out loud.
---

# Ruby style

> **Scope — Ruby language style and idioms.** This skill governs how Ruby source
> is laid out, named, and structured, and how a project's RuboCop config is
> seeded. It is plain-Ruby: Rails and RSpec conventions live in their own
> guides. If the task falls outside "Use this skill for" below, this skill
> doesn't apply — say so rather than stretching it.

**Use this skill for**

- Writing or refactoring Ruby classes, modules, methods, scripts.
- Choosing between Ruby idioms (guard clause vs. nesting, `map` vs. `collect`,
  `fetch` vs. `[]`, ternary vs. `if`).
- Structuring exception handling.
- Naming methods, variables, constants, files.
- Creating or adjusting `.rubocop.yml`.
- Reviewing any of the above in a pull request.

**Do not use this skill for**

- Rails-specific patterns — models, controllers, migrations, callbacks,
  ActiveRecord query style. Those belong to a Rails style guide.
- RSpec test suites. This guide uses **Minitest** (see [Testing](#testing)).
- Gem selection, version constraints, Bundler, or `gemspec` authoring.
- Non-style architectural decisions (service boundaries, data modelling).

The guidance is: a **[RuboCop base config](#rubocop-setup)** to enforce the
mechanical parts, then a **[condensed conventions catalogue](#conventions)**
adapted from Shopify's Ruby Style Guide for the parts that need judgment.

---

## RuboCop setup

**[references/rubocop.yml](references/rubocop.yml)** is a starter config for new
projects. Copy it to the project root as `.rubocop.yml` and set
`AllCops.TargetRubyVersion`.

Its design: **only enable cops RuboCop can fix itself.** Anything that needs a
human to refactor or rename — `Metrics` (complexity, method/class/block length,
parameter lists), `Naming`, and the `Style` cops that require a real rewrite —
is turned off, so save-on-autocorrect never nags about something you must stop
and think about. The `Lint` department stays **on**: those flag latent bugs, and
you want to see them even though most don't autocorrect.

**Relationship to `rubocop-shopify`.** Shopify publishes the
[`rubocop-shopify`](https://github.com/Shopify/rubocop-shopify) gem, which
enforces their *entire* guide:

```ruby
# Gemfile
gem "rubocop-shopify", require: false
```
```yaml
# .rubocop.yml
inherit_gem:
  rubocop-shopify: rubocop.yml
```

Use `rubocop-shopify` when you want the whole guide enforced (CI, established
teams). Use `references/rubocop.yml` when you want a low-friction default that
only touches auto-fixable formatting. They agree on the load-bearing defaults:
2-space indent, 120-column lines, trailing commas in multiline literals, and —

**Strings use double quotes.** `"text"`, always, even with no interpolation.
This is the one string choice worth calling out explicitly: it overrides
RuboCop's built-in default of `single_quotes`, so `references/rubocop.yml`
pins `Style/StringLiterals` and `Style/StringLiteralsInInterpolation` to
`double_quotes` to keep linter and guide in agreement. It matches Shopify's
guide.

---

## Conventions

Condensed from Shopify's Ruby Style Guide. For anything not covered here,
consult **[references/community-ruby-style-guide.adoc](references/community-ruby-style-guide.adoc)**
— the upstream community guide (rubystyle.guide) that Shopify's is built on.

### General

- Keep every line of a method at the **same level of abstraction**.
- Prefer a **functional style** — avoid mutation and side effects where you can;
  don't mutate arguments.
- **Avoid defensive programming** against errors that can't actually occur.
- Avoid monkeypatching, needless metaprogramming, long methods, long parameter
  lists, and more than three levels of block nesting.
- Prefer `public_send` over `send` so visibility isn't silently bypassed.
- Write `ruby -w` clean code.

### Layout

- UTF-8 source, 2-space indent (no tabs), Unix line endings, newline at EOF.
- One expression per line — no `;`.
- Spaces around operators, after commas/colons/semicolons, inside `{ }`. No
  space after `(` `[`, before `]` `)`, after `!`, inside range literals
  (`1..5`), around `.`, inside `->()` lambda params.
- `case`/`when`: indent `when` to the same column as `case`.
- Align a conditional assignment's branches with the **variable**, not the
  keyword:

  ```ruby
  result = if some_cond
    calc_something
  else
    calc_something_else
  end
  ```

- Same for an assigned `begin` block — `rescue`/`ensure`/`end` align to the
  start of the line.
- Blank lines between method defs and to separate logical paragraphs inside a
  method. A blank line above and below a visibility modifier and around
  `attr_*`. No blank lines padding the start/end of a class/module/method/block
  body.
- Indent multi-line method-call arguments **one level** from the call, and put
  the closing `)` on its own line after the last argument (which keeps a
  trailing comma):

  ```ruby
  Mailer.deliver(
    to: "bob@example.com",
    from: "us@example.com",
    body: source.text,
  )
  ```

- When wrapping a method chain, put the receiver on its own line and indent each
  call one level:

  ```ruby
  User
    .pluck(:name)
    .sort(&:casecmp)
    .chunk { |n| n[0] }
  ```

- One element/argument per line when a call, array, or hash wraps. Closing
  `]`/`}` goes on the line after the last element.
- 120-column lines. No trailing whitespace. No block comments (`=begin`/`=end`).
- Separate a magic comment (`# frozen_string_literal: true`) from code/docs with
  a blank line.

### Syntax

- `::` only for constants and constructors (`Array()`, `Nokogiri::HTML()`) —
  not for regular method calls, and not for defining classes/modules (breaks
  constant lookup).
- `def` with parens when there are params; omit them when there are none.
- Avoid `for`, `then`, `and`/`or` (use `&&`/`||`), `not` (use `!`).
- Prefer the **ternary** over one-line `if/then/else`; one expression per
  branch; never nest ternaries; never make a ternary multiline — use `if`.
- `unless` over `if` for a negative condition; never `unless … else`.
- Parens around method-call arguments, **except** for `require`, `raise`,
  `puts`, `yield`, class-macro calls with an implicit receiver (`has_many
  :posts`), and operator-sugar calls. Omit outer braces on an implicit options
  hash.
- Block-arg shorthand when the block is a single method call:
  `names.map(&:upcase)`.
- `{ … }` for single-line blocks, `do … end` for multi-line.
- Omit `return`, `self`, and `()` on no-arg calls where possible (`self` is
  still needed to call an attribute writer).
- Wrap an assignment used as a condition in parens: `if (v = /re/.match(s))`.
- `||=` to memoize, but **not** for booleans — use `@x = true if @x.nil?`.
- `->(a, b) { … }` lambda literal over `lambda`; `proc` over `Proc.new`.
- Prefix unused block params with `_`.
- **Guard clauses** — bail early on invalid data instead of wrapping the body in
  a conditional:

  ```ruby
  def compute_thing(thing)
    return unless thing[:foo]
    return re_compute(thing) unless thing[:foo][:bar]
    partial_compute(thing)
  end
  ```

- Keyword arguments over an options hash.
- `map`/`find`/`select`/`size` over `collect`/`detect`/`find_all`/`length`.
- `Time` over `DateTime`; `Time.iso8601` over `Time.parse` for ISO-8601 input.
- Don't `return` from inside a `begin` used in an assignment context (silently
  skips the assignment → memoization bugs).

### Naming

- `snake_case` for methods, variables, symbols, files, directories.
- `CamelCase` for classes/modules; keep acronyms uppercase (`HTTPClient`,
  `XMLParser`).
- `SCREAMING_SNAKE_CASE` for other constants.
- One class/module per file, filename = `snake_case` of the name.
- Predicate methods end in `?` and return a boolean; don't end a non-boolean
  method with `?`; don't prefix with `is_` or `get_`.
- Only use a `!` suffix when a non-bang counterpart exists (bang = the more
  dangerous variant).
- Name a binary-operator parameter `other` (except `<<` and `[]`).
- No magic numbers — name a constant.
- Avoid nomenclature with discriminatory origins.

### Comments

- Add the context a reader might lack; keep comments in sync with code; proper
  capitalization and punctuation.
- Explain **why**, not how. Cut superfluous comments.

### Classes and modules

- Prefer a **module** to a class that only has class methods. Classes are for
  things you instantiate.
- `extend self` over `module_function`.
- Group class methods in a single `class << self` block; put `private` inside it
  for private class methods (`def self.foo` after a `private` is a silent bug).
- `attr_reader`/`attr_writer`/`attr_accessor` for trivial accessors; prefer them
  over `attr`.
- No class variables (`@@x`) — use `class << self` + `attr_accessor`, or a
  constant.
- Indent `public`/`protected`/`private` to the method-def level, one blank line
  above and below.
- `alias_method` over `alias`.
- Respect the Liskov Substitution Principle in hierarchies.

### Exceptions

- Signal with `raise`. Omit `RuntimeError` in the two-arg form — `raise
  "message"` already means `RuntimeError`.
- Pass **class + message as two arguments**, not an instance:
  `raise SomeError, "message"` (consistent with the three-arg backtrace form).
- Use an **implicit begin** — `def foo … rescue … end`, no explicit `begin`.
- Never `return` from an `ensure` block (swallows the exception).
- No empty `rescue`; no `rescue` modifier form (`do_something rescue nil`); no
  `rescue Exception` — a bare `rescue` catches `StandardError`, which is what you
  want.
- Prefer stdlib exception classes over inventing new ones.
- Name the rescued variable meaningfully: `rescue => error`, not `=> e`.

### Collections

- Literal `[]` / `{}` over `Array.new` / `Hash.new` (unless passing constructor
  args, e.g. a `Hash.new` default block).
- Literal array of strings/symbols over `%w`/`%i`.
- Trailing comma in every multiline literal.
- `first`/`last` over `[0]`/`[-1]`.
- No mutable objects as hash keys.
- Shorthand `{ a: 1, b: 2 }` when all keys are symbols; hash rockets when keys
  are mixed (`{ :a => 1, "b" => 2 }`).
- `Hash#key?` / `Hash#value?` over `has_key?` / `has_value?`.
- `Hash#fetch` for keys that must be present; `fetch(key, default)` for defaults
  (works correctly with `false`/`nil` values, unlike `||`).

### Strings

- **Double-quoted** always (see [RuboCop setup](#rubocop-setup)).
- String interpolation or `format` over concatenation. No padded spaces inside
  `#{ }`. `{}` around interpolated `@ivar` / `$global`. Don't call `.to_s` on an
  interpolated object — it's implicit.
- Avoid `?x` character literals.
- Prefer a specialized method over `String#gsub`: `sub`, `tr`, `delete`.
- Multi-line strings: **squiggly heredoc** `<<~END`; indent contents and the
  closing token to the opening's level.

### Regular expressions

- Plain-text search (`string["text"]`) over a regexp when you can.
- Non-capturing groups `(?:…)` when you don't use the capture.
- `Regexp#match` / `match?` over Perl `$1`, `$~`; named groups over numbered.
- `\A` and `\z` (not `^` / `$`) to anchor to string start/end.

### Percent literals

- `%()` only for a single-line string needing **both** interpolation and
  embedded `"`. Multi-line → heredoc.
- Avoid `%q` unless the string has both `'` and `"`. Avoid `%s` — use
  `:"with spaces"`.
- `%w`/`%i` are discouraged (see Collections); `%r` only for patterns
  containing a literal `/`.
- Delimit `%` literals with `()`, falling back to `{}`, `[]`, `<>` only when the
  literal itself contains the delimiter.

### Testing

- **Minitest** is the framework. Treat test code as real code.
- `test "description do … end"` style over `def test_foo`.
- One aspect per test case; split complex cases into isolated ones.
- Separate setup / action / assertion with blank lines:

  ```ruby
  test "sending a password reset email clears the hash and sets a token" do
    user = User.create!(email: "bob@example.com")
    user.mark_as_verified

    user.send_password_reset_email

    assert_nil user.password_hash
    refute_nil user.reset_token
  end
  ```

- Use the most descriptive assertion: `assert_equal "tobi", user.name` and
  `assert_predicate user, :valid?` over `assert user.name == "tobi"`.
- No `assert_nothing_raised` — make a positive assertion instead.
- Prefer assertions over mocking expectations (expectations are brittle,
  especially with singletons).

---

## Quick reference

| Situation | Do | Don't |
| --- | --- | --- |
| String literal | `"text"` | `'text'` |
| Multi-line string | `<<~HEREDOC` | `"a\n" + "b\n"`, `<<-HEREDOC` |
| Early exit on bad input | guard clause: `return unless valid` | wrap the whole body in `if valid` |
| Raise | `raise ArgumentError, "bad"` | `raise ArgumentError.new("bad")`, `raise RuntimeError, "x"` |
| Rescue | `def f … rescue => error … end` | `begin … rescue => e … end`, `rescue Exception`, `x rescue nil` |
| Missing-key access | `h.fetch(:k)` / `h.fetch(:k, default)` | `h[:k]`, `h[:k] || default` |
| Class-methods-only type | `module M; extend self` | `class M` with only `def self.` |
| Class method group | one `class << self` block | scattered `def self.x` |
| Conditional assignment | align branches to the variable | align to `if` / double-indent |
| Lambda | `->(a, b) { a + b }` | `lambda { |a, b| … }` |
| Iterate + filter | derive list, then iterate | `select`-in-loop, nested conditionals |
| Boolean memoization | `@x = true if @x.nil?` | `@x ||= true` |
| Multiline literal | trailing comma, `]`/`}` on next line | no trailing comma |
| Predicate name | `empty?` | `is_empty?`, `check_empty` |

## Reference files

- **[references/rubocop.yml](references/rubocop.yml)** — the autocorrect-only
  base config. Copy to a project as `.rubocop.yml`; set `TargetRubyVersion`.
- **[references/community-ruby-style-guide.adoc](references/community-ruby-style-guide.adoc)**
  — the full upstream community guide (rubystyle.guide) Shopify's is derived
  from. Consult it for rules and rationale not condensed above.
