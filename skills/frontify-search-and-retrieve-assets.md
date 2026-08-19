---
name: Search and retrieve Frontify assets
description: Find brand assets in a Frontify library or workspace and read their metadata, tags and download URLs.
api: graphql/frontify.graphql
generated: '2026-08-13'
method: generated
source: graphql/frontify.graphql (every field name below was grepped from the live schema)
operations:
  - RootQuery.brands
  - RootQuery.brand
  - RootQuery.library
  - RootQuery.workspaceProject
  - Library.assets
  - Library.assetsByExternalId
  - RootQuery.asset
  - RootQuery.assets
---

# Search and retrieve Frontify assets

Frontify's API is GraphQL. There is no REST endpoint and no OpenAPI. Everything below is a
single `POST` to `https://{instance}.frontify.com/graphql` with `Content-Type: application/json`
and an `Authorization: Bearer <token>` header. `{instance}` is the customer's own Frontify
subdomain — there is no shared host.

## Before you start

- Scope required: `basic:read`. See `scopes/frontify-scopes.yml`.
- Schema introspection works without a token, so you can always re-read the contract. Data
  does not: an unauthenticated data query returns HTTP 200 with
  `errors[0].message = "UserId not set in identity."` and `extensions.category = "permission"`.
- Pagination is `limit` / `page` (defaults 25 / 1). It is NOT Relay — there is no `pageInfo`,
  no `edges`, no cursor. Every list returns `{ total, page, limit, hasNextPage, items }`.

## 1. Find the brand

```graphql
query { brands { id name slug } }
```

## 2. Find the library or workspace

`Brand.libraries` returns typed libraries (`MediaLibrary`, `LogoLibrary`, `IconLibrary`,
`DocumentLibrary`). `Brand.workspaceProjects` returns `Workspace` objects. Do not use
`Brand.projects` — it has been `@deprecated` since a stated removal date of 2023-01-01.

```graphql
query Libraries($brandId: ID!) {
  brand(id: $brandId) {
    libraries(limit: 25, page: 1) {
      total
      hasNextPage
      items { __typename id name }
    }
  }
}
```

## 3. Search assets

`Library.assets` takes an `AssetQueryInput` with `search`, `types` (`[AssetType]`:
`IMAGE | VIDEO | AUDIO | DOCUMENT | FILE`), `sortBy` (`AssetQueryFilterSortType`:
`RELEVANCE | NEWEST | OLDEST | TITLE_ASCENDING | TITLE_DESCENDING`), `filter`, `inFolder`
and `externalId`.

```graphql
query Search($libraryId: ID!, $q: String!) {
  library(id: $libraryId) {
    assets(limit: 25, page: 1, query: { search: $q, types: [IMAGE], sortBy: RELEVANCE }) {
      total
      hasNextPage
      items {
        __typename
        id
        title
        description
        alternativeText
        status
        tags { ... }
        customMetadata { ... }
      }
    }
  }
}
```

`Asset` is an interface. Every result is one of `Image`, `Video`, `Audio`, `Document`,
`File` or `EmbeddedContent` — always select `__typename` and handle all six, or use inline
fragments for type-specific fields (dimensions, duration, page count).

## 4. Read one asset

```graphql
query Asset($id: ID!) {
  asset(id: $id) {
    __typename
    id
    title
    author
    copyright { ... }
    availability { ... }
    expiresAt
    currentUserPermissions { ... }
  }
}
```

`currentUserPermissions` is worth reading before you plan any write: it tells you what this
identity may do with the object without you having to attempt the mutation and fail.

Respect `availability` and `expiresAt` — a Frontify asset can be published with a validity
window, and serving an expired asset is a brand-compliance failure, not a technical one.

## 5. Look an asset up by your own identifier

If your system already stored a Frontify asset under your own key, do not search for it by
title. Use the external-id lookup, the only cursor-paginated field in the schema:

```graphql
query ByExternalId($libraryId: ID!, $externalId: ID!) {
  library(id: $libraryId) {
    assetsByExternalId(externalId: $externalId, limit: 25) { ... }
  }
}
```

## Failure handling

- HTTP 200 with `errors[].extensions.category = "permission"` — token missing, expired, or
  lacking `basic:read`, or the identity has no access to that object. `data` will be
  partially populated; siblings still resolve.
- HTTP 422 `{"errors":[{"message":"Invalid query"}]}` — your document did not validate.
  Frontify returns no `locations` and no `path`, so it will not tell you which field was
  wrong. Validate against `graphql/frontify.graphql` locally before sending.
- HTTP 400 `{"errors":[{"message":"Bad request"}]}` — syntax error.
- Every response carries `extensions.complexityScore`. Frontify publishes no complexity
  budget, so treat it as a relative cost signal: keep queries narrow and avoid deep nesting
  rather than trying to hit a number.
