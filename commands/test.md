# /gleam-test - Run Gleam Tests

Run tests with appropriate configuration and error handling.

## Usage

```
/gleam-test [--target=erlang|javascript] [--watch] [pattern]
```

## Workflow

1. **Run all tests**
   ```bash
   gleam test
   ```

2. **Run for specific target**
   ```bash
   gleam test --target erlang
   gleam test --target javascript
   ```

3. **Analyze test failures**
   - Read error messages carefully
   - Check `let assert` patterns match actual values
   - Verify types are correct
   - Look for missing edge cases

4. **Fix failures**
   - Update test assertions
   - Fix implementation bugs
   - Add missing test cases

5. **Verify all tests pass**
   ```bash
   gleam test --target erlang && gleam test --target javascript
   ```

## Common Issues

### Test Discovery
- Functions must end in `_test`
- Functions must be `pub`
- Files must be in `test/` directory

### Assertions
- Use `let assert` NOT deprecated `should` module
- Pattern must match exactly
- Use `True = actual == expected` for comparisons

### Timeouts (Erlang only)
- Default timeout: 5 seconds
- For longer tests, use Timeout type pattern

See: [Test Timeouts](https://gearsco.de/blog/gleam-test-timeouts/)

## References

- [Gleeunit Documentation](https://hexdocs.pm/gleeunit/)
- [Testing Practices](../rules/testing-practices.md)
- [Testing Skill](../skills/gleam-testing/SKILL.md)

## Best Practices

- Run tests before committing
- Test both happy and error paths
- Keep tests independent
- Use clear test names
- Test one thing per test function
