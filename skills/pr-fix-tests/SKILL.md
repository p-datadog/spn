---
name: pr-fix-tests
description: This skill should be used when the user asks to "fix tests", "fix test failures", "fix failing specs", or mentions fixing test CI failures in a pull request.
version: 0.2.0
---

# PR Fix Tests

This skill automatically fixes test failures in a pull request. Each fix is committed separately with a descriptive message, and all commits are pushed at the end.

## Overview

The skill automates the process of:
1. Fetching a specific pull request
2. Checking CI job status to identify test failures
3. Filtering for TEST failures only (not lint/static analysis/typing)
4. Analyzing and fixing each test failure
5. Committing each fix with a descriptive message
6. Pushing all commits to the PR branch
7. **Continuously monitoring CI status** (polling every minute)
8. **Fixing any new or remaining test failures** until all tests pass

## When This Skill Applies

Use this skill when:
- User asks to "fix test failures in PR"
- User wants to "fix failing specs"
- User mentions "fix RSpec", "fix Jest tests", "fix pytest failures"
- PR has test CI failures that need to be resolved
- User wants to get tests passing before review

## Types of Failures Addressed

This skill ONLY addresses **test failures**:

### ✅ Addressed (Test Failures)
- **Unit tests:** RSpec, Jest, pytest, JUnit, etc.
- **Integration tests:** API tests, database tests
- **E2E tests:** Selenium, Cypress, Playwright
- **Spec failures:** Any RSpec/Jest/Mocha specs
- **Test suites:** Any job with "test" in the name
- **Build + test:** Jobs that build and run tests

### ❌ NOT Addressed (Non-Test Failures)
- Linting failures (RuboCop, ESLint, Pylint)
- Static analysis (Brakeman, CodeQL)
- Type checking (TypeScript, Sorbet, mypy)
- Code formatting (Prettier, Black)
- Style checks

**Exception:** If fixing a test requires type changes, that's acceptable. But don't go out of your way to fix unrelated type/lint issues.

## CRITICAL: Handling Flaky and Intermittent Test Failures

**⚠️ Pay close attention to test flakiness and intermittent failures.**

When a test fails inconsistently or shows signs of being flaky:

**DO:**
- **Investigate the root cause** - Why does it fail sometimes?
- **Fix the underlying issue** - Race conditions, timing, cleanup, etc.
- **Look for patterns** - Does it fail in specific environments or conditions?
- **Analyze the failure** - Read error messages carefully for clues

**Common root causes:**
- **Race conditions** - Async operations not properly awaited
- **Improper cleanup** - Previous test state leaking into current test
- **Timing dependencies** - Relying on arbitrary sleeps or delays
- **External dependencies** - Network, database, or filesystem state
- **Resource leaks** - Unclosed connections, file handles, etc.
- **Non-deterministic data** - Random values, timestamps without mocking

**DON'T:**
- Don't add `sleep()` calls to "fix" timing issues
- Don't skip or comment out flaky tests
- Don't add retry logic to mask flakiness
- Don't ignore intermittent failures hoping they'll go away
- Don't apply band-aid workarounds that hide the problem

**Proper fixes:**
- Use proper synchronization (Queue, ConditionVariable, callbacks)
- Add explicit cleanup (after blocks, ensure clauses)
- Mock time/random values for determinism
- Use test transaction rollback for database cleanup
- Wait for explicit conditions, not arbitrary timeouts

**If you can't determine the root cause:**
- Document what you've tried
- Provide analysis of the failure pattern
- Request manual review with your findings
- Don't merge without understanding the issue

## Workflow

### Step 1: Fetch PR and Check CI Status

```bash
# Checkout the PR
gh pr checkout <PR_NUMBER>

# Get PR information
gh pr view <PR_NUMBER> --json number,title,headRefName,baseRefName

# Get check status
gh pr checks <PR_NUMBER> --json name,state,link,detailsUrl
```

**Important:** Focus on checks where:
- `state` = "FAILURE"
- Name DOES contain "test", "spec", "e2e", "integration", or "build"

### Step 2: Identify Test Failures

Look for failed checks matching these patterns:

**Test Jobs:**
- Contains "test" (case-insensitive)
- Contains "spec"
- Contains "e2e"
- Contains "integration"
- Contains "build" + "test"
- "Test (Ruby X.X)"
- "Jest", "RSpec", "pytest"
- "Cypress", "Playwright", "Selenium"

**Exclude these patterns:**
- "RuboCop", "rubocop", "lint"
- "ESLint", "Prettier"
- "Brakeman", "CodeQL"
- "Sorbet", "mypy", "type-check"
- Pure formatting/linting jobs

### Step 3: Analyze Each Test Failure

For each identified test failure:

```bash
# Get the run ID from the check's link
run_id=$(gh pr checks <PR_NUMBER> --json name,link | \
  jq -r '.[] | select(.name == "<CHECK_NAME>") | .link | split("/") | .[-3]')

# View the failure logs
gh run view $run_id --log > /tmp/test_failure_${CHECK_NAME}.txt

# Or use the details URL
details_url=$(gh pr checks <PR_NUMBER> --json name,detailsUrl | \
  jq -r '.[] | select(.name == "<CHECK_NAME>") | .detailsUrl')
```

Parse the logs to understand:
- Which test file(s) are failing
- What assertions are failing
- Error messages and stack traces
- Root cause of the failure

### Step 4: Fix Each Test Failure (Commit Separately)

**CRITICAL: Make ONE commit per logical fix**

Group related test fixes together, but separate unrelated fixes:
- ✅ Fix all tests in one file → one commit
- ✅ Fix one test class/describe block → one commit
- ✅ Fix related tests across multiple files → one commit if they share a root cause
- ❌ Don't mix unrelated fixes in one commit

#### Common Test Failure Patterns and Fixes

**Pattern 1: Outdated test expectations**
```ruby
# Test expects old behavior
expect(user.status).to eq("active")
# But code now returns "activated"

# Fix: Update the expectation
expect(user.status).to eq("activated")
```

**Pattern 2: Missing test setup**
```ruby
# Test fails because data isn't set up
it "processes order" do
  expect(order.process).to be true  # order is nil!
end

# Fix: Add setup
it "processes order" do
  order = create(:order)  # Add missing setup
  expect(order.process).to be true
end
```

**Pattern 3: Race conditions or timing issues**
```javascript
// Test fails intermittently
test('async operation completes', () => {
  startAsync()
  expect(result).toBe('done')  // May not be done yet!
})

// Fix: Add proper async handling
test('async operation completes', async () => {
  await startAsync()
  expect(result).toBe('done')
})
```

**Pattern 4: Changed method signatures**
```ruby
# Test calls method with old signature
allow(service).to receive(:process).with(data)

# But method now requires additional parameter
service.process(data, options)

# Fix: Update mock
allow(service).to receive(:process).with(data, anything)
```

**Pattern 5: Missing dependencies or fixtures**
```python
# Test fails because fixture doesn't exist
def test_user_creation():
    user = User.objects.get(email='test@example.com')  # DoesNotExist!

# Fix: Create the fixture
def test_user_creation():
    user = User.objects.create(email='test@example.com')
    # Test continues...
```

**Pattern 6: Database state issues**
```ruby
# Test fails because database has stale data
it "finds only active users" do
  expect(User.active.count).to eq(1)  # Fails if previous tests left data
end

# Fix: Clean state in before block
before do
  User.delete_all
  create(:user, :active)
end

it "finds only active users" do
  expect(User.active.count).to eq(1)
end
```

### Step 5: Commit Each Fix

After fixing a test or group of related tests:

```bash
# Add the changed files
git add path/to/test_file.rb path/to/source_file.rb

# Commit with descriptive message
git commit -m "$(cat <<'EOF'
Fix <test description> test

<Brief explanation of what was wrong and how you fixed it>

Fixes failing test in <test file>:<line>.

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>
EOF
)"
```

#### Commit Message Examples

**Example 1: Update test expectations**
```
Fix user status expectation in user_spec.rb

Update test to expect "activated" instead of "active" to match
the recent change to user status values.

Fixes failing test in spec/models/user_spec.rb:45.

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>
```

**Example 2: Add missing test setup**
```
Add missing order setup in checkout_spec.rb

Create order instance before testing order.process method.
Test was failing with NoMethodError on nil.

Fixes failing test in spec/services/checkout_spec.rb:89.

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>
```

**Example 3: Fix async timing**
```
Fix async timing in payment test

Add await to ensure async payment processing completes before
checking result. Test was flaky due to race condition.

Fixes failing test in __tests__/payment.test.js:34.

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>
```

**Example 4: Update mock signature**
```
Update service mock to match new signature

Add options parameter to service.process mock to match updated
method signature from PR #4521.

Fixes failing test in spec/services/processor_spec.rb:112.

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>
```

### Step 6: Push All Commits

After all fixes are committed:

```bash
# Verify commits were made
git log origin/$(git branch --show-current)..HEAD --oneline

# Push all commits to the PR branch
git push origin HEAD
```

### Step 7: Continuously Monitor CI and Fix Remaining Failures

**IMPORTANT:** After pushing fixes, continuously monitor CI status and fix any remaining or new test failures.

**Why this is needed:**
- Initial fixes may not catch all issues
- CI environment may reveal failures not reproducible locally
- Some tests may only fail in specific CI configurations
- Flaky tests may fail intermittently (investigate root cause, don't mask with workarounds)
- New failures may appear after fixing others

**Monitoring process:**

```bash
# Wait for CI to start running after push (30 seconds)
echo "Waiting for CI to start running new checks..."
sleep 30

# Begin polling loop
iteration=1
while true; do
  echo "=== Monitoring CI Status (Check #$iteration) ==="
  echo "Time: $(date)"

  # Get current CI status
  checks=$(gh pr checks <PR_NUMBER> --json name,state,conclusion,link,detailsUrl | \
    jq '[.[] | select(.name | test("test|spec|e2e|integration|build.*test"; "i"))]')

  # Count running checks
  running=$(echo "$checks" | jq '[.[] | select(.state == "IN_PROGRESS" or .state == "QUEUED" or .state == "PENDING")] | length')

  # Count failed test checks
  failed=$(echo "$checks" | jq '[.[] | select(.conclusion == "FAILURE")] | length')

  # Count successful test checks
  passed=$(echo "$checks" | jq '[.[] | select(.conclusion == "SUCCESS")] | length')

  echo "Test checks: $passed passed, $failed failed, $running running"

  # If there are still running checks, wait and continue
  if [ "$running" -gt 0 ]; then
    echo "CI still running, waiting 1 minute before next check..."
    sleep 60
    iteration=$((iteration + 1))
    continue
  fi

  # If there are failed test checks, analyze and fix them
  if [ "$failed" -gt 0 ]; then
    echo "Found $failed test failure(s), analyzing..."

    # Get list of failed test checks
    failed_checks=$(echo "$checks" | jq -r '.[] | select(.conclusion == "FAILURE") | .name')

    echo "Failed checks:"
    echo "$failed_checks"

    # Analyze and fix each failure (same process as initial fixes)
    # ... (follow Steps 3-6 again for new failures)

    # After fixing and pushing, restart the monitoring loop
    echo "Fixes pushed, restarting CI monitoring..."
    sleep 30
    iteration=1
    continue
  fi

  # All test checks passed!
  echo "✅ All test checks passed! No more failures to fix."
  break
done
```

**Key behaviors:**
1. **Poll every 1 minute** - Check CI status at 1-minute intervals
2. **Wait for running checks** - Don't check for failures while tests are still running
3. **Fix new failures** - When failures appear, fix them using the same process as initial fixes
4. **Commit and push** - Each round of fixes gets committed and pushed
5. **Continue monitoring** - After pushing fixes, wait and check again
6. **Exit when all pass** - Stop monitoring when all test checks are successful

**Stopping conditions:**
- ✅ All test checks pass (conclusion == "SUCCESS")
- ⚠️ Manual intervention needed (if fixes don't resolve failures after 3 rounds)
- ⚠️ User cancels the monitoring

**Example monitoring output:**

```
=== Monitoring CI Status (Check #1) ===
Time: 2024-03-10 10:15:30
Test checks: 8 passed, 0 failed, 4 running
CI still running, waiting 1 minute before next check...

=== Monitoring CI Status (Check #2) ===
Time: 2024-03-10 10:16:30
Test checks: 10 passed, 2 failed, 0 running
Found 2 test failure(s), analyzing...
Failed checks:
Test (Ruby 3.0)
Test (Ruby 2.7)

Analyzing Test (Ruby 3.0) failures...
  - spec/models/user_spec.rb:52 - assertion failure

Fixing spec/models/user_spec.rb...
✅ Committed: Fix user validation test

Analyzing Test (Ruby 2.7) failures...
  - Same issue as Ruby 3.0, already fixed

Pushing fixes...
Fixes pushed, restarting CI monitoring...

=== Monitoring CI Status (Check #1) ===
Time: 2024-03-10 10:18:00
Test checks: 0 passed, 0 failed, 12 running
CI still running, waiting 1 minute before next check...

=== Monitoring CI Status (Check #2) ===
Time: 2024-03-10 10:19:00
Test checks: 12 passed, 0 failed, 0 running
✅ All test checks passed! No more failures to fix.
```

**Important notes:**
- Only monitor and fix **test** failures (not lint/static analysis)
- Each round of fixes should be committed separately
- If the same test fails multiple times, investigate more deeply
- If a test cannot be fixed after 3 attempts, report for manual review
- Always verify fixes locally before pushing when possible

## Running Tests Locally

Before pushing, verify fixes work:

**Ruby (RSpec):**
```bash
# Run specific test file
bundle exec rspec spec/path/to/test_spec.rb

# Run specific test line
bundle exec rspec spec/path/to/test_spec.rb:45

# Run all tests
bundle exec rspec
```

**JavaScript (Jest):**
```bash
# Run specific test file
npm test path/to/test.test.js

# Run tests matching pattern
npm test -- --testNamePattern="payment processing"

# Run all tests
npm test
```

**Python (pytest):**
```bash
# Run specific test file
pytest tests/path/to/test_file.py

# Run specific test
pytest tests/path/to/test_file.py::test_function_name

# Run all tests
pytest
```

## Implementation Steps

When the skill is invoked with a PR number:

1. **Checkout PR:**
   ```bash
   gh pr checkout <PR_NUMBER>
   ```

2. **Get failed test checks:**
   ```bash
   failed_tests=$(gh pr checks <PR_NUMBER> --json name,state,link,detailsUrl | \
     jq -r '[.[] | select(.state == "FAILURE")] |
     map(select(.name | test("test|spec|e2e|integration|build.*test"; "i")))')
   ```

3. **For each test failure:**
   - Download and analyze the failure logs
   - Identify which test file(s) are failing
   - Read the test file and understand what's expected
   - Read the source code being tested
   - Determine the root cause
   - Fix the issue (in test file or source code as needed)
   - Run the test locally to verify fix
   - Commit with descriptive message

4. **Count commits made:**
   ```bash
   commits_made=$(git log origin/$(git branch --show-current)..HEAD --oneline | wc -l)
   ```

5. **Push if commits were made:**
   ```bash
   if [ "$commits_made" -gt 0 ]; then
     echo "Pushing $commits_made commit(s) to PR branch..."
     git push origin HEAD
   else
     echo "No test fixes were needed or all fixes failed"
     exit 0
   fi
   ```

6. **Continuously monitor CI and fix remaining failures:**
   ```bash
   # Wait for CI to start
   sleep 30

   # Polling loop
   iteration=1
   max_fix_rounds=10  # Safety limit
   fix_rounds=0

   while [ "$fix_rounds" -lt "$max_fix_rounds" ]; do
     echo "=== CI Status Check #$iteration ==="

     # Get test check status
     checks=$(gh pr checks <PR_NUMBER> --json name,state,conclusion | \
       jq '[.[] | select(.name | test("test|spec|e2e|integration"; "i"))]')

     running=$(echo "$checks" | jq '[.[] | select(.state == "IN_PROGRESS" or .state == "QUEUED")] | length')
     failed=$(echo "$checks" | jq '[.[] | select(.conclusion == "FAILURE")] | length')

     # Wait if still running
     if [ "$running" -gt 0 ]; then
       echo "CI running, waiting 1 minute..."
       sleep 60
       iteration=$((iteration + 1))
       continue
     fi

     # Fix failures if any
     if [ "$failed" -gt 0 ]; then
       echo "Found $failed failure(s), fixing..."
       fix_rounds=$((fix_rounds + 1))

       # Repeat steps 2-5 for new failures
       # (analyze, fix, commit, push)

       sleep 30  # Wait for CI to start
       iteration=1
       continue
     fi

     # All passed!
     echo "✅ All tests passing!"
     break
   done

   if [ "$fix_rounds" -ge "$max_fix_rounds" ]; then
     echo "⚠️ Reached maximum fix rounds. Manual review needed."
   fi
   ```

## Decision Tree for Fixes

```
Is check failing?
  ├─ No → Skip
  └─ Yes → Does name contain "test"/"spec"/"e2e"?
      ├─ No → Skip (non-test failure, use pr-fix-lint instead)
      └─ Yes → Analyze test failure logs
          ├─ Outdated expectation → Update test → Commit
          ├─ Missing setup → Add setup → Commit
          ├─ Async timing → Fix async handling → Commit
          ├─ Changed signature → Update test → Commit
          ├─ Code bug → Fix source code → Commit
          └─ Other → Investigate → Fix → Commit
```

## Fixing Strategy

### 1. Read the Failure First
- Don't guess at fixes
- Read the full error message and stack trace
- Understand what the test expects vs what it gets

### 2. Locate the Test
- Find the exact test file and line number
- Read the test code to understand intent
- Check related test files for context

### 3. Understand the Source Code
- Read the code being tested
- Check recent changes in the PR
- Look for related changes that might affect the test

### 4. Determine Root Cause
- Is the test wrong? (needs updating)
- Is the code wrong? (needs fixing)
- Is setup missing? (needs adding)
- Is it a timing issue? (needs async handling)

### 5. Make Minimal Fix
- Change only what's necessary
- Don't refactor while fixing tests
- Don't add unrelated improvements

### 6. Verify Locally
- Run the specific test
- Run related tests
- Run full test suite if possible

## Type Changes Are OK

While this skill focuses on test failures, you may need to change types:

**Acceptable type changes:**
```ruby
# Updating type signature to match test expectations
sig { params(user: User).returns(Boolean) }  # Was T::Boolean
def process(user)
  # ...
end
```

```typescript
// Fixing type to match test usage
function process(data: ProcessData): Promise<Result> {  // Was Result
  // ...
}
```

**NOT acceptable (out of scope):**
- Fixing type errors in unrelated files
- Adding type signatures to uncovered code
- Refactoring types for better structure
- Fixing lint/static analysis issues

## Example Execution

```
$ /pr-fix-tests 4567

# PR Fix Tests - PR #4567

## Checking PR #4567: Add new feature
Branch: feature/new-thing
Base: main

## Checking CI Status...
Found 3 failed checks:
  ❌ Test (Ruby 3.0) - 2 failing tests
  ❌ Test (Ruby 2.7) - 2 failing tests
  ❌ Jest Tests - 1 failing test
  ✅ RuboCop - Skipped (non-test job)

## Analyzing Test (Ruby 3.0) failures...

### Failure 1: spec/models/user_spec.rb:45
Error: expected "active" but got "activated"
Root cause: Test expectation outdated after status change

### Failure 2: spec/services/processor_spec.rb:112
Error: wrong number of arguments (1 for 2)
Root cause: Method signature changed, mock needs update

## Fixing spec/models/user_spec.rb...
✅ Committed: Fix user status expectation in user_spec.rb

## Fixing spec/services/processor_spec.rb...
✅ Committed: Update service mock to match new signature

## Analyzing Jest Tests failures...

### Failure: __tests__/payment.test.js:34
Error: Expected true but received undefined
Root cause: Missing await on async function

## Fixing __tests__/payment.test.js...
✅ Committed: Fix async timing in payment test

## Running tests locally to verify...
✅ All fixed tests passing locally

## Summary - Initial Round
- Total checks analyzed: 4
- Non-test jobs skipped: 1
- Test failures fixed: 3
- Commits created: 3
- Commits pushed: 3

## Monitoring CI for remaining failures...
Waiting for CI to start running new checks...

=== CI Status Check #1 ===
Time: 2024-03-10 10:20:00
Test checks: 0 passed, 0 failed, 12 running
CI running, waiting 1 minute...

=== CI Status Check #2 ===
Time: 2024-03-10 10:21:00
Test checks: 0 passed, 0 failed, 12 running
CI running, waiting 1 minute...

=== CI Status Check #3 ===
Time: 2024-03-10 10:22:00
Test checks: 10 passed, 2 failed, 0 running
Found 2 failure(s), fixing...

## Analyzing new failures...

### Failure: spec/integration/workflow_spec.rb:89
Error: undefined method `process_async` for nil:NilClass
Root cause: Integration test needs additional setup after our changes

## Fixing spec/integration/workflow_spec.rb...
✅ Committed: Add missing workflow setup in integration test

Pushing fixes...

## Monitoring CI for remaining failures...
Waiting for CI to start running new checks...

=== CI Status Check #1 ===
Time: 2024-03-10 10:24:30
Test checks: 0 passed, 0 failed, 12 running
CI running, waiting 1 minute...

=== CI Status Check #2 ===
Time: 2024-03-10 10:25:30
Test checks: 5 passed, 0 failed, 7 running
CI running, waiting 1 minute...

=== CI Status Check #3 ===
Time: 2024-03-10 10:26:30
Test checks: 12 passed, 0 failed, 0 running

✅ All tests passing! No more failures to fix.

## Final Summary
- Total fix rounds: 2
- Total commits created: 4
- All test checks passing in CI
```

## Edge Cases

**Multiple test failures in same file:**
- Fix all failures in one file
- Make one commit for the entire file
- Example: "Fix 3 failing tests in user_spec.rb"

**Test failure requires source code fix:**
- Fix both source and test in same commit
- Explain the source code fix in commit message
- Example: "Fix user.process to handle nil input"

**Flaky test:**
- **ALWAYS investigate root cause** - Why does it fail inconsistently?
- Common causes: race conditions, improper cleanup, timing dependencies, leaked state
- Add proper synchronization (Queue, ConditionVariable, explicit waits)
- Fix cleanup issues (after blocks, ensure clauses, transaction rollback)
- Mock non-deterministic values (time, random, external data)
- **Don't apply workarounds:** No sleep(), no retries, no skipping
- If root cause is unclear, document investigation and request manual review
- Example root cause fixes:
  ```ruby
  # BAD - Masks the problem
  sleep 0.5
  expect(result).to be_ready

  # GOOD - Fixes race condition
  queue = Queue.new
  async_operation { |r| queue.push(r) }
  result = Timeout.timeout(2) { queue.pop }
  expect(result).to be_ready
  ```

**Test failure in generated code:**
- Regenerate the code if possible (e.g., `rails g`)
- Update generator templates if needed
- Commit both generated code and template changes

**Cannot reproduce failure locally:**
- Check for environment-specific issues
- Look for missing dependencies or setup
- Check Ruby/Node version differences
- Still attempt to fix based on error message

**Test requires database migration:**
- Run migrations: `rails db:migrate`
- Commit both migration run and test fix
- Note in commit message: "Run migration to fix test"

**No test failures found:**
- Report: "No test failures found in CI."
- Check if failures are lint/type/static analysis (use pr-fix-lint)
- Exit without making commits

**Fixes don't resolve CI failure:**
- Commit what was fixed anyway
- Report: "⚠️ Fixes applied but some tests may still fail. Manual review needed."
- Suggest running full test suite: "Run `bundle exec rspec` to verify locally"

**Protected branch (can't push):**
- Report error with branch protection info
- Show commits made locally
- Suggest: "Contact repo admin to adjust branch protection"

**Conflicts during push:**
- Report: "⚠️ Push failed due to conflicts. PR branch has been updated."
- Instruct: "Run `git pull --rebase` and push again manually."

### Continuous Monitoring Edge Cases

**Same test fails multiple times:**
- If a test fails after 2 fix attempts, investigate more deeply
- Check if the fix is actually addressing the root cause
- Look for race conditions or environment-specific issues
- After 3 failed attempts, report for manual review

**CI takes very long to run:**
- Continue polling at 1-minute intervals
- Don't increase polling frequency (respect CI resources)
- Report progress to user every 5 minutes

**New test failures appear after fixing others:**
- This is expected - some tests may depend on others
- Fix the new failures using the same process
- Commit and push each round of fixes separately

**All tests pass locally but fail in CI:**
- CI environment may differ (Ruby version, dependencies, timezone, etc.)
- Analyze CI-specific error messages carefully
- Fix based on CI logs even if tests pass locally
- Consider adding CI-specific test setup if needed

**CI stuck in "PENDING" state:**
- Wait up to 10 minutes for CI to start
- If still pending after 10 minutes, report issue
- May indicate CI infrastructure problems
- Suggest user check GitHub Actions status

**Non-test checks fail during monitoring:**
- Ignore them - only fix test failures
- Don't restart monitoring for lint/type check failures
- Report them in final summary but don't attempt to fix

**Maximum fix rounds reached (10 rounds):**
- Stop monitoring and report to user
- Show which tests are still failing
- Suggest manual review: "Some tests may require deeper investigation"
- List all commits made and fixes attempted

**User cancels monitoring:**
- Stop polling immediately
- Report status at time of cancellation
- Show commits made so far
- Note which tests were still failing or running

**PR is merged during monitoring:**
- Detect merge by checking PR status
- Stop monitoring gracefully
- Report: "PR was merged during monitoring"

**Branch is deleted during monitoring:**
- Detect branch deletion
- Stop monitoring
- Report: "PR branch was deleted"

## Required Tools

This skill requires:
- GitHub CLI (`gh`) authenticated
- Git configured
- Language-specific tools:
  - Ruby: `bundle`, `rspec`
  - JavaScript: `npm`/`yarn`, `jest`
  - Python: `pip`, `pytest`
  - Go: `go test`

## Commands Reference

```bash
# Checkout PR
gh pr checkout <PR_NUMBER>

# Get failed test checks
gh pr checks <PR_NUMBER> --json name,state,link,detailsUrl | \
  jq '[.[] | select(.state == "FAILURE")] |
      map(select(.name | test("test|spec|e2e|integration"; "i")))'

# Get test failure logs
run_id=$(gh pr checks <PR_NUMBER> --json name,link | \
  jq -r '.[] | select(.name == "<CHECK_NAME>") | .link | split("/") | .[-3]')
gh run view $run_id --log

# Run tests locally
bundle exec rspec spec/path/to/test_spec.rb:45    # Ruby - specific line
bundle exec rspec spec/path/to/test_spec.rb       # Ruby - entire file
npm test path/to/test.test.js                     # JavaScript
pytest tests/path/to/test_file.py::test_name      # Python

# Commit fix with heredoc (proper formatting)
git commit -m "$(cat <<'EOF'
Fix <test description> test

Explanation of what was wrong and how fixed.

Fixes failing test in <file>:<line>.

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>
EOF
)"

# Count new commits
git log origin/$(git branch --show-current)..HEAD --oneline | wc -l

# Push all commits
git push origin HEAD

# ============================================
# CI Monitoring Commands
# ============================================

# Check current CI status with state and conclusion
gh pr checks <PR_NUMBER> --json name,state,conclusion,link,detailsUrl

# Get test checks only (filter by name pattern)
gh pr checks <PR_NUMBER> --json name,state,conclusion | \
  jq '[.[] | select(.name | test("test|spec|e2e|integration|build.*test"; "i"))]'

# Count running test checks
gh pr checks <PR_NUMBER> --json name,state,conclusion | \
  jq '[.[] | select(.name | test("test|spec|e2e|integration"; "i"))] |
      [.[] | select(.state == "IN_PROGRESS" or .state == "QUEUED" or .state == "PENDING")] | length'

# Count failed test checks
gh pr checks <PR_NUMBER> --json name,state,conclusion | \
  jq '[.[] | select(.name | test("test|spec|e2e|integration"; "i"))] |
      [.[] | select(.conclusion == "FAILURE")] | length'

# Count passed test checks
gh pr checks <PR_NUMBER> --json name,state,conclusion | \
  jq '[.[] | select(.name | test("test|spec|e2e|integration"; "i"))] |
      [.[] | select(.conclusion == "SUCCESS")] | length'

# List names of failed test checks
gh pr checks <PR_NUMBER> --json name,state,conclusion | \
  jq -r '[.[] | select(.name | test("test|spec|e2e|integration"; "i"))] |
         [.[] | select(.conclusion == "FAILURE")] | .[].name'

# Check PR status (to detect if merged or closed)
gh pr view <PR_NUMBER> --json state,merged | jq -r '.state, .merged'

# Full monitoring loop example
iteration=1
while true; do
  echo "=== Check #$iteration at $(date) ==="

  checks=$(gh pr checks <PR_NUMBER> --json name,state,conclusion | \
    jq '[.[] | select(.name | test("test|spec|e2e|integration"; "i"))]')

  running=$(echo "$checks" | jq '[.[] | select(.state == "IN_PROGRESS" or .state == "QUEUED")] | length')
  failed=$(echo "$checks" | jq '[.[] | select(.conclusion == "FAILURE")] | length')
  passed=$(echo "$checks" | jq '[.[] | select(.conclusion == "SUCCESS")] | length')

  echo "Status: $passed passed, $failed failed, $running running"

  if [ "$running" -gt 0 ]; then
    echo "Waiting 1 minute..."
    sleep 60
    iteration=$((iteration + 1))
    continue
  fi

  if [ "$failed" -gt 0 ]; then
    echo "Fixing $failed failure(s)..."
    # Fix and restart loop
    break
  fi

  echo "All tests passing!"
  break
done
```

## Best Practices

1. **One commit per logical fix** - Group related fixes, separate unrelated ones
2. **Descriptive commit messages** - Clearly state what test was fixed and why
3. **Always include test location** - File and line number in commit message
4. **Verify fixes locally** - Run tests before pushing when possible
5. **Fix root cause, not symptoms** - Don't just change assertions to pass
6. **Minimal changes** - Don't refactor or add features while fixing tests
7. **Include Co-Authored-By** - Credit Claude in all commits
8. **Read failure logs carefully** - Understand before fixing
9. **Keep related changes together** - Fix test + source code in same commit if related
10. **Push after each fix round** - Push commits, then monitor CI for new failures
11. **Monitor continuously** - Poll CI every 1 minute until all tests pass
12. **Be patient with CI** - Wait for running checks to complete before analyzing failures
13. **Fix iteratively** - Each round of failures gets its own commits and push
14. **Set reasonable limits** - Stop after 10 fix rounds and request manual review
15. **Follow code style** - Use trailing commas in di/ and symbol_database/ (see CLAUDE.md)

## Safety Checks

Before pushing:
- Verify all fixed tests pass locally
- Check no unrelated files were modified
- Confirm changes are minimal and focused
- Review `git diff origin/branch HEAD` for unexpected changes
- Ensure no test was skipped or commented out

## Notes

- Test failures often indicate bugs in the source code, not just test issues
- Some test failures require understanding business logic
- **Flaky tests require root cause investigation** - Never mask with sleep(), retries, or skipping
- **Intermittent failures are real failures** - Investigate why they happen inconsistently
- If a fix seems too complex or risky, report for manual review
- Always prioritize correctness over speed
- When in doubt, ask for clarification on expected behavior
- **Continuous monitoring is essential** - initial fixes may not catch all failures
- CI environment often reveals issues not seen locally
- New test failures can appear after fixing others (cascading failures)
- Be patient - CI runs take time, poll at 1-minute intervals
- The skill keeps working until all tests pass or manual review is needed
