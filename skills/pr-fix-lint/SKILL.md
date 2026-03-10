---
name: pr-fix-lint
description: This skill should be used when the user asks to "fix lint", "fix CI", "fix static analysis", "fix typing errors", or mentions fixing non-test CI failures in a pull request.
version: 0.1.0
---

# PR Fix Lint

This skill automatically fixes non-test CI failures in a pull request, including linting, static analysis, and type checking issues. Each fix is committed separately with a descriptive message, and all commits are pushed at the end.

## Overview

The skill automates the process of:
1. Fetching a specific pull request
2. Checking CI job status to identify failures
3. Filtering for NON-TEST failures (lint, static analysis, typing)
4. Fixing each type of failure
5. Committing each fix with a descriptive message
6. Pushing all commits to the PR branch

## When This Skill Applies

Use this skill when:
- User asks to "fix lint failures in PR"
- User wants to "fix CI failures" (excluding test failures)
- User mentions "fix RuboCop", "fix static analysis", "fix typing errors"
- PR has non-test CI failures that can be automatically fixed
- User wants to clean up code quality issues before review

## Types of Failures Addressed

This skill ONLY addresses **non-test failures**:

### ✅ Addressed (Non-Test Failures)
- **Linting:** RuboCop, ESLint, Pylint, etc.
- **Static Analysis:** Brakeman, Sorbet, mypy, etc.
- **Type Checking:** TypeScript, Flow, Sorbet strict mode
- **Code Formatting:** Prettier, Black, gofmt, rubyfmt
- **Security Scanning:** CodeQL, bundler-audit (auto-fixable issues)
- **Style Checks:** Code style violations

### ❌ NOT Addressed (Test Failures)
- Unit test failures
- Integration test failures
- E2E test failures
- Spec failures (RSpec, Jest, pytest, etc.)
- Any job with "test" in the name
- Build failures that run tests

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
- Name does NOT contain "test", "spec", "e2e", "integration"

### Step 2: Identify Non-Test Failures

Look for failed checks matching these patterns:

**Linting:**
- "RuboCop", "rubocop", "lint"
- "ESLint", "eslint"
- "Pylint", "flake8", "ruff"
- "golangci-lint", "go lint"

**Static Analysis:**
- "Brakeman"
- "CodeQL"
- "Sorbet", "sorbet"
- "mypy", "pyright"
- "psalm", "phpstan"

**Type Checking:**
- "TypeScript", "tsc", "type-check"
- "Flow"
- "Sorbet" (strict mode)

**Formatting:**
- "Prettier", "prettier"
- "Black", "black"
- "gofmt", "goimports"
- "rubyfmt", "standardrb"

**Exclude these patterns:**
- Contains "test"
- Contains "spec"
- Contains "e2e"
- Contains "integration"
- Contains "build" + "test"
- Contains "Ruby X.X" (likely test jobs)

### Step 3: Analyze Each Non-Test Failure

For each identified non-test failure:

```bash
# Get the run ID from the check's link
run_id=$(gh pr checks <PR_NUMBER> --json name,link | \
  jq -r '.[] | select(.name == "<CHECK_NAME>") | .link | split("/") | .[-3]')

# View the failure logs
gh run view $run_id --log > /tmp/check_log_${CHECK_NAME}.txt

# Or use the details URL to view in browser if needed
details_url=$(gh pr checks <PR_NUMBER> --json name,detailsUrl | \
  jq -r '.[] | select(.name == "<CHECK_NAME>") | .detailsUrl')
```

Parse the logs to understand:
- What tool is failing (RuboCop, ESLint, mypy, etc.)
- What files have issues
- What specific violations are reported
- Whether the tool has an auto-fix option

### Step 4: Fix Each Type of Failure (Commit Separately)

**CRITICAL: Make ONE commit per fix type**

Each fix type should be:
1. Fixed completely
2. Committed with a descriptive message
3. Include Co-Authored-By: Claude line

#### Fix Linting Issues

**RuboCop (Ruby):**
```bash
# Auto-fix RuboCop violations
bundle exec rubocop -A

# Or for specific files
bundle exec rubocop -A path/to/file.rb

# Commit the fix
git add -A
git commit -m "$(cat <<'EOF'
Fix RuboCop violations

Auto-correct code style issues reported by RuboCop CI check.

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>
EOF
)"
```

**ESLint (JavaScript/TypeScript):**
```bash
# Auto-fix ESLint violations
npx eslint --fix .

# Or for specific files
npx eslint --fix path/to/file.js

# Commit the fix
git add -A
git commit -m "$(cat <<'EOF'
Fix ESLint violations

Auto-correct JavaScript/TypeScript style issues.

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>
EOF
)"
```

**Pylint/Ruff (Python):**
```bash
# Auto-fix with ruff
ruff check --fix .

# Or use black for formatting
black .

# Commit the fix
git add -A
git commit -m "$(cat <<'EOF'
Fix Python linting violations

Auto-correct code style issues reported by linting tools.

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>
EOF
)"
```

**Go linting:**
```bash
# Auto-fix with goimports
goimports -w .

# Run golangci-lint fix
golangci-lint run --fix

# Commit the fix
git add -A
git commit -m "$(cat <<'EOF'
Fix Go linting violations

Auto-correct Go code style and import issues.

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>
EOF
)"
```

#### Fix Formatting Issues

**Prettier:**
```bash
# Auto-format with Prettier
npx prettier --write .

# Commit the fix
git add -A
git commit -m "$(cat <<'EOF'
Fix code formatting with Prettier

Apply consistent code formatting across all files.

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>
EOF
)"
```

**Black (Python):**
```bash
# Format with Black
black .

# Commit the fix
git add -A
git commit -m "$(cat <<'EOF'
Fix Python formatting with Black

Apply consistent Python code formatting.

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>
EOF
)"
```

#### Fix Type Checking Issues

**TypeScript:**
```bash
# Analyze TypeScript errors
npx tsc --noEmit

# Read the error output and fix issues manually
# (Type errors usually require manual fixes)

# After fixing, commit
git add -A
git commit -m "$(cat <<'EOF'
Fix TypeScript type errors

Resolve type checking errors reported by tsc.

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>
EOF
)"
```

**Sorbet (Ruby):**
```bash
# Check Sorbet errors
bundle exec srb tc

# Fix type signatures and annotations
# (Manual fixes required for type errors)

# After fixing, commit
git add -A
git commit -m "$(cat <<'EOF'
Fix Sorbet type checking errors

Add missing type signatures and resolve type mismatches.

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>
EOF
)"
```

**mypy (Python):**
```bash
# Check mypy errors
mypy .

# Fix type hints
# (Manual fixes required)

# After fixing, commit
git add -A
git commit -m "$(cat <<'EOF'
Fix mypy type checking errors

Add type hints and resolve type inconsistencies.

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>
EOF
)"
```

#### Fix Static Analysis Issues

**Brakeman (Ruby Security):**
```bash
# Check Brakeman warnings
bundle exec brakeman

# Review and fix security issues
# (Manual fixes usually required)

# After fixing, commit
git add -A
git commit -m "$(cat <<'EOF'
Fix Brakeman security warnings

Address security vulnerabilities identified by static analysis.

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>
EOF
)"
```

### Step 5: Push All Commits

After all fixes are committed:

```bash
# Check that we have commits to push
git log origin/$(git branch --show-current)..HEAD --oneline

# Push all commits to the PR branch
git push origin HEAD
```

## Commit Message Format

Each commit should follow this format:

```
Fix <tool> <issue type>

<1-2 sentence description of what was fixed>

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>
```

Examples:
- `Fix RuboCop style violations`
- `Fix ESLint errors in components`
- `Fix TypeScript type checking errors`
- `Fix code formatting with Prettier`
- `Fix mypy type annotations`
- `Fix Sorbet type signatures`

## Implementation Steps

When the skill is invoked with a PR number:

1. **Checkout PR:**
   ```bash
   gh pr checkout <PR_NUMBER>
   ```

2. **Get failed checks:**
   ```bash
   failed_checks=$(gh pr checks <PR_NUMBER> --json name,state,link,detailsUrl | \
     jq -r '[.[] | select(.state == "FAILURE")] |
     map(select(.name | test("test|spec|e2e|integration|build.*test"; "i") | not))')
   ```

3. **For each non-test failure:**
   - Identify the tool (RuboCop, ESLint, etc.)
   - Get the failure logs
   - Determine if auto-fixable
   - Apply the appropriate fix command
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
     echo "No fixes were needed or all fixes failed"
   fi
   ```

## Decision Tree for Fixes

```
Is check failing?
  ├─ No → Skip
  └─ Yes → Does name contain "test"/"spec"/"e2e"?
      ├─ Yes → Skip (test failure, not in scope)
      └─ No → What tool is it?
          ├─ RuboCop → Run `rubocop -A` → Commit
          ├─ ESLint → Run `eslint --fix` → Commit
          ├─ Prettier → Run `prettier --write` → Commit
          ├─ Black → Run `black .` → Commit
          ├─ TypeScript → Analyze errors → Fix manually → Commit
          ├─ Sorbet → Analyze errors → Fix manually → Commit
          ├─ mypy → Analyze errors → Fix manually → Commit
          └─ Other → Investigate logs → Fix if possible → Commit
```

## Example Execution

```
$ /pr-fix-lint 4567

# PR Fix Lint - PR #4567

## Checking PR #4567: Add new feature
Branch: feature/new-thing
Base: main

## Checking CI Status...
Found 4 failed checks:
  ❌ RuboCop
  ❌ ESLint
  ✅ Test (Ruby 3.0) - Skipped (test job)
  ✅ Test (Ruby 2.7) - Skipped (test job)

## Fixing RuboCop violations...
Running: bundle exec rubocop -A
Fixed 23 violations in 5 files
✅ Committed: Fix RuboCop style violations

## Fixing ESLint errors...
Running: npx eslint --fix .
Fixed 12 violations in 3 files
✅ Committed: Fix ESLint errors in JavaScript files

## Summary
- Total checks analyzed: 4
- Test jobs skipped: 2
- Non-test failures fixed: 2
- Commits created: 2
- Commits pushed: 2

✅ All non-test CI failures have been fixed and pushed to the PR branch.
```

## Edge Cases

**Multiple failures of same type:**
- Fix all at once, make one commit
- Example: RuboCop failing in multiple jobs → one `rubocop -A` fixes all

**Tool not available in environment:**
- Report error: "Cannot fix RuboCop: bundle exec rubocop not found"
- Skip this fix, continue with others

**No auto-fix available:**
- For manual fixes (type errors, complex static analysis):
  - Read the error output
  - Analyze the issues
  - Make targeted fixes
  - Commit with detailed message explaining what was fixed

**No failures to fix:**
- Report: "No non-test CI failures found. All checks passing or only test failures present."
- Exit without pushing

**Fixes don't resolve CI failure:**
- Commit what was fixed anyway (shows effort)
- Report: "⚠️ Fixes applied but CI may still fail. Manual review needed."

**Protected branch (can't push):**
- Report error with instructions to configure branch protection
- Show the commits that were made locally

**Conflicts during push:**
- Report: "⚠️ Push failed due to conflicts. PR branch has been updated."
- Instruct: "Run `git pull --rebase` and push again manually."

## Required Tools

This skill requires:
- GitHub CLI (`gh`) authenticated
- Git configured
- Language-specific tools as needed:
  - Ruby: `bundle`, `rubocop`
  - JavaScript/TypeScript: `npm`/`yarn`, `eslint`, `prettier`, `tsc`
  - Python: `pip`, `black`, `ruff`, `mypy`
  - Go: `go`, `golangci-lint`, `goimports`

## Commands Reference

```bash
# Checkout PR
gh pr checkout <PR_NUMBER>

# Get failed non-test checks
gh pr checks <PR_NUMBER> --json name,state,link,detailsUrl | \
  jq '[.[] | select(.state == "FAILURE")] |
      map(select(.name | test("test|spec|e2e|integration"; "i") | not))'

# Common fix commands
bundle exec rubocop -A              # Ruby linting
npx eslint --fix .                  # JavaScript linting
npx prettier --write .              # Code formatting
black .                             # Python formatting
ruff check --fix .                  # Python linting
golangci-lint run --fix             # Go linting
npx tsc --noEmit                    # TypeScript checking

# Commit with heredoc (proper formatting)
git commit -m "$(cat <<'EOF'
Fix <tool> <issue>

Description of fix.

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>
EOF
)"

# Count new commits
git log origin/$(git branch --show-current)..HEAD --oneline | wc -l

# Push all commits
git push origin HEAD
```

## Best Practices

1. **One commit per tool/fix type** - Don't mix RuboCop and ESLint fixes in one commit
2. **Descriptive commit messages** - Clearly state what was fixed
3. **Auto-fix first** - Try automated fixes before manual intervention
4. **Verify fixes** - Run the tool again to confirm issues are resolved
5. **Push at the end** - Make all commits first, then push once
6. **Include Co-Authored-By** - Credit Claude in all commits
7. **Skip test failures** - Only fix non-test issues
8. **Report clearly** - Show what was fixed and what was skipped

## Safety Checks

Before pushing:
- Verify no test files were modified (unless fixing linting in tests)
- Confirm commits only contain formatting/style changes
- Check that no functionality was accidentally changed
- Run `git diff origin/branch HEAD` to review all changes

## Notes

- Always use auto-fix options when available (`-A`, `--fix`, `--write`)
- For manual fixes, be conservative - only fix what's clearly wrong
- If a fix is ambiguous or complex, skip it and report for manual review
- Some tools (like TypeScript) may require multiple iterations
- Be aware of tool configuration files (`.rubocop.yml`, `.eslintrc`, etc.)
- Respect the project's existing code style and conventions
