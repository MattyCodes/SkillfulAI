---
name: frontend-practices
description: >-
  Best practices for JavaScript/TypeScript front-ends — Vue, React, and React
  Native alike: component and prop naming/style conventions, component
  architecture (primitive components built up into domain components), where
  state should live (component-local by default; Pinia/Zustand/Redux stores only
  for truly app-wide state), and consuming a GraphQL API with Apollo (codegen'd
  hooks/composables, the Apollo cache as the single source of truth for server
  data). Use this when building, structuring, styling, naming, or reviewing
  front-end components, deciding between local state and a global store,
  designing a component hierarchy, judging whether two similar UI pieces should
  share an abstraction, or wiring Apollo queries and mutations into components.
  NOT for GraphQL schema/SDL/resolver design (see `graphql-api-design`), backend
  or database code, or build-tool / bundler / CI configuration. Apply it even if
  the user doesn't say "best practices" out loud.
---

# Front-end practices (Vue / React / React Native)

> **Scope — front-end component code.** This skill governs how UI components are
> named, structured, styled, given state, and connected to a GraphQL API via
> Apollo. It is framework-agnostic across Vue, React, and React Native; where the
> frameworks genuinely differ, the difference is called out. If the task falls
> outside "Use this skill for" below, this skill doesn't apply — say so rather
> than stretching it.

**Use this skill for**

- Creating or refactoring Vue / React / React Native components.
- Naming components, props, component files, and style classes.
- Deciding component-local state vs. lifted state vs. a global store.
- Structuring a component hierarchy — primitives, presentational, domain.
- Consuming a GraphQL API from the front end with Apollo: queries, mutations,
  cache updates and invalidation.
- Judging whether two similar-looking UI pieces should share an abstraction.
- Reviewing any of the above in a pull request.

**Do not use this skill for**

- GraphQL schema / SDL / resolver design — that's `graphql-api-design`.
- Backend, API, or database implementation.
- Build tooling, bundler, linter, or CI/deploy config (Vite, webpack, Metro,
  ESLint, GitHub Actions).
- Data-fetching stacks other than Apollo (TanStack Query, SWR, Relay). The
  state principles in §2 still rhyme, but the specifics assume Apollo + GraphQL.

The guidance has two halves:

1. **[Style and naming conventions](#1-style-and-naming-conventions)** —
   adapted from the Vue Style Guide, generalized to React and React Native.
2. **[Design practices](#2-design-practices)** — Apollo/data-fetching, state
   location, component architecture, and how far to take DRY.

---

## 1. Style and naming conventions

These come from the Vue Style Guide (Priority A "Essential" and Priority B
"Strongly Recommended"). The *principles* apply equally to React and React
Native — a `.tsx` component and a `.vue` SFC have the same readability needs.

The full catalogue, with per-framework notes, is in
**[references/style-rules.md](references/style-rules.md)**. The high-value rules:

### Casing (house preference)

- **Components: `PascalCase`** — in the filename (`UserProfileCard.vue` /
  `UserProfileCard.tsx`), in JS/JSX/imports, and when referenced in templates
  (`<UserProfileCard />`, never `<user-profile-card />` in an SFC). JSX enforces
  this already; Vue templates should follow the same rule for consistency across
  the codebase.
- **Props: `camelCase`, not `kebab-case`** — declare `greetingText`, and write
  `<WelcomeMessage greetingText="hi" />` in templates and JSX. Vue permits
  kebab-case in SFC templates; **prefer `camelCase` everywhere** so props read
  identically in the template, the `defineProps` call, and the component body.
  Pick one casing and never mix.

### Component naming

- **One component per file**, filename identical to the component name.
- **Multi-word component names** (`TodoItem`, not `Todo`; `AppHeader`, not
  `Header`) — single-word names collide with current and future HTML elements.
  The app root (`App`) is the only exception.
- **Primitive/base components share a prefix** — `Base…`, `App…`, or `V…`
  (`BaseButton`, `AppButton`, `VButton`). See §2.3.
- **Tightly-coupled children carry the parent's name as a prefix** —
  `TodoList` → `TodoListItem` → `TodoListItemButton`. Files sort together and
  the coupling is visible.
- **Word order: general → specific.** `SearchButtonClear`, not
  `ClearSearchButton`; `SettingsCheckboxTerms`, not `TermsCheckbox`.
- **Full words over abbreviations** — `UserProfileOptions`, not `UProfOpts`.

### Templates / JSX

- **Detailed prop definitions.** Props are a contract: always give a type
  (TypeScript interface, or `defineProps` with `type` + `required` +
  `validator`). Bare `defineProps(['status'])` / untyped props are prototype-only.
- **Every list item gets a stable `key`** — a domain id, never the array index.
  In Vue, `key` is required on `v-for`; in React it's required on `.map`.
- **Don't filter or branch inside the loop.** Derive the list first — a
  `computed` / `useMemo` returning `activeUsers` — then iterate it. (Vue: never
  `v-if` + `v-for` on one element. React: `.filter().map()`, not a `null`
  return inside `.map`.)
- **Keep expressions in markup simple.** Anything past a property access or a
  short ternary moves into a named `computed` / `useMemo` / helper. Markup
  should say *what* renders, not *how* it's computed.
- **Split complex derived values** into several small named ones rather than one
  dense `computed`/`useMemo`.
- **One attribute per line** once an element has several.
- **Self-close empty components** (`<Spinner />`).

### Styling

- **Component-scoped by default** — `<style scoped>`, CSS Modules, BEM,
  Tailwind utility classes, or `StyleSheet.create` in React Native. Global CSS
  belongs only in the app root and layout components. Scope keeps class names
  short and low-specificity without collision risk.

---

## 2. Design practices

### 2.1 Apollo / data-fetching: the cache is the source of truth

For any front end that consumes a GraphQL API, **the Apollo normalized cache is
your server-state store.** Don't rebuild it in Pinia/Redux/Zustand, and don't
snapshot it into local refs.

**Use codegen'd hooks and composables, not the raw client.**

Generate operation-specific hooks (`useGetUserQuery`, `useUpdateUserMutation`,
`useUserListQuery`) with GraphQL Code Generator and call those from components.
Reaching for `apolloClient.query(...)` / `apolloClient.mutate(...)` directly in
a component is a smell — you lose the generated types, the reactive result
bindings, and the standard loading/error surface.

```ts
// Vue
const { result, loading, error } = useGetUserQuery(() => ({ id: props.userId }))
// React
const { data, loading, error } = useGetUserQuery({ variables: { id: userId } })
```

**Bind to the hook's result directly. Never copy it into local state.**

The reason is reactivity: the hook's `result` / `data` is a live view of the
cache. If another component's mutation or query updates that `User` in the
cache, everything bound to the live result re-renders. The moment you do
`const user = ref(result.value?.user)` or
`const [user, setUser] = useState(data?.user)`, you've forked a copy that no
longer tracks the cache and will silently go stale.

```ts
// Bad — forked copy, goes stale when the cache changes elsewhere
const user = ref(result.value?.user)

// Good — render straight from the live result
// template:  {{ result?.user.name }}
//   or a read-only computed:  const user = computed(() => result.value?.user)
```

If you need a *local editable draft* (a form the user is typing into), that's
legitimately local state — copy the fields you're editing into a form model,
keep rendering the canonical value from the cache elsewhere, and write back via
a mutation.

**After a mutation, invalidate — don't hand-patch.**

When a mutation changes server data, don't manually splice the new value into
component state. Let Apollo reconcile:

- If the mutation returns the modified entity with its `id` and the changed
  fields, the normalized cache updates itself — nothing else to do.
- If a mutation creates/deletes entities or changes a list's membership,
  invalidate the affected queries/entities — `refetchQueries`, `cache.evict` +
  `cache.gc`, or an `update` function that writes the canonical result — and
  let every bound component re-render from the refreshed cache.

The mental model: **components declare what they need; mutations tell the cache
what changed; Apollo pushes updates everywhere.** Components never move data
around by hand.

### 2.2 State location: local first, stores rarely

**Default to component-local state.** Whether a drawer is open, a panel
expanded, a row hovered, which wizard step is active, the current value of an
uncommitted input — all local. Communicate outward with events / callback props,
and lift state only to the nearest common ancestor that genuinely needs it.

**Reach for a global store (Pinia, Zustand, Redux, a Context provider) only for
state that is genuinely application-wide** — many unrelated components across the
tree both read and write it, and there's no sensible common ancestor to hang it
on. Typical legitimate cases:

- A toast / notification queue that any component can push to.
- The authenticated user / session.
- App-wide theme or locale.
- Feature flags.

Litmus test: if you can't name several unrelated components that would each both
read *and* write this state, it isn't global — keep it local and pass it.

**Server data is not "global state."** It feels global because many components
want it, but that's Apollo's job (§2.1), not a store's.

### 2.3 Component architecture: primitives, then build up

**Maintain a layer of primitive components** — `Card`, `Button`, `Input`,
`Dialog`, `Badge`, `Spinner` — that own only presentation, accessibility, and
basic interaction. They carry no domain knowledge and no feature-specific
styling.

- **Adopt a primitive UI library where you can** — shadcn/ui, Radix, Headless
  UI, Ark, React Native Paper. It's a well-tested starting layer.
- **Even without one, build your own primitives.** Don't scatter raw
  `<div class="rounded border shadow p-4">` across features — wrap it in a
  `Card` once.

**Compose domain components on top of primitives.** A
`DashboardPromotionalCard` *renders* a `Card` with the specific classes and
content that context needs; the domain knowledge lives in
`DashboardPromotionalCard`, while `Card` stays generic and reusable elsewhere.

**Extend primitives through their API — props, slots/`children`,
`class`/`className` — never by forking them** to add a one-off variant.

### 2.4 DRY is good, but it is not the whole of the law

Componentize the **small building blocks** that genuinely repeat — form inputs,
field labels, section headers, empty states, list rows. High reuse, small stable
interface, real savings.

Be much more skeptical about **abstracting whole pages, forms, or flows** just
because they look alike. Example: an email-verification form and a
phone-verification form share a layout, but their validation rules, the
mutations they call, and their error handling all differ. Componentize the
shared inputs and headers — but folding both into one
`ContactInformationVerificationForm` driven by a pile of props and conditionals
is almost always messier, harder to read, and barely shorter than two direct
components.

Heuristics:

- **Abstract when** the shared thing has one clear responsibility and a small
  interface. **Duplicate when** unifying would require flags/branches to paper
  over real differences.
- **Rule of three** — wait for the third occurrence before extracting a
  non-obvious abstraction. Two similar things might still diverge.
- "Prefer a little duplication over the wrong abstraction" — a bad abstraction
  is far more expensive to unwind than duplicated markup is to edit.

---

## Quick reference

| Situation | Do | Don't |
| --- | --- | --- |
| Component name | `PascalCase`, multi-word: `UserProfileCard` | `userProfileCard`, `Card` (single word), `user-profile-card` in an SFC |
| Prop name | `camelCase`: `greetingText` | `kebab-case`: `greeting-text` |
| Coupled child component | `TodoListItem` (parent prefix) | `Item` next to `TodoList` |
| Component word order | `SearchButtonClear` (general → specific) | `ClearSearchButton` |
| List rendering | stable `key`/`:key` from a domain id | index as key; `v-if` on the `v-for` element |
| Reading server data | bind to the codegen'd hook's live `result`/`data` | `ref(result.value?.user)` / `useState(data?.user)` |
| Running an operation | `useUpdateUserMutation()` | `apolloClient.mutate(...)` in a component |
| After a mutation | invalidate/evict cache entities, let Apollo re-render | hand-patch local component state |
| Drawer open / panel expanded | component-local state + events | a Pinia/Redux store |
| Global store | toast queue, session, theme, flags | server data, one-off screen state |
| Reused card styling | `Card` primitive, composed into `DashboardPromotionalCard` | copy the same styled `<div>` per feature |
| Two similar forms | share the inputs/labels; keep two form components | one mega-form with `type` props and branches |

## Reference files

- **[references/style-rules.md](references/style-rules.md)** — the full
  generalized Vue Style Guide catalogue (Essential + Strongly Recommended), each
  rule with its rationale and Vue / React / React Native specifics.
