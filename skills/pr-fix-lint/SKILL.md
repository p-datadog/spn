---
name: pr-fix-lint
description: This skill should be used when the user asks to "fix lint", "fix CI", "fix static analysis", "fix typing errors", "fix type check", "fix steep", "fix standard", or mentions fixing non-test CI failures in a pull request.
version: 0.3.0
---

# PR Fix Lint

This skill automatically fixes non-test CI failures in a pull request, including linting, static analysis, and type checking issues. Each fix is committed separately with a descriptive message, and **all commits are automatically pushed to the PR branch at the end**.

## Overview

The skill automates the process of:
1. Fetching a specific pull request
2. **Reading CLAUDE.md** - Understanding repository coding guidelines
3. Checking CI job status to identify failures
4. Filtering for NON-TEST failures (lint, static analysis, typing)
5. Fixing each type of failure
6. **Applying CLAUDE.md rules** - Ensuring fixes follow all guidelines
7. Committing each fix with a descriptive message
8. Pushing all commits to the PR branch

## Default Behavior

**IMPORTANT:** When invoked with a PR number or URL, this skill automatically:
1. ✅ Commits each fix separately
2. ✅ Pushes all commits to the PR branch
3. ✅ No user confirmation required

This is the **default behavior**. Fixes are committed and pushed automatically.

## When This Skill Applies

Use this skill when:
- User asks to "fix lint failures in PR"
- User wants to "fix CI failures" (excluding test failures)
- User mentions "fix StandardRB", "fix standard", "fix static analysis", "fix typing errors", "fix type check", "fix steep"
- PR has non-test CI failures that can be automatically fixed
- User wants to clean up code quality issues before review

## Types of Failures Addressed

This skill ONLY addresses **non-test failures**:

### ✅ Addressed (Non-Test Failures)
- **Linting:** StandardRB, ESLint, Pylint, etc.
- **Static Analysis:** Brakeman, Sorbet, mypy, etc.
- **Type Checking:** TypeScript, Flow, Sorbet, Steep (Ruby), mypy (Python)
- **Code Formatting:** Prettier, Black, gofmt, standardrb
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

### Step 1: Read Repository Guidelines

**CRITICAL:** Before making any fixes, read the repository's CLAUDE.md file to understand coding standards and guidelines.

```bash
# Read CLAUDE.md if it exists
if [ -f CLAUDE.md ]; then
  cat CLAUDE.md
fi
```

Key guidelines to look for:
- Trailing comma requirements
- Code style conventions
- Testing requirements
- Error handling patterns
- Any project-specific rules

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
- Name does NOT contain "test", "spec", "e2e", "integration"

### Step 3: Identify Non-Test Failures

Look for failed checks matching these patterns:

**Linting:**
- "StandardRB", "standard", "lint"
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

### Step 4: Analyze Each Non-Test Failure

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
- What tool is failing (StandardRB, ESLint, mypy, etc.)
- What files have issues
- What specific violations are reported
- Whether the tool has an auto-fix option

### Step 5: Fix Each Type of Failure (Commit Separately)

**CRITICAL: Make ONE commit per fix type**

Each fix type should be:
1. Fixed completely
2. Committed with a descriptive message
3. Include Co-Authored-By: Claude line

#### Fix Linting Issues

**StandardRB (Ruby):**
```bash
# Auto-fix StandardRB violations
bundle exec rake standard:fix

# Or run check first to see issues
bundle exec rake standard

# Commit the fix
git add -A
git commit -m "$(cat <<'EOF'
Fix StandardRB violations

Auto-correct code style issues reported by StandardRB CI check.

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

**Steep (Ruby):**

**CRITICAL RULES FOR STEEP:**

1. **Fix genuine code issues** - If type error reveals a real bug, fix the code
2. **Silence unclear errors** - If unclear whether it's genuine, add steep ignore directives
3. **NEVER change code only to satisfy steep** - Don't refactor just to make errors go away
4. **Nil checks** - When steep thinks a value is possibly nil but you're confident it's not, silence the error, don't change code

**Known false positives (always silence, don't fix):**
- **Type narrowing** - Value was checked to be not nil then used, steep doesn't understand the nil check happened
- **Cross-scope assignments** - Value being set across scopes (e.g., in lambda or block)
- **Multi-type containers** - Arrays, hashes with values of multiple types

```bash
# Check Steep errors
bundle exec steep check

# Analyze each error:
# - Is this a genuine bug? → Fix the code
# - Is this a false positive? → Add steep ignore directive
# - Not sure? → Add steep ignore directive

# Example: Type narrowing false positive
# Before:
#   user = find_user(id)
#   if user
#     user.name  # Steep error: receiver may be nil
#   end
#
# Don't change to: user&.name (changes behavior!)
# Instead, add steep ignore:

# @type var user: User?
user = find_user(id)
if user
  # steep:ignore:start NoMethod
  user.name  # Steep thinks this might be nil, but we checked above
  # steep:ignore:end
end

# Example: Value set in block
# Before:
#   result = nil
#   some_block do
#     result = compute_value()  # Sets result
#   end
#   process(result)  # Steep error: argument may be nil
#
# Don't refactor the code!
# Instead, add steep ignore:

result = nil
some_block do
  result = compute_value()
end
# steep:ignore:start
process(result)  # Steep doesn't track that result was set in block
# steep:ignore:end

# Example: Multi-type array
# Before:
#   items = [1, "hello", :symbol]
#   items.each { |item| puts item }  # Steep error: union types
#
# Don't split into separate arrays!
# Instead, add steep ignore or type annotation:

# @type var items: Array[Integer | String | Symbol]
items = [1, "hello", :symbol]
items.each { |item| puts item }

# After fixing genuine issues and silencing false positives, commit
git add -A
git commit -m "$(cat <<'EOF'
Fix Steep type checking errors

Address genuine type issues and silence false positives with
steep ignore directives. Steep errors silenced for:
- Type narrowing after nil checks
- Values set across lambda/block scopes
- Multi-type containers

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>
EOF
)"
```

**Steep ignore directive formats:**

```ruby
# Single line ignore
user.name # steep:ignore NoMethod

# Block ignore
# steep:ignore:start
code_with_false_positives()
more_code()
# steep:ignore:end

# Ignore specific error types
# steep:ignore NoMethod
user.name

# steep:ignore UnresolvedOverloading
array.map { |x| process(x) }

# Type annotations to help Steep understand
# @type var value: String
value = might_return_nil_but_we_know_it_wont()

# @type var items: Array[Integer | String]
items = [1, 2, "hello"]
```

**Decision tree for Steep errors:**

```
Steep reports error
  ├─ Does this reveal a real bug? (wrong type, missing nil check for actual nil)
  │  └─ YES → Fix the code (add nil check, fix type)
  │
  ├─ Is this a false positive?
  │  ├─ Type narrowing not recognized? → Add steep ignore
  │  ├─ Value set in block/lambda? → Add steep ignore
  │  ├─ Multi-type container? → Add type annotation or steep ignore
  │  └─ Other false positive? → Add steep ignore
  │
  └─ Not sure if genuine or false positive?
     └─ Add steep ignore (safer to silence than to change working code)
```

**Examples of genuine bugs to fix:**

```ruby
# BAD - Genuine bug, steep is right
user = User.find_by(id: params[:id])  # Returns nil if not found
user.update(name: "test")  # Bug! user might be nil

# GOOD - Fix the code
user = User.find_by(id: params[:id])
if user
  user.update(name: "test")
else
  handle_not_found()
end

# BAD - Genuine type error
def process(value: String)
  puts value.upcase
end

process(value: 123)  # Bug! passing Integer instead of String

# GOOD - Fix the call
process(value: "123")
```

**Examples of false positives to silence:**

```ruby
# FALSE POSITIVE - Type narrowing
user = find_user(id)
if user  # Nil check here
  # steep:ignore:start
  user.name  # Steep doesn't understand we checked nil above
  # steep:ignore:end
end

# FALSE POSITIVE - Value set in block
result = nil
transaction do
  result = create_user()
end
# steep:ignore:start
send_notification(result)  # Steep doesn't track assignment in block
# steep:ignore:end

# FALSE POSITIVE - Multi-type array
# @type var mixed: Array[Integer | String]
mixed = [1, 2, "three"]
mixed.each { |item| puts item }  # Steep might complain about union type
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

### Step 6: Push All Commits

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
- `Fix StandardRB style violations`
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
   # IMPORTANT: Use file-based jq filter to avoid shell quoting issues
   cat > /tmp/filter_failed_nontests.jq << 'EOF'
[.[] | select(.state == "FAILURE")] |
map(select(.name | test("test|spec|e2e|integration|build.*test"; "i") | not))
EOF

   failed_checks=$(gh pr checks <PR_NUMBER> --json name,state,link,detailsUrl | \
     jq -r -f /tmp/filter_failed_nontests.jq)
   ```

3. **For each non-test failure:**
   - Identify the tool (StandardRB, ESLint, etc.)
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
          ├─ StandardRB → Run `bundle exec rake standard:fix` → Commit
          ├─ ESLint → Run `eslint --fix` → Commit
          ├─ Prettier → Run `prettier --write` → Commit
          ├─ Black → Run `black .` → Commit
          ├─ TypeScript → Analyze errors → Fix manually → Commit
          ├─ Sorbet → Analyze errors → Fix manually → Commit
          ├─ Steep → Analyze each error:
          │   ├─ Genuine bug? → Fix code → Commit
          │   ├─ Type narrowing false positive? → Add steep ignore → Commit
          │   ├─ Cross-scope assignment? → Add steep ignore → Commit
          │   ├─ Multi-type container? → Add steep ignore → Commit
          │   └─ Unclear? → Add steep ignore → Commit
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
  ❌ StandardRB
  ❌ ESLint
  ✅ Test (Ruby 3.0) - Skipped (test job)
  ✅ Test (Ruby 2.7) - Skipped (test job)

## Fixing StandardRB violations...
Running: bundle exec rake standard:fix
Fixed 23 violations in 5 files
✅ Committed: Fix StandardRB style violations

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
- Example: StandardRB failing in multiple jobs → one `bundle exec rake standard:fix` fixes all

**Tool not available in environment:**
- Report error: "Cannot fix StandardRB: bundle exec rake standard not found"
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
  - Ruby: `bundle`, `standard` (via rake tasks)
  - JavaScript/TypeScript: `npm`/`yarn`, `eslint`, `prettier`, `tsc`
  - Python: `pip`, `black`, `ruff`, `mypy`
  - Go: `go`, `golangci-lint`, `goimports`

## Commands Reference

```bash
# Checkout PR
gh pr checkout <PR_NUMBER>

# Get failed non-test checks (using file-based jq filter)
cat > /tmp/filter_nontests.jq << 'EOF'
[.[] | select(.state == "FAILURE")] |
map(select(.name | test("test|spec|e2e|integration"; "i") | not))
EOF

gh pr checks <PR_NUMBER> --json name,state,link,detailsUrl | \
  jq -f /tmp/filter_nontests.jq

# Common fix commands
bundle exec rake standard:fix       # Ruby linting (auto-fix)
bundle exec rake standard           # Ruby linting (check only)
npx eslint --fix .                  # JavaScript linting
npx prettier --write .              # Code formatting
black .                             # Python formatting
ruff check --fix .                  # Python linting
golangci-lint run --fix             # Go linting
npx tsc --noEmit                    # TypeScript checking
bundle exec steep check             # Steep type checking (Ruby)

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

1. **One commit per tool/fix type** - Don't mix StandardRB and ESLint fixes in one commit
2. **Descriptive commit messages** - Clearly state what was fixed
3. **Auto-fix first** - Try automated fixes before manual intervention
4. **Verify fixes** - Run the tool again to confirm issues are resolved
5. **Push at the end** - Make all commits first, then push once
6. **Include Co-Authored-By** - Credit Claude in all commits
7. **Skip test failures** - Only fix non-test issues
8. **Report clearly** - Show what was fixed and what was skipped
9. **Follow code style** - Use trailing commas in di/ and symbol_database/ (see CLAUDE.md)
10. **Steep errors** - Prefer steep ignore directives over code changes; only fix genuine bugs

## Safety Checks

Before pushing:
- Verify no test files were modified (unless fixing linting in tests)
- Confirm commits only contain formatting/style changes
- Check that no functionality was accidentally changed
- Run `git diff origin/branch HEAD` to review all changes

## Notes

- Always use auto-fix options when available (`standard:fix`, `--fix`, `--write`)
- For manual fixes, be conservative - only fix what's clearly wrong
- If a fix is ambiguous or complex, skip it and report for manual review
- Some tools (like TypeScript) may require multiple iterations
- Be aware of tool configuration files (`.standard.yml`, `.customcops.yml`, `.eslintrc`, etc.)
- Respect the project's existing code style and conventions
- **Steep type checker**: Most errors are false positives due to limitations in type narrowing and cross-scope tracking. When in doubt, add `steep:ignore` directives rather than changing working code. Only fix the code if the error reveals a genuine bug (actual nil dereference, wrong type passed, etc.)
