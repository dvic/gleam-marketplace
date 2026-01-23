---
name: gleam-web-development
description: Guides Claude through Gleam backend web development with Wisp and Mist. Use for REST APIs, web services, and server-side rendering. For frontend/SPA development with Lustre, use the gleam-lustre-development skill instead.
---

# Gleam Web Development Skill

This skill guides Claude Code through Gleam web development workflows.

## Primary Sources

1. **[Wisp Documentation](https://hexdocs.pm/wisp/)** - Practical web framework
2. **[Mist Documentation](https://hexdocs.pm/mist/)** - HTTP server
3. **[Lustre Documentation](https://hexdocs.pm/lustre/)** - Frontend framework
4. **[Gleam HTTP Documentation](https://hexdocs.pm/gleam_http/)** - HTTP types
5. **[Deploying to Fly.io](https://gleam.run/deployment/fly/)** - Deployment guide

## Web Stack Components

### Backend Frameworks
- **[Wisp](https://hexdocs.pm/wisp/)** (v2.2.0) - Practical server-side framework
- **[Mist](https://hexdocs.pm/mist/)** (v5.0.4) - Lightweight HTTP server
- **[Arctic](https://hexdocs.pm/arctic/)** - Fast response times, serverless-friendly

### Frontend Frameworks
- **[Lustre](https://hexdocs.pm/lustre/)** (v5.5.2) - Elm-inspired framework for SPAs, HTML templates, server components
- **[Nakai](https://hexdocs.pm/nakai/)** - HTML generation library

### HTTP Clients
- **[Gleam HTTP Client (httpc)](https://hexdocs.pm/gleam_httpc/)** - Erlang's built-in HTTP client
- **[Gleam Fetch](https://hexdocs.pm/gleam_fetch/)** - JavaScript Fetch API
- **[Hackney](https://hexdocs.pm/gleam_hackney/)** - Alternative Erlang HTTP client

## Handler Pattern: Parse → Process → Present

**ALL handlers should follow this three-part structure:**

### 1. PARSE - Extract and Validate Input

Extract data from the request and validate it:

```gleam
fn create_user(req: Request, ctx: Context) -> Response {
  // Parse JSON body
  use json_body <- wisp.require_json(req)

  // Decode and validate
  let result = {
    use request <- result.try(
      decode.run(json_body, user_request_decoder())
      |> result.map_error(DecodeError),
    )

    // Domain validation
    use validated_user <- result.try(
      user.new(request.email, request.name)
      |> result.map_error(DomainError),
    )

    // Continue to process step...
  }
}
```

**Parse step includes:**
- Query parameters: `wisp.get_query(req)`
- Path parameters: passed as function arguments
- JSON body: `wisp.require_json(req)` + `decode.run()`
- Headers: `wisp.get_header(req, "...")`
- Domain validation: `domain_type.new(...)`

### 2. PROCESS - Execute Business Logic

Perform the actual operation (database, external API, computation):

```gleam
fn create_user(req: Request, ctx: Context) -> Response {
  use json_body <- wisp.require_json(req)

  let result = {
    // Parse (from step 1)
    use request <- result.try(decode.run(json_body, decoder()))
    use validated_user <- result.try(user.new(request.email, request.name))

    // PROCESS - Business logic
    db.create_user(ctx.db, validated_user)
    |> result.map_error(DatabaseError)
  }

  // Continue to present step...
}
```

**Process step includes:**
- Database operations
- External API calls
- Business calculations
- State modifications

### 3. PRESENT - Convert Result to HTTP Response

Transform the result into an HTTP response:

```gleam
fn create_user(req: Request, ctx: Context) -> Response {
  use json_body <- wisp.require_json(req)

  let result = {
    use request <- result.try(decode.run(json_body, decoder()))
    use validated_user <- result.try(user.new(request.email, request.name))
    db.create_user(ctx.db, validated_user) |> result.map_error(DatabaseError)
  }

  // PRESENT - Convert to HTTP response
  case result {
    Ok(created_user) -> web.created(user_to_json(created_user))
    Error(DecodeError(errors)) -> web.error_response(web.ValidationError(errors))
    Error(DomainError(user.InvalidEmail)) ->
      web.error_response(web.BadRequest("Invalid email"))
    Error(DatabaseError(_)) ->
      web.error_response(web.InternalError("Failed to create user"))
  }
}
```

**Present step includes:**
- Success responses: `web.json_response()`, `web.created()`, `web.no_content()`
- Error responses: `web.error_response()`
- Domain to JSON conversion: `to_json()` functions

### Complete Example: Create Endpoint

```gleam
fn create_shop(req: Request, ctx: Context) -> Response {
  use json_body <- wisp.require_json(req)

  let result = {
    // 1. PARSE - Decode and validate
    use request <- result.try(
      decode.run(json_body, create_shop_decoder())
      |> result.map_error(DecodeError),
    )

    // Generate ID and timestamp
    let shop_id = id.new_shop_id(uuid.v7() |> uuid.to_string)
    let now = timestamp.system_time()

    // Domain validation
    use new_shop <- result.try(
      shop.new(
        id: shop_id,
        name: request.name,
        slug: request.slug,
        email: request.email,
        created_at: now,
      )
      |> result.map_error(fn(err) {
        case err {
          shop.EmptyName ->
            DecodeError([
              decode.DecodeError(
                expected: "Non-empty string",
                found: "Empty string",
                path: ["name"],
              ),
            ])
          shop.InvalidSlug(reason) ->
            DecodeError([
              decode.DecodeError(
                expected: "Valid slug",
                found: reason,
                path: ["slug"],
              ),
            ])
          // ... other validations
        }
      }),
    )

    // 2. PROCESS - Database operation
    db_shop.create(ctx.db, new_shop) |> result.map_error(DatabaseError)
  }

  // 3. PRESENT - HTTP response
  case result {
    Ok(created_shop) -> web.created(shop.to_json(created_shop))
    Error(DecodeError(errors)) -> web.error_response(web.ValidationError(errors))
    Error(DatabaseError(db_shop.DatabaseError(pog_error))) -> {
      // Handle specific database errors
      case pog_error {
        pog.ConstraintViolated(constraint: "shops_slug_key", ..) ->
          web.error_response(
            web.ValidationError([
              decode.DecodeError(
                expected: "Unique slug",
                found: "A shop with this slug already exists",
                path: ["slug"],
              ),
            ]),
          )
        _ -> web.error_response(web.InternalError("Failed to create shop"))
      }
    }
    Error(DatabaseError(_)) ->
      web.error_response(web.InternalError("Failed to create shop"))
  }
}
```

### List Endpoint with Query Parameters

```gleam
fn list_shops(req: Request, ctx: Context) -> Response {
  // 1. PARSE - Query parameters
  let query_params = wisp.get_query(req)

  let page =
    list.key_find(query_params, "page")
    |> result.try(int.parse)
    |> result.unwrap(1)

  let limit =
    list.key_find(query_params, "limit")
    |> result.try(int.parse)
    |> result.unwrap(50)
    |> int.clamp(1, 100)  // Validate range

  let search = list.key_find(query_params, "search") |> option.from_result
  let status = list.key_find(query_params, "status") |> option.from_result

  // 2. PROCESS - Database query
  let result = db_shop.list(ctx.db, search, status, page, limit)

  // 3. PRESENT - HTTP response
  case result {
    Ok(paginated) -> web.json_response(shop.paginated_encoder(paginated), 200)
    Error(db_error) -> web.error_response(web.InternalError("Failed to list shops"))
  }
}
```

### Update Endpoint

```gleam
fn update_shop(req: Request, ctx: Context, id_string: String) -> Response {
  use json_body <- wisp.require_json(req)

  let result = {
    // 1. PARSE - Decode payload and ID
    use update_payload <- result.try(
      decode.run(json_body, shop_update_decoder())
      |> result.map_error(DecodeError),
    )

    let shop_id = id.new_shop_id(id_string)

    // 2. PROCESS - Update operation
    db_shop.update(
      ctx.db,
      shop_id,
      update_payload.status,
      update_payload.stripe_account_id,
    )
    |> result.map_error(DatabaseError)
  }

  // 3. PRESENT - HTTP response
  case result {
    Ok(Nil) -> web.no_content()
    Error(DecodeError(errors)) -> web.error_response(web.ValidationError(errors))
    Error(DatabaseError(db_shop.ShopIdNotFound)) ->
      web.error_response(web.NotFound("Shop not found"))
    Error(DatabaseError(_)) ->
      web.error_response(web.InternalError("Failed to update shop"))
  }
}
```

### Get Endpoint

```gleam
fn get_shop(ctx: Context, id_string: String) -> Response {
  let result = {
    // 1. PARSE - Path parameter
    let shop_id = id.new_shop_id(id_string)

    // 2. PROCESS - Database query
    db_shop.find_by_id(ctx.db, shop_id)
  }

  // 3. PRESENT - HTTP response
  case result {
    Ok(shop) -> web.json_response(shop.to_json(shop), 200)
    Error(db_shop.InvalidUuid) ->
      web.error_response(web.BadRequest("Invalid UUID"))
    Error(db_shop.ShopIdNotFound) ->
      web.error_response(web.NotFound("Shop not found"))
    Error(db_shop.DatabaseError(_)) ->
      web.error_response(web.InternalError("Failed to get shop"))
  }
}
```

## Common Workflows

### Creating a New Web Project

```bash
gleam new my_web_app
cd my_web_app
gleam add wisp mist gleam_http gleam_erlang
```

See: [Writing Gleam](https://gleam.run/writing-gleam/)

### Basic Wisp Application Structure

```gleam
import wisp
import wisp/wisp_mist
import mist
import gleam/erlang/process
import gleam/http/request.{type Request}
import gleam/http/response.{type Response}

pub fn main() {
  let secret_key_base = wisp.random_string(64)
  let assert Ok(_) =
    handle_request
    |> wisp_mist.handler(secret_key_base)
    |> mist.new
    |> mist.port(8000)
    |> mist.start_http

  process.sleep_forever()
}

fn handle_request(req: Request(Connection)) -> Response(ResponseData) {
  use req <- middleware(req)

  case wisp.path_segments(req) {
    [] -> home_page(req)
    ["api", "users"] -> list_users(req)
    ["api", "users", id] -> get_user(req, id)
    _ -> wisp.not_found()
  }
}
```

For complete examples, see: [Wisp Documentation](https://hexdocs.pm/wisp/)

### Routing Patterns

Consult Wisp routing documentation:
- Path parameters: [Wisp - Routing](https://hexdocs.pm/wisp/)
- Query parameters: [Wisp - Request](https://hexdocs.pm/wisp/wisp.html#path_segments)
- Request methods: [Gleam HTTP - Methods](https://hexdocs.pm/gleam_http/)

### JSON APIs

Use `gleam/json` for JSON handling:
- Encoding: [gleam/json - Encoding](https://hexdocs.pm/gleam_json/)
- Decoding: [gleam/json - Decoding](https://hexdocs.pm/gleam_json/)

### Database Integration

Common database libraries:
- **[Pog](https://hexdocs.pm/pog/)** - PostgreSQL client
- **[SQLight](https://hexdocs.pm/sqlight/)** - SQLite
- **[Cake](https://hexdocs.pm/cake/)** - SQL query builder (PostgreSQL, SQLite, MariaDB, MySQL)
- **[Squirrel](https://hexdocs.pm/squirrel/)** - Type-safe SQL

### Static Files

See Wisp documentation for serving static files:
[Wisp - Static Assets](https://hexdocs.pm/wisp/)

### Middleware

Middleware patterns in Wisp:
[Wisp - Middleware](https://hexdocs.pm/wisp/)

## Frontend Development

### Lustre SPAs

For single-page applications:
[Lustre - Getting Started](https://hexdocs.pm/lustre/)

### Server-Side Rendering

For HTML templates:
- [Lustre - SSR](https://hexdocs.pm/lustre/)
- [Nakai](https://hexdocs.pm/nakai/)

## Deployment

### Fly.io Deployment

Complete deployment guide:
[Deploying Gleam to Fly.io](https://gleam.run/deployment/fly/)

Key steps:
1. Create Dockerfile with Gleam
2. Configure fly.toml
3. Deploy with `flyctl deploy`

### Docker Configuration

See the Fly.io guide for production-ready Dockerfile examples.

### Environment Configuration

Use `envoy` for environment variables:
[Envoy Documentation](https://hexdocs.pm/envoy/)

## Testing Web Applications

### Request Testing

```gleam
import wisp/simulate
import gleam/http

pub fn home_route_test() {
  let req = simulate.request(http.Get, "/")
  let response = handle_request(req)
  let assert 200 = response.status
}
```

See: [Wisp Testing](https://hexdocs.pm/wisp/)

### Integration Testing

Consult testing documentation:
- [Gleeunit](https://hexdocs.pm/gleeunit/)
- [Testing Practices](../../rules/testing-practices.md)

## Common Patterns

### Error Responses

```gleam
fn handle_error(error: MyError) -> Response(ResponseData) {
  case error {
    NotFound -> wisp.not_found()
    Unauthorized -> wisp.response(401)
    ValidationError(msg) -> wisp.unprocessable_content()
    _ -> wisp.internal_server_error()
  }
}
```

### Request Validation

Use Result types for validation:
```gleam
fn validate_user_input(req: Request) -> Result(ValidUser, ValidationError) {
  use body <- result.try(wisp.read_body_as_json(req))
  use email <- result.try(get_email(body))
  use name <- result.try(get_name(body))
  Ok(ValidUser(email, name))
}
```

## Security

### CORS

See Wisp documentation for CORS middleware:
[Wisp - CORS](https://hexdocs.pm/wisp/)

### Authentication

Common patterns:
- Session-based: Use Wisp's session support
- JWT: Use [gwt](https://hexdocs.pm/gwt/) library
- OAuth: Integrate with external libraries

### HTTPS

For production HTTPS, see deployment guides:
- [Fly.io Deployment](https://gleam.run/deployment/fly/)
- [Mist TLS Configuration](https://hexdocs.pm/mist/)

## Performance

### Caching

Consider:
- ETS tables (Erlang)
- External caching (Redis, Memcached)

### Monitoring

Use standard BEAM monitoring tools:
- Observer (Erlang)
- Telemetry libraries
- [Palabres - Logging](https://hexdocs.pm/palabres/)

---

**When building web applications, always consult official framework documentation for current best practices and examples.**
