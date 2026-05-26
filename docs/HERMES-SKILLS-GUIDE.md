# Hermes Skills — Complete Guide

> **146 skills across 26 categories** | **Auto-loading** | **Skill-selector scores every turn**
> **Docs:** [hermes-agent.nousresearch.com/docs/reference/skills-catalog](https://hermes-agent.nousresearch.com/docs/reference/skills-catalog)

Skills are Hermes Agent's self-improvement system. They're Markdown procedure documents that the agent loads when relevant — encoding tested workflows, not just facts. With every session, Hermes learns your specific environment and improves.

---

## Table of Contents

1. [What Are Skills?](#1-what-are-skills)
2. [How Skills Load — Three Mechanisms](#2-how-skills-load--three-mechanisms)
3. [The 26 Categories — Full Reference](#3-the-26-categories--full-reference)
4. [Skill Selector — How Auto-Matching Works](#4-skill-selector--how-auto-matching-works)
5. [Core Skills to Start With](#5-core-skills-to-start-with)
6. [Authoring New Skills](#6-authoring-new-skills)
7. [Skill File Format](#7-skill-file-format)
8. [Limitations and Pitfalls](#8-limitations-and-pitfalls)
9. [Skill Management Commands](#9-skill-management-commands)

---

## 1. What Are Skills?

A skill is a Markdown file with YAML frontmatter that encodes a reusable, tested procedure. Unlike a one-shot prompt, a skill is:

- **Persistent** — lives in `~/.hermes/skills/`, loaded on demand
- **Contextual** — auto-matched to your message using keyword + description scoring
- **Versioned** — tracked in git, self-improving over time
- **Tested** — verified workflow before being relied upon

**Why skills beat prompts:**

| Prompt | Skill |
|--------|-------|
| "Write code" | TDD → RED → GREEN → REFACTOR with test file placement conventions |
| "Review code" | Security checklist + code quality rubric + PR comment format |
| "Deploy" | Build → push → CI check → notify failure chain |

A bad procedure produces bad results. Skills encode *how* to do something correctly, not just *what* to do.

---

## 2. How Skills Load — Three Mechanisms

### Mechanism 1: Auto-Match (skill-selector)

Every turn, the skill-selector scores all 146 skills against your message. Matches ≥3.0 load silently. Matches between 1.5–3.0 load with a notification in the agent's response. Below 1.5, not loaded.

Scoring uses:
- Keyword overlap (your message words vs skill name/tags)
- Description relevance (semantic similarity)
- Workspace context (which repos you're working in)
- Category affinity (certain categories reinforce each other)

**Example:** You type "morning briefing setup" → `morning-briefing` skill scores 4.2 and loads silently. You get the full briefing workflow without asking for it.

### Mechanism 2: Explicit /skill name

During chat, type `/skill skill-name` to load it immediately. Current session only — does not persist after `/reset`.

```
You: /skill github-pr-workflow
→ Skill loaded: github-pr-workflow
→ Ready to review PRs with full workflow
```

### Mechanism 3: -s Flag at Startup

Preload skills when Hermes starts:

```bash
# Preload multiple skills
hermes -s github-auth,coder,reviewer

# In config.yaml
model:
  default: anthropic/claude-sonnet-4
  startup_skills:
    - github-auth
    - coder
```

Good for skills you use every session — saves the auto-match overhead on common tasks.

---

## 3. The 26 Categories — Full Reference

| Category | # Skills | What It Covers |
|----------|----------|----------------|
| `agents` | 17 | Orchestrator agents (zoul, planner, coder, reviewer, etc.) |
| `autonomous-ai-agents` | 7 | Spawning/coordination of autonomous coding agents |
| `autonomous-cron-pipeline` | 4 | Multi-phase cron job chains with state files |
| `creative` | 20 | ASCII art, video, design, songwriting, image generation |
| `data-science` | 1 | Jupyter live kernel, data exploration |
| `devops` | 15 | Docker, Git, CI/CD, deployment patterns, infra-as-code |
| `github` | 7 | PR workflow, issues, repo management, code review |
| `hermes` | 2 | Hermes Agent itself (CLI, config, gateway) |
| `hermes-computer` | 1 | hermes-web-computer (Go+Svelte5 workspace) |
| `mcp` | 1 | Model Context Protocol servers and integrations |
| `media` | 5 | YouTube, Spotify, GIF search, audio spectrograms |
| `mlops` | 1 | HuggingFace hub, model serving |
| `mlops/evaluation` | 2 | LLM benchmarking, W&B experiment tracking |
| `mlops/inference` | 4 | vLLM serving, llama.cpp, GGUF quantization |
| `mlops/models` | 2 | SAM segmentation, AudioCraft music generation |
| `mlops/research` | 1 | DSPy declarative LM programs |
| `mlops/training` | 3 | Axolotl, TRL RLHF/DPO, Unsloth fine-tuning |
| `note-taking` | 1 | Obsidian vault integration |
| `planning` | 3 | Blueprint, nanobot integration, product lens |
| `productivity` | 15 | Gmail, Google Calendar, Notion, Airtable, Maps, PowerPoint |
| `red-teaming` | 1 | LLM jailbreaking techniques |
| `repo-transmute` | 1 | Vision-driven codebase migration |
| `research` | 5 | arXiv, blog monitoring, LLM Wiki, Polymarket |
| `smart-home` | 2 | Philips Hue lighting control |
| `social-media` | 3 | X/Twitter posting and monitoring |
| `software-development` | 22 | Debugging, TDD, architecture, E2E testing, Svelte, Vite |

### The Skill Taxonomy (Visual)

```
HERMES AGENT
  ├── AGENTS (17)              ← orchestrators, workers
  ├── AUTONOMOUS-AI-AGENTS (7) ← spawn code agents (Codex, Claude Code)
  ├── AUTONOMOUS-CRON-PIPELINE (4) ← chained cron jobs
  │
  ├── CREATIVE (20)            ← art, video, design, music
  ├── MEDIA (5)                ← YouTube, Spotify, GIF
  │
  ├── DEVOPS (15)              ← Docker, Git, CI/CD, IaC
  ├── GITHUB (7)               ← PR, issues, repo
  │
  ├── HERMES (2)               ← agent CLI/config
  ├── HERMES-COMPUTER (1)      ← web-computer workspace
  ├── MCP (1)                  ← Model Context Protocol
  │
  ├── MLOPS (13 total)
  │   ├── evaluation (2)       ← benchmarks, W&B
  │   ├── inference (4)        ← vLLM, llama.cpp
  │   ├── models (2)           ← SAM, AudioCraft
  │   ├── research (1)         ← DSPy
  │   └── training (3)        ← Axolotl, TRL, Unsloth
  │
  ├── NOTE-TAKING (1)          ← Obsidian
  ├── PLANNING (3)             ← blueprint, nanobot
  ├── PRODUCTIVITY (15)         ← Gmail, Calendar, Notion, etc.
  ├── RED-TEAMING (1)          ← LLM jailbreaking
  ├── REPO-TRANSMUTE (1)       ← codebase migration
  ├── RESEARCH (5)             ← arXiv, blogs, Polymarket
  ├── SMART-HOME (2)            ← Hue lights
  ├── SOCIAL-MEDIA (3)         ← X/Twitter
  │
  └── SOFTWARE-DEVELOPMENT (22) ← debugging, TDD, architecture
```

---

## 4. Skill Selector — How Auto-Matching Works

The `skill-selector` is itself a skill that runs every turn. It scores all installed skills against your message using:

```
score = keyword_score × 0.4 + description_score × 0.3 + tag_score × 0.2 + workspace_score × 0.1
```

**Keyword score:** Word overlap between your message and the skill's name/tags
**Description score:** Semantic similarity (simple keyword overlap on description words)
**Tag score:** Tag overlap with message context
**Workspace score:** Skills matching the current workspace get a bonus

**Load thresholds:**
- ≥ 3.0 → load silently ("matched: github-pr-workflow")
- 1.5–3.0 → load with notification ("loading: morning-briefing")
- < 1.5 → not loaded

**For cron jobs:** The skill-selector doesn't run in cron job contexts. You must explicitly list skills in the `skills:` parameter when creating a cron job.

---

## 5. Core Skills to Start With

### Start Here

| Skill | Category | Why You Need It |
|-------|----------|-----------------|
| `hermes-agent` | hermes | Complete CLI/config reference. Load when anything isn't working. |
| `skill-selector` | hermes | Understand how auto-matching works — essential for debugging skill loading |
| `github-auth` | github | Set up HTTPS tokens and SSH keys correctly. Do this first before any Git operations. |

### Daily Driving

| Skill | Category | Use Case |
|-------|----------|----------|
| `github-pr-workflow` | github | Review PRs: diff → comments → CI check → merge |
| `morning-briefing` | productivity | Mon-Fri 8am: email + calendar + weather → Discord |
| `autonomous-cron-pipeline` | autonomous-cron-pipeline | Multi-phase projects that span multiple cron ticks |
| `coder` | agents | Code generation with best practices |
| `reviewer` | agents | QA before code is committed |

### Specialized

| Skill | Category | Use Case |
|-------|----------|----------|
| `repo-transmute` | repo-transmute | Large codebase migration (OpenClaw → Hermes, etc.) |
| `llama-cpp` | mlops/inference | Local GGUF inference with HF Hub model discovery |
| `serving-llms-vllm` | mlops/inference | High-throughput LLM serving with OpenAI API |
| `e2e-testing` | software-development | Playwright E2E test patterns |
| `test-driven-development` | software-development | TDD workflow (RED → GREEN → REFACTOR) |
| `vite-patterns` | software-development | Vite build tool patterns and plugins |
| `fine-tuning-with-trl` | mlops/training | RLHF/DPO/GRPO training with TRL |
| `unsloth` | mlops/training | 2-5x faster LoRA/QLoRA fine-tuning |
| `huggingface-hub` | mlops | Search/download/upload models and datasets |
| `blogwatcher` | research | Monitor blogs and RSS/Atom feeds |
| `polymarket` | research | Query prediction markets for research |

---

## 6. Authoring New Skills

Write a skill when:
- You solved a complex problem and want to reuse the approach
- You discovered a non-obvious workflow that others should know
- You want to capture a multi-step procedure consistently
- A skill for your specific setup doesn't exist

### When to Create a Skill

✅ **Create when:**
- Multi-step procedure with 5+ tool calls
- Non-obvious approach discovered through iteration
- Environment-specific workflow (your setup, your conventions)
- Recurring task with consistent steps

❌ **Don't create when:**
- One-off quick task
- Trivial procedure (just 1-2 tool calls)
- Already covered by an existing skill

### The Skill Authoring Process

1. **Do the task manually first** — get the workflow right
2. **Write the SKILL.md** — encode the exact steps that worked
3. **Test in a throwaway session** — `/skill your-new-skill`, use it
4. **Iterate** — fix wrong steps, add pitfalls discovered during testing
5. **Save as skill** — use the `skill` tool to save it to `~/.hermes/skills/`

### After Creating a Skill

- Test it in a real session
- If it has issues, update immediately with `skill_manage(action='patch')`
- Skills that aren't maintained become liabilities — keep them current
- For complex skills, save the supporting scripts/templates in the skill's `scripts/` or `references/` directories

---

## 7. Skill File Format

Every skill is a Markdown file with YAML frontmatter:

```markdown
---
name: example-skill
category: software-development
description: What this skill does and when to use it
tags: [debugging, python, testing]
---

# Example Skill

## When to Use
Describe the exact situation this skill handles.

## Prerequisites
What must be true before using this skill?

## Steps

1. **Do X** — exact command or action
2. **Then Y** — the next step
3. **Finally Z** — completion check

## Verification
How do you know it worked?

## Pitfalls
Common mistakes and how to avoid them.
```

### Frontmatter Fields

| Field | Required | Description |
|-------|----------|-------------|
| `name` | Yes | Skill identifier (kebab-case, max 64 chars) |
| `category` | Yes | One of the 26 categories |
| `description` | Yes | 1-2 sentences — used by skill-selector for matching |
| `tags` | Recommended | Array of keywords — used for scoring |

### Good Skill Descriptions

**Bad:** "Manages cron jobs."
**Good:** "Create, monitor, pause, and resume Hermes Agent cron jobs. Includes schedule formats, delivery options, and state file patterns for multi-phase work."

**Bad:** "GitHub tools."
**Good:** "Full GitHub PR lifecycle: fetch diff, review code, post comments, check CI status, and merge. Works with gh CLI and GitHub REST API."

---

## 8. Limitations and Pitfalls

### ⚠️ Cron Threat Scanning

Skills containing `authorized_keys` trigger blocks in cron job context. The threat scanner flags SSH key management terms. Use paraphrases:
- ❌ "authorized_keys file"
- ✅ "SSH key file" or "key file" or "SSH authorized keys"

### ⚠️ No Mid-Session Reload

Editing or installing skills mid-session requires `/reset` for changes to take effect. The agent reads skills at session start only.

### ⚠️ Large Skills (>100KB)

Always asks before auto-loading large skills. Use explicit `/skill large-skill-name` for these.

### ⚠️ Category Conflicts

Last-loaded wins in the same category. If you load two skills from the same category, the second overwrites the first's tool context. Check for overlapping skills before installing new ones.

### ⚠️ Skills Are Procedures, Not Prompts

A bad procedure produces bad results. If a skill's steps are wrong, the output will be wrong. Always test new skills in throwaway sessions before relying on them.

### ⚠️ Skills in Cron Contexts

`MEMORY.md` and user profile are NOT loaded in cron job contexts. Cron jobs only get the attached skills and their own prompt. If a cron job needs user preferences, include them in the prompt.

### ⚠️ Skills Don't Persist /reset

Explicitly loaded skills don't persist through `/reset`. If you always need a skill, add it to the `-s` flag or `startup_skills` in config.yaml.

---

## 9. Skill Management Commands

```bash
# Browse skill catalog (interactive)
hermes skills browse

# List all installed skills
hermes skills list

# Show skill info
hermes skills info <skill-name>

# Install from hub
hermes skills install <skill-id>

# Remove a skill
hermes skills remove <skill-name>

# Update (pull latest from hub)
hermes skills update <skill-name>

# Check skill health
hermes skills doctor

# View skill file directly
cat ~/.hermes/skills/<skill-name>/SKILL.md
```

### Via WebUI

- `/skills` — browse, filter, install, remove
- Click any skill → see full content, readme, references
- Risk badges on high-risk skills
- Category filter + search

---

## Skill Naming Conventions

Skills follow kebab-case naming: `github-pr-workflow`, `morning-briefing`, `test-driven-development`.

When creating new skills, check the naming conventions:
- Agents: `zoul`, `codi`, `coder` (agent names, short)
- Workflows: `github-pr-workflow` (noun-verb)
- Platforms: `huggingface-hub`, `notion-api` (platform-thing)
- Categories: consistent with existing skills in the same category

---

## External Resources

- **Skills Catalog:** [hermes-agent.nousresearch.com/docs/reference/skills-catalog](https://hermes-agent.nousresearch.com/docs/reference/skills-catalog)
- **Skill Authoring Guide:** [hermes-agent.nousresearch.com/docs/user-guide/authoring-skills](https://hermes-agent.nousresearch.com/docs/user-guide/authoring-skills)
- **Skills Hub (Nous):** [github.com/NousResearch/hermes-agent-skills](https://github.com/NousResearch/hermes-agent-skills)