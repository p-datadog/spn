---
name: di-pr-respond
description: This skill should be used when the user asks to "respond to PR comments", "respond to review", "address PR feedback", or mentions responding to Datadog dynamic instrumentation pull request reviews.
version: 0.1.0
---

# Datadog dd-trace-rb Dynamic Instrumentation PR Response

This skill handles responding to review comments on pull requests in the `datadog/dd-trace-rb` repository related to dynamic instrumentation.

## Overview

This skill systematically processes PR review comments, makes requested changes, finds and fixes similar issues throughout the codebase, and responds appropriately to each comment.

## Default Behavior

**IMPORTANT:** When invoked with a PR number or URL, this skill automatically:
1. ✅ Commits each fix separately with descriptive messages
2. ✅ Pushes all commits to the PR branch after addressing all comments
3. ✅ No user confirmation required

This is the **default behavior**. All fixes are committed and pushed automatically.

## When This Skill Applies

Use this skill when:
- Addressing review comments on a dd-trace-rb PR
- Responding to feedback on dynamic instrumentation changes
- Fixing issues identified in PR reviews
- Answering questions from reviewers

## Response Rules

**IMPORTANT:** All responses must be posted inline as replies to the original review comment. Never post general PR comments - always reply directly to the specific comment thread.

### Rule 1: Change Requests - Fix Everywhere

**When:** Comment requests a code change or fix.

**Actions:**
1. Make the requested change at the commented location
2. Search the entire diff for similar code patterns
3. Fix ALL similar occurrences in the changed files
4. **Commit the changes with a descriptive message**
5. Respond **inline** to the comment with a list of all locations where the fix was applied
6. **DO NOT** resolve the comment (let the reviewer verify and resolve)

**Example response format:**
```
Fixed in the following locations:
- lib/datadog/di/probe_manager.rb:67-72 (original location)
- lib/datadog/di/probe_manager.rb:145-150 (similar pattern)
- lib/datadog/di/snapshot_collector.rb:89-94 (similar pattern)
- spec/datadog/di/probe_manager_spec.rb:234 (test coverage added)

All TracePoint callbacks now have proper error boundaries with logging and telemetry.
```

**How to find similar code:**

**CRITICAL:** This is NOT optional - it's a mandatory step. When a reviewer asks for a fix, they expect consistency everywhere in the PR, not just the one line they commented on.

**1. Extract the pattern from the comment:**
- "stop saying X" → search for "X" (including in comments and docs)
- "add error boundary" → search for `TracePoint.new`, callback patterns
- "use constant" → search for magic numbers or hardcoded values
- "rename X to Y" → search for "X" everywhere
- "remove sleep" → search for `sleep` in test files

**2. Search the PR diff first:**
```bash
# Get full diff and search for pattern
gh pr diff <PR_NUMBER> | grep -i "<pattern>" > /tmp/matches.txt

# Review all matches
cat /tmp/matches.txt
```

**3. Search changed files directly:**
```bash
# Get list of changed files
gh pr diff <PR_NUMBER> --name-only

# Search in those files
grep -rn "<pattern>" <changed_files>
```

**4. Search for BOTH code and semantic mentions:**
- Don't just search for code (method calls, checks, conditions)
- **ALSO search for comments, docs, and messages** that mention the pattern
- Example: "requires DI" appears in comments, not just code
- Example: "sleep" appears in both code and code comments explaining waits

**5. Try multiple search patterns:**
```bash
# Try variations to ensure completeness
gh pr diff <PR> | grep -i "requires DI"           # Exact phrase
gh pr diff <PR> | grep -i "symbol.*database.*DI"  # Semantic match
gh pr diff <PR> | grep -i "symdb.*di"             # Abbreviation
```

**6. Verify completeness after fixing:**
```bash
# After fixing, verify pattern is gone
gh pr diff <PR_NUMBER> | grep -i "<pattern>"
# Should return nothing (or only irrelevant matches)
```

**Examples of common mistakes:**
- ❌ Searching only for code: `unless settings.dynamic_instrumentation.enabled`
- ✅ Searching for semantics: "requires DI", "depends on DI", "needs DI"
- ❌ Fixing one file only
- ✅ Searching entire PR diff and fixing all files
- ❌ Case-sensitive search missing variations
- ✅ Using `-i` flag for case-insensitive search

**Examples of Proper Pattern Searching:**

These examples demonstrate the complete workflow for finding and fixing all similar occurrences:

**Example 1: "stop saying symdb requires di"**

❌ **Wrong approach:**
- Only fix the code check: `unless settings.dynamic_instrumentation.enabled`
- Miss comments saying "requires DI"

✅ **Right approach:**
```bash
# Search the PR diff for the semantic pattern
gh pr diff 5431 | grep -i "requires DI"

# Results found:
# - lib/datadog/symbol_database/component.rb:26 (doc comment)
# - lib/datadog/symbol_database/component.rb:41 (code check)
# - lib/datadog/core/configuration/components.rb:173 (comment)
# - lib/datadog/core/configuration/components.rb:241 (comment)
# - lib/datadog/core/remote/client/capabilities.rb:43 (comment)

# Fix ALL 5 occurrences (1 code + 4 comments)
# Verify: gh pr diff 5431 | grep -i "requires DI"  # Should return empty
```

**Example 2: "add error boundary to TracePoint callbacks"**

❌ **Wrong approach:**
- Only fix the commented callback
- Miss similar callbacks elsewhere

✅ **Right approach:**
```bash
# Search for all TracePoint usage in changed files
gh pr diff <PR> | grep -B 2 -A 5 "TracePoint.new"

# Also search in the affected directories
grep -rn "TracePoint.new" lib/datadog/di/

# Fix ALL callbacks found (often 5-10 locations)
```

**Example 3: "use CONSTANT instead of magic number"**

❌ **Wrong approach:**
- Only replace the one number in the commented location

✅ **Right approach:**
```bash
# Search for the magic number (e.g., 60) in the PR diff
gh pr diff <PR> | grep -n "\b60\b"

# Search in context to find similar uses
grep -rn "timeout.*60\|60.*second" lib/datadog/

# Replace ALL occurrences with the constant
```

**Example 4: "remove sleep from tests"**

❌ **Wrong approach:**
- Only remove the one sleep

✅ **Right approach:**
```bash
# Search for all sleep calls in test files
gh pr diff <PR> | grep -n "sleep"

# Also check for Thread.pass and other timing-based waits
gh pr diff <PR> | grep -n "sleep\|Thread.pass\|Time.now"

# Replace ALL with deterministic synchronization (queues, waits)
```

**Key Principle:**

When a reviewer says "fix X", they mean:
1. Fix the code that implements X
2. Fix the comments/docs that describe X
3. Fix the tests that verify X
4. Fix ALL similar occurrences in the PR diff

**This is the mandatory workflow - not optional.**

### Rule 2: Disagreements - Explain Reasoning

**When:** You disagree with the requested change or believe the comment is incorrect.

**Actions:**
1. Respond inline to the comment
2. Clearly explain your reasoning
3. Provide evidence (code references, documentation, test results)
4. Suggest alternatives if applicable
5. Be respectful and open to discussion

**Example response format:**
```
I believe the current implementation is correct for the following reasons:

1. The error boundary is not needed here because this code runs outside the TracePoint callback (lib/datadog/di/probe_manager.rb:45-50), so exceptions will not propagate to customer code.

2. This method is only called during initialization, before any customer code is instrumented.

3. Test coverage shows that exceptions here are handled by the outer rescue block at line 30.

However, if you prefer an additional error boundary for defense-in-depth, I'm happy to add it. Let me know your preference.
```

**Guidelines for disagreements:**
- Never be dismissive or defensive
- Acknowledge the reviewer's concern
- Provide specific code references
- Offer to make the change if the reviewer still prefers it
- Remember: the reviewer may have context you don't

### Rule 3: Questions - Answer Inline

**When:** Comment asks a question.

**Actions:**
1. Respond inline with a clear, direct answer
2. Provide code references or examples if helpful
3. Offer to make clarifying changes if needed (comments, docs, better naming)

**Example response format:**
```
Good question! The timeout is set to 5 seconds to match the agent's default snapshot upload interval (see lib/datadog/di/configuration.rb:89).

This ensures we don't accumulate snapshots faster than we can export them. The value is configurable via settings.dynamic_instrumentation.snapshot_timeout.

Would you like me to add a comment explaining this in the code?
```

**Guidelines for questions:**
- Answer directly and concisely
- Don't assume the question implies a problem
- Offer to improve code clarity if the question suggests confusion
- Link to relevant documentation or code

## Response Process

Follow these steps when responding to PR comments:

### 1. Fetch PR and Comments

```bash
# Checkout the PR branch
gh pr checkout <PR_NUMBER>

# View PR details and comments
gh pr view <PR_NUMBER>

# Get review comments as JSON (only non-outdated)
gh api repos/DataDog/dd-trace-rb/pulls/<PR_NUMBER>/comments | \
  jq '[.[] | select(.outdated == false)]'
```

**CRITICAL: Ignore outdated comments**

**What are outdated comments?**
- Comments made on previous versions of code that has since been changed
- GitHub marks comments as `outdated: true` when the code they reference is modified
- They're collapsed in the UI under "Show outdated" sections

**Why ignore them?**
- They've usually already been addressed (that's why the code changed)
- The reviewer hasn't re-commented on new code, so no action needed
- You'd be responding to code that no longer exists
- Wastes time on historical context

**How to identify them:**
```bash
# Filter out outdated comments using jq
gh api repos/DataDog/dd-trace-rb/pulls/<PR_NUMBER>/comments | \
  jq '[.[] | select(.outdated == false)]' > /tmp/current_comments.json

# Only process non-outdated comments
cat /tmp/current_comments.json
```

**When to address an "outdated" comment:**
- Only if the reviewer explicitly re-posts it on current code
- Only if they say "this still applies to the new code"
- Never respond to outdated comments unless explicitly asked

### 2. Categorize Each Comment

For each **non-outdated** comment, determine:
- Is this a change request?
- Is this a question?
- Do I disagree with this?

**Skip any comments with `outdated: true` - they've been superseded by code changes.**

### 3. Process Change Requests

For each change request:

```bash
# 1. Read the file being commented on
# Use Read tool to understand context

# 2. Make the requested change
# Use Edit tool for precise changes

# 3. Search for similar patterns in the diff
# CRITICAL: Search for BOTH code and semantic mentions (comments, docs)
gh pr diff <PR_NUMBER> | grep -i "pattern" > /tmp/matches.txt
cat /tmp/matches.txt  # Review all matches

# Try variations to ensure completeness
gh pr diff <PR_NUMBER> | grep -i "semantic_pattern"

# Verify you found everything
gh pr diff <PR_NUMBER> | grep -i "pattern" | wc -l  # Count occurrences

# 4. Fix all similar occurrences found
# Use Edit tool for each location (code + comments + docs)

# 5. Commit the changes with a descriptive message
git add <changed-files>
git commit -m "$(cat <<'EOF'
Fix <brief description of what was fixed>

Address review comment: <summary of the review feedback>

- Fixed in <file>:<line>
- Fixed in <file>:<line> (similar pattern)
- Added test coverage in <test_file>:<line>

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>
EOF
)"

# 6. Verify completeness - pattern should be gone
gh pr diff <PR_NUMBER> | grep -i "pattern"
# Should return nothing (or only irrelevant matches)
# If it still finds matches, go back to step 4 and fix them

# 7. Respond inline to the comment with all locations fixed
gh api repos/DataDog/dd-trace-rb/pulls/comments/<COMMENT_ID>/replies \
  -f body="Fixed in the following locations: ..."
```

### 4. Respond to All Comments

**All responses must be inline to the original comment:**

```bash
# Respond inline to any comment (change request, question, or disagreement)
gh api repos/DataDog/dd-trace-rb/pulls/comments/<COMMENT_ID>/replies \
  -f body="Your response here..."
```

**Note:** Every comment gets an inline response, whether it's a change request showing all fixes, a question being answered, or a disagreement being discussed.

### 5. Verify Changes

```bash
# Run tests to verify fixes
bundle exec rspec

# Check code style
bundle exec rubocop

# Review the diff
git diff origin/$(git branch --show-current)..HEAD
```

### 6. Push All Commits

**After responding to all comments, if any changes were made:**

```bash
# Check if any commits were made
commits_made=$(git log origin/$(git branch --show-current)..HEAD --oneline | wc -l)

# Push all commits to the PR branch
if [ "$commits_made" -gt 0 ]; then
  echo "Pushing $commits_made commit(s) to PR branch..."
  git push origin HEAD
else
  echo "No changes were made, nothing to push"
fi
```

**IMPORTANT:** Only push after all comments have been addressed and all changes committed. This ensures all fixes are pushed together.

## Commit Message Format

Each change addressing a review comment should be committed separately with a descriptive message:

```
Fix <brief description of what was fixed>

Address review comment: <summary of the review feedback>

- Fixed in <file>:<line> (original location)
- Fixed in <file>:<line> (similar pattern)
- Fixed in <file>:<line> (similar pattern)
- Added test coverage in <test_file>:<line>

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>
```

### Commit Message Examples

**Example 1: Adding error boundaries**
```
Fix missing error boundaries in TracePoint callbacks

Address review comment: Add error boundaries to prevent exceptions
from propagating to customer code.

- Fixed in lib/datadog/di/probe_manager.rb:67-75
- Fixed in lib/datadog/di/probe_manager.rb:145-153
- Fixed in lib/datadog/di/line_probe.rb:89-97
- Added test coverage in spec/datadog/di/probe_manager_spec.rb:345

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>
```

**Example 2: Adding telemetry**
```
Add missing telemetry to error handlers

Address review comment: All rescue blocks must report errors via
telemetry for observability.

- Fixed in lib/datadog/di/snapshot_serializer.rb:89-92
- Fixed in lib/datadog/di/config_loader.rb:134-137
- Added test coverage in spec/datadog/di/snapshot_serializer_spec.rb:256

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>
```

**Example 3: Removing sleep from tests**
```
Replace sleep with deterministic synchronization

Address review comment: Remove non-deterministic sleep calls from
tests and use queue-based synchronization.

- Fixed in spec/datadog/di/async_spec.rb:78
- Fixed in spec/datadog/di/worker_spec.rb:134
- Fixed in spec/datadog/di/background_processor_spec.rb:201

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>
```

### Commit Guidelines

- **One commit per review comment** - Group all fixes for one comment into one commit
- **Descriptive title** - Clearly state what was fixed (under 72 characters)
- **Reference the review** - Include "Address review comment" in the body
- **List all locations** - Show everywhere the fix was applied
- **Include test changes** - Note any test coverage added
- **Co-Authored-By line** - Always credit Claude

## Finding Similar Code Patterns

When fixing a change request, search for these similar patterns:

**ALWAYS start by searching the PR diff, not the entire codebase.** This ensures you fix all occurrences within the scope of the PR.

### Error Boundaries
If fixing missing error boundary, search for:
- All TracePoint callbacks
- All prepended method definitions
- All probe execution points
- All instrumentation entry points

```bash
# FIRST: Search the PR diff
gh pr diff <PR_NUMBER> | grep -B 2 -A 5 "TracePoint.new"
gh pr diff <PR_NUMBER> | grep "def.*super"
gh pr diff <PR_NUMBER> | grep "probe.execute"

# THEN: Search affected directories if needed
grep -rn "TracePoint.new" lib/datadog/di/
grep -rn "def.*super" lib/datadog/di/
grep -rn "probe.execute" lib/datadog/di/
```

### Missing Telemetry
If adding telemetry, search for:
- All rescue blocks in changed files
- Similar error conditions

```bash
# FIRST: Search the PR diff
gh pr diff <PR_NUMBER> | grep -B 2 -A 5 "rescue"

# THEN: Search specific changed files
grep -rn "rescue\s*=>" lib/datadog/di/changed_file.rb
```

### Test Issues
If fixing a test pattern, search for:
- Similar test cases
- Same test helper usage
- Related functionality tests

```bash
# FIRST: Search the PR diff
gh pr diff <PR_NUMBER> | grep "describe.*similar_pattern"
gh pr diff <PR_NUMBER> | grep "it.*similar_behavior"

# THEN: Search test directories if needed
grep -rn "describe.*similar_pattern" spec/
grep -rn "it.*similar_behavior" spec/
```

### Sleep Removal
If removing sleep, search for:
- All sleep calls in test files
- Thread.pass in loops
- Time-based waits

```bash
# FIRST: Search the PR diff (catches all sleep in changed files)
gh pr diff <PR_NUMBER> | grep -n "sleep\|Thread.pass"

# Verify completeness
gh pr diff <PR_NUMBER> | grep -n "sleep"  # Should be empty after fixes

# THEN: Search entire spec directory only if pattern spans beyond PR
grep -rn "sleep\|Thread.pass" spec/
```

### Semantic Comments and Documentation

**CRITICAL:** Don't forget to search for semantic mentions in comments and docs:

```bash
# Example: If fixing "symdb requires DI", search for the phrase
gh pr diff <PR_NUMBER> | grep -i "requires DI"
gh pr diff <PR_NUMBER> | grep -i "symbol.*database.*DI"

# Example: If renaming a concept, search for the old name
gh pr diff <PR_NUMBER> | grep -i "old_name"
```

**Remember:** Comments and documentation often describe patterns that code implements. Fix both!

## Response Template

When responding to a change request, use this format:

```markdown
Fixed in the following locations:

1. **[FILE_PATH:LINE]** (original location)
   - [Brief description of what was fixed]

2. **[FILE_PATH:LINE]** (similar pattern found)
   - [Brief description of what was fixed]

3. **[FILE_PATH:LINE]** (similar pattern found)
   - [Brief description of what was fixed]

**Tests updated:**
- [TEST_FILE:LINE] - Added test coverage for error boundary
- [TEST_FILE:LINE] - Added test for telemetry

**Pattern searched:** `[grep pattern used to find similar code]`

**Verification:**
- ✅ All tests passing
- ✅ RuboCop clean
- ✅ Coverage maintained at 100%
```

## Code Style Requirements

When making changes to code, follow these guidelines from `CLAUDE.md`:

### Trailing Commas

**MANDATORY** in these directories:
- `lib/datadog/di/` and all subdirectories
- `lib/datadog/symbol_database/` and all subdirectories

**RECOMMENDED** everywhere else when adding new code.

Always use trailing commas in multi-line arrays, hashes, and method calls:

```ruby
# ✅ Correct
config = {
  timeout: 30,
  retries: 3,
  enabled: true,
}

# ❌ Wrong (in di/ and symbol_database/)
config = {
  timeout: 30,
  retries: 3,
  enabled: true
}
```

See `CLAUDE.md` for complete guidelines.

## Important Guidelines

### DO:
- Fix all similar issues proactively
- Search both implementation and test files
- Add test coverage for any behavior changes
- Run tests after making changes
- **Commit each change separately with descriptive message**
- Provide specific file and line references
- Be thorough in searching for patterns
- Be respectful in all responses
- Acknowledge good catches by reviewers
- **Push all commits after responding to all comments**
- **Use trailing commas in di/ and symbol_database/ directories**

### DON'T:
- Resolve comments (let reviewers do it)
- Make changes you disagree with without discussion
- Fix only the pointed-out issue without checking for similar ones
- Argue unnecessarily - code reviews are collaborative
- Make assumptions about what the reviewer meant
- Skip running tests after changes
- Forget to update test coverage
- **Push before addressing all comments**
- **Mix unrelated fixes in one commit**

## Example Workflow

Here's a complete example of responding to a review comment:

**Review Comment:**
> File: lib/datadog/di/probe_manager.rb:67
>
> This TracePoint callback needs an error boundary. Exceptions here will propagate to customer code.

**Response Steps:**

1. **Read the file and understand the issue:**
```bash
# Read the file to see the code
```

2. **Make the fix at the commented location:**
```ruby
# Add begin/rescue/end with logging and telemetry
```

3. **Search for similar patterns:**
```bash
grep -rn "TracePoint.new" lib/datadog/di/
```

4. **Fix all similar occurrences found:**
- lib/datadog/di/probe_manager.rb:67
- lib/datadog/di/probe_manager.rb:145
- lib/datadog/di/line_probe.rb:89

5. **Add test coverage:**
- Test that exceptions in callbacks are caught
- Test that telemetry is emitted

6. **Run tests:**
```bash
bundle exec rspec
```

7. **Commit the changes:**
```bash
git add lib/datadog/di/probe_manager.rb lib/datadog/di/line_probe.rb spec/datadog/di/*_spec.rb
git commit -m "$(cat <<'EOF'
Fix missing error boundaries in TracePoint callbacks

Address review comment: Add error boundaries to prevent exceptions
from propagating to customer code.

- Fixed in lib/datadog/di/probe_manager.rb:67-75 (original location)
- Fixed in lib/datadog/di/probe_manager.rb:145-153 (similar pattern)
- Fixed in lib/datadog/di/line_probe.rb:89-97 (similar pattern)
- Added test coverage in spec/datadog/di/probe_manager_spec.rb:345
- Added test coverage in spec/datadog/di/line_probe_spec.rb:234

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>
EOF
)"
```

8. **Respond to comment:**
```markdown
Fixed in the following locations:

1. **lib/datadog/di/probe_manager.rb:67-75** (original location)
   - Added error boundary to :call/:return TracePoint callback
   - Added DEBUG logging and telemetry on exceptions

2. **lib/datadog/di/probe_manager.rb:145-153** (similar pattern)
   - Added error boundary to :line TracePoint callback
   - Added DEBUG logging and telemetry on exceptions

3. **lib/datadog/di/line_probe.rb:89-97** (similar pattern)
   - Added error boundary to probe-specific TracePoint
   - Added DEBUG logging and telemetry on exceptions

**Tests updated:**
- spec/datadog/di/probe_manager_spec.rb:345 - Test TracePoint with probe.execute raising
- spec/datadog/di/probe_manager_spec.rb:362 - Test telemetry emission on error
- spec/datadog/di/line_probe_spec.rb:234 - Test error boundary in line probe

**Pattern searched:** `TracePoint.new` in all DI files

**Verification:**
- ✅ All tests passing (bundle exec rspec)
- ✅ RuboCop clean
- ✅ Coverage maintained at 100%
```

## Conflict Resolution

If you're uncertain whether a comment is a change request or just a question:
- Treat it as a question first
- Ask for clarification: "Would you like me to make this change, or were you asking for clarification?"
- Wait for reviewer response before making changes

If a change request conflicts with repository guidelines:
- Point out the conflict
- Reference the specific guideline (CLAUDE.md, CONTRIBUTING.md)
- Ask how to proceed
- Suggest alternatives that satisfy both

## Success Criteria

A successful response includes:
- ✅ All requested changes made
- ✅ Similar issues found and fixed proactively
- ✅ All questions answered clearly
- ✅ Disagreements explained respectfully with evidence
- ✅ **Each change committed separately with descriptive message**
- ✅ Tests passing after changes
- ✅ Test coverage maintained or improved
- ✅ Detailed response showing all locations fixed
- ✅ Comments NOT resolved (left for reviewer)
- ✅ **All commits pushed to PR branch after all comments addressed**
