# /gleam-new - Create New Gleam Project

Create a new Gleam project with best practices and proper structure.

## Usage

```
/gleam-new [project-name] [--type=app|lib|web|otp]
```

## Workflow

1. **Create project structure**
   ```bash
   gleam new <project-name>
   cd <project-name>
   ```

2. **Add dependencies based on type**

   For **app** (default):
   ```bash
   gleam add gleam_stdlib
   gleam add --dev gleeunit
   ```

   For **lib** (library):
   ```bash
   gleam add gleam_stdlib
   gleam add --dev gleeunit
   ```

   For **web** (web application):
   ```bash
   gleam add wisp mist gleam_http gleam_erlang gleam_json envoy
   gleam add --dev gleeunit
   ```

   For **otp** (OTP application):
   ```bash
   gleam add gleam_otp gleam_erlang
   gleam add --dev gleeunit
   ```

3. **Initialize git**
   ```bash
   git init
   git add .
   git commit -m "Initial commit: Gleam project setup"
   ```

4. **Run initial test**
   ```bash
   gleam test
   ```

5. **Format code**
   ```bash
   gleam format
   ```

## References

- [Writing Gleam](https://gleam.run/writing-gleam/)
- [gleam.toml Documentation](https://gleam.run/writing-gleam/gleam-toml/)
- [Project Structure](../rules/coding-standards.md)

## Next Steps

- Update `README.md` with project description
- Configure CI/CD (GitHub Actions recommended)
- Set up project-specific CLAUDE.md if needed
