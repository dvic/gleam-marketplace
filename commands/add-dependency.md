# /gleam-add - Add Package Dependency

Add and configure package dependencies for Gleam projects.

## Usage

```
/gleam-add <package-name> [--dev]
```

## Workflow

1. **Search for package** (if needed)
   - Visit [hex.pm](https://hex.pm/)
   - Or use MCP Gleam packages server (if available)
   - Check package documentation and compatibility

2. **Add dependency**
   ```bash
   gleam add <package-name>
   ```

   For development dependencies:
   ```bash
   gleam add --dev <package-name>
   ```

3. **Download dependencies**
   ```bash
   gleam deps download
   ```

4. **Verify installation**
   ```bash
   gleam build
   ```

5. **Import and use**
   ```gleam
   import package_name

   pub fn main() {
     package_name.function()
   }
   ```

## Common Packages

### Core
- `gleam_stdlib` - Standard library (usually pre-installed)
- `gleam_json` - JSON encoding/decoding
- `gleam_http` - HTTP types

### Web Development
- `wisp` - Web framework
- `mist` - HTTP server
- `lustre` - Frontend framework

### OTP
- `gleam_otp` - OTP framework
- `gleam_erlang` - Erlang integration

### Testing
- `gleeunit` - Test framework (dev dependency)
- `birdie` - Snapshot testing
- `gleam_qcheck` - Property-based testing

### Utilities
- `envoy` - Environment variables
- `filepath` - Path operations
- `gleam_time` - Date/time handling

See: [Awesome Gleam](https://github.com/gleam-lang/awesome-gleam)

## Managing Dependencies

### Update dependencies
```bash
gleam deps update
```

### Check for outdated packages
```bash
gleam deps outdated
```

### Remove unused dependencies
Edit `gleam.toml` and remove the dependency, then:
```bash
gleam deps download
```

## Version Constraints

In `gleam.toml`:

```toml
[dependencies]
gleam_stdlib = ">= 0.34.0 and < 2.0.0"  # Range
gleam_json = "~> 3.0"                    # Compatible with 3.x
specific = "= 1.2.3"                     # Exact version
```

## References

- [gleam.toml - Dependencies](https://gleam.run/writing-gleam/gleam-toml/)
- [Package Development](../skills/gleam-package-development/skill.md)
- [Hex.pm](https://hex.pm/)

## Best Practices

- Minimize dependencies
- Keep dependencies updated
- Review package quality and maintenance
- Use dev dependencies for tooling
- Commit `manifest.toml` to version control
