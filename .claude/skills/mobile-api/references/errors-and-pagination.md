# Reference: Error Envelope and Pagination

Load this when wiring API failure responses or paginating a list endpoint.

## One Error Shape Everywhere

Every failure — validation, authorization, not-found, rate limit — returns the same
envelope. A client that can parse one error can parse all of them; a client facing a
different shape per endpoint grows a branch per endpoint.

```json
{
  "errors": [
    { "code": "blank", "field": "name", "detail": "can't be blank" },
    { "code": "too_long", "field": "name", "detail": "is too long (maximum is 100 characters)" }
  ]
}
```

- `errors` is always an **array**, even for a single error. Validation failures are
  naturally plural, and a client that special-cases one is a client that breaks on two.
- `code` is a stable, machine-readable symbol. The client branches on this. It must not
  change when the wording changes.
- `field` is `null` for errors that are not attribute-specific.
- `detail` is human-readable and safe to show. Never leak a backtrace, SQL, or an internal
  class name into it.

## Wiring

Handle it once in the API base controller. Every controller below inherits it, so no
action needs a rescue of its own.

```ruby
# app/controllers/api/base_controller.rb
module Api
  class BaseController < ActionController::API
    rescue_from ActiveRecord::RecordNotFound,  with: :not_found
    rescue_from ActiveRecord::RecordInvalid,   with: :unprocessable
    rescue_from Pundit::NotAuthorizedError,    with: :forbidden
    rescue_from ActionController::ParameterMissing, with: :bad_request

    private

    def render_errors(errors, status:)
      render json: { errors: Array(errors) }, status: status
    end

    def not_found(exception)
      render_errors({ code: "not_found", field: nil,
                      detail: "#{exception.model} not found" },
                    status: :not_found)
    end

    def unprocessable(exception)
      render_errors(serialize_model_errors(exception.record), status: :unprocessable_entity)
    end

    def forbidden(_exception)
      render_errors({ code: "forbidden", field: nil,
                      detail: "You are not allowed to perform this action" },
                    status: :forbidden)
    end

    def bad_request(exception)
      render_errors({ code: "parameter_missing", field: exception.param.to_s,
                      detail: "is required" },
                    status: :bad_request)
    end

    def serialize_model_errors(record)
      record.errors.map do |error|
        { code: error.type.to_s, field: error.attribute.to_s, detail: error.message }
      end
    end
  end
end
```

`error.type` gives the stable symbol (`:blank`, `:taken`, `:too_long`) while
`error.message` gives the translated text. That split is exactly the `code`/`detail`
split the client needs — do not send only the message.

## Status Codes

| Situation | Status | `code` |
|---|---|---|
| Validation failed | 422 | the attribute's `error.type` |
| Not authenticated | 401 | `unauthenticated` |
| Authenticated but denied | 403 | `forbidden` |
| Record missing, or hidden from this viewer | 404 | `not_found` |
| Malformed or missing parameter | 400 | `parameter_missing` |
| Rate limited | 429 | `rate_limited` |

Return 404 rather than 403 when the existence of a record is itself private. A 403 confirms
that the record exists, which leaks membership to anyone able to enumerate ids. Decide this
per resource: it is a privacy requirement of the domain, not a default.

## Cursor Pagination

Offset pagination breaks on any list that grows at the head. New rows arrive between
requests, so page 2 re-sends rows already shown and skips others. Cursors are stable
because they describe a position in the ordering, not a count.

```ruby
# app/queries/entities/cursor_query.rb
module Entities
  class CursorQuery
    # Page size is a product decision — pick it per endpoint, do not inherit this number.
    PAGE_SIZE = 25

    def initialize(scope:, cursor: nil, page_size: PAGE_SIZE)
      @scope = scope
      @cursor = cursor
      @page_size = page_size
    end

    def call
      relation = @scope.order(created_at: :desc, id: :desc).limit(@page_size)
      return relation if @cursor.blank?

      relation.where("(created_at, id) < (?, ?)", *decode(@cursor))
    end

    def self.next_cursor(records, page_size: PAGE_SIZE)
      return nil if records.size < page_size

      last = records.last
      Base64.urlsafe_encode64({ created_at: last.created_at.iso8601(6), id: last.id }.to_json)
    end

    private

    def decode(cursor)
      payload = JSON.parse(Base64.urlsafe_decode64(cursor))
      [payload.fetch("created_at"), payload.fetch("id")]
    rescue ArgumentError, JSON::ParserError, KeyError
      raise ActionController::BadRequest, "invalid cursor"
    end
  end
end
```

The tuple comparison `(created_at, id) < (?, ?)` is what makes this correct — ordering by
a timestamp alone is ambiguous when two rows share it, and the duplicate resurfaces on the
next page. Index it to match: `add_index :entities, [:created_at, :id], order: { created_at: :desc, id: :desc }`.

Response:

```json
{ "data": [ ... ], "meta": { "next_cursor": "eyJpZCI6MX0" } }
```

`next_cursor` is `null` on the last page. The client stops when it is null — never by
comparing a page count it should not have to track.

Offset pagination is still correct for lists that do not grow at the head — a settings
table, an archive sorted oldest-first, any collection with a fixed order. Use it there and
keep cursors for the lists that receive new rows at the top.
