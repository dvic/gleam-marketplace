---
name: gleam-javascript-interop
description: Guides Claude through integrating Gleam with JavaScript code using FFI, external functions, and NPM packages. Use when building for JavaScript target, using browser APIs, or wrapping JS libraries.
---

# Gleam JavaScript Interop Skill

This skill guides Claude Code through integrating Gleam with JavaScript code.

## Primary Sources

1. **[Gleam Externals - JavaScript](https://gleam.run/documentation/externals/)** - Official FFI guide
2. **[Gleam JavaScript Package](https://hexdocs.pm/gleam_javascript/)** - JavaScript integration utilities
3. **[Gleam Fetch](https://hexdocs.pm/gleam_fetch/)** - Fetch API bindings
4. **[Conversation](https://hexdocs.pm/conversation/)** - Request/Response API bindings
5. **[ESGleam](https://hexdocs.pm/esgleam/)** - esbuild integration

## JavaScript Target

### Building for JavaScript

```bash
gleam build --target javascript
```

Output in `build/dev/javascript/`.

### Running JavaScript Builds

```bash
node build/dev/javascript/my_project/my_module.mjs
```

## External Functions

### Basic JavaScript External

```gleam
@external(javascript, "./my_module.mjs", "myFunction")
pub fn my_function(arg: String) -> Int
```

**Important**: Module paths are relative. Function names match JavaScript exports.

See: [External Functions - JavaScript](https://gleam.run/documentation/externals/)

### Creating FFI Files

Create `my_module.mjs` in your project:

```javascript
// src/my_module.mjs or priv/my_module.mjs

export function myFunction(arg) {
  return arg.length;
}
```

## Data Type Mapping

### Gleam to JavaScript

| Gleam Type | JavaScript Type |
|------------|----------------|
| `Int` | `number` |
| `Float` | `number` |
| `String` | `string` |
| `Bool` | `boolean` |
| `List(a)` | Custom List object |
| `#(a, b)` | Array `[a, b]` |
| `Result(a, e)` | Custom Result object |
| Custom types | Custom objects |

### Constructing Gleam Types in JavaScript

Import constructors from generated Gleam code:

```javascript
// Importing from generated Gleam modules
import { Ok, Error } from "../gleam.mjs";
import { Some, None } from "../gleam_stdlib/gleam/option.mjs";

export function parseNumber(input) {
  const num = parseInt(input);
  if (isNaN(num)) {
    return new Error("Not a number");
  }
  return new Ok(num);
}

export function findFirst(arr) {
  if (arr.length === 0) {
    return new None();
  }
  return new Some(arr[0]);
}
```

### Custom Type Constructors

Format: `TypeName$Variant`

```gleam
// In Gleam
pub type User {
  Guest
  LoggedIn(id: Int, name: String)
}

@external(javascript, "./user_ffi.mjs", "createGuest")
pub fn create_guest() -> User

@external(javascript, "./user_ffi.mjs", "createLoggedIn")
pub fn create_logged_in(id: Int, name: String) -> User
```

```javascript
// In user_ffi.mjs
import { User$Guest, User$LoggedIn } from "./user.mjs";

export function createGuest() {
  return new User$Guest();
}

export function createLoggedIn(id, name) {
  return new User$LoggedIn(id, name);
}
```

## Working with Lists

Lists in Gleam JavaScript are custom objects, not arrays.

### List Helpers

```javascript
import { List, Empty } from "../gleam.mjs";

// Check if list is empty
List.isEmpty(myList)

// For non-empty lists
List.first(myList)  // Get first element
List.rest(myList)   // Get tail

// Convert array to List
function arrayToList(arr) {
  let list = new Empty();
  for (let i = arr.length - 1; i >= 0; i--) {
    list = new List(arr[i], list);
  }
  return list;
}
```

See: [External Functions - Lists](https://gleam.run/documentation/externals/)

## JavaScript Runtime APIs

### Promises

Use `gleam_javascript` for Promise integration:

```gleam
import gleam/javascript/promise.{type Promise}

@external(javascript, "./async_ffi.mjs", "fetchData")
pub fn fetch_data(url: String) -> Promise(Result(String, Error))

pub fn get_user_data() {
  fetch_data("https://api.example.com/user")
  |> promise.await(fn(result) {
    case result {
      Ok(data) -> process_data(data)
      Error(err) -> handle_error(err)
    }
  })
}
```

See: [gleam_javascript - Promise](https://hexdocs.pm/gleam_javascript/)

### Fetch API

Use `gleam_fetch` for HTTP requests:

```gleam
import gleam/fetch
import gleam/javascript/promise

pub fn get_user(id: String) {
  let url = "https://api.example.com/users/" <> id
  fetch.send(fetch.to(url))
  |> promise.try_await(fetch.read_text_body)
}
```

See: [gleam_fetch Documentation](https://hexdocs.pm/gleam_fetch/)

### Console API

```gleam
import gleam/io

pub fn main() {
  io.println("Hello from Gleam!")  // Uses console.log on JavaScript
}
```

### setTimeout / setInterval

```gleam
@external(javascript, "./timer_ffi.mjs", "setTimeout")
pub fn set_timeout(callback: fn() -> Nil, ms: Int) -> Nil

@external(javascript, "./timer_ffi.mjs", "setInterval")
pub fn set_interval(callback: fn() -> Nil, ms: Int) -> Nil
```

```javascript
// timer_ffi.mjs
export function setTimeout(callback, ms) {
  globalThis.setTimeout(callback, ms);
}

export function setInterval(callback, ms) {
  globalThis.setInterval(callback, ms);
}
```

## NPM Integration

### Installing NPM Packages

```bash
npm install <package-name>
```

### Using NPM Packages

Create FFI wrapper:

```javascript
// src/moment_ffi.mjs
import moment from "moment";

export function now() {
  return moment().format();
}

export function parse(dateString) {
  const m = moment(dateString);
  if (!m.isValid()) {
    return new Error("Invalid date");
  }
  return new Ok(m.format());
}
```

```gleam
// src/moment.gleam
@external(javascript, "./moment_ffi.mjs", "now")
pub fn now() -> String

@external(javascript, "./moment_ffi.mjs", "parse")
pub fn parse(date_string: String) -> Result(String, Nil)
```

## Build Tools

### esbuild Integration

Use `esgleam` for bundling:

```bash
gleam add --dev esgleam
```

See: [ESGleam Documentation](https://hexdocs.pm/esgleam/)

### Bundling

Create build script:

```javascript
// build.mjs
import * as esbuild from "esbuild";

await esbuild.build({
  entryPoints: ["build/dev/javascript/my_project/my_module.mjs"],
  bundle: true,
  outfile: "dist/bundle.js",
  format: "esm",
});
```

## Browser APIs

### DOM Manipulation

```gleam
@external(javascript, "./dom_ffi.mjs", "getElementById")
pub fn get_element_by_id(id: String) -> Result(Element, Nil)

@external(javascript, "./dom_ffi.mjs", "setInnerHTML")
pub fn set_inner_html(element: Element, html: String) -> Nil
```

```javascript
// dom_ffi.mjs
import { Ok, Error } from "../gleam.mjs";

export function getElementById(id) {
  const element = document.getElementById(id);
  if (element === null) {
    return new Error(undefined);
  }
  return new Ok(element);
}

export function setInnerHTML(element, html) {
  element.innerHTML = html;
}
```

### Event Listeners

```gleam
pub type EventListener

@external(javascript, "./events_ffi.mjs", "addEventListener")
pub fn add_event_listener(
  element: Element,
  event: String,
  handler: fn(Event) -> Nil,
) -> Nil
```

```javascript
// events_ffi.mjs
export function addEventListener(element, event, handler) {
  element.addEventListener(event, handler);
}
```

## Frontend Frameworks

### Lustre (Gleam's Elm-inspired Framework)

For full frontend applications:

See: [Lustre Documentation](https://hexdocs.pm/lustre/)

### React Bindings

For React integration:

See: [Redraw](https://hexdocs.pm/redraw/) - React bindings for Gleam

## Testing JavaScript Target

```bash
gleam test --target javascript
```

Platform-specific tests:

```gleam
@target(javascript)
pub fn javascript_specific_test() {
  let assert Ok(result) = javascript_only_function()
  let assert expected = result
}
```

## Debugging

### Console Logging

```gleam
import gleam/io
import gleam/string

pub fn debug(value: anything) -> anything {
  io.debug(value)  // Outputs to console with inspect format
  value
}
```

### Source Maps

Enable source maps in your build tool for easier debugging.

## Common Patterns

### Callback Conversion

```gleam
// Gleam function
pub fn process_async(callback: fn(Result(Data, Error)) -> Nil) -> Nil {
  // Implementation
}

// JavaScript FFI
@external(javascript, "./async_ffi.mjs", "processAsync")
pub fn process_async_js(callback: fn(Result(Data, Error)) -> Nil) -> Nil
```

```javascript
// async_ffi.mjs
export function processAsync(gleamCallback) {
  // Convert JavaScript promise to Gleam callback
  fetch("https://api.example.com/data")
    .then(response => response.json())
    .then(data => gleamCallback(new Ok(data)))
    .catch(error => gleamCallback(new Error(error.message)));
}
```

### Error Handling

Always convert JavaScript errors to Gleam Results:

```javascript
import { Ok, Error } from "../gleam.mjs";

export function riskyOperation(input) {
  try {
    const result = doSomethingRisky(input);
    return new Ok(result);
  } catch (error) {
    return new Error(error.message);
  }
}
```

## Limitations

### JavaScript Target Limitations

- No UTF codepoint pattern matching in bit arrays
- No `native` endianness option
- Different runtime behavior from Erlang target

See: [External Functions - Platform Limitations](https://gleam.run/documentation/externals/)

### Performance Considerations

- JavaScript lacks tail call optimization
- Different garbage collection characteristics
- Consider target when designing recursive algorithms

---

**Remember**: Use JavaScript interop sparingly. Prefer Gleam implementations when possible for type safety and cross-platform compatibility.

See: [External Functions](../rules/external-functions.md)
