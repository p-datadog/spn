# Coding Guidelines for dd-trace-rb Work

This document contains coding standards and guidelines to follow when working on the Datadog dd-trace-rb repository.

## Code Style

### Trailing Commas

**Rule:** Always use trailing commas in multi-line array and hash literals.

**Mandatory Locations:**
- `lib/datadog/di/` directory and all subdirectories
- `lib/datadog/symbol_database/` directory and all subdirectories

**Recommended Everywhere:**
- When adding new code anywhere in the codebase, prefer trailing commas

**Rationale:**
- Cleaner git diffs when adding items
- Prevents missing comma syntax errors
- Consistent with Ruby community best practices
- Easier to reorder items

#### Examples

**✅ Correct:**

```ruby
# Arrays
result = [
  :first_item,
  :second_item,
  :third_item,
]

# Hashes
config = {
  timeout: 30,
  retries: 3,
  enabled: true,
}

# Method calls with multi-line arguments
send_telemetry(
  'event_name',
  error: e,
  probe_id: probe.id,
)

# Hash in method definition
def process(
  data:,
  options: {},
  callback: nil,
)
  # ...
end
```

**❌ Incorrect:**

```ruby
# Arrays without trailing comma
result = [
  :first_item,
  :second_item,
  :third_item
]

# Hashes without trailing comma
config = {
  timeout: 30,
  retries: 3,
  enabled: true
}

# Method calls without trailing comma
send_telemetry(
  'event_name',
  error: e,
  probe_id: probe.id
)
```

#### Single-line Exceptions

Trailing commas are NOT required for single-line arrays/hashes:

```ruby
# OK - single line
result = [:first, :second, :third]
config = { timeout: 30, retries: 3 }
```

#### When to Apply

1. **New Code:** Always use trailing commas in new code in di/ and symbol_database/
2. **Modifying Existing Code:**
   - If you're modifying a multi-line structure in di/ or symbol_database/, add trailing commas
   - If modifying code elsewhere, prefer adding trailing commas but not required
3. **Whole File Changes:** If reformatting an entire file in di/ or symbol_database/, add trailing commas throughout

#### Git Workflow

When adding trailing commas as part of a larger change:
- Include trailing comma changes in the same commit as the functional change
- Don't create separate commits just for trailing commas
- Mention in commit message if making widespread trailing comma additions: "Add trailing commas per coding guidelines"

#### RuboCop Configuration

This style should be enforced by RuboCop's `Style/TrailingCommaInArrayLiteral` and `Style/TrailingCommaInHashLiteral` cops. Check `.rubocop.yml` configuration:

```yaml
Style/TrailingCommaInArrayLiteral:
  EnforcedStyleForMultiline: consistent_comma

Style/TrailingCommaInHashLiteral:
  EnforcedStyleForMultiline: consistent_comma

Style/TrailingCommaInArguments:
  EnforcedStyleForMultiline: consistent_comma
```

---

## Future Guidelines

Additional coding standards will be added here as they are established.

**Topics to document:**
- Error handling patterns
- Testing requirements
- Documentation standards
- Performance considerations
- Security best practices
