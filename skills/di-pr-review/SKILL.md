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

## Fundamental Principle: Test Failures Are Always Relevant

**CRITICAL MINDSET:** All test failures indicate real problems. There is no such thing as an "irrelevant" test failure.

### The Wrong Mindset

❌ **"Tests fail due to environment issues, so the code is fine and tests are the problem"**
- Leads to skipping tests
- Leads to accepting flaky tests
- Leads to working around test failures instead of fixing root causes
- Results in untestable, unreliable code

### The Correct Mindset

✅ **"If tests can't reliably verify behavior, the code isn't testable. That's a design flaw."**

When a test fails or can't pass reliably, one of these is true:
1. **Code needs refactoring** - Add hooks, expose state, inject dependencies
2. **Code needs redesign** - Use a different architecture that's testable
3. **Feature should be removed** - If behavior can't be verified, it can't be trusted

**Never acceptable:** "It works in production so test failures don't matter"

### What Makes Code Testable

**✅ Testable code has:**
- **Observable state** - Can check results and verify outcomes
- **Controllable inputs** - Can inject mocks, fakes, test doubles
- **Deterministic behavior** - Same inputs always produce same outputs
- **Synchronization hooks** - Callbacks, queues, or completion signals for async operations

**❌ Untestable code has:**
- **Hidden internal state** - No way to observe or verify behavior
- **Hard-coded dependencies** - Can't inject test doubles or control behavior
- **Non-deterministic timing** - Relies on sleep, race conditions, arbitrary waits
- **Fire-and-forget async** - Threads/background work with no completion hooks

### For Any Failing Test

**Don't ask:** "How do I make this test pass despite the code?"

**Ask:** "What needs to change in the code to make it testable?"

**Then:** Refactor the code, don't work around it in tests.

### Examples of Testability Problems

**❌ Example 1: Fire-and-forget thread**
```ruby
def start_worker
  Thread.new do
    loop do
      process_queue
      sleep 1
    end
  end
  nil  # No way to know when worker started or check its state
end

# Test has no choice but to use sleep and hope
it 'processes items' do
  worker.start_worker
  sleep 2  # Flaky: hope worker started and ran
  expect(queue).to be_empty
end
```

**✅ Fix: Add observable state and synchronization**
```ruby
def start_worker
  @worker_started = Queue.new
  @worker_thread = Thread.new do
    @worker_started.push(true)
    loop do
      process_queue
      sleep 1
    end
  end
  @worker_started.pop  # Block until worker confirms it started
end

def stop_worker
  @worker_thread.kill if @worker_thread
end

# Test is deterministic
it 'processes items' do
  worker.start_worker  # Blocks until worker is ready
  expect(queue).to be_empty
  worker.stop_worker
end
```

**❌ Example 2: Hard-coded dependency**
```ruby
def export_data
  HTTPClient.post('https://api.example.com/data', data)
end

# Test can't control HTTP behavior, must use real network or VCR
it 'exports data' do
  # Flaky: depends on network, real API, timing
  expect { exporter.export_data }.not_to raise_error
end
```

**✅ Fix: Inject dependency**
```ruby
def initialize(http_client: HTTPClient.new)
  @http_client = http_client
end

def export_data
  @http_client.post('https://api.example.com/data', data)
end

# Test is fast and deterministic
it 'exports data' do
  mock_client = double('http_client')
  expect(mock_client).to receive(:post).with(anything, data)
  exporter = Exporter.new(http_client: mock_client)
  exporter.export_data
end
```

**❌ Example 3: Hidden state**
```ruby
def process_probe(probe)
  @internal_cache[probe.id] = probe
  update_internal_metrics
  # No way to verify cache was updated or metrics were recorded
end

# Test can't verify behavior
it 'processes probe' do
  processor.process_probe(probe)
  # ??? How to check if it worked?
end
```

**✅ Fix: Expose state or return results**
```ruby
def process_probe(probe)
  @probes[probe.id] = probe
  metrics = update_metrics(probe)
  { cached: true, metrics: metrics }
end

attr_reader :probes  # Or provide a query method

# Test can verify behavior
it 'processes probe' do
  result = processor.process_probe(probe)
  expect(result[:cached]).to be true
  expect(processor.probes[probe.id]).to eq(probe)
  expect(result[:metrics][:count]).to eq(1)
end
```

### Application to Review

When reviewing a PR:

1. **If tests are skipped:** Don't accept it. The code needs to be refactored to be testable.

2. **If tests use sleep:** The code needs synchronization hooks (callbacks, queues, completion signals).

3. **If tests are flaky:** The code has non-deterministic behavior that needs to be fixed.

4. **If tests can't verify behavior:** The code needs observable state or return values.

5. **If code "works in production but tests fail":** The code isn't properly testable and needs redesign.

### Summary

**Test failures are not test problems - they're code design problems.**

The solution is always to improve the code's testability, not to skip tests, add sleeps, or work around the tests.

This is a fundamental principle that applies to ALL the review requirements below.

## CRITICAL Review Requirements

All 7 requirements below are **BLOCKING**. PRs must not be approved until all issues are resolved.

### 1. NO Skipped Tests (CRITICAL)

**Requirement:** All tests must run and pass. Skipped tests are NOT acceptable.

**Principle:** If a test can't pass, the code isn't testable. Fix the code, not the test. (See "Fundamental Principle" section above.)

**Review actions:**
- Search for skip patterns in test files:
  - RSpec: `skip`, `pending`, `xit`, `xdescribe`, `skip_if`
  - Minitest: `skip`, `skip_if`, `skip_unless`
  - Any commented-out test methods
- Flag EVERY skipped test as **CRITICAL**
- Required resolution: Tests MUST be fixed to run and pass
- The fix is almost always to refactor the CODE to be testable, not to work around it in tests
- If you cannot fix the test, ask for assistance
- "Will fix later" is NOT acceptable
- Deleting tests is NOT acceptable

**Common reasons for skipped tests and how to fix:**

| Reason | Wrong Fix | Right Fix |
|--------|-----------|-----------|
| "Flaky test" | Skip it | Add synchronization hooks to code (callbacks, queues) |
| "Timing issues" | Use longer sleep | Add deterministic waits, expose completion signals |
| "Hard to test" | Skip it | Refactor code to inject dependencies |
| "Environment specific" | Skip conditionally | Make code work in all environments or mock environment |
| "Race condition" | Skip it | Add proper synchronization primitives to code |

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

**The problem is NOT the test - it's that the code isn't testable.**

The fork behavior needs to be refactored to be observable and verifiable:
- Add hooks to detect when fork has completed
- Expose state to verify fork behavior
- Use dependency injection to make fork behavior controllable in tests

If you cannot fix this test, ask for assistance from the team.
Do not delete tests - they exist for a reason and must be made to work.

Skipped tests create false confidence. This is blocking approval.
```

### 2. NO Sleep/Wait Calls in Tests (CRITICAL)

**Requirement:** Tests must use deterministic waits, not arbitrary sleep calls.

**Principle:** Sleep in tests is a symptom of untestable code. The code needs synchronization hooks, not the test needs longer sleeps. (See "Fundamental Principle" section above.)

**Review actions:**
- Search for sleep patterns:
  - `sleep`, `Kernel.sleep`
  - `Thread.pass` in loops
  - Any time-based delays
- Flag ALL sleep calls as **CRITICAL**
- Require replacement with deterministic synchronization **in the production code**, not workarounds in tests

**The root cause is always untestable code:**
- Code starts async work but provides no completion signal
- Code has internal timing dependencies with no hooks
- Code lacks observable state to check readiness
- Code provides no way to wait for operations to complete

**The fix is to refactor the CODE to add:**
- Completion callbacks or signals
- Observable state (status flags, queues)
- Synchronization primitives (Queue, ConditionVariable)
- Return values indicating completion
- Hooks for test control

**Acceptable patterns (require code changes):**
- Queue-based synchronization: `queue.pop` with timeout (add queue to code)
- Condition variables: `ConditionVariable#wait` (add ConditionVariable to code)
- Mock time: `Timecop.freeze`, `travel_to` (for time-based logic only)
- Explicit callbacks: `on_complete { queue.push(:done) }` (add callback to code)

**CRITICAL: dd-trace-rb Test Suite Default - Flush Calls**

**In dd-trace-rb, the default synchronization mechanism is flush calls, NOT sleep.**

The test suite uses flush methods to ensure async operations complete:
- `tracer.flush` - Flushes traces to backend
- `writer.flush` - Flushes writer queue
- `worker.flush` - Flushes worker queue
- `buffer.flush` - Flushes buffer contents

**When you see sleep after flush:**

```ruby
# SUSPICIOUS PATTERN - Investigate carefully
tracer.flush
sleep 0.1  # Why is this needed?
expect(backend.traces).to include(expected_trace)
```

**Required analysis:**

1. **First, investigate if sleep is needed at all:**
   - Try removing the sleep - does the test still pass?
   - If test passes without sleep, **DEMAND REMOVAL** - the sleep is cargo-culted
   - If test fails without sleep, proceed to step 2

2. **If flush alone is insufficient, require clear explanation:**
   - **WHY is flush insufficient?** Provide specific technical reason
   - **PROOF:** Show test failure without sleep (stack trace, timing info)
   - **ROOT CAUSE:** What async operation is flush not waiting for?

3. **Then suggest proper solutions (in order of preference):**

   **a) Fix the flush method to be complete:**
   ```ruby
   # BEST - Make flush actually wait for everything
   def flush
     writer.flush  # Wait for writer
     worker.join   # Wait for worker thread
     backend.wait  # Wait for backend completion
   end
   ```

   **b) Add explicit wait for the missing operation:**
   ```ruby
   # GOOD - Wait for the specific operation flush doesn't cover
   tracer.flush
   backend.wait_for_upload  # Explicit wait for what flush misses
   expect(backend.traces).to include(expected_trace)
   ```

   **c) Use queue-based synchronization:**
   ```ruby
   # ACCEPTABLE - Add completion signal
   completion = Queue.new
   tracer.flush { completion.push(:done) }
   Timeout.timeout(2) { completion.pop }
   expect(backend.traces).to include(expected_trace)
   ```

   **d) Sleep is ONLY acceptable as LAST RESORT with:**
   - Clear comment explaining WHY flush is insufficient
   - Reference to issue/ticket to fix the underlying problem
   - Timeout to fail fast if operation never completes

   ```ruby
   # LAST RESORT - Document why this is needed
   tracer.flush
   # TODO(#12345): flush doesn't wait for background upload thread
   # Remove this sleep once flush is fixed to wait for upload completion
   Timeout.timeout(2) do
     sleep 0.01 until backend.traces.include?(expected_trace)
   end
   ```

**Example feedback patterns:**

**Pattern 1: Sleep probably not needed**
```markdown
❌ CRITICAL: Sleep after flush without justification

File: spec/datadog/tracing/flush_spec.rb:45
Code:
```ruby
tracer.flush
sleep 0.1
expect(backend.traces).to eq([trace])
```

**This sleep is likely not needed.** Flush should wait for all operations.

**Required action:**
1. Remove the sleep
2. Run the test 100 times: `for i in {1..100}; do bundle exec rspec spec/file.rb:45; done`
3. If it passes consistently → commit without sleep
4. If it fails → proceed to analysis (see next pattern)
```

**Pattern 2: Flush insufficient - needs investigation**
```markdown
❌ CRITICAL: Sleep after flush without clear explanation

File: spec/datadog/tracing/flush_spec.rb:45
Code:
```ruby
tracer.flush
sleep 0.1  # Wait for background thread
expect(backend.traces).to eq([trace])
```

**Why is flush insufficient? Provide proof.**

**Required information:**
1. **What operation is flush not waiting for?**
   - Background thread? Which one?
   - Network upload? To where?
   - Queue processing? Which queue?

2. **Proof that sleep is needed:**
   - Show test failure without sleep (paste stack trace)
   - Show timing info: when does operation actually complete?
   - Show what flush DOES wait for vs what's missing

3. **Proper fix (not sleep):**
   - Fix flush to wait for background thread: `worker.join`
   - Add explicit wait: `backend.wait_for_upload`
   - Add completion callback: `tracer.flush { queue.push(:done) }`

**Do not use sleep as a workaround for incomplete flush.**
```

**Pattern 3: Acceptable sleep (rare)**
```markdown
✅ Sleep after flush - acceptable with clear justification

File: spec/datadog/tracing/flush_spec.rb:45
Code:
```ruby
tracer.flush
# TODO(#12345): flush doesn't wait for background upload thread completion
# The upload happens in a separate thread that flush doesn't join
# Fix: Make flush wait for upload_thread.join
Timeout.timeout(2) do
  sleep 0.01 until backend.received_upload?
end
expect(backend.traces).to eq([trace])
```

**This is acceptable because:**
1. Clear comment explains why flush is insufficient (background upload thread)
2. TODO with ticket number to fix the underlying issue
3. Timeout to fail fast (2 seconds)
4. Polls for explicit condition (not arbitrary delay)

**However, still recommend fixing flush to wait for upload thread.**
```

**Summary: Sleep After Flush Checklist**

When reviewing sleep after flush:
- [ ] Question: Is sleep needed? Remove and test 100x
- [ ] If needed: Require explanation WHY flush insufficient
- [ ] Require proof: Test failure without sleep
- [ ] Require root cause: What operation is flush missing?
- [ ] Suggest fixes: Fix flush, add explicit wait, or use queue
- [ ] Last resort: Sleep with comment, TODO ticket, and timeout
- [ ] Never accept: Sleep without clear technical justification

**Default stance: Flush should be sufficient. If it's not, fix flush, don't add sleep.**

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

**The problem is NOT the test - the PRODUCTION CODE lacks synchronization hooks.**

Current code (untestable):
```ruby
# lib/datadog/di/worker.rb
def start_async
  Thread.new { do_work }  # Fire and forget - no way to know when done
end
```

Test has no choice but to sleep:
```ruby
# BAD (but only option with current code)
worker.start_async
sleep 0.5  # Hope it's done?
expect(worker.completed?).to be true
```

**Required fix: Add completion callback to production code**

```ruby
# lib/datadog/di/worker.rb - REFACTORED
def start_async(&on_complete)
  Thread.new do
    do_work
    on_complete&.call
  end
end
```

Now test can be deterministic:
```ruby
# GOOD (requires code change above)
queue = Queue.new
worker.start_async { queue.push(:done) }
Timeout.timeout(2) { queue.pop }
expect(worker.completed?).to be true
```

**This requires changing lib/datadog/di/worker.rb, not just the test.**
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
  - Review CI/CD requirements
- Verify compliance with:
  - Code style (per CI checks)
  - Commit message format
  - Documentation requirements
  - CHANGELOG updates
  - Version compatibility

**Note:** dd-trace-rb no longer uses RuboCop. Code style is enforced with StandardRB via CI checks.

**How to check:**
```bash
# Check for guideline files
ls -la CLAUDE.md CONTRIBUTING.md

# Check CI status for style/linting failures
gh pr checks

# CI will report any style violations
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

### 7. Method Documentation (CRITICAL)

**Requirement:** All public methods must have YARD-style docstrings with:
1. Brief but descriptive summary of what the method does
2. Parameter documentation with types
3. Return value documentation

**Principle:** Good documentation is essential for maintainability, debugging, and onboarding new team members. Methods without documentation are hard to understand and maintain.

**Review actions:**
- Check ALL new/modified public methods have docstrings
- Verify docstrings include `@param` tags for all parameters
- Verify docstrings include `@return` tag
- Ensure descriptions are meaningful (not just repeating method name)
- Check parameter types are documented
- Verify exception documentation (`@raise`) where applicable

**Required YARD tags:**
- `@param [Type] name description` - For each parameter
- `@return [Type] description` - Return value and type
- `@raise [ExceptionClass] description` - For exceptions (if raised)
- `@option` - For hash options parameters
- `@yield` - For methods that accept blocks

**How to check:**
```bash
# Find public methods without docstrings in changed files
grep -B 1 "^\s*def " lib/datadog/di/ lib/datadog/symbol_database/ | \
  grep -v "^\s*#" | grep "def "

# Check for methods missing @param tags
# Look for methods with parameters but no @param documentation
```

**Good Examples:**

**Example 1: Simple method with parameters**
```ruby
# Reverses the contents of a string.
#
# @param [String] text the text to reverse
# @return [String] the reversed text
def reverse_text(text)
  text.reverse
end
```

**Example 2: Method with multiple parameters**
```ruby
# Downloads and saves a web page to disk.
#
# @param [String] url the URL of the page to download
# @param [String] directory the directory to save to
# @return [String] the path to the saved file
def load_page(url, directory: 'pages')
  # method body
end
```

**Example 3: Method with hash options**
```ruby
# Extracts method parameters from a Ruby method definition.
#
# @param [String] method_name the name of the method to inspect
# @param [Hash] options additional configuration
# @option options [Boolean] :include_defaults (true) include parameter defaults
# @option options [Boolean] :type_info (false) include parameter type information
# @return [Array<Hash>] array of parameter information
# @raise [NameError] if the method does not exist
def extract_parameters(method_name, options = {})
  # method body
end
```

**Example 4: Method with block parameter**
```ruby
# Iterates over items and yields each to the block.
#
# @param [Array] items the collection to iterate over
# @yield [item] passes each item to the block
# @yieldparam [Object] item the current item from the collection
# @return [Array] items for which the block returned true
def filter_items(items)
  items.select { |item| yield(item) }
end
```

**Example 5: Method with exception documentation**
```ruby
# Loads configuration from a file.
#
# @param [String] file_path the path to the configuration file
# @return [Hash] the parsed configuration
# @raise [FileNotFoundError] if the file does not exist
# @raise [JSON::ParserError] if the file contains invalid JSON
def load_config(file_path)
  # method body
end
```

**BAD patterns to flag:**

```ruby
# BAD - No docstring at all
def important_method(param1, param2)
  # ...
end

# BAD - Missing parameter documentation
# Does something with data.
def process_data(data, options = {})
  # ...
end

# BAD - Missing return documentation
# Processes the input.
# @param [String] input the input to process
def process_input(input)
  # ...
end

# BAD - Vague/unhelpful description
# Handles the thing.
# @param [Object] thing the thing
# @return [Object] result
def handle_thing(thing)
  # ...
end

# BAD - Just repeating method name
# Gets the value.
# @param [Symbol] key the key
# @return [Object] the value
def get_value(key)
  # ...
end
```

**What makes a GOOD docstring:**

✅ **Clear and specific:**
```ruby
# Extracts parameter names and types from a method definition using reflection.
```

❌ **Vague or generic:**
```ruby
# Handles parameters.
```

✅ **Describes behavior and purpose:**
```ruby
# Serializes local variables from a binding context, filtering out
# internal variables and applying redaction rules for sensitive data.
```

❌ **Just restates the obvious:**
```ruby
# Serializes locals.
```

✅ **Complete documentation:**
```ruby
# Installs a line probe at the specified location.
#
# @param [String] file_path the absolute path to the target file
# @param [Integer] line_number the line number where the probe should be installed (1-indexed)
# @param [Hash] options probe configuration options
# @option options [Boolean] :capture_locals (true) whether to capture local variables
# @option options [String] :condition optional condition expression
# @return [Probe] the installed probe instance
# @raise [ProbeError] if probe installation fails
# @raise [ArgumentError] if line_number is invalid
def install_line_probe(file_path, line_number, options = {})
  # method body
end
```

**Common types in dd-trace-rb:**

| Ruby Type | YARD Syntax | Example |
|-----------|-------------|---------|
| String | `[String]` | `@param [String] name` |
| Integer | `[Integer]` | `@param [Integer] count` |
| Boolean | `[Boolean]` | `@param [Boolean] enabled` |
| Hash | `[Hash]` | `@param [Hash] options` |
| Array | `[Array<Type>]` | `@param [Array<String>] names` |
| Symbol | `[Symbol]` | `@param [Symbol] key` |
| Nil | `[nil]` | `@return [nil]` |
| Multiple types | `[Type1, Type2]` | `[String, Symbol]` |
| Any object | `[Object]` | `@param [Object] data` |
| Duck typing | `[#method_name]` | `[#to_s]` |

**Private methods:**

Private methods should also have docstrings, but they can be briefer:

```ruby
private

# Validates probe configuration before installation.
#
# @param [Hash] config the probe configuration
# @return [Boolean] true if valid
# @raise [ConfigError] if configuration is invalid
def validate_config(config)
  # method body
end
```

**Example feedback:**

```
❌ CRITICAL: Missing method documentation

**File:** lib/datadog/di/probe_manager.rb:145

**Code:**
```ruby
def register_probe(probe_id, file_path, line_number, options = {})
  # method body
end
```

**Issue:** Public method lacks docstring with parameter and return documentation.

**Required fix:**
```ruby
# Registers a new probe for dynamic instrumentation.
#
# Adds the probe to the internal registry and installs the TracePoint
# hook at the specified location.
#
# @param [String] probe_id unique identifier for the probe
# @param [String] file_path absolute path to the target source file
# @param [Integer] line_number line number where probe should be installed (1-indexed)
# @param [Hash] options probe configuration options
# @option options [Boolean] :capture_locals (true) capture local variables at probe location
# @option options [String] :condition optional conditional expression
# @return [Probe] the registered probe instance
# @raise [ProbeError] if probe is already registered or installation fails
# @raise [ArgumentError] if file_path or line_number is invalid
def register_probe(probe_id, file_path, line_number, options = {})
  # method body
end
```

**Why this matters:**
- Other developers need to understand what this method does
- Parameters need clear type and purpose documentation
- Return value and exceptions must be documented
- Hash options should use `@option` tags for each key
```

**Exceptions to docstring requirement:**

These methods do NOT require docstrings:
- `initialize` - Constructor is self-documenting (but complex ones should still have docs)
- One-line trivial accessors: `attr_reader :name`, `attr_accessor :value`
- Obvious delegators: `def foo(...) @delegate.foo(...) end`
- Override methods where parent has documentation (but use `@see ParentClass#method`)

**For inherited/overridden methods:**
```ruby
# @see BaseClass#process for parameter and return documentation
def process(data)
  # specific implementation
end
```

**References:**
- [YARD Documentation Guide](https://yardoc.org/)
- [YARD Tags Overview](https://rubydoc.info/gems/yard/file/docs/Tags.md)
- [YARD Getting Started](https://rubydoc.info/gems/yard/file/docs/GettingStarted.md)

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

## Defaulted Positional Arguments

**Overview:** Methods with defaulted positional arguments should use keyword arguments instead. The codebase strongly prefers keyword arguments for clarity, flexibility, and maintainability.

**Critical Requirement:** When reviewing method signatures with default values, challenge why they are positional arguments instead of keyword arguments.

### Why Keyword Arguments Are Preferred

**Clarity at call sites:**
```ruby
# ❌ BAD - What do these values mean?
processor.run(data, 30, 3, true)

# ✅ GOOD - Clear and self-documenting
processor.run(data, timeout: 30, retries: 3, verbose: true)
```

**Flexibility:**
- Keyword arguments can be passed in any order
- New parameters can be added without breaking existing calls
- Optional parameters are self-documenting at call sites

**Maintainability:**
- Adding new parameters doesn't require updating all call sites
- Parameter meaning is clear without reading method definition
- Refactoring is safer (can reorder, add, or remove parameters)

### What to Flag

**❌ Flag these patterns:**
```ruby
# Positional arguments with defaults
def process(data, timeout = 30, retries = 3)
  # ...
end

# Mixed positional and defaulted positional
def configure(name, path, enabled = true, verbose = false)
  # ...
end

# Optional positional arguments
def send_telemetry(event, error = nil, probe_id = nil)
  # ...
end
```

**✅ Prefer these patterns:**
```ruby
# Required positional, optional keyword
def process(data, timeout: 30, retries: 3)
  # ...
end

# All keyword arguments
def configure(name:, path:, enabled: true, verbose: false)
  # ...
end

# Required and optional keyword arguments
def send_telemetry(event:, error: nil, probe_id: nil)
  # ...
end
```

### When Positional Arguments Are Acceptable

**✅ Acceptable:**
- Required positional arguments (no defaults)
- Single-argument methods
- Methods that take a block as the primary argument

```ruby
# OK - Required positional, no defaults
def process(data, options)
  # ...
end

# OK - Single argument
def validate(input)
  # ...
end

# OK - Block is primary argument
def with_instrumentation(&block)
  # ...
end
```

**❌ Not acceptable:**
- Positional arguments with default values
- Multiple optional positional arguments
- Positional arguments after keyword arguments

### How to Check

Search for method definitions with default values:

```bash
# Find methods with defaulted positional arguments
grep -rn "def .*=.*)" lib/datadog/di/ lib/datadog/symbol_database/

# Look for patterns like:
# - def method(arg, opt = value)
# - def method(arg1, arg2 = value, arg3 = value)
# - def method(arg = value)
```

### Review Questions to Ask

When you see a method with defaulted positional arguments:

1. **Why are these positional instead of keyword arguments?**
   - Is there a specific reason they need to be positional?
   - Would keyword arguments be clearer?

2. **How is this method called?**
   - Are call sites clear about what each argument means?
   - Would keyword arguments make call sites more readable?

3. **Could this method evolve?**
   - Might more parameters be added in the future?
   - Would keyword arguments make the method more maintainable?

4. **Is this a public API?**
   - Public methods especially should use keyword arguments
   - Breaking changes are harder with positional arguments

### Example Review Comment

```markdown
This method uses defaulted positional arguments. The codebase prefers keyword arguments for clarity and maintainability.

Current:
def process_probe(probe, timeout = 5, max_depth = 3)

Suggested:
def process_probe(probe, timeout: 5, max_depth: 3)

This makes call sites more readable:
- Before: process_probe(probe, 10, 5)
- After: process_probe(probe, timeout: 10, max_depth: 5)

Is there a reason these need to be positional arguments?
```

### Exceptions

**Rare cases where positional defaults might be acceptable:**
- Internal/private methods with very stable signatures
- Methods matching external API signatures (compatibility)
- Performance-critical code (keyword arguments have minor overhead)

**Even in these cases, consider:**
- Is the clarity trade-off worth it?
- Could you use keyword arguments anyway?
- Will future maintainers understand the choice?

## Hardcoded /tmp Paths

**Overview:** Code must not use hardcoded paths in `/tmp`. These paths can lead to file collisions, security issues, and test flakiness in parallel execution environments.

**Critical Requirement:** Any code that needs temporary files or directories must use `tmpdir` or similar functions to create isolated temporary locations.

**⚠️ ANTI-RATIONALIZATION WARNING:**

This rule is absolute. You will be tempted to rationalize violations as "different" or "special":
- ❌ "This is just a test assertion, not actually using /tmp"
- ❌ "This is testing filter logic, so it's an exception"
- ❌ "This is just a string literal, not a real path"
- ❌ "This is documentation, so it doesn't count"
- ❌ "This is mocking/stubbing, not real file operations"

**All of these are still violations.** The rule exists to prevent exactly this rationalization.

**If you find yourself thinking "but this is different because...", STOP. Flag it as a violation.**

### Why Hardcoded /tmp Paths Are Problematic

**Security issues:**
- Multiple processes writing to the same path creates race conditions
- Predictable paths are security vulnerabilities (symlink attacks, file overwrites)
- File permissions may not be set correctly

**Reliability issues:**
- Tests running in parallel can collide on the same path
- Previous test runs may leave files that affect subsequent tests
- CI environments may not have write access to specific /tmp paths
- Different systems may have different /tmp structures

**Examples:**

```ruby
# ❌ BAD - Hardcoded /tmp path
File.write('/tmp/snapshot.json', data)
FileUtils.mkdir_p('/tmp/di_probes')
file = File.open('/tmp/probe_cache', 'w')

# ✅ GOOD - Use tmpdir
require 'tmpdir'
Dir.mktmpdir('di_snapshot') do |dir|
  File.write(File.join(dir, 'snapshot.json'), data)
end

# ✅ GOOD - Tempfile for single files
require 'tempfile'
Tempfile.create('probe_cache') do |file|
  file.write(data)
  file.flush
  # Use file
end
```

### How to Check for Hardcoded /tmp Paths

**Search for ALL occurrences - do not skip any:**

```bash
# Find ALL hardcoded /tmp paths (most comprehensive)
grep -rn "'/tmp/\|\"\/tmp/" lib/ spec/

# This catches ALL of:
# - File operations: File.write('/tmp/foo')
# - Test assertions: expect(x).to eq('/tmp/foo')
# - Mocks/stubs: allow(File).to receive(:open).with('/tmp/foo')
# - String literals: path = '/tmp/cache'
# - Constants: TEMP_DIR = '/tmp/data'
# - Documentation: "Save to /tmp/output.json"

# Find specific operations (use these for double-checking)
grep -rn "File\.\(write\|read\|open\).*['\"]\/tmp\/" lib/ spec/
grep -rn "FileUtils\..*['\"]\/tmp\/" lib/ spec/
grep -rn "Dir\..*['\"]\/tmp\/" lib/ spec/

# Find in test assertions (commonly rationalized away)
grep -rn "expect.*['\"]\/tmp\/" spec/
grep -rn "allow.*['\"]\/tmp\/" spec/
grep -rn "stub.*['\"]\/tmp\/" spec/

# After finding violations, exclude only Dir.tmpdir calls
grep -rn "'/tmp/\|\"\/tmp/" lib/ spec/ | grep -v "Dir\.tmpdir"
```

**Common patterns to flag (ALL are violations):**
- `'/tmp/anything'` or `"/tmp/anything"` - Any string literal
- `File.write('/tmp/...')` - File operations
- `File.open('/tmp/...')` - File operations
- `FileUtils.mkdir_p('/tmp/...')` - Directory operations
- `Dir.chdir('/tmp/...')` - Directory operations
- `expect(result).to eq('/tmp/...')` - **Test assertions (not exempt!)**
- `allow(File).to receive(:open).with('/tmp/...')` - **Mocks/stubs (not exempt!)**
- `TEMP_PATH = '/tmp/...'` - **Constants (not exempt!)**
- `path = '/tmp/...'` - **Variables (not exempt!)**
- Any string literal containing `/tmp/` in ANY context

### Correct Patterns

**For temporary directories:**

```ruby
require 'tmpdir'

# Automatically cleaned up when block exits
Dir.mktmpdir('di_probes') do |dir|
  probe_file = File.join(dir, 'probe.json')
  File.write(probe_file, data)
  # Use probe_file
end

# Or without block (manual cleanup)
dir = Dir.mktmpdir('di_probes')
begin
  # Use dir
ensure
  FileUtils.remove_entry(dir)
end
```

**For temporary files:**

```ruby
require 'tempfile'

# Single file, automatically cleaned up
Tempfile.create('snapshot') do |file|
  file.write(data)
  file.flush
  file.rewind
  # Use file
end

# Multiple files in same directory
Dir.mktmpdir('di_test') do |dir|
  file1 = File.join(dir, 'config.json')
  file2 = File.join(dir, 'output.json')
  File.write(file1, config)
  File.write(file2, output)
end
```

**In tests:**

```ruby
# RSpec - use around hook for temp directory
around do |example|
  Dir.mktmpdir('di_test') do |dir|
    @tmpdir = dir
    example.run
  end
end

it 'processes snapshot' do
  snapshot_file = File.join(@tmpdir, 'snapshot.json')
  File.write(snapshot_file, data)
  # Test code
end
```

### Verification Checklist

When hardcoded /tmp paths are found:

- [ ] **Identify all hardcoded /tmp paths** in the PR
  ```bash
  gh pr diff <PR_NUMBER> | grep -E "'/tmp/|\"\/tmp\/"
  ```

- [ ] **Verify each usage is replaced with tmpdir/tempfile**
  - Check that `require 'tmpdir'` or `require 'tempfile'` is added
  - Confirm `Dir.mktmpdir` or `Tempfile.create` is used
  - Verify cleanup (block form or ensure block)

- [ ] **Check for proper cleanup:**
  - Block form (automatic): `Dir.mktmpdir('prefix') do |dir|`
  - Manual form with ensure: `ensure FileUtils.remove_entry(dir)`
  - Tempfile automatic cleanup: `Tempfile.create('name') do |f|`

- [ ] **Verify tests pass with isolated temp directories:**
  - Each test gets its own temp directory
  - No shared state between tests
  - Parallel test execution safe

### Exceptions

**CRITICAL: Do not rationalize your way around this rule. If you find yourself thinking "but this is different because...", you are probably wrong.**

**The ONLY acceptable /tmp references are:**

1. **The exact string `Dir.tmpdir`** (reading system temp directory):
   ```ruby
   Dir.tmpdir  # Returns system temp directory path
   ```

2. **Comments explicitly warning against /tmp** (must include "don't", "avoid", "not", or "instead"):
   ```ruby
   # Don't use /tmp directly, use Dir.mktmpdir instead
   # Avoid hardcoded /tmp paths - use Tempfile.create instead
   ```

   Note: This is ONLY for warnings/guidance, not for examples or explanations of behavior.

**Everything else is a violation. No exceptions. Including:**

❌ **Test assertions with /tmp strings** - Still a violation:
```ruby
# VIOLATION - even though it's "just testing"
expect(result).to eq('/tmp/foo')
expect(path).to include('/tmp/')
allow(File).to receive(:write).with('/tmp/test.txt', anything)
```

❌ **"Testing tmpdir functionality"** - Still a violation unless using `Dir.tmpdir`:
```ruby
# VIOLATION - testing filter logic doesn't exempt you
it 'filters tmp paths' do
  expect(filter('/tmp/test')).to be_filtered  # WRONG
  expect(filter(File.join(Dir.tmpdir, 'test'))).to be_filtered  # CORRECT
end
```

❌ **String literals in any context** - Still a violation:
```ruby
# VIOLATION - doesn't matter that it's "just a string"
EXCLUDED_PATHS = ['/tmp', '/var/tmp']  # WRONG
EXCLUDED_PATHS = [Dir.tmpdir]  # CORRECT
```

**Never acceptable:**
- Writing to `/tmp/myapp/...`
- Reading from `/tmp/cache/...`
- Creating directories in `/tmp/specific_name/`
- Any hardcoded path under `/tmp`
- Test assertions checking for `/tmp/` paths
- Test mocks stubbing `/tmp/` paths
- String literals containing `/tmp/` paths
- Constants or variables assigned `/tmp/` strings
- **ANY OTHER USE YOU CAN THINK OF**

**If you're unsure whether something is an exception, it's not. Flag it.**

### Feedback Template

When hardcoded /tmp paths are found:

```markdown
❌ CRITICAL: Hardcoded /tmp paths found

**Files with hardcoded /tmp paths:**

1. **lib/datadog/di/snapshot_exporter.rb:45**
   ```ruby
   File.write('/tmp/snapshot.json', data)
   ```

   Replace with:
   ```ruby
   require 'tempfile'
   Tempfile.create('snapshot.json') do |file|
     file.write(data)
     file.flush
     # Use file.path
   end
   ```

2. **spec/datadog/di/probe_spec.rb:123**
   ```ruby
   FileUtils.mkdir_p('/tmp/di_test')
   ```

   Replace with:
   ```ruby
   require 'tmpdir'
   Dir.mktmpdir('di_test') do |dir|
     # Use dir
   end
   ```

**Why this matters:**
- Hardcoded /tmp paths cause test collisions in parallel execution
- Security vulnerability (predictable paths, symlink attacks)
- Tests may fail in CI environments with restricted /tmp access
- Previous test runs can leave files affecting subsequent tests

**Required action:**
- Replace all hardcoded /tmp paths with `Dir.mktmpdir` or `Tempfile.create`
- Ensure proper cleanup (use block form or ensure blocks)
- Verify tests pass after changes
- Run tests in parallel to confirm no collisions: `bundle exec rspec --order random`
```

## Debugging Diagnostics Left in PR

**Overview:** Debugging statements and diagnostic code must be removed before merging. These create noise in logs, can leak internal information, and indicate incomplete cleanup.

**Critical Requirement:** All debugging code must be removed from production code and tests.

### Common Debugging Patterns to Flag

**Ruby debugging statements:**
- `puts` / `p` / `pp` / `print` - Console output for debugging
- `Kernel.puts` / `Kernel.p` / `Kernel.print` - Explicit kernel methods
- `STDERR.puts` / `STDOUT.puts` - Direct stream writes
- `warn` - Warnings used for debugging (not actual warnings)

**Interactive debuggers:**
- `binding.pry` - Pry debugger breakpoint
- `binding.irb` - IRB debugger breakpoint
- `debugger` - Generic debugger statement
- `byebug` - Byebug debugger breakpoint

**Diagnostic comments:**
- `# DEBUG:` / `# DIAGNOSTIC:` - Debug markers in comments
- `# TODO: remove` - Temporary code markers
- `# FIXME: debug` - Debug-related fixme notes
- `# XXX` - Placeholder markers

### How to Check for Debugging Code

```bash
# Search for debugging statements in lib/ (production code)
grep -rn "^\s*puts\s\|^\s*p\s\|^\s*pp\s\|^\s*print\s" lib/datadog/di/ lib/datadog/symbol_database/
grep -rn "Kernel\.\(puts\|p\|print\)" lib/datadog/di/ lib/datadog/symbol_database/
grep -rn "STDERR\.puts\|STDOUT\.puts" lib/datadog/di/ lib/datadog/symbol_database/
grep -rn "^\s*warn\s" lib/datadog/di/ lib/datadog/symbol_database/

# Search for debugger breakpoints
grep -rn "binding\.pry\|binding\.irb\|debugger\|byebug" lib/datadog/di/ lib/datadog/symbol_database/ spec/

# Search for diagnostic comments
grep -rn "# DEBUG:\|# DIAGNOSTIC:\|# TODO: remove\|# FIXME: debug\|# XXX" lib/datadog/di/ lib/datadog/symbol_database/

# Search the PR diff specifically
gh pr diff <PR_NUMBER> | grep -E "^\+.*\b(puts|warn|binding\.pry|# DEBUG:|# DIAGNOSTIC:)"
```

### Acceptable vs Unacceptable

**❌ NEVER acceptable in production code (lib/):**

```ruby
# BAD - Debugging output
def extract_method_parameters(method_name)
  warn "[SymDB] extract_method_parameters: method=#{method_name} params=#{params.inspect}"
  # ...
end

# BAD - Console debugging
def process_probe(probe)
  puts "Processing probe: #{probe.id}"
  p probe
  # ...
end

# BAD - Debugger breakpoint
def execute
  binding.pry if probe.id == '123'  # Left over from debugging
  # ...
end

# BAD - Diagnostic comment
def serialize
  # DEBUG: This is failing intermittently
  # DIAGNOSTIC: stderr logging for CI debugging
  serialize_data
end
```

**✅ Acceptable in production code:**

```ruby
# GOOD - Proper logger usage at appropriate level
def extract_method_parameters(method_name)
  Datadog.logger.debug { "Extracting parameters for #{method_name}" }
  # ...
end

# GOOD - Telemetry for observability
def process_probe(probe)
  Datadog.telemetry.count('di.probe.processed', 1, tags: ["probe_id:#{probe.id}"])
  # ...
end

# GOOD - Appropriate warning for customer-fixable issues
def load_config(file)
  unless File.exist?(file)
    Datadog.logger.warn("Configuration file not found: #{file}")
  end
end
```

**⚠️ May be acceptable in tests (requires judgment):**

```ruby
# Generally BAD in tests - but less critical than in production
it 'processes probe' do
  puts "Debug: probe state = #{probe.inspect}"  # Should be removed
  expect(probe.execute).to be_truthy
end

# ACCEPTABLE in tests - debugging test infrastructure itself
# (But should have a clear comment explaining why it's needed)
around do |example|
  # Output test name for debugging CI failures
  puts "Running: #{example.description}" if ENV['DEBUG_TESTS']
  example.run
end
```

### Exceptions

**The only acceptable "debugging" output is:**

1. **Proper Datadog logger usage:**
   ```ruby
   Datadog.logger.debug { "..." }  # ✅ OK
   Datadog.logger.warn("...")      # ✅ OK (for customer-fixable issues)
   ```

2. **Telemetry for observability:**
   ```ruby
   Datadog.telemetry.error('event', error: e)  # ✅ OK
   Datadog.metrics.increment('counter')        # ✅ OK
   ```

3. **Test infrastructure with clear justification:**
   ```ruby
   # Output for debugging CI-only failures (with ENV guard)
   puts "CI Debug: #{state}" if ENV['CI_DEBUG']  # ✅ OK with clear comment
   ```

4. **Documentation showing what NOT to do:**
   ```ruby
   # Don't debug with puts, use Datadog.logger.debug instead
   # BAD: puts variable.inspect
   # GOOD: Datadog.logger.debug { variable.inspect }
   ```

### Example from Real PR

**From your example:**

```ruby
# DIAGNOSTIC: stderr logging for CI debugging
warn "[SymDB] extract_method_parameters: method=#{method_name} params=#{params.inspect}"
```

**Issues:**
1. Using `warn` for debugging instead of proper logger
2. Comment indicates this is temporary diagnostic code
3. Should use `Datadog.logger.debug` if logging is needed
4. If truly debugging CI issues, should be removed after debugging is complete

**Fix:**

```ruby
# Remove entirely if no longer needed, or replace with proper logging:
Datadog.logger.debug { "[SymDB] Extracting parameters for #{method_name}" }
```

### Feedback Template

```markdown
❌ Debugging diagnostics left in PR

**Files with debugging code:**

1. **lib/datadog/symbol_database/code_tracking.rb:245**
   ```ruby
   # DIAGNOSTIC: stderr logging for CI debugging
   warn "[SymDB] extract_method_parameters: method=#{method_name} params=#{params.inspect}"
   ```

   **Issue:** Temporary debugging statement left in production code.

   **Fix:** Remove entirely or replace with proper logger:
   ```ruby
   Datadog.logger.debug { "[SymDB] Extracting parameters for #{method_name}" }
   ```

2. **lib/datadog/di/probe_manager.rb:156**
   ```ruby
   puts "DEBUG: probe state = #{probe.inspect}"
   ```

   **Issue:** Console debugging output left in production code.

   **Fix:** Remove this line.

3. **spec/datadog/di/worker_spec.rb:89**
   ```ruby
   binding.pry  # TODO: remove
   ```

   **Issue:** Debugger breakpoint left in test.

   **Fix:** Remove this line.

**Why this matters:**
- Debugging statements create noise in production logs
- Can leak internal implementation details
- Indicates incomplete cleanup
- May have performance impact (inspect, string interpolation)
- `warn` goes to STDERR and can trigger monitoring alerts

**Required action:**
- Remove all debugging statements from lib/ (production code)
- Remove debugger breakpoints from tests
- Replace with proper `Datadog.logger.debug` if logging is actually needed
- Remove diagnostic comments (# DEBUG:, # DIAGNOSTIC:, # TODO: remove)
```

## Useless Comments

**Overview:** Code comments should add value by explaining WHY something is done, not WHAT is being done. Comments that merely restate the code are noise and must be removed.

**Critical Requirement:** All comments in the PR must provide meaningful information that isn't obvious from reading the code itself.

### Systematic Comment Review

**CRITICAL:** You MUST systematically review ALL comments added in the PR diff, not just sample a few files.

**How to find ALL new comments in the PR:**

```bash
# Find ALL new comments (lines starting with + and containing #)
# Exclude frozen_string_literal magic comments
gh pr diff <PR_NUMBER> | grep -n "^\+.*#" | grep -v "frozen_string_literal"

# Get count of new comments
gh pr diff <PR_NUMBER> | grep "^\+.*#" | grep -v "frozen_string_literal" | wc -l

# Save to file for systematic review
gh pr diff <PR_NUMBER> | grep -n "^\+.*#" | grep -v "frozen_string_literal" > /tmp/pr_comments.txt
cat /tmp/pr_comments.txt
```

**This command finds:**
- All lines added in the diff (starting with `+`)
- That contain a `#` character (comments)
- Excluding magic comments like `# frozen_string_literal: true`

**Common mistake to avoid:**
- ❌ Reviewing only one or two files
- ❌ Spot-checking a few obvious comments
- ❌ Assuming some files won't have useless comments
- ✅ Systematically reviewing EVERY comment in the PR diff

### What Makes a Comment Useless

**Useless comments:**
- Restate what the code obviously does
- Document the WHAT instead of the WHY
- Describe implementation details already clear from code
- Narrate the code flow step-by-step
- Were added during development as "thought process notes"
- State the obvious

**Useful comments:**
- Explain WHY a decision was made
- Document non-obvious constraints or requirements
- Reference external documentation or tickets
- Explain workarounds for known issues
- Clarify complex business logic
- Document performance considerations
- Explain security or safety requirements

### Examples of Useless Comments (DEMAND REMOVAL)

**❌ Example 1: Restating what code does**
```ruby
# BAD - Comment just restates the code
# Create shared probe repository first - no dependencies
probe_repository = Datadog::DI::ProbeRepository.new

# REMOVE - The code is self-documenting
probe_repository = Datadog::DI::ProbeRepository.new
```

**❌ Example 2: Obvious implementation narration**
```ruby
# BAD - Narrating obvious code flow
# Initialize the configuration
config = Config.new

# Set the timeout value
config.timeout = 30

# Start the worker
worker.start

# REMOVE ALL - Code is obvious, comments add no value
config = Config.new
config.timeout = 30
worker.start
```

**❌ Example 3: Documenting variable assignments**
```ruby
# BAD - Obvious from variable name
# Store the probe ID
probe_id = probe.id

# Get the current timestamp
timestamp = Time.now

# REMOVE - Variable names are clear
probe_id = probe.id
timestamp = Time.now
```

**❌ Example 4: Step-by-step code narration**
```ruby
# BAD - Narrating implementation steps
def process_snapshot
  # Extract the data
  data = snapshot.data

  # Filter out sensitive fields
  filtered = filter_sensitive(data)

  # Serialize to JSON
  json = JSON.dump(filtered)

  # Return the result
  json
end

# REMOVE ALL - Method name and code are clear
def process_snapshot
  data = snapshot.data
  filtered = filter_sensitive(data)
  JSON.dump(filtered)
end
```

**❌ Example 5: Restating control flow**
```ruby
# BAD - Control flow is obvious
# If probe is enabled, execute it
if probe.enabled?
  probe.execute
end

# Check if we have capacity
if buffer.full?
  # Return false if full
  return false
end

# REMOVE - Control flow is self-evident
if probe.enabled?
  probe.execute
end

return false if buffer.full?
```

**❌ Example 6: Method summary that adds nothing**
```ruby
# BAD - Just restating method name
# Process the probe
def process_probe(probe)
  # ...
end

# BAD - Obvious from parameters
# Initialize with settings
def initialize(settings)
  @settings = settings
end

# REMOVE - Use YARD docs if documentation needed
# See Section 7 for proper method documentation
```

### Examples of Useful Comments (KEEP)

**✅ Example 1: Explaining WHY, not WHAT**
```ruby
# GOOD - Explains constraint/reasoning
# ProbeRepository must be created before other components because
# ProbeManager and CodeTracker both depend on it for probe lookups
probe_repository = Datadog::DI::ProbeRepository.new
```

**✅ Example 2: Non-obvious constraints**
```ruby
# GOOD - Documents important constraint
# Must disable TracePoint in ensure block to prevent memory leaks
# from binding references held by the TracePoint object
ensure
  @trace.disable if @trace
end
```

**✅ Example 3: Workarounds or gotchas**
```ruby
# GOOD - Explains workaround
# Ruby 2.3 doesn't support #then, use #tap instead
result.tap { |r| process(r) }

# GOOD - Documents known issue
# TODO(#12345): Remove this after upgrading to Ruby 3.0
# Ruby 2.x requires explicit encoding for special characters
string.force_encoding('UTF-8')
```

**✅ Example 4: Business logic explanation**
```ruby
# GOOD - Explains business rule
# Snapshots are limited to 1000 variables to prevent memory exhaustion
# in customer applications. This limit is configurable via settings.
return variables.first(1000)
```

**✅ Example 5: Performance considerations**
```ruby
# GOOD - Explains performance tradeoff
# Using Hash lookup instead of Array#find for O(1) vs O(n) performance
# when matching probes. Critical for applications with 100+ probes.
@probe_cache[key] = probe
```

**✅ Example 6: Security or safety requirements**
```ruby
# GOOD - Explains safety requirement
# Never propagate exceptions to customer code - all instrumentation
# failures must be contained. See Section 5 for error boundary requirements.
rescue => e
  log_error(e)
end
```

### How to Evaluate Comments

**Ask these questions for EACH comment:**

1. **Can I understand the code without this comment?**
   - Yes → Comment is probably useless
   - No → Comment may be useful

2. **Does the comment explain WHY instead of WHAT?**
   - WHY → Probably useful
   - WHAT → Probably useless

3. **Would removing this comment make the code harder to understand?**
   - No → Comment is useless, remove it
   - Yes → Comment is useful, keep it

4. **Does the comment add information not obvious from:**
   - Method/variable names?
   - Code structure?
   - Standard Ruby idioms?
   - If it adds nothing → Useless

5. **Is this a "thought process" comment from development?**
   - Yes → Definitely useless, remove it
   - No → Evaluate further

### Common Patterns of Useless Comments

**Pattern 1: Implementation narration**
```ruby
# Create the object
obj = Object.new
```

**Pattern 2: Variable description**
```ruby
# The user's name
name = user.name
```

**Pattern 3: Control flow description**
```ruby
# Loop through items
items.each do |item|
```

**Pattern 4: Method call description**
```ruby
# Call the process method
process(data)
```

**Pattern 5: Assignment description**
```ruby
# Set the value
@value = 42
```

**All of these should be REMOVED.**

### Systematic Review Process

**Step 1: Extract ALL comments from PR diff**
```bash
gh pr diff <PR_NUMBER> | grep -n "^\+.*#" | grep -v "frozen_string_literal" > /tmp/all_comments.txt
```

**Step 2: Review each comment individually**
```bash
# Count total comments to review
total=$(wc -l < /tmp/all_comments.txt)
echo "Total comments to review: $total"

# Review each one
cat /tmp/all_comments.txt
```

**Step 3: Identify useless comments**
For each comment, ask:
- Does it restate what code does?
- Is it obvious from reading the code?
- Would removing it reduce clarity? (No = useless)

**Step 4: Demand removal of ALL useless comments**
List every single useless comment with file and line number.

### Feedback Template

When useless comments are found:

```markdown
❌ Useless comments found in PR

**Overview:** Found X comments that restate what the code does instead of explaining WHY. These must be removed.

**Systematic review:**
- Total comments added in PR: Y
- Useless comments found: X
- Useful comments: (Y - X)

**Comments to remove:**

1. **lib/datadog/di/component.rb:89**
   ```ruby
   # Create shared probe repository first - no dependencies
   probe_repository = Datadog::DI::ProbeRepository.new
   ```

   **Issue:** Comment just restates what the code does.

   **Why useless:** The code is self-documenting - it's clearly creating a ProbeRepository.

   **Action:** Remove the comment entirely.

2. **lib/datadog/di/component.rb:95**
   ```ruby
   # Initialize the probe manager
   probe_manager = Datadog::DI::ProbeManager.new(settings, probe_repository, code_tracker)
   ```

   **Issue:** Comment narrates obvious initialization.

   **Why useless:** Method name and parameters make this clear.

   **Action:** Remove the comment.

3. **lib/datadog/di/probe_manager.rb:145**
   ```ruby
   # Set the probe state to active
   probe.state = :active
   ```

   **Issue:** Comment restates the assignment.

   **Why useless:** The code `probe.state = :active` is self-explanatory.

   **Action:** Remove the comment.

**What makes a comment useful:**
- Explains WHY a decision was made (not WHAT is being done)
- Documents non-obvious constraints or requirements
- References tickets, issues, or external documentation
- Explains workarounds or gotchas
- Clarifies complex business logic

**Example of converting useless → useful:**

Before (useless):
```ruby
# Create shared probe repository first - no dependencies
probe_repository = Datadog::DI::ProbeRepository.new
```

After (if comment is actually needed):
```ruby
# ProbeRepository must be initialized before ProbeManager and CodeTracker
# because both depend on it for probe lookups. Creating it first avoids
# circular dependency issues.
probe_repository = Datadog::DI::ProbeRepository.new
```

Or better yet (if comment isn't needed):
```ruby
# No comment - code is self-documenting
probe_repository = Datadog::DI::ProbeRepository.new
```

**Required action:**
- Remove ALL useless comments listed above
- Use meaningful method/variable names instead of comments
- Only keep comments that explain WHY, not WHAT
- Use YARD docstrings for method documentation (see Section 7)

**Systematic verification:**
After removal, verify no useless comments remain:
```bash
gh pr diff <PR_NUMBER> | grep -n "^\+.*#" | grep -v "frozen_string_literal"
# Review each remaining comment for utility
```
```

### How to Check for Useless Comments

**Required commands:**

```bash
# Step 1: Find ALL comments in the PR diff
gh pr diff <PR_NUMBER> | grep -n "^\+.*#" | grep -v "frozen_string_literal" > /tmp/comments.txt

# Step 2: Count them
echo "Total comments to review: $(wc -l < /tmp/comments.txt)"

# Step 3: Review each one systematically
cat /tmp/comments.txt

# Step 4: For each file with comments, read the context
# Don't just look at grep output - read the actual file to see if comment adds value

# Step 5: Identify useless comments
# Ask: Does this explain WHY or just restate WHAT?

# Step 6: List ALL useless comments in feedback
```

**Common sources of useless comments:**
- Added during initial implementation (thought process notes)
- Copied from other code without considering if needed
- Added "for clarity" but just restate obvious code
- Left over from refactoring (describe old approach)
- Written before code became self-documenting

### Distinction from Debugging Diagnostics

**Debugging diagnostics (separate section):**
- `# DEBUG:`, `# DIAGNOSTIC:`, `# TODO: remove`
- Temporary markers for active debugging
- Should be removed after debugging complete

**Useless comments (this section):**
- Regular code comments that add no value
- Restate what code does
- Were thought process notes during development
- Make code noisier without adding understanding

**Both should be removed, but for different reasons:**
- Debugging → Temporary, indicates incomplete cleanup
- Useless → Permanent but pointless, reduces code clarity

### Prevention

**During code writing:**
- Write self-documenting code (clear names, simple structure)
- Only add comments when code alone isn't sufficient
- Ask: "Does this comment explain WHY?"
- If explaining WHAT, improve the code instead

**During review:**
- Systematically check ALL comments
- Apply the evaluation questions
- Be strict: when in doubt, remove it
- Better to have no comment than a useless one

## Review Checklist

When reviewing a dd-trace-rb DI PR, verify:

**CRITICAL (Blocking Approval):**
- [ ] ✅ NO skipped tests (no skip, pending, xit, xdescribe)
- [ ] ✅ NO sleep in tests (use Queue, ConditionVariable, mock time)
- [ ] ✅ 100% code coverage (all lines, branches, error paths)
- [ ] ✅ Complies with repository guidelines (CLAUDE.md, CONTRIBUTING.md, CI checks)
- [ ] ✅ NO exception propagation (all instrumentation errors contained)
- [ ] ✅ Proper error handling (all rescues have DEBUG/WARN logging + telemetry)
- [ ] ✅ Method documentation (all public methods have YARD docstrings with @param and @return)

**Additional Quality Checks:**
- [ ] NO useless comments (see Useless Comments section - systematic check required)
- [ ] NO debugging diagnostics (puts, warn, binding.pry, # DEBUG:, # DIAGNOSTIC:)
- [ ] NO defaulted positional arguments (challenge: why not keyword arguments? codebase prefers keyword args)
- [ ] TracePoint callbacks have proper cleanup (tp.disable in ensure)
- [ ] No infinite recursion (instrumentation doesn't trace itself)
- [ ] Bounded memory usage (no binding leaks)
- [ ] Performance overhead documented and <5%
- [ ] Thread-safe shared state access
- [ ] Fork/fiber edge cases handled
- [ ] Line probe test files verified (see Line Probe Test Files section)
- [ ] NO hardcoded /tmp paths (use tmpdir instead)
- [ ] Documentation updated
- [ ] CHANGELOG updated if required
- [ ] Trailing commas used in di/ and symbol_database/ directories (per CLAUDE.md)

## Review Process

To review a dd-trace-rb dynamic instrumentation PR:

**⚠️ CRITICAL: GitHub API Pagination**

**You MUST use `--paginate` when fetching comments, or you will silently miss comments.**

- GitHub API returns 30 items per page by default
- Without `--paginate`, you only get the first 30 comments
- Large PRs can have 60+ comments
- **Missing comments means incomplete review - this causes silent failures**
- Always verify: Log the comment count after fetching

```bash
# WRONG - Only gets first 30 comments
gh api repos/DataDog/dd-trace-rb/pulls/<PR>/comments

# CORRECT - Gets all comments
gh api repos/DataDog/dd-trace-rb/pulls/<PR>/comments --paginate
```

1. **Fetch the PR**: Use `gh pr view <number>` or `gh pr checkout <number>`
2. **Read changed files**: Focus on new instrumentation code
3. **Review existing comments** (if any):
   - Ignore outdated comments (marked `outdated: true` in API, collapsed in UI)
   - Only consider current comments on the latest code
   - Outdated comments were already addressed by subsequent code changes
   - See "Understanding Outdated Comments" section below
4. **Check for line probe test file modifications**:
   - Identify if any line probe test files were modified
   - Verify line number changes are safe (additions at end) or tests are updated
5. **Run critical checks**:
   - **Systematically review ALL comments** in PR diff for utility (see Useless Comments section)
   - Search for skipped tests
   - Search for sleep in tests
   - Search for hardcoded /tmp paths
   - Search for defaulted positional arguments (see Defaulted Positional Arguments section)
   - Search for debugging diagnostics (puts, warn, binding.pry, # DEBUG:, # DIAGNOSTIC:)
   - Check code coverage report
   - Verify error boundaries (all TracePoint callbacks, prepended methods)
   - Check error handling (all rescue blocks have logging + telemetry)
   - Check method documentation (all public methods have YARD docstrings)
6. **Review repository compliance**: Check CLAUDE.md, CONTRIBUTING.md, CI checks
7. **Run tests**: `bundle exec rspec` with coverage
8. **Provide feedback**: Use the checklist above, flag all CRITICAL issues

## Understanding Outdated Comments

**What are outdated comments?**

Outdated comments are GitHub PR review comments that were made on previous versions of code that has since been changed or replaced.

**How comments become outdated:**
1. Reviewer comments on line 145: "Add error boundary here"
2. You push new commits that modify lines around line 145
3. GitHub marks that comment as `outdated: true` in the API
4. The comment is collapsed in the UI under "Show outdated" sections
5. The line numbers referenced may no longer exist or contain different code

**Why ignore outdated comments during review:**
- They reference code that no longer exists in the current PR
- They've usually been addressed (that's why the code changed)
- The reviewer hasn't re-commented on the new code
- Reviewing outdated comments wastes time on superseded context
- If the concern still applies, the reviewer will re-post it on current code

**How to identify them:**

**In the GitHub UI:**
- Collapsed under "Show outdated" expandable sections
- Grayed out or visually de-emphasized
- Show old commit SHA references

**Via GitHub API:**
```bash
# Get ALL current (non-outdated) comments with pagination
# CRITICAL: Must use --paginate to get all comments (GitHub returns 30 per page by default)
# Note: outdated can be false OR null for current comments

# IMPORTANT: Use file-based jq filter to avoid shell quoting issues
filter_file=$(mktemp)
cat > "$filter_file" << 'EOF'
[.[] | select(.outdated != true)]
EOF

gh api repos/DataDog/dd-trace-rb/pulls/<PR_NUMBER>/comments --paginate | \
  jq -f "$filter_file"
rm -f "$filter_file"

# See which comments are outdated
filter_file=$(mktemp)
cat > "$filter_file" << 'EOF'
[.[] | select(.outdated == true) | {path, line, body: .body[0:100]}]
EOF

gh api repos/DataDog/dd-trace-rb/pulls/<PR_NUMBER>/comments | \
  jq -f "$filter_file"
rm -f "$filter_file"
```

**When to address an "outdated" comment:**
- **Never, unless explicitly asked** - They've been superseded by code changes
- Only if reviewer re-posts it on current code: "This still applies"
- Only if reviewer explicitly references the outdated comment: "My comment from line 145 still applies to the new implementation"

**Example scenario:**

**Initial code (line 145):**
```ruby
def process_probe
  probe.execute  # No error handling
end
```

**Reviewer comments:** "Add error boundary here" (on line 145)

**You push new commit that refactors the entire method:**
```ruby
def process_probe
  begin
    probe.execute
  rescue => e
    log_error(e)
  end
end
```

**Result:**
- Original comment is now marked `outdated: true`
- The concern was addressed by your refactor
- No need to respond to the outdated comment
- Reviewer sees the new code and either approves or leaves new comments

**Summary:** When reviewing a PR, only consider current comments on the latest code. Outdated comments are historical context that's been superseded.

## Commands to Run

```bash
# Fetch PR
gh pr checkout <PR_NUMBER>

# View current (non-outdated) review comments
# CRITICAL: Must use --paginate to get all comments (GitHub returns 30 per page by default)
# Note: outdated can be false OR null for current comments

# IMPORTANT: Use file-based jq filters to avoid shell quoting issues
filter_file=$(mktemp)
cat > "$filter_file" << 'EOF'
[.[] | select(.outdated != true) | {path, line, body}]
EOF

gh api repos/DataDog/dd-trace-rb/pulls/<PR_NUMBER>/comments --paginate | \
  jq -f "$filter_file"
rm -f "$filter_file"

# Count current vs outdated comments
# CRITICAL: Must use --paginate to get all comments (GitHub returns 30 per page by default)
# Note: outdated can be false OR null for current comments

filter_current=$(mktemp)
filter_outdated=$(mktemp)

cat > "$filter_current" << 'EOF'
[.[] | select(.outdated != true)] | length
EOF

cat > "$filter_outdated" << 'EOF'
[.[] | select(.outdated == true)] | length
EOF

echo "Current comments:"
gh api repos/DataDog/dd-trace-rb/pulls/<PR_NUMBER>/comments --paginate | \
  jq -f "$filter_current"
echo "Outdated comments (ignore these):"
gh api repos/DataDog/dd-trace-rb/pulls/<PR_NUMBER>/comments --paginate | \
  jq -f "$filter_outdated"

rm -f "$filter_current" "$filter_outdated"

# CRITICAL: Systematically check ALL code comments for utility
# This finds EVERY comment added in the PR - review each one
gh pr diff <PR_NUMBER> | grep -n "^\+.*#" | grep -v "frozen_string_literal"
# Review each comment: Does it explain WHY or just restate WHAT?
# Demand removal of comments that just restate what code does

# Check for skipped tests
grep -rn "^\s*skip\|^\s*pending\|^\s*xit\|^\s*xdescribe" spec/ test/

# Check for sleep in tests
grep -rn "sleep\s\|Kernel\.sleep" spec/ test/

# Check for hardcoded /tmp paths
grep -rn "'/tmp/\|\"\/tmp\/" lib/ spec/
grep -rn "File\.\(write\|read\|open\).*['\"]\/tmp\/" lib/ spec/
grep -rn "FileUtils\..*['\"]\/tmp\/" lib/ spec/

# Check for defaulted positional arguments
grep -rn "def .*=.*)" lib/datadog/di/ lib/datadog/symbol_database/
# Review each match: Should these be keyword arguments instead?

# Check for debugging diagnostics in production code
grep -rn "^\s*puts\s\|^\s*p\s\|^\s*pp\s\|^\s*print\s" lib/datadog/di/ lib/datadog/symbol_database/
grep -rn "Kernel\.\(puts\|p\|print\)" lib/datadog/di/ lib/datadog/symbol_database/
grep -rn "STDERR\.puts\|STDOUT\.puts" lib/datadog/di/ lib/datadog/symbol_database/
grep -rn "^\s*warn\s" lib/datadog/di/ lib/datadog/symbol_database/
grep -rn "binding\.pry\|binding\.irb\|debugger\|byebug" lib/ spec/
grep -rn "# DEBUG:\|# DIAGNOSTIC:\|# TODO: remove.*debug" lib/datadog/di/ lib/datadog/symbol_database/

# Check for debugging diagnostics in the PR diff
gh pr diff <PR_NUMBER> | grep -E "^\+.*\b(puts|warn|binding\.pry|# DEBUG:|# DIAGNOSTIC:)"

# Run tests with coverage
bundle exec rspec --require simplecov

# Note: dd-trace-rb uses StandardRB (not RuboCop)
# Code style is checked by StandardRB via CI - see gh pr checks output

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

### ❌ 7. Method Documentation
[Any public methods without YARD docstrings]

## Additional Issues

### ⚠️ Debugging Diagnostics
[If any debugging code is found:]
- List all files with debugging statements
- Show the problematic lines (puts, warn, binding.pry, # DEBUG:, etc.)
- Indicate whether in production code (lib/) or tests (spec/)
- Suggest removal or replacement with proper logger

### ⚠️ Useless Comments
[If any useless comments are found:]
- Report total comments added in PR vs useless comments found
- List each useless comment with file:line
- Show the comment and explain why it's useless
- Demand removal (not rewriting - just delete them)
- Note: Comments should explain WHY, not WHAT

**Systematic check required:**
```bash
gh pr diff <PR_NUMBER> | grep -n "^\+.*#" | grep -v "frozen_string_literal"
```
Review EVERY comment found, not just a sample.

### ⚠️ Hardcoded /tmp Paths
[If any hardcoded /tmp paths are found:]
- List all files with hardcoded /tmp paths
- Show the problematic lines
- Suggest using Dir.mktmpdir or Tempfile.create instead
- Verify proper cleanup (block form or ensure block)

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

## Follow-Up Actions After Review

**IMPORTANT:** After providing the review, if you are asked to fix any issues you identified, **FIX THEM** rather than providing recommendations or comments.

### When User Says "Fix X"

When the user says things like:
- "Fix the sleep in tests"
- "Fix the skipped test"
- "Fix the linting errors"
- "Fix the Steep type errors"
- "Add the missing error boundary"
- "Fix all the issues"

**Determine which fix skill to use:**

**Use `/pr-fix-tests` for:**
- Test failures (skipped tests, failing specs, test errors)
- Sleep in tests
- Test-related issues

**Use `/pr-fix-lint` for:**
- Linting errors (StandardRB, ESLint, etc.)
- Type checking errors (Steep, Sorbet, TypeScript)
- Formatting issues (Prettier, Black)
- Static analysis issues

**Note:** dd-trace-rb uses StandardRB for code style (not RuboCop).

**Fix manually (in review skill) for:**
- Issues that don't fit the fix skills (missing error boundaries, missing telemetry, hardcoded /tmp paths)
- Mixed issues that span multiple categories
- Issues requiring review context

**Process:**
1. **Identify issue type** - Test, lint, or manual fix
2. **Delegate to appropriate fix skill** - Use `/pr-fix-tests` or `/pr-fix-lint` with PR number
3. **Or fix manually** - For issues not covered by fix skills

**Example delegations:**
- "Fix the sleep in tests" → `/pr-fix-tests <PR_NUMBER>`
- "Fix the linting violations" → `/pr-fix-lint <PR_NUMBER>`
- "Add the missing error boundary" → Fix manually (doesn't fit either skill)
- "Fix all the test failures" → `/pr-fix-tests <PR_NUMBER>`
- "Fix all the lint issues" → `/pr-fix-lint <PR_NUMBER>`

**When fixing manually (not using fix skills):**
1. **Actually fix the code** - Make the necessary changes using Edit tool
2. **Follow review requirements** - Apply all quality standards from this review skill
3. **Add test coverage** - Ensure fixes have full test coverage (100% for DI code)
4. **Commit the fix** - Create a commit with descriptive message
5. **Verify the fix** - Confirm the issue is resolved

### CRITICAL: How to Fix Test Issues

**Remember the fundamental principle:** Test failures indicate code design problems, not test problems.

**When fixing test issues (skipped tests, sleep in tests, flaky tests):**

1. **DO NOT just change the test** - The test is revealing a design flaw
2. **DO refactor the production code** - Make the code testable
3. **THEN update the test** - To use the new testability hooks

**Example: Fixing a skipped test**

**❌ Wrong approach (only changing test):**
```ruby
# Before: test is skipped because it's flaky
skip "flaky test - sometimes worker doesn't finish in time"

# Wrong fix: increase timeout and hope
it 'processes data' do
  worker.start
  sleep 5  # Increased from 1 second, still flaky
  expect(worker.queue).to be_empty
end
```

**✅ Right approach (refactor production code first):**

**Step 1: Refactor production code to add testability hooks**
```ruby
# lib/datadog/di/worker.rb - ADD OBSERVABILITY
class Worker
  def initialize
    @completion_queue = Queue.new  # NEW: for tests to wait on
  end

  attr_reader :completion_queue  # NEW: expose for tests

  def start
    Thread.new do
      process_all
      @completion_queue.push(:done)  # NEW: signal completion
    end
  end
end
```

**Step 2: Update test to use new hooks**
```ruby
# spec/datadog/di/worker_spec.rb - USE NEW HOOKS
it 'processes data' do
  worker.start
  Timeout.timeout(2) { worker.completion_queue.pop }  # Deterministic
  expect(worker.queue).to be_empty
end
```

**Example: Fixing sleep in tests**

**❌ Wrong approach:**
```ruby
# Just remove sleep and hope test still works
it 'exports data' do
  exporter.export_async
  # Removed sleep, now test is even more flaky
  expect(last_export).to eq(data)
end
```

**✅ Right approach:**

**Step 1: Add callback support to production code**
```ruby
# lib/datadog/di/exporter.rb - ADD CALLBACK
def export_async(data, &on_complete)
  Thread.new do
    export(data)
    on_complete&.call  # NEW: call completion callback
  end
end
```

**Step 2: Use callback in test**
```ruby
# spec/datadog/di/exporter_spec.rb - USE CALLBACK
it 'exports data' do
  queue = Queue.new
  exporter.export_async(data) { queue.push(:done) }
  Timeout.timeout(2) { queue.pop }  # Wait for callback
  expect(last_export).to eq(data)
end
```

**Common patterns for making code testable:**

| Untestable Pattern | Testable Refactor |
|-------------------|-------------------|
| Fire-and-forget thread | Add completion callback/queue |
| Hidden internal state | Expose via attr_reader or query method |
| Hard-coded dependency | Inject via initialize parameter |
| No way to wait | Add observable state or completion signal |
| Background loop | Add stop method and status query |
| Race condition | Add synchronization primitives (Mutex, Queue) |

**Files you'll need to edit:**
- Production code file (lib/datadog/di/*.rb) - to add testability
- Test file (spec/datadog/di/*_spec.rb) - to use new hooks
- Both changes go in the same commit

### Example Follow-Up Flows

**Example 1: Test issue → Delegate to pr-fix-tests**

**Review identifies:**
```
❌ CRITICAL: Sleep in tests
File: spec/datadog/di/worker_spec.rb:78
```

**User says:** "Fix the sleep"

**Correct response:**
```
I'll use the pr-fix-tests skill to fix this test issue.
/pr-fix-tests <PR_NUMBER>
```

The pr-fix-tests skill will:
- Read CLAUDE.md for guidelines
- Fix the sleep with proper synchronization
- Ensure 100% test coverage
- Apply all review requirements
- Monitor CI until tests pass

---

**Example 2: Lint issue → Delegate to pr-fix-lint**

**Review identifies:**
```
❌ Repository Guidelines: Linting violations
Files with violations: lib/datadog/di/probe_manager.rb
```

**User says:** "Fix the linting issues"

**Correct response:**
```
I'll use the pr-fix-lint skill to fix the linting issues.
/pr-fix-lint <PR_NUMBER>
```

The pr-fix-lint skill will:
- Read CLAUDE.md for guidelines
- Run StandardRB (dd-trace-rb's linter)
- Apply trailing commas per CLAUDE.md
- Commit and push fixes

**Note:** dd-trace-rb uses StandardRB for code style (not RuboCop).

---

**Example 3: Manual fix → Fix directly**

**Review identifies:**
```
❌ CRITICAL: Missing error boundary
File: lib/datadog/di/probe_manager.rb:67
TracePoint callback has no error handling
```

**User says:** "Add the error boundary"

**Correct response:**
This requires a manual fix (not covered by fix skills):
1. Read the file
2. Add rescue block with logging and telemetry
3. Add test coverage for error boundary
4. Commit the change

**Why manual?** Adding error boundaries with proper logging and telemetry requires context-specific code that the fix skills don't handle.

---

**Incorrect responses:**
- "You should replace the sleep with a Queue..." ❌ (explaining instead of fixing)
- "Here's the code to add: [snippet]" ❌ (providing code without applying it)
- Using wrong fix skill ❌ (e.g., pr-fix-lint for test issues)

### Fixing Multiple Issues

When asked to fix multiple issues:

```bash
# User: "Fix all the sleep calls"
# Your actions:
1. Read all test files with sleep
2. Fix each sleep call with proper synchronization
3. Apply review requirements to each fix
4. Commit each file separately with descriptive messages
5. Report what was fixed
```

### Following Review Requirements When Fixing

**CRITICAL:** When fixing issues, you must follow ALL requirements from this review skill.

**Before making any fix, apply these review requirements:**
- 100% test coverage for new/changed code
- Error boundaries where needed
- Proper error handling (logging + telemetry)
- Trailing commas in di/ and symbol_database/ directories
- No hardcoded /tmp paths
- No sleep in tests
- No skipped tests

**Example: Fixing a missing error boundary**

When adding an error boundary, you must also:
- ✅ Add test coverage for the error boundary (100% coverage requirement)
- ✅ Use trailing commas in di/ and symbol_database/ directories
- ✅ Add both logging (DEBUG/WARN) and telemetry
- ✅ Test that exceptions are caught and don't propagate

**Example: Fixing sleep in tests**

When removing sleep from a test, you must also:
- ✅ Ensure the replacement (Queue, ConditionVariable) has test coverage
- ✅ Use trailing commas in di/ and symbol_database/ directories
- ✅ Verify the test is deterministic (no flakiness)
- ✅ Confirm the test actually tests what it claims to test

**Example: Fixing hardcoded /tmp path**

When replacing a hardcoded /tmp path, you must also:
- ✅ Use `Dir.mktmpdir` or `Tempfile.create`
- ✅ Ensure proper cleanup (block form or ensure block)
- ✅ Add test coverage for the temporary file/directory usage
- ✅ Use trailing commas in di/ and symbol_database/ directories
- ✅ Verify tests pass with isolated temp directories

**Common mistakes to avoid:**
- ❌ Fixing the immediate issue but ignoring test coverage requirement
- ❌ Adding code without trailing commas in di/ directories
- ❌ Adding error boundary without telemetry
- ❌ Fixing sleep but not verifying deterministic behavior

**Verification checklist after each fix:**
- [ ] Fixed the specific issue identified in review
- [ ] Added full test coverage (100% for di/ code)
- [ ] Applied trailing commas in di/ and symbol_database/
- [ ] Added error handling where needed (logging + telemetry)
- [ ] No hardcoded /tmp paths introduced
- [ ] No sleep introduced in tests
- [ ] Tests pass locally
- [ ] Committed with descriptive message

### Pattern: Review → Fix Cycle

```
┌─────────────────────────────────────────────────┐
│ 1. Perform Review                               │
│    - Identify all issues                        │
│    - Categorize as CRITICAL or Additional       │
│    - Provide detailed feedback                  │
└─────────────────────────────────────────────────┘
                     ↓
┌─────────────────────────────────────────────────┐
│ 2. User Requests Fix                            │
│    "Fix the skipped tests"                      │
│    "Add error boundaries"                       │
│    "Fix all the issues you found"               │
└─────────────────────────────────────────────────┘
                     ↓
┌─────────────────────────────────────────────────┐
│ 3. Apply Fixes (DO THIS, DON'T EXPLAIN)        │
│    - Read the problematic code                  │
│    - Make the necessary changes                 │
│    - Add full test coverage                     │
│    - Apply all review requirements              │
│    - Commit with descriptive message            │
│    - Repeat for each issue                      │
└─────────────────────────────────────────────────┘
                     ↓
┌─────────────────────────────────────────────────┐
│ 4. Report Completion                            │
│    "Fixed X issues:"                            │
│    "- Removed sleep from worker_spec.rb:78"     │
│    "- Added error boundary to probe_manager..."│
│    "- Replaced /tmp path with Dir.mktmpdir..." │
└─────────────────────────────────────────────────┘
```

### Scope of Fixes

**Always fix:**
- Issues you identified in your review
- Code quality problems found during review
- Test issues (skipped tests, sleep calls, coverage gaps)
- Missing error boundaries or telemetry
- Hardcoded /tmp paths
- Repository guideline violations

**Use judgment for:**
- Architectural changes (may need discussion first)
- Breaking changes (discuss impact)
- Large refactorings (may need planning)

**When in doubt:**
- If the fix is straightforward and clear from your review → Fix it
- If the fix requires architectural decisions → Discuss first
- If unclear what the user wants fixed → Ask for clarification

### Commit Messages for Review Fixes

Format:
```
Fix <issue> found in review

Address review feedback: <what you fixed and why>

<List of specific changes>
- Fixed in <file>:<line>
- Fixed in <file>:<line>

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>
```

Examples:
```
Fix skipped tests in probe_spec.rb

Address review feedback: Enable and fix previously skipped tests
for probe registration and error handling.

- Fixed probe registration test (was skipped due to race condition)
- Added proper synchronization with Queue
- Fixed error handling test with correct expectations

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>
```

```
Add missing error boundary to TracePoint callback

Address review feedback: Wrap probe.execute in error boundary to
prevent exceptions from propagating to customer code.

- Added rescue block with DEBUG logging
- Added telemetry reporting for errors
- Added test coverage for error boundary

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>
```

### Summary

**The key principle:** After a review, when told to fix something, **be proactive and fix it**. Don't wait, don't explain, don't recommend. Read the code, make the change, commit it, and report what you did.

This makes the review process efficient: review → fix → done, rather than review → explain → wait → fix → done.
