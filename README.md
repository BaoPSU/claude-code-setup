# Claude Code Setup Guide

Everything you need to get Claude Code running the right way — flags, tokens, GitHub integration, web tools, and CLAUDE.md config.

---

## Table of Contents

1. [How Claude Code Actually Works](#how-it-works)
2. [Install Claude Code](#install)
3. [Key Flags](#key-flags)
4. [GitHub Setup & Tokens](#github-setup)
5. [CLAUDE.md Configuration](#claudemd)
6. [Remote Control & Web Access](#remote-control)
7. [Adding Images via Web](#images)
8. [Saving Tokens with /compact](#compact)
9. [Working with Repos — Don't Just Paste the URL](#repos-warning)
10. [Lecture Summaries in CLAUDE.md](#lecture-summaries)
11. [LaTeX Cheat Sheets](#latex)
12. [Recommended File Organization](#file-org)
13. [Tone & Explanation Style in CLAUDE.md](#tone)
14. [Using Claude as a TA](#ta)
15. [Tips & Patterns](#tips)

---

## How Claude Code Actually Works

Before anything else — understanding this makes every other section in this guide make sense.

### The session model

Every time you start Claude Code, it spins up a fresh session. It has no memory of your last conversation by default. It does not remember what files you edited yesterday, what you told it about your project, or what rules you set up. Every session starts blank.

This sounds annoying. The fix is CLAUDE.md.

### What fires at the start of every session

The very first thing Claude does when you open a session is look for and read CLAUDE.md files. It reads them in this order:

```
1. ~/.claude/CLAUDE.md          ← your global rules (applies everywhere)
2. ./CLAUDE.md                  ← project root (applies to this repo)
3. ./src/CLAUDE.md              ← subfolder (applies when working in src/)
```

Each level adds to the one above. Subfolder rules stack on top of project rules, which stack on top of global rules. If there's a conflict, the closer one wins.

**This means: anything you write in a CLAUDE.md file is automatically loaded into Claude's context before it reads a single message from you.** It's already "briefed" by the time you type your first word.

### What this means practically

Without CLAUDE.md, every session sounds like this:

> "Hey Claude, I'm working on my ECE332 cheat sheet, it's a 6pt 3-column LaTeX file, only edit the Exam1 file not the HW ones, always compile from the src/ subfolder, the images are in img/..."

With CLAUDE.md, Claude already knows all of that. You just say:

> "Add a section on skin depth."

And it does it correctly without you explaining the format, the file, the compile path, or anything else.

### The CLAUDE.md is your persistent brain for Claude

Think of it as a briefing document that gets handed to Claude at the start of every shift. A good CLAUDE.md answers:

- What is this project?
- What files are you allowed to touch?
- What files must you never touch?
- What format / style / rules apply?
- What tools / commands do you use here?
- How should you explain things to me?

Everything in this guide — cheat sheet rules, TA mode, explanation style, file organization — gets wired in through CLAUDE.md. That's why it's the most important thing to set up correctly.

### Memory across sessions

Claude does not carry memory between sessions on its own. Your options:

| Method | What it does |
|--------|-------------|
| `CLAUDE.md` | Permanent facts about the project — always loaded |
| `--continue` | Resume the most recent session's conversation history |
| `--resume` | Pick any past session to resume from |
| `/compact` | Compress a long session to save tokens, keeps going |

For anything you want Claude to know forever (project structure, rules, your explanation style), put it in CLAUDE.md. For anything you want it to remember just from earlier today, use `--continue`.

### How reading order affects what Claude knows

Claude reads files in context order — earlier context can be overridden by later context. This is why:

- Global `~/.claude/CLAUDE.md` sets defaults
- Project `./CLAUDE.md` overrides or extends them
- Subfolder `./Notes/CLAUDE.md` overrides again for that specific folder

So if your global CLAUDE.md says "always ask before running git push" but your project CLAUDE.md says "git push is allowed" — in that project, it's allowed. Closer always wins.

### The session startup sequence, in order

```
1. Read ~/.claude/CLAUDE.md            (global rules)
2. Read ./CLAUDE.md                    (project rules)
3. Read any subfolder CLAUDE.md        (scoped rules)
4. Read .claude/settings.json          (permissions)
5. Now Claude reads your first message
```

By the time you type anything, Claude has already loaded your entire briefing. That's the whole trick. Write good CLAUDE.md files and you never have to re-explain yourself.

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

## Saving Tokens with /compact

Every message in a session costs tokens — and long sessions accumulate fast. Use `/compact` to compress prior conversation history into a summary while keeping the important context, so you can keep working without blowing through your limits.

### When to use it

- After a big chunk of work is done (e.g. you finished debugging a module) and you're switching to a new task
- When the session has gotten long and responses start feeling slower or more expensive
- Before switching topics — compact the old context so only the relevant summary carries forward
- Any time you see the context usage bar filling up

### How to use it

Type in the session:

```
/compact
```

Claude summarizes everything so far, drops the raw back-and-forth, and continues from the summary. You lose the raw transcript but keep the meaning.

### Compact with a custom focus

You can tell Claude what to preserve:

```
/compact focus on the database schema changes and ignore the debugging tangents
```

This keeps the summary tight around what actually matters.

### Token-saving habits

- `/compact` mid-session before switching tasks
- Keep CLAUDE.md focused — long CLAUDE.md files are read every session
- Use `-p` for one-shot tasks instead of interactive sessions when you don't need back-and-forth
- Don't paste entire files into chat — tell Claude to read them with a file path instead

---

## Working with Repos — Don't Just Paste the URL

**Do not just hand Claude a GitHub repo URL and say "look at this."** Claude will attempt to read every file it can find, burn through your context window, and may still miss what you actually needed.

### The problem

```
# BAD — Claude will spider the repo and read everything
"Here's my project: https://github.com/BaoPSU/my-project — help me fix the bug"
```

This causes Claude to read README, every source file, configs, lock files, test fixtures — most of it irrelevant. You'll hit context limits fast and the actual answer gets buried.

### What to do instead

**Tell Claude exactly which files matter:**

```
# GOOD — specific files, specific question
"Look at src/models.py and src/routes/auth.py — the login endpoint is returning 403 for valid tokens"
```

**Or generate a summary first and pin it in CLAUDE.md:**

```markdown
## Repo structure (key files only)
- src/models.py — User, Session, Token data models
- src/routes/auth.py — login/logout/refresh endpoints
- config/settings.py — env vars and app config
- tests/test_auth.py — auth test suite

## Do NOT read unless asked
- node_modules/, .venv/, dist/, *.lock files
- legacy/ folder — old code, not in use
```

**Use Claude to generate that summary once, then reuse it:**

```
Read the top-level folder structure and the first 30 lines of each file in src/ — 
write me a one-paragraph summary of each file I can paste into CLAUDE.md
```

Now every future session starts with a map, not a full read.

### For large repos

```markdown
## Key entry points
- main.py — app startup, reads config, starts server
- src/api/ — all HTTP handlers, one file per resource
- src/db/ — database models and migrations (SQLAlchemy)
- src/services/ — business logic, called by handlers

## Where NOT to start
- Do not read src/vendor/ or scripts/legacy/ without explicit instruction
```

---

## Lecture Summaries in CLAUDE.md

If you're using Claude Code for coursework, you can build a running knowledge base of your lecture material directly in CLAUDE.md. The key is to make summaries **detailed enough to actually be useful** — not just bullet points. Include images, diagrams, and context so Claude can answer questions, write assignments, and explain concepts with the real course material as its reference.

### Structure

```markdown
## Course: ECE 410 — Machine Learning Hardware

### Lecture 1 — Intro to Neural Network Accelerators
**Key concepts:** MAC operations, systolic arrays, roofline model  
**Summary:**  
Covered why CPUs are bad at deep learning (serial execution, cache bottlenecks) vs. purpose-built accelerators like TPUs and NPUs that exploit data parallelism. The roofline model shows whether your design is compute-bound or memory-bound — most real workloads are memory-bound, so bandwidth matters more than raw FLOPS.

**Diagram:**  
![Roofline Model](https://raw.githubusercontent.com/BaoPSU/ECE410/main/lectures/L1_roofline.png)

**Formulas:**  
- Arithmetic intensity = FLOPs / bytes accessed  
- If intensity > ridge point → compute-bound; else → memory-bound

**What to remember:**  
Systolic array reuse pattern: each PE passes data to its neighbor so every weight is used before being evicted. That's the whole trick.

---

### Lecture 2 — Quantization
**Key concepts:** INT8 vs FP32, scale factors, quantization-aware training  
**Summary:**  
Reducing weight precision from FP32 → INT8 cuts memory by 4x and speeds up inference on hardware that has INT8 units (most modern accelerators do). The tradeoff is accuracy loss, which you recover through quantization-aware training (QAT) — simulating low precision during training so the model learns to be robust to it.

**Image:**  
![Quantization Error](https://raw.githubusercontent.com/BaoPSU/ECE410/main/lectures/L2_quant.png)

**Key numbers from lecture:**  
- FP32: 4 bytes/weight, INT8: 1 byte/weight  
- Post-training quantization (PTQ): fast but ~1-2% accuracy drop  
- QAT: slower but recovers most accuracy

**My notes:**  
Zero-point offset matters for asymmetric quantization — don't forget it when converting back.
```

### Tips for detailed summaries

- **Images are key** — host them on GitHub (raw URL) or Imgur and embed with `![alt](url)`. Claude can see them in future sessions.
- **Include actual numbers** — percentages, latencies, error rates from the slides. Generic summaries are useless.
- **Add a "what to remember" or "my notes" field** — the non-obvious insight, not just what the slide said.
- **Formula blocks** — math-heavy courses, write out the key equations explicitly so Claude has them verbatim.
- **Cross-reference lectures** — if Lecture 5 builds on Lecture 2, say so: `See also: Lecture 2 — Quantization`.

### Generating summaries from slides

If you have the PDF or images:

```
Here are slides from Lecture 3: [attach PDF or images]
Write a detailed CLAUDE.md-style summary following this format:
- Key concepts (one line)
- Summary paragraph (what was actually taught, not just topic names)
- Key formulas or numbers
- What to remember (the non-obvious insight)
Include any diagrams as image references if I give you the file paths.
```

Then paste the output directly into CLAUDE.md under the right lecture heading.

---

## LaTeX Cheat Sheets

The real template below is pulled directly from `ECE332-EMAG-II-Portland-State-University` — a working 6pt, 3-column, two-sided exam cheat sheet with colored section headers, framed content boxes, and a bottom reference strip. Use this as your base and adapt the colors and content per course.

### Real working preamble (ECE332 EMAG II)

```latex
\documentclass[6pt,letterpaper]{article}
\usepackage[margin=0.25in, top=0.3in, bottom=0.25in]{geometry}
\usepackage{amsmath,amssymb,multicol,xcolor,mdframed}
\usepackage[expansion=false]{microtype}   % required at sub-6pt — see microtype note below
\usepackage{fancyhdr,booktabs,colortbl,array,graphicx}

% ── Colors (bg + text pair for each topic) ──
\definecolor{purple}{HTML}{534AB7}\definecolor{purplebg}{HTML}{EEEDFE}
\definecolor{teal}{HTML}{0F6E56}\definecolor{tealbg}{HTML}{E1F5EE}
\definecolor{coral}{HTML}{993C1D}\definecolor{coralbg}{HTML}{FAECE7}
\definecolor{amber}{HTML}{854F0B}\definecolor{amberbg}{HTML}{FAEEDA}
\definecolor{green}{HTML}{3B6D11}\definecolor{greenbg}{HTML}{EAF3DE}
\definecolor{blue}{HTML}{0C447C}\definecolor{bluebg}{HTML}{E6F1FB}
\definecolor{gray}{HTML}{444441}\definecolor{graybg}{HTML}{F1EFE8}
\definecolor{pink}{HTML}{72243E}\definecolor{pinkbg}{HTML}{FBEAF0}
\definecolor{rowB}{HTML}{F2F2EF}   % alternating table row tint

% ── Content box (ebox) ──
\newmdenv[linewidth=0.4pt,innerleftmargin=3pt,innerrightmargin=3pt,
  innertopmargin=2pt,innerbottommargin=2pt,skipabove=1pt,skipbelow=1pt]{ebox}

% ── Section header: \shead{bgcolor}{textcolor}{TITLE} ──
\newcommand{\shead}[3]{%
  \noindent\colorbox{#1}{\parbox{\dimexpr\linewidth-2\fboxsep\relax}%
  {\color{#2}\bfseries\fontsize{6}{7}\selectfont #3}}\vspace{0.5pt}}

% ── Equation label: \eq{label}{when-to-use} ──
\newcommand{\eq}[2]{\noindent{\color{gray}\bfseries\fontsize{5.8}{6}\selectfont#1}%
  \enspace{\color{teal}\fontsize{5.5}{6}\selectfont\textit{#2}}}

\setlength{\parindent}{0pt}\setlength{\parskip}{0pt}
\setlength{\columnsep}{5pt}\setlength{\multicolsep}{1pt}

\pagestyle{fancy}\fancyhf{}
\renewcommand{\headrulewidth}{0.3pt}
\fancyhead[L]{\fontsize{6.5}{7}\selectfont\textbf{ECE332 --- Exam 1 Cheat Sheet}}
\fancyhead[R]{\fontsize{6.5}{7}\selectfont Your Name}

% images live in Midterm1/img/ — compile from src/ subfolder
\graphicspath{{../img/}}
```

### Color scheme — assign one per topic

Pick a color pair and use it consistently for the same topic across the whole sheet. This makes it scannable at a glance.

| Color  | Background  | Text    | Use for                        |
|--------|-------------|---------|--------------------------------|
| purple | `purplebg`  | `purple`| Core laws, key definitions     |
| teal   | `tealbg`    | `teal`  | Field relationships, tables    |
| coral  | `coralbg`   | `coral` | Lossy media, warnings          |
| amber  | `amberbg`   | `amber` | Rotating/circular, generators  |
| green  | `greenbg`   | `green` | Power, Poynting, energy        |
| blue   | `bluebg`    | `blue`  | Reference comparisons          |
| gray   | `graybg`    | `gray`  | Quick-ref, step-by-step        |
| pink   | `pinkbg`    | `pink`  | Phasor/complex math            |

### Body layout — 3-column with bottom reference strip

```latex
\begin{document}
\fontsize{6}{7.5}\selectfont
\setlength{\abovedisplayskip}{1pt}\setlength{\belowdisplayskip}{1.5pt}
\setlength{\abovedisplayshortskip}{0pt}\setlength{\belowdisplayshortskip}{0pt}

\begin{multicols}{3}

\shead{purplebg}{purple}{SECTION TITLE}
\begin{ebox}
\eq{Formula name}{when to use this}
\[V_\text{emf} = -\frac{d\Phi}{dt}\]
\textit{short note on the equation}
\end{ebox}

\vspace{2pt}
\shead{tealbg}{teal}{NEXT SECTION}
\begin{ebox}
...
\end{ebox}

\columnbreak   % ← force column break here; use 2 per page for 3-col layout

\end{multicols}

% ── Bottom reference strip: 5 equal minipages ──
\vfill
\noindent
\begin{minipage}[t]{0.19\linewidth}
\shead{purplebg}{purple}{VARIABLES}
\begin{ebox}
$\mathbf{E}$ — electric field (V/m)\\
$\mathbf{B}$ — mag flux density (T)\\
...
\end{ebox}
\end{minipage}\hfill
\begin{minipage}[t]{0.19\linewidth}
\shead{tealbg}{teal}{CONSTANTS}
\begin{ebox}
$\varepsilon_0=8.85\times10^{-12}$ F/m\\
$\mu_0=4\pi\times10^{-7}$ H/m\\
$c=3\times10^8$ m/s\\
...
\end{ebox}
\end{minipage}\hfill
% ... repeat for columns 3–5

\newpage
% PAGE 2 — tighter spacing for dense material
\fontsize{6}{7}\selectfont
\setlength{\abovedisplayskip}{0pt}\setlength{\belowdisplayskip}{0.5pt}
\begin{multicols}{3}
...
\end{multicols}
\end{document}
```

### Table pattern — required to prevent rowcolor bleed

Every table with `\rowcolor` rows **must** use this exact wrapper. Without `\makebox`, the colored row fill bleeds to the full column width.

```latex
{\centering\makebox[0.98\linewidth][c]{{\setlength{\tabcolsep}{2pt}\begin{tabular}{@{}p{0.28\linewidth}p{0.66\linewidth}@{}}
\toprule
Header A & Header B\\
\midrule
\rowcolor{rowB}Row 1 col A & Row 1 col B\\
Row 2 col A & Row 2 col B\\
\bottomrule
\end{tabular}}}\par}
```

Rules:
- `\makebox[0.98\linewidth][c]` — constrains rowcolor fill to table width
- `\setlength{\tabcolsep}{2pt}` — reduces padding so `p{}` columns fit
- `@{}` at both ends of column spec — removes leading/trailing padding
- `p{}` column widths must sum to ≈ 0.95\linewidth or less

For auto-sized columns (`c`, `l`, `r`) the `\setlength` wrapper is optional.

### Images inside an ebox

Place images in an `img/` subfolder next to the cheatsheet folder, not inside `src/`. The `\graphicspath{{../img/}}` in the preamble handles the path so you just use the filename:

```latex
\begin{ebox}
{\centering\includegraphics[width=0.90\linewidth,keepaspectratio]{coaxial_fig.png}\par}
\textit{Caption or annotation here}
\end{ebox}
```

For a full-width diagram: `width=\linewidth`. For a smaller inset: `width=0.55\linewidth`.

### Spacing knobs — tight vs. readable

```latex
% Ultra-tight (page 2, dense content)
\fontsize{6}{7}\selectfont
\setlength{\abovedisplayskip}{0pt}\setlength{\belowdisplayskip}{0.5pt}
% innertopmargin=0.5pt, innerbottommargin=0.5pt in \newmdenv

% Readable (page 1, more breathing room)
\fontsize{6}{7.5}\selectfont
\setlength{\abovedisplayskip}{1pt}\setlength{\belowdisplayskip}{1.5pt}
% innertopmargin=2pt, innerbottommargin=2pt in \newmdenv
```

You can switch mid-document with `\setlength` after `\newpage`.

### microtype fix for sub-6pt fonts

If you get `pdfTeX error: auto expansion is only possible with scalable fonts`, add:

```latex
\usepackage[expansion=false]{microtype}
```

This hits when `\eq{}{}` renders labels at 5.5pt and bitmap fonts kick in. `expansion=false` fixes it with no visible quality loss.

### Page length — if it overflows, fix in this order

1. Remove or shorten a section
2. Reduce `\vspace{}` between boxes (try `0.3pt`)
3. Tighten ebox margins (`innertopmargin`, `innerbottommargin`)
4. Condense multi-line items to one line

Do not just shrink the font below 6pt — it becomes unreadable under exam conditions.

### Full workflow: download → edit → compile → push via API

Because Claude works in `/tmp` and that gets cleared between sessions, always fetch the file fresh from GitHub at the start of a session, then push back via the API (the file is too large to pass as a CLI arg).

```bash
# 1. Download .tex from GitHub
gh api repos/BaoPSU/ECE332-EMAG-II-Portland-State-University/contents/Notes/Cheatsheets/Midterm1/src/ECE332_Exam1_cheatsheet.tex \
  --jq '.content' | base64 -d > /tmp/ECE332_Exam1_cheatsheet.tex

# 2. Edit the file (Claude uses the Edit tool on /tmp/ECE332_Exam1_cheatsheet.tex)

# 3. Compile — MUST run from /tmp so \graphicspath resolves correctly
#    (or copy img/ folder to /tmp first if images are needed)
cd /tmp && pdflatex -interaction=nonstopmode ECE332_Exam1_cheatsheet.tex
# Check: "Output written on ... (2 pages)" — must be exactly 2 pages (front+back)

# 4. Push PDF via API (too large for CLI arg, must base64-encode)
SHA=$(gh api repos/BaoPSU/ECE332-EMAG-II-Portland-State-University/contents/Notes/Cheatsheets/Midterm1/ECE332_Exam1_cheatsheet.pdf --jq '.sha')
base64 -w 0 /tmp/ECE332_Exam1_cheatsheet.pdf > /tmp/pdf_b64.txt
python3 -c "
import json
with open('/tmp/pdf_b64.txt') as f: content = f.read().strip()
json.dump({'message': 'update Exam1 cheat sheet', 'content': content, 'sha': '$SHA'}, open('/tmp/payload.json','w'))
"
gh api --method PUT repos/BaoPSU/ECE332-EMAG-II-Portland-State-University/contents/Notes/Cheatsheets/Midterm1/ECE332_Exam1_cheatsheet.pdf \
  --input /tmp/payload.json --jq '.commit.sha'

# 5. Push .tex the same way
SHA2=$(gh api repos/BaoPSU/ECE332-EMAG-II-Portland-State-University/contents/Notes/Cheatsheets/Midterm1/src/ECE332_Exam1_cheatsheet.tex --jq '.sha')
python3 -c "
import json, base64
content = base64.b64encode(open('/tmp/ECE332_Exam1_cheatsheet.tex','rb').read()).decode()
json.dump({'message': 'update Exam1 cheat sheet', 'content': content, 'sha': '$SHA2'}, open('/tmp/payload_tex.json','w'))
"
gh api --method PUT repos/BaoPSU/ECE332-EMAG-II-Portland-State-University/contents/Notes/Cheatsheets/Midterm1/src/ECE332_Exam1_cheatsheet.tex \
  --input /tmp/payload_tex.json --jq '.commit.sha'
```

Put the full workflow in CLAUDE.md so Claude always follows it without being told.

### Prompting Claude to edit a cheat sheet

Always name the exact file and state the constraint:

```
Download ECE332_Exam1_cheatsheet.tex from GitHub, add a section on skin depth 
after the lossy media section. Use \shead{coralbg}{coral}{SKIN DEPTH}. 
Two equations: general form and good-conductor approximation. Keep it under 
8 lines of LaTeX. Must still compile to 2 pages.
```

Never say "update my cheat sheet" without a path or Claude will guess — and with multiple `.tex` files it will guess wrong.

---

## Recommended File Organization

This structure is taken directly from `BaoPSU/ECE332-EMAG-II-Portland-State-University` — use it as your template for any STEM course repo.

### Actual ECE332 repo layout

```
ECE332-EMAG-II-Portland-State-University/
│
├── Notes/
│   ├── CLAUDE.md                        ← Claude's instructions (lives here, not root)
│   │
│   ├── Cheatsheets/
│   │   ├── Midterm1/
│   │   │   ├── ECE332_Exam1_cheatsheet.pdf   ← compiled PDF (viewable on GitHub)
│   │   │   ├── ECE332_HW1_cheatsheet.pdf
│   │   │   ├── ECE332_HW2_cheatsheet.pdf
│   │   │   ├── img/                          ← images referenced by .tex files
│   │   │   │   ├── coaxial_fig.png
│   │   │   │   ├── pol_ellipse.jpg
│   │   │   │   ├── UnitCircle.jpg
│   │   │   │   └── conductor_3d.png
│   │   │   └── src/                          ← .tex source files live here
│   │   │       ├── ECE332_Exam1_cheatsheet.tex
│   │   │       ├── ECE332_HW1_cheatsheet.tex
│   │   │       └── ECE332_HW2_cheatsheet.tex
│   │   └── Final/
│   │       └── src/                          ← drop final cheat sheet .tex here
│   │
│   ├── lecture01.pdf                    ← original lecture slide PDFs
│   ├── lecture02.pdf
│   └── ...
│
├── Homework/
│   ├── HW1/
│   │   ├── HW1_Generate.tex             ← HW writeup source
│   │   ├── HW1_Generate.pdf
│   │   ├── hw1.pdf                      ← blank assignment
│   │   ├── hw1solns.pdf                 ← solutions
│   │   └── cs_crops/                    ← cropped screenshots from the cheat sheet
│   │       ├── cs_bfields.png           ← used as image references in answers
│   │       ├── cs_emf.png
│   │       └── ...
│   └── HW2/
│       ├── HW2.pdf
│       ├── HW2_solutions.pdf
│       ├── HW2_P3a.png                  ← plot generated by Python script
│       └── HW2_P3a.py
│
├── Exams/
│   ├── ECE332_Exam1_Fall2025.pdf        ← blank exam
│   ├── ECE332_Exam1_Fall2025_AnswerKey.pdf
│   └── src/
│       ├── ECE332_Exam1_Fall2025.tex
│       └── ECE332_Exam1_Fall2025_AnswerKey.tex
│
├── Labs/
│   ├── ECE332 Lab Report Template.docx
│   ├── Lab 1/ECE332_Lab1_Signal_Integrity.pdf
│   ├── Lab 2/ECE332_Lab2_WirelessPower.pdf
│   └── Lab 3/ECE332_Lab3_Waveguides_Official.pdf
│
├── README.md
└── syllabus.pdf
```

### Key structural rules

**PDFs go in the folder, source goes in `src/`**  
Compiled PDFs are viewable directly on GitHub. Keeping `.tex` in `src/` prevents clutter and makes clear which files Claude should edit.

**Images go in `img/` next to `src/`**  
The preamble uses `\graphicspath{{../img/}}` so the `.tex` in `src/` finds images one level up. Always compile from `src/` — compiling from `/tmp` without copying `img/` will silently drop all figures.

**CLAUDE.md goes in `Notes/`**  
This scopes Claude's instructions to the notes/cheatsheets work. Claude reads the CLAUDE.md closest to where it's working. Putting it at the repo root makes it apply to homework and labs too, which you usually don't want.

**`cs_crops/` inside each HW folder**  
When an HW answer references a formula from the cheat sheet, crop that section as a PNG and save it here. Embed it in the HW `.tex` as a figure instead of re-typesetting the formula — faster and consistent with the cheat sheet.

### CLAUDE.md for the Notes folder (real ECE332 version)

```markdown
# ECE332 Cheat Sheet — Claude Setup Guide

## Folder structure
Notes/Cheatsheets/
├── Midterm1/
│   ├── *.pdf          ← compiled output
│   ├── img/           ← images for the cheat sheet
│   └── src/*.tex      ← source files (edit these)
└── Final/
    └── src/

Always compile from the src/ subfolder — images use \graphicspath{{../../}}
which resolves to Notes/. Compiling from /tmp will silently drop figures.

## Workflow
1. Download the .tex from GitHub via gh api ... | base64 -d > /tmp/<file>.tex
2. Edit /tmp/<file>.tex
3. Compile: cd /tmp && pdflatex -interaction=nonstopmode <file>.tex
   Check: output must stay on exactly N pages
4. Push PDF and .tex back to GitHub via gh api --method PUT (base64-encoded)

## Layout
- \documentclass[6pt,letterpaper]{article}, margins 0.25in
- 3-column multicols body + 5-box minipage strip at bottom
- \columnbreak after col 1 and col 2

## Color scheme (bg, text)
purple=core laws | teal=field relations | coral=lossy/warnings
amber=rotating/circular | green=power/Poynting | blue=comparisons
gray=quick-ref | pink=phasors

## Table pattern (prevents rowcolor bleed)
{\centering\makebox[0.98\linewidth][c]{{\setlength{\tabcolsep}{2pt}
\begin{tabular}{@{}...@{}}...\end{tabular}}}\par}

## ebox environment
\newmdenv[linewidth=0.4pt,innerleftmargin=3pt,innerrightmargin=3pt,
  innertopmargin=2pt,innerbottommargin=2pt,skipabove=1pt,skipbelow=1pt]{ebox}
Inner \linewidth inside ebox ≈ 182.7 pt (col width 188.7 pt minus 6 pt margins)

## Page length control — if it overflows, fix in this order
1. Remove or shorten a section
2. Reduce \vspace{} between boxes (try 0.3pt)
3. Tighten ebox margins (innertopmargin/innerbottommargin)
4. Condense multi-line items to one line

## Known harmless warnings
Two Overfull \hbox warnings at lines ~223 and ~241 (lossy media section) — expected, ignore.

## Files Claude is allowed to edit
- Notes/Cheatsheets/Midterm1/src/ECE332_Exam1_cheatsheet.tex
- Notes/Cheatsheets/Final/src/*.tex (when created)

## Files Claude must NOT touch
- Homework/** — ask before changing any HW file
- Exams/**    — never edit exam source without explicit instruction
- Any file not in Notes/Cheatsheets/
```

### Naming conventions

| Type | Pattern | Example |
|------|---------|---------|
| Exam cheat sheet | `CourseCode_ExamN_cheatsheet.tex` | `ECE332_Exam1_cheatsheet.tex` |
| HW cheat sheet | `CourseCode_HWN_cheatsheet.tex` | `ECE332_HW2_cheatsheet.tex` |
| Exam paper | `CourseCode_ExamN_Term.tex` | `ECE332_Exam1_Fall2025.tex` |
| Answer key | `..._AnswerKey.tex` | `ECE332_Exam1_Fall2025_AnswerKey.tex` |
| HW folder | `HWN/` | `HW3/` |
| HW writeup | `HWN_Generate.tex` | `HW1_Generate.tex` |
| Lecture slide | `lectureNN.pdf` | `lecture05.pdf` |
| CS crop | `cs_<topic>.png` | `cs_bfields.png` |

Zero ambiguity. Claude won't ask which file you mean and won't touch the wrong one.

### Top-level layout across courses

```
~/school/
├── ECE332-EMAG-II/          ← one git repo per course
├── ECE410-ML-Hardware/
├── ECE424-Prof-Practice/
└── shared/
    └── latex-templates/
        ├── cheatsheet_3col.tex   ← the ECE332 preamble stripped of content
        ├── homework.tex
        └── lab_report.tex
```

Each course is its own repo pushed to GitHub. Never mix courses in one repo — it makes the CLAUDE.md too generic and Claude ends up with permission to edit everything.

### .gitignore for LaTeX course repos

```gitignore
*.aux
*.log
*.out
*.synctex.gz
*.fls
*.fdb_latexmk
*.toc
# Keep PDFs committed — viewable on GitHub without local LaTeX install
# *.pdf   ← intentionally excluded from gitignore
```

---

## Tone & Explanation Style in CLAUDE.md

You can tell Claude exactly how to talk to you — and it will actually follow it. This is genuinely one of the most underrated things you can put in a CLAUDE.md.

### The line that actually works

```markdown
## How to explain things
Please explain questions like I'm a fucking idiot. Short sentences,
real analogies, zero textbook language. It really helps ngl.
```

That's it. One paragraph. Claude will follow it every single session without you having to ask.

**Before** (default Claude, unprompted):

> "The Poynting vector represents the directional energy flux density of an electromagnetic field, formally defined as **S** = **E** × **H**, where the cross product yields a vector orthogonal to both field components indicating the direction of power flow per unit area..."

**After** (with the line above in CLAUDE.md):

> "It's basically a GPS arrow for where the energy is going and how much of it. Cross E and H with your right hand — your thumb points where the power flows. Units are watts per square meter, so it's power through a window."

Same Claude. One line in a file. Completely different experience.

### Why it works so well

Claude's default voice is trained to sound like a textbook because that's "correct." But correct is not the same as useful when you're trying to actually understand something at 11pm before an exam. Telling it explicitly to drop the formality gives it permission to just talk to you like a person.

The vulgar framing works especially well because it signals clearly: do not hedge, do not define terms I didn't ask about, do not start with the general case. Just answer the question.

### More variations that work

```markdown
## Explanation style options (pick what fits you)

# Option 1 — the classic
Explain things like I'm an idiot. Seriously. Short, blunt, real examples.
Skip the formal definitions unless I ask.

# Option 2 — for problem-solving mode
When I give you a problem, just walk me through the steps. Numbered.
No theory unless I ask "why". I just need to get through this HW.

# Option 3 — for circuits people learning EM
I know circuits cold. When explaining EM concepts, map them to 
voltage/current/impedance/RC first. Then do the field theory version.

# Option 4 — answer first, explain second
Lead with the answer or the punchline. Explanation after.
I'll ask "why" if I want it. Don't bury the answer in a paragraph.
```

### Stack them so you can switch on the fly

```markdown
## Explanations
- Default: short, blunt, idiot-proof. Real examples before formulas.
- If I say "formal": textbook definition, full precision
- If I say "steps": numbered steps only, no explanation at all
- If I say "why": explain the physical intuition behind the last thing you said
```

Now you can just say "why" or "formal" mid-session and Claude knows exactly what gear to shift into. No re-explaining your preferences every conversation.

### For coursework CLAUDE.md

```markdown
## How to talk to me
Explain things like I'm completely new to this, even if I'm not.
Intuition first, math second. If I ask "what does X mean physically" —
give me the gut-check version, not the Wikipedia definition.
I'm an EE student so circuits analogies hit different.
```

This + detailed lecture summaries in the same CLAUDE.md = Claude that actually helps you understand the material instead of just restating it fancier.

---

## Using Claude as a TA

The worst way to use Claude for coursework is "just give me the answer." You learn nothing, and when the exam comes you're cooked. The best way is to treat it like a TA that has infinite patience, has read every textbook and lecture, and will never make you feel dumb for asking the same question five different ways.

Here's the full workflow that actually works.

---

### Step 1 — Build a HW cheat sheet before you start the homework

Before touching a single problem, have Claude pull every equation and concept the homework will need from your lectures and textbook. This becomes `ECE332_HW1_cheatsheet.tex` — a reference sheet scoped to exactly that assignment.

```
I'm starting HW1 for ECE332. The topics are Faraday's Law, EMF, 
and B fields from wires and toroids. 

Look at lecture01.pdf and lecture02.pdf and pull out every equation, 
definition, and formula that could show up on this homework. 
Build me a LaTeX cheat sheet using the same format as ECE332_Exam1_cheatsheet.tex.
One section per topic. Include a "when to use this" note next to each equation.
```

Claude reads your actual lecture slides, not generic internet knowledge. Every equation it pulls is one your professor actually taught — which means it's one that will actually be on the exam.

---

### Step 2 — Work through problems with it, don't just get answers

When you're stuck on a problem, don't say "solve this." Say "guide me through it step by step."

```
HW1 Problem 3: A rectangular loop sits next to an infinite wire carrying 
I(t) = 5cos(1000t) A. Find the induced EMF.

Don't solve it. Walk me through it one step at a time — give me the first 
thing I need to figure out, wait for my answer, then give me the next step.
```

Claude will stop after each step and wait for you to respond. It might ask: is the loop moving or stationary? Once you answer, it gives you the next question. This is exactly how a good TA runs office hours — they don't solve it for you, they ask you questions until you solve it yourself.

**Structure to ask for explicitly:**

```
Guide me through this problem step by step. After each step, 
wait for me to answer before continuing. If I'm wrong, 
tell me what I got wrong and ask me to try again — don't just 
give me the right answer.
```

**If you're completely lost:**

```
I have no idea where to even start. What's the very first 
question I need to ask myself when I see a problem like this?
```

This forces the back-and-forth that actually builds understanding. A real TA doesn't just write the answer on the board — they ask "okay what do you know so far?"

---

### Generate similar practice problems from your homework

Once you finish (or understand) a homework problem, use it as a template to generate more like it. This is how you actually get good at a problem type — not by doing it once, but by doing five variations until the pattern is automatic.

```
Here's HW1 Problem 3: [paste problem]

Generate 3 similar problems with different numbers and slightly different 
setups — change the loop geometry, the current waveform, the wire configuration.
Don't solve them yet, just give me the problems.
```

Then work through each one step by step. When you're done:

```
Now show me the solutions and tell me where my reasoning was wrong.
```

**Variations to ask for:**

```
Make a harder version of this problem — add a second wire, or make the loop moving.
```

```
Make an easier version first so I can see the pattern, then give me the original.
```

```
Give me a version that tests the same concept but looks completely different 
on the surface — I want to make sure I actually understand it and 
didn't just memorize this specific setup.
```

That last one is the most important. Exams never look exactly like the homework. If you can only solve the problem when it looks familiar, you don't actually know it yet.

**Before an exam, do this:**

```
I have Exam 1 coming up covering HW1 and HW2. 
Generate a 5-problem practice exam using my HW problems as templates.
Mix the topics. Make the problems exam-length, not homework-length.
Don't tell me the answers yet.
```

Work through it. Then:

```
Grade my work. For each problem I got wrong, don't just show me the answer —
tell me the specific step where my reasoning broke down.
```

---

### Step 3 — When something doesn't click, ask for it a different way

Claude will never get impatient. Every time something doesn't make sense, ask for a new angle:

```
I still don't get why the sign of the EMF matters. 
Can you explain it a different way?
```

```
Can you give me a physical intuition for this instead of the math?
```

```
Can you walk me through a super simple example, like the simplest 
possible version of this problem, before we do the real one?
```

```
Why does adding more turns to the coil increase the EMF? 
Like physically, what's actually happening?
```

None of these are dumb questions. These are exactly the questions you should be asking. The difference between a student who gets it and one who doesn't is usually just that the first one asked "why" three more times.

---

### The quality test for a good cheat sheet

**You know the cheat sheet is done when you can complete the entire homework using only the cheat sheet — no textbook, no lecture slides, no Google.**

If you hit a problem and have to go look something up, that thing belongs on the cheat sheet. Add it, then close the tab. By the time you've finished the homework, the cheat sheet contains literally everything you needed. That's exactly what you want going into the exam.

```
I finished HW2. I had to look up [the skin depth formula for good conductors] 
mid-homework — it wasn't on my cheat sheet. Add it to ECE332_HW2_cheatsheet.tex 
with a "when to use" note.
```

Do this every time you look something up. The cheat sheet self-corrects as you work.

---

### Step 4 — Update the HW cheat sheet as you go

Every time you figure something out — a trick, a pattern, a "oh that's what that means" moment — add it to the HW cheat sheet:

```
Add a note to the EMF section: "if the loop is stationary and B is changing, 
use transformer EMF. If B is static and the loop is moving, use motional EMF. 
If both — use the full flux form, it always works."
```

These notes are yours. They're written in your words, based on your confusion, and they'll make more sense to you on the exam than anything in the textbook.

---

### Step 5 — Combine HW cheat sheets into the exam cheat sheet

Before each midterm or final, merge all the individual HW cheat sheets into one. Claude handles this automatically:

```
I have ECE332_HW1_cheatsheet.tex and ECE332_HW2_cheatsheet.tex.
Combine them into ECE332_Exam1_cheatsheet.tex using the same format.
Midterm 1 covers lectures 1-10 and both homeworks.

Rules:
- Remove duplicate equations (keep the one with the better "when to use" note)
- If HW1 and HW2 have overlapping topics, merge them into one section
- Add a bottom reference strip with key variables and constants
- Must compile to exactly 2 pages (front and back of one sheet)
```

The result is a cheat sheet that was built from your actual confusion — every equation in it is one that came up on real homework problems you actually struggled with.

---

### CLAUDE.md additions for the TA workflow

Add this to your course CLAUDE.md:

```markdown
## TA mode
When I'm working on homework, don't just solve problems for me.
Ask me questions to help me figure out the right approach first.
If I'm completely lost, walk me through it step by step — but ask me
to confirm each step before moving to the next one.

## HW cheat sheet workflow
For each homework assignment:
1. Before I start: pull equations from the relevant lecture PDFs into a new HW cheat sheet
2. While I work: update the cheat sheet with tricks and notes I figure out
3. After I finish: clean up the cheat sheet so it's exam-ready

## Exam cheat sheet workflow
Before each exam: merge all HW cheat sheets for the covered topics.
Remove duplicates, merge overlapping sections, add reference strip.
Must fit on one physical sheet (2 PDF pages).

## How to explain things
Explain things like I'm an idiot. If I don't get it, try a different angle.
Never make me feel bad for asking the same question multiple times.
Physical intuition before math. Simple example before the general case.
```

---

### Questions that are always worth asking

Real students ask these all the time. Don't be afraid of them:

- *"What's the difference between X and Y and when do I use each one?"*
- *"Can you show me the simplest possible example of this?"*
- *"Why does this formula look like this? Where does it come from?"*
- *"I got [wrong answer] — can you help me figure out where I went wrong without just solving it?"*
- *"What's the most common mistake people make on this type of problem?"*
- *"If I only have 30 seconds on the exam, what's the one thing I need to remember about this?"*
- *"Can you quiz me on this topic?"*

That last one is underused. Before an exam:

```
Quiz me on plane wave propagation. Ask me one question at a time, 
wait for my answer, then tell me if I'm right and what I missed.
Don't give me the next question until I've answered the current one.
```

This is literally how you study. You now have a TA available 24/7 who will do this forever without getting tired.

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
