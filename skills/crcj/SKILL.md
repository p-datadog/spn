---
name: crcj
description: This skill should be used when the user asks to "check CI jobs", "restart failed CI", "fix CI failures", "check p-datadog PRs", or mentions checking and restarting failed GitHub Actions jobs.
version: 0.1.0
---

# Check and Restart CI Jobs (crcj)

This skill checks all PRs with the `p-datadog` label in the `DataDog/dd-trace-rb` repository and restarts failed CI jobs that appear to be infrastructure-related failures.

## Overview

The skill automates the process of:
1. Finding all open PRs with the `p-datadog` label
2. Skipping PRs with the `no-restart` label (these are intentionally excluded from automatic CI restarts)
3. Checking CI job status for each PR
4. Investigating failed jobs
5. Restarting jobs that failed due to infrastructure issues
6. Special handling for the "All Required Checks Passed" job

## When This Skill Applies

Use this skill when:
- User asks to "check CI for p-datadog PRs"
- User wants to "restart failed CI jobs"
- User mentions "fix infrastructure failures in CI"
- Periodic maintenance of p-datadog labeled PRs

## Workflow

### Step 1: Find All p-datadog PRs

```bash
# List all open PRs with p-datadog label
gh pr list \
  --repo DataDog/dd-trace-rb \
  --label "p-datadog" \
  --state open \
  --json number,title,url,headRefName
```

### Step 2: Check for no-restart Label

Before processing a PR, check if it has the "no-restart" label:

```bash
# Get PR labels
gh pr view <PR_NUMBER> --repo DataDog/dd-trace-rb --json labels --jq '.labels[].name'

# If "no-restart" label is present, skip this PR entirely
```

PRs with the "no-restart" label are intentionally excluded from automatic CI job restarts. This allows developers to prevent automatic intervention on specific PRs where they want to manually control CI behavior.

### Step 3: For Each PR, Check CI Status

```bash
# Get check status for a specific PR
gh pr checks <PR_NUMBER> --repo DataDog/dd-trace-rb

# Get detailed check information with JSON output
gh pr checks <PR_NUMBER> --repo DataDog/dd-trace-rb --json name,status,conclusion,detailsUrl
```

### Step 4: Identify Failed Jobs

Look for checks where:
- `status` = "completed"
- `conclusion` = "failure"

### Step 5: Investigate Failure Reasons

For each failed check, examine the logs:

```bash
# View check run details
gh api repos/DataDog/dd-trace-rb/check-runs/<CHECK_RUN_ID>

# Get logs URL from the response
gh run view <RUN_ID> --repo DataDog/dd-trace-rb --log
```

**Infrastructure failure patterns to look for:**
- GitHub API errors: "API rate limit exceeded", "GitHub is unavailable"
- Network issues: "connection timeout", "connection refused", "network unreachable"
- GitHub Actions issues: "failed to download action", "runner communication error"
- Dependency download failures: "gem install failed", "bundle install timeout" (if transient)
- Flaky tests that are clearly environment-related (not code issues)
- "Internal server error" from GitHub
- "Unable to process request at this time"
- Container/image pull failures: "docker pull failed", "image not found" (if transient)

**NOT infrastructure failures (don't restart):**
- Test failures with clear assertion errors
- Code compilation errors
- Linting/RuboCop failures
- Code coverage failures
- Security vulnerabilities detected
- Consistent, reproducible test failures

### Step 6: Restart Infrastructure Failures

```bash
# Get the workflow run ID from the check
gh api repos/DataDog/dd-trace-rb/commits/<COMMIT_SHA>/check-runs \
  --jq '.check_runs[] | select(.name == "<CHECK_NAME>") | .id'

# Restart the failed job
gh run rerun <RUN_ID> --repo DataDog/dd-trace-rb --failed
```

## Special Case: "All Required Checks Passed" Job

**Job name:** "All Required Checks Passed" (or similar - verify exact name from actual PR)

**Special handling:**
- This is a status check that depends on all other checks passing
- If this is the ONLY failing job and all other checks are green, restart it immediately
- No investigation needed - this is always safe to restart

**Detection:**
```bash
# Check if only "All Required Checks Passed" is failing
checks=$(gh pr checks <PR_NUMBER> --repo DataDog/dd-trace-rb --json name,conclusion)

# Parse to see if only this check failed
echo "$checks" | jq 'map(select(.conclusion == "failure")) | length'
# If count is 1, check if it's the "All Required Checks Passed" job
echo "$checks" | jq -r 'map(select(.conclusion == "failure"))[0].name'
```

**Immediate restart:**
```bash
# If it's the only failure and it's the "All Required Checks Passed" job
gh run rerun <RUN_ID> --repo DataDog/dd-trace-rb
```

## Implementation Steps

When the skill is invoked:

1. **Fetch all p-datadog PRs:**
   ```bash
   prs=$(gh pr list --repo DataDog/dd-trace-rb --label "p-datadog" --state open --json number,title,headRefOid)
   ```

2. **For each PR:**
   ```bash
   for pr in $(echo "$prs" | jq -r '.[].number'); do
     echo "Checking PR #$pr"

     # Check if PR has "no-restart" label - if so, skip it
     labels=$(gh pr view $pr --repo DataDog/dd-trace-rb --json labels --jq '.labels[].name')
     if echo "$labels" | grep -q "no-restart"; then
       echo "  ⏭️  Skipping PR #$pr (has 'no-restart' label)"
       continue
     fi

     # Get check status
     checks=$(gh pr checks $pr --repo DataDog/dd-trace-rb --json name,status,conclusion,workflowName)

     # Find failed checks
     failed=$(echo "$checks" | jq -r '.[] | select(.conclusion == "failure")')

     if [ -z "$failed" ]; then
       echo "  ✅ All checks passing"
       continue
     fi

     # Check for "All Required Checks Passed" special case
     failed_count=$(echo "$checks" | jq '[.[] | select(.conclusion == "failure")] | length')
     if [ "$failed_count" -eq 1 ]; then
       failed_name=$(echo "$checks" | jq -r '[.[] | select(.conclusion == "failure")][0].name')
       if [[ "$failed_name" == *"Required"*"Checks"* ]] || [[ "$failed_name" == *"All"*"green"* ]]; then
         echo "  🔄 Only 'All Required Checks Passed' failing, restarting immediately"
         # Get run ID and restart
         # gh run rerun <RUN_ID> --repo DataDog/dd-trace-rb
       fi
     fi

     # Investigate other failures
     echo "$failed" | jq -r '.name' | while read check_name; do
       echo "  ❌ Failed: $check_name"
       # Investigate and restart if infrastructure-related
     done
   done
   ```

3. **For each failed check, get logs and analyze:**
   ```bash
   # Get workflow run for the PR
   runs=$(gh api repos/DataDog/dd-trace-rb/commits/$commit_sha/check-runs --jq '.check_runs[] | select(.name == "'$check_name'") | {id: .id, run_id: .run_id}')

   run_id=$(echo "$runs" | jq -r '.run_id')

   # View logs
   gh run view $run_id --repo DataDog/dd-trace-rb --log > /tmp/check_log.txt

   # Analyze for infrastructure failures
   if grep -qiE "(github.*unavailable|api rate limit|connection timeout|runner.*error|internal server error)" /tmp/check_log.txt; then
     echo "    🔄 Infrastructure failure detected, restarting..."
     gh run rerun $run_id --repo DataDog/dd-trace-rb --failed
   else
     echo "    ⚠️  Appears to be a code/test failure, not restarting"
   fi
   ```

4. **Report summary:**
   - Total PRs checked
   - PRs with failures
   - Jobs restarted (with reasons)
   - Jobs NOT restarted (with reasons)

## Infrastructure Failure Detection Heuristics

Use these patterns to detect infrastructure failures in logs:

**GitHub/Actions Issues:**
- `API rate limit exceeded`
- `GitHub is unavailable`
- `Service Unavailable`
- `Internal Server Error` (from github.com)
- `failed to download action`
- `Unable to process your request`
- `runner.*communication error`
- `Actions service is currently unavailable`

**Network Issues:**
- `connection timeout`
- `connection refused`
- `network unreachable`
- `Could not resolve host`
- `Connection reset by peer`
- `TLS handshake timeout`

**Dependency/Package Issues (transient):**
- `Temporary failure resolving` (DNS)
- `Failed to fetch` (intermittent)
- `gem install.*timeout`
- `bundle install.*timed out`
- `npm.*network socket hung up`
- `docker pull.*timeout`
- `docker pull.*TLS handshake timeout`

**Test Infrastructure (flaky, not code):**
- `Selenium.*driver.*error` (intermittent)
- `ChromeDriver.*not reachable`
- `Database.*connection pool exhausted` (if rare)
- Test framework crashes (not assertion failures)

**DO NOT RESTART for:**
- RSpec/Minitest assertion failures
- `expected X, got Y`
- RuboCop offenses
- SimpleCov coverage below threshold
- Bundler-audit security findings
- Syntax errors
- NameError, NoMethodError, etc. (code errors)

## Example Output Format

```
# CI Job Check Report - p-datadog PRs

## Summary
- Total PRs checked: 15
- PRs with failures: 4
- Jobs restarted: 6
- Jobs skipped: 2

## PR #4567: Add dynamic instrumentation support
✅ All checks passing

## PR #4568: Fix probe serialization
❌ Failed checks: 2

  🔄 RESTARTED: "Test (Ruby 3.0)"
     Reason: GitHub Actions runner communication error
     Run ID: 12345678

  🔄 RESTARTED: "Test (Ruby 2.7)"
     Reason: Connection timeout downloading dependencies
     Run ID: 12345679

## PR #4569: Update telemetry reporting
❌ Failed checks: 1

  ⚠️  SKIPPED: "Test (Ruby 3.1)"
     Reason: Test assertion failure (code issue, not infrastructure)
     Failed test: spec/datadog/di/probe_spec.rb:145
     Error: expected true, got false

## PR #4570: Refactor probe manager
❌ Failed checks: 1

  🔄 RESTARTED: "All Required Checks Passed"
     Reason: Only failing job (all other checks green)
     Run ID: 12345680

## Actions Taken
✅ Restarted 6 infrastructure-related failures
⚠️  Skipped 2 code-related failures (require developer attention)
```

## Edge Cases

**PRs with "no-restart" label:**
- Skip these PRs entirely, regardless of CI status
- Report that the PR was skipped due to the label
- This allows developers to opt-out of automatic CI restarts for specific PRs

**Multiple failures in same PR:**
- Investigate each independently
- Restart infrastructure failures even if some are code issues
- Report all actions taken

**All checks already passing:**
- Report success, no action needed

**PR is draft:**
- Check CI anyway (p-datadog label takes precedence)

**Recently restarted jobs:**
- Consider checking timestamps to avoid restart loops
- If a job was restarted in the last 10 minutes, skip it

**Protected branches:**
- May require admin permissions to restart
- Report if unable to restart

## Required Permissions

This skill requires:
- GitHub CLI (`gh`) authenticated with a token that has:
  - `repo` scope (read PR and check status)
  - `actions:write` scope (restart workflow runs)

## Commands Reference

```bash
# List PRs with label
gh pr list --repo DataDog/dd-trace-rb --label "p-datadog" --state open

# Check PR status
gh pr checks <PR_NUMBER> --repo DataDog/dd-trace-rb

# Get check details JSON
gh pr checks <PR_NUMBER> --repo DataDog/dd-trace-rb --json name,status,conclusion

# Get workflow runs for commit
gh api repos/DataDog/dd-trace-rb/commits/<SHA>/check-runs

# View run logs
gh run view <RUN_ID> --repo DataDog/dd-trace-rb --log

# Rerun failed jobs only
gh run rerun <RUN_ID> --repo DataDog/dd-trace-rb --failed

# Rerun entire workflow
gh run rerun <RUN_ID> --repo DataDog/dd-trace-rb
```

## Testing the Skill

To test without actually restarting jobs:

1. Add a `--dry-run` mode that reports what would be restarted
2. Use `echo` instead of actual `gh run rerun` commands
3. Verify detection logic with known failure patterns

## Notes

- Always err on the side of caution: if unsure whether a failure is infrastructure-related, DON'T restart
- Log all restart decisions for audit trail
- Consider rate limiting to avoid overwhelming GitHub Actions
- The "All Required Checks Passed" job name may vary - check actual PR to confirm exact name
