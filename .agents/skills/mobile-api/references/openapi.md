# Reference: OpenAPI Contract with rswag

Load this when defining or regenerating the API contract consumed by a mobile client.

## Why This and Not a Written Doc

A hand-maintained API doc drifts the day someone renames a field. `rswag` writes the
OpenAPI schema from request specs, so the contract is produced by the same suite that
verifies behavior — the schema cannot describe an endpoint the tests do not exercise.

For a mobile client that is the whole payoff: typed Swift and Kotlin models generated from
a schema you know is accurate.

## Setup

```ruby
# Gemfile
group :development, :test do
  gem "rswag-specs"
end

# API docs UI, optional — skip if the schema is only consumed by client generators
gem "rswag-api"
gem "rswag-ui"
```

```bash
bin/rails generate rswag:specs:install
```

## Shared Schemas

Define every reusable shape once in `spec/openapi_helper.rb` and reference it. Inline
schemas duplicated across specs are how a contract starts drifting from itself.

```ruby
# spec/openapi_helper.rb
config.openapi_specs = {
  "v1/openapi.json" => {
    openapi: "3.0.0",
    info: { title: "API V1", version: "v1" },
    components: {
      securitySchemes: {
        bearer_auth: { type: :http, scheme: :bearer }
      },
      schemas: {
        error_object: {
          type: :object,
          properties: {
            code: { type: :string },
            field: { type: :string, nullable: true },
            detail: { type: :string }
          },
          required: %w[code detail]
        },
        errors: {
          type: :object,
          properties: {
            errors: { type: :array, items: { "$ref" => "#/components/schemas/error_object" } }
          },
          required: %w[errors]
        },
        owner: {
          type: :object,
          properties: {
            id: { type: :integer },
            display_name: { type: :string },
            image_url: { type: :string, nullable: true }
          },
          required: %w[id display_name]
        },
        entity: {
          type: :object,
          properties: {
            id: { type: :integer },
            name: { type: :string },
            created_at: { type: :string, format: "date-time" },
            owner: { "$ref" => "#/components/schemas/owner" },
            children_count: { type: :integer },
            editable_by_viewer: { type: :boolean }
          },
          required: %w[id name created_at owner children_count editable_by_viewer]
        }
      }
    }
  }
}
```

Mark a field `required` only when the API guarantees it on every response. In a typed
client, `required` becomes non-optional — promising a field you sometimes omit is how you
crash the app on a null.

## Spec DSL

```ruby
# spec/requests/api/v1/entities_spec.rb
require "swagger_helper"

RSpec.describe "Entities API", type: :request do
  path "/api/v1/entities" do
    get "Returns a page of entities" do
      tags "Entities"
      produces "application/json"
      security [bearer_auth: []]
      parameter name: :cursor, in: :query, required: false, schema: { type: :string }

      response 200, "a page of entities" do
        schema type: :object,
               properties: {
                 data: { type: :array, items: { "$ref" => "#/components/schemas/entity" } },
                 meta: {
                   type: :object,
                   properties: { next_cursor: { type: :string, nullable: true } }
                 }
               },
               required: %w[data meta]

        let(:Authorization) { "Bearer #{token_for(user)}" }
        let(:cursor) { nil }

        before { create_list(:entity, 3) }

        run_test! do |response|
          expect(response.parsed_body["data"].size).to eq(3)
        end
      end

      response 401, "missing or invalid token" do
        schema "$ref" => "#/components/schemas/errors"

        let(:Authorization) { "Bearer nope" }
        let(:cursor) { nil }

        run_test!
      end
    end
  end
end
```

`run_test!` performs the request, asserts the status, and validates the body against the
declared schema. A response that does not match fails the spec — which is what keeps the
contract honest. Pass a block to add assertions beyond the shape.

## Generating the Schema

```bash
bundle exec rake rswag:specs:swaggerize
```

Writes `swagger/v1/openapi.json` (path configurable). Commit it: a diff on that file is a
visible API change, and reviewers should see it in the PR alongside the code.

Wire it into CI so a schema regenerated from a changed API fails when it was not committed:

```bash
bundle exec rake rswag:specs:swaggerize && git diff --exit-code swagger/
```

## Generating Clients

```bash
# Swift, for iOS
openapi-generator generate -i swagger/v1/openapi.json -g swift5 -o ../ios/Generated

# Kotlin, for Android
openapi-generator generate -i swagger/v1/openapi.json -g kotlin -o ../android/generated
```

Treat generated client code as build output, never edited by hand. When the schema
changes, regenerate and let the client fail to compile — that compile error is the point.
It surfaces a breaking API change at build time instead of as a crash in production.
