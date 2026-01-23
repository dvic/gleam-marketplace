# /gleam-format - Format Gleam Code

Format Gleam code according to official style guidelines.

## Usage

```
/gleam-format [--check]
```

## Workflow

1. **Format all code**
   ```bash
   gleam format
   ```

   This formats:
   - All `.gleam` files in `src/`
   - All `.gleam` files in `test/`
   - All `.gleam` files in `dev/`

2. **Check formatting (CI/CD)**
   ```bash
   gleam format --check
   ```

   Exits with error if code is not formatted.

3. **Commit formatted code**
   ```bash
   git add .
   git commit -m "Format code"
   ```

## What Gets Formatted

- Indentation (2 spaces)
- Line length (default 80, configurable)
- Import organization
- Pattern matching alignment
- Function call formatting
- Pipe operator formatting

## Pre-commit Hook

Create `.git/hooks/pre-commit`:

```bash
#!/bin/sh
gleam format --check
if [ $? -ne 0 ]; then
  echo "Code is not formatted. Run: gleam format"
  exit 1
fi
```

```bash
chmod +x .git/hooks/pre-commit
```

## CI/CD Integration

In GitHub Actions:

```yaml
- name: Check formatting
  run: gleam format --check
```

## References

- [Writing Gleam](https://gleam.run/writing-gleam/)
- [Tool Configuration](../rules/coding-standards.md)

## Best Practices

- Format code before committing
- Use `--check` in CI/CD
- Configure max_width if needed
- Let the formatter handle style
- Don't fight the formatter
