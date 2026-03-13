---
name: crcj
description: This skill should be used when the user asks to "check CI jobs", "restart failed CI", "fix CI failures", "check p-datadog PRs", or mentions checking and restarting failed GitHub Actions jobs.
version: 0.1.0
---

# Check and Restart CI Jobs (crcj)

This skill checks all PRs with the `p-datadog` label in the `DataDog/dd-trace-rb` repository and restarts failed CI jobs that failed due to infrastructure issues (GitHub Actions, runners, network, authentication, etc.).

## Overview

The skill automates the process of:
1. Finding all open PRs with the `p-datadog` label
2. Skipping PRs with the `no-restart` label (these are intentionally excluded from automatic CI restarts)
3. Checking CI job status for each PR
4. Investigating failed jobs
5. Restarting jobs that failed due to infrastructure issues
6. Special handling for the "All Required Checks Passed" job
7. Printing a final summary table showing all PRs, their titles, and actions taken

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
gh pr checks <PR_NUMBER> --repo DataDog/dd-trace-rb --json name,state,detailsUrl
```

### Step 4: Identify Failed Jobs

Look for checks where:
- `status` = "completed"
- `state` = "FAILURE"

### Step 5: Detect Infrastructure Failures

**CRITICAL:** Use infrastructure detection patterns to identify jobs that should be restarted.

**Source:** Infrastructure failure patterns are defined in `https://github.com/p-datadog/bells/blob/master/docs/infrastructure-failure-detection.md`

For each failed check, scan the job logs for infrastructure failure patterns:

```bash
# Get the run ID from the check
run_id=$(gh pr checks <PR_NUMBER> --repo DataDog/dd-trace-rb --json name,link | \
  jq -r '.[] | select(.name == "<CHECK_NAME>") | .link | split("/") | .[-3]')

# Fetch first ~500 lines of logs (infrastructure failures usually appear early)
gh run view $run_id --repo DataDog/dd-trace-rb --log 2>&1 | head -500 > /tmp/job_logs.txt

# Check for infrastructure failure patterns
if grep -qE '(fatal: could not read (Username|Password)|terminal prompts disabled|exit code 128|401 \(Unauthorized\)|403 \(Forbidden\)|404 \(Not Found\)|429.*rate limit|5[0-9]{2}.*Server Error|API rate limit exceeded|failed to download action|Unable to download|runner.*lost communication|runner.*terminated|Unable to resolve host|Connection timed out|Network is unreachable|Operation canceled|Connection reset|TLS handshake timeout|No space left on device|Out of memory|Disk quota exceeded)' /tmp/job_logs.txt; then
  echo "🔄 INFRASTRUCTURE FAILURE detected - will restart"
else
  echo "⚠️  CODE FAILURE - skip (not infrastructure)"
fi
```

**What gets restarted:**
- GitHub Actions/API failures (401, 403, 404, 429, 5xx errors)
- Git/checkout authentication failures
- Runner communication/termination issues
- Network timeouts and connectivity failures
- Resource exhaustion (disk, memory)

**What does NOT get restarted (code issues):**
- Test assertion failures (`expected X, got Y`)
- StandardRB/linting offenses
- Code coverage failures
- Security vulnerabilities
- Syntax errors, NoMethodError, NameError, etc.

**Note:** dd-trace-rb uses StandardRB for linting (not RuboCop).

### Step 6: Restart Infrastructure Failures

```bash
# Get the workflow run ID from the check
gh api repos/DataDog/dd-trace-rb/commits/<COMMIT_SHA>/check-runs \
  --jq '.check_runs[] | select(.name == "<CHECK_NAME>") | .id'

# Restart the failed job
gh run rerun <RUN_ID> --repo DataDog/dd-trace-rb --failed
```

### Step 7: Print Final Summary

After processing all PRs, print a summary table showing:
- PR number
- PR title
- Actions taken (All passing / Skipped / Restarted N jobs / No action)

```bash
# Print summary at the end
echo ""
echo "========================================="
echo "SUMMARY"
echo "========================================="
printf "%-8s | %-50s | %s\n" "PR #" "Title" "Actions"
echo "---------|-----------------------------------------------------|------------------"
# For each PR, print: PR number, title, and action summary
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
checks=$(gh pr checks <PR_NUMBER> --repo DataDog/dd-trace-rb --json name,state)

# Parse to see if only this check failed
echo "$checks" | jq 'map(select(.state == "FAILURE")) | length'
# If count is 1, check if it's the "All Required Checks Passed" job
echo "$checks" | jq -r 'map(select(.state == "FAILURE"))[0].name'
```

**Immediate restart:**
```bash
# If it's the only failure and it's the "All Required Checks Passed" job
gh run rerun <RUN_ID> --repo DataDog/dd-trace-rb
```

## Implementation Steps

When the skill is invoked:

1. **Initialize summary tracking:**
   ```bash
   # Create a temporary file to track PR summaries
   summary_file=$(mktemp)
   echo "PR_NUM|PR_TITLE|ACTION" > "$summary_file"
   ```

2. **Fetch all p-datadog PRs:**
   ```bash
   prs=$(gh pr list --repo DataDog/dd-trace-rb --label "p-datadog" --state open --json number,title,headRefOid)
   ```

3. **For each PR:**
   ```bash
   echo "$prs" | jq -c '.[]' | while read pr_data; do
     pr_num=$(echo "$pr_data" | jq -r '.number')
     pr_title=$(echo "$pr_data" | jq -r '.title')

     echo "Checking PR #$pr_num: $pr_title"

     # Check if PR has "no-restart" label - if so, skip it
     labels=$(gh pr view $pr_num --repo DataDog/dd-trace-rb --json labels --jq '.labels[].name')
     if echo "$labels" | grep -q "no-restart"; then
       echo "  ⏭️  Skipping PR #$pr_num (has 'no-restart' label)"
       echo "$pr_num|$pr_title|Skipped (no-restart label)" >> "$summary_file"
       continue
     fi

     # Get check status
     checks=$(gh pr checks $pr_num --repo DataDog/dd-trace-rb --json name,state,workflowName)

     # Find failed checks
     failed=$(echo "$checks" | jq -r '.[] | select(.state == "FAILURE")')

     if [ -z "$failed" ]; then
       echo "  ✅ All checks passing"
       echo "$pr_num|$pr_title|All checks passing" >> "$summary_file"
       continue
     fi

     # Track restart count for this PR
     restart_count=0

     # Check for "All Required Checks Passed" special case
     failed_count=$(echo "$checks" | jq '[.[] | select(.state == "FAILURE")] | length')
     if [ "$failed_count" -eq 1 ]; then
       failed_name=$(echo "$checks" | jq -r '[.[] | select(.state == "FAILURE")][0].name')
       if [[ "$failed_name" == *"Required"*"Checks"* ]] || [[ "$failed_name" == *"All"*"green"* ]]; then
         echo "  🔄 Only 'All Required Checks Passed' failing, restarting immediately"
         # Get run ID and restart
         # gh run rerun <RUN_ID> --repo DataDog/dd-trace-rb
         restart_count=$((restart_count + 1))
       fi
     fi

     # Investigate other failures
     echo "$failed" | jq -r '.name' | while read check_name; do
       echo "  ❌ Failed: $check_name"
       # Investigate and restart if infrastructure-related
       # If restarted: restart_count=$((restart_count + 1))
     done

     # Record summary for this PR
     if [ "$restart_count" -gt 0 ]; then
       echo "$pr_num|$pr_title|Restarted $restart_count job(s)" >> "$summary_file"
     else
       echo "$pr_num|$pr_title|Failed (no restarts)" >> "$summary_file"
     fi
   done
   ```

4. **For each failed check, get logs and analyze:**
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

5. **Print final summary:**
   ```bash
   echo ""
   echo "========================================="
   echo "SUMMARY"
   echo "========================================="
   printf "%-8s | %-60s | %s\n" "PR #" "Title" "Actions"
   echo "---------|--------------------------------------------------------------|------------------------"

   # Read and format summary from tracking file
   tail -n +2 "$summary_file" | while IFS='|' read pr_num pr_title action; do
     # Truncate title if too long
     if [ ${#pr_title} -gt 60 ]; then
       pr_title="${pr_title:0:57}..."
     fi
     printf "%-8s | %-60s | %s\n" "$pr_num" "$pr_title" "$action"
   done

   echo "========================================="

   # Cleanup
   rm -f "$summary_file"
   ```

## Infrastructure Failure Detection Reference

**CRITICAL:** All infrastructure failure patterns are defined in the bells repository:

**Source:** `https://github.com/p-datadog/bells/blob/master/docs/infrastructure-failure-detection.md`

**Summary of detected patterns:**
- GitHub Actions/API errors (401, 403, 404, 429, 5xx, rate limits, download failures)
- Git authentication failures (fatal errors, exit code 128)
- Runner communication/termination issues
- Network connectivity failures (timeouts, DNS, connection errors)
- Resource exhaustion (disk space, memory)

**DO NOT RESTART:**
- Test assertion failures (code issues)
- Linting/formatting offenses (StandardRB, etc.)
- Coverage failures
- Security vulnerabilities
- Syntax/runtime errors (NameError, NoMethodError, etc.)

**Note:** dd-trace-rb uses StandardRB for linting (not RuboCop).

See Step 5 for the complete regex pattern used for detection.

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

=========================================
SUMMARY
=========================================
PR #     | Title                                                        | Actions
---------|--------------------------------------------------------------|------------------------
4567     | Add dynamic instrumentation support                         | All checks passing
4568     | Fix probe serialization                                      | Restarted 2 job(s)
4569     | Update telemetry reporting                                   | Failed (no restarts)
4570     | Refactor probe manager                                       | Restarted 1 job(s)
4571     | Add new telemetry endpoint for dynamic instrumentation       | Skipped (no-restart label)
=========================================
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
gh pr checks <PR_NUMBER> --repo DataDog/dd-trace-rb --json name,state

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
