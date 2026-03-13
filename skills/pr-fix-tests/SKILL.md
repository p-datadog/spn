---
name: pr-fix-tests
description: This skill should be used when the user asks to "fix tests", "fix test failures", "fix failing specs", or mentions fixing test CI failures in a pull request.
version: 0.2.0
---

# PR Fix Tests

This skill automatically fixes test failures in a pull request. Each fix is committed separately with a descriptive message, and **all commits are automatically pushed to the PR branch**. The skill then continuously monitors CI and fixes any remaining failures.

## Overview

The skill automates the process of:
1. Fetching a specific pull request
2. **Reading CLAUDE.md** - Understanding repository coding guidelines
3. Checking CI job status to identify test failures
4. Filtering for TEST failures only (not lint/static analysis/typing)
5. Analyzing and fixing each test failure
6. **Applying CLAUDE.md rules** - Ensuring fixes follow all guidelines
7. Committing each fix with a descriptive message
8. Pushing all commits to the PR branch
9. **Continuously monitoring CI status** (polling every minute)
10. **Fixing any new or remaining test failures** until all tests pass

## Default Behavior

**IMPORTANT:** When invoked with a PR number or URL, this skill automatically:
1. ✅ Commits each fix separately
2. ✅ Pushes commits to the PR branch after each fix round
3. ✅ Monitors CI continuously and fixes new failures
4. ✅ No user confirmation required

This is the **default behavior**. Fixes are committed and pushed automatically until all tests pass.

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
- Linting failures (StandardRB, ESLint, Pylint)
- Static analysis (Brakeman, CodeQL)
- Type checking (TypeScript, Sorbet, mypy)
- Code formatting (Prettier, Black)
- Style checks

**Note:** dd-trace-rb uses StandardRB for linting (not RuboCop).
- **Infrastructure failures** (see below)

**Exception:** If fixing a test requires type changes, that's acceptable. But don't go out of your way to fix unrelated type/lint issues.

## Infrastructure Failures

**CRITICAL:** This skill automatically restarts infrastructure failures and does not attempt to fix them.

### What are Infrastructure Failures?

Infrastructure failures are CI job failures caused by GitHub Actions, runners, network, authentication, or other system issues rather than actual code problems.

**See:** `https://github.com/p-datadog/bells/blob/master/docs/infrastructure-failure-detection.md` for the complete list of patterns.

**Common examples:**
- GitHub Actions download failures (401, 403, 404, 429, 5xx errors)
- Git authentication errors (`fatal: could not read Username/Password`)
- Runner communication/termination issues
- Network timeouts and connectivity failures
- Resource exhaustion (disk space, memory)

### How to Handle Infrastructure Failures

1. **Detect** infrastructure failures by scanning job logs for patterns
2. **Restart** them immediately - don't attempt to fix infrastructure issues
3. **Report** them separately from test failures
4. **Focus** on actual test failures that need code fixes

**Infrastructure detection takes precedence:** Even if a job is named "Test XYZ", if it failed due to infrastructure issues, it should be restarted, not fixed.

**Automatic restart:** When infrastructure failures are detected, this skill automatically restarts them and moves on to fix actual test failures.

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

**Preferred approach for test failure analysis:**

1. **Always try JUnit artifacts FIRST** (Step 3.5)
   - Fast, structured data
   - Easy to parse
   - Complete failure information

2. **Fall back to log parsing** (Step 4) only if:
   - JUnit artifacts not available
   - Artifact download fails
   - Non-RSpec frameworks

**📋 Complete Workflow:**

1. Read repository guidelines (CLAUDE.md) and review requirements
2. Fetch PR and check CI status
3. Identify failed test jobs
4. **→ Check for infrastructure failures** (scan logs, skip if found)
5. **→ Download and analyze JUnit artifacts** (preferred, for non-infra failures)
6. → Fall back to log parsing (if artifacts unavailable)
7. Fix test failures (commit separately)
8. Push commits and monitor CI
9. Repeat until all tests pass

### Step 1: Read Repository Guidelines and Review Requirements

**CRITICAL:** Before making any fixes, read both the repository's coding guidelines and the review requirements.

**Read CLAUDE.md:**
```bash
# Read CLAUDE.md if it exists
if [ -f CLAUDE.md ]; then
  cat CLAUDE.md
fi
```

Key guidelines to look for:
- Trailing comma requirements
- Code style conventions
- Testing requirements (100% coverage for DI code)
- Error handling patterns
- Any project-specific rules

**Read di-pr-review skill requirements:**

Read `skills/di-pr-review/SKILL.md` to understand the quality standards that must be met:

**CRITICAL Requirements (must always follow):**
- ✅ **NO skipped tests** - Don't skip tests, fix them to pass or delete them
- ✅ **NO sleep in tests** - Use deterministic synchronization (Queue, ConditionVariable)
- ✅ **100% code coverage** - All new/changed code must have full test coverage
- ✅ **NO exception propagation** - Add error boundaries where needed
- ✅ **Proper error handling** - All rescue blocks need DEBUG/WARN logging + telemetry
- ✅ **NO hardcoded /tmp paths** - Use Dir.mktmpdir or Tempfile.create
- ✅ **Trailing commas** - Required in di/ and symbol_database/ directories

**Why read review requirements?**
When fixing tests, you must ensure your fixes comply with ALL review standards. A test fix that introduces sleep, skips tests, or lacks coverage will fail review even if it makes tests pass.

### Step 2: Fetch PR and Check CI Status

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

### Step 3: Identify Test Failures

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
- "StandardRB", "standard", "lint"
- "ESLint", "Prettier"
- "Brakeman", "CodeQL"
- "Sorbet", "mypy", "type-check"
- Pure formatting/linting jobs

**Note:** dd-trace-rb uses StandardRB (not RuboCop).

### Step 3.5: Check for Infrastructure Failures (Critical Filter)

**⚠️ IMPORTANT: Before attempting to fix test failures, check if they're infrastructure-related.**

Infrastructure failures should be **restarted**, not fixed. Skip any test jobs that failed due to infrastructure issues.

#### How to Detect Infrastructure Failures

For each failed test job, fetch a sample of the logs and scan for infrastructure patterns:

```bash
# Get the run ID from the check's link
run_id=$(gh pr checks <PR_NUMBER> --json name,link | \
  jq -r '.[] | select(.name == "<CHECK_NAME>") | .link | split("/") | .[-3]')

# Fetch the first ~500 lines of logs (infrastructure failures usually appear early)
gh run view $run_id --log 2>&1 | head -500 > /tmp/job_sample.txt

# Check for infrastructure failure patterns
if grep -qE '(fatal: could not read (Username|Password)|terminal prompts disabled|exit code 128|401 \(Unauthorized\)|403 \(Forbidden\)|404 \(Not Found\)|429.*rate limit|5[0-9]{2}.*Server Error|API rate limit exceeded|failed to download action|Unable to download|runner.*lost communication|runner.*terminated|Unable to resolve host|Connection timed out|Network is unreachable|Operation canceled|Connection reset|TLS handshake timeout|No space left on device|Out of memory|Disk quota exceeded)' /tmp/job_sample.txt; then
  echo "🔄 INFRASTRUCTURE FAILURE detected in <CHECK_NAME>"
  echo "   Restarting job automatically..."

  # Restart the failed job
  gh run rerun $run_id --failed

  echo "   ✅ Job restarted successfully"
  # Skip this job for fixing, move to next
fi
```

#### Infrastructure Failure Detection Pattern

**Source:** Patterns are based on `https://github.com/p-datadog/bells/blob/master/docs/infrastructure-failure-detection.md`

Use this regex pattern to detect infrastructure failures:

```regex
(fatal: could not read (Username|Password)|
 terminal prompts disabled|
 exit code 128|
 401 \(Unauthorized\)|
 403 \(Forbidden\)|
 404 \(Not Found\)|
 429.*rate limit|
 5[0-9]{2}.*Server Error|
 API rate limit exceeded|
 failed to download action|
 Unable to download|
 runner.*lost communication|
 runner.*terminated|
 Unable to resolve host|
 Connection timed out|
 Network is unreachable|
 Operation canceled|
 Connection reset|
 TLS handshake timeout|
 No space left on device|
 Out of memory|
 Disk quota exceeded)
```

This covers: GitHub Actions/API failures, Git authentication errors, runner failures, network issues, and resource exhaustion.

#### When to Skip vs Fix

**Skip (Infrastructure - restart instead):**
- GitHub Actions download failures
- Authentication errors during checkout
- Runner communication errors
- Network timeouts
- Resource exhaustion (disk, memory)
- HTTP 4xx/5xx errors from GitHub API

**Fix (Actual Test Failures - this skill's job):**
- RSpec/Jest/pytest assertion failures
- Test expectation mismatches
- Missing test setup
- Code bugs causing test failures
- Race conditions in test code
- Outdated test expectations

#### Example: Detecting Infrastructure Failure

```bash
# Failed job: "Test (Ruby 3.0)"
run_id=12345678

# Sample logs
gh run view $run_id --log 2>&1 | head -500

# Output contains:
# ##[error]fatal: could not read Username for 'https://github.com': terminal prompts disabled
# ##[error]The process '/usr/bin/git' failed with exit code 128

# Detection and restart:
echo "🔄 INFRASTRUCTURE FAILURE: Git authentication error"
echo "   Pattern matched: 'fatal: could not read Username'"
echo "   Restarting job automatically..."
gh run rerun $run_id --failed
echo "   ✅ Job restarted successfully"
# Don't analyze JUnit artifacts or logs for this job - move to next
```

#### Workflow After Detection

```bash
# Categorize each failed test job
infrastructure_jobs=()
test_failure_jobs=()

for job in $failed_test_jobs; do
  run_id=$(get_run_id "$job")

  # Sample logs and check for infrastructure patterns
  if gh run view $run_id --log 2>&1 | head -500 | \
     grep -qE '(fatal: could not read|401 \(Unauthorized\)|failed to download action|Connection timed out|No space left)'; then
    echo "🔄 INFRASTRUCTURE: $job (restarting)"
    infrastructure_jobs+=("$job")

    # Restart the job immediately
    echo "   Restarting $job (run $run_id)..."
    gh run rerun $run_id --failed
    echo "   ✅ Restarted successfully"
  else
    echo "🔍 TEST FAILURE: $job (will analyze and fix)"
    test_failure_jobs+=("$job")
  fi
done

# Report infrastructure failures
if [ ${#infrastructure_jobs[@]} -gt 0 ]; then
  echo ""
  echo "=== Infrastructure Failures Restarted ==="
  echo "The following jobs were restarted due to infrastructure issues:"
  for job in "${infrastructure_jobs[@]}"; do
    echo "  ✅ $job"
  done
  echo ""
fi

# Only proceed to fix test failures
echo "=== Test Failures to Fix ==="
for job in "${test_failure_jobs[@]}"; do
  echo "  - $job"
done
```

**Only proceed to Steps 4-7 for jobs in `test_failure_jobs` array.**

### Step 4: Download and Analyze JUnit Artifacts (Preferred Method)

**⚠️ IMPORTANT: Always try JUnit artifacts FIRST before parsing logs (for non-infrastructure failures).**

JUnit XML artifacts provide structured test failure data that's much easier to parse than unstructured logs. This is the PREFERRED method for analyzing test failures.

**Note:** This step is only for jobs that are NOT infrastructure failures. Infrastructure failures should have been filtered out in Step 3.5.

#### Why JUnit Artifacts are Better

**JUnit XML provides:**
- ✅ Structured failure data (easy to parse)
- ✅ All failures in one place (no log searching)
- ✅ Exact test names and locations
- ✅ Complete failure messages and stack traces
- ✅ Test timing information
- ✅ Faster than parsing multi-megabyte logs

**Raw logs require:**
- ❌ Parsing unstructured text
- ❌ Searching through megabytes of output
- ❌ Handling different log formats
- ❌ Finding test names in various formats
- ❌ Slow and error-prone

#### How JUnit Artifacts Work in dd-trace-rb

**During test runs:**
1. RSpec configured to generate JUnit XML (via `RspecJunitFormatter`)
2. Each test batch creates one XML file in `tmp/rspec/`
3. Files uploaded as GitHub Actions artifacts with names like `junit-ruby-27-standard-6`

**Artifact naming pattern:** `junit-{ruby-version}-{job-type}-{batch-number}`

Examples:
- `junit-ruby-34-standard-1`
- `junit-jruby-93-misc-2`
- `junit-ruby-27-standard-6`

#### Step-by-Step: Download and Parse JUnit Artifacts

**1. Get run ID from failed check:**
```bash
# Extract run ID from check link
run_id=$(gh pr checks <PR_NUMBER> --json name,link | \
  jq -r '.[] | select(.name == "<CHECK_NAME>") | .link' | \
  grep -oP 'runs/\K[0-9]+')

echo "Run ID: $run_id"
```

**2. List available JUnit artifacts:**
```bash
# List all junit artifacts for this run
gh api /repos/DataDog/dd-trace-rb/actions/runs/$run_id/artifacts --paginate | \
  jq -r '.artifacts[] | select(.name | startswith("junit-")) | "\(.name) (\(.size_in_bytes) bytes)"'
```

**Example output:**
```
junit-ruby-27-standard-6 (211374 bytes)
junit-ruby-34-standard-1 (458921 bytes)
```

**3. Download all JUnit artifacts:**
```bash
# Create temp directory for artifacts
temp_dir=$(mktemp -d)
cd "$temp_dir"

# Download all junit artifacts for this run
for artifact in $(gh api /repos/DataDog/dd-trace-rb/actions/runs/$run_id/artifacts --paginate | \
  jq -r '.artifacts[] | select(.name | startswith("junit-")) | .name'); do
  echo "Downloading $artifact..."
  gh run download $run_id --repo DataDog/dd-trace-rb --name "$artifact"
done

echo "Downloaded to: $temp_dir"
ls -lh
```

**4. Find XML files containing failures:**
```bash
# Find all XML files with failures
for xml in *.xml **/*.xml; do
  [ -f "$xml" ] || continue
  failures=$(grep -c '<failure' "$xml" 2>/dev/null || echo 0)
  errors=$(grep -c '<error' "$xml" 2>/dev/null || echo 0)

  if [ "$failures" -gt 0 ] || [ "$errors" -gt 0 ]; then
    echo "$failures failures, $errors errors: $xml"
  fi
done
```

**5. Extract failure details from XML:**
```bash
# Function to parse JUnit XML failures
parse_junit_failures() {
  local xml_file="$1"

  echo "=== Failures in $xml_file ==="

  # Extract test suite info
  grep '<testsuite' "$xml_file" | head -1

  # Extract each failure
  grep -A 20 '<failure' "$xml_file" | while read -r line; do
    if [[ "$line" =~ '<testcase' ]]; then
      # Extract test name and file
      name=$(echo "$line" | grep -oP 'name="\K[^"]+')
      file=$(echo "$line" | grep -oP 'file="\K[^"]+')
      echo ""
      echo "FAILED TEST: $name"
      echo "FILE: $file"
    elif [[ "$line" =~ '<failure' ]]; then
      # Extract failure message
      message=$(echo "$line" | grep -oP 'message="\K[^"]+')
      type=$(echo "$line" | grep -oP 'type="\K[^"]+')
      echo "ERROR TYPE: $type"
      echo "MESSAGE: $message"
    elif [[ "$line" =~ '</failure>' ]]; then
      echo "---"
    else
      # Stack trace line
      echo "$line"
    fi
  done
}

# Parse all XML files with failures
for xml in *.xml **/*.xml; do
  [ -f "$xml" ] || continue
  if grep -q '<failure' "$xml" 2>/dev/null; then
    parse_junit_failures "$xml"
  fi
done
```

**6. Group failures by pattern:**
```bash
# Extract just failure messages to identify patterns
echo "=== Failure Patterns ==="
grep -h -oP 'message="\K[^"]+' *.xml **/*.xml 2>/dev/null | sort | uniq -c | sort -rn
```

**Example output:**
```
     11 Test script produced unexpected output: /.../scope_context.rb:218: warning: assigned but unused variable - e
      3 Timeout::Error: execution expired after 5 seconds
      1 Expected 3 to equal 4
```

#### JUnit XML Structure Reference

**Test suite container:**
```xml
<testsuite name="spec.loading_spec" tests="20" failures="11" errors="0" time="7.234">
```

**Passing test:**
```xml
<testcase classname="spec.loading_spec"
          name="loading of products datadog/tracing loads successfully"
          file="./spec/loading_spec.rb"
          time="0.352308">
</testcase>
```

**Failing test:**
```xml
<testcase classname="spec.loading_spec"
          name="loading of products datadog/tracing produces no output"
          file="./spec/loading_spec.rb"
          time="0.352308">
  <failure message="Test script produced unexpected output: warning: unused variable - e"
           type="RuntimeError">
Failure/Error: raise("Test script produced unexpected output: #{out}") unless out.empty?

RuntimeError:
  Test script produced unexpected output: /.../scope_context.rb:218: warning: assigned but unused variable - e
./spec/loading_spec.rb:70:in `block (4 levels) in &lt;top (required)&gt;'
  </failure>
</testcase>
```

**Key fields:**
- `<testsuite>`: Group of tests (tests, failures, errors, time)
- `<testcase>`: Individual test (classname, name, file, time)
- `<failure>`: Failure details (message, type, + stack trace in inner text)

#### When JUnit Artifacts are NOT Available

**Fallback to log parsing only if:**
1. No JUnit artifacts found in the run
2. Artifact download fails
3. XML files are empty or corrupted

**In these cases, proceed to Step 5 (log analysis).**

#### Real Example: Ruby 2.7 Loading Test Failures

**Scenario:** 11 identical test failures in loading_spec.rb

**Using JUnit artifacts (FAST):**
```bash
# Download artifact
gh run download 22979950861 --name junit-ruby-27-standard-6

# Find failures (takes 2 seconds)
grep '<failure' *.xml | wc -l
# Output: 11

# Extract pattern (takes 1 second)
grep -oP 'message="\K[^"]+' *.xml | head -1
# Output: Test script produced unexpected output: /.../scope_context.rb:218: warning: assigned but unused variable - e

# Identify fix location immediately
grep 'scope_context.rb:218' *.xml
```

**Without JUnit artifacts (SLOW):**
```bash
# Download massive log file
gh run view 22979950861 --log > /tmp/log.txt  # 5MB, takes 30 seconds

# Search for failures (takes 20 seconds)
grep -n "FAILED\|Error\|Failure" /tmp/log.txt | wc -l
# Output: 847 lines (mostly noise)

# Try to find actual test failures (manual searching through 847 lines)
# Parse different test output formats
# Extract stack traces from unstructured text
# Identify which failures are related
# Manually trace back to source file
```

**Result:** JUnit analysis takes 3 seconds vs 5+ minutes of log parsing.

#### Checklist: Use JUnit Artifacts First

Before parsing logs, **ALWAYS:**
- [ ] Extract run ID from failed check
- [ ] List artifacts with `gh api .../artifacts`
- [ ] Look for `junit-*` artifact names
- [ ] Download JUnit artifacts if available
- [ ] Parse XML for failure details
- [ ] Group failures by pattern
- [ ] Identify root cause from structured data

**Only parse logs if JUnit artifacts are not available.**

### Step 5: Analyze Test Failures from Logs (Fallback Method)

**⚠️ Use this method ONLY if JUnit artifacts are not available (Step 4).**

**When to use log parsing:**
- No JUnit artifacts found in Step 4
- JUnit artifact download failed
- XML files are empty or corrupted
- Non-RSpec test frameworks without JUnit output

**Prefer Step 4 (JUnit artifacts) whenever possible - it's faster and more reliable.**

**Note:** This step is only for jobs that are NOT infrastructure failures. Infrastructure failures should have been filtered out in Step 3.5.

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

### Step 6: Fix Each Test Failure (Commit Separately)

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

### Step 7: Commit Each Fix

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

### Step 8: Push All Commits

After all fixes are committed:

```bash
# Verify commits were made
git log origin/$(git branch --show-current)..HEAD --oneline

# Push all commits to the PR branch
git push origin HEAD
```

### Step 9: Continuously Monitor CI and Fix Remaining Failures

**IMPORTANT:** After pushing fixes, continuously monitor CI status and fix any remaining or new test failures.

**Why this is needed:**
- Initial fixes may not catch all issues
- CI environment may reveal failures not reproducible locally
- Some tests may only fail in specific CI configurations
- Flaky tests may fail intermittently (investigate root cause, don't mask with workarounds)
- New failures may appear after fixing others

**Monitoring process:**

**⚠️ IMPORTANT: Bash Variable Naming**

Avoid bash reserved variable names to prevent "read-only variable" errors:
- ❌ **DON'T use:** `status`, `PATH`, `HOME`, `USER`, `SHELL`, `PWD`, `RANDOM`, `LINENO`
- ✅ **DO use:** `ci_status`, `check_results`, `checks`, `ci_checks`

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
  # IMPORTANT: Using 'checks' variable (safe), NOT 'status' (reserved in bash)
  checks=$(gh pr checks <PR_NUMBER> --json name,state,link,detailsUrl | \
    jq '[.[] | select(.name | test("test|spec|e2e|integration|build.*test"; "i"))]')

  # Count running checks
  running=$(echo "$checks" | jq '[.[] | select(.state == "IN_PROGRESS" or .state == "QUEUED" or .state == "PENDING")] | length')

  # Count failed test checks
  failed=$(echo "$checks" | jq '[.[] | select(.state == "FAILURE")] | length')

  # Count successful test checks
  passed=$(echo "$checks" | jq '[.[] | select(.state == "SUCCESS")] | length')

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
    failed_checks=$(echo "$checks" | jq -r '.[] | select(.state == "FAILURE") | .name')

    echo "Failed checks:"
    echo "$failed_checks"

    # IMPORTANT: Filter out infrastructure failures
    # Check each failed job for infrastructure patterns
    # Only fix jobs that are actual test failures, not infrastructure issues
    # (Infrastructure failures should be restarted via /crcj)

    # Analyze and fix each failure (same process as initial fixes)
    # ... (follow Steps 3.5-7 again for new failures)

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
- ✅ All test checks pass (state == "SUCCESS")
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

**IMPORTANT: Prefer targeted test runs over full suite**

Running the full `spec:main` suite takes a long time. Always prefer:
1. **Individual test examples** (specific line number)
2. **Single test files** (one spec file)
3. **Component-scoped tests** (spec:di, spec:telemetry, etc.)

Only run the full `spec:main` suite if you've made widespread changes affecting multiple components.

**Ruby (RSpec):**
```bash
# PREFERRED: Run specific test example (fastest)
bundle exec rspec spec/datadog/di/probe_spec.rb:45

# PREFERRED: Run specific test file
bundle exec rspec spec/datadog/di/probe_spec.rb

# PREFERRED: Run component-scoped tests (for DI changes)
bundle exec rspec spec/datadog/di/

# PREFERRED: Run tests matching a pattern
bundle exec rspec spec/datadog/di/ -e "probe execution"

# LAST RESORT: Run all core tests (SLOW - only if necessary)
bundle exec rake spec:main

# Note: dd-trace-rb uses rake tasks as the primary test entry point
# CI runs: bundle exec rake spec:main, spec:redis, spec:profiling, etc.
# But for local verification, prefer targeted runs for speed
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

3. **Detect and restart infrastructure failures:**
   ```bash
   # For each failed test check, scan logs for infrastructure patterns
   infrastructure_jobs=()
   test_failure_jobs=()

   for job in $failed_tests; do
     run_id=$(get_run_id "$job")

     # Sample first 500 lines of logs
     if gh run view $run_id --log 2>&1 | head -500 | \
        grep -qE '(fatal: could not read|401 \(Unauthorized\)|failed to download action|Connection timed out|No space left)'; then
       echo "🔄 INFRASTRUCTURE: $job (restarting)"
       infrastructure_jobs+=("$job")

       # Restart immediately
       gh run rerun $run_id --failed
       echo "   ✅ Restarted successfully"
     else
       test_failure_jobs+=("$job")
     fi
   done

   # Report infrastructure failures
   if [ ${#infrastructure_jobs[@]} -gt 0 ]; then
     echo "=== Infrastructure Failures (restarted) ==="
     printf '  ✅ %s\n' "${infrastructure_jobs[@]}"
   fi
   ```

4. **For each non-infrastructure test failure:**
   - Download and analyze JUnit artifacts (preferred) or logs
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
     checks=$(gh pr checks <PR_NUMBER> --json name,state | \
       jq '[.[] | select(.name | test("test|spec|e2e|integration"; "i"))]')

     running=$(echo "$checks" | jq '[.[] | select(.state == "IN_PROGRESS" or .state == "QUEUED")] | length')
     failed=$(echo "$checks" | jq '[.[] | select(.state == "FAILURE")] | length')

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

       # Filter out infrastructure failures first (Step 3)
       # Then repeat steps 3.5-8 for new non-infrastructure failures
       # (check infra, analyze, fix, commit, push)

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
      └─ Yes → Check logs for infrastructure patterns
          ├─ Infrastructure failure detected → Skip (restart via /crcj)
          └─ Not infrastructure → Analyze test failure
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
Found 4 failed checks:
  ❌ Test (Ruby 3.0) - Infrastructure failure
  ❌ Test (Ruby 2.7) - 2 failing tests
  ❌ Test (Ruby 3.1) - 2 failing tests
  ❌ Jest Tests - 1 failing test
  ✅ StandardRB - Skipped (non-test job)

## Checking for infrastructure failures...
🔄 INFRASTRUCTURE FAILURE: Test (Ruby 3.0)
   Pattern matched: 'fatal: could not read Username'
   Restarting job automatically...
   ✅ Job restarted successfully

## Test failures to fix (non-infrastructure):
  🔍 Test (Ruby 2.7)
  🔍 Test (Ruby 3.1)
  🔍 Jest Tests

## Analyzing Test (Ruby 2.7) failures...

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
- Total checks analyzed: 5
- Non-test jobs skipped: 1
- Infrastructure failures restarted: 1
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
- Suggest running affected tests: "Run `bundle exec rspec spec/path/to/affected_spec.rb` or `bundle exec rspec spec/datadog/component/` to verify locally"

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

**Infrastructure failures appear during monitoring:**
- Skip them - don't attempt to fix infrastructure issues
- Infrastructure failures should be restarted, not fixed
- Report them separately from test failures
- Continue monitoring other test jobs

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

# Run tests locally (prefer targeted runs for speed)
bundle exec rspec spec/path/to/test_spec.rb:45    # Ruby - specific line (FASTEST, preferred)
bundle exec rspec spec/path/to/test_spec.rb       # Ruby - entire file (preferred)
bundle exec rspec spec/datadog/di/                # Ruby - component directory (preferred)
bundle exec rake spec:main                        # Ruby - all core tests (SLOW, avoid unless necessary)
npm test path/to/test.test.js                     # JavaScript - specific file
pytest tests/path/to/test_file.py::test_name      # Python - specific test

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

# Check current CI status
gh pr checks <PR_NUMBER> --json name,state,link,detailsUrl

# Get test checks only (filter by name pattern)
gh pr checks <PR_NUMBER> --json name,state | \
  jq '[.[] | select(.name | test("test|spec|e2e|integration|build.*test"; "i"))]'

# Count running test checks
gh pr checks <PR_NUMBER> --json name,state | \
  jq '[.[] | select(.name | test("test|spec|e2e|integration"; "i"))] |
      [.[] | select(.state == "IN_PROGRESS" or .state == "QUEUED" or .state == "PENDING")] | length'

# Count failed test checks
gh pr checks <PR_NUMBER> --json name,state | \
  jq '[.[] | select(.name | test("test|spec|e2e|integration"; "i"))] |
      [.[] | select(.state == "FAILURE")] | length'

# Count passed test checks
gh pr checks <PR_NUMBER> --json name,state | \
  jq '[.[] | select(.name | test("test|spec|e2e|integration"; "i"))] |
      [.[] | select(.state == "SUCCESS")] | length'

# List names of failed test checks
gh pr checks <PR_NUMBER> --json name,state | \
  jq -r '[.[] | select(.name | test("test|spec|e2e|integration"; "i"))] |
         [.[] | select(.state == "FAILURE")] | .[].name'

# Check PR status (to detect if merged or closed)
gh pr view <PR_NUMBER> --json state,merged | jq -r '.state, .merged'

# Full monitoring loop example
iteration=1
while true; do
  echo "=== Check #$iteration at $(date) ==="

  checks=$(gh pr checks <PR_NUMBER> --json name,state | \
    jq '[.[] | select(.name | test("test|spec|e2e|integration"; "i"))]')

  running=$(echo "$checks" | jq '[.[] | select(.state == "IN_PROGRESS" or .state == "QUEUED")] | length')
  failed=$(echo "$checks" | jq '[.[] | select(.state == "FAILURE")] | length')
  passed=$(echo "$checks" | jq '[.[] | select(.state == "SUCCESS")] | length')

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

## Applying Review Requirements to Test Fixes

**CRITICAL:** Every test fix must comply with ALL review requirements from di-pr-review skill.

### When Fixing Test Failures

**Avoid introducing new violations:**
- ❌ Don't add sleep to make tests pass
- ❌ Don't skip tests that are hard to fix
- ❌ Don't use hardcoded /tmp paths in fixes
- ❌ Don't leave code uncovered by tests

**Ensure compliance:**
- ✅ Use Queue, ConditionVariable, or mock time instead of sleep
- ✅ Fix skipped tests to pass or delete them
- ✅ Use Dir.mktmpdir or Tempfile.create for temp files
- ✅ Add test coverage for any new code added
- ✅ Use trailing commas in di/ and symbol_database/

### When Fixing Source Code (to make tests pass)

If fixing source code to make tests pass:

**Add error boundaries if needed:**
```ruby
# If adding code that could propagate exceptions
TracePoint.new(:call) do |tp|
  begin
    probe.execute(tp)  # Could raise
  rescue => e
    Datadog.logger.debug("Probe execution failed: #{e}")
    Datadog.telemetry.error('probe_execution_failed', error: e)
  end
end
```

**Add proper error handling:**
```ruby
# All rescues need logging + telemetry
begin
  process_snapshot
rescue => e
  Datadog.logger.debug("Snapshot processing failed: #{e}")
  Datadog.telemetry.error('snapshot_processing_failed', error: e)
end
```

**Use trailing commas in di/ and symbol_database/:**
```ruby
# ✅ Correct
config = {
  timeout: 30,
  retries: 3,
  enabled: true,
}

# ❌ Wrong in di/ directories
config = {
  timeout: 30,
  retries: 3,
  enabled: true
}
```

**No hardcoded /tmp paths:**
```ruby
# ✅ Correct
Dir.mktmpdir('di_test') do |dir|
  file = File.join(dir, 'data.json')
end

# ❌ Wrong
file = '/tmp/di_test/data.json'
```

### Verification Checklist

Before committing each fix, verify:
- [ ] Test fix doesn't introduce sleep
- [ ] Test fix doesn't skip tests
- [ ] All code has test coverage
- [ ] No hardcoded /tmp paths added
- [ ] Trailing commas used in di/ and symbol_database/
- [ ] Error boundaries added where needed
- [ ] Rescue blocks have logging + telemetry
- [ ] Tests pass locally

## Best Practices

1. **One commit per logical fix** - Group related fixes, separate unrelated ones
2. **Descriptive commit messages** - Clearly state what test was fixed and why
3. **Always include test location** - File and line number in commit message
4. **Verify fixes locally** - Run tests before pushing when possible
5. **Run targeted tests, not full suite** - Prefer specific test examples, single files, or component directories (spec/datadog/di/) over spec:main which is slow
6. **Fix root cause, not symptoms** - Don't just change assertions to pass
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
- Verify all fixed tests pass locally (use targeted test runs for speed)
- Check no unrelated files were modified
- Confirm changes are minimal and focused
- Review `git diff origin/branch HEAD` for unexpected changes
- Ensure no test was skipped or commented out

## Notes

- Test failures often indicate bugs in the source code, not just test issues
- Some test failures require understanding business logic
- **Prefer targeted test runs** - Run specific examples, files, or component directories instead of the full spec:main suite which takes a long time
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
