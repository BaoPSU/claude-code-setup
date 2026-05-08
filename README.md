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
7. [Saving Tokens with /compact](#compact)
8. [Working with Repos — Don't Just Paste the URL](#repos-warning)
9. [Lecture Summaries in CLAUDE.md](#lecture-summaries)
10. [LaTeX Cheat Sheets](#latex)
11. [Recommended File Organization](#file-org)
12. [Tone & Explanation Style in CLAUDE.md](#tone)
13. [Tips & Patterns](#tips)

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

You can tell Claude exactly how to explain things to you — and it will actually follow it. The most useful thing you can add to any CLAUDE.md is a plain-language instruction about how you want concepts broken down.

### The line that actually works

```markdown
## How to explain things
When I ask you to explain a concept, explain it like I'm a complete idiot.
Short sentences. Real analogies. No textbook language.
```

This one line changes how Claude answers every question in that project. Instead of:

> "The Poynting vector represents the directional energy flux density of an electromagnetic field, defined as the cross product of the electric and magnetic field vectors..."

You get:

> "It's basically which direction the wave is carrying energy and how much per square meter. Point your fingers along E, curl them toward H — your thumb points where the power goes."

### More tone options to put in CLAUDE.md

Pick the one that matches how your brain works:

```markdown
## Explanation style: ELI5
Explain everything like I'm new to this topic. Start with the physical intuition
before any math. Use one concrete example before showing the formula.

## Explanation style: step-by-step only
Never explain the theory unless I ask. When I give you a problem, 
just walk me through the steps to solve it. Number each step.

## Explanation style: compare to something I know
I'm a circuits person. When explaining EM concepts, relate them back 
to voltage, current, resistance, and impedance whenever possible.

## Explanation style: bottom line first
Give me the answer or the key takeaway in the first sentence.
Explanation after. I'll ask if I want more detail.
```

### Stack them for different situations

```markdown
## Explanations
- Default: plain english, short sentences, intuition before math
- If I say "formal": give the textbook definition
- If I say "just the steps": numbered steps only, skip all explanation
- If I say "ELI5": pretend I've never seen this topic before
```

Now you can switch modes mid-session just by saying "formal" or "ELI5" without re-explaining your preferences every time.

### Why this matters more than you think

Claude defaults to a textbook voice — complete, precise, and utterly useless for actually learning something fast. A single line in CLAUDE.md flips that for every session in that project. You write it once and never have to say "can you explain that more simply" again.

If you're using Claude for coursework, pair it with your lecture summaries:

```markdown
## Course style note
I'm an EE student. Strong on circuits and math, weaker on field theory intuition.
When explaining EM: circuits analogies first, field theory second.
When I ask "what does X mean physically" — give me the gut-check version, not the definition.
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
