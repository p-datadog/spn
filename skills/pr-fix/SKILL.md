---
name: pr-fix
description: This skill should be used when the user asks to "fix PR", "pr fix <number>", "fix pr <number>", or mentions fixing a specific PR in dd-trace-rb without specifying what type of failure to fix.
version: 0.1.0
---

# PR Fix (Smart Router)

This skill intelligently investigates CI failures in a dd-trace-rb PR and delegates to the appropriate specialized fix skills based on what actually failed.

## Overview

The skill automates the process of:
1. Checking CI status for the specified PR
2. Categorizing failures by type (tests, linting, type checking, etc.)
3. Delegating to appropriate specialized skills in priority order
4. Providing clear feedback about what's being fixed
5. Handling edge cases (no failures, multiple failure types, etc.)

## When This Skill Applies

Use this skill when:
- User says "fix PR #5448"
- User says "pr fix 5448"
- User says "fix pr 5448"
- User wants to fix a PR but doesn't specify what type of failure
- Context is dd-trace-rb repository

**Do NOT use this skill when:**
- User specifies what to fix: "fix tests in PR 5448" → use `/pr-fix-tests` directly
- User wants to review: "review PR 5448" → use `/di-pr-review` directly
- User wants to respond to comments: "respond to PR 5448 comments" → use `/di-pr-respond` directly
- Batch processing p-datadog PRs: "check all p-datadog PRs" → use `/crcj` directly
- Not in dd-trace-rb context

## Default Behavior

**IMPORTANT:** This skill is a ROUTER, not a fixer itself. It:
1. ✅ Investigates CI failures
2. ✅ Delegates to specialized skills automatically
3. ✅ No user confirmation required for delegation
4. ❌ Does NOT fix issues directly

This is the **default behavior**. The skill determines what needs fixing and delegates automatically.

## Workflow

### Step 1: Validate Context

Verify this is a dd-trace-rb PR:

```bash
# Get PR information
pr_info=$(gh pr view <PR_NUMBER> --repo DataDog/dd-trace-rb --json number,title,url,state)

# Check if PR exists
if [ $? -ne 0 ]; then
  echo "Error: PR #<PR_NUMBER> not found in DataDog/dd-trace-rb"
  exit 1
fi

# Check if PR is open
state=$(echo "$pr_info" | jq -r '.state')
if [ "$state" != "OPEN" ]; then
  echo "Warning: PR #<PR_NUMBER> is $state (not OPEN)"
fi

# Display PR info
echo "PR #<PR_NUMBER>: $(echo "$pr_info" | jq -r '.title')"
```

### Step 2: Check CI Status

Get all CI check results:

**⚠️ IMPORTANT: Bash Variable Naming**

Avoid bash reserved variable names to prevent "read-only variable" errors:
- ❌ **DON'T use:** `status`, `PATH`, `HOME`, `USER`, `SHELL`, `PWD`, `RANDOM`
- ✅ **DO use:** `ci_status`, `check_results`, `ci_checks`, `test_status`

```bash
# Get all checks with their state
# Note: Using 'checks' variable (safe), NOT 'status' (reserved)
checks=$(gh pr checks <PR_NUMBER> --repo DataDog/dd-trace-rb --json name,state,link)

# Count failures
total_checks=$(echo "$checks" | jq 'length')
failed_checks=$(echo "$checks" | jq '[.[] | select(.state == "FAILURE")] | length')
pending_checks=$(echo "$checks" | jq '[.[] | select(.state == "PENDING" or .state == "QUEUED")] | length')
passed_checks=$(echo "$checks" | jq '[.[] | select(.state == "SUCCESS")] | length')

echo "CI Status:"
echo "  Total checks: $total_checks"
echo "  Failed: $failed_checks"
echo "  Pending: $pending_checks"
echo "  Passed: $passed_checks"
```

### Step 3: Handle Edge Cases

**Case 1: No failures (all passing or pending)**

```bash
if [ "$failed_checks" -eq 0 ]; then
  if [ "$pending_checks" -gt 0 ]; then
    echo "✅ No failures found. $pending_checks check(s) still running."
    echo "Wait for checks to complete before fixing."
    exit 0
  else
    echo "✅ All checks passing! Nothing to fix."
    exit 0
  fi
fi
```

**Case 2: No checks found**

```bash
if [ "$total_checks" -eq 0 ]; then
  echo "⚠️ No CI checks found for this PR."
  echo "Possible reasons:"
  echo "  - PR is too new (checks not started yet)"
  echo "  - PR has no commits"
  echo "  - CI workflows not configured for this branch"
  exit 0
fi
```

### Step 4: Categorize Failures

Categorize each failed check into one of these types:

**Test Failures:**
- Name contains: "test", "spec", "rspec", "jest", "e2e", "integration"
- Name contains: "build & test", "build and test"
- Examples: "Ruby 2.7 / build & test", "Integration tests", "E2E specs"

**Linting Failures:**
- Name contains: "standard", "lint", "eslint", "style"
- Name contains: "prettier", "format"
- Examples: "StandardRB", "Standard", "Linting", "Code style"

**Type Checking Failures:**
- Name contains: "steep", "sorbet", "typecheck", "type-check"
- Name contains: "mypy", "typescript"
- Examples: "Steep", "Type checking"

**Other Failures:**
- Everything else (security scans, dependency checks, etc.)

```bash
# Categorize failures
# IMPORTANT: Use file-based jq filters to avoid shell quoting issues

filter_test=$(mktemp)
filter_lint=$(mktemp)
filter_type=$(mktemp)

cat > "$filter_test" << 'EOF'
[.[] | select(.state == "FAILURE") | select(.name | test("test|spec|rspec|jest|e2e|integration|build.*test"; "i"))]
| length
EOF

cat > "$filter_lint" << 'EOF'
[.[] | select(.state == "FAILURE") | select(.name | test("standard|standardrb|lint|eslint|style|prettier|format"; "i"))]
| length
EOF

cat > "$filter_type" << 'EOF'
[.[] | select(.state == "FAILURE") | select(.name | test("steep|sorbet|typecheck|type-check|mypy|typescript"; "i"))]
| length
EOF

test_failures=$(echo "$checks" | jq -r -f "$filter_test")
lint_failures=$(echo "$checks" | jq -r -f "$filter_lint")
type_failures=$(echo "$checks" | jq -r -f "$filter_type")
other_failures=$((failed_checks - test_failures - lint_failures - type_failures))

rm -f "$filter_test" "$filter_lint" "$filter_type"

echo "Failure breakdown:"
echo "  Test failures: $test_failures"
echo "  Linting failures: $lint_failures"
echo "  Type checking failures: $type_failures"
echo "  Other failures: $other_failures"
```

### Step 5: Determine Delegation Strategy

**Priority order:**
1. **Tests first** - Must pass before anything else matters
2. **Linting/type checking** - After tests pass
3. **Other issues** - Manual investigation needed

**Decision tree:**

```bash
if [ "$test_failures" -gt 0 ]; then
  echo "🎯 Detected test failures. Delegating to pr-fix-tests skill..."
  # Delegate to pr-fix-tests
  SKILL_TO_USE="pr-fix-tests"

elif [ "$lint_failures" -gt 0 ] || [ "$type_failures" -gt 0 ]; then
  echo "🎯 Detected linting/type failures. Delegating to pr-fix-lint skill..."
  # Delegate to pr-fix-lint
  SKILL_TO_USE="pr-fix-lint"

elif [ "$other_failures" -gt 0 ]; then
  echo "⚠️ Detected failures that require manual investigation:"
  echo "$checks" | jq -r '.[] | select(.state == "FAILURE") | "  - \(.name): \(.link)"'
  echo ""
  echo "These failures don't match known patterns (tests/linting/type checking)."
  echo "Please investigate manually or specify which skill to use."
  exit 0
fi
```

**Special case: Multiple failure types**

```bash
if [ "$test_failures" -gt 0 ] && ([ "$lint_failures" -gt 0 ] || [ "$type_failures" -gt 0 ]); then
  echo "📋 Multiple failure types detected:"
  echo "  - $test_failures test failure(s)"
  [ "$lint_failures" -gt 0 ] && echo "  - $lint_failures linting failure(s)"
  [ "$type_failures" -gt 0 ] && echo "  - $type_failures type checking failure(s)"
  echo ""
  echo "Strategy: Fix tests FIRST, then linting/types."
  echo "Reason: Tests must pass before code quality checks matter."
  echo ""
  echo "🎯 Starting with test failures..."
  SKILL_TO_USE="pr-fix-tests"
fi
```

### Step 6: Delegate to Appropriate Skill

**Invoke the determined skill:**

```bash
case "$SKILL_TO_USE" in
  "pr-fix-tests")
    echo "Invoking: /pr-fix-tests <PR_NUMBER>"
    # Use Skill tool to invoke pr-fix-tests
    ;;

  "pr-fix-lint")
    echo "Invoking: /pr-fix-lint <PR_NUMBER>"
    # Use Skill tool to invoke pr-fix-lint
    ;;
esac
```

**After delegation:**

The invoked skill will:
- Fix the issues
- Commit changes
- Push to PR branch
- Monitor CI
- Report results

This skill's job is complete after successful delegation.

## Output Format

### Successful Delegation

```
Analyzing PR #5448...
📋 PR #5448: Add dynamic instrumentation support for method parameters

CI Status:
  Total checks: 15
  Failed: 3
  Pending: 0
  Passed: 12

Failure breakdown:
  Test failures: 3
  Linting failures: 0
  Type checking failures: 0
  Other failures: 0

🎯 Detected test failures. Delegating to pr-fix-tests skill...

Invoking: /pr-fix-tests 5448
```

### Multiple Failure Types

```
Analyzing PR #5448...
📋 PR #5448: Add dynamic instrumentation support

CI Status:
  Total checks: 15
  Failed: 5
  Pending: 0
  Passed: 10

Failure breakdown:
  Test failures: 3
  Linting failures: 2
  Type checking failures: 0
  Other failures: 0

📋 Multiple failure types detected:
  - 3 test failure(s)
  - 2 linting failure(s)

Strategy: Fix tests FIRST, then linting.
Reason: Tests must pass before code quality checks matter.

🎯 Starting with test failures...

Invoking: /pr-fix-tests 5448

Note: After tests pass, run '/pr-fix-lint 5448' to fix linting issues.
```

### No Failures

```
Analyzing PR #5448...
📋 PR #5448: Add dynamic instrumentation support

CI Status:
  Total checks: 15
  Failed: 0
  Pending: 0
  Passed: 15

✅ All checks passing! Nothing to fix.
```

### Pending Checks

```
Analyzing PR #5448...
📋 PR #5448: Add dynamic instrumentation support

CI Status:
  Total checks: 15
  Failed: 0
  Pending: 5
  Passed: 10

✅ No failures found. 5 check(s) still running.
Wait for checks to complete before fixing.
```

### Unknown Failure Types

```
Analyzing PR #5448...
📋 PR #5448: Add dynamic instrumentation support

CI Status:
  Total checks: 15
  Failed: 2
  Pending: 0
  Passed: 13

Failure breakdown:
  Test failures: 0
  Linting failures: 0
  Type checking failures: 0
  Other failures: 2

⚠️ Detected failures that require manual investigation:
  - Security scan: https://github.com/DataDog/dd-trace-rb/runs/12345
  - Dependency audit: https://github.com/DataDog/dd-trace-rb/runs/12346

These failures don't match known patterns (tests/linting/type checking).
Please investigate manually or specify which skill to use.
```

## Skills This Router Can Delegate To

| Skill | When Used | What It Does |
|-------|-----------|--------------|
| `pr-fix-tests` | Test failures detected | Analyzes test failures, fixes code, monitors CI |
| `pr-fix-lint` | Linting or type checking failures | Fixes standard/linting violations, type errors |

**NOT delegated to:**
- `crcj` - Only for batch p-datadog PR maintenance (not single PR fixes)
- `di-pr-review` - Manual review, not automatic fixing
- `di-pr-respond` - For responding to review comments, not CI failures

## Decision Logic Summary

```
Is it a dd-trace-rb PR?
  No → Error: "Not a dd-trace-rb PR"
  Yes ↓

Get CI status
  No checks found → "No CI checks found"
  No failures, some pending → "Wait for checks to complete"
  No failures, all passed → "All checks passing!"
  Failures found ↓

Categorize failures:
  Test failures? → /pr-fix-tests
  Lint/type failures? → /pr-fix-lint
  Other failures? → "Requires manual investigation"

Multiple types?
  Tests + Lint → /pr-fix-tests (then suggest /pr-fix-lint after)
  Tests + Type → /pr-fix-tests (then suggest /pr-fix-lint after)
  Lint + Type → /pr-fix-lint
```

## Important Notes

**This skill is a router, not a fixer:**
- ✅ Investigates what failed
- ✅ Delegates to appropriate skill
- ❌ Does not fix issues directly

**Priority order matters:**
- Tests MUST pass before linting matters
- If both failing, fix tests first
- Inform user about remaining issues

**Edge cases handled:**
- No failures → Nothing to do
- Pending checks → Wait
- Unknown failures → Manual investigation needed
- Multiple types → Fix in priority order

**NOT a replacement for:**
- Manual code review (`di-pr-review`)
- Responding to feedback (`di-pr-respond`)
- Batch maintenance (`crcj`)

## Example Invocations

### User says: "pr fix 5448"
```
This skill activates → Checks CI → Delegates to pr-fix-tests or pr-fix-lint
```

### User says: "fix PR 5448"
```
This skill activates → Checks CI → Delegates appropriately
```

### User says: "fix tests in PR 5448"
```
This skill SHOULD NOT activate → pr-fix-tests activates directly
```

### User says: "check all p-datadog PRs"
```
This skill SHOULD NOT activate → crcj activates directly
```

## Success Criteria

A successful execution includes:
- ✅ PR validated and exists
- ✅ CI status checked
- ✅ Failures categorized correctly
- ✅ Appropriate skill delegated to
- ✅ Clear feedback provided to user
- ✅ Edge cases handled gracefully

## Failure Modes

**What this skill cannot handle:**
- PRs outside dd-trace-rb (wrong repository)
- Security scan failures (requires manual review)
- Dependency vulnerability failures (requires manual review)
- Infrastructure failures (not code issues)
- Merge conflicts (different workflow)

**In these cases:**
- Provide clear message about what cannot be fixed
- Suggest manual investigation
- Show relevant links/information
