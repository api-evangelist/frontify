---
name: Export a creative from a Frontify template
description: Render a branded creative from a Frontify creative template with your own variable values, then poll the async export job for the result.
api: graphql/frontify.graphql
generated: '2026-08-13'
method: generated
source: graphql/frontify.graphql (every field and input name below was grepped from the live schema)
operations:
  - Brand.creativeTemplates
  - RootQuery.creativeTemplate
  - RootMutation.exportCreative
  - RootQuery.creativeExport
  - RootMutation.cancelExportCreatives
---

# Export a creative from a Frontify template

This is Frontify's automation flow: take an approved, on-brand template, substitute text /
image / colour variables, and render a finished creative. It is **asynchronous** — the
mutation starts a job, it does not return the file.

- Scopes: `basic:read` to list templates, `basic:write` to export.
- Endpoint: `POST https://{instance}.frontify.com/graphql`.

## 1. List the templates available on a brand

```graphql
query Templates($brandId: ID!) {
  brand(id: $brandId) {
    creativeTemplates(limit: 25, page: 1) {
      total
      hasNextPage
      items { id name }
    }
  }
}
```

## 2. Read the template's variables before exporting

```graphql
query Template($id: ID!) {
  creativeTemplate(id: $id) { id name ... }
}
```

Do not guess variable names. Read them off the template — an export with an unknown variable
name is a validation failure Frontify will not explain (see failure handling).

Note that `CreativeTemplate.brandId` and `CreativeTemplate.assetId` are `@deprecated` with a
stated removal date of 2027-01-01; do not build against them.

## 3. Start the export

`exportCreative(input: ExportCreativeInput!)`:

| Field | Type | Notes |
|---|---|---|
| `templateId` | `ID!` | Required. |
| `variables` | `[CreativeVariableInput]` | Your text / image / colour substitutions. |
| `destination` | `ExportCreativeDestinationInput` | Where the rendered asset lands in Frontify. |
| `options` | `ExportCreativeOptionsInput` | Format / render options. |

```graphql
mutation Export($input: ExportCreativeInput!) {
  exportCreative(input: $input) { ... }
}
```

Keep the returned job id.

## 4. Poll the job

```graphql
query ExportStatus($id: ID!) {
  creativeExport(id: $id) {
    id
    status
    result { ... }
  }
}
```

`status` is a `CreativeJobStatus`. Only read `result` once the job reports completion —
`result` is nullable and will be null while the job is running.

Poll with backoff. Frontify publishes no rate limit, but every response returns
`extensions.complexityScore` and there is no published budget, so a tight poll loop is
uncosted risk. Start at ~2s and back off.

## 5. Cancel if needed

`cancelExportCreatives(input:)` cancels one or more jobs that are still processing or
rendering. It is the correct response to a user abort — do not simply stop polling and leave
render capacity consumed.

## Failure handling

- No idempotency key exists. If `exportCreative` times out, retrying may start a **second
  render job**. Prefer polling for a job you may already have started over blind retry, and
  cancel duplicates with `cancelExportCreatives`.
- HTTP 422 `"Invalid query"` — usually a wrong variable name or a malformed
  `ExportCreativeInput`. Frontify returns no `path` or `locations`; diff your document
  against `graphql/frontify.graphql`.
- HTTP 200 with `extensions.category = "permission"` — the token lacks `basic:write` or the
  identity cannot export from that brand.
