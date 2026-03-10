---
name: pr-fix-tests
description: This skill should be used when the user asks to "fix tests", "fix test failures", "fix failing specs", or mentions fixing test CI failures in a pull request.
version: 0.1.0
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

## Summary
- Total checks analyzed: 4
- Non-test jobs skipped: 1
- Test failures fixed: 3
- Commits created: 3
- Commits pushed: 3

✅ All test failures have been fixed and pushed to the PR branch.
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
- If test passes sometimes, investigate root cause
- Add proper synchronization or cleanup
- Don't just skip the test

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
```

## Best Practices

1. **One commit per logical fix** - Group related fixes, separate unrelated ones
2. **Descriptive commit messages** - Clearly state what test was fixed and why
3. **Always include test location** - File and line number in commit message
4. **Verify fixes locally** - Run tests before pushing
5. **Fix root cause, not symptoms** - Don't just change assertions to pass
6. **Minimal changes** - Don't refactor or add features while fixing tests
7. **Include Co-Authored-By** - Credit Claude in all commits
8. **Read failure logs carefully** - Understand before fixing
9. **Keep related changes together** - Fix test + source code in same commit if related
10. **Push at the end** - Make all commits first, then push once
11. **Follow code style** - Use trailing commas in di/ and symbol_database/ (see CLAUDE.md)

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
- Flaky tests should be fixed properly, not just ignored
- If a fix seems too complex or risky, report for manual review
- Always prioritize correctness over speed
- When in doubt, ask for clarification on expected behavior
