# Gleam Coding Standards and Conventions

This document outlines the mandatory conventions and recommended patterns for writing Gleam code.

## Naming Conventions

### Case Conventions (MANDATORY)

Gleam enforces these naming conventions:

- **snake_case** - Variables, constants, and functions
- **PascalCase** - Types and variants

```gleam
// Good
pub type UserRole {
  Admin
  Member
  Guest
}

pub const max_retries = 3

pub fn calculate_total(items: List(Item)) -> Int {
  list.fold(items, 0, fn(acc, item) { acc + item.price })
}

// Bad
pub type user_role {  // Should be PascalCase
  admin             // Should be PascalCase
  member
}

pub const MaxRetries = 3  // Should be snake_case

pub fn CalculateTotal(items: List(Item)) -> Int {  // Should be snake_case
  // ...
}
```

### Acronyms (MANDATORY)

Treat acronyms as single words, not all caps:

```gleam
// Good
pub type Json = ...
pub fn parse_json(input: String) -> Result(Json, ParseError)
pub fn json_to_xml(json: Json) -> Xml

// Bad
pub type JSON = ...  // Will become j_s_o_n in BEAM
pub fn parse_JSON(input: String) -> Result(JSON, ParseError)
```

### Module Names (MANDATORY)

Module names are singular, not plural:

```gleam
// Good
import app/user
import app/payment/invoice

// Bad
import app/users
import app/payments/invoices
```

### Conversion Functions (MANDATORY)

Use `x_to_y` pattern for conversion functions:

```gleam
// Good
pub fn json_to_string(data: Json) -> String
pub fn string_to_int(str: String) -> Result(Int, Nil)
pub fn date_to_rfc3339(date: Date) -> String

// Bad
pub fn json_as_string(data: Json) -> String
pub fn string_into_int(str: String) -> Result(Int, Nil)
pub fn string_of_date(date: Date) -> String
```

When module name matches type name, omit type name from function:

```gleam
// In src/my_app/identifier.gleam

// Good
pub fn to_string(id: Identifier) -> String

// Bad
pub fn identifier_to_string(id: Identifier) -> String
```

Use descriptive names when appropriate:

```gleam
// Good - describes the operation
pub fn round(value: Float) -> Int
pub fn parse(input: String) -> Result(Json, ParseError)

// Less good - too generic
pub fn float_to_int(value: Float) -> Int
pub fn string_to_json(input: String) -> Result(Json, ParseError)
```

## Import Conventions

### Qualified Imports (MANDATORY)

Always use qualified syntax for functions and constants from other modules:

```gleam
// Good
import gleam/list
import gleam/string

pub fn reverse(input: String) -> String {
  input
  |> string.to_graphemes
  |> list.reverse
  |> string.concat
}

// Bad
import gleam/list.{reverse}
import gleam/string.{to_graphemes, concat}

pub fn reverse(input: String) -> String {
  input
  |> to_graphemes
  |> reverse  // Ambiguous! Which reverse?
  |> concat
}
```

**Exception**: Types and record constructors may use unqualified syntax when it improves readability:

```gleam
// OK for types
import gleam/option.{type Option, None, Some}

pub fn get_first(items: List(a)) -> Option(a) {
  case items {
    [first, ..] -> Some(first)
    [] -> None
  }
}
```

## Type Annotations

### Annotate All Public Functions (MANDATORY)

All public functions must have type annotations for arguments and return type:

```gleam
// Good
pub fn calculate_total(amounts: List(Float), tax_rate: Float) -> Float {
  let subtotal = list.fold(amounts, 0.0, float.add)
  subtotal *. { 1.0 +. tax_rate }
}

// Bad - missing annotations
pub fn calculate_total(amounts, tax_rate) {
  let subtotal = list.fold(amounts, 0.0, float.add)
  subtotal *. { 1.0 +. tax_rate }
}

// Bad - missing return annotation
pub fn calculate_total(amounts: List(Float), tax_rate: Float) {
  let subtotal = list.fold(amounts, 0.0, float.add)
  subtotal *. { 1.0 +. tax_rate }
}
```

### Annotate Private Functions (RECOMMENDED)

While not mandatory, annotating private functions improves code clarity:

```gleam
// Good
fn parse_line(line: String) -> Result(Record, ParseError) {
  // ...
}

// Acceptable but less clear
fn parse_line(line) {
  // ...
}
```

## Error Handling

### Use Result, Not Option (MANDATORY)

Use `Result` for all fallible functions. Never use `Option` for error cases:

```gleam
// Good
pub fn first(list: List(a)) -> Result(a, Nil) {
  case list {
    [item, ..] -> Ok(item)
    [] -> Error(Nil)
  }
}

// Bad - using Option
pub fn first(list: List(a)) -> Option(a) {
  case list {
    [item, ..] -> Some(item)
    [] -> None
  }
}
```

**Why Result?**
- Consistent error handling across all code
- No conversion boilerplate between Result and Option
- Error types can carry information (not just `None`)
- Matches Gleam's ecosystem conventions

### Libraries Never Panic (MANDATORY)

Libraries must never use `panic` or `let assert`. Always return `Result`:

```gleam
// Good - library code
pub fn parse_config(input: String) -> Result(Config, ParseError) {
  case parse(input) {
    Ok(config) -> Ok(config)
    Error(err) -> Error(InvalidFormat(err))
  }
}

// Bad - library code
pub fn parse_config(input: String) -> Config {
  case parse(input) {
    Ok(config) -> config
    Error(_) -> panic as "invalid config"  // Never do this in libraries!
  }
}
```

Application code may use panics at the top level when appropriate:

```gleam
// OK - application startup code
pub fn main() {
  let assert Ok(config) = load_config()
  let assert Ok(_) = start_server(config)
}
```

### Error Handling Pattern for Applications

For database and external operations in application code:

```gleam
import gleam/io

pub fn find_by_id(
  conn: pog.Connection,
  id: String,
) -> Result(Option(Record), Nil) {
  case uuid.from_string(id) {
    Error(_) -> {
      io.println_error("Invalid UUID: " <> id)
      Error(Nil)
    }
    Ok(uuid) ->
      case db.query(conn, uuid) {
        Ok(rows) -> Ok(list.first(rows) |> option.from_result)
        Error(err) -> {
          io.println_error("Database error: " <> string.inspect(err))
          Error(Nil)
        }
      }
  }
}
```

## Code Organization

### Project Structure

Standard Gleam project structure:

```
my_project/
├── gleam.toml              # Project configuration
├── manifest.toml           # Locked dependencies
├── README.md
├── src/
│   ├── my_project.gleam    # Main module
│   ├── user.gleam          # Domain modules
│   ├── user/
│   │   ├── model.gleam
│   │   └── repository.gleam
│   └── internal/           # Internal modules (not for public use)
│       └── helpers.gleam
├── test/
│   ├── my_project_test.gleam
│   └── user_test.gleam
└── build/                  # Generated (git ignore)
```

### Module Organization

Organize code within modules logically:

```gleam
// 1. Imports
import gleam/list
import gleam/result

// 2. Type definitions
pub type User {
  User(id: Int, name: String, email: String)
}

pub type UserError {
  NotFound
  InvalidEmail
  DatabaseError
}

// 3. Public functions
pub fn create_user(name: String, email: String) -> Result(User, UserError) {
  // ...
}

pub fn find_user(id: Int) -> Result(User, UserError) {
  // ...
}

// 4. Private helper functions
fn validate_email(email: String) -> Bool {
  // ...
}
```

## Documentation

### Document Public APIs

All public functions, types, and modules should have documentation:

```gleam
/// Represents a user in the system.
///
pub type User {
  User(id: Int, name: String, email: String)
}

/// Creates a new user with the given name and email.
///
/// Returns `Error(InvalidEmail)` if the email format is invalid.
///
/// ## Examples
///
/// ```gleam
/// create_user("Alice", "alice@example.com")
/// // -> Ok(User(1, "Alice", "alice@example.com"))
/// ```
///
pub fn create_user(name: String, email: String) -> Result(User, UserError) {
  // ...
}
```

### Comment Liberally (RECOMMENDED)

Use comments to explain _why_ and _what_:

```gleam
pub fn classify_file(content: String) -> FileOrigin {
  // In newer versions of squirrel this is always at the beginning,
  // but in older versions the module comment is not present, so we
  // need to check for function-level comments too.
  let likely_generated =
    string.contains(content, "> 🐿️ This module was generated")
    || string.contains(content, "> 🐿️ This function was generated")

  case likely_generated {
    True -> Generated
    False -> Manual
  }
}
```

## Common Patterns

### Builder Pattern

For complex types with many optional fields:

```gleam
pub type Config {
  Config(port: Int, host: String, debug: Bool, timeout: Int)
}

pub fn new() -> Config {
  Config(port: 8080, host: "localhost", debug: False, timeout: 5000)
}

pub fn with_port(config: Config, port: Int) -> Config {
  Config(..config, port: port)
}

pub fn with_debug(config: Config, debug: Bool) -> Config {
  Config(..config, debug: debug)
}

// Usage
let config = new()
  |> with_port(3000)
  |> with_debug(True)
```

### Result Chaining

Use `result.try` (or the `use` keyword) for sequential operations:

```gleam
pub fn process_user(id: Int) -> Result(User, Error) {
  use raw_user <- result.try(fetch_user(id))
  use validated <- result.try(validate_user(raw_user))
  use enriched <- result.try(enrich_user(validated))
  Ok(enriched)
}
```

### Make Invalid States Impossible

Design types that can't represent invalid states:

```gleam
// Bad - can be in invalid state
pub type Connection {
  Connection(url: String, connected: Bool, socket: Option(Socket))
}

// Good - invalid states impossible
pub type Connection {
  Disconnected(url: String)
  Connected(url: String, socket: Socket)
}
```

## Anti-Patterns

### Don't Fragment Modules

Keep related functionality together. Don't split into too many small modules:

```gleam
// Bad - over-fragmented
user/create.gleam
user/update.gleam
user/delete.gleam
user/find.gleam

// Good - cohesive module
user.gleam  // All user operations
```

### Don't Use Category Theory Names

Use domain-specific names instead of abstract terms:

```gleam
// Bad
pub fn fmap(f: fn(a) -> b, fa: MyType(a)) -> MyType(b)
pub fn bind(ma: MyType(a), f: fn(a) -> MyType(b)) -> MyType(b)

// Good
pub fn map(container: MyType(a), f: fn(a) -> b) -> MyType(b)
pub fn then(result: MyType(a), f: fn(a) -> MyType(b)) -> MyType(b)
```

### Don't Check Then Assert

Don't check a condition and then assert on it:

```gleam
// Bad
case list.first(items) {
  Ok(_) -> {
    let assert Ok(item) = list.first(items)  // Already checked!
    use_item(item)
  }
  Error(_) -> handle_empty()
}

// Good
case list.first(items) {
  Ok(item) -> use_item(item)
  Error(_) -> handle_empty()
}
```

## Tool Configuration

### Formatting (MANDATORY)

Always format code with `gleam format`:

```bash
gleam format

# Check formatting in CI
gleam format --check
```

Gleam's formatter is opinionated and not configurable. This ensures consistent style across all Gleam code.

### Linting

While Gleam doesn't have an official linter, the compiler provides excellent warnings:

```bash
# Enable all warnings
gleam build

# Treat warnings as errors in CI
gleam build --warnings-as-errors
```

## External Resources

- [Gleam Style Guide](https://gleam.run/writing-gleam/)
- [Gleam Conventions & Patterns](https://github.com/gleam-lang/website/blob/patterns/documentation/conventions-patterns-anti-patterns.djot)
- [Gleam Language Tour](https://tour.gleam.run/)
- [gleam.toml Reference](https://gleam.run/writing-gleam/gleam-toml/)
