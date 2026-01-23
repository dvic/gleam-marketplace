---
name: gleam-erlang-interop
description: Guides Claude through integrating Gleam with Erlang and Elixir code using FFI, external functions, and BEAM libraries. Use when building for Erlang target, calling Erlang libraries, or integrating with existing BEAM systems.
---

# Gleam Erlang Interop Skill

This skill guides Claude Code through integrating Gleam with Erlang and Elixir code.

## Primary Sources

1. **[Gleam Externals - Erlang](https://gleam.run/documentation/externals/)** - Official FFI guide
2. **[Gleam Erlang Package](https://hexdocs.pm/gleam_erlang/)** - Erlang integration utilities
3. **[Gleam OTP](https://hexdocs.pm/gleam_otp/)** - OTP framework integration
4. **[Glixir](https://hexdocs.pm/glixir/)** - Elixir OTP interop
5. **[Gleam FAQ - Elixir Integration](https://gleam.run/frequently-asked-questions/)** - Using Elixir code

## Erlang Target

### Building for Erlang

```bash
gleam build  # Defaults to Erlang target
gleam build --target erlang
```

Output in `build/dev/erlang/`.

### Running Erlang Builds

```bash
gleam run  # Run your main function
erl -pa build/dev/erlang/*/ebin  # Start Erlang shell with your code
```

## External Functions

### Basic Erlang External

```gleam
@external(erlang, "lists", "reverse")
pub fn reverse_list(list: List(element)) -> List(element)
```

Format: `@external(erlang, "module_name", "function_name")`

See: [External Functions - Erlang](https://gleam.run/documentation/externals/)

### Erlang Module Naming

Erlang modules use lowercase with underscores:
- `lists` (Erlang standard library)
- `crypto` (Erlang/OTP module)
- `my_erlang_module` (your custom module)

## Data Type Mapping

### Gleam to Erlang

| Gleam Type | Erlang Type |
|------------|-------------|
| `Int` | Integer |
| `Float` | Float |
| `String` | Binary `<<"text"/utf8>>` |
| `Bool` (`True`/`False`) | Atoms `true`/`false` |
| `List(a)` | List `[a]` |
| `#(a, b)` | Tuple `{a, b}` |
| `Ok(x)` | Tagged tuple `{ok, x}` |
| `Error(x)` | Tagged tuple `{error, x}` |
| Custom types | Tagged tuples |
| `Nil` | Atom `nil` |

### Custom Type Representation

```gleam
pub type Result {
  Ok(value: Int)
  Error(reason: String)
}

// In Erlang becomes:
// {ok, 42}
// {error, <<"reason"/utf8>>}

pub type User {
  Guest
  LoggedIn(id: Int, name: String)
}

// In Erlang becomes:
// {guest}
// {logged_in, 123, <<"Alice"/utf8>>}
```

## Common Erlang Modules

### Lists Module

```gleam
@external(erlang, "lists", "reverse")
pub fn reverse(list: List(a)) -> List(a)

@external(erlang, "lists", "sort")
pub fn sort(list: List(a)) -> List(a)

@external(erlang, "lists", "flatten")
pub fn flatten(list: List(List(a))) -> List(a)
```

### String/Binary Module

```gleam
@external(erlang, "string", "uppercase")
pub fn uppercase(s: String) -> String

@external(erlang, "binary", "split")
pub fn split(s: String, pattern: String) -> List(String)
```

### Crypto Module

```gleam
@external(erlang, "crypto", "hash")
pub fn hash(algorithm: String, data: String) -> BitArray

@external(erlang, "crypto", "strong_rand_bytes")
pub fn random_bytes(n: Int) -> BitArray
```

### Timer Module

```gleam
@external(erlang, "timer", "sleep")
pub fn sleep(milliseconds: Int) -> Nil
```

## Elixir Integration

### Calling Elixir Modules

Elixir modules require "Elixir." prefix:

```gleam
@external(erlang, "Elixir.String", "upcase")
pub fn upcase(string: String) -> String

@external(erlang, "Elixir.Enum", "map")
pub fn enum_map(list: List(a), f: fn(a) -> b) -> List(b)

@external(erlang, "Elixir.Phoenix.HTML", "safe_to_string")
pub fn safe_to_string(html: SafeHtml) -> String
```

See: [External Functions - Elixir](https://gleam.run/documentation/externals/)

### Elixir Data Structures

Elixir maps, structs, and atoms work with Gleam:

```gleam
// Elixir %{key: "value"} maps to Gleam Dict
import gleam/dict.{type Dict}

@external(erlang, "Elixir.MyModule", "get_config")
pub fn get_config() -> Dict(String, String)
```

### Using Elixir Macros

**Elixir macros cannot be called from Gleam**. They must be wrapped in regular functions.

```elixir
# In Elixir (lib/my_wrapper.ex)
defmodule MyWrapper do
  def use_macro(arg) do
    # Call macro here
    SomeMacro.macro_function(arg)
  end
end
```

```gleam
// In Gleam
@external(erlang, "Elixir.MyWrapper", "use_macro")
pub fn use_macro(arg: String) -> Result(Value, Error)
```

See: [Gleam FAQ - Elixir Macros](https://gleam.run/frequently-asked-questions/)

## OTP Integration

### Erlang Processes

Use `gleam_erlang/process`:

```gleam
import gleam/erlang/process.{type Subject}

pub fn spawn_process() {
  process.start(fn() {
    // Process logic
  }, linked: True)
}
```

See: [gleam_erlang - Process](https://hexdocs.pm/gleam_erlang/gleam/erlang/process.html)

### Calling Erlang gen_server

```gleam
@external(erlang, "gen_server", "call")
pub fn gen_server_call(
  server: Subject(a),
  request: b,
  timeout: Int,
) -> c
```

**Better approach**: Use `gleam_otp/actor` for type safety:

See: [Gleam OTP Documentation](https://hexdocs.pm/gleam_otp/)

### ETS (Erlang Term Storage)

```gleam
pub type Table

@external(erlang, "ets", "new")
pub fn new(name: Atom, options: List(Atom)) -> Table

@external(erlang, "ets", "insert")
pub fn insert(table: Table, objects: List(#(a, b))) -> Bool

@external(erlang, "ets", "lookup")
pub fn lookup(table: Table, key: a) -> List(#(a, b))
```

### Atoms

```gleam
import gleam/erlang/atom.{type Atom}

pub fn create_atom(name: String) -> Atom {
  atom.create_from_string(name)
}
```

See: [gleam_erlang - Atom](https://hexdocs.pm/gleam_erlang/gleam/erlang/atom.html)

## Bit Strings / Binary Data

### Pattern Matching on Binaries

```gleam
pub fn parse_packet(data: BitArray) -> Result(Packet, ParseError) {
  case data {
    <<version:8, type_:8, length:16, body:bytes>> -> {
      Ok(Packet(version, type_, length, body))
    }
    _ -> Error(InvalidPacket)
  }
}
```

See: [Bit Array Syntax](https://gearsco.de/blog/bit-array-syntax/)

### Bit Array Options

- Size: `value:32` (32 bits)
- Type: `int`, `float`, `bytes`, `bits`, `utf8`, `utf16`, `utf32`
- Signedness: `signed`, `unsigned`
- Endianness: `big`, `little`, `native`
- Unit: Multiplier for size

```gleam
<<value:size(32)-unsigned-big-integer>> = data
```

## Calling Erlang Libraries

### Hackney (HTTP Client)

```gleam
@external(erlang, "hackney", "request")
pub fn request(
  method: Atom,
  url: String,
  headers: List(#(String, String)),
  body: String,
  options: List(a),
) -> Result(Response, Error)
```

Or use: [gleam_hackney](https://hexdocs.pm/gleam_hackney/)

### Cowboy (HTTP Server)

For web servers, prefer:
- [Mist](https://hexdocs.pm/mist/) - Native Gleam HTTP server
- [Wisp](https://hexdocs.pm/wisp/) - Web framework

### Mnesia (Database)

```gleam
@external(erlang, "mnesia", "create_table")
pub fn create_table(name: Atom, options: List(#(Atom, anything))) -> Result(Atom, Error)

@external(erlang, "mnesia", "transaction")
pub fn transaction(fn: fn() -> a) -> Result(a, Error)
```

## Compiling with Elixir Code

### Mix Projects

Gleam code can be part of Elixir Mix projects:

1. Add `mix_gleam` to dependencies
2. Place Gleam code in `src/`
3. Run `mix compile`

See: [mix_gleam](https://hexdocs.pm/mix_gleam/)

### Using Gleam from Elixir

```elixir
# Call Gleam module from Elixir
:my_gleam_module.my_function("arg")
```

Gleam module `my_package/my_module` becomes Erlang module `:my_package@my_module`.

## Error Handling Interop

### Converting Erlang Error Tuples

```gleam
@external(erlang, "file", "read_file")
fn do_read_file(path: String) -> Result(BitArray, Atom)

pub fn read_file(path: String) -> Result(String, FileError) {
  case do_read_file(path) {
    Ok(content) -> {
      case bit_array.to_string(content) {
        Ok(text) -> Ok(text)
        Error(_) -> Error(InvalidEncoding)
      }
    }
    Error(reason) -> Error(FileSystemError(atom.to_string(reason)))
  }
}
```

### Erlang Exceptions

Erlang exceptions become Gleam panics. Wrap risky Erlang calls:

```gleam
pub fn safe_erlang_call() -> Result(Value, Error) {
  case attempt_erlang_operation() {
    value -> Ok(value)
    _ -> Error(OperationFailed)
  }
}
```

## Testing Erlang Integration

```gleam
pub fn erlang_interop_test() {
  let assert [3, 2, 1] = reverse_list([1, 2, 3])
  let assert "HELLO" = uppercase("hello")
}
```

Test both Gleam and Erlang implementations when available.

## Performance Considerations

### Tail Call Optimization

Erlang optimizes tail calls. Write recursive functions in tail position:

```gleam
// Tail recursive (optimized)
pub fn sum(list: List(Int), acc: Int) -> Int {
  case list {
    [] -> acc
    [x, ..rest] -> sum(rest, acc + x)
  }
}

// Not tail recursive
pub fn sum_bad(list: List(Int)) -> Int {
  case list {
    [] -> 0
    [x, ..rest] -> x + sum_bad(rest)  // Not tail position
  }
}
```

### Binary Handling

Binaries are efficient on BEAM. Use them for large data:

```gleam
// Efficient: builds binary once
pub fn build_large_string(parts: List(String)) -> String {
  string.concat(parts)
}
```

## Debugging

### Erlang Observer

Monitor your application:

```bash
erl -pa build/dev/erlang/*/ebin
```

Then in Erlang shell:
```erlang
observer:start().
```

### IO Debugging

```gleam
import gleam/io

pub fn debug(value: a) -> a {
  io.debug(value)
  value
}
```

## Common Patterns

### Wrapping Erlang Libraries

Create type-safe wrappers for Erlang code:

```gleam
// Low-level external
@external(erlang, "my_erlang_lib", "risky_function")
fn do_risky_function(arg: String) -> anything

// Safe wrapper
pub fn safe_function(arg: String) -> Result(Value, Error) {
  case do_risky_function(arg) {
    #("ok", value) -> Ok(value)
    #("error", reason) -> Error(ConvertReason(reason))
    _ -> Error(UnexpectedResponse)
  }
}
```

### Port Communication

For external programs:

```gleam
@external(erlang, "erlang", "open_port")
pub fn open_port(name: #(Atom, String), options: List(Atom)) -> Port

@external(erlang, "erlang", "port_command")
pub fn port_command(port: Port, data: BitArray) -> Bool
```

---

**Remember**: Gleam's Erlang target gives you full access to the BEAM ecosystem. Wrap external code for type safety.

See: [External Functions](../rules/external-functions.md)
