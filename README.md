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

### 2. `crcj` - Check and Restart CI Jobs
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
   /crcj
   ```

### Option 3: Use from another directory

You can reference skills from another directory using the full skill path. In your Claude Code session:

```bash
# Navigate to your working directory
cd /path/to/your/project

# Use the skill with full plugin name
/spn:di-pr-review <PR_URL>
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

## Repository

- **GitHub:** https://github.com/p-datadog/spn
- **Issues:** https://github.com/p-datadog/spn/issues

## License

MIT
