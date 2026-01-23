# /gleam:web-endpoint - Create Web API Endpoint

Add a new web endpoint to a Wisp application following official Wisp patterns.

## Usage

```
/gleam:web-endpoint <resource> [--with-db]
```

## Examples

```
/gleam:web-endpoint users
/gleam:web-endpoint products --with-db
/gleam:web-endpoint comments
```

## Workflow

### 1. Create Feature Module

Create `src/app/web/<resource>.gleam`:

```gleam
//// Handlers for /<resource> endpoints.

import gleam/dynamic/decode
import gleam/http.{Delete, Get, Post}
import gleam/json
import gleam/result.{try}
import wisp.{type Request, type Response}
import app/web.{type Context}

// TYPES -----------------------------------------------------------------------

pub type Item {
  Item(name: String, description: String)
}

// HANDLERS --------------------------------------------------------------------

/// Handle /<resource> - list and create
pub fn all(req: Request, ctx: Context) -> Response {
  case req.method {
    Get -> list_items(ctx)
    Post -> create_item(req, ctx)
    _ -> wisp.method_not_allowed([Get, Post])
  }
}

/// Handle /<resource>/:id - read, update, delete
pub fn one(req: Request, ctx: Context, id: String) -> Response {
  case req.method {
    Get -> read_item(ctx, id)
    Delete -> delete_item(ctx, id)
    _ -> wisp.method_not_allowed([Get, Delete])
  }
}

// LIST ------------------------------------------------------------------------

fn list_items(ctx: Context) -> Response {
  case db.list_items(ctx.db) {
    Ok(items) -> {
      let json = json.to_string(json.object([
        #("items", json.array(items, item_to_json)),
      ]))
      wisp.json_response(json, 200)
    }
    Error(_) -> wisp.internal_server_error()
  }
}

// CREATE ----------------------------------------------------------------------

fn create_item(req: Request, ctx: Context) -> Response {
  // Parse JSON body
  use json_body <- wisp.require_json(req)
  
  let result = {
    // Decode JSON
    use item <- try(
      decode.run(json_body, item_decoder())
      |> result.replace_error(Nil)
    )
    
    // Save to database
    use id <- try(db.create_item(ctx.db, item))
    
    // Return ID
    Ok(json.to_string(json.object([#("id", json.string(id))])))
  }
  
  case result {
    Ok(json) -> wisp.json_response(json, 201)
    Error(_) -> wisp.unprocessable_content()
  }
}

// READ ------------------------------------------------------------------------

fn read_item(ctx: Context, id: String) -> Response {
  case db.find_item(ctx.db, id) {
    Ok(item) -> {
      let json = json.to_string(json.object([
        #("id", json.string(id)),
        #("name", json.string(item.name)),
        #("description", json.string(item.description)),
      ]))
      wisp.json_response(json, 200)
    }
    Error(_) -> wisp.not_found()
  }
}

// DELETE ----------------------------------------------------------------------

fn delete_item(ctx: Context, id: String) -> Response {
  case db.delete_item(ctx.db, id) {
    Ok(_) -> wisp.no_content()
    Error(_) -> wisp.not_found()
  }
}

// DECODERS & ENCODERS ---------------------------------------------------------

fn item_decoder() -> decode.Decoder(Item) {
  use name <- decode.field("name", decode.string)
  use description <- decode.field("description", decode.string)
  decode.success(Item(name:, description:))
}

fn item_to_json(item: Item) -> json.Json {
  json.object([
    #("name", json.string(item.name)),
    #("description", json.string(item.description)),
  ])
}
```

### 2. Add Routes to Router

In `src/app/router.gleam`, add the routes:

```gleam
import app/web/<resource>

pub fn handle_request(req: Request, ctx: Context) -> Response {
  use req <- web.middleware(req)
  
  case wisp.path_segments(req) {
    // ... existing routes ...
    
    // Add new resource routes
    ["<resource>"] -> <resource>.all(req, ctx)
    ["<resource>", id] -> <resource>.one(req, ctx, id)
    
    _ -> wisp.not_found()
  }
}
```

### 3. (Optional) With Database

If using `--with-db`, the feature module uses the database from context:

```gleam
// In app/web.gleam - ensure Context has db
pub type Context {
  Context(db: pog.Connection)
}

// In feature module - use ctx.db
fn list_items(ctx: Context) -> Response {
  case pog.query(
    "SELECT id, name, description FROM items",
    ctx.db,
    [],
    item_row_decoder(),
  ) {
    Ok(returned) -> {
      let json = json.to_string(json.object([
        #("items", json.array(returned.rows, item_to_json)),
      ]))
      wisp.json_response(json, 200)
    }
    Error(_) -> wisp.internal_server_error()
  }
}
```

## Patterns

### Single Resource (no collection)

For resources like `/profile` or `/settings`:

```gleam
/// Handle /profile - read and update only
pub fn profile(req: Request, ctx: Context) -> Response {
  case req.method {
    Get -> read_profile(ctx)
    Post -> update_profile(req, ctx)
    _ -> wisp.method_not_allowed([Get, Post])
  }
}
```

### Nested Resources

For routes like `/users/:user_id/posts`:

```gleam
// In router
case wisp.path_segments(req) {
  ["users", user_id, "posts"] -> user_posts.all(req, ctx, user_id)
  ["users", user_id, "posts", post_id] -> user_posts.one(req, ctx, user_id, post_id)
}

// In handler
pub fn all(req: Request, ctx: Context, user_id: String) -> Response {
  case req.method {
    Get -> list_user_posts(ctx, user_id)
    Post -> create_user_post(req, ctx, user_id)
    _ -> wisp.method_not_allowed([Get, Post])
  }
}
```

### With Query Parameters

```gleam
fn list_items(req: Request, ctx: Context) -> Response {
  let query = wisp.get_query(req)
  
  let page = 
    list.key_find(query, "page")
    |> result.try(int.parse)
    |> result.unwrap(1)
  
  let limit = 
    list.key_find(query, "limit")
    |> result.try(int.parse)
    |> result.unwrap(20)
    |> int.clamp(1, 100)
  
  let search = list.key_find(query, "q") |> option.from_result
  
  case db.list_items(ctx.db, page, limit, search) {
    Ok(result) -> {
      let json = json.to_string(json.object([
        #("items", json.array(result.items, item_to_json)),
        #("total", json.int(result.total)),
        #("page", json.int(page)),
        #("limit", json.int(limit)),
      ]))
      wisp.json_response(json, 200)
    }
    Error(_) -> wisp.internal_server_error()
  }
}
```

### Error Responses

```gleam
fn create_item(req: Request, ctx: Context) -> Response {
  use json_body <- wisp.require_json(req)
  
  let result = {
    use item <- try(decode.run(json_body, item_decoder()))
    use validated <- try(validate_item(item))
    db.create_item(ctx.db, validated)
  }
  
  case result {
    Ok(id) -> {
      let json = json.to_string(json.object([#("id", json.string(id))]))
      wisp.json_response(json, 201)
    }
    
    // Decode/validation errors
    Error(DecodeError(errors)) -> {
      let json = json.to_string(json.object([
        #("error", json.string("Validation failed")),
        #("details", json.array(errors, fn(e) { json.string(e.path) })),
      ]))
      wisp.Response(
        ..wisp.response(422),
        body: wisp.Text(json),
      )
      |> wisp.set_header("content-type", "application/json")
    }
    
    // Uniqueness violation
    Error(DbError(pog.ConstraintViolated(constraint: "items_name_key", ..))) -> {
      let json = json.to_string(json.object([
        #("error", json.string("Name already exists")),
      ]))
      wisp.Response(
        ..wisp.response(409),
        body: wisp.Text(json),
      )
      |> wisp.set_header("content-type", "application/json")
    }
    
    // Other database errors
    Error(DbError(_)) -> wisp.internal_server_error()
  }
}
```

## Checklist

- [ ] Create feature module in `src/app/web/<resource>.gleam`
- [ ] Define types for the resource
- [ ] Implement `all` handler (list + create)
- [ ] Implement `one` handler (read + update + delete)
- [ ] Add decoders for request parsing
- [ ] Add encoders for JSON responses
- [ ] Add routes to `src/app/router.gleam`
- [ ] Handle error cases appropriately
- [ ] Add tests in `test/app/web/<resource>_test.gleam`

## References

- [Wisp Examples](https://github.com/lpil/wisp/tree/main/examples)
- [Wisp Documentation](https://hexdocs.pm/wisp/)
- [gleam-web-development skill](../skills/gleam-web-development/SKILL.md)
