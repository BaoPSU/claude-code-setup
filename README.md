# Claude Code Setup Guide

Everything you need to get Claude Code running the right way — flags, tokens, GitHub integration, web tools, and CLAUDE.md config.

---

## Table of Contents

1. [Install Claude Code](#install)
2. [Key Flags](#key-flags)
3. [GitHub Setup & Tokens](#github-setup)
4. [CLAUDE.md Configuration](#claudemd)
5. [Remote Control & Web Access](#remote-control)
6. [Adding Images via Web](#images)
7. [Tips & Patterns](#tips)

---

## Install

```bash
npm install -g @anthropic/claude-code
```

Or with npx (no install):

```bash
npx @anthropic/claude-code
```

Set your API key:

```bash
export ANTHROPIC_API_KEY=sk-ant-...
# Add to ~/.bashrc or ~/.zshrc to persist
```

---

## Key Flags

### `--dangerously-skip-permissions`

Skips all permission prompts — Claude will read, write, run commands, and push without asking first. Use this when you trust the task fully or are running in a container/CI where prompts would hang.

```bash
claude --dangerously-skip-permissions
```

**Combine with a one-shot prompt:**

```bash
claude --dangerously-skip-permissions "fix all lint errors and commit"
```

> **Warning:** This gives Claude full autonomy. Only use on repos you control and when you understand what the task will do.

---

### `--continue`

Picks up your most recent conversation without re-stating context. Useful when you close the terminal and come back.

```bash
claude --continue
```

Equivalent to typing `/continue` inside a session.

---

### `--resume`

Lets you choose from a list of past sessions to resume — not just the most recent one.

```bash
claude --resume
```

You'll see a numbered list of prior conversations. Pick one and continue from exactly where you left off, including all prior context and tool results.

---

### Combining Flags

```bash
# Resume last session and skip all permission prompts
claude --continue --dangerously-skip-permissions

# One-shot non-interactive task, no prompts
claude --dangerously-skip-permissions -p "add docstrings to all functions in src/"
```

---

## GitHub Setup

### 1. Install gh CLI

```bash
# Ubuntu/Debian
sudo apt install gh

# Mac
brew install gh
```

### 2. Authenticate

```bash
gh auth login
# Choose: GitHub.com → HTTPS → Login with browser (or paste token)
```

### 3. Create a Personal Access Token (PAT)

Go to: **GitHub → Settings → Developer Settings → Personal Access Tokens → Tokens (classic)**

Scopes needed:
- `repo` — full repo access (read, write, push)
- `read:org` — if working with org repos
- `gist` — optional

```bash
# Store token so Claude and git can use it
echo "ghp_yourTokenHere" > ~/.github_token
chmod 600 ~/.github_token

# Tell git to use it
git config --global credential.helper store
echo "https://BaoPSU:$(cat ~/.github_token)@github.com" > ~/.git-credentials
```

### 4. Create and Push a Repo

```bash
# Initialize local repo
git init my-project
cd my-project

# Create on GitHub and link it
gh repo create my-project --public --source=. --remote=origin --push

# Or for private
gh repo create my-project --private --source=. --remote=origin --push
```

### 5. Subsequent Pushes

```bash
git add .
git commit -m "update"
git push
```

No password prompt if your credentials are stored.

---

### Giving Claude Push Access

In your CLAUDE.md or settings, allow git push:

```json
// .claude/settings.json
{
  "permissions": {
    "allow": [
      "Bash(git push*)",
      "Bash(gh *)"
    ]
  }
}
```

Or run with `--dangerously-skip-permissions` for full autonomy.

---

## CLAUDE.md

`CLAUDE.md` is the brain of your project for Claude. Put it at the project root. Claude reads it at the start of every session.

### Basic Template

```markdown
# Project: My App

## What this is
Brief description of the project.

## Stack
- Language: Python 3.11
- Framework: FastAPI
- DB: PostgreSQL

## How to run
\`\`\`bash
pip install -r requirements.txt
uvicorn main:app --reload
\`\`\`

## Rules
- Never use em dashes in writing
- Always write tests for new functions
- Commit messages: imperative mood, under 72 chars

## My writing style (for docs/comments)
- Lead with numbers and results
- Punchy endings, no textbook language
- Use plain connectors: "so", "then", "for example"
```

### Layered CLAUDE.md (Global + Per-Project)

**Global** — `~/.claude/CLAUDE.md`  
Applies to every session. Good for personal style rules, token locations, preferred tools.

```markdown
# Global Rules

- GitHub token: ~/.github_token
- Never use em dashes
- Default model: claude-opus-4-7
- Always use absolute paths in scripts
```

**Project** — `./CLAUDE.md`  
Overrides or extends global. Project-specific commands, stack, team conventions.

**Sub-directory** — `./src/CLAUDE.md`  
Scoped to that folder. Useful for monorepos where different parts have different rules.

### Useful CLAUDE.md Patterns

```markdown
## Do not touch
- legacy/old_api.py  ← do not refactor without asking

## Preferred commands
- Lint: ruff check .
- Test: pytest -x
- Build: docker build -t app .

## Token locations
- GitHub: ~/.github_token
- HuggingFace: ~/.hf_token

## Context on key files
- src/models.py — core data models, don't change field names without migration
- config/prod.yaml — production config, never commit secrets here
```

---

## Remote Control & Web Access

Claude Code can browse the web, fetch URLs, and run in remote/headless mode.

### Web Fetch in a Session

Just ask Claude to fetch a URL:

```
fetch https://docs.anthropic.com/en/docs/claude-code
```

Or in your prompt:

```bash
claude "read https://example.com/api-docs and write a Python client for it"
```

Claude will use its WebFetch tool to pull the page and work with it.

### Web Search

Claude can search the web mid-task:

```
search for the latest numpy release notes
```

Or:

```bash
claude "search for how to configure PX4 SITL with Gazebo 11 and summarize the steps"
```

### Headless / Remote Mode

Run Claude Code as a background agent — no interactive terminal needed:

```bash
# Non-interactive, one-shot
claude -p "run tests and open a PR if they pass" --dangerously-skip-permissions

# Pipe input
echo "summarize CHANGELOG.md" | claude --dangerously-skip-permissions
```

### Scheduled / Automated

Use `claude --schedule` or invoke via cron:

```bash
# crontab -e
0 9 * * 1 cd /home/bao/project && claude --dangerously-skip-permissions -p "check for dependency updates and open a PR"
```

---

## Adding Images via Web

### Reference a URL in your prompt

Claude can read images hosted online:

```
Look at this screenshot: https://i.imgur.com/example.png — what's wrong with the layout?
```

```bash
claude "here is the circuit diagram: https://example.com/schematic.png — identify the voltage divider"
```

### Upload a local image

Drag and drop images directly into the Claude Code terminal (supported in the desktop app), or reference a local path:

```
/path/to/screenshot.png — why is the button misaligned?
```

### In CLAUDE.md for context

```markdown
## Reference images
- Architecture diagram: https://raw.githubusercontent.com/BaoPSU/my-project/main/docs/arch.png
- PCB layout: ./docs/pcb_layout.png
```

---

## Tips & Patterns

### Minimal friction setup

```bash
# ~/.bashrc or ~/.zshrc
export ANTHROPIC_API_KEY=sk-ant-...
export GITHUB_TOKEN=$(cat ~/.github_token)
alias cc="claude --continue"
alias ccf="claude --continue --dangerously-skip-permissions"
alias cr="claude --resume"
```

### Let Claude manage git for you

In `.claude/settings.json`:

```json
{
  "permissions": {
    "allow": [
      "Bash(git *)",
      "Bash(gh *)",
      "Bash(npm *)",
      "Bash(python *)"
    ]
  }
}
```

Then just say: `commit and push everything` and Claude handles it.

### Per-project settings

```
my-project/
├── CLAUDE.md              ← instructions for Claude
├── .claude/
│   └── settings.json      ← permissions and hooks
└── src/
    └── CLAUDE.md          ← sub-directory overrides
```

### Hooks — automate on events

`.claude/settings.json`:

```json
{
  "hooks": {
    "Stop": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "echo 'Claude session ended' >> ~/.claude_log"
          }
        ]
      }
    ]
  }
}
```

Run a script after every Claude session stops, before a tool runs, etc.

---

## Quick Reference

| Task | Command |
|------|---------|
| Start fresh | `claude` |
| Continue last session | `claude --continue` |
| Pick a past session | `claude --resume` |
| No permission prompts | `claude --dangerously-skip-permissions` |
| One-shot task | `claude -p "do the thing"` |
| Skip prompts + one-shot | `claude --dangerously-skip-permissions -p "..."` |
| Check auth | `gh auth status` |
| Create repo + push | `gh repo create name --public --source=. --push` |
| View sessions | `claude --resume` |

---

## Resources

- [Claude Code Docs](https://docs.anthropic.com/en/docs/claude-code)
- [CLAUDE.md Reference](https://docs.anthropic.com/en/docs/claude-code/memory)
- [Settings & Permissions](https://docs.anthropic.com/en/docs/claude-code/settings)
- [GitHub CLI Docs](https://cli.github.com/manual/)
