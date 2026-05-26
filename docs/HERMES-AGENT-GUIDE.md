# Hermes Agent — Complete User Guide

> **Version:** 2.x | **Install:** `curl -fsSL https://raw.githubusercontent.com/NousResearch/hermes-agent/main/scripts/install.sh | bash`
> **Docs:** [hermes-agent.nousresearch.com/docs](https://hermes-agent.nousresearch.com/docs/)
> **GitHub:** [github.com/NousResearch/hermes-agent](https://github.com/NousResearch/hermes-agent)

This guide covers Hermes Agent as running in your Docker container environment. Path mappings (host → container) are noted throughout.

---

## Table of Contents

1. [What Is Hermes Agent?](#1-what-is-hermes-agent)
2. [The Three Interfaces](#2-the-three-interfaces)
3. [How the Agent Loop Works](#3-how-the-agent-loop-works)
4. [Sessions — Fork, Resume, Prune](#4-sessions--fork-resume-prune)
5. [Memory — Three-Tier System](#5-memory--three-tier-system)
6. [Example Workflows](#6-example-workflows)
7. [Cron Jobs — Autonomous Engine](#7-cron-jobs--autonomous-engine)
8. [Delegation & Multi-Agent](#8-delegation--multi-agent)
9. [Configuration Deep Dive](#9-configuration-deep-dive)
10. [Key Gotchas & Limitations](#10-key-gotchas--limitations)
11. [Troubleshooting](#11-troubleshooting)
12. [CLI Reference](#12-cli-reference)

---

## 1. What Is Hermes Agent?

Hermes Agent is an open-source AI agent framework by Nous Research. It runs autonomously in your terminal, as a messaging bot, or as a browser workspace — using tool-calling to read files, run shell commands, search the web, manage cron jobs, send messages, and more. It has persistent memory, a self-improvement skill system, and supports 20+ LLM providers.

**What makes it different from a chatbot:**

| Chat Bot | Hermes Agent |
|----------|--------------|
| Responds once, forgets | Tool calls → real actions → persistent memory |
| One conversation | Multi-session with fork, resume, branch |
| Fixed knowledge | Learns via skills, remembers across sessions |
| Single interface | CLI + TUI + Telegram + Discord + 15+ platforms |
| One model | Swaps providers mid-workflow |

**In this setup**, Hermes Agent runs inside a Docker container (`hermes`). External access is via:
- **Hermes WebUI** (`:9119`) — browser-based chat + workspace UI
- **Hermes TUI** (terminal UI) — `docker exec -it hermes /opt/hermes/.venv/bin/hermes --tui`
- **Messaging platforms** (Telegram, Discord) via the gateway service

---

## 2. The Three Interfaces

### 2.1 WebUI (Recommended)

The browser-based command center at `http://localhost:9119`. Full workspace with chat, files, terminal, skills browser, memory, cron jobs, and dashboard. Installable as a PWA for an app-like experience on mobile.

**Start:** `docker compose -f ~/.hermes/hermes-agent/docker/docker-compose.yml up -d`

### 2.2 CLI (One-shot)

Single queries from the terminal, no TTY needed:

```bash
# Single query, returns and exits
docker exec hermes /opt/hermes/.venv/bin/hermes chat -q "What version is Docker?"

# With a specific model/provider
docker exec hermes /opt/hermes/.venv/bin/hermes chat -q "Explain GRPO" \
  --provider openrouter -m anthropic/claude-sonnet-4

# Quiet mode (no banner — good for scripts)
docker exec hermes /opt/hermes/.venv/bin/hermes chat -q "hello" -Q

# Pipe mode (NO session info, just output)
docker exec hermes /opt/hermes/.venv/bin/hermes -z "hello world"
```

### 2.3 TUI (Interactive Terminal UI)

The Ink/React-based text interface with rich formatting, spinner animations, and command autocomplete. Requires a real pseudo-terminal (`-it`):

```bash
docker exec -it hermes /opt/hermes/.venv/bin/hermes --tui
```

### 2.4 Messaging Bots (24/7)

Telegram and Discord bots running via the gateway service. Configure via `hermes gateway setup`, start via `hermes gateway run`.

---

## 3. How the Agent Loop Works

Every message you send goes through the same cycle:

```
1. YOU send message
       ↓
2. System prompt injected
   ├── SOUL.md (agent personality)
   ├── MEMORY.md (curated facts about you)
   ├── session history (recent conversation)
   └── skills (auto-loaded or manually loaded)
       ↓
3. LLM decides:
   ├── TEXT RESPONSE → return to you
   └── TOOL CALL(S) → execute each tool
                        ↓
                   Feed results back
                   as new messages → step 3
```

**Example tool calls:** `read_file`, `terminal`, `search_files`, `web_search`, `send_message`, `cronjob`, `delegate_task`, `execute_code`, `todo`

The loop repeats until the model gives a plain text response. Each tool execution feeds results back as a new message, giving the model full context of what happened.

---

## 4. Sessions — Fork, Resume, Prune

Every conversation is stored as an indexed session. Sessions are the backbone of continuity — they enable branching, resumption, and memory across time.

### Viewing Sessions

```bash
hermes sessions list                    # List all sessions with IDs and dates
hermes sessions browse                  # Interactive picker
hermes sessions stats                  # Session store statistics
```

### Branching (Fork)

Use `/fork` or `/branch` during a chat to create a parallel branch of the conversation. This lets you experiment without losing the original:

```
You: /fork
→ New session branched from [session name]
→ Original preserved, new branch active
```

### Resuming

```bash
# Resume by name
hermes --continue my-session-name

# Resume by ID
hermes --resume 20260225_143052_a1b2c3

# Interactive picker
hermes sessions browse
```

### Pruning (Cleanup)

Sessions accumulate fast with frequent cron jobs. Prune old ones regularly:

```bash
hermes sessions prune --older-than 7    # Delete sessions older than 7 days
hermes sessions prune --older-than 30   # Keep last 30 days only
```

### Export

```bash
hermes sessions export ~/hermes-sessions-2026-05.jsonl
```

---

## 5. Memory — Three-Tier System

### Tier 1: SOUL.md — Identity

`~/.hermes/SOUL.md` (container: `/opt/data/SOUL.md`)

The agent's personality and core principles. Always loaded first, never asks questions. Defines how Hermes approaches problems — genuine helpfulness over performative politeness, having opinions, being resourceful before asking.

**Contents typically include:**
- Core truths ("be genuinely helpful, not performatively helpful")
- Personality traits ("have opinions", "be concise by default")
- Session behavior rules ("every session, wake up fresh")
- Key conventions for this setup

### Tier 2: MEMORY.md — Curated Knowledge

`~/.hermes/MEMORY.md` (container: `/opt/data/MEMORY.md`)

Curated long-term facts about you and your environment. Loaded in main sessions only (not in cron contexts). Updated by the agent after significant events, lessons, or decisions.

**Contents typically include:**
- User preferences (concise output, preferred workflows)
- Environment details (host IP, Docker setup, custom paths)
- Project conventions (branching strategy, testing approach)
- Learned lessons (don't do X, always use Y instead)

### Tier 3: Daily Logs — Raw History

`~/.hermes/memory/YYYY-MM-DD.md`

Raw session logs in daily files. Searchable via `session_search` tool. The agent distills significant events into MEMORY.md during periodic maintenance heartbeats. These files are the raw material — MEMORY.md is the curated summary.

### Memory Behavior Summary

| File | Loaded In | Purpose | Updated By |
|------|-----------|---------|------------|
| `SOUL.md` | Every session | Who Hermes is | Rarely (core identity) |
| `MEMORY.md` | Main sessions | What Hermes knows | Agent after significant events |
| `memory/YYYY-MM-DD.md` | session_search | What happened when | Every session automatically |

---

## 6. Example Workflows

### 6.1 Morning Briefing Workflow

A complete end-to-end automation that fires every weekday at 8am.

**1. Schedule the cron job:**
```
hermes cron create 0 8 * * MON-FRI
```
Attach skill `morning-briefing`. Deliver to your Discord channel.

**2. Job fires at 8am → loads morning-briefing skill**

**3. Skill executes:**
- Reads email via Gmail API (unread, important)
- Checks Google Calendar via gcalcli (today's meetings)
- Fetches weather via OpenWeatherMap

**4. Synthesizes briefing:**
> "Good morning Sean — 2 meetings today (standup 9:30am, architecture review 2pm), rain expected at 2pm so bring an umbrella, one urgent email from the team about the deadline."

**5. Delivers to Discord via send_message → your #daily channel**

**6. You wake up, open phone, read briefing, start your day.**

---

### 6.2 Code Review Workflow

Full PR lifecycle from review request to merge.

**1. Initiate review:**
```
hermes chat -q "Review PR #47 in ChonSong/agent-os" -s github-pr-workflow
```

**2. github-pr-workflow skill loads → fetches PR diff:**
```bash
gh pr diff #47
```

**3. Reads changed files, analyzes:**
- Code quality and style
- Security issues (OWASP checklist)
- Test coverage
- Documentation completeness

**4. Posts review comments via:**
```bash
gh pr comment #47 --body "LGTM overall. Two nits: ..."
```

**5. Checks CI status:**
```bash
gh pr checks #47
```
- If pending → waits and retries
- If failed → summarizes failures

**6. If all green → merges:**
```bash
gh pr merge #47 --admin --squash
```
If not green → posts summary of what needs fixing.

---

### 6.3 Multi-Agent Migration Workflow

Using repo-transmute + autonomous-cron-pipeline for large codebase migrations.

**1. Identify the legacy codebase** (e.g., OpenClaw → Hermes)

**2. Extract** — repo-transmute AST-parses the codebase:
- Identifies 629 components
- Maps 263 API endpoints
- Groups by dependency layers

**3. Migrate** — LLM rewrites each component against target framework:
- Batch processing (50 components per phase)
- Progress tracked via PHASE_TRACKER.json

**4. Verify** — Playwright screenshots compare old vs new:
- Visual regression detection
- Behavioral equivalence checks
- Delta highlighted for manual review

**5. Self-heal** — mismatches automatically re-fed to LLM:
```
Mismatch detected at auth/login.tsx
→ Re-feeding to LLM for automatic correction
→ Verification pass
→ Continue
```

**6. Deploy** — verified code pushed to GitHub:
- CI runs full test suite
- Reviewer agent byte-verifies output
- Kanban card moves to done

---

### 6.4 Multi-Agent Orchestration Pattern

For complex projects that need multiple specialized agents.

```
┌─────────────────────────────────────────────────────┐
│              ORCHESTRATOR (Planner)                 │
│  "Build a FastAPI auth service for agent-os repo"   │
└───────────────┬─────────────────────┬───────────────┘
        ┌───────┴───┐           ┌──────┴──────┐
        ▼           ▼           ▼             ▼
  [Researcher] [Coder]   [Reviewer]    [Reporter]
  API auth best  Writes   Security QA    Updates
  practices     endpoints    ↓         kanban
        │           │         │             │
        └───────────┴────┬────┘             │
                         ▼                  │
                  [Byte-verified]           │
                  review gate              │
                         │                  │
                         └──────────────────►│
                                          [You]
                              Results delivered to Discord
```

**Key patterns:**
- **delegate_task** — for quick parallel subtasks within a session
- **tmux spawning** — for long-lived workers that outlive the parent session
- **Swarm mode** — persistent workers with role-based dispatch, kanban, review gates

---

## 7. Cron Jobs — Autonomous Engine

Cron jobs are the autonomous engine of Hermes. Schedule complex workflows that run while you sleep.

### Schedule Formats

| Format | Meaning |
|--------|---------|
| `30m` | Every 30 minutes |
| `every 2h` | Every 2 hours |
| `0 8 * * MON-FRI` | Weekdays at 8am (cron syntax) |
| `0 */6 * * *` | Every 6 hours |

### Creating Cron Jobs

```bash
# Interactive creation
hermes cron create

# Non-interactive
hermes cron create SCHEDULE "your prompt here"
```

**Via tool (recommended):** Use the `cronjob` tool in the WebUI — it has all parameters and works reliably.

### Cron Job Components

| Component | Purpose |
|-----------|---------|
| `schedule` | When it fires (30m, hourly, daily, cron syntax) |
| `prompt` | What to do (the task description) |
| `skills` | Which skills to load (attach relevant ones) |
| `deliver` | Where to send output (origin, discord, local) |
| `workdir` | Working directory for the job |
| `enabled_toolsets` | Which tools the job can use |

### Multi-Phase Work (autonomous-cron-pipeline)

For complex projects that span multiple cron ticks:

1. **PHASE_TRACKER.json** stores current phase + next steps
2. Each cron tick reads the tracker → executes one phase → updates tracker
3. Continues until all phases complete
4. Delivers final output to configured channel

### Delivery Options

| `deliver` value | Behavior |
|----------------|----------|
| `origin` | Send to the session that created the job |
| `local` | Save to filesystem only (silent) |
| `discord:CHANNEL_ID` | Post to specific Discord channel |
| `all` | Fan out to all connected home channels |

---

## 8. Delegation — Multi-Agent

### Three Patterns

**Pattern 1: delegate_task (in-session)**

Spawns parallel subagents within the current session. Fast, bounded, shared context.

```
Used for: Quick parallel subtasks (research A + research B simultaneously)
Limitation: Bounded by parent session's toolset + iteration limit
```

**Pattern 2: tmux spawning (long-lived)**

Long-lived autonomous missions that outlive the parent session.

```bash
tmux new-session -d -s agent1 -x 120 -y 40 'hermes'
tmux send-keys -t agent1 'Build a FastAPI auth service' Enter
# Agent runs for hours, independently
```

**Pattern 3: Swarm mode (orchestrated)**

Persistent tmux workers with role-based dispatch (builders, reviewers, QA, ops). Kanban board, byte-verified review gates.

### When to Use Each

| | delegate_task | tmux spawning | Swarm mode |
|-|---------------|---------------|------------|
| Duration | Minutes | Hours/days | Hours/days |
| Isolation | Shared context | Independent | Independent |
| Tool access | Subset of parent | Full | Full |
| Use case | Quick parallel work | Long autonomous missions | Complex multi-role projects |

---

## 9. Configuration Deep Dive

### Key Files

| File | Purpose |
|------|---------|
| `~/.hermes/config.yaml` | Main configuration |
| `~/.hermes/.env` | API keys and secrets |
| `~/.hermes/skills/` | Installed skills |
| `~/.hermes/sessions/` | Session transcripts |
| `~/.hermes/logs/` | Logs |

### Essential Config Sections

```yaml
# ~/.hermes/config.yaml

model:
  default: anthropic/claude-sonnet-4
  provider: openrouter

agent:
  max_turns: 150        # Default 90, raise for long debugging sessions

toolsets:
  - web
  - terminal
  - file
  - delegation
  - cronjob
  - skills
  - memory

delegation:
  max_spawn_depth: 1    # 1 = no nested delegation

memory:
  memory_enabled: true
  user_profile_enabled: true
```

### Viewing and Editing Config

```bash
hermes config              # View current config
hermes config edit         # Open in $EDITOR
hermes config set KEY VAL  # Set a value
hermes config path         # Print config path
hermes config check        # Check for missing/outdated config
```

---

## 10. Key Gotchas & Limitations

### ⚡ /reset Required After Tool Changes

Editing `config.yaml` mid-session doesn't reload tools. You must `/reset` (new session) for tool changes to take effect.

### 🔑 Discord Numeric User IDs

Settings → Advanced → Developer Mode → right-click user → Copy User ID. Usernames don't work.

### 📁 ~/workspace Not Mounted in Container

Scripts that cron jobs run inside the container must live in `/opt/data/hermes-sync/scripts/` (git-synced, read-only mount) or `/opt/data/` (writable). Any reference to `~/workspace` or `/home/sean/workspace` will fail.

### 🔁 max_turns Iteration Limit

Default: 90. Long debugging sessions (~300+ messages) hit this routinely. Set `agent.max_turns: 150` or higher in `config.yaml`, then `/reset`.

### 💾 Cron Threat Scanning

Skills containing `authorized_keys` trigger cron threat scanning blocks. Use paraphrases like `SSH key file` or `key file` instead.

### 🧠 /goal Is Single-Session Only

The `/goal` command (Ralph loop) works within ONE session. It does NOT survive session death, context compaction, or timeouts. For multi-phase work, use `autonomous-cron-pipeline` skill instead.

### 📝 Session Disk Bloat

Every session writes a JSON transcript. Cron jobs run frequently and accumulate files fast. Prune regularly:
```bash
hermes sessions prune --older-than 7
```

### ⚠️ deliver ≠ Discord Delivery

`deliver: discord` is accepted by the cron tool but the scheduler has no Discord delivery code path. For reliable Discord delivery, include `send_message` tool call in the job prompt itself.

---

## 11. Troubleshooting

### Tool not available
1. `hermes tools list` — verify the toolset is enabled
2. Check `.env` has the required API key
3. `/reset` after enabling tools

### "No web search provider configured"
**Two independent checks required — one does not imply the other:**
1. Is `web` in your `toolsets:` list in config.yaml?
2. Is your Tavily API key valid? (Get from [app.tavily.com](https://app.tavily.com) → API Keys)

```bash
# Step 1: Verify toolset
grep -A1 "^toolsets:" ~/.hermes/config.yaml

# Step 2: Test API key
curl -s "https://api.tavily.com/search?q=test&api_key=YOUR_KEY" | head -c 200
```

### Gateway dies on SSH logout
Enable linger: `sudo loginctl enable-linger $USER`

### Gateway dies on WSL2 close
WSL2 requires `systemd=true` in `/etc/wsl.conf` for systemd services to work.

### Docker build fails with --chmod error
Old Docker builder doesn't support `--chmod` on COPY. Use BuildKit: `DOCKER_BUILDKIT=1 docker build ...`

### Session files permission errors
Privileged container operations create files owned by `root`. Fix:
```bash
sudo chown -R $(id -u):$(id -g) ~/.hermes/sessions/
```

---

## 12. CLI Reference

```bash
# Chat
hermes chat -q "question"          # One-shot query
hermes --tui                       # Interactive TUI
hermes --continue [name]          # Resume session

# Config
hermes config edit                 # Edit config.yaml
hermes config set section.key val  # Set a value
hermes doctor                      # Diagnose issues
hermes status                      # Show component status

# Tools & Skills
hermes tools list                  # Show enabled toolsets
hermes skills browse               # Browse skill catalog
hermes skills install <id>         # Install from hub

# Sessions
hermes sessions list              # List all sessions
hermes sessions prune --older-than 7  # Clean old sessions
hermes sessions export OUT       # Export to JSONL

# Cron
hermes cron list                  # List scheduled jobs
hermes cron create SCHEDULE       # Create new job

# Gateway
hermes gateway run               # Start messaging bot
hermes gateway status            # Check status

# Model
hermes model                     # Interactive model picker
hermes model --provider openrouter -m anthropic/claude-sonnet-4
```

### Slash Commands (In-Session)

| Command | Action |
|---------|--------|
| `/new` or `/reset` | Fresh session |
| `/skill name` | Load a skill |
| `/goal [text]` | Set persistent goal (Ralph loop) |
| `/fork` or `/branch` | Branch current session |
| `/resume [name]` | Resume a session |
| `/tools` | Manage toolsets |
| `/status` | Session info |
| `/verbose` | Cycle verbose modes |
| `/yolo` | Toggle approval bypass |
| `/quit` | Exit CLI |

---

## External Resources

- **Docs:** [hermes-agent.nousresearch.com/docs](https://hermes-agent.nousresearch.com/docs/)
- **GitHub:** [github.com/NousResearch/hermes-agent](https://github.com/NousResearch/hermes-agent)
- **Skills Catalog:** [hermes-agent.nousresearch.com/docs/reference/skills-catalog](https://hermes-agent.nousresearch.com/docs/reference/skills-catalog)
- **Configuration Guide:** [hermes-agent.nousresearch.com/docs/user-guide/configuration](https://hermes-agent.nousresearch.com/docs/user-guide/configuration)
- **Provider Reference:** [hermes-agent.nousresearch.com/docs/integrations/providers](https://hermes-agent.nousresearch.com/docs/integrations/providers)
- **Messaging Platforms:** [hermes-agent.nousresearch.com/docs/user-guide/messaging](https://hermes-agent.nousresearch.com/docs/user-guide/messaging/)