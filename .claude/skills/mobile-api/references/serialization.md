# Reference: Serialization with Alba

Load this when defining or changing serializers for a first-party JSON API.

## Resource Classes

Serializers live in `app/serializers/` and are named after what they render, suffixed
`Resource`. One resource per shape, not necessarily one per model — a list row and a
detail screen render the same record differently, and two small resources beat one
resource full of conditionals.

```ruby
# app/serializers/entity_resource.rb
class EntityResource
  include Alba::Resource

  root_key :entity, :entities

  attributes :id, :name, :created_at

  one :owner, resource: OwnerResource
end
```

`root_key` takes singular and plural forms; Alba picks based on what it is given. Call
`serializable_hash` when the controller will render it, `serialize` when you need the JSON
string directly.

```ruby
EntityResource.new(entity).serializable_hash               # single
EntityResource.new(entities).serializable_hash             # collection, same class
EntityResource.new(entities).serialize(root_key: :entities) # explicit root
```

## Computed Attributes

A block attribute receives the object. Keep the block to formatting and reads that are
already loaded — anything that queries belongs in the controller or a query object.

```ruby
class EntityResource
  include Alba::Resource

  attributes :id, :name

  attribute :created_at do |entity|
    entity.created_at.iso8601
  end

  attribute :summary do |entity|
    entity.description.truncate(140)
  end
end
```

## Conditional Attributes and Params

`params` flows from the constructor into every attribute and nested resource. Use it for
viewer-dependent state, never for authorization decisions — those belong in a Pundit policy.

```ruby
class EntityResource
  include Alba::Resource

  attributes :id, :name

  attribute :editable_by_viewer do |entity|
    params[:editable_ids].include?(entity.id)
  end

  attributes :internal_notes, if: proc { params[:staff] }
end

EntityResource.new(entities, params: { staff: current_user.admin? }).serializable_hash
```

Nested resources inherit params by default. Override per association when a child should
see something different:

```ruby
class OwnerResource
  include Alba::Resource

  one :latest_entity, resource: EntityResource, params: { staff: false }
end
```

## Metadata

`meta` has access to `object`, so one resource can describe both a record and a collection.

```ruby
class EntityResource
  include Alba::Resource

  root_key :entity, :entities

  attributes :id, :name

  meta do
    { count: object.size } if object.is_a?(Enumerable)
  end
end
```

For pagination, prefer building `meta` in the controller alongside the cursor — see
`errors-and-pagination.md`. Keep the serializer ignorant of request state.

## Preloading

A serializer that triggers queries is the most common source of N+1 in an API. Alba does
not load anything for you; whatever the resource touches must already be loaded.

```ruby
# app/controllers/api/v1/entities_controller.rb
entities = Entity.includes(:owner)
                 .merge(policy_scope(Entity))
                 .limit(PAGE_SIZE)

render json: EntityResource.new(entities, params: serializer_params).serializable_hash
```

Rules that keep it fast:

- Every `one`/`many` in a resource needs a matching `includes` at the query site.
- Counts come from counter caches (`children_count`), never `entity.children.count` in a block.
- Viewer state comes from one preloaded set, not a query per row:

```ruby
editable_ids = policy_scope(Entity).where(id: entities.map(&:id)).pluck(:id).to_set
EntityResource.new(entities, params: { editable_ids: editable_ids })
```

Assert it in specs so a regression fails the build:

```ruby
it "loads the collection without N+1" do
  create_list(:entity, 3, :with_owner)

  expect { get "/api/v1/entities", headers: auth_headers(user) }
    .to make_database_queries(count: 1..5)
end
```

## Choosing Between Serializers

| Gem | Status | Notes |
|---|---|---|
| `alba` | v4.0.0, 2026-08 | Default here. Zero runtime dependencies, fast, plain Ruby DSL |
| `blueprinter` | v1.3.0, 2026-04 | Comparable. View-based DSL, good when one model has many shapes |
| `panko_serializer` | v0.8.5, 2026-04 | Fastest of the three; C extension, tighter coupling to ActiveRecord |
| `jsonapi-serializer` | v2.2.0, 2021-03 | Only if you deliberately chose the JSON:API spec |
| `active_model_serializers` | v0.10.16, 2025-12 | Long-stalled history; avoid for new work |

Any of the first three is a defensible choice. What is not defensible is mixing them, or
hand-rolling `render json:` hashes alongside them — the shape drifts, and the OpenAPI
schema stops matching reality.
