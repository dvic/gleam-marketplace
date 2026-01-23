# External Functions and FFI

This document covers using Foreign Function Interface (FFI) to call JavaScript and Erlang code from Gleam.

Based on official Gleam documentation: https://gleam.run/documentation/externals

## Core Principles

### Always Prefer Gleam Solutions (MANDATORY)

**Use externals only when there is no suitable alternative.**

Externals should be used for:
- **Runtime APIs** - Accessing VM features (memory stats, process info)
- **Existing code** - When Gleam implementation is impractical
- **No pure-Gleam alternative** - Functionality not yet available

### When NOT to Use FFI

Avoid FFI when:
- **Pure Gleam exists** - Prefer type-safe Gleam implementations
- **Simple operations** - Don't use FFI for basic logic
- **Learning Gleam** - Start with pure Gleam first
- **Cross-platform needed** - FFI ties you to a specific target

## @external Attribute Syntax

The `@external` attribute marks functions implemented in other languages.

**Syntax**: `@external(target, module, function_name)`

- **target**: `erlang` or `javascript`
- **module**: Where the function is exported from
- **function_name**: The external function's identifier

**Type annotations are mandatory** for external functions.

### Erlang External Example

```gleam
@external(erlang, "lists", "reverse")
pub fn reverse_list(list: List(element)) -> List(element)
```

This calls the Erlang `lists:reverse/1` function.

### JavaScript External Example

```gleam
@external(javascript, "./project_ffi.mjs", "reverse_list")
pub fn reverse_list(list: List(element)) -> List(element)
```

The JavaScript implementation in `project_ffi.mjs`:

```javascript
export function reverse_list(list) {
  return list.toReversed();
}
```

**Important**: JavaScript modules must use `.mjs` extension for ES module compatibility.

## Multi-Target Support

Functions can have implementations for both targets:

```gleam
@external(erlang, "lists", "reverse")
@external(javascript, "./project_ffi.mjs", "reverse_list")
pub fn reverse_list(list: List(element)) -> List(element)
```

The appropriate implementation is used based on the compilation target.

## Gleam Fallbacks

A function can combine external and Gleam implementations:

```gleam
@external(erlang, "lists", "reverse")
pub fn reverse_list(list: List(element)) -> List(element) {
  // Gleam implementation used when Erlang target unavailable
  do_reverse(list, [])
}

fn do_reverse(remaining: List(a), reversed: List(a)) -> List(a) {
  case remaining {
    [] -> reversed
    [x, ..xs] -> do_reverse(xs, [x, ..reversed])
  }
}
```

The external version is used when available; otherwise the Gleam implementation executes. This enables platform-specific optimization while maintaining portability.

## Type Safety with FFI

### The Compiler Cannot Verify External Types

From the official documentation:

> The Gleam compiler will ensure that all uses of the function will be correct for the annotated types, but **it cannot verify that the function implemented in the other language returns the specified types**.

**You are responsible for ensuring the external function matches the types you declare.**

### Write More Tests (MANDATORY)

Because externals bypass compiler analysis:

- **Write more unit tests than usual** when using external functions
- Test all edge cases thoroughly
- Verify type conversions are correct
- Test error handling paths

### Always Decode Dynamic Values

When receiving data from external code, decode it properly:

```gleam
import gleam/dynamic.{type Dynamic}

@external(javascript, "./api.mjs", "fetch_user")
fn fetch_user_ffi(id: Int) -> Dynamic

pub fn fetch_user(id: Int) -> Result(User, String) {
  let decoder = dynamic.decode2(
    User,
    dynamic.field("name", dynamic.string),
    dynamic.field("email", dynamic.string),
  )

  fetch_user_ffi(id)
  |> decoder
  |> result.map_error(fn(_) { "Failed to decode user" })
}
```

### Never Use `dynamic.unsafe_coerce` (FORBIDDEN)

```gleam
// FORBIDDEN - unsafe and can crash!
@external(javascript, "./api.mjs", "get_user")
fn get_user_ffi(id: Int) -> Dynamic

pub fn get_user(id: Int) -> User {
  get_user_ffi(id)
  |> dynamic.unsafe_coerce  // This can crash at runtime!
}

// Good - safe decoding
pub fn get_user(id: Int) -> Result(User, String) {
  get_user_ffi(id)
  |> decode_user
}
```

## JavaScript FFI Patterns

### Module Paths

JavaScript modules are specified as file paths relative to the Gleam source file:

```gleam
@external(javascript, "./ffi.mjs", "my_function")
@external(javascript, "../utils/helper.mjs", "helper")
@external(javascript, "../../lib/api.mjs", "fetch_data")
```

### NPM Package Imports (Node.js)

```gleam
// Using lodash from npm
@external(javascript, "lodash", "debounce")
pub fn debounce(fn: fn() -> Nil, wait: Int) -> fn() -> Nil
```

Add to `package.json`:
```json
{
  "dependencies": {
    "lodash": "^4.17.21"
  }
}
```

### Working with Promises

```gleam
import gleam/javascript/promise.{type Promise}

@external(javascript, "./api.mjs", "fetch_data")
pub fn fetch_data(url: String) -> Promise(Dynamic)

pub fn get_data(url: String) -> Promise(Result(Data, String)) {
  fetch_data(url)
  |> promise.map(decode_data)
}
```

```javascript
// api.mjs
export async function fetch_data(url) {
  const response = await fetch(url);
  return response.json();
}
```

### JavaScript API for Gleam Data (v1.13+)

Gleam v1.13+ provides construction functions for data types in JavaScript:

**Lists**:
```javascript
import { List$Empty, List$NonEmpty } from "./gleam.mjs";

// Empty list
const empty = List$Empty();

// List [1, 2, 3]
const list = List$NonEmpty(1, List$NonEmpty(2, List$NonEmpty(3, List$Empty())));
```

**Results**:
```javascript
import { Result$Ok, Result$Error } from "./gleam.mjs";

const success = Result$Ok(42);
const failure = Result$Error("not found");
```

**Custom Types**:
```javascript
// For: pub type User { User(name: String, age: Int) }
import { User$User } from "./app.mjs";

const user = User$User("Alice", 30);

// Accessing fields
const name = user["User$User$0"]; // "Alice"
const age = user["User$User$1"];   // 30
```

Pattern: `TypeName$VariantName(fields...)` for construction
Pattern: `TypeName$VariantName$index` for field access

## Erlang FFI Patterns

### Calling Erlang Functions

```gleam
@external(erlang, "crypto", "hash")
fn hash_ffi(algorithm: Dynamic, data: BitArray) -> BitArray

import gleam/dynamic

pub fn sha256(data: String) -> BitArray {
  data
  |> bit_array.from_string
  |> hash_ffi(dynamic.from("sha256"), _)
}
```

### Calling Elixir Code

Elixir modules compile to Erlang modules with `Elixir.` prefix:

```gleam
// Calling Elixir.MyApp.Math.add/2
@external(erlang, "Elixir.MyApp.Math", "add")
pub fn add(a: Int, b: Int) -> Int
```

**Important**: The target is still `erlang`, not `elixir`.

### Working with OTP Libraries

```gleam
@external(erlang, "httpc", "request")
fn http_request_ffi(url: String) -> Dynamic

pub fn http_get(url: String) -> Result(Response, String) {
  http_request_ffi(url)
  |> decode_response
  |> result.map_error(fn(_) { "HTTP request failed" })
}
```

## Data Type Mappings

### Gleam to Erlang

| Gleam | Erlang | Notes |
|-------|--------|-------|
| String | UTF8 binary | `<<"hello">>` |
| Nil | `nil` atom | |
| Bool | `true`/`false` atoms | |
| Int | Integer | |
| Float | Float | |
| List | Proper list | `[1, 2, 3]` |
| Tuple | Tuple | `{1, 2}` |
| Custom types | Tagged tuples/atoms | |
| BitArray | Binary | |

**Custom type variants**: `PascalCase` variants convert to `snake_case` atoms in Erlang.

```gleam
pub type Status {
  Loading
  Success
  Error
}
```

Maps to Erlang atoms: `loading`, `success`, `error`

### Gleam to JavaScript

| Gleam | JavaScript | Notes |
|-------|-----------|-------|
| String | String | |
| Nil | `undefined` | |
| Bool | Boolean | |
| Int | Number | |
| Float | Number | |
| List | Object | Use helper functions |
| Tuple | Array | `[1, 2]` |
| BitArray | Uint8Array | |
| Custom types | Object | Tagged with variant |

## Target-Specific Code

### Conditional Compilation

Only one version is compiled per target:

```gleam
@target(javascript)
pub fn get_platform() -> String {
  "JavaScript"
}

@target(erlang)
pub fn get_platform() -> String {
  "Erlang"
}
```

### Target-Specific Types

```gleam
@target(javascript)
pub type FileHandle = Dynamic

@target(erlang)
pub type FileHandle = #(Atom, Int)

// Different implementations for different targets
@target(javascript)
pub fn open_file(path: String) -> Result(FileHandle, String) {
  // JavaScript implementation
}

@target(erlang)
pub fn open_file(path: String) -> Result(FileHandle, String) {
  // Erlang implementation
}
```

## External Types

External types represent data from other languages that cannot be constructed directly in Gleam:

```gleam
pub type Connection

@external(erlang, "pgsql", "connect")
pub fn connect(config: String) -> Result(Connection, String)

@external(erlang, "pgsql", "query")
pub fn query(conn: Connection, sql: String) -> Result(Dynamic, String)
```

You must provide external functions to create and manipulate external types.

## Best Practices

### DO: Wrap FFI in Safe APIs (MANDATORY)

Always wrap external functions with proper error handling and type decoding:

```gleam
// Good - safe wrapper
@external(javascript, "./api.mjs", "dangerous_call")
fn dangerous_call_ffi(input: String) -> Dynamic

pub fn safe_call(input: String) -> Result(Output, String) {
  dangerous_call_ffi(input)
  |> decode_output
  |> result.map_error(error_to_string)
}
```

### DO: Document Platform Requirements (MANDATORY)

```gleam
/// Hashes data using SHA-256.
///
/// **Platform**: Erlang only
/// **Requires**: Erlang crypto module
///
@external(erlang, "crypto", "hash")
pub fn sha256(data: String) -> BitArray {
  // ...
}
```

### DO: Test Extensively (MANDATORY)

Write comprehensive tests for all external functions:

```gleam
pub fn external_function_test() {
  let result = external_function("test input")
  let assert Ok(output) = result
  assert output == expected_output
}

pub fn external_error_handling_test() {
  let result = external_function("invalid")
  let assert Error(_) = result
}
```

### DO: Prefer Multi-Target When Possible

```gleam
@external(erlang, "crypto", "strong_rand_bytes")
@external(javascript, "node:crypto", "randomBytes")
pub fn random_bytes(size: Int) -> BitArray
```

### DON'T: Skip Error Handling (FORBIDDEN)

```gleam
// Bad - no error handling
@external(javascript, "./api.mjs", "fetch_data")
pub fn fetch_data() -> Dynamic

// Good - proper error handling
@external(javascript, "./api.mjs", "fetch_data")
fn fetch_data_ffi() -> Dynamic

pub fn fetch_data() -> Result(Data, FetchError) {
  fetch_data_ffi()
  |> decode_data
}
```

### DON'T: Use FFI for Simple Operations (ANTI-PATTERN)

```gleam
// Bad - unnecessary FFI
@external(javascript, "./math.mjs", "add")
pub fn add(a: Int, b: Int) -> Int

// Good - pure Gleam
pub fn add(a: Int, b: Int) -> Int {
  a + b
}
```

## Limitations and Considerations

From the official documentation, external functions have these limitations:

1. **No compiler type checking** - External code is not verified
2. **No language server assistance** - IDE features unavailable for externals
3. **Cross-platform compilation** - Using externals for one target prevents compilation to other targets
4. **Runtime errors** - Type mismatches only discovered at runtime
5. **Testing required** - Must write comprehensive tests

## Package Development Considerations

### Libraries Should Minimize FFI

Libraries that use FFI:
- Are tied to specific targets
- Require users to have dependencies installed
- Are harder to test and maintain
- Cannot be compiled to other targets

Prefer pure Gleam implementations for maximum portability.

### Document FFI Dependencies (MANDATORY)

If your package uses FFI, document:
- Which target(s) it supports
- What external dependencies are needed
- How to install those dependencies

```gleam
/// # Platform Support
///
/// This package requires:
/// - **JavaScript**: Node.js 18+ with `axios` npm package
/// - **Erlang**: OTP 27+
///
pub fn http_get(url: String) -> Result(Response, Error) {
  // ...
}
```

### Provide Pure Gleam Fallbacks When Possible

```gleam
@external(erlang, "crypto", "strong_rand_bytes")
pub fn random_bytes(size: Int) -> BitArray {
  // Pure Gleam fallback (less cryptographically secure)
  do_random_bytes(size)
}
```

## External Resources

- [Gleam Externals Documentation](https://gleam.run/documentation/externals)
- [Dynamic Decoding](https://hexdocs.pm/gleam_stdlib/gleam/dynamic.html)
- [Gleam JavaScript Documentation](https://gleam.run/writing-gleam/javascript/)
