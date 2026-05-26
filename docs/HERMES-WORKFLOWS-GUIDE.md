# Hermes Workflows — Complete Guide

> **7 end-to-end workflows** | **Multi-agent patterns** | **Cron automation** | **Self-hosting optimized**

Workflows are the practical applications of Hermes Agent, WebUI, and Skills working together. This guide walks through real-world automation patterns you can use today.

---

## Table of Contents

1. [Morning Briefing](#1-morning-briefing)
2. [Code Review](#2-code-review)
3. [Multi-Agent Orchestration](#3-multi-agent-orchestration)
4. [Multi-Phase Migration](#4-multi-phase-migration)
5. [Autonomous Cron Pipeline](#5-autonomous-cron-pipeline)
6. [Daily Git Workflow](#6-daily-git-workflow)
7. [Media Pipeline](#7-media-pipeline)
8. [Troubleshooting Workflows](#8-troubleshooting-workflows)

---

## 1. Morning Briefing

**Automates:** Email → Calendar → Weather → Discord
**Schedule:** Weekdays at 8am via cron job
**Skills needed:** `morning-briefing`, `gmail`, `gcalcli-calendar`

### The Workflow

```
8:00 AM — Cron job fires
    ↓
morning-briefing skill loads
    ↓
1. Gmail API → fetch unread, important emails
2. gcalcli → today's meetings with times
3. OpenWeatherMap → current weather + forecast
4. Synthesize briefing text
    ↓
5. Discord send_message → #daily channel
    ↓
You wake up, read phone, start your day
```

### Setup

```bash
# Create the cron job via the WebUI or CLI
hermes cron create 0 8 * * MON-FRI

# Prompt:
"Run morning briefing for Sean. Read Gmail (unread important),
check Google Calendar for today, fetch weather, then post
summary to Discord channel #daily."

# Attach skills:
- morning-briefing
- gmail
- gcalcli-calendar

# Deliver: origin (sends back to this session)
```

### What the Briefing Contains

> "Good morning Sean — Tuesday, May 26, 2026.
>
> 📧 **Email:** 3 unread (1 urgent: 'Q2 budget review needed by noon')
> 📅 **Meetings:** Standup 9:30am, Architecture review 2:00pm
> 🌤️ **Weather:** 72°F, partly cloudy, rain expected 3-5pm
>
> You have 5 hours of focus time before your first meeting."

### Why This Works

- Fires before you're at your desk — briefing is ready when you wake up
- All async — no interaction needed once set up
- Delivered to phone via Discord push notification
- NoLLM-dependent — skill does the work, not just a prompt

---

## 2. Code Review

**Automates:** PR diff → analysis → comments → CI check → merge
**Trigger:** Manual (run when you have a PR to review)
**Skills needed:** `github-pr-workflow`, `github-auth`, `security-review`

### The Workflow

```
You: "Review PR #47 in ChonSong/agent-os"
    ↓
github-pr-workflow skill loads
    ↓
1. gh pr diff #47 → fetch full diff
2. Read changed files + context
3. Run security checklist (OWASP top 10)
4. Check code quality (naming, complexity, tests)
5. Post inline comments via gh pr comment
    ↓
6. gh pr checks #47 → wait for CI
    ↓
7a. If all green → gh pr merge #47
7b. If failures → summarize what needs fixing
    ↓
Report delivered to you (or Discord if you prefer)
```

### Example Review Output

```
PR #47: Add rate limiting middleware
Reviewer: hermes-agent ✏️

✅ Security: No injection vectors, input validation present
✅ Tests: +42 lines, 3 new test cases
⚠️  Complexity: `rateLimit.js` has 3 nested callbacks — consider async/await
❌ Docs: Missing JSDoc on `rateLimit()` function

CI Status: ✅ 7 checks passing

Recommendation: Merge after addressing the JSDoc issue.
```

### Key Features

- **Inline comments** — specific line feedback via `gh pr comment`
- **CI integration** — waits for all checks before merging
- **Security checklist** — OWASP-aware, flags common issues
- **Byte-verified** — can't merge if CI is red

---

## 3. Multi-Agent Orchestration

**Automates:** Planning → Research → Build → Review → Deploy
**Trigger:** Complex projects requiring multiple specialties
**Skills needed:** `planner`, `researcher`, `coder`, `reviewer`, `autonomous-cron-pipeline`

### The Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                     ORCHESTRATOR (Planner)                   │
│  "Build a FastAPI auth service for ChonSong/agent-os"         │
│  State: PHASE_TRACKER.json                                   │
└─────────────┬────────────────────┬──────────────────────────┘
      ┌───────┴───────┐     ┌───────┴───────┐
      ▼               ▼     ▼               ▼
 [Researcher]    [Coder]  [Reviewer]    [Reporter]
 API auth best   Writes   Security QA    Updates kanban
 practices      endpoints  byte-verify     reports to you
      │               │         │               │
      └───────────────┴────┬────┘               │
                           ▼                    │
                    [Byte-verified]            │
                    review gate                │
                           │                    │
                           └────────────────────┘
                                    │
                              [You / Discord]
```

### Phase Breakdown

| Phase | Agent | Responsibility |
|-------|-------|----------------|
| 1. Research | `researcher` | API auth best practices, existing patterns in repo |
| 2. Build | `coder` | Write endpoints, models, middleware |
| 3. Review | `reviewer` | Security QA, code quality, tests |
| 4. Gate | `reviewer` | Byte-verification (code actually runs) |
| 5. Report | `reporter` | Update kanban, summarize for you |

### Delegation Patterns

**Quick parallel work (within session):**
```
delegate_task(context="...", goal="research API auth patterns", toolsets=["web"])
```
→ Research agent fires in parallel, results come back when done.

**Long-lived workers (tmux, outlive session):**
```bash
tmux new-session -d -s builder -x 120 -y 40 'hermes --continue build-auth-service'
tmux new-session -d -s reviewer -x 80 -y 30 'hermes --continue review-auth-service'
```

**Swarm mode (full orchestration):**
→ Persistent workers with role-based dispatch. Orchestrator assigns tasks. Kanban board tracks progress. Review gates verify before handoff.

---

## 4. Multi-Phase Migration

**Automates:** Legacy codebase → modern framework (AST parse → LLM rewrite → verify → self-heal)
**Trigger:** Manual (run when migrating a codebase)
**Skills needed:** `repo-transmute`, `autonomous-cron-pipeline`, `e2e-testing`

### Repo-Transmute Process

```
┌─────────────────────────────────────────────────────┐
│  EXTRACT — AST parsing                               │
│  • Parse source code into AST                        │
│  • Identify 629 components                           │
│  • Map 263 API endpoints                            │
│  • Group by dependency layers                        │
└─────────────────────────────────────────────────────┘
                         ↓
┌─────────────────────────────────────────────────────┐
│  MIGRATE — LLM rewrite                              │
│  • Batch process (50 components/phase)              │
│  • Rewrite each component to target framework       │
│  • Track progress in PHASE_TRACKER.json             │
└─────────────────────────────────────────────────────┘
                         ↓
┌─────────────────────────────────────────────────────┐
│  VERIFY — Playwright visual regression               │
│  • Compare old vs new screenshots                   │
│  • Behavioral equivalence checks                    │
│  • Delta highlighted for manual review              │
└─────────────────────────────────────────────────────┘
                         ↓
┌─────────────────────────────────────────────────────┐
│  SELF-HEAL — Automatic correction                    │
│  • Mismatches re-fed to LLM                         │
│  • Automatic correction                              │
│  • Re-verify → continue                             │
└─────────────────────────────────────────────────────┘
                         ↓
┌─────────────────────────────────────────────────────┐
│  DEPLOY — GitHub + CI                               │
│  • Push verified code                               │
│  • CI runs full test suite                         │
│  • Reviewer agent byte-verifies                    │
│  • Kanban card → done                              │
└─────────────────────────────────────────────────────┘
```

### Example: OpenClaw → Hermes Migration

```
Source: OpenClaw/agent-os (legacy Node/Express+React+Postgres)
Target: Hermes Agent (Go+Svelte5)
Components: 629 identified
APIs: 263 mapped

Phase 1: Extract all routes, components, models
Phase 2: Migrate auth layer (50 components)
Phase 3: Migrate API routes
Phase 4: Migrate frontend components
Phase 5: Migration of worker processes
Phase 6: Final integration + E2E tests
```

---

## 5. Autonomous Cron Pipeline

**Automates:** Long-running work across multiple cron ticks
**Trigger:** Schedule or one-shot with state tracking
**Skills needed:** `autonomous-cron-pipeline`, any domain-specific skills

### How It Works

```
Cron tick fires
    ↓
Read PHASE_TRACKER.json (what phase are we on?)
    ↓
Execute current phase
    ↓
Update PHASE_TRACKER.json (advance to next phase)
    ↓
If more phases → wait for next cron tick
If done → deliver output → cleanup
```

### PHASE_TRACKER.json Example

```json
{
  "project": "hermes-guide-docs",
  "total_phases": 5,
  "current_phase": 2,
  "phases": [
    { "id": 1, "status": "done", "output": "Extracted 34 slides from deepseek reference" },
    { "id": 2, "status": "in_progress", "output": "Rebuilding slideshow with workflows" },
    { "id": 3, "status": "pending", "task": "Fix skills-graph embed" },
    { "id": 4, "status": "pending", "task": "Add screenshots" },
    { "id": 5, "status": "pending", "task": "Deploy + CNAME" }
  ],
  "created_at": "2026-05-26T01:00:00Z",
  "last_updated": "2026-05-26T01:31:00Z"
}
```

### Chain Example: Multi-Agent Migration

```bash
# Phase 1: Extract
hermes cron create "0 */3 * * *" \
  --prompt "Extract codebase components using repo-transmute" \
  --skills repo-transmute,autonomous-cron-pipeline \
  --deliver origin

# Phase 2: Migrate
# (auto-triggered when phase 1 done, using context_from)

# Phase 3: Verify
# (auto-triggered when phase 2 done)

# Phase 4: Deploy
# (auto-triggered when phase 3 done)
```

### When to Use

- ✅ Large codebase migration (spans hours/days)
- ✅ Multi-step build processes
- ✅ Research tasks that need to run overnight
- ✅ Automated testing suites across multiple repos

- ❌ Quick one-off tasks (use direct `hermes chat -q`)
- ❌ Tasks requiring human input mid-process

---

## 6. Daily Git Workflow

**Automates:** Branch → Commit → PR → Review → Merge
**Trigger:** Daily or per-feature
**Skills needed:** `github-pr-workflow`, `git-workflow`, `code-review`

### The Flow

```
Morning: Start new feature
    ↓
hermes: "Create a branch for the new auth feature"
    ↓
Work on feature throughout the day
    ↓
hermes: "Commit changes with meaningful message"
    ↓
hermes: "Push branch and open PR"
    ↓
CI runs automatically
    ↓
hermes: "Review PR when CI passes"
    ↓
Merge or request changes
    ↓
End of day: summary of what was done
```

### Specific Commands

```bash
# Create branch
hermes chat -q "Create a branch called feature/auth-overhaul in ChonSong/hermes-agent"

# Commit with message
hermes chat -q "Commit the auth changes with message 'refactor: extract auth middleware to separate module'"

# Push and create PR
hermes chat -q "Push feature/auth-overhaul and create PR against main with title 'refactor: auth middleware' and body template from .github/PULL_REQUEST_TEMPLATE.md"

# Check CI status
hermes chat -q "Check CI status on PR #47"

# Review PR
hermes chat -q "Review PR #47 in ChonSong/hermes-agent" -s github-pr-workflow

# Merge if green
hermes chat -q "Merge PR #47 if all checks pass"
```

---

## 7. Media Pipeline

**Automates:** Content research → writing → media generation → posting
**Trigger:** Manual or scheduled
**Skills needed:** `youtube-content`, `spotify`, `manim-video`, `xurl`, `media-maker`

### Content Pipeline

```
Research: youtube-content → transcripts to summaries
    ↓
Writing: Notion/Google Docs draft
    ↓
Media: manim-video (animations), spotify (music), pixel-art (visuals)
    ↓
Post: xurl (X/Twitter), youtube (YouTube), discord (community)
    ↓
Monitor: blogwatcher (track engagement)
```

### YouTube to Blog Pipeline

```bash
# Get transcript
hermes chat -q "Get the transcript for https://youtube.com/watch?v=XXXXX" -s youtube-content

# Summarize into blog post
hermes chat -q "Turn this transcript into a blog post with sections, code examples where relevant, and a summary"

# Post to Notion
hermes chat -q "Create a new Notion page titled '[Video Title]' with the blog content" -s notion

# Share on X
hermes chat -q "Post a thread about the video to X/Twitter" -s xurl
```

---

## 8. Troubleshooting Workflows

### Workflow: Diagnose a Broken Tool

```
You: "Web search isn't working"
    ↓
Step 1: hermes tools list → verify web toolset enabled
    ↓
Step 2: Check .env has TAVILY_API_KEY
    ↓
Step 3: curl -s "https://api.tavily.com/search?q=test" --header "Authorization: Bearer $TAVILY_API_KEY"
    ↓
Step 4: /reset (reload tools)
    ↓
Result: Which step failed → what's broken
```

### Workflow: Fix a Failing Cron Job

```
Cron job fails
    ↓
1. hermes cron list → find the failing job
2. hermes cron logs <job-id> → see the error
3. hermes doctor → check system health
    ↓
Common fixes:
- Missing skill → re-attach the skill
- API key expired → update .env
- /reset required → new session for tool changes
- Threat scan block → remove 'authorized_keys' term from prompt
```

### Workflow: Debug Session Issues

```
Session acting strange
    ↓
1. /status → current session info
2. hermes sessions list → check session count
3. hermes sessions prune --older-than 7 → clean up old sessions
4. /reset → fresh session
    ↓
If still broken:
- Check .env for API key issues
- hermes doctor
- Check logs in ~/.hermes/logs/
```

---

## Workflow Templates

### Template: Any New Cron Job

```bash
hermes cron create SCHEDULE \
  --prompt "What to do" \
  --skills skill1,skill2 \
  --deliver origin \
  --workdir /opt/data
```

### Template: Any Multi-Phase Project

```
1. Create PHASE_TRACKER.json
2. Create cron jobs for each phase with context_from linking
3. Monitor via hermes cron list
4. Deliver final output to Discord
```

### Template: Code Review Anywhere

```
hermes chat -q "Review PR #[NUMBER] in [REPO]" -s github-pr-workflow
```

---

## External Resources

- **Morning Briefing Skill:** `morning-briefing` (installed with Hermes)
- **GitHub PR Workflow:** `github-pr-workflow`
- **Multi-Agent Patterns:** `autonomous-cron-pipeline`, `planner`, `coder`, `reviewer`
- **Repo Transmute:** [github.com/ChonSong/repo-transmute](https://github.com/ChonSong/repo-transmute)
- **Hermes Docs:** [hermes-agent.nousresearch.com/docs](https://hermes-agent.nousresearch.com/docs/)