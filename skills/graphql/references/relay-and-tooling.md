# Relay connections, custom scalars, and schema tooling

Supporting detail for the [GraphQL skill](../SKILL.md). Load this when you're
writing pagination types, wiring schema linting, or need the custom-scalar
cheat sheet.

## Contents

- [Relay cursor connections](#relay-cursor-connections)
- [Common custom scalars](#common-custom-scalars)
- [Enforcing conventions with graphql-eslint](#enforcing-conventions-with-graphql-eslint)
- [Schema validation in CI](#schema-validation-in-ci)

---

## Relay cursor connections

Full shape for a paginated list. Use `{Type}Connection` / `{Type}Edge` naming.

```graphql
type Query {
  posts(first: Int, after: String, last: Int, before: String): PostConnection!
}

type PostConnection {
  edges: [PostEdge!]!
  pageInfo: PageInfo!
  "Total number of posts matching the query, ignoring pagination."
  totalCount: Int
}

type PostEdge {
  node: Post!
  "Opaque cursor for this edge; pass as `after`/`before`."
  cursor: String!
}

type PageInfo {
  hasNextPage: Boolean!
  hasPreviousPage: Boolean!
  startCursor: String
  endCursor: String
}
```

**When to use a connection vs. a plain list:**

| Connection | Plain `[Item!]!` |
| --- | --- |
| Dynamic, frequently changing data | Static, rarely changing data |
| Large datasets needing pagination | Small datasets (< ~100 items) |
| Items added/removed over time | Config or reference data |
| Feeds, activity streams, search | Enum-like lookup tables |

If a list could ever grow unbounded, start with a connection — retrofitting
pagination onto a shipped `[Item!]!` field is a breaking change.

---

## Common custom scalars

| Scalar | Replaces | Purpose |
| --- | --- | --- |
| `DateTime` | `String` | ISO 8601 timestamp with timezone |
| `Date` | `String` | Calendar date, no time |
| `Email` | `String` | Validated email address |
| `URL` | `String` | Valid URL |
| `UUID` | `ID` / `String` | Standardized unique identifier |
| `JSON` | `String` | Arbitrary JSON where schema flexibility is required |

Use them when the format has clear validation rules, multiple fields share the
constraint, and codegen benefits from the distinct type. Don't create scalars
for "string with a business rule" (`Username`, `ProductCode`) — enforce those in
resolvers. Libraries: `graphql-scalars` (JS), `scalars.graphql.org` for
community specs.

```graphql
# Without: no validation, unclear format
type User {
  id: ID!
  email: String!
  createdAt: String!
}

# With: self-documenting, validated before resolvers run
type User {
  id: ID!
  email: Email!
  createdAt: DateTime!
}
```

---

## Enforcing conventions with graphql-eslint

`@graphql-eslint/eslint-plugin` can mechanically enforce most of §1 and §3 of
the skill. Baseline config:

```js
// eslint.config.js
export default {
  overrides: [
    {
      files: ["**/*.graphql"],
      parser: "@graphql-eslint/eslint-plugin",
      plugins: ["@graphql-eslint"],
      rules: {
        "@graphql-eslint/naming-convention": [
          "error",
          {
            types: "PascalCase",
            FieldDefinition: "camelCase",
            InputValueDefinition: "camelCase",
            Argument: "camelCase",
            DirectiveDefinition: "camelCase",
            EnumValueDefinition: "UPPER_CASE",
            // §3.1 — no verb prefixes on query fields
            "FieldDefinition[parent.name.value=Query]": {
              forbiddenPrefixes: ["get", "fetch", "list", "load", "find"],
            },
            // §3.2 — mutations are verb-first CRUD; block the "Mutation" suffix
            "FieldDefinition[parent.name.value=Mutation]": {
              forbiddenSuffixes: ["Mutation"],
            },
            // §3.3 — input types end in "Input"
            "InputObjectTypeDefinition": {
              style: "PascalCase",
              requiredSuffixes: ["Input"],
            },
          },
        ],
        "@graphql-eslint/require-description": [
          "error",
          { types: true, FieldDefinition: true },
        ],
        "@graphql-eslint/require-deprecation-reason": "error",
      },
    },
  ],
};
```

For breaking-change detection, schema diffing, and coverage analysis beyond
linting, use `graphql-inspector`.

---

## Schema validation in CI

```yaml
name: Schema Validation
on:
  pull_request:
    paths:
      - "schema/**/*.graphql"

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: "20"

      - name: Install dependencies
        run: npm ci

      - name: Lint schema
        run: npx eslint "schema/**/*.graphql"

      - name: Check documentation coverage
        run: |
          npx graphql-schema-linter \
            --rules fields-have-descriptions \
            --rules types-have-descriptions \
            schema/**/*.graphql
```
