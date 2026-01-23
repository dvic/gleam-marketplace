# /gleam-web-endpoint - Create Web API Endpoint

Add a new web endpoint to a Wisp application.

## Usage

```
/gleam-web-endpoint <path> <method>
```

## Workflow

### Handler Pattern: Parse → Process → Present

**CRITICAL: All handlers must follow this three-part structure**

1. **Define route in router**
   ```gleam
   fn handle_request(req: Request(Connection)) -> Response(ResponseData) {
     case wisp.path_segments(req), req.method {
       ["api", "users"], Get -> list_users(req, ctx)
       ["api", "users", id], Get -> get_user(req, ctx, id)
       ["api", "users"], Post -> create_user(req, ctx)
       _ -> wisp.not_found()
     }
   }
   ```

2. **Implement handler with Parse → Process → Present**

   **GET Example:**
   ```gleam
   fn get_user(req: Request, ctx: Context, id: String) -> Response {
     let result = {
       // 1. PARSE - Path parameter
       let user_id = id.new_user_id(id)

       // 2. PROCESS - Database query
       db.find_user_by_id(ctx.db, user_id)
     }

     // 3. PRESENT - HTTP response
     case result {
       Ok(user) -> web.json_response(user_to_json(user), 200)
       Error(db.InvalidUuid) -> web.error_response(web.BadRequest("Invalid ID"))
       Error(db.UserNotFound) -> web.error_response(web.NotFound("User not found"))
       Error(db.DatabaseError(_)) ->
         web.error_response(web.InternalError("Failed to get user"))
     }
   }
   ```

   **POST Example:**
   ```gleam
   fn create_user(req: Request, ctx: Context) -> Response {
     use json_body <- wisp.require_json(req)

     let result = {
       // 1. PARSE - Decode and validate
       use request <- result.try(
         decode.run(json_body, user_request_decoder())
         |> result.map_error(DecodeError),
       )

       // Domain validation
       use validated_user <- result.try(
         user.new(request.email, request.name)
         |> result.map_error(DomainError),
       )

       // 2. PROCESS - Database operation
       db.create_user(ctx.db, validated_user)
       |> result.map_error(DatabaseError)
     }

     // 3. PRESENT - HTTP response
     case result {
       Ok(created_user) -> web.created(user_to_json(created_user))
       Error(DecodeError(errors)) ->
         web.error_response(web.ValidationError(errors))
       Error(DomainError(user.InvalidEmail)) ->
         web.error_response(web.BadRequest("Invalid email address"))
       Error(DatabaseError(_)) ->
         web.error_response(web.InternalError("Failed to create user"))
     }
   }
   ```

   **LIST Example (with query params):**
   ```gleam
   fn list_users(req: Request, ctx: Context) -> Response {
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
       |> int.clamp(1, 100)

     let search = list.key_find(query_params, "search") |> option.from_result

     // 2. PROCESS - Database query
     let result = db.list_users(ctx.db, search, page, limit)

     // 3. PRESENT - HTTP response
     case result {
       Ok(paginated) -> web.json_response(users_to_json(paginated), 200)
       Error(_) -> web.error_response(web.InternalError("Failed to list users"))
     }
   }
   ```

   **UPDATE Example:**
   ```gleam
   fn update_user(req: Request, ctx: Context, id: String) -> Response {
     use json_body <- wisp.require_json(req)

     let result = {
       // 1. PARSE
       use update <- result.try(
         decode.run(json_body, user_update_decoder())
         |> result.map_error(DecodeError),
       )

       let user_id = id.new_user_id(id)

       // 2. PROCESS
       db.update_user(ctx.db, user_id, update.name, update.email)
       |> result.map_error(DatabaseError)
     }

     // 3. PRESENT
     case result {
       Ok(Nil) -> web.no_content()
       Error(DecodeError(errors)) ->
         web.error_response(web.ValidationError(errors))
       Error(DatabaseError(db.UserNotFound)) ->
         web.error_response(web.NotFound("User not found"))
       Error(DatabaseError(_)) ->
         web.error_response(web.InternalError("Failed to update"))
     }
   }
   ```

4. **Add tests**
   ```gleam
   import wisp/simulate
   import gleam/http

   pub fn get_user_endpoint_test() {
     let req = simulate.request(http.Get, "/api/users/123")
     let response = handle_request(req)

     let assert 200 = response.status
     let assert Ok(json) = json.decode(response.body)
     // Assert on json content
   }
   ```

5. **Test endpoint**
   ```bash
   gleam test
   gleam run
   # Test with curl or API client
   ```

## Response Helpers

### Success Responses
```gleam
wisp.ok()                                    // 200 OK
wisp.created()                               // 201 Created
wisp.json_response(json, 200)               // JSON response
wisp.html_response(html, 200)               // HTML response
```

### Error Responses
```gleam
wisp.not_found()                            // 404 Not Found
wisp.method_not_allowed([Get, Post])       // 405 Method Not Allowed
wisp.unprocessable_content()               // 422 Unprocessable Content
wisp.internal_server_error()               // 500 Internal Server Error
wisp.response(status_code)                 // Custom status
```

## Request Handling

### Query Parameters
```gleam
let query = wisp.get_query(req)
case dict.get(query, "page") {
  Ok(page_str) -> parse_page(page_str)
  Error(_) -> 1  // Default
}
```

### Request Body
```gleam
use json <- wisp.require_json(req)
// json is now available
```

### Headers
```gleam
case wisp.get_header(req, "authorization") {
  Ok(token) -> verify_token(token)
  Error(_) -> wisp.response(401)
}
```

## Middleware

```gleam
fn with_auth(
  req: Request(Connection),
  handler: fn(Request(Connection), User) -> Response(ResponseData),
) -> Response(ResponseData) {
  case authenticate(req) {
    Ok(user) -> handler(req, user)
    Error(_) -> wisp.response(401)
  }
}

// Usage
fn protected_route(req: Request(Connection)) -> Response(ResponseData) {
  use user <- with_auth(req)
  // Handle authenticated request
}
```

## References

- [Wisp Documentation](https://hexdocs.pm/wisp/)
- [Gleam HTTP](https://hexdocs.pm/gleam_http/)
- [Web Development Skill](../skills/gleam-web-development/SKILL.md)

## Best Practices

- Validate all inputs
- Use appropriate status codes
- Handle errors gracefully
- Add logging for debugging
- Test all endpoints
- Use middleware for cross-cutting concerns
