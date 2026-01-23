# Testing Practices for Gleam

This document outlines testing best practices and conventions for Gleam projects.

## Core Testing Principles

### Use `let assert`, NOT the `should` module

The `gleeunit/should` module is **deprecated** and must not be used. Always use `let assert` with pattern matching for test assertions.

**Why `let assert`?**
- Built-in - No external dependencies required
- Clear errors - Shows exactly what value was received vs what was expected
- Pattern matching - Leverages Gleam's type system
- Type safe - Compiler verified, catching errors at compile time
- Readable - Intent is immediately clear from the pattern
- Future-proof - Not dependent on deprecated libraries

### Common Test Assertion Patterns

```gleam
// Asserting success
let assert Ok(value) = result

// Asserting errors
let assert Error(Nil) = result
let assert Error(specific_error) = result

// Asserting specific values
let assert "expected string" = actual
let assert 42 = number
let assert True = boolean

// Asserting optional values
let assert None = empty_optional
let assert Some(value) = filled_optional

// Asserting equality (when pattern matching isn't enough)
let assert True = actual == expected
let assert True = list.length(items) == 5

// Or use compact assert syntax for boolean expressions
assert actual == expected
assert list.length(items) == 5
assert result.is_valid
assert !result.is_empty

// Asserting list contents
let assert [first, second, third] = my_list
let assert [] = empty_list
let assert [item] = single_item_list
```

**Note:** The `assert <boolean expression>` syntax is more compact than `let assert True = <boolean expression>` for checking boolean conditions.

## Test Organization

### Test Function Naming

Test functions should be descriptive and follow this pattern:

```gleam
// Good - describes what is being tested
pub fn parse_valid_json_test() {
  let assert Ok(result) = json.parse("{\"key\": \"value\"}")
  let assert True = dict.size(result) == 1
}

pub fn parse_invalid_json_returns_error_test() {
  let assert Error(_) = json.parse("not json")
}

// Bad - vague names
pub fn test1() { ... }
pub fn it_works() { ... }
```

### Test File Structure

Place tests in `test/` directory, mirroring the structure of `src/`:

```
project/
├── src/
│   ├── parser.gleam
│   └── validator.gleam
└── test/
    ├── parser_test.gleam
    └── validator_test.gleam
```

### Module Test Organization

```gleam
import gleeunit
import my_app/module

pub fn main() {
  gleeunit.main()
}

// Group related tests together
pub fn parse_empty_string_test() { ... }
pub fn parse_single_item_test() { ... }
pub fn parse_multiple_items_test() { ... }

pub fn validate_email_valid_test() { ... }
pub fn validate_email_invalid_test() { ... }
```

## Testing Different Scenarios

### Testing Results

```gleam
pub fn successful_operation_test() {
  let result = perform_operation()
  let assert Ok(value) = result
  let assert "expected" = value
}

pub fn failing_operation_test() {
  let result = perform_failing_operation()
  let assert Error(Nil) = result
}

pub fn specific_error_test() {
  let result = validate_input("invalid")
  let assert Error(InvalidInput("invalid")) = result
}
```

### Testing Options

```gleam
pub fn found_item_test() {
  let items = [1, 2, 3]
  let result = list.first(items)
  let assert Ok(1) = result
}

pub fn not_found_test() {
  let items = []
  let result = list.first(items)
  let assert Error(Nil) = result
}
```

### Testing Lists

```gleam
pub fn empty_list_test() {
  let result = process([])
  let assert [] = result
}

pub fn single_item_test() {
  let result = process([1])
  let assert [2] = result
}

pub fn multiple_items_test() {
  let result = process([1, 2, 3])
  let assert [2, 4, 6] = result
}
```

### Testing Custom Types

```gleam
pub type Result {
  Success(value: String)
  Failure(reason: String)
}

pub fn success_case_test() {
  let result = perform_action()
  let assert Success(value) = result
  let assert "expected" = value
}

pub fn failure_case_test() {
  let result = perform_failing_action()
  let assert Failure(reason) = result

  // For simple checks, use assert
  assert string.contains(reason, "error")

  // For complex error messages, use birdie
  // reason |> birdie.snap(title: "failure reason")
}
```

## Test Execution

### Running Tests

```bash
# Run all tests
gleam test

# Run with specific target
gleam test --target erlang
gleam test --target javascript

# Run with verbose output
gleam test --verbose
```

### Test Timeouts

For long-running tests on Erlang target:

```gleam
import gleam/erlang/process

pub fn long_running_test() {
  // Set timeout to 10 seconds
  process.trap_exits(True)

  let result = perform_long_operation()
  let assert Ok(_) = result
}
```

See: [Gleam Test Timeouts](https://gearsco.de/blog/gleam-test-timeouts/)

## Integration Testing

### Testing with External Dependencies

```gleam
// Use dependency injection for testability
pub type Database {
  Database(connection: Connection)
}

pub type Connection {
  Real(conn: pog.Connection)
  Mock(data: List(Record))
}

// Production code
pub fn create_with_real_db(conn: pog.Connection) -> Database {
  Database(Real(conn))
}

// Test code
pub fn query_returns_results_test() {
  let mock_data = [Record(id: 1, name: "test")]
  let db = Database(Mock(mock_data))

  let result = query(db, "SELECT * FROM records")
  let assert Ok(records) = result
  let assert [Record(1, "test")] = records
}
```

### Testing HTTP Handlers

```gleam
import wisp
import wisp/testing

pub fn get_users_returns_list_test() {
  let request = testing.get("/api/users", [])
  let response = handle_request(request)

  let assert 200 = response.status
  let assert Ok(body) = wisp.read_body(response)

  // For complex output, use birdie for snapshot testing
  body
  |> birdie.snap(title: "users list response")
}

pub fn post_user_creates_record_test() {
  let body = "{\"name\": \"Alice\"}"
  let request = testing.post("/api/users", [], body)
  let response = handle_request(request)

  let assert 201 = response.status
}
```

### Snapshot Testing with Birdie

For complex outputs like JSON, HTML, or formatted text that are difficult to assert by hand, use [Birdie](https://hexdocs.pm/birdie/) for snapshot testing:

```gleam
import birdie

pub fn complex_json_output_test() {
  let result = generate_complex_json()

  // Birdie captures the output and compares it to a saved snapshot
  result
  |> birdie.snap(title: "complex json structure")
}

pub fn html_rendering_test() {
  let html = render_page(user: User(name: "Alice", role: Admin))

  html
  |> birdie.snap(title: "admin user page")
}

pub fn formatted_report_test() {
  let report = generate_report(data: sample_data())

  report
  |> birdie.snap(title: "monthly report format")
}
```

**When to use Birdie:**
- Complex JSON/XML responses
- HTML rendering
- Formatted text output (reports, logs)
- Large data structures
- Any output that's tedious to manually assert

**When NOT to use Birdie:**
- Simple equality checks (use `assert` or `let assert`)
- Testing specific behavior (use targeted assertions)
- Validating error messages (assert the error type/pattern)

**Updating snapshots:**
```bash
# Review and approve snapshot changes
gleam test -- --accept
```

## Best Practices

### DO: Test edge cases

```gleam
pub fn empty_input_test() { ... }
pub fn single_item_test() { ... }
pub fn large_input_test() { ... }
pub fn invalid_input_test() { ... }
```

### DO: Test error paths

```gleam
pub fn network_failure_test() { ... }
pub fn invalid_data_test() { ... }
pub fn timeout_test() { ... }
```

### DO: Use descriptive names

```gleam
pub fn parse_valid_email_succeeds_test() { ... }
pub fn parse_invalid_email_returns_error_test() { ... }
```

### DO: Use birdie for complex outputs

When output is too complex to assert by hand (JSON, HTML, formatted text), use birdie:

```gleam
// Good - snapshot testing for complex output
pub fn api_response_test() {
  let response = generate_response()
  response |> birdie.snap(title: "api response")
}

// Bad - manually checking complex structures
pub fn api_response_test() {
  let response = generate_response()
  assert string.contains(response, "field1")
  assert string.contains(response, "field2")
  assert string.contains(response, "field3")
  // This gets tedious and fragile!
}
```

### DON'T: Use deprecated should module

```gleam
// Bad - deprecated
result |> should.be_ok
result |> should.equal(expected)

// Good - use let assert
let assert Ok(_) = result
let assert True = result == expected
```

### DON'T: Test implementation details

Test behavior, not internal implementation.

### DON'T: Write tests that depend on each other

Each test should be independent and able to run in any order.

## CI/CD Integration

### GitHub Actions Example

```yaml
name: Test

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: erlef/setup-beam@v1
        with:
          otp-version: "27.0"
          gleam-version: "1.10.0"
      - run: gleam test
```

## External Resources

- [Gleeunit Documentation](https://hexdocs.pm/gleeunit/)
- [Birdie - Snapshot Testing](https://hexdocs.pm/birdie/)
- [Gleam Test Timeouts](https://gearsco.de/blog/gleam-test-timeouts/)
- [Gleam Language Tour - Testing](https://tour.gleam.run/)
