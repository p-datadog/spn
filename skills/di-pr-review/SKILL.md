---
name: di-pr-review
description: This skill should be used when the user asks to "review DI PR", "review dynamic instrumentation", "review dd-trace-rb PR", or mentions Datadog dynamic instrumentation pull requests.
version: 0.1.0
---

# Datadog dd-trace-rb Dynamic Instrumentation PR Review

This skill provides specialized review for pull requests in the `datadog/dd-trace-rb` repository related to dynamic instrumentation.

## Overview

This skill enforces strict quality standards for Ruby dynamic instrumentation PRs in the Datadog tracer library. Dynamic instrumentation modifies application behavior at runtime to collect traces and metrics, and must meet the highest standards for safety, correctness, and observability.

## When This Skill Applies

Use this skill when reviewing PRs in `datadog/dd-trace-rb` that involve:
- Dynamic instrumentation features
- Runtime code modification
- TracePoint API usage
- Method interception
- Profiling or tracing instrumentation

## CRITICAL Review Requirements

All 6 requirements below are **BLOCKING**. PRs must not be approved until all issues are resolved.

### 1. NO Skipped Tests (CRITICAL)

**Requirement:** All tests must run and pass. Skipped tests are NOT acceptable.

**Review actions:**
- Search for skip patterns in test files:
  - RSpec: `skip`, `pending`, `xit`, `xdescribe`, `skip_if`
  - Minitest: `skip`, `skip_if`, `skip_unless`
  - Any commented-out test methods
- Flag EVERY skipped test as **CRITICAL**
- Required resolution: Tests MUST be fixed to run and pass
- If you cannot fix the test, ask for assistance
- "Will fix later" is NOT acceptable
- Deleting tests is NOT acceptable

**How to check:**
```bash
# Search for skipped tests
grep -r "^\s*skip\|^\s*pending\|^\s*xit\|^\s*xdescribe" spec/ test/
```

**Example feedback:**
```
❌ CRITICAL: Skipped test found

File: spec/dynamic_instrumentation/probe_spec.rb:145
Line: skip "test fork behavior"

This test MUST be fixed to run and pass reliably.

If you cannot fix this test, ask for assistance from the team.
Do not delete tests - they exist for a reason and must be made to work.

Skipped tests create false confidence. This is blocking approval.
```

### 2. NO Sleep/Wait Calls in Tests (CRITICAL)

**Requirement:** Tests must use deterministic waits, not arbitrary sleep calls.

**Review actions:**
- Search for sleep patterns:
  - `sleep`, `Kernel.sleep`
  - `Thread.pass` in loops
  - Any time-based delays
- Flag ALL sleep calls as **CRITICAL**
- Require replacement with deterministic synchronization

**Acceptable alternatives:**
- Queue-based synchronization: `queue.pop` with timeout
- Condition variables: `ConditionVariable#wait`
- Mock time: `Timecop.freeze`, `travel_to`
- Explicit callbacks: `on_complete { queue.push(:done) }`

**How to check:**
```bash
# Search for sleep in tests
grep -r "sleep\s\|Kernel\.sleep" spec/ test/
```

**Example feedback:**
```
❌ CRITICAL: Non-deterministic sleep found

File: spec/dynamic_instrumentation/async_spec.rb:78
Line: sleep 0.5  # Wait for async processing

This makes tests flaky and slow. Replace with:

# BAD
worker.start_async
sleep 0.5
expect(worker.completed?).to be true

# GOOD
queue = Queue.new
worker.on_complete { queue.push(:done) }
worker.start_async
Timeout.timeout(2) { queue.pop }
expect(worker.completed?).to be true
```

### 3. 100% Code Coverage (CRITICAL)

**Requirement:** Every executable line must be exercised by at least one test.

**Review actions:**
- Verify SimpleCov or similar coverage report is included
- Check that coverage is 100% for all new/modified code
- Ensure all branches are tested (if/else, case/when)
- Verify error paths have test coverage
- Check that rescue blocks are exercised

**How to check:**
```bash
# Run tests with coverage
bundle exec rspec --require simplecov

# Check coverage report
open coverage/index.html
```

**What must be covered:**
- All new methods and classes
- Every branch (if/else, unless, case/when)
- All rescue blocks
- Edge cases and error conditions
- Early returns and guard clauses

**Example feedback:**
```
❌ CRITICAL: Incomplete code coverage (87%)

Uncovered lines in lib/datadog/di/probe_manager.rb:
- Lines 145-152: Error handling for probe registration failure
- Lines 203-207: Cleanup when max probes exceeded
- Line 289: Fork detection branch

Required action:
Add tests that exercise:
1. Probe registration failures
2. Max probe limit scenarios
3. Post-fork behavior

100% coverage is required. No exceptions.
```

### 4. Repository Guidelines Compliance (CRITICAL)

**Requirement:** PR must comply with all dd-trace-rb repository guidelines.

**Review actions:**
- Check for repository-specific guidelines:
  - Read `CLAUDE.md` if present
  - Read `CONTRIBUTING.md`
  - Check `.rubocop.yml` configuration
  - Review CI/CD requirements
- Verify compliance with:
  - Code style (RuboCop)
  - Commit message format
  - Documentation requirements
  - CHANGELOG updates
  - Version compatibility

**How to check:**
```bash
# Check for guideline files
ls -la CLAUDE.md CONTRIBUTING.md .rubocop.yml

# Run RuboCop
bundle exec rubocop

# Check CI status
gh pr checks
```

**Example feedback:**
```
❌ CRITICAL: Repository guidelines not followed

Per CLAUDE.md Section 3.2:
"All dynamic instrumentation features must document performance impact and provide overhead measurements."

Missing required documentation:
- No performance impact section in lib/datadog/di/probe.rb
- No overhead benchmarks in PR description
- No sampling rate recommendations

This is blocking approval.
```

### 5. Error Boundaries - NO Exception Propagation (CRITICAL)

**Requirement:** Dynamic instrumentation must NEVER propagate exceptions to customer applications.

**Exception:** If `settings.dynamic_instrumentation.internal.propagate_all_exceptions` is set to true, exceptions may escape DI code. This is an internal setting used for debugging and testing purposes only.

**Principle:** Customer code must execute identically whether instrumentation is enabled or disabled. Any instrumentation failure must be contained, logged, and reported—never raised to customer code.

**Review actions:**
- Verify ALL TracePoint callbacks rescue exceptions
- Check that all instrumentation entry points have error boundaries
- Ensure exceptions are logged but never propagated
- Validate safe fallback behavior on errors
- Confirm instrumentation failures don't affect application flow

**Common error boundary locations:**
- TracePoint callbacks
- Method interception (prepended modules)
- Probe execution
- Metric collection
- Serialization/export

**How to check:**
```bash
# Look for TracePoint without rescue
grep -A 10 "TracePoint.new" lib/datadog/di/ | grep -v "rescue"

# Look for prepended methods without error handling
grep -A 5 "def.*super" lib/datadog/di/ | grep -v "rescue"
```

**Required pattern:**
```ruby
# GOOD - Error boundary in TracePoint
trace = TracePoint.new(:call) do |tp|
  begin
    probe.execute(tp)
  rescue => e
    Datadog.logger.debug("Probe execution failed: #{e}")
    telemetry.error('probe_execution_failed', e)
  end
end

# GOOD - Error boundary in method interception
module ProbeInstrumentation
  def instrumented_method(*args)
    begin
      probe.before_call
    rescue => e
      Datadog.logger.debug("Probe failed: #{e}")
      telemetry.error('probe_before_failed', e)
    end
    super
  end
end
```

**BAD patterns to flag:**
```ruby
# BAD - No error boundary
trace = TracePoint.new(:call) do |tp|
  probe.execute(tp)  # Can raise to customer code!
end

# BAD - Partial error boundary
def instrumented_method(*args)
  probe.before_call  # Can raise!
  begin
    super
  rescue => e
    # Wrong place for rescue!
  end
end
```

**Example feedback:**
```
❌ CRITICAL: Exception can propagate to customer code

File: lib/datadog/di/probe_manager.rb:67-72

Code:
```ruby
@trace = TracePoint.new(:call, :return) do |tp|
  probe = find_probe(tp.path, tp.lineno)
  probe.execute(tp) if probe  # Line 70 - NO ERROR HANDLING
end
```

This violates error boundary requirements. If probe.execute raises,
the exception will propagate to the customer's application code.

Required fix:
```ruby
@trace = TracePoint.new(:call, :return) do |tp|
  begin
    probe = find_probe(tp.path, tp.lineno)
    probe.execute(tp) if probe
  rescue => e
    Datadog.logger.debug("DI probe execution failed: #{e}")
    Datadog.telemetry.error('di.probe.execution_failed', error: e)
  end
end
```

Required test coverage for error boundaries:
- [ ] Test TracePoint with probe.execute raising exceptions
- [ ] Test with resource exhaustion (full buffers, etc.)
- [ ] Test with invalid probe configuration
- [ ] Verify customer code unaffected by instrumentation failures
- [ ] Test exception types: StandardError, NoMemoryError, etc.
```

### 6. Error Handling and Telemetry (CRITICAL)

**Requirement:** All rescued exceptions must be:
1. Logged at appropriate level (DEBUG or WARN)
2. Reported via telemetry

**Logging level rules:**
- **DEBUG**: Internal errors customer cannot fix (I/O errors, internal state issues, resource exhaustion)
- **WARN**: Configuration/environment issues customer CAN fix (invalid config, missing files, permissions)

**Review actions:**
- Check EVERY rescue block has both logging AND telemetry
- Verify log level matches whether customer can fix the issue
- Ensure telemetry includes sufficient context
- Check that error metrics are exported

**How to check:**
```bash
# Find rescue blocks
grep -A 5 "rescue\s*=>" lib/datadog/di/

# Check for missing telemetry
grep -A 5 "rescue\s*=>" lib/datadog/di/ | grep -L "telemetry"
```

**Required pattern:**
```ruby
# GOOD - Internal error (DEBUG + telemetry)
begin
  export_snapshots(buffer)
rescue IOError => e
  Datadog.logger.debug("Snapshot export failed: #{e}")
  Datadog.telemetry.error('di.snapshot.export_failed',
    error: e,
    buffer_size: buffer.size
  )
end

# GOOD - Customer-fixable error (WARN + telemetry)
begin
  load_probe_config(file)
rescue ConfigError => e
  Datadog.logger.warn("Invalid probe configuration in #{file}: #{e.message}")
  Datadog.telemetry.error('di.config.invalid',
    error: e,
    config_file: file
  )
end
```

**BAD patterns to flag:**
```ruby
# BAD - Silent failure (no logging or telemetry)
begin
  collect_snapshot
rescue
  # Nothing!
end

# BAD - Logged but no telemetry
begin
  collect_snapshot
rescue => e
  Datadog.logger.debug("Error: #{e}")
  # Missing telemetry!
end

# BAD - Telemetry but no logging
begin
  collect_snapshot
rescue => e
  Datadog.telemetry.error('snapshot_failed', error: e)
  # Missing logging!
end

# BAD - Wrong log level (internal error at WARN)
begin
  flush_buffer
rescue IOError => e
  Datadog.logger.warn("Buffer flush failed: #{e}")  # Should be DEBUG
  Datadog.telemetry.error('buffer_flush_failed', error: e)
end

# BAD - Wrong log level (customer-fixable at DEBUG)
begin
  validate_probe_config(config)
rescue ValidationError => e
  Datadog.logger.debug("Config invalid: #{e}")  # Should be WARN
  Datadog.telemetry.error('config_invalid', error: e)
end
```

**Telemetry requirements:**
- Include error class and message
- Include context (operation, affected component)
- Include relevant IDs (probe_id, snapshot_id, etc.)
- Increment error counter metric

**Example feedback:**
```
❌ CRITICAL: Rescued exception not properly handled

File: lib/datadog/di/snapshot_serializer.rb:89-92

Code:
```ruby
begin
  serialize_locals(binding)
rescue => e
  Datadog.logger.debug("Serialization failed: #{e}")
end
```

Issues:
1. Missing telemetry reporting
2. Insufficient error context in log

Required fix:
```ruby
begin
  serialize_locals(binding)
rescue => e
  Datadog.logger.debug("Snapshot variable serialization failed: #{e}")
  Datadog.telemetry.error('di.snapshot.serialization_failed',
    error: e,
    probe_id: probe.id,
    variable_count: binding.local_variables.size
  )
  Datadog.metrics.increment('di.errors.serialization')
end
```

---

File: lib/datadog/di/config_loader.rb:134-137

Code:
```ruby
begin
  parse_probe_definition(json)
rescue JSON::ParserError => e
  Datadog.logger.debug("Parse failed: #{e}")  # Wrong level!
end
```

Issues:
1. Wrong log level (customer CAN fix invalid JSON)
2. Missing telemetry

Required fix:
```ruby
begin
  parse_probe_definition(json)
rescue JSON::ParserError => e
  Datadog.logger.warn("Failed to parse probe definition: #{e.message}. Check probe JSON syntax.")
  Datadog.telemetry.error('di.probe.parse_failed',
    error: e,
    json_snippet: json[0..100]
  )
end
```

Decision guide for log levels:
- DEBUG: Network errors, I/O errors, serialization failures, buffer full, internal state issues
- WARN: Invalid configuration, missing required files, permission errors, unsupported Ruby version
```

## Line Probe Test Files

**Overview:** The DI test suite includes special test files that use line numbers to test line probes. These files have specific line numbers referenced in test assertions, and changing line numbers breaks the tests.

**Critical Requirement:** When these files are modified, verify that:
1. Changes only add lines at the end of the file, OR
2. All test assertions referencing line numbers are updated accordingly

### Identifying Line Probe Test Files

Line probe test files typically:
- Are located in `spec/datadog/di/` directory
- Have names like `*_line_probe_*`, `*_target_*`, or `*_probe_test_file_*`
- Contain comments marking line numbers: `# line probe at 42`, `# Line: 123`, etc.
- Are referenced by tests that specify exact line numbers in probe configurations

**Common patterns to search for:**
```bash
# Find test files with line number markers
grep -r "# line" spec/datadog/di/
grep -r "# Line:" spec/datadog/di/

# Find test files referenced by line number in specs
grep -r "lineno.*=.*[0-9]" spec/datadog/di/
grep -r "line:.*[0-9]" spec/datadog/di/
```

### Verification Steps

When a line probe test file is modified in the PR:

1. **Check the nature of changes:**
   ```bash
   # View the diff for the file
   gh pr diff <PR_NUMBER> -- path/to/test_file.rb
   ```

2. **Determine if lines were added/removed/changed:**
   - ✅ **Safe:** Lines only added at the end of the file
   - ⚠️ **Requires verification:** Lines added/removed/changed in the middle

3. **If middle changes detected, find all line number references:**
   ```bash
   # Find tests that reference this file by line number
   grep -r "$(basename test_file.rb)" spec/datadog/di/ | grep -E "lineno|line:"
   ```

4. **Verify test updates:**
   - Check that all line number references in tests are updated
   - Ensure line numbers still point to the intended code
   - Run the specific tests to verify they pass

### Example Scenarios

#### Scenario 1: Safe - Lines Added at End ✅

```ruby
# File: spec/datadog/di/some_test_file.rb
class TestTarget
  def method_one  # Line 2 - referenced in tests
    puts "one"
  end

  def method_two  # Line 6 - referenced in tests
    puts "two"
  end

  # NEW: Lines added at end - no line numbers changed
  def method_three  # Line 11 - NEW
    puts "three"
  end
end
```

**Verification:** ✅ No updates needed - existing line numbers unchanged

#### Scenario 2: Dangerous - Lines Added in Middle ⚠️

```ruby
# File: spec/datadog/di/some_test_file.rb
class TestTarget
  def method_one  # Line 2 - referenced in tests
    puts "one"
  end

  # NEW: Line added here shifts everything below
  def method_new  # Line 7 - NEW
    puts "new"
  end

  def method_two  # Line 11 - MOVED FROM LINE 6!
    puts "two"
  end
end
```

**Required verification:**
- Tests referencing line 6 must now reference line 11
- Check all tests that use this file with line numbers

#### Scenario 3: Test Updated Correctly ✅

```ruby
# Before PR:
it 'sets probe at method_two' do
  probe = Datadog::DI::Probe.new(
    file: 'spec/datadog/di/some_test_file.rb',
    lineno: 6  # method_two
  )
  # test continues...
end

# After PR (correctly updated):
it 'sets probe at method_two' do
  probe = Datadog::DI::Probe.new(
    file: 'spec/datadog/di/some_test_file.rb',
    lineno: 11  # method_two - UPDATED
  )
  # test continues...
end
```

#### Scenario 4: Test NOT Updated - BUG ❌

```ruby
# After PR (NOT updated - still references old line):
it 'sets probe at method_two' do
  probe = Datadog::DI::Probe.new(
    file: 'spec/datadog/di/some_test_file.rb',
    lineno: 6  # BUG: This is now method_new, not method_two!
  )
  # test continues...
end
```

**This will cause test failures or incorrect behavior!**

### Review Checklist for Line Probe Files

When a line probe test file is modified:

- [ ] **Identify the file as a line probe test file**
  - Check filename patterns
  - Look for line number markers in comments
  - Find references in test specs

- [ ] **Analyze the changes:**
  - [ ] Lines only added at end? → No further action needed ✅
  - [ ] Lines added/removed in middle? → Continue verification ⚠️

- [ ] **Find all line number references:**
  ```bash
  grep -r "$(basename modified_file.rb)" spec/datadog/di/
  ```

- [ ] **For each reference, verify:**
  - [ ] Line number still points to correct method/statement
  - [ ] Test assertions are still valid
  - [ ] Test passes with updated line numbers

- [ ] **Run affected tests:**
  ```bash
  bundle exec rspec spec/datadog/di/*_spec.rb -e "pattern matching test name"
  ```

### Identifying Line Probe Test Files

Line probe test files are typically used in tests that create probes with specific line numbers. To identify them:

```bash
# Find files referenced with lineno in tests
grep -r "lineno:" spec/datadog/di/ | grep -oP 'file:.*\.rb' | sort -u

# Find test files that might be used as probe targets
find spec/datadog/di -name "*target*" -o -name "*probe_test*" -o -name "code_tracker_test_class*"
```

Common patterns:
- Files with `target`, `probe_test`, or `test_class` in the name
- Files in `spec/datadog/di/` that are imported but not run as tests
- Files referenced with `lineno:` parameter in test specs

**Note:** Always search for line number references when modifying any file in `spec/datadog/di/`.

### Commands Reference

```bash
# Find files referenced with line numbers in tests
grep -r "lineno:" spec/datadog/di/ | grep -oP 'file:.*\.rb' | sort -u

# Find potential line probe test files
find spec/datadog/di -name "*target*" -o -name "*probe_test*" -o -name "code_tracker_test_class*"

# Check if a specific file is referenced by line number
file="your_file.rb"
grep -r "$file" spec/datadog/di/ | grep -E "lineno|line:"

# View PR diff for specific file
gh pr diff <PR_NUMBER> -- path/to/file.rb

# Run specific tests
bundle exec rake spec:main
```

### Feedback Template

When line probe test files are modified incorrectly:

```markdown
⚠️ WARNING: Line probe test file modified

**File:** [path to modified file]

**Issue:** Lines were added/removed in the middle of the file, which shifts line numbers for existing methods. Tests that reference this file by line number may now be broken.

**Analysis:**
- Line X was `def method_name`, now it's `def new_method`
- Line Y is now `def method_name` (moved from line X)

**Tests that need verification:**
[List specific test files and line numbers that reference the modified file]

**Required action:**
1. Search for all line number references to this file:
   ```bash
   grep -r "$(basename modified_file.rb)" spec/datadog/di/ | grep "lineno"
   ```
2. Update all line number references to match the new line positions
3. Verify the tests still pass and test the intended behavior
4. Consider adding comments to mark important line numbers:
   ```ruby
   def method_name  # Line Y - used in probe tests
     # method body
   end
   ```

**Alternative:** If possible, add new methods at the end of the file to avoid shifting existing line numbers.
```

## Review Checklist

When reviewing a dd-trace-rb DI PR, verify:

**CRITICAL (Blocking Approval):**
- [ ] ✅ NO skipped tests (no skip, pending, xit, xdescribe)
- [ ] ✅ NO sleep in tests (use Queue, ConditionVariable, mock time)
- [ ] ✅ 100% code coverage (all lines, branches, error paths)
- [ ] ✅ Complies with repository guidelines (CLAUDE.md, CONTRIBUTING.md, RuboCop)
- [ ] ✅ NO exception propagation (all instrumentation errors contained)
- [ ] ✅ Proper error handling (all rescues have DEBUG/WARN logging + telemetry)

**Additional Quality Checks:**
- [ ] TracePoint callbacks have proper cleanup (tp.disable in ensure)
- [ ] No infinite recursion (instrumentation doesn't trace itself)
- [ ] Bounded memory usage (no binding leaks)
- [ ] Performance overhead documented and <5%
- [ ] Thread-safe shared state access
- [ ] Fork/fiber edge cases handled
- [ ] Line probe test files verified (see Line Probe Test Files section)
- [ ] Documentation updated
- [ ] CHANGELOG updated if required
- [ ] Trailing commas used in di/ and symbol_database/ directories (per CLAUDE.md)

## Review Process

To review a dd-trace-rb dynamic instrumentation PR:

1. **Fetch the PR**: Use `gh pr view <number>` or `gh pr checkout <number>`
2. **Read changed files**: Focus on new instrumentation code
3. **Check for line probe test file modifications**:
   - Identify if any line probe test files were modified
   - Verify line number changes are safe (additions at end) or tests are updated
4. **Run critical checks**:
   - Search for skipped tests
   - Search for sleep in tests
   - Check code coverage report
   - Verify error boundaries (all TracePoint callbacks, prepended methods)
   - Check error handling (all rescue blocks have logging + telemetry)
5. **Review repository compliance**: Check CLAUDE.md, CONTRIBUTING.md, RuboCop
6. **Run tests**: `bundle exec rspec` with coverage
7. **Provide feedback**: Use the checklist above, flag all CRITICAL issues

## Commands to Run

```bash
# Fetch PR
gh pr checkout <PR_NUMBER>

# Check for skipped tests
grep -rn "^\s*skip\|^\s*pending\|^\s*xit\|^\s*xdescribe" spec/ test/

# Check for sleep in tests
grep -rn "sleep\s\|Kernel\.sleep" spec/ test/

# Run tests with coverage
bundle exec rspec --require simplecov

# Check RuboCop compliance
bundle exec rubocop

# Check for TracePoint without error boundaries
grep -A 10 "TracePoint.new" lib/datadog/di/ | grep -v "rescue"

# Check for rescue blocks without telemetry
grep -rn "rescue\s*=>" lib/datadog/di/ | while read line; do
  file=$(echo "$line" | cut -d: -f1)
  lineno=$(echo "$line" | cut -d: -f2)
  if ! sed -n "${lineno},$((lineno+5))p" "$file" | grep -q "telemetry"; then
    echo "Missing telemetry: $file:$lineno"
  fi
done

# Check for modified line probe test files
gh pr diff <PR_NUMBER> --name-only | grep -E "spec/datadog/di/.*target|spec/datadog/di/.*probe_test"

# Find line number references in tests
grep -rn "lineno.*=.*[0-9]\|line:.*[0-9]" spec/datadog/di/

# Check if specific file is referenced by line number
file="test_target.rb"
grep -r "$file" spec/datadog/di/ | grep -E "lineno|line:"
```

## Output Format

Provide review feedback in this structure:

```
# Dynamic Instrumentation PR Review

## Summary
[Brief overview of what the PR does]

## CRITICAL Issues (Blocking Approval)

### ❌ 1. Skipped Tests
[List all skipped tests with file:line]

### ❌ 2. Sleep in Tests
[List all sleep calls with file:line]

### ❌ 3. Code Coverage
[Coverage percentage and uncovered lines]

### ❌ 4. Repository Guidelines
[Any guideline violations]

### ❌ 5. Error Boundaries
[Any missing error boundaries in instrumentation]

### ❌ 6. Error Handling
[Any rescue blocks without logging or telemetry]

## Additional Issues

### ⚠️ Line Probe Test Files
[If any line probe test files were modified:]
- List modified files
- Note if lines were added in middle (shifting line numbers)
- List tests that may need line number updates
- Verify tests have been updated or additions were at end of file

[Other non-blocking issues]

## Recommendations
[Suggestions for improvement]

## Approval Status
- [ ] All CRITICAL issues resolved
- [ ] Ready for approval
```
