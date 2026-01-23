# /gleam-build - Build Gleam Project

Build the Gleam project and handle compilation errors.

## Usage

```
/gleam-build [--target=erlang|javascript]
```

## Workflow

1. **Build project**
   ```bash
   gleam build
   ```

   Or for specific target:
   ```bash
   gleam build --target erlang
   gleam build --target javascript
   ```

2. **Handle compilation errors**

   **Type errors**:
   - Read error message carefully
   - Check function signatures match
   - Verify all cases handled in pattern matching
   - Ensure imports are correct

   **Missing dependencies**:
   ```bash
   gleam add <package-name>
   ```

   **Module not found**:
   - Check file path matches module name
   - Verify file is in `src/` directory
   - Check import statement

3. **Verify build success**
   ```bash
   gleam build && gleam test
   ```

4. **Run application**
   ```bash
   gleam run
   ```

## Build Output

- **Erlang target**: `build/dev/erlang/`
- **JavaScript target**: `build/dev/javascript/`

## Common Errors

### Type Mismatch
```
error: Type mismatch
  ┌─ src/main.gleam:10:15
  │
10│   let result = parse_int("42")
  │                ^^^^^^^^^^^^^^^
  │
  This returns: Result(Int, Nil)
  But expected: Int
```

**Fix**: Handle Result with pattern matching or use `result.unwrap`

### Exhaustiveness Check Failed
```
error: This case expression does not match all possibilities
```

**Fix**: Add missing patterns or use catch-all `_`

### Circular Dependency
```
error: Module imports form a cycle
```

**Fix**: Refactor to break circular dependency, extract common code

## Build for Production

### Erlang Target
```bash
gleam export erlang-shipment
```

Creates optimized release in `build/erlang-shipment/`

### JavaScript Target
```bash
gleam build --target javascript
```

Then bundle with esbuild or other bundler.

## References

- [Writing Gleam](https://gleam.run/writing-gleam/)
- [Coding Standards](../rules/coding-standards.md)
- [Deployment](../skills/gleam-deployment/skill.md)

## Best Practices

- Build frequently during development
- Fix warnings (use `-Werror` for strict mode)
- Test both targets if cross-platform
- Use `gleam export` for production builds
- Keep dependencies updated
