# spn - Claude Code Skills for Datadog dd-trace-rb

A collection of Claude Code skills for working with Datadog's dd-trace-rb repository.

## Coding Guidelines

See `CLAUDE.md` for coding standards and guidelines when working with dd-trace-rb, including:
- Trailing comma requirements for di/ and symbol_database/ directories
- Code style conventions
- Best practices

## Skills

### 1. `di-pr-review` - Review Dynamic Instrumentation PRs
Reviews PRs against 6 critical DI requirements for the dd-trace-rb repository.

**Usage:**
```bash
/di-pr-review <PR_URL>
```

**Example:**
```bash
/di-pr-review https://github.com/DataDog/dd-trace-rb/pull/5434
```

**What it checks:**
- NO skipped tests
- NO sleep in tests
- 100% code coverage
- Repository guidelines compliance (CLAUDE.md)
- NO exception propagation (error boundaries)
- Proper error handling (logging + telemetry)
- Line probe test files (line number verification)
- Trailing commas in di/ and symbol_database/

### 2. `di-pr-respond` - Respond to PR Review Comments
Systematically processes PR review comments, makes requested changes, finds and fixes similar issues throughout the codebase, and responds to each comment.

**Usage:**
```bash
/di-pr-respond <PR_NUMBER>
```

**What it does:**
- Fetches all review comments from a PR
- For change requests: Fixes the issue AND finds similar patterns to fix proactively
- For questions: Provides clear answers with code references
- For disagreements: Explains reasoning respectfully with evidence
- Commits each change separately with descriptive messages
- Responds inline to each comment (never resolves them)
- Pushes all commits after addressing all comments

### 3. `pr-fix-lint` - Fix Non-Test CI Failures
Automatically fixes lint, static analysis, type checking, and formatting failures in a PR.

**Usage:**
```bash
/pr-fix-lint <PR_NUMBER>
```

**What it fixes:**
- **Linting:** RuboCop, ESLint, Pylint, etc.
- **Type Checking:** TypeScript, Steep (Ruby), Sorbet, mypy
- **Formatting:** Prettier, Black, gofmt
- **Static Analysis:** Brakeman, CodeQL
- **Steep-specific rules:**
  - Fix genuine code issues revealed by type errors
  - Silence unclear errors with steep:ignore directives
  - NEVER change code only to satisfy steep
  - Known false positives: type narrowing, cross-scope assignments, multi-type containers

**Process:**
- Identifies non-test CI failures
- Applies auto-fixes where available
- Commits each fix type separately
- Pushes all commits at the end

### 4. `pr-fix-tests` - Fix Test Failures (with Continuous Monitoring)
Automatically fixes test failures in a PR and continuously monitors CI until all tests pass.

**Usage:**
```bash
/pr-fix-tests <PR_NUMBER>
```

**What it does:**
- Identifies test CI failures (RSpec, Jest, pytest, etc.)
- Analyzes failure logs to determine root cause
- Fixes each test failure (updates tests or source code)
- Commits each fix separately with descriptive messages
- Pushes commits to PR branch
- **Continuously monitors CI status** (polls every 1 minute)
- Fixes any new or remaining test failures that appear in CI
- Keeps working until all tests pass or 10 fix rounds reached

**Test failure patterns:**
- Outdated expectations
- Missing setup
- Async timing issues
- Changed method signatures
- Code bugs

### 5. `crcj` - Check and Restart CI Jobs
Automatically checks CI status for PRs by the p-datadog user and restarts failed jobs that are infrastructure-related.

**Usage:**
```bash
/crcj
```

**What it does:**
- Finds all open PRs by the `p-datadog` user
- Checks CI job status for each PR
- Skips PRs with >10 failing test jobs (likely code issues)
- Investigates failed jobs to determine if they're infrastructure-related
- Restarts jobs that failed due to GitHub/network/transient issues
- Special handling: If only the `all-jobs-are-green` job is failing, restarts it immediately

## Installation

### Option 1: Use from this directory (Current session)
If you're already in this directory, the skills are automatically available.

### Option 2: Clone and use in any session

1. **Clone the repository:**
   ```bash
   git clone https://github.com/p-datadog/spn.git
   cd spn
   ```

2. **Start Claude Code in this directory:**
   ```bash
   claude
   ```

3. **Use the skills:**
   ```bash
   /di-pr-review <PR_URL>
   /di-pr-respond <PR_NUMBER>
   /pr-fix-lint <PR_NUMBER>
   /pr-fix-tests <PR_NUMBER>
   /crcj
   ```

### Option 3: Use from another directory

You can reference skills from another directory using the full skill path. In your Claude Code session:

```bash
# Navigate to your working directory
cd /path/to/your/project

# Use the skill with full plugin name
/spn:di-pr-review <PR_URL>
/spn:di-pr-respond <PR_NUMBER>
/spn:pr-fix-lint <PR_NUMBER>
/spn:pr-fix-tests <PR_NUMBER>
/spn:crcj
```

**Note:** You may need to configure Claude Code to load plugins from custom locations. Check Claude Code documentation for plugin paths.

## How It Works

Claude Code detects plugins by looking for:
- A `.claude-plugin/` directory with a `plugin.json` file
- A `skills/` directory containing skill definitions (SKILL.md files)

When you start Claude Code in a directory with this structure, the skills become available as slash commands.

## Sharing Across Multiple Projects

To use these skills across multiple projects:

1. **Keep the plugin in a central location:**
   ```bash
   # Clone to a central location
   git clone https://github.com/p-datadog/spn.git ~/claude-plugins/spn
   ```

2. **Reference from any project:**
   - Option A: Symlink the plugin directory into your project
   - Option B: Use the full plugin name (`/spn:skill-name`)
   - Option C: Configure Claude Code's plugin search paths (if supported)

## Updating the Skills

When new features are added to the skills:

```bash
cd /path/to/spn
git pull origin master
```

The updated skills will be available in your next Claude Code session.

## Contributing

To add or modify skills:

1. Edit or create skill files in `skills/<skill-name>/SKILL.md`
2. Update the plugin version in `.claude-plugin/plugin.json`
3. Commit and push your changes
4. Use `git pull` in other sessions to get updates

## Skill Structure

Each skill is defined in `skills/<skill-name>/SKILL.md` with this format:

```markdown
---
name: skill-name
description: Brief description used for triggering the skill
version: 0.1.0
---

# Skill Title

Detailed instructions and documentation for Claude to follow when executing this skill...
```

## Quick Reference

**Which skill should I use?**

| Task | Skill | Command |
|------|-------|---------|
| Review a DI PR for quality standards | `di-pr-review` | `/di-pr-review <PR_URL>` |
| Respond to PR review comments | `di-pr-respond` | `/di-pr-respond <PR_NUMBER>` |
| Fix lint/type check/formatting failures | `pr-fix-lint` | `/pr-fix-lint <PR_NUMBER>` |
| Fix test failures (with CI monitoring) | `pr-fix-tests` | `/pr-fix-tests <PR_NUMBER>` |
| Restart failed CI jobs for p-datadog PRs | `crcj` | `/crcj` |

**Typical workflow:**

1. Create PR with new code
2. Wait for CI to run
3. Use `/pr-fix-lint <PR_NUMBER>` to fix any linting/type checking issues
4. Use `/pr-fix-tests <PR_NUMBER>` to fix any test failures (monitors CI until all pass)
5. Use `/di-pr-review <PR_URL>` to verify PR meets DI quality standards
6. Wait for human review
7. Use `/di-pr-respond <PR_NUMBER>` to address review comments
8. Use `/crcj` to restart any infrastructure-related CI failures

## Repository

- **GitHub:** https://github.com/p-datadog/spn
- **Issues:** https://github.com/p-datadog/spn/issues

## License

MIT
