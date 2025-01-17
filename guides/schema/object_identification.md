---
layout: guide
doc_stub: false
search: true
title: Object Identification
section: Schema
desc: Working with Relay-style global IDs
index: 8
---

Relay uses [global object identification](https://facebook.github.io/relay/graphql/objectidentification.htm) to support some of its features:

- __Caching__: Unique IDs are used as primary keys in Relay's client-side cache.
- __Refetching__: Relay uses unique IDs to refetch objects when it determines that its cache is stale. (It uses the `Query.node` field to refetch objects.)

### Defining UUIDs

You must provide a function for generating UUIDs and fetching objects with them. In your schema, define `self.id_from_object` and `self.object_from_id`:

```ruby
class MySchema < GraphQL::Schema
  def self.id_from_object(object, type_definition, query_ctx)
    # Call your application's UUID method here
    # It should return a string
    MyApp::GlobalId.encrypt(object.class.name, object.id)
  end

  def self.object_from_id(id, query_ctx)
    class_name, item_id = MyApp::GlobalId.decrypt(id)
    # "Post" => Post.find(item_id)
    Object.const_get(class_name).find(item_id)
  end
end
```

An unencrypted ID generator is provided in the gem. It uses `Base64` to encode values. You can use it like this:

```ruby
class MySchema < GraphQL::Schema
  # Create UUIDs by joining the type name & ID, then base64-encoding it
  def self.id_from_object(object, type_definition, query_ctx)
    GraphQL::Schema::UniqueWithinType.encode(type_definition.graphql_name, object.id)
  end

  def self.object_from_id(id, query_ctx)
    type_name, item_id = GraphQL::Schema::UniqueWithinType.decode(id)
    # Now, based on `type_name` and `item_id`
    # find an object in your application
    # ....
  end
end
```

You could also use the [`GlobalID`](https://github.com/rails/globalid) gem (included in Ruby on Rails since v4.2) to achieve this:

```ruby
class MySchema < GraphQL::Schema
  def self.id_from_object(object, type_definition, query_ctx)
    # Generates a URI like `gid://myproject/User/1` base64-encoded.
    object.to_gid.to_param
  end

  def self.object_from_id(id, query_ctx)
    # Finds the object using the base64-encoded URI.
    GlobalID.find(id)
  end
end
```

### Node interface

One requirement for Relay's object management is implementing the `"Node"` interface.

To implement the node interface, add {{ "GraphQL::Types::Relay::Node" | api_doc }} to your definition:

```ruby
class Types::PostType < GraphQL::Schema::Object
  # Implement the "Node" interface for Relay
  implements GraphQL::Types::Relay::Node
  # ...
end
```

To tell GraphQL how to resolve members of the `Node` interface, you must also define `Schema.resolve_type`:

```ruby
class MySchema < GraphQL::Schema
  # You'll also need to define `resolve_type` for
  # telling the schema what type Relay `Node` objects are
  def self.resolve_type(type, obj, ctx)
    case obj
    when Post
      Types::PostType
    when Comment
      Types::CommentType
    else
      raise("Unexpected object: #{obj}")
    end
  end
end
```

### UUID fields

Relay Nodes must have a field named `"id"` which returns a globally unique ID.

To add a UUID field named `"id"`, use the `global_id_field` helper:

```ruby
class Types::PostType < GraphQL::Schema::Object
  # `id` exposes the UUID
  global_id_field :id
  # ...
end
```

This field will call the previously-defined `id_from_object` class method.

### `node` field (find-by-UUID)

You should also provide a root-level `node` field so that Relay can refetch objects from your schema. You can attach it like this:

```ruby
class Types::QueryType < GraphQL::Schema::Object
  # Used by Relay to lookup objects by UUID:
  # Add `node(id: ID!)
  include GraphQL::Types::Relay::HasNodeField
  # ...
end
```

### `nodes` field

You can also provide a root-level `nodes` field so that Relay can refetch objects by IDs:

```ruby
class Types::QueryType < GraphQL::Schema::Object
  # Fetches a list of objects given a list of IDs
  # Add `nodes(ids: [ID!]!)`
  include GraphQL::Types::Relay::HasNodesField
  # ...
end
```
