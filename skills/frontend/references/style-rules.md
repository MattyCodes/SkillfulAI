# Style rules catalogue (Vue Style Guide, generalized)

Supporting detail for the [front-end practices skill](../SKILL.md §1). Every rule
below is from the Vue Style Guide's **Priority A (Essential)** or **Priority B
(Strongly Recommended)** tiers, restated so it applies across Vue SFCs, React
`.tsx`, and React Native.

Rule of thumb: Essential rules prevent bugs — follow them everywhere. Strongly
Recommended rules improve readability — deviations should be rare and justified.

## Contents

- [Essential](#essential)
  - [Multi-word component names](#multi-word-component-names)
  - [Detailed prop definitions](#detailed-prop-definitions)
  - [Keyed list rendering](#keyed-list-rendering)
  - [Never filter/branch on the loop element](#never-filterbranch-on-the-loop-element)
  - [Component-scoped styling](#component-scoped-styling)
- [Strongly recommended](#strongly-recommended)
  - [One component per file](#one-component-per-file)
  - [Filename casing](#filename-casing)
  - [Base / primitive component names](#base--primitive-component-names)
  - [Tightly coupled component names](#tightly-coupled-component-names)
  - [Word order in component names](#word-order-in-component-names)
  - [Full words over abbreviations](#full-words-over-abbreviations)
  - [Self-closing empty components](#self-closing-empty-components)
  - [Component name casing in markup](#component-name-casing-in-markup)
  - [Component name casing in JS/JSX](#component-name-casing-in-jsjsx)
  - [Prop name casing](#prop-name-casing)
  - [Multi-attribute elements](#multi-attribute-elements)
  - [Simple expressions in markup](#simple-expressions-in-markup)
  - [Simple computed / derived values](#simple-computed--derived-values)
  - [Quoted attribute values](#quoted-attribute-values)
  - [Directive shorthands (Vue)](#directive-shorthands-vue)

---

## Essential

### Multi-word component names

User component names should always be multi-word, except a root `App`. Single
words collide with existing and future HTML elements (`<Item>`, `<Header>`,
`<Table>`).

- **Vue:** `TodoItem`, not `Todo`.
- **React / RN:** JSX already forces an uppercase identifier, but the naming
  concern stands — `Header` shadows intent and clashes conceptually with the DOM
  element; prefer `AppHeader`, `SiteHeader`, `CardHeader`.

### Detailed prop definitions

Props are the component's public contract. In committed code, specify at least a
type; ideally required-ness and validation too.

```ts
// Vue — minimum
const props = defineProps<{ status: 'syncing' | 'synced' | 'error' }>()

// Vue — runtime validation where values are constrained
const props = defineProps({
  status: {
    type: String,
    required: true,
    validator: (v) => ['syncing', 'synced', 'error'].includes(v),
  },
})
```

```tsx
// React / RN — the interface is the contract; make illegal states unrepresentable
type StatusIndicatorProps = {
  status: 'syncing' | 'synced' | 'error'
  onRetry?: () => void
}
```

Bare `defineProps(['status'])` or an untyped `props: any` is acceptable only
while prototyping.

### Keyed list rendering

Every rendered list item needs a stable `key` tied to the item's identity
(a domain id), never the array index — index keys corrupt component state and
animations when the list reorders or items are inserted/removed.

```vue
<li v-for="todo in todos" :key="todo.id">{{ todo.text }}</li>
```

```tsx
{todos.map((todo) => <TodoRow key={todo.id} todo={todo} />)}
```

- **Vue:** `key` is required on `v-for` over components; use it on elements too.
- **React:** `key` is required on any `.map`-rendered array.

### Never filter/branch on the loop element

Don't combine iteration with a per-item conditional on the same element. Derive
the list you actually want to render, then iterate it — the intent is clearer
and the derived list is reusable and memoisable.

```vue
<!-- Bad: v-if + v-for on one element -->
<li v-for="u in users" v-if="u.isActive" :key="u.id">{{ u.name }}</li>

<!-- Good: derive first -->
<li v-for="u in activeUsers" :key="u.id">{{ u.name }}</li>
```

```ts
const activeUsers = computed(() => users.value.filter((u) => u.isActive))
```

```tsx
// React: filter before map — don't return null from inside map
const activeUsers = useMemo(() => users.filter((u) => u.isActive), [users])
return activeUsers.map((u) => <UserRow key={u.id} user={u} />)
```

If the goal is to hide the *whole* list, put the condition on a container, not
on each row.

### Component-scoped styling

Only the app root and layout components may carry global styles. Everything else
is scoped, by whatever mechanism the stack uses:

- **Vue:** `<style scoped>`, `<style module>` (CSS Modules), or a class
  convention like BEM. Component libraries should prefer the class-based
  approach over `scoped` so consumers can override without a specificity war.
- **React:** CSS Modules, styled-components/Emotion, or Tailwind utilities.
- **React Native:** `StyleSheet.create` local to the component file.

Scoping keeps class names human-readable and low-specificity while making
collisions effectively impossible.

---

## Strongly recommended

### One component per file

Whenever a build system can concatenate files, each component lives in its own
file. Faster to locate, review, and diff.

### Filename casing

Pick one and hold to it repo-wide: **`PascalCase`** (`UserCard.vue` /
`UserCard.tsx` — best editor autocomplete, matches JS/JSX references) or
**`kebab-case`** (`user-card.vue` — safe on case-insensitive filesystems). Don't
mix. House preference: `PascalCase` (see SKILL.md §1).

### Base / primitive component names

Presentational components that only apply app styling and conventions share a
prefix — `Base`, `App`, or `V`:

```
BaseButton.vue   AppButton.vue   VButton.vue
BaseIcon.vue     AppIcon.vue     VIcon.vue
BaseTable.vue    AppTable.vue    VTable.vue
```

Pick one prefix per project. This groups the primitive layer alphabetically and
signals "no domain logic in here". Ties into SKILL.md §2.3.

### Tightly coupled component names

A child that only makes sense inside one parent takes the parent's name as a
prefix:

```
TodoList.vue
TodoListItem.vue
TodoListItemButton.vue

SearchSidebar.vue
SearchSidebarNavigation.vue
```

The coupling becomes visible and the files sort next to each other.

### Word order in component names

Start with the highest-level (most general) word, end with the modifier. This
makes a components directory self-grouping:

```
Bad                          Good
ClearSearchButton.vue        SearchButtonClear.vue
RunSearchButton.vue          SearchButtonRun.vue
SearchInput.vue              SearchInputQuery.vue
ExcludeFromSearchInput.vue   SearchInputExcludeGlob.vue
TermsCheckbox.vue            SettingsCheckboxTerms.vue
LaunchOnStartupCheckbox.vue  SettingsCheckboxLaunchOnStartup.vue
```

### Full words over abbreviations

Editor autocomplete makes long names cheap; unclear abbreviations are expensive.
`StudentDashboardSettings`, not `SdSettings`; `UserProfileOptions`, not
`UProfOpts`.

### Self-closing empty components

A component with no content should self-close in SFCs, string templates, and
JSX: `<MyComponent />`. It communicates "meant to have no content," and drops a
redundant closing tag. (In-DOM templates can't — HTML forbids self-closing
custom elements — so there use `<my-component></my-component>`.)

### Component name casing in markup

`PascalCase` in SFCs and string templates (`<MyComponent />`) — enables editor
autocomplete, is more visually distinct from single-word HTML elements, and
stays distinct from web components. `kebab-case` is only forced in in-DOM
templates. JSX is always `PascalCase`. Consistency repo-wide matters more than
the choice; house preference is `PascalCase` (SKILL.md §1).

### Component name casing in JS/JSX

Always `PascalCase` in imports, `name` fields, and references:
`import UserCard from './UserCard.vue'`. `kebab-case` strings are tolerable only
in simple apps using global `app.component('my-component', …)` registration.

### Prop name casing

Declare props in `camelCase` (`greetingText`). In-DOM templates force
`kebab-case` on use; SFC templates and JSX accept either — **house preference is
`camelCase` on use too**, so the name is identical in the template, in
`defineProps`, and in the component body. Never mix the two styles in one
codebase.

```vue
<!-- declaration -->
const props = defineProps({ greetingText: String })
<!-- use (house style) -->
<WelcomeMessage greetingText="hi" />
```

### Multi-attribute elements

Once an element has more than one or two attributes, put one per line — same
reasoning as splitting multi-property object literals.

```vue
<MyComponent
  foo="a"
  bar="b"
  baz="c"
/>
```

### Simple expressions in markup

Markup should hold only simple expressions — a property access, a short ternary.
Move anything heavier into a named `computed` / `useMemo` / method so the markup
stays declarative and the logic is reusable and testable.

```vue
<!-- Bad -->
{{ fullName.split(' ').map(w => w[0].toUpperCase() + w.slice(1)).join(' ') }}
<!-- Good -->
{{ normalizedFullName }}
```

### Simple computed / derived values

Split a dense computed into several small named ones. Each intermediate is
independently readable, reusable, and debuggable.

```ts
const basePrice   = computed(() => manufactureCost.value / (1 - profitMargin.value))
const discount    = computed(() => basePrice.value * (discountPercent.value || 0))
const finalPrice  = computed(() => basePrice.value - discount.value)
```

Same idea in React with chained `useMemo`s or plain derived consts.

### Quoted attribute values

Always quote non-empty attribute values, even when HTML would allow bare values.
Unquoted values pressure you into avoiding spaces and hurt readability.

### Directive shorthands (Vue)

Use the shorthands (`:` for `v-bind:`, `@` for `v-on:`, `#` for `v-slot:`)
*always* or *never* within a project — not a mix. Most Vue codebases choose
"always". (No equivalent in React/RN.)
