---
name: mobile-api
description: >-
  Designs JSON APIs consumed by a first-party client you own — a mobile app or
  SPA. Covers serialization with Alba, screen-shaped endpoints, consistent error
  and pagination envelopes, and an OpenAPI contract that generates typed clients.
  Use when building an API for a mobile app, defining response shapes, choosing a
  serializer, or when the user mentions mobile client, API contract, OpenAPI,
  Swagger, or typed client generation. WHEN NOT: URL versioning and controller
  structure (use api-versioning), the strict JSON:API spec with compound
  documents, HTML/Turbo Stream responses, or GraphQL.
paths: "app/serializers/**/*.rb, app/controllers/api/**/*.rb, spec/requests/api/**/*.rb"
---

# JSON APIs for a First-Party Client

This covers the case where **you own both sides** — a Rails backend and a mobile app or
SPA built by the same team. That ownership is what makes the design cheap: the contract
can be whatever serves the client best, and both sides ship together.

For where endpoints live and how they are versioned, see the `api-versioning` skill. This
skill decides what they return.

## Why Not JSON:API Here

JSON:API standardizes the envelope for clients you do **not** control. With one
first-party client it charges without delivering:

- `included` plus linkage by `type`/`id` forces the client to index and join on every
  screen, instead of rendering what arrived.
- The envelope repeats `type` and wraps every attribute set, inflating the
  highest-volume endpoints on cellular connections.
- Aggregated lists, search results, and ranked collections are views, not resources.
  Resource semantics fight them.

What actually buys rigor for a mobile client is a **contract**, not an envelope. That is
OpenAPI: typed Swift/Kotlin models generated from the schema, request validation, and
documentation that cannot drift from the tests. Adopt JSON:API only if third parties or
many client types you do not control appear later — migrating then is cheaper than
carrying the cost now.

## Serializer

**Use `alba`.** Actively maintained (v4.0.0, August 2026), fast, zero runtime dependencies,
and it produces the shape you define rather than a prescribed envelope.

```ruby
# Gemfile
gem "alba"
```

```ruby
# app/serializers/entity_resource.rb
class EntityResource
  include Alba::Resource

  root_key :entity, :entities

  attributes :id, :name, :created_at

  attribute :editable_by_viewer do |entity|
    params[:editable_ids].include?(entity.id)
  end

  one :owner, resource: OwnerResource
end
```

```ruby
EntityResource.new(entities, params: { editable_ids: editable_ids }).serializable_hash
```

`blueprinter` and `panko_serializer` are comparable and also maintained — pick one and
keep it. Avoid `jsonapi-serializer` and `fast_jsonapi` unless you deliberately chose the
JSON:API spec; avoid `active_model_serializers` for new work.

See [references/serialization.md](references/serialization.md) for resource classes,
conditional attributes, params propagation, metadata, and avoiding N+1 in serializers.

## Response Conventions

Pick these once and never vary them — an inconsistent envelope costs the client more than
a verbose one.

- **Envelope**: `{ "data": ... }` for success, `{ "errors": [...] }` for failure. Never a
  bare array at the top level; it leaves no room to add `meta` later.
- **Ids**: serialize as-is. Only stringify if the client language loses precision on
  64-bit integers.
- **Timestamps**: ISO 8601 UTC (`created_at.iso8601`), never Unix epochs or localized
  strings.
- **Absent values**: `null`, not omitted. An omitted key and a null value are different
  states in a typed client, and only one of them is intentional.
- **Booleans**: name them as predicates the client can read — `editable_by_viewer`, not `edit`.

## Screen-Shaped Endpoints

A mobile screen that needs three round trips is a slow screen. Design the endpoint around
the screen, not around the table: one request should carry everything one screen renders,
including the associated records, the counters, and any viewer-dependent state.

```ruby
GET /api/v1/entities
{ "data": [ { "id": 1, "name": "...", "created_at": "...",
              "owner": { "id": 7, "display_name": "..." },
              "children_count": 12,
              "editable_by_viewer": true } ],
  "meta": { "next_cursor": "eyJpZCI6MX0" } }
```

Denormalize deliberately, not accidentally. Counters belong in counter caches, viewer
state in a single preloaded query — never in a per-row callback inside the serializer.

## Errors and Pagination

Both need one shape used everywhere. Validation failures, authorization denials, and
not-found must be the same envelope, or the client grows a branch per endpoint.

See [references/errors-and-pagination.md](references/errors-and-pagination.md) for the
error envelope, `rescue_from` wiring, Pundit and ActiveRecord mapping, and cursor
pagination — which any list ordered by recency needs, since offset pagination duplicates
and skips rows as new records arrive at the head.

## Contract

Generate OpenAPI from request specs with `rswag`, so the schema is verified by the same
suite that tests behavior and cannot drift.

See [references/openapi.md](references/openapi.md) for the spec DSL, shared component
schemas, and generating typed clients for iOS and Android.

## Authentication

Token auth for a mobile client — sessions and cookies do not fit. See the
`authentication-flow` skill for Rails 8 `has_secure_password` and token issuance;
authorize per record with Pundit as usual (see the `policies` rule).

## Testing

Request specs assert the contract, not just the status. With `rswag` the same spec emits
the schema.

```ruby
it "returns entities with owner and viewer state" do
  get "/api/v1/entities", headers: auth_headers(user)

  expect(response).to have_http_status(:ok)
  entity = response.parsed_body["data"].first
  expect(entity).to include("id", "name", "created_at", "editable_by_viewer")
  expect(entity["owner"]).to include("display_name")
  expect(entity["created_at"]).to match(/\A\d{4}-\d{2}-\d{2}T/)
end
```

## Workflow Checklist

1. Define the screen the endpoint serves, then the payload it needs — in that order.
2. Add an Alba resource under `app/serializers/`; keep query loading in the controller or a query object.
3. Apply the response conventions above; verify timestamps and null handling.
4. Route every failure through the shared error envelope.
5. Use cursor pagination for any list that grows at the head; offset is fine for the rest.
6. Write the request spec as an `rswag` spec so the OpenAPI schema is generated and verified.
7. Regenerate the client models from the schema; treat a schema diff as an API change.

## References

- [references/serialization.md](references/serialization.md) — Alba resources, conditional attributes, params, metadata, preloading
- [references/errors-and-pagination.md](references/errors-and-pagination.md) — error envelope, `rescue_from` wiring, Pundit/ActiveRecord mapping, cursor pagination
- [references/openapi.md](references/openapi.md) — rswag setup, spec DSL, shared schemas, typed client generation
