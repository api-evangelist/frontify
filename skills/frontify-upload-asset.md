---
name: Upload an asset to Frontify
description: Run Frontify's three-step upload handshake to add a new asset to a library, then tag it and set custom metadata.
api: graphql/frontify.graphql
generated: '2026-08-13'
method: generated
source: graphql/frontify.graphql (every field and input name below was grepped from the live schema)
operations:
  - RootMutation.uploadFile
  - RootMutation.createAsset
  - RootMutation.addAssetTags
  - RootMutation.addCustomMetadata
  - RootMutation.replaceAsset
  - Library.assetsByExternalId
---

# Upload an asset to Frontify

Uploading is a **three-step handshake**, not one call. Step 2 is a plain HTTP PUT that does
not go through GraphQL at all. Getting this wrong is the most common Frontify integration
mistake.

- Scope required: `basic:write` (plus `basic:read` to verify).
- Endpoint: `POST https://{instance}.frontify.com/graphql`.

## Step 1 — reserve the upload

`uploadFile(input: UploadFileInput!)` takes exactly two fields: `filename: String!` and
`size: BigInt!` (the byte length). It returns `UploadFile { id, urls }`.

```graphql
mutation Reserve($filename: String!, $size: BigInt!) {
  uploadFile(input: { filename: $filename, size: $size }) {
    id
    urls
  }
}
```

`urls` is a list of presigned upload URLs. Keep `id` — it is the `fileId` you need in step 3.

## Step 2 — PUT the bytes

HTTP `PUT` the file content to the returned URL(s). Do not add an `Authorization` header —
the URLs are presigned. For a multi-URL response, the file is chunked in order.

## Step 3 — create the asset record

`createAsset(input: CreateAssetInput!)`. Required: `fileId` (from step 1) and `title`.
Everything else is optional but you should set most of it at creation time rather than
patching afterwards:

| Field | Why it matters |
|---|---|
| `externalId` | Your own identifier. The only dedupe handle Frontify offers — set it. |
| `alternativeText` | Accessibility; brand teams audit for this. |
| `description`, `author` | Discoverability. |
| `tags: [TagInput]` | Search. Cheaper here than a follow-up `addAssetTags`. |
| `copyright: CreateCopyrightInput` | Usage rights. |
| `availability: DateTimeRangeInput`, `expiresAt` | Publication window. |
| `parentId` / `directory: [String]` | Placement in the folder tree. |
| `skipFileMetadata` | Defaults `false`; leave it so EXIF/IPTC is ingested. |

```graphql
mutation Create($input: CreateAssetInput!) {
  createAsset(input: $input) { ... }
}
```

## Idempotency — read this before you retry

**Frontify has no idempotency key.** There is no `Idempotency-Key` header and no idempotency
argument on any mutation. If step 3 times out and you retry, you can create a duplicate asset.

The only safe pattern:

1. Always set `externalId` on create.
2. Before retrying, query `library(id:).assetsByExternalId(externalId:)`.
3. If it returns a match, the earlier call succeeded — stop. Do not create again.

Frontify's processing is asynchronous, so a freshly created asset may come back with
`status: PROCESSING` (`AssetStatusType` is `FINISHED | PROCESSING | PROCESSING_FAILED`).
Poll `asset(id:) { status }` rather than assuming the asset is usable immediately, and treat
`PROCESSING_FAILED` as a terminal failure requiring re-upload.

## Adding a new revision instead of a new asset

To version an existing asset rather than create a sibling, run steps 1 and 2 the same way,
then call `replaceAsset` with the new `fileId` instead of `createAsset`.

## Follow-up mutations

- `addAssetTags(input: AddAssetTagsInput!)` — `{ id, tags: [TagInput] }`. Adds. Use
  `syncAssetTags` to make the tag set exactly match, or `removeAssetTags` to subtract.
- `addCustomMetadata` — set values against a `CustomMetadataProperty`. Do **not** use
  `addAssetMetadataFieldValue` / `createMetadataField` / `removeMetadataValue`: the whole
  `MetadataField` family is `@deprecated` with a stated removal date of 2026-07-01, in
  favour of the `CustomMetadata*` types.

## Failure handling

- HTTP 200 with `errors[].extensions.category = "permission"` — missing `basic:write`, or no
  write access to that library. Check `Library.currentUserPermissions` first.
- HTTP 422 `"Invalid query"` — validation failure with no detail. Most often a wrong
  `CreateAssetInput` field name or a missing `fileId`/`title`.
- A failed PUT in step 2 leaves a reserved-but-unused `fileId`; simply restart from step 1.
