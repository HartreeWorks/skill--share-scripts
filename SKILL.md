---
name: share-scripts
description: This skill should be used when the user asks to "share a script", "make a script public", "publish a script", "share my script", or mentions making a Claude Code script available publicly. Publishes scripts from ~/.claude/scripts/ to the HartreeWorks/claude-scripts repository.
---

# Share Script

This skill publishes a Claude Code script to the public HartreeWorks/claude-scripts repository.

## What it does

1. Validates the script file exists in `~/.claude/scripts/`
2. **CRITICAL: Security & privacy review** - checks for credentials, hardcoded paths, and private information
3. Creates the public repo if it doesn't exist (first time only)
4. Copies the script to the repo
5. Updates the README index
6. Commits and pushes to GitHub

## Prerequisites

- GitHub CLI (`gh`) must be installed and authenticated
- The script must exist in `~/.claude/scripts/`

## Workflow

When the user asks to share a script, follow these steps:

### Step 1: Validate the script

```bash
SCRIPT_NAME="{script-filename}"  # With extension (e.g., script.py, script.sh)
SCRIPTS_DIR=~/.claude/scripts
SCRIPT_PATH="$SCRIPTS_DIR/$SCRIPT_NAME"

# Check script file exists
ls "$SCRIPT_PATH"
```

If the script doesn't exist, list available scripts:

```bash
ls "$SCRIPTS_DIR"
```

### Step 2: Security & privacy review (CRITICAL)

**This step is mandatory. Do NOT proceed to publishing without completing this review.**

#### 2a: Read and review the script file

```bash
cat "$SCRIPT_PATH"
```

**Look for:**

| Type | Examples | Replacement |
|------|----------|-------------|
| Client company names | Acme Corp, 80,000 Hours | HartreeWorks LTD |
| Real people's names | John Smith | Alice, Bob |
| Email addresses | john@client.com | alice@example.com |
| API keys or tokens | sk-xxx, token-xxx | {YOUR_API_KEY} or environment variable |
| Hardcoded private paths | /Users/john/secret/ | Relative paths or placeholders |
| Phone numbers | Real numbers | +44 20 1234 5678 |
| Hardcoded URLs with private domains | client.internal.com | example.com |

**Script-specific checks:**
- Hardcoded credentials or secrets
- Hardcoded absolute paths specific to your machine
- Private IP addresses or internal hostnames
- Database connection strings
- Any sensitive default values

#### 2b: Report findings and get approval

Present findings clearly and use AskUserQuestion:
```
question: "I've completed the security review. Should I make any suggested changes and proceed?"
header: "Review"
options:
  - label: "Apply changes & proceed"
    description: "Make all suggested replacements and continue publishing"
  - label: "Show me the changes first"
    description: "Display the exact edits before applying"
  - label: "Stop - I'll review manually"
    description: "Abort so you can review and edit files yourself"
```

**Only proceed after user explicitly approves.**

### Step 3: Ensure the public repo exists

```bash
REPO_DIR="$SCRIPTS_DIR/../skills/share-scripts/claude-scripts"

# Check if we have a local clone
if [ -d "$REPO_DIR" ]; then
    cd "$REPO_DIR"
    git pull origin main
else
    # Check if remote repo exists
    if gh repo view HartreeWorks/claude-scripts --json url 2>/dev/null; then
        # Clone existing repo
        gh repo clone HartreeWorks/claude-scripts "$REPO_DIR"
    else
        # Create the repo for the first time
        mkdir -p "$REPO_DIR"
        cd "$REPO_DIR"
        git init

        # Create initial README
        cat > README.md << 'EOF'
# Claude Scripts

A collection of utility scripts for [Claude Code](https://claude.com/claude-code).

These scripts are used with Claude Code hooks, commands, and other automation workflows.

## Installation

Copy any script to your `~/.claude/scripts/` directory and ensure it's executable:

```bash
chmod +x ~/.claude/scripts/script-name.py
```

## Available scripts

| Script | Description |
|--------|-------------|

## About

Created by [Peter Hartree](https://x.com/peterhartree). For updates, follow [AI Wow](https://wow.pjh.is), my AI uplift newsletter.

Find more Claude Code resources at [HartreeWorks](https://github.com/HartreeWorks).
EOF

        git add README.md
        git commit -m "Initial commit"

        # Create public repo
        gh repo create HartreeWorks/claude-scripts --public --source=. --remote=origin --push
    fi
fi
```

### Step 4: Copy the script to the repo

```bash
cd "$REPO_DIR"
cp "$SCRIPT_PATH" ./
```

### Step 5: Update the README index

Extract a description from the script's docstring or header comment. For Python scripts, look for the module docstring. For shell scripts, look for a comment block at the top.

Edit `README.md` to add a row to the "Available scripts" table in alphabetical order:

```markdown
| [{script-name}](./{script-name}) | {description} |
```

Keep the table sorted alphabetically by script name.

### Step 6: Commit and push

```bash
cd "$REPO_DIR"
git add .
git commit -m "Add $SCRIPT_NAME

$(head -20 "$SCRIPT_NAME" | grep -E '^#|^"""' | head -3)

Generated with [Claude Code](https://claude.com/claude-code)"
git push origin main
```

### Step 7: Confirm success

Output:

```
Done! Script "{script-name}" is now public.

Repository: https://github.com/HartreeWorks/claude-scripts

Others can install it by copying the script to their ~/.claude/scripts/ directory.
```

## Error handling

| Error | Cause | Solution |
|-------|-------|----------|
| Script not found | Typo in name | List available scripts with `ls ~/.claude/scripts/` |
| Privacy review failed | User chose to stop | User reviews manually and runs share again |
| gh auth error | Not logged in | Run `gh auth login` |
| Push failed | Network issue | Retry with `git push` |

## Example usage

User: "Share the bash-safety-hook.py script"

1. Validate: `~/.claude/scripts/bash-safety-hook.py` exists
2. Security review:
   - Read script file
   - Check for private paths, API keys, personal info
   - Report findings and get approval
3. User approves
4. Ensure repo exists (create if first time)
5. Copy `bash-safety-hook.py` to repo
6. Update README table with script description
7. Commit and push
8. Report success with repo URL

## Notes

- Scripts can be any executable format (`.py`, `.sh`, `.js`, etc.)
- Target repository: `HartreeWorks/claude-scripts`
- Local clone kept at `~/.claude/skills/share-scripts/claude-scripts/`
- **Always complete security review before publishing**
- Script descriptions extracted from docstrings or header comments

## Update check

This is a shared skill. Before executing, check `~/.claude/skills/.update-config.json`.
If `auto_check_enabled` is true and `last_checked_timestamp` is older than `check_frequency_days`,
mention: "It's been a while since skill updates were checked. Run `/update-skills` to see available updates."
Do NOT perform network operations - just check the local timestamp.
