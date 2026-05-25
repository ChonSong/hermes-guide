# Hermes Agent — Complete User Guide

> **Version:** 2.x | **Stack:** Python + SQLite + React/Ink TUI | **Install:** `curl -fsSL https://raw.githubusercontent.com/NousResearch/hermes-agent/main/scripts/install.sh | bash`
>
> This guide covers Hermes Agent as running in your Docker container environment. Path mappings (host → container) are noted throughout. See [HERMES_CHEATSHEET.md](./HERMES_CHEATSHEET.md) for a quick-reference version of the container commands.

---

## Table of Contents

1. [What Is Hermes Agent?](#1-what-is-hermes-agent)
2. [The Three Ways to Talk to It](#2-the-three-ways-to-talk-to-it)
3. [Core Concepts](#3-core-concepts)
4. [How the Agent Loop Works](#4-how-the-agent-loop-works)
5. [Chat Interface](#5-chat-interface)
6. [Sessions](#6-sessions)
7. [Memory](#7-memory)
8. [Skills](#8-skills)
9. [Cron Jobs](#9-cron-jobs)
10. [Delegation & Multi-Agent](#10-delegation--multi-agent)
11. [Profiles](#11-profiles)
12. [MCP Servers](#12-mcp-servers)
13. [Voice & Transcription](#13-voice--transcription)
14. [Configuration Deep Dive](#14-configuration-deep-dive)
15. [Gateway & Messaging Platforms](#15-gateway--messaging-platforms)
16. [Toolsets Reference](#16-toolsets-reference)
17. [Limitations & Gotchas](#17-limitations--gotchas)
18. [Troubleshooting](#18-troubleshooting)

---

## 1. What Is Hermes Agent?

Hermes Agent is an open-source AI agent framework by Nous Research. It runs autonomously in your terminal or as a messaging bot, using tool-calling to read files, run shell commands, search the web, manage cron jobs, send messages, and more. It has persistent memory, a skill system for learning from experience, and supports 20+ LLM providers.

**What makes it different from a chat bot:**

| Chat Bot | Hermes Agent |
|---------|-------------|
| Responds once, forgets | Tool calls → real actions → persistent memory |
| One conversation | Multi-session with fork, resume, branch |
| Fixed knowledge | Learns via skills, remembers across sessions |
| Single interface | CLI + TUI + Telegram + Discord + 15+ platforms |
| One model | Swaps providers mid-workflow |

**In this setup**, Hermes Agent runs inside a Docker container (`hermes`). External access is via:
- **Hermes WebUI** (`:9119`) — browser-based chat + workspace UI
- **Hermes TUI** (terminal UI, `docker exec -it hermes /opt/hermes/.venv/bin/hermes --tui`)
- **Messaging platforms** (Telegram, Discord) via the gateway service
- **Hermes Workspace** (Electron desktop app) — full desktop experience

---

## 2. The Three Ways to Talk to It

### 2.1 CLI (one-shot, no TTY needed)

```bash
# Single query, returns and exits
docker exec hermes /opt/hermes/.venv/bin/hermes chat -q "What version is Docker?"

# With a specific model/provider
docker exec hermes /opt/hermes/.venv/bin/hermes chat -q "Explain GRPO" \
  --provider openrouter -m anthropic/claude-sonnet-4

# Quiet mode (no banner — good for scripts)
docker exec hermes /opt/hermes/.venv/bin/hermes chat -q "hello" -Q

# One-shot pipe mode (NO session info, just output)
docker exec hermes /opt/hermes/.venv/bin/hermes -z "hello world"
```

### 2.2 TUI (interactive terminal UI)

The TUI is an Ink/React-based text interface with rich formatting, spinner animations, and command autocomplete. You need a real pseudo-terminal (`-it`):

```bash
# Basic TUI
docker exec -it hermes /opt/hermes/.venv/bin/hermes --tui

# With specific model
docker exec -it hermes /opt/hermes/.venv/bin/hermes --tui -m anthropic/claude-sonnet-4

# With skills preloaded
docker exec -it hermes /opt/hermes/.venv/bin/hermes --tui -s github-auth,coder

# Resume last session
docker exec -it hermes /opt/hermes/.venv/bin/hermes --tui -c

# Resume by session name
docker exec -it hermes /opt/hermes/.venv/bin/hermes --tui -c "my project"
```

> **Note:** `-it` is required. Without it the TUI can't render. If you're piping or cron'ing, use the CLI `-q` mode instead.

### 2.3 Gateway (messaging platforms)

The gateway runs Hermes as a bot across Telegram, Discord, Slack, WhatsApp, Signal, Email, SMS, Matrix, and more — simultaneously. Each platform is a channel; you can have the same agent handling all of them.

```bash
# Check gateway status
docker exec hermes /opt/hermes/.venv/bin/hermes gateway status

# Setup a new platform (interactive TTY)
docker exec -it hermes /opt/hermes/.venv/bin/hermes gateway setup

# Restart gateway (after config changes)
sg docker -c "docker compose restart gateway"
```

The gateway is already running in this setup. Platforms are configured via environment variables in `~/.hermes/.env` on the host (mapped to `/opt/data/.env` inside the container).

---

## 3. Core Concepts

### 3.1 The Agent Loop

Hermes works by sending your message plus the full conversation history to an LLM, along with a list of available tools. The LLM can respond with text *or* request tool calls. If it requests a tool, Hermes executes it and feeds the result back as a new message. This repeats until the LLM gives a plain text response.

```
You → "Write a fastapi auth service"
LLM → [tool_call: write_file(path="/workspace/auth.py", ...)]
Hermes → executes write_file, returns result
LLM → [tool_call: terminal(command="pip install fastapi")]
Hermes → executes terminal, returns result
LLM → "Here's the complete auth service..."
```

Each tool call round is one **turn**. `max_turns` (default 90) caps how many rounds per session. Long debugging sessions can hit this — increase it in config if needed.

### 3.2 Toolsets

Tools are grouped into **toolsets** — collections that can be enabled or disabled together. When you `/reset` (new session), Hermes loads whatever toolset list is in `config.yaml`.

```bash
# List all toolsets and their status
docker exec hermes /opt/hermes/.venv/bin/hermes tools list

# Interactive enable/disable (TTY)
docker exec -it hermes /opt/hermes/.venv/bin/hermes tools
```

Available toolsets: `web`, `browser`, `terminal`, `file`, `code_execution`, `vision`, `image_gen`, `tts`, `skills`, `memory`, `session_search`, `delegation`, `cronjob`, `clarify`, `messaging`, `search`, `todo`, `homeassistant`, and more.

> **Tool changes require `/reset` to take effect.** Mid-session edits to config don't reload tools — start a new session.

### 3.3 Skills

Skills are reusable procedure documents (`.md` files) that Hermes loads to handle specialized tasks. When it encounters a task that matches a skill's triggers, it loads the skill's instructions and follows them. Skills accumulate over time — the more you use Hermes, the better it gets at your specific environment.

Skills live at `~/.hermes/skills/` on the host (`/opt/data/skills/` in the container). See [HERMES-SKILLS-GUIDE.md](./HERMES-SKILLS-GUIDE.md) for full detail.

### 3.4 Memory

Hermes has persistent cross-session memory. The built-in system uses Markdown files in `~/.hermes/`:

| File | Purpose |
|------|---------|
| `MEMORY.md` | Long-term curated memory — who you are, your preferences, lessons learned |
| `USER.md` | User profile and preferences |
| `memory/YYYY-MM-DD.md` | Daily logs of what happened |
| `memories/` | Additional curated memories |

On every new session, Hermes reads these files automatically. You can also edit them mid-conversation. External memory providers (Honcho, Mem0) are supported via plugin.

### 3.5 Sessions

Every conversation is stored as a session. Sessions can be:
- **Resumed** — pick up where you left off
- **Forked/Branched** — create a copy to try things without losing the original
- **Named** — give sessions meaningful titles
- **Pruned** — clean up old sessions to save disk space

Session transcripts are stored as JSON in `~/.hermes/sessions/` on the host (`/opt/data/sessions/` in the container).

---

## 4. How the Agent Loop Works

Understanding the loop helps you work *with* it, not around it.

```
User message
    ↓
System prompt (assembled from SOUL.md, context files, active skills)
    ↓
Conversation history (up to context window limit)
    ↓
LLM call (model + provider from config)
    ↓
┌─ Text response → Return to user ✓
└─ Tool calls → Execute each → Feed results back as tool messages
        ↓
    LLM call again
        ↓
    ... (loop until text response or max_turns reached)
```

**Context compression** triggers automatically when the conversation approaches the model's context limit. Hermes summarizes earlier messages to make room. This is transparent to you, but very long sessions may lose some detail from early messages.

**Key config options affecting the loop:**

```yaml
agent:
  max_turns: 200          # Max tool-call rounds per session (default 90)
  tool_use_enforcement: true  # Agent must use tools, not just answer

compression:
  enabled: true
  threshold: 0.50         # Compress when context is 50% full
  target_ratio: 0.20      # Target reducing to 20% of original size

model:
  default: minimax/MiniMax-M2.7
  provider: minimax
```

---

## 5. Chat Interface

### 5.1 Slash Commands (In-Session)

These are typed during an interactive chat (TUI or gateway) to control the session:

**Session Control:**
```
/new          Fresh session
/clear        Clear screen + new session
/retry        Resend last message
/undo         Remove last exchange
/title [name] Name the session
/compress     Manually trigger context compression
/stop         Kill background processes
/rollback [N]  Restore filesystem checkpoint (if checkpoints enabled)
/background   Run prompt in background
/queue        Queue prompt for next turn
/resume [name] Resume a named session
/goal [text]  Set a standing goal (auto-continues until achieved)
/goal pause|resume|clear|status  Control active goal
```

**Configuration:**
```
/config       Show config (TUI)
/model [name] Show or change model
/reasoning [level]  Set reasoning effort (none|minimal|low|medium|high|xhigh|show|hide)
/verbose      Cycle verbose mode
/voice [on|off|tts]  Voice mode
/yolo         Toggle approval bypass (run commands without asking)
/skin [name]  Change theme
/statusbar    Toggle status bar
```

**Tools & Skills:**
```
/tools        Manage tools (TUI)
/toolsets     List toolsets (TUI)
/skills       Search/install skills (TUI)
/skill <name> Load a skill into this session
/cron         Manage cron jobs (TUI)
/reload-mcp   Reload MCP servers
/plugins      List plugins (TUI)
```

**Gateway:**
```
/approve      Approve a pending command
/deny         Deny a pending command
/restart      Restart gateway
/sethome      Set current chat as home channel
/platforms    Show platform connection status
```

**Utility:**
```
/branch       Fork current session
/btw          Ephemeral side question (doesn't interrupt main task)
/fast         Toggle priority/fast processing
/browser     Open CDP browser connection
/history      Show conversation history
/save         Save conversation to file
/image        Attach local image
/help         Show commands
/usage        Token usage
/insights     Usage analytics
/quit         Exit CLI
```

### 5.2 The Ralph Loop (`/goal`)

`/goal your objective here` sets a persistent standing goal. After each turn, an auxiliary judge model evaluates whether the goal is satisfied. If not, it feeds a continuation prompt back into the session. This continues until the goal is achieved, the turn budget runs out, or you send a new message (which preempts the goal).

**Best for:** Single-session tasks — research summaries, quick fixes, single-file refactors.

**NOT for:** Multi-phase projects, marathon work, or anything that needs to survive session restarts. Use the Persistent Phase Engine (cron-based) for large projects.

**Requirements:**
- A judge model must be configured in `config.yaml` under `auxiliary.goal_judge`
- The judge provider must be accessible

**`/goal` pitfalls:**
- Does NOT survive session restarts or context compaction
- Judge model must be configured — otherwise always returns "continue" and burns turns
- Best for tasks completable within one session

---

## 6. Sessions

### 6.1 Managing Sessions

```bash
# List recent sessions
docker exec hermes /opt/hermes/.venv/bin/hermes sessions list

# Interactive session browser
docker exec -it hermes /opt/hermes/.venv/bin/hermes sessions browse

# Session stats (size, count)
docker exec hermes /opt/hermes/.venv/bin/hermes sessions stats

# Rename a session
docker exec hermes /opt/hermes/.venv/bin/hermes sessions rename <session_id> "My Project"

# Delete sessions older than 30 days
docker exec hermes /opt/hermes/.venv/bin/hermes sessions prune --older-than 30d

# Export all sessions to JSONL
docker exec hermes /opt/hermes/.venv/bin/hermes sessions export --output sessions.jsonl

# Resume by ID
docker exec -it hermes /opt/hermes/.venv/bin/hermes --resume <session_id>

# Resume by name
docker exec -it hermes /opt/hermes/.venv/bin/hermes --continue "my project"
```

### 6.2 Session Disk Bloat

Every session (including cron job runs) writes a JSON transcript. Frequent cron jobs produce many small files. Prune regularly:

```bash
# Prune sessions older than 7 days
docker exec hermes /opt/hermes/.venv/bin/hermes sessions prune --older-than 7

# Delete raw API request dumps (not needed for session_search)
find ~/.hermes/sessions/ -name 'request_dump_*' -delete

# Compress old sessions (gzip, still searchable)
find ~/.hermes/sessions/ -name 'session_*.json' -mtime +7 -exec gzip {} \;
```

### 6.3 Session Permission Errors

Privileged operations can create session files owned by `root:root`. The `hermes` user can't read them:

```bash
# Diagnose
ls -la ~/.hermes/sessions/ | grep root

# Fix
sudo chown -R $(id -u):$(id -g) ~/.hermes/sessions/
```

---

## 7. Memory

### 7.1 Built-in Memory Files

On session start, Hermes reads in order:
1. `SOUL.md` — the agent's personality and core principles
2. `USER.md` — who the user is, preferences, communication style
3. `MEMORY.md` — long-term curated memory
4. `memory/YYYY-MM-DD.md` — today's daily log (if it exists)
5. `memories/` — additional curated memory files

**You can edit memory mid-conversation** — Hermes reads it fresh on each new session, but edits within a session don't auto-reload. To pick up changes: `/reset`.

### 7.2 Memory Commands

```bash
# Check memory provider status
docker exec hermes /opt/hermes/.venv/bin/hermes memory status

# Setup external memory provider (Honcho, Mem0, etc.)
docker exec -it hermes /opt/hermes/.venv/bin/hermes memory setup

# Disable external provider (use built-in files only)
docker exec hermes /opt/hermes/.venv/bin/hermes memory off

# Reset built-in memory (clears MEMORY.md and USER.md)
docker exec hermes /opt/hermes/.venv/bin/hermes memory reset
```

### 7.3 What to Put in MEMORY.md

- User preferences (concise output, specific workflows)
- Environment facts (paths, installed tools, project conventions)
- Things to remember across sessions (ongoing projects, current priorities)
- Lessons learned (what worked, what didn't, corrections)

**Do NOT put:**
- Task progress or session logs (use daily memory files)
- Secrets or API keys
- Raw data dumps

### 7.4 Daily Memory Files

Create `memory/YYYY-MM-DD.md` to log what happened today. Format is free — just write what matters. These accumulate into a searchable history.

---

## 8. Skills

See [HERMES-SKILLS-GUIDE.md](./HERMES-SKILLS-GUIDE.md) for the full guide. Quick reference:

```bash
# List installed skills
docker exec hermes /opt/hermes/.venv/bin/hermes skills list

# Search available skills in the catalog
docker exec hermes /opt/hermes/.venv/bin/hermes skills search "github"

# Install a skill
docker exec hermes /opt/hermes/.venv/bin/hermes skills install github-auth

# Load a skill into current session
/skill github-auth

# Preload skills at startup
docker exec -it hermes /opt/hermes/.venv/bin/hermes --tui -s github-auth,coder
```

**Key skills for this setup:**
| Skill | When to use |
|-------|-------------|
| `hermes-agent` | Everything — CLI, config, tools, delegation, gateway |
| `github-*` | GitHub PRs, issues, repos |
| `autonomous-cron-pipeline` | Multi-phase autonomous projects |
| `morning-briefing` | Daily news + weather + calendar |
| `dogfood` | Testing your own web app |
| `browser-agent` | Web scraping and browser automation |
| `repo-transmute` | Migrating codebases |
| `hermes-computer` | Building the HWC tiling desktop |

---

## 9. Cron Jobs

### 9.1 Cron Jobs vs Heartbeats

**Cron jobs** are scheduled autonomous tasks. They run even when you're not interacting with Hermes, can execute complex multi-step workflows, and deliver output to Discord, Telegram, or files.

**Heartbeats** are lightweight periodic checks that run within your active session context. They're cheaper but only fire when you're chatting.

Use cron for:
- Exact timing ("9:00 AM every Monday")
- Isolated long-running tasks
- Tasks that need a different model or thinking budget

Use heartbeats for:
- Batch checks within a conversation (inbox + calendar + notifications)
- Things that benefit from conversational context

### 9.2 Managing Cron Jobs

> **Note:** The `hermes cron create` CLI subcommand is unreliable (always exits code 2). Use the `cronjob` tool instead — it's reliable and supports all parameters.

```bash
# List all cron jobs
docker exec hermes /opt/hermes/.venv/bin/hermes cron list

# Check scheduler status
docker exec hermes /opt/hermes/.venv/bin/hermes cron status

# Pause / resume / remove
docker exec hermes /opt/hermes/.venv/bin/hermes cron pause <job_id>
docker exec hermes /opt/hermes/.venv/bin/hermes cron resume <job_id>
docker exec hermes /opt/hermes/.venv/bin/hermes cron remove <job_id>

# Trigger immediately
docker exec hermes /opt/hermes/.venv/bin/hermes cron run <job_id>
```

**Creating cron jobs** is done via the `cronjob` tool in a chat session. The tool supports:
- `schedule` — `'30m'`, `'every 2h'`, `'0 9 * * *'`, or ISO timestamp
- `prompt` — self-contained task description
- `skills` — list of skills to load before executing
- `deliver` — where to send output (`origin`, `local`, `all`, or platform:chat_id:thread_id)
- `workdir` — working directory for the job
- `model` — override model for this job
- `no_agent` — run a raw script instead of LLM-driven task

### 9.3 Cron Job Delivery — Critical Gotchas

**`deliver: discord` does NOT work reliably.** The scheduler has no Discord delivery code path — Discord messages from timer-fired jobs only work when the job *prompt itself calls `send_message` as a tool step*. Do NOT rely on `deliver: discord`.

**`deliver: local` saves output to filesystem — you never see it.** All current cron jobs in this setup use `deliver: local` and their output is silently stored in `/opt/data/cron/output/`.

**`last_status: ok` ≠ output was delivered.** The job ran successfully but delivery can silently fail (especially Discord).

**Test delivery in the same session you configure the cron.** Fire it immediately with `cron run` and verify the output actually arrives where you expect.

**For reliable Discord delivery:** include `send_message` tool call in the job prompt itself, don't rely on `deliver: discord`.

### 9.4 Cron Environment Limitations

Cron job sessions run in a constrained environment. `terminal()` often fails with path errors. `execute_code` (Python) works reliably and can access host paths via `/home/sean/.hermes/`.

**Reliable tools in cron contexts:**
- `browser_navigate` + `browser_snapshot` — scrape web UI
- `web_extract` — fetch web/API content
- `send_message` — post to Discord, Telegram (if token is in .env)
- `execute_code` — Python subprocesses

**Unreliable in cron contexts:**
- `terminal()` for shell commands
- `gh` CLI
- `curl | python3` pipelines

### 9.5 Cron Job Threat Scanning

The cron system scans prompts for suspicious strings (SSH keys, authorized_keys, etc.) and blocks execution if found. If a skill you want to attach contains SSH tutorial content, either:
1. Rephrase the skill to avoid the trigger strings
2. Don't attach the skill to the cron job — load it at runtime in the prompt instead

---

## 10. Delegation & Multi-Agent

### 10.1 When to Delegate vs Spawn

| | `delegate_task` | Spawning `hermes` process |
|-|-----------------|--------------------------|
| Isolation | Separate conversation, same process | Fully independent process |
| Duration | Minutes | Hours/days |
| Tool access | Subset of parent's tools | Full tool access |
| Context | None (pass explicitly via `context`) | Full session history |
| Use case | Quick parallel subtasks | Long autonomous missions |

**`delegate_task`** is for quick parallel subtasks within a session. The parent agent waits for results.

**Spawning via `tmux`** is for long-running autonomous work that needs to outlive the parent session.

### 10.2 delegate_task

```python
delegate_task(
    goal="Research GRPO papers, write summary to /tmp/grpo.md",
    context="Focus on DeepMind and Anthropic papers from 2024+",
    tasks=[...],  # batch mode: up to 3 in parallel
    toolsets=["web", "file"],
)
```

Subagents have **no memory** of the parent conversation. Pass all relevant context explicitly. Results return as summaries (not verified facts) — verify external side-effects yourself.

### 10.3 Spawning via tmux

For long autonomous missions:

```bash
# Start a named tmux session running hermes
docker exec hermes tmux new-session -d -s backend -x 120 -y 40 'hermes'

# Wait for startup, send first message
sleep 8 && docker exec hermes tmux send-keys -t backend 'Build a REST API for user management' Enter

# Read output
docker exec hermes tmux capture-pane -t backend -p | tail -30

# Send follow-up
docker exec hermes tmux send-keys -t backend 'Add JWT authentication middleware' Enter

# Exit cleanly
docker exec hermes tmux send-keys -t backend '/exit' Enter && sleep 2 && docker exec hermes tmux kill-session -t backend
```

### 10.4 Multi-Agent Coordination

```bash
# Agent A: backend work
docker exec hermes tmux new-session -d -s backend 'hermes -w'
sleep 8 && docker exec hermes tmux send-keys -t backend 'Build FastAPI user auth service' Enter

# Agent B: frontend work
docker exec hermes tmux new-session -d -s frontend 'hermes -w'
sleep 8 && docker exec hermes tmux send-keys -t frontend 'Build React login form for the auth API' Enter

# Relay context between them via tmux send-keys with file content
```

The `-w` flag enables worktree mode — isolated git working trees that prevent git conflicts when multiple agents work on the same repo.

### 10.5 Delegation Config

```yaml
delegation:
  model: anthropic/claude-sonnet-4      # Model for subagents
  provider: openrouter
  max_iterations: 50                  # Max turns per subagent
  max_spawn_depth: 1                   # 1 = leaf subagents can't spawn
```

> **Note:** Cron jobs typically restrict `enabled_toolsets` and do NOT include `"delegation"` by default. Even with `orchestrator_enabled: true`, cron jobs cannot spawn subagents unless delegation is explicitly added.

---

## 11. Profiles

Profiles let you run multiple isolated Hermes instances — separate configs, sessions, skills, and memory. Useful for completely different contexts (work vs personal, different clients, different capability sets).

```bash
# List all profiles
docker exec hermes /opt/hermes/.venv/bin/hermes profile list

# Create a new profile
docker exec hermes /opt/hermes/.venv/bin/hermes profile create work

# Clone an existing profile
docker exec hermes /opt/hermes/.venv/bin/hermes profile create work-dev --clone existing-profile

# Use a profile
docker exec hermes /opt/hermes/.venv/bin/hermes --profile work --tui

# Delete a profile
docker exec hermes /opt/hermes/.venv/bin/hermes profile delete old-profile

# Export/import
docker exec hermes /opt/hermes/.venv/bin/hermes profile export my-profile
docker exec hermes /opt/hermes/.venv/bin/hermes profile import my-profile.tar.gz
```

Each profile lives in `~/.hermes/profiles/<name>/` with its own `config.yaml`, `.env`, `sessions/`, `skills/`, etc.

---

## 12. MCP Servers

MCP (Model Context Protocol) lets Hermes connect to external tool servers. This is different from Hermes's native tool system — MCP servers expose tools that Hermes can call.

```bash
# List configured MCP servers
docker exec hermes /opt/hermes/.venv/bin/hermes mcp list

# Add an MCP server (SSE endpoint)
docker exec hermes /opt/hermes/.venv/bin/hermes mcp add my-server --url http://localhost:3001/sse

# Add an MCP server (stdio command)
docker exec hermes /opt/hermes/.venv/bin/hermes mcp add filesystem \
  --command npx --args -y,@modelcontextprotocol/server-filesystem,/opt/workspace

# Test connection
docker exec hermes /opt/hermes/.venv/bin/hermes mcp test my-server

# Reload MCP servers (in session)
docker exec hermes /opt/hermes/.venv/bin/hermes mcp reload
/reload-mcp   # in-session slash command
```

MCP servers are configured in `config.yaml` under `mcp:`. The WebUI has a full `/mcp` page with catalog, marketplace, and config CRUD.

---

## 13. Voice & Transcription

### 13.1 STT (Voice → Text)

Voice messages from messaging platforms are auto-transcribed. Provider priority:

1. **Local faster-whisper** — free, no API key needed: `pip install faster-whisper`
2. **Groq Whisper** — free tier: set `GROQ_API_KEY`
3. **OpenAI Whisper** — paid: set `VOICE_TOOLS_OPENAI_KEY`
4. **Mistral Voxtral** — set `MISTRAL_API_KEY`

```yaml
stt:
  enabled: true
  provider: local
  local:
    model: base   # tiny, base, small, medium, large-v3
```

### 13.2 TTS (Text → Voice)

| Provider | Free? | Env var |
|----------|-------|---------|
| Edge TTS | Yes (default) | None |
| ElevenLabs | Free tier | `ELEVENLABS_API_KEY` |
| OpenAI | Paid | `VOICE_TOOLS_OPENAI_KEY` |
| MiniMax | Paid | `MINIMAX_API_KEY` |
| Mistral | Paid | `MISTRAL_API_KEY` |
| NeuTTS | Free | `pip install neutts[all]` + `espeak-ng` |

Voice commands (in session):
- `/voice on` — voice-to-voice mode
- `/voice tts` — always convert response to audio
- `/voice off` — disable voice

---

## 14. Configuration Deep Dive

### 14.1 Key Paths

| Purpose | Host | Container |
|---------|------|-----------|
| Main config | `~/.hermes/config.yaml` | `/opt/data/config.yaml` |
| API keys / secrets | `~/.hermes/.env` | `/opt/data/.env` |
| Skills | `~/.hermes/skills/` | `/opt/data/skills/` |
| Sessions | `~/.hermes/sessions/` | `/opt/data/sessions/` |
| Logs | `~/.hermes/logs/` | `/opt/data/logs/` |
| Hermes source | `~/.hermes/hermes-agent/` | `/opt/hermes/` |
| Workspace | `~/workspace/` | `/opt/workspace/` (not in this container setup) |

### 14.2 config.yaml Sections

```yaml
model:
  default: minimax/MiniMax-M2.7     # Default model
  provider: minimax                  # Default provider
  base_url: ""                      # Custom endpoint (leave blank for standard providers)
  api_key: ""                       # API key (prefer .env)
  context_length: 1000000           # Model context window size

agent:
  max_turns: 200                    # Max tool-call rounds per session
  tool_use_enforcement: true        # Must use tools, not just answer

terminal:
  backend: local                    # local, docker, ssh, modal
  cwd: /opt/data                    # Working directory inside container
  timeout: 180                      # Command timeout in seconds

compression:
  enabled: true
  threshold: 0.50                   # Trigger when context is 50% full
  target_ratio: 0.20                # Target compressing to 20% of original

display:
  skin: default                     # CLI theme
  tool_progress: true               # Show tool execution progress
  show_reasoning: true              # Show thinking/reasoning content
  show_cost: false                  # Show token cost

stt:
  enabled: true
  provider: local
  local:
    model: base

tts:
  provider: edge                    # edge, elevenlabs, openai, minimax, mistral

memory:
  memory_enabled: true
  user_profile_enabled: true
  provider: built-in                # or honcho, mem0

delegation:
  model: anthropic/claude-sonnet-4
  provider: openrouter
  max_iterations: 50
  max_spawn_depth: 1

checkpoints:
  enabled: false                   # Enable /rollback
  max_snapshots: 50
```

### 14.3 Editing Config

```bash
# View current config
docker exec hermes /opt/hermes/.venv/bin/hermes config

# Edit in $EDITOR (interactive TTY)
docker exec -it hermes /opt/hermes/.venv/bin/hermes config edit

# Set a value directly
docker exec hermes /opt/hermes/.venv/bin/hermes config set agent.max_turns 200

# Edit on host directly
nano ~/.hermes/config.yaml
# Then restart: sg docker -c "docker compose restart gateway"
```

### 14.4 Provider Configuration

20+ providers. Set via `hermes model` (interactive) or directly in config.yaml:

```yaml
# OpenRouter (multi-provider unified access)
model:
  provider: openrouter
  api_key: sk-or-v1-...

# Anthropic direct
model:
  provider: anthropic
  api_key: sk-ant-...

# Google Gemini
model:
  provider: gemini
  api_key: AIza...

# Nous Portal (OAuth)
model:
  provider: nous
# Then: docker exec -it hermes /opt/hermes/.venv/bin/hermes login --provider nous

# Local Ollama
model:
  provider: ollama
  base_url: http://localhost:11434
  default: llama3

# DeepSeek
model:
  provider: deepseek
  api_key: sk-deepseek-...
```

Credential pooling (multiple API keys for same provider):
```bash
docker exec -it hermes /opt/hermes/.venv/bin/hermes auth add openrouter
docker exec hermes /opt/hermes/.venv/bin/hermes auth list openrouter
docker exec hermes /opt/hermes/.venv/bin/hermes auth reset openrouter  # clear exhaustion status
```

### 14.5 Environment Variables (`.env`)

```bash
# Required API keys
ANTHROPIC_API_KEY=sk-ant-...
OPENAI_API_KEY=sk-...
OPENROUTER_API_KEY=sk-or-v1-...
MINIMAX_API_KEY=...

# Messaging platforms
TELEGRAM_BOT_TOKEN=...
DISCORD_BOT_TOKEN=...
DISCORD_ALLOWED_USERS=291686310714933258  # numeric user ID, NOT username

# Voice
ELEVENLABS_API_KEY=...
GROQ_API_KEY=...

# Security
HERMES_AGENT_TIMEOUT=300  # seconds before a running agent session is considered hung

# Paths
HERMES_HOME=/opt/data      # inside container
TERMINAL_CWD=/opt/data     # working directory
```

---

## 15. Gateway & Messaging Platforms

### 15.1 Supported Platforms

Telegram, Discord, Slack, WhatsApp, Signal, Email, SMS, Matrix, Mattermost, Home Assistant, DingTalk, Feishu, WeCom, Weixin (WeChat), BlueBubbles (iMessage), API Server (webhooks), and more.

### 15.2 Discord-Specific Gotchas

**Bot is silent, no response:**
- Must enable **Message Content Intent** in Bot → Privileged Gateway Intents
- Bot must be in `allowed_channels` in `config.yaml`

**Messages silently dropped (no response, no error):**
- A long-running non-ephemeral session can hold `_running_agents` and block new Discord messages
- `_busy_input_mode: "interrupt"` silently drops messages while session is locked
- Fix: send `/stop` in the Discord thread — it releases the running state lock
- `/reset` does NOT release the lock

**"Unauthorized user" even with correct `DISCORD_ALLOWED_USERS`:**
- Value must be a **numeric Discord user ID**, not a username
- The `_resolve_legacy_usernames()` function resolves stored strings to numeric IDs at startup — if the stored value doesn't match the actual username (including case), resolution fails silently and ALL messages are rejected
- Fix: set the numeric ID directly (find it: Discord → Settings → Advanced → Developer Mode → right-click user → "Copy User ID")

### 15.3 Telegram-Specific

Multiple bot accounts supported. Configure in `config.yaml` under `telegram.accounts`. Token from `@BotFather`.

### 15.4 Gateway Logs

```bash
# Gateway logs (if running as user service)
journalctl -u hermes-gateway -f

# Agent logs
docker exec hermes /opt/hermes/.venv/bin/hermes logs
docker exec hermes /opt/hermes/.venv/bin/hermes logs errors

# Docker container logs
docker logs hermes --tail 100 -f
```

The gateway log file at `~/.hermes/logs/gateway.log` may be empty — real-time logs go to journalctl when running as a systemd service.

---

## 16. Toolsets Reference

### 16.1 Enabling/Disabling Toolsets

```bash
# Interactive TTY tool manager
docker exec -it hermes /opt/hermes/.venv/bin/hermes tools

# Enable/disable specific toolsets
docker exec hermes /opt/hermes/.venv/bin/hermes tools enable web
docker exec hermes /opt/hermes/.venv/bin/hermes tools disable browser

# Changes take effect on /reset (new session)
```

### 16.2 Toolset Contents

| Toolset | Tools included |
|---------|---------------|
| `web` | `web_search`, `web_extract` — Tavily (default), SerpAPI, DuckDuckGo |
| `browser` | `browser_navigate`, `browser_snapshot`, `browser_console` — requires browser backend |
| `terminal` | `terminal` — shell commands, process management |
| `file` | `read_file`, `write_file`, `search_files`, `patch` — file operations |
| `code_execution` | `execute_code` — sandboxed Python |
| `vision` | `vision_analyze` — image understanding |
| `image_gen` | `text_to_image` — AI image generation |
| `tts` | `text_to_speech` — voice output |
| `skills` | `skill_view`, `skill_manage`, `skills_list` — skill management |
| `memory` | `memory` — read/write persistent memory |
| `session_search` | `session_search` — search past conversations |
| `delegation` | `delegate_task` — spawn subagents |
| `cronjob` | `cronjob` — scheduled task management |
| `clarify` | `clarify` — ask user questions mid-session |
| `messaging` | `send_message` — send to Telegram, Discord, etc. |
| `search` | `search_files` — fast regex file search |
| `todo` | `todo` — in-session task tracking |
| `homeassistant` | Smart home control via Home Assistant |

### 16.3 Web Tool Provider Configuration

```yaml
web:
  tavily_api_key: tvly-...    # from app.tavily.com (NOT tvly-dev-)
  # Alternative providers:
  # serpapi_api_key: ...
  # duckduckgo_api_key: ...
```

Without a valid `tavily_api_key`, `web` toolset will fail with "No web search provider configured" even if the toolset is enabled.

---

## 17. Limitations & Gotchas

### 17.1 Container Filesystem

The workspace (`~/workspace/` on the host) is **not mounted** in the container. Scripts that cron jobs or agents run inside the container must live in:
- `/opt/data/hermes-sync/scripts/` — git-synced, read-only mount (put scripts here)
- `/opt/data/` — writable (for generated output)

Any `cd /home/sean/workspace` or reference to `~/workspace` inside the container will fail with `No such file or directory`.

### 17.2 Iteration Limits

`max_turns` (default 90) caps tool-call rounds per session. Extended debugging sessions (~300+ messages) hit this routinely. Fix:

```yaml
agent:
  max_turns: 200  # or higher
```

After patch e39bb62, Hermes warns at 80% and 90% of the limit, saves session state at 90%, and returns a human-readable summary instead of a cryptic error. Sessions that hit the limit cleanly (via the summary path) are safe to resume.

### 17.3 Context Compaction

Context compression automatically summarizes long conversations to stay within token limits. Very long sessions may lose detail from early messages. Manual trigger: `/compress`.

### 17.4 Session Isolation

`delegate_task` subagents have **no memory** of the parent conversation. Always pass context explicitly. Subagent results are **self-reports** — verify external side-effects (file writes, API calls, deployments) yourself.

### 17.5 Cron vs Regular Session

Cron job sessions are **isolated** from your regular session context. They:
- Have no access to your current conversation
- Run with a restricted toolset (unless delegation is explicitly added)
- Cannot use `clarify` (no user present to answer questions)
- May have broken `PATH` — use absolute paths in commands

### 17.6 Tool Changes Require `/reset`

Editing config.yaml mid-session to add/remove tools doesn't take effect immediately. The tool list is loaded at session start. Do `/reset` to start a new session with updated tools.

### 17.7 SSH Key Path (Container)

When SSHing from inside the container to the host:

```bash
# Correct
ssh -i /home/hermeswebui/.hermes/container_key sean@172.19.0.1

# Wrong
ssh -i /opt/data/container_key ...       # wrong path
ssh -i ... sean@localhost               # localhost port 22 not reachable
```

`172.19.0.1` is the Docker host gateway IP. The container has network access to the host at this IP.

### 17.8 Discord Token Must Be in `.env`

Discord cron delivery only works if `DISCORD_BOT_TOKEN` is set in `~/.hermes/.env`. Without it, the cron job fails with HTTP 401 and the error propagates as a misleading "empty response" message. Check `~/.hermes/logs/gateway.log` for the actual HTTP 401 error.

### 17.9 Copilot API Auth

GitHub Copilot uses a **different auth** from regular `gh auth login`. The Copilot API requires OAuth device code flow via `hermes login --provider openai-codex` or selecting GitHub Copilot in `hermes model`. Regular GitHub tokens do NOT work for Copilot.

### 17.10 OpenRouter Credits

When OpenRouter prepaid credits are exhausted (402 errors), auxiliary tasks (compression, session_search, summarization) fail silently. The agent falls back to other providers for main conversation but auxiliary tasks may not have alternates configured. Top up at https://openrouter.ai/settings/credits or configure auxiliary providers explicitly.

---

## 18. Troubleshooting

### "hermes: executable not found"

Use the full path inside the container:
```bash
/opt/hermes/.venv/bin/hermes ...
```
The binary is not on PATH inside the Docker container.

### Tool not available

```bash
# Step 1: Verify toolset is enabled
grep -A1 "^toolsets:" ~/.hermes/config.yaml

# Step 2: /reset to reload tools in a new session
```

### Web tools failing ("No web search provider configured")

Two independent checks required:
1. `web` must be in `toolsets:` list in config.yaml
2. Valid API key must be set in config.yaml (`web.tavily_api_key`)

Even with the toolset enabled, a bad/expired key causes the same error. Test: `curl -s "https://api.tavily.com/search?q=test&api_key=YOUR_KEY" | head -c 200` — 401 means the key is invalid. Get a fresh `tvly-` prefixed key from app.tavily.com.

### Iteration limit reached

Set `agent.max_turns: 200` in config.yaml, then `/reset`.

### Gateway died on SSH logout

Enable linger: `sudo loginctl enable-linger $USER`

### Container keeps restarting

```bash
docker logs hermes --tail 100
docker exec hermes /opt/hermes/.venv/bin/hermes doctor
```

### Session file permission errors

```bash
sudo chown -R $(id -u):$(id -g) ~/.hermes/sessions/
```

### TUI won't start

Must use `-it`: `docker exec -it hermes /opt/hermes/.venv/bin/hermes --tui`

### Model/provider issues

```bash
docker exec hermes /opt/hermes/.venv/bin/hermes doctor
docker exec hermes /opt/hermes/.venv/bin/hermes login --provider nous  # re-auth OAuth
```

### API calls time out inside container

`urllib.request.urlopen` respects `HTTP_PROXY`/`HTTPS_PROXY` env vars. Proxy may be inherited from host. Fix: patch the provider to use a no-proxy opener, or unset proxy env vars in the container run command.

### Docker build fails with `--chmod`

Run `DOCKER_BUILDKIT=1 docker build ...` — old Docker builder doesn't support `--chmod` on COPY.

### Changes not taking effect

- **Tools/skills:** `/reset` starts a new session
- **Config changes:** `sg docker -c "docker compose restart gateway"` for gateway; exit and relaunch for CLI
- **Code changes:** restart the process

---

## Quick Command Reference

```bash
# Chat (one-shot)
hermes chat -q "question"

# Interactive chat
hermes --tui

# Config
hermes config edit
hermes config set section.key value

# Model/provider
hermes model
hermes login --provider nous

# Tools
hermes tools list
hermes tools enable web

# Skills
hermes skills list
hermes skills install <name>

# Sessions
hermes sessions list
hermes --continue "name"

# Cron
hermes cron list
hermes cron status

# Gateway
hermes gateway status
hermes gateway setup

# Logs
hermes logs
hermes logs errors -f

# Health
hermes doctor

# Update
hermes update
```

---

*This guide covers Hermes Agent as deployed in your Docker container setup. For the WebUI guide, see [HERMES-WEBUI-GUIDE.md](./HERMES-WEBUI-GUIDE.md). For the skills guide, see [HERMES-SKILLS-GUIDE.md](./HERMES-SKILLS-GUIDE.md). For a quick-reference version of container commands, see [HERMES_CHEATSHEET.md](./HERMES_CHEATSHEET.md).*