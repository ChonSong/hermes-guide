# Hermes Skills — Complete Guide

> **Version:** 2.x | **Location:** `~/.hermes/skills/` (host) / `/opt/data/skills/` (container)
>
> Skills are Hermes Agent's self-improvement system. When Hermes encounters a task, learns a workflow, or gets corrected, it can save that knowledge as a skill document that loads automatically in future sessions. Over time, the agent gets better at your specific tasks and environment. This guide covers everything about finding, using, managing, and authoring skills.

---

## Table of Contents

1. [What Are Skills?](#1-what-are-skills)
2. [How Skills Work](#2-how-skills-work)
3. [Finding the Right Skill](#3-finding-the-right-skill)
4. [Installing Skills](#4-installing-skills)
5. [Loading Skills into a Session](#5-loading-skills-into-a-session)
6. [Key Skills Catalog](#6-key-skills-catalog)
7. [Skill Categories](#7-skill-categories)
8. [Authoring New Skills](#8-authoring-new-skills)
9. [Skill Authoring Format](#9-skill-authoring-format)
10. [Best Practices](#10-best-practices)
11. [Skill Management](#11-skill-management)
12. [Skill Limitations & Gotchas](#12-skill-limitations--gotchas)

---

## 1. What Are Skills?

Skills are Markdown documents (`.md` files) that encode reusable procedures, workflows, and knowledge. They're the difference between Hermes as a general-purpose assistant and Hermes customized for *your* specific setup.

**Think of them as:**

| Analogy | What it means |
|---------|---------------|
| **Apps for the agent** | Skills are installable capabilities — like apps on a phone |
| **Experienced memory** | Rather than relearning how to do something, Hermes loads the skill and follows the proven steps |
| **Trainable procedures** | You can teach Hermes new workflows by writing or generating skill documents |
| **Context shortcuts** | Instead of explaining the full context every time, just load the relevant skill |

**Skills are NOT:**
- Code that runs on its own — Hermes reads them and follows the instructions
- Prompts — they're structured procedure documents with frontmatter metadata
- Magic — a bad skill produces bad results

### 1.1 What a Skill Looks Like

```markdown
---
name: github-pr-workflow
description: GitHub PR lifecycle: branch, commit, open, CI, merge.
category: github
---

# GitHub PR Workflow

## When to Use
Use when you need to create a PR, check CI status, or merge code.

## Prerequisites
- `gh` CLI authenticated (`gh auth login`)
- Repo cloned locally
- Working directory is the repo root

## Step 1 — Branch
Create a new branch:
```bash
git checkout -b feature/my-feature
```

## Step 2 — Commit
Stage and commit:
```bash
git add .
gh commit -m "feat: add my feature"
```

## Step 3 — Push
Push to origin:
```bash
git push -u origin HEAD
```

## Step 4 — Create PR
```bash
gh pr create --title "feat: my feature" --body "Description"
```

## Step 5 — Check CI
```bash
gh pr checks
```

## Step 6 — Merge
After CI passes:
```bash
gh pr merge --squash
```
```

### 1.2 Skill vs SOUL.md vs MEMORY.md

| File | Purpose | When it loads |
|------|---------|---------------|
| **SOUL.md** | Agent personality and core principles | Every session, always |
| **MEMORY.md** | Long-term user facts, preferences, lessons | Every session, always |
| **Skill** | Specific task procedure | On demand or matching trigger |
| **Daily memory** | Session logs and what happened today | Every session, always |

Skills are situational — loaded when relevant. MEMORY.md and SOUL.md are always loaded.

---

## 2. How Skills Work

### 2.1 Skill Loading

When you type a message, Hermes checks all installed skills against your message using a lightweight scorer. Matching skills are loaded into the system prompt for that session. The more specific the match, the better the skill applies.

**Loading methods:**

| Method | How | When it applies |
|--------|-----|----------------|
| **Auto-match** | Skill triggers (keywords in message) match against your input | Every turn, automatically |
| **Slash command** | `/skill skill-name` during chat | Current session only |
| **Startup flag** | `hermes -s skill1,skill2` | Session lifetime |
| **cronjob tool** | `skills: ["skill-name"]` in cron job config | Job execution only |

### 2.2 Auto-Matching

Skills have a `description` field and optional `tags`. The skill selector scores each skill against the current message. High-scoring skills load silently. If >100MB of skills would load, Hermes asks before loading.

The skill selector runs every turn and is lightweight — it's based on cached metadata from skill-selector-prep (a weekly cron job that syncs metadata for all skills).

### 2.3 Skill Priority and Conflict

If two skills cover the same ground, Hermes loads both but the order matters. Skills loaded by keyword match are prepended to the system prompt before skills loaded by slash command. If procedures conflict, the last-loaded wins.

### 2.4 Skill Expiration

Skills don't expire. Once installed, they persist until you uninstall them. However, skills that haven't been used in a long time may have outdated instructions — check periodically.

---

## 3. Finding the Right Skill

### 3.1 Search from CLI

```bash
# Search by keyword
docker exec hermes /opt/hermes/.venv/bin/hermes skills search "github"

# Search by category
docker exec hermes /opt/hermes/.venv/bin/hermes skills search "devops"

# Browse all available skills
docker exec hermes /opt/hermes/.venv/bin/hermes skills browse
```

### 3.2 Search in WebUI

Open `/skills` in Hermes WebUI. Browse by category, search by name/description/tags/triggers, filter by risk level.

### 3.3 Ask Hermes

The simplest method: "What skill should I use for [task]?" Hermes will search its installed skills and recommend one.

### 3.4 Skill Categories (WebUI)

| Category | What it covers |
|----------|---------------|
| All | Everything |
| Web & Frontend | CSS, HTML, JS, React, Svelte |
| Coding Agents | Coder, reviewer, refactor, debug agents |
| Git & GitHub | PRs, issues, repos, git workflows |
| DevOps & Cloud | Docker, CI/CD, deployment, infra |
| Browser & Automation | Playwright, browser agents, scraping |
| Image & Video | Image generation, video, ASCII art |
| Search & Research | arXiv, web search, blogwatcher |
| AI & LLMs | Model serving, fine-tuning, evaluation |
| Productivity | Docs, calendar, email, Notion, Airtable |
| Marketing & Sales | Social media, copywriting |
| Communication | Discord, Telegram, Slack |
| Data & Analytics | Jupyter, data processing, visualization |
| Finance & Crypto | (specialized financial tools) |

---

## 4. Installing Skills

### 4.1 From the CLI

```bash
# List installed skills
docker exec hermes /opt/hermes/.venv/bin/hermes skills list

# Install a skill from the catalog
docker exec hermes /opt/hermes/.venv/bin/hermes skills install github-auth

# Inspect a skill before installing
docker exec hermes /opt/hermes/.venv/bin/hermes skills inspect skill-name

# Check for updates
docker exec hermes /opt/hermes/.venv/bin/hermes skills check

# Update outdated skills
docker exec hermes /opt/hermes/.venv/bin/hermes skills update

# Uninstall
docker exec hermes /opt/hermes/.venv/bin/hermes skills uninstall skill-name
```

### 4.2 From the WebUI

1. Go to `/skills`
2. Browse the **Marketplace** tab
3. Search or filter by category
4. Click **Install** on any skill
5. The skill downloads to `~/.hermes/skills/`

### 4.3 From Git

```bash
# Add a GitHub repo as a skill source
docker exec hermes /opt/hermes/.venv/bin/hermes skills tap add https://github.com/user/skill-repo

# Then install from that source
docker exec hermes /opt/hermes/.venv/bin/hermes skills install skill-from-repo
```

### 4.4 Manually (Write Your Own)

Create a `.md` file in `~/.hermes/skills/your-skill/SKILL.md`. See the [Authoring section](#9-skill-authoring-format) for the format.

---

## 5. Loading Skills into a Session

### 5.1 Slash Command (Current Session)

```
/skill github-auth
```

The skill loads immediately and its instructions are available for the rest of the session. `/reset` unloads everything and starts fresh.

### 5.2 Startup Flag (Session Lifetime)

```bash
# Load multiple skills at startup
docker exec -it hermes /opt/hermes/.venv/bin/hermes --tui -s github-auth,coder,researcher

# In Docker Compose, set in environment or command
```

### 5.3 cronjob Tool (Job Execution)

```bash
# Create a cron job that loads specific skills
cronjob(
  action='create',
  name='Morning research',
  schedule='0 8 * * MON-FRI',
  prompt='Summarize the top 5 AI news stories from yesterday',
  skills=['researcher', 'web'],
  deliver='origin'
)
```

### 5.4 Auto-Match (Automatic)

Skills with matching triggers load automatically. For example, a skill with `triggers: ["github pr", "pull request"]` loads automatically when you mention "create a PR" or "check the CI status".

---

## 6. Key Skills Catalog

These are the most important skills for this Hermes setup:

### 6.1 Core System Skills

| Skill | What it does |
|-------|-------------|
| **`hermes-agent`** | Complete guide to Hermes Agent — CLI, config, delegation, gateway, tools. **Load this first if anything isn't working.** |
| **`hermes-computer`** | Building and extending Hermes Web Computer — Go + Svelte 5 tiling desktop |
| **`hermes-docker-workflow`** | Docker build, run, and troubleshooting for Hermes containers |
| **`hermes-agent-skill-authoring`** | How to write good skill documents — format, frontmatter, validator |
| **`skill-selector`** | Every-turn skill scorer — auto-loads matching skills. Runs silently every turn. |

### 6.2 Git & GitHub

| Skill | When to use |
|-------|-------------|
| **`github-auth`** | Authenticate GitHub — tokens, SSH keys, `gh` CLI login |
| **`github-pr-workflow`** | Create PRs, check CI, merge — the full PR lifecycle |
| **`github-code-review`** | Review PR diffs with inline comments |
| **`github-repo-management`** | Clone, fork, manage remotes and releases |
| **`github-issues`** | Create, triage, label, assign GitHub issues |

### 6.3 Coding & Development

| Skill | When to use |
|-------|-------------|
| **`coder`** | Code generation — write new code, features, scripts |
| **`reviewer`** | Quality assurance — code review, testing, best practices |
| **`codi`** | Code ingestion and refactoring — understand large codebases |
| **`subagent-driven-development`** | Execute plans via subagents with 2-stage review |
| **`autonomous-development`** | Build features end-to-end from spec |
| **`test-driven-development`** | TDD workflow — tests before code |
| **`systematic-debugging`** | 4-phase root cause debugging |
| **`e2e-testing`** | Playwright E2E testing patterns |
| **`svelte-development`** | Svelte 5 runes mode syntax and patterns |
| **`vite-patterns`** | Vite build tool patterns |
| **`node-inspect-debugger`** | Debug Node.js via Chrome DevTools Protocol |
| **`python-debugpy`** | Debug Python via debugpy remote DAP |

### 6.4 Multi-Agent & Delegation

| Skill | When to use |
|-------|-------------|
| **`delegation-patterns`** | How to delegate to subagents — when to use what pattern |
| **`claude-code`** | Delegate to Claude Code CLI for PRs and features |
| **`codex`** | Delegate to OpenAI Codex CLI |
| **`opencode`** | Delegate to OpenCode CLI |
| **`roadmap-engine`** | Long-horizon autonomous planning — runs nightly |
| **`self-improvement-engine`** | Process learnings, rank candidates, author new skills |
| **`autonomous-cron-pipeline`** | Multi-phase autonomous projects via chained cron jobs |
| **`kanban-orchestrator`** | Decomposition playbook for orchestrating Kanban workers |
| **`kanban-worker`** | Best practices for Kanban worker agents |

### 6.5 Research & Web

| Skill | When to use |
|-------|-------------|
| **`researcher`** | Web search, data extraction, research synthesis |
| **`arxiv`** | Search arXiv papers by keyword, author, category |
| **`blogwatcher`** | Monitor RSS/Atom feeds |
| **`polymarket`** | Query prediction markets |
| **`academic-source-compilation`** | Collect and verify academic sources, APA7 references |
| **`dogfood`** | Exploratory QA of web apps — find bugs and report them |

### 6.6 Automation & Cron

| Skill | When to use |
|-------|-------------|
| **`autonomous-cron-pipeline`** | **The key skill for multi-phase autonomous work.** Chain cron jobs with dependencies. |
| **`cron-script-only`** | Run raw scripts as cron jobs (no LLM) |
| **`webhook-subscriptions`** | Event-driven agent runs via webhooks |
| **`automation-workflows`** | Design automation workflows across tools |

### 6.7 Creative & Media

| Skill | When to use |
|-------|-------------|
| **`excalidraw`** | Hand-drawn Excalidraw diagrams |
| **`ascii-art`** | ASCII art generation |
| **`ascii-video`** | Convert video to colored ASCII |
| **`comfyui`** | Generate images/video with ComfyUI |
| **`humanizer`** | Strip AI-isms from text, add real voice |
| **`p5js`** | Generative art, shaders, interactive sketches |
| **`pixel-art`** | Pixel art with era palettes |
| **`architecture-diagram`** | Dark-themed SVG architecture diagrams |

### 6.8 Productivity

| Skill | When to use |
|-------|-------------|
| **`notion`** | Notion API — pages, databases, markdown |
| **`notion-api`** | Generic Notion CLI |
| **`obsidian`** | Read, search, edit Obsidian vault notes |
| **`gcalcli-calendar`** | Google Calendar via gcalcli |
| **`gmail`** | Gmail API integration |
| **`airtable`** | Airtable REST API — CRUD, filters, upserts |
| **`linear`** | Linear project management via GraphQL |
| **`powerpoint`** | Create and edit PowerPoint decks |
| **`ocr-and-documents`** | Extract text from PDFs and scans |
| **`nano-pdf`** | Edit PDF text via NL prompts |

### 6.9 Morning Briefing

| Skill | When to use |
|-------|-------------|
| **`morning-briefing`** | Daily news + weather + calendar + email summary. Schedule via cron at 8 AM Mon-Fri. |

### 6.10 Deployment & DevOps

| Skill | When to use |
|-------|-------------|
| **`deployment-patterns`** | CI/CD pipelines, Docker, health checks |
| **`docker-patterns`** | Docker Compose patterns, security, networking |
| **`infrastructure-as-code`** | Terraform for Cloudflare, AWS |
| **`git-workflow`** | Branching strategies, conventional commits |
| **`canary-watch`** | Post-deploy monitoring for regressions |
| **`repo-init`** | Initialize new project repos from scratch |

### 6.11 Home & Smart Home

| Skill | When to use |
|-------|-------------|
| **`openhue`** | Control Philips Hue lights and scenes |
| **`sonoscli`** | Control Sonos speakers |

### 6.12 Platform Integration

| Skill | When to use |
|-------|-------------|
| **`discord-bot`** | Post to Discord via REST API |
| **`xurl`** | X/Twitter — post, search, DM, media |
| **`spotify`** | Spotify playback control |

---

## 7. Skill Categories

Skills are organized in `~/.hermes/skills/` as subdirectories by category:

```
~/.hermes/skills/
├── agents/          # Named agent identities (zoul, codi, coach, etc.)
├── autonomous-ai-agents/  # Claude Code, Codex, OpenCode, roadmap engine
├── creative/        # ASCII art, diagrams, image generation
├── data-science/    # Jupyter, data processing
├── devops/         # Docker, git, CI/CD, deployment
├── github/         # GitHub PRs, issues, repos
├── hermes/         # Hermes Agent itself
├── hermes-computer/ # HWC development
├── mcp/           # MCP server configuration
├── media/         # YouTube, Spotify, GIFs
├── mlops/         # Model serving, fine-tuning, evaluation
├── note-taking/   # Obsidian, Notion
├── planning/      # Blueprint, product lens
├── productivity/  # Docs, calendar, email, Airtable
├── red-teaming/   # Security testing
├── research/     # arXiv, blogwatcher, Polymarket
├── smart-home/    # Hue, Sonos
└── social-media/  # X/Twitter, Discord
```

---

## 8. Authoring New Skills

### 8.1 When to Write a Skill

Write a skill when:
- You solved a complex problem and want to remember the procedure
- You discovered a non-obvious workflow that works well
- You want to teach Hermes a task-specific procedure
- A recurring task keeps failing because Hermes doesn't know the right steps
- You want to capture a multi-step workflow that's hard to explain each time

**Don't write a skill for:**
- One-off tasks you'll never do again
- Things that are already in the documentation
- Simple tasks that Hermes handles well without a skill

### 8.2 Skill Authoring Best Practices

1. **Start with the trigger condition** — what message or situation should cause this skill to load?
2. **Be specific about prerequisites** — what must be true before starting?
3. **Number the steps** — procedures should be numbered, not paragraphs
4. **Use real commands** — include actual CLI commands, not descriptions of commands
5. **Add a pitfalls section** — what commonly goes wrong and how to fix it
6. **Include verification** — how do you know the skill worked?
7. **Keep it focused** — one skill, one workflow. Split compound skills.

### 8.3 Loading the Authoring Skill

```bash
# Load the authoring skill first
/skill hermes-agent-skill-authoring

# Or from CLI
docker exec -it hermes /opt/hermes/.venv/bin/hermes --tui -s hermes-agent-skill-authoring
```

This skill has the full format specification, validator, and examples.

### 8.4 Testing a Skill Before Publishing

```bash
# Install from local path (for testing)
docker exec hermes /opt/hermes/.venv/bin/hermes skills install /path/to/skill-dir

# Or load directly without installing
docker exec hermes /opt/hermes/.venv/bin/hermes -s /path/to/skill-name

# In WebUI: install from local path or paste the SKILL.md content
```

### 8.5 Publishing Skills

```bash
# Publish to the registry
docker exec hermes /opt/hermes/.venv/bin/hermes skills publish /path/to/skill-dir
```

The skill goes to the Hermes skill registry and becomes available for others to install.

---

## 9. Skill Authoring Format

### 9.1 Required Frontmatter

```markdown
---
name: skill-name
description: One-line description of what this skill does.
category: category-slug    # e.g., github, devops, creative
---
```

### 9.2 Optional Frontmatter

```markdown
---
name: skill-name
description: ...
category: ...
tags: [tag1, tag2, tag3]          # For search and auto-match
triggers: ["keyword1", "keyword2"]  # Phrases that auto-load this skill
related_skills: [skill1, skill2]   # Skills that complement this one
author: YourName
version: 1.0.0
metadata:
  hermes:
    license: MIT
    homepage: https://github.com/you/skill-repo
---
```

### 9.3 Required Sections

#### When to Use
When this skill should be loaded. What tasks it handles. What it should NOT be used for.

#### Prerequisites
What must be true before starting. Tools, API keys, permissions, environment state.

#### Step-by-Step Procedure
Numbered steps. Each step should be:
- Atomic (one logical action)
- Specific (what to do, not just "do it right")
- Executable (real commands, real API calls)

#### Verification
How to confirm the skill worked. Expected output, file changes, API responses.

### 9.4 Recommended Sections

#### Troubleshooting / Pitfalls
Common failure modes and how to fix them. This is the most valuable section for complex skills.

#### Examples
Real examples with expected input and output.

#### Variations
Different approaches for different environments or constraints.

### 9.5 Example: Complete Skill

```markdown
---
name: github-auth
description: GitHub auth setup — HTTPS tokens, SSH keys, gh CLI login.
category: github
tags: [github, auth, ssh, token, cli]
triggers: ["github auth", "gh auth", "setup github", "authenticate github"]
related_skills: [github-pr-workflow, github-code-review]
---

# GitHub Authentication

## When to Use
Use when you need to authenticate with GitHub for any operation — pushing code,
creating PRs, checking CI, or accessing private repos. Also use when auth is
failing with "Authentication failed" or "Permission denied" errors.

## Prerequisites
- `gh` CLI installed (`gh --version`)
- Git installed (`git --version`)
- A GitHub account

## Step 1 — Check Current Auth Status
```bash
gh auth status
```
If already authenticated, this shows your account and token scope.
If not authenticated, it prompts to log in.

## Step 2 — HTTPS Token Auth (Recommended for most cases)
```bash
gh auth login --hostname github.com
# Select: HTTPS (not SSH)
# Browser: Yes (opens browser) or Token (paste manually)
# Token: paste your GitHub PAT (Classic token, with repo scope)
```

**Token format:** `ghp_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx`
Needs `repo` scope for private repos, `workflow` scope for CI.

## Step 3 — SSH Key Auth (For Git operations)
```bash
# Check for existing key
ls ~/.ssh/id_*.pub

# Generate new key if needed
ssh-keygen -t ed25519 -C "your@email.com" -f ~/.ssh/github_ed25519

# Add to GitHub
# Settings → SSH and GPG keys → New SSH key → paste contents of github_ed25519.pub

# Configure git to use SSH
gh auth setup-git
```

## Step 4 — Verify Authentication
```bash
gh auth status
gh api user
```
`gh auth status` should show "Logged in to github.com as <username>"
`gh api user` should return your account JSON.

## Step 5 — Store Credentials (Optional but Recommended)
```bash
# gh stores credentials by default in ~/.config/gh/hosts.yml
# To use git credential helper:
git config --global credential.helper store
git config --global github.token YOUR_TOKEN_HERE
```

## Verification
Run:
```bash
gh repo list
```
Should return your repositories without error. If "Authentication failed",
repeat Step 2 with a valid token.

## Troubleshooting

### "Permission denied (publickey)" on git push
You used SSH auth but your key isn't added to GitHub. Go to GitHub →
Settings → SSH keys → add your public key. Or switch to HTTPS:
```bash
git remote set-url origin https://github.com/username/repo.git
```

### Token works for `gh` but not for git operations
Git needs the credential helper configured:
```bash
git config --global credential.helper store
```

### 2FA-enabled account
Use OAuth device code flow, not password:
```bash
gh auth login --hostname github.com --web-based
```
Or create a Personal Access Token at github.com/settings/tokens.

### Expired token
Go to github.com/settings/tokens, generate a new token, update:
```bash
gh auth refresh --hostname github.com
```
Or re-run `gh auth login`.
```

---

## 10. Best Practices

### 10.1 Skill Naming

- **Lowercase, hyphens** — `github-auth`, `hermes-docker-workflow`, `morning-briefing`
- **Descriptive** — `codi` is the agent, `code-ingestion` might be a skill about ingestion
- **Consistent category** — match the category slug to the directory it lives in

### 10.2 Making Skills Auto-Load

Add `triggers` to the frontmatter. Triggers are phrases that cause the skill to load automatically:

```yaml
triggers: ["github pr", "create a pull request", "check ci status"]
```

When your message matches these phrases, the skill loads automatically before Hermes processes your request.

### 10.3 Skill Dependencies

Don't assume other skills are loaded. If your skill depends on `github-auth`, either:
1. Include the prerequisite steps in your skill, or
2. Document it clearly in the Prerequisites section

### 10.4 Keeping Skills Updated

Skills can become stale. If you find a skill producing wrong results:
1. Check if the external tool/service has changed
2. Update the skill with the correct procedure
3. Test the updated skill
4. Run `hermes skills check` to verify

### 10.5 Skill Testing Checklist

Before saving a skill:
- [ ] Does the description accurately describe what it does?
- [ ] Are the triggers specific enough to match the right input but not so broad they load randomly?
- [ ] Are all commands real (not examples)?
- [ ] Are there error handling steps in the Pitfalls section?
- [ ] Does it have a Verification section?
- [ ] Is it focused (one workflow, not multiple)?
- [ ] Is the frontmatter complete (name, description, category)?

---

## 11. Skill Management

### 11.1 Installed Skills Location

```bash
# Host
~/.hermes/skills/

# Container
/opt/data/skills/
```

Each skill is a directory containing a `SKILL.md` file (and optionally `references/`, `scripts/`, `assets/`).

### 11.2 Skill Sources

Skills come from three sources:
1. **Bundled** — comes with Hermes Agent in `~/.hermes/hermes-agent/skills/`
2. **Hub-installed** — downloaded from the Hermes skill registry
3. **Custom** — written by you or cloned from Git repos

```bash
# Add a GitHub repo as a skill source
docker exec hermes /opt/hermes/.venv/bin/hermes skills tap add https://github.com/user/skill-repo
```

### 11.3 Skill Update

```bash
# Check for outdated skills
docker exec hermes /opt/hermes/.venv/bin/hermes skills check

# Update all outdated skills
docker exec hermes /opt/hermes/.venv/bin/hermes skills update
```

### 11.4 Skill Permissions

Skills must be readable by the Hermes user. If you get permission errors:
```bash
# Diagnose
ls -la ~/.hermes/skills/

# Fix
sudo chown -R $(id -u):$(id -g) ~/.hermes/skills/
```

### 11.5 Skill Threat Scanning (Cron Jobs)

The cron system scans skill content for suspicious strings. Trigger patterns include:
- `authorized_keys` → SSH backdoor detection

If a skill contains SSH tutorial content, either:
1. Rephrase to avoid the trigger string (e.g., "SSH authorized keys file" instead of "authorized_keys")
2. Don't attach the skill to a cron job

---

## 12. Skill Limitations & Gotchas

### 12.1 Skill Content Scanned by Cron Threat Detection

If attaching a skill to a cron job, the skill's content is scanned for threat patterns. `authorized_keys` in a skill will cause the cron job to fail with `RuntimeError: Potential cron threat detected`. Rephrase or remove the skill attachment.

### 12.2 Skills Don't Auto-Reload Mid-Session

Loading a skill with `/skill` makes it available for the current session. If you edit the skill file mid-session, Hermes won't pick up the changes until `/reset`.

### 12.3 Skill Auto-Match Is Approximate

The auto-match scorer is lightweight. Very similar skill descriptions may both match the same message, causing both to load. If you want precise control, use `/skill` explicitly.

### 12.4 Skills Can Confict

If two loaded skills give contradictory instructions for the same step, Hermes may follow one and ignore the other. When writing skills, check for overlap with existing skills in the same category.

### 12.5 Large Skills Slow Down Context

Each loaded skill contributes to the context window. Loading 10+ large skills on every message can eat into available tokens for your actual task. Use targeted loading via `/skill` rather than relying solely on auto-match for large skills.

### 12.6 Skill Execution Is Mental, Not Mechanical

Skills are read and followed by Hermes — they're not executed as code. A poorly written skill (ambiguous steps, missing error handling) produces unreliable results. Invest time in writing clear, numbered steps with verification.

### 12.7 cronjob Tool Skills Attachment vs Slash Loading

| | `skills: ["skill-name"]` in cronjob | `/skill skill-name` in chat |
|-|-------------------------------------|------------------------------|
| Scope | Job execution only | Session lifetime |
| Timing | Loaded when job fires | Loaded immediately and persists |
| Threat scan | Yes — scanned by cron threat detection | No scan |
| Can fail | Yes — skill may be blocked by threat scan | No |

---

*For the Hermes Agent backend guide, see [HERMES-AGENT-GUIDE.md](./HERMES-AGENT-GUIDE.md). For the WebUI guide, see [HERMES-WEBUI-GUIDE.md](./HERMES-WEBUI-GUIDE.md). For container quick reference, see [HERMES_CHEATSHEET.md](./HERMES_CHEATSHEET.md).*