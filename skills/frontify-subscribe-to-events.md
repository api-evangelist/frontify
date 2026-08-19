---
name: Subscribe to Frontify events with a webhook
description: Install and configure a project webhook, choose which of the 24 asset and workflow events to receive, and verify delivery signatures.
api: graphql/frontify.graphql
generated: '2026-08-13'
method: generated
source: >-
  graphql/frontify.graphql (AssetWebhookEvent, Webhook, ProjectWebhook,
  InstallProjectWebhookInput, ConfigureProjectWebhookInput) + asyncapi/frontify-webhooks.yml
operations:
  - RootMutation.installProjectWebhook
  - RootMutation.configureProjectWebhook
  - RootQuery.webhooks
  - RootMutation.uninstallWebhook
---

# Subscribe to Frontify events with a webhook

Frontify webhooks are **project-scoped**, signed with a per-webhook secret, and have two
payload versions. This is the only push surface Frontify offers — there is no streaming API
and no AsyncAPI document.

- Scopes: `webhooks:read`, `webhooks:write`, `basic:write`.
- Endpoint: `POST https://{instance}.frontify.com/graphql`.
- **These operations are not available through Frontify's MCP server.** An agent connected
  over MCP cannot subscribe Frontify to events; it must call GraphQL directly. See
  `mcp/frontify-tool-crosswalk.yml`.

## 1. Install the webhook

`installProjectWebhook(input: InstallProjectWebhookInput!)` takes `projectId: ID!`,
`notificationUrl: Url!` and `name: String!`.

```graphql
mutation Install($projectId: ID!, $url: Url!, $name: String!) {
  installProjectWebhook(input: { projectId: $projectId, notificationUrl: $url, name: $name }) {
    webhook { id name notificationUrl secret }
  }
}
```

**Capture `secret` on this response.** It is the shared secret you verify delivery signatures
against, and you should treat it as a credential — store it in your secret manager, never in
source.

Your `notificationUrl` must already be live and returning 2xx before you install. Frontify
gives you no delivery-log surface to replay from.

## 2. Configure version and event subscription

`configureProjectWebhook(input: ConfigureProjectWebhookInput!)` takes `projectId: ID!`,
`notificationUrl: String!`, `name: String!`, `version: Int!` (payload version, 1 or 2) and
`subscribeTo: [AssetWebhookEvent!]`.

**`subscribeTo` defaults to everything.** The schema states: "Subscribe to specific webhook
events. If not defined the webhook will subscribe to all events." Always name the events you
actually want — an unfiltered subscription on a busy DAM is a firehose.

The 24 events in `AssetWebhookEvent`:

- Lifecycle — `ASSET_CREATED`, `ASSET_UPDATED`, `ASSET_DELETED`, `ASSET_MOVED`,
  `ASSET_PROCESSED`, `ASSET_REVISION_ADDED`
- Variants — `ASSET_VARIANT_ADDED`, `ASSET_VARIANT_UPDATED`, `ASSET_VARIANT_REMOVED`
- Attachments — `ASSET_ATTACHMENT_ADDED`, `ASSET_ATTACHMENT_REMOVED`
- Relations — `ASSET_RELATION_ADDED`, `ASSET_RELATION_REMOVED`
- Organization — `ASSET_COLLECTION_ADDED`, `ASSET_COLLECTION_REMOVED`, `ASSET_TAG_ADDED`,
  `ASSET_TAG_REMOVED`, `ASSET_TARGET_CHANGED`, `ASSET_CUSTOM_METADATA_UPDATED`
- Licensing — `ASSET_LICENCE_ADDED`, `ASSET_LICENCE_REMOVED` (note Frontify's British
  spelling — `LICENCE`, not `LICENSE`)
- Workflow — `WORKFLOW_TASK_ASSIGNED`, `WORKFLOW_TASK_MOVED`, `WORKFLOW_TASK_DELETED`

For an ingest pipeline that mirrors Frontify into another system, `ASSET_CREATED`,
`ASSET_UPDATED`, `ASSET_DELETED`, `ASSET_MOVED` and `ASSET_PROCESSED` are usually enough.
Note that `ASSET_CREATED` can fire before the file is usable — wait for `ASSET_PROCESSED`, or
re-read `asset(id:) { status }`, before you download.

## 3. Verify every delivery

Frontify signs deliveries against the webhook `secret`. Reject any request whose signature
does not verify — replaying a captured sample will not validate, because only events genuinely
sent by Frontify carry a valid event signature.

The exact signature header name and algorithm are published only on Frontify's JS-rendered
webhooks documentation page (`developer.frontify.com/d/wJcTnsuhwb6T/webhooks`) and are not
captured in this repository. Read them there before implementing the verifier — do not guess
a scheme.

## 4. List and remove

```graphql
query { webhooks(limit: 25, page: 1) { total hasNextPage items { id name notificationUrl } } }
```

`uninstallWebhook(input: UninstallWebhookInput!)` removes one. Its `webhook` return field is
`@deprecated` (stated removal 2026-07-01) in favour of the returned `id` — select `id`.

## Payload caution

Frontify publishes no per-event payload schema anywhere machine-readable. Write your consumer
defensively: match on the event type, treat every other field as optional, and re-read the
object from GraphQL by id rather than trusting the payload to carry everything you need.
