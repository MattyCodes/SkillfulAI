---
name: graphql-api-design
description: >-
  House conventions for DESIGNING and AUTHORING GraphQL schemas: naming queries,
  mutations, types, enums, arguments, and input/payload types; nullability and
  pagination choices; custom scalars; the top-level error model and its
  `ErrorCode` enum; and versionless schema evolution (when to add
  `updateUserV2`). Use this when writing or reviewing a `.graphql` schema / SDL,
  adding, renaming, or deprecating a query or mutation, shaping an `*Input` or
  `*Payload` type, defining the `ErrorCode` enum, or deciding how a resolver
  should classify and raise errors. NOT for writing tests against a GraphQL API,
  authoring client-side query/mutation documents, wiring a GraphQL client
  (Apollo/urql/Relay) or its `onError` handlers, code generation setup, or
  resolver business logic — those touch GraphQL but are separate concerns. Apply
  it even if the user doesn't say "best practices" or "conventions" out loud.
---

# GraphQL conventions and design standards

> **Scope — schema design and authorship only.** This skill governs how a
> GraphQL schema is *shaped and named*. It does not cover consuming a GraphQL
> API or testing one. If the task falls outside "Use this skill for" below,
> this skill doesn't apply — say so and stop rather than stretching it to fit.

**Use this skill for**

- Writing or editing `.graphql` / SDL schema files.
- Adding, renaming, or deprecating a query or mutation.
- Designing a type, interface, union, enum, or custom scalar.
- Shaping `*Input` and `*Payload` types and the fields inside them.
- Defining or extending the `ErrorCode` enum, and deciding which code a given
  resolver failure maps to.
- Choosing nullability, pagination style, or an evolution strategy (additive
  change vs. a `V2` mutation).
- Reviewing any of the above in a pull request.

**Do not use this skill for**

- Writing tests (unit, integration, contract, snapshot) against a GraphQL API.
- Authoring client-side operation documents — the `query { ... }` /
  `mutation { ... }` strings an app sends.
- Wiring or configuring a GraphQL client (Apollo, urql, Relay): links, cache
  policies, `onError` handlers. §2 shows a client snippet only to illustrate
  the error contract; implementing it is a separate task.
- GraphQL code generation setup (`graphql-codegen`, schema-to-types pipelines).
- Implementing resolver business logic, data loaders, batching, or persistence.
- Designing non-GraphQL APIs (REST, gRPC). The naming ideas may rhyme, but the
  specifics here assume GraphQL.

---

The GraphQL spec deliberately leaves naming and design open. Teams that skip
conventions end up with schemas where `getUser`, `user`, and `userById` all
coexist, code generators produce awkward types, and every new field is a
negotiation. This skill encodes a consistent set of choices so schemas stay
predictable as they grow.

It has three parts:

1. **[Naming and design baseline](#1-naming-and-design-baseline)** — the
   community conventions from graphql.org, restated for quick application.
2. **[Error handling](#2-error-handling)** — raise errors as top-level
   exceptions carrying a strongly typed `errorCode`.
3. **[House naming rules](#3-house-naming-rules)** — the specific query,
   mutation, input/output, and versioning patterns to follow here. Where the
   baseline offers a choice, these rules pick one.

When the baseline and the house rules disagree, **the house rules win** — their
job is to remove the "either/or".

---

## 1. Naming and design baseline

### Casing

| Element | Convention | Example |
| --- | --- | --- |
| Fields | `camelCase` | `firstName`, `createdAt` |
| Arguments | `camelCase` | `userId`, `includeArchived` |
| Types | `PascalCase` | `User`, `ProductConnection` |
| Enums | `PascalCase` | `OrderStatus`, `Currency` |
| Enum values | `SCREAMING_SNAKE_CASE` | `IN_PROGRESS`, `COMPLETED` |
| Interfaces | `PascalCase` | `Node`, `Timestamped` |
| Unions | `PascalCase` | `SearchResult`, `MediaItem` |
| Directives | `camelCase` | `@include`, `@semanticNonNull` |

### Field-level patterns

- **Boolean fields start with `is` or `has`** so the return type is obvious from
  the name: `isActive`, `hasSubscription`. (The spec itself does this with
  `isDeprecated`.)
- **List fields use plural nouns**: `posts: [Post!]!`, not `postList`. A single
  related object keeps the singular: `featuredPost: Post`.
- **Model relationships as graph edges, not foreign keys.** Expose
  `author: User!`, never `authorId: ID!`. Clients traverse the graph; they
  should not re-implement joins.
- **The schema reflects the business domain, not the database.** No
  `user_id`, no `created_timestamp`, no `UserTable`. Rename as you expose.

### Identifiers

Use `ID!` for identity fields and id arguments, not `Int!`/`String!`. `ID`
signals "opaque handle, don't do arithmetic on it" and serializes as a string.
Reach for `Int!` only when the value is genuinely a number the client computes
with.

### Nullability

- GraphQL fields are **nullable by default**; non-null is a promise you must
  always keep. When a non-null field resolves to `null`, GraphQL propagates the
  `null` up to the nearest nullable parent ("null bubbling"), so an overly
  strict inner field can blank out a whole object.
- **Lists: prefer `[Item!]!`** so clients get `[]` instead of `null` and never
  null-check the list itself or its elements.
- **Input types have a three-way distinction** — field omitted, field set to
  `null`, field set to a value. This matters for partial updates. Document the
  intended behavior in the field description, e.g.
  `"Pass a value to update, null to clear, or omit to leave unchanged."`
  If the ambiguity is dangerous, add an explicit `clearBio: Boolean` flag
  instead of overloading `null`.

### Custom scalars

Use custom scalars when a format has real validation rules (`DateTime`, `Date`,
`Email`, `URL`, `UUID`, `JSON`) and several fields share the constraint.
Validation then happens at the GraphQL layer, before resolvers run.

Do **not** invent scalars for "string with a business rule" types like
`Username` or `ProductCode` — enforce those in resolver logic. Custom scalars
cost implementation effort in every client and server and hurt portability.

### Pagination with connections

Use the Relay Cursor Connections shape for lists that are large, unbounded, or
frequently changing (feeds, activity streams, search results):

```graphql
type Query {
  posts(first: Int, after: String): PostConnection!
}
```

Name the types `{Type}Connection` / `{Type}Edge`, with a `PageInfo` and an
optional `totalCount`. Full boilerplate is in
[references/relay-and-tooling.md](references/relay-and-tooling.md).

Plain `[Item!]!` lists are fine for small (< ~100), static, reference-style data
(lookup tables, config, enum-like sets). If a list *might* grow unbounded, start
with a connection.

### Documentation

Write a description for **every type** and **every non-obvious field**.
Descriptions render in GraphiQL and generated docs — treat them as
user-facing. Use the triple-quote block form for multi-line notes, and call out
nullability conditions and units (`"Current price in USD cents. May be null
during off-season."`).

### Anti-patterns to flag in review

- Exposing `xxxId: ID!` where an object reference belongs.
- Type/field names that mirror table and column names.
- `get`/`fetch`/`list`/`find` prefixes on query fields (see §3).
- Mutation input/payload types whose contents don't track the resource — a
  payload that never exposes the affected object, or inputs named
  `UserAttributes` / `UserDraft` (see §3).
- Non-null inner fields that can realistically fail to resolve and take their
  parent down with them.

Tooling to enforce a lot of this automatically (`graphql-eslint`,
`graphql-inspector`, CI wiring) is in
[references/relay-and-tooling.md](references/relay-and-tooling.md).

---

## 2. Error handling

**Every error in a query or mutation — validation failure, missing record,
permission denial, upstream outage — is raised as a top-level GraphQL error
(the `errors` array in the response), not returned as a field in the data.**

Each raised error carries a machine-readable code in `extensions.code`, and that
code is a value of an `ErrorCode` enum **published in the schema**.

### Why this way

- **One handling path on the client.** Apollo Link's `onError`, `errorPolicy`,
  urql's `errorExchange`, and every similar hook see the `errors` array. Errors
  buried in `data` bypass all of it and every caller re-invents detection.
- **Exhaustive, typed branching.** Because `ErrorCode` is in the schema,
  codegen emits it as a TypeScript union. Client code can `switch` on it and the
  compiler flags unhandled cases when you add one.
- **Transport-agnostic classification.** `NOT_FOUND` means the same thing
  whether it came from a query or a mutation, so retry / redirect / toast logic
  lives in one place.

### The enum

Publish a small, stable set of classifications. Model it on HTTP status
semantics — that keeps it intuitive and maps cleanly onto gateways and logs:

```graphql
"""
Machine-readable classification for a top-level GraphQL error. Always present in
`extensions.code` on errors this API raises. Clients should branch on this
rather than parsing `message`.
"""
enum ErrorCode {
  "Malformed input, bad arguments, failed input validation."
  BAD_REQUEST
  "No credentials, or credentials are invalid/expired."
  UNAUTHENTICATED
  "Authenticated, but not allowed to perform this operation."
  FORBIDDEN
  "The requested resource does not exist."
  NOT_FOUND
  "The request conflicts with current state (e.g. duplicate, version mismatch)."
  CONFLICT
  "Well-formed but semantically invalid (business-rule violation)."
  UNPROCESSABLE_ENTITY
  "Caller has sent too many requests."
  RATE_LIMITED
  "Unexpected server-side failure. The client cannot do anything but retry/report."
  INTERNAL_SERVER_ERROR
}
```

Keep it coarse. Codes are a contract — adding one is cheap, changing or removing
one is a breaking change. Put row-level detail (which field, which id) in
`extensions`, not in new enum values.

### Server: raise, don't catch-and-return

The mechanism is the same across implementations — throw/raise a real error
whose `extensions` contains `code`:

```ts
// graphql-js / Apollo Server / Yoga
import { GraphQLError } from "graphql";

if (!user) {
  throw new GraphQLError("User not found", {
    extensions: { code: "NOT_FOUND", entity: "User", id },
  });
}
```

```python
# graphql-core / Strawberry / Ariadne
from graphql import GraphQLError

if user is None:
    raise GraphQLError(
        "User not found",
        extensions={"code": "NOT_FOUND", "entity": "User", "id": id},
    )
```

Guidelines:

- **Do not wrap resolver bodies in try/except that returns an error object.**
  Let it propagate.
- **Map known failure classes to codes centrally** — one error-formatting hook
  (`formatError` / a middleware) that reads a `code` off your domain exceptions,
  fills in `INTERNAL_SERVER_ERROR` for anything unrecognized, and strips stack
  traces / internal messages in production.
- **Never send a raw internal message or stack trace to the client.** The
  `message` is for humans debugging; the `code` is the contract.

### Client: branch on the code

```ts
import { onError } from "@apollo/client/link/error";
import type { ErrorCode } from "./generated/graphql"; // codegen'd from the enum

const errorLink = onError(({ graphQLErrors }) => {
  for (const err of graphQLErrors ?? []) {
    switch (err.extensions?.code as ErrorCode) {
      case "UNAUTHENTICATED":
        redirectToLogin();
        break;
      case "FORBIDDEN":
        toast("You don't have access to that.");
        break;
      case "RATE_LIMITED":
        scheduleRetryWithBackoff();
        break;
      // codegen makes the compiler complain when a new ErrorCode is unhandled
    }
  }
});
```

### The one deliberate exception

A mutation whose **expected** outcome includes per-field validation errors the
UI must render inline (a signup form showing "email already taken" next to the
field) *may* additionally model those as data — a `userErrors: [UserError!]!`
list on the payload. This is an opt-in per mutation, decided deliberately and
documented. It never replaces the top-level `errorCode` for anything
exceptional, and everything in §2 above is still the default.

---

## 3. House naming rules

### 3.1 Queries name the thing they return

A query field is a noun. The operation type already says it's a read; `get` /
`fetch` / `list` / `load` add nothing and break symmetry with nested fields
(you'd never write `user.getPosts`).

```graphql
type Query {
  user(id: ID!): User
  users(first: Int, after: String): UserConnection!
  currentUser: User
  searchArticles(query: String!): [Article!]!
}
```

```graphql
# Avoid
type Query {
  getUser(id: ID!): User
  fetchUsers: [User!]!
  loadCurrentUser: User
}
```

Singular field + `id` argument for one; plural field for a collection. A query
that is genuinely an action phrased as a question (`searchArticles`,
`validateCoupon`) can keep its verb — but a plain lookup never does.

### 3.2 Mutations use CRUD names

Unless there is a real business/domain reason to do otherwise, mutation names
are exactly `create<Resource>`, `update<Resource>`, `delete<Resource>`:

```graphql
type Mutation {
  createUser(input: CreateUserInput!): CreateUserPayload!
  updateUser(input: UpdateUserInput!): UpdateUserPayload!
  deleteUser(input: DeleteUserInput!): DeleteUserPayload!
}
```

- **Verb-first, `camelCase`, singular resource.**
- Do **not** use synonyms — no `newUser`, `modifyUser`, `editUser`,
  `removeUser`, `destroyUser`. One verb per operation, everywhere.
- **Genuine domain operations that aren't CRUD get their own verb-first
  action name**: `publishArticle`, `archiveProject`, `sendPasswordResetEmail`,
  `transferOwnership`. Don't contort these into `updateArticle` with a magic
  flag, and don't invent a fake noun to make them look like CRUD.

### 3.3 Input and output types mirror the resource

Mutations take a `<Verb><Resource>Input` and return a `<Verb><Resource>Payload`.
Most server frameworks — the Ruby `graphql` gem's generators, Relay-style
tooling, etc. — produce these wrapper types for you, and that's expected. What
matters is that their **contents** track the resource:

```graphql
type Mutation {
  createUser(input: CreateUserInput!): CreateUserPayload!
  updateUser(input: UpdateUserInput!): UpdateUserPayload!
}

input CreateUserInput {
  email: String!
  name: String!
  role: UserRole!
}

type CreateUserPayload {
  "The user that was just created."
  user: User!
}
```

The mental model is **`UserInput` in, `User` out** — the wrapper is plumbing
around that core:

- **The payload surfaces the affected resource** under a field named for it
  (`user: User`). That's what the caller came for.
- **The input's fields mirror the resource's fields** (`email`, `name`, `role`),
  so composing a call is just "fill in the User."
- **Keep the resource's own vocabulary.** `CreateUserInput` /
  `CreateUserPayload` are good — the word `User` is right there. Avoid
  `UserDraft`, `UserAttributes`, `UserParams`: renaming the concept forces a
  lookup.

**Extra fields and arguments are expected and welcome** — the correspondence
just shouldn't get buried:

- Payloads often carry more than the resource: `clientMutationId`, related
  entities that changed, a server-generated token, a `userErrors` list (§2).
- Inputs often carry more than the resource's fields: `id` on updates,
  `sendWelcomeEmail: Boolean`, an idempotency key.

**Create vs. update:** because each operation has its own input type,
`CreateUserInput` can mark genuinely required fields non-null while
`UpdateUserInput` keeps them nullable (plus `id: ID!`). Document per field when
omitted vs. `null` differ (see §1 nullability).

**Delete:** `deleteUser(input: DeleteUserInput!)` where `DeleteUserInput` is
typically `{ id: ID! }`; the payload exposes the deleted resource (or at least
its `id`) so clients can evict it from cache. A bare `deleteUser(id: ID!)`
argument is an acceptable shorthand when your tooling doesn't force a wrapper.

### 3.4 Versionless by default; suffix `V2` when you truly can't be

GraphQL is designed to evolve without versions. Default posture:

- **Add, don't change.** New optional arguments, new nullable input fields, new
  fields on types — all backward compatible, all fine to ship.
- **Deprecate, don't delete.** Mark the old field
  `@deprecated(reason: "Use <x> instead. Removal scheduled for <date/release>.")`,
  watch usage drop to zero, then remove it in a planned cleanup.

When a mutation's **input shape or semantics must change incompatibly** and
additive evolution genuinely can't express it, do not mutate it in place.
Create a sibling with the same name plus a capital-`V` version suffix, and
deprecate the original:

```graphql
type Mutation {
  updateUser(input: UpdateUserInput!): UpdateUserPayload!
    @deprecated(reason: "Use updateUserV2. Removal scheduled for 2026-12-01.")
  updateUserV2(input: UpdateUserV2Input!): UpdateUserV2Payload!
}
```

- Suffix is `V2`, `V3`, … — capital `V`, no separator, on the mutation name,
  and on its input/payload types when their shape changed
  (`UpdateUserV2Input`, `UpdateUserV2Payload`).
- Only mutations normally need this. Query and type evolution is almost always
  expressible additively.
- Two versions is the ceiling — once `V2` is adopted, retire the original on a
  schedule so you're never carrying three.

---

## Quick reference

| Situation | Do | Don't |
| --- | --- | --- |
| Read one record | `user(id: ID!): User` | `getUser`, `userById` |
| Read a collection | `users: UserConnection!` | `fetchUsers`, `userList` |
| Create / update / delete | `createUser` / `updateUser` / `deleteUser` | `newUser`, `modifyUser`, `removeUser` |
| Domain action | `publishArticle(id: ID!)` | `updateArticle(input: { published: true })` |
| Mutation input type | `CreateUserInput` / `UpdateUserInput`, fields mirror `User` | `UserDraft`, `UserAttributes`, `UserParams` |
| Mutation payload | `CreateUserPayload` exposing `user: User` (+ extras as needed) | payload that hides or renames the affected resource |
| Any error | raise top-level, `extensions.code` from `ErrorCode` enum | return `{ error }` inside `data` |
| Incompatible mutation change | add `updateUserV2`, deprecate `updateUser` | edit `updateUser` in place |
| Boolean field | `isActive`, `hasSubscription` | `active`, `subscription` |
| List field | `[Post!]!`, plural name | `[Post]`, singular name |
| Identity | `ID!` | `Int!` for ids |
| Relationship | `author: User!` | `authorId: ID!` |

## Reference files

- **[references/relay-and-tooling.md](references/relay-and-tooling.md)** — full
  Relay connection/`PageInfo` boilerplate, the `graphql-eslint` naming-rule
  config that enforces §1 and §3, a CI workflow for schema linting, and the
  common custom-scalar table.
