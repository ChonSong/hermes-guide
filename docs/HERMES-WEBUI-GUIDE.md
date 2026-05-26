# Hermes WebUI — Complete User Guide

> **Version:** 2.x | **URL:** `http://localhost:9119` (when running) | **Backend:** Requires Hermes Agent gateway running
> **Docs:** [hermes-agent.nousresearch.com/docs](https://hermes-agent.nousresearch.com/docs/)

Hermes WebUI is the browser-based command center for Hermes Agent. Chat, files, terminal, memory, skills, scheduled jobs, and agent management — all in one responsive web app. It can also run as a PWA (installable) and via Tailscale for remote access.

---

## Table of Contents

1. [What Is Hermes WebUI?](#1-what-is-hermes-webui)
2. [Getting Connected](#2-getting-connected)
3. [The Chat Screen](#3-the-chat-screen)
4. [Files & Code Editor](#4-files--code-editor)
5. [Terminal](#5-terminal)
6. [Memory Browser](#6-memory-browser)
7. [Skills Browser](#7-skills-browser)
8. [Jobs (Cron)](#8-jobs-cron)
9. [Dashboard](#9-dashboard)
10. [Swarm Mode](#10-swarm-mode)
11. [Keyboard Shortcuts](#11-keyboard-shortcuts)
12. [Themes](#12-themes)
13. [Mobile & PWA](#13-mobile--pwa)
14. [How It Connects to Hermes Agent](#14-how-it-connects-to-hermes-agent)
15. [Known Limitations](#15-known-limitations)

---

## 1. What Is Hermes WebUI?

Hermes WebUI (also called **Hermes Workspace** or **hermes-workspace**) is the React-based web interface that talks to your Hermes Agent gateway. It's not a wrapper around an LLM API — it's a full workspace UI that:

- Sends messages to your Hermes Agent gateway
- Renders tool calls, streaming text, and markdown in real time
- Manages sessions, skills, memory, cron jobs, files, and terminal
- Provides a built-in PTY terminal backed by Python's pty module
- Supports multi-agent Swarm mode

### WebUI vs TUI vs CLI vs Messaging

| Interface | Best for | Access |
|-----------|----------|--------|
| **WebUI** (this) | Daily driving, mobile access, visual session management | Browser at `:9119` |
| **TUI** (Ink/React terminal) | Rich CLI experience, when you have SSH access | `hermes --tui` in container |
| **CLI** (one-shot) | Scripts, quick questions | `hermes chat -q "..."` |
| **Messaging bots** | On-the-go, async | Telegram/Discord |

The WebUI is the recommended interface for most daily use.

---

## 2. Getting Connected

### 2.1 Starting the WebUI

```bash
# Via Docker Compose (already configured in your setup)
cd ~/.hermes/hermes-agent
sg docker -c "docker compose up -d"

# The WebUI runs at http://127.0.0.1:9119
```

### 2.2 Connection Status

On first load, Hermes WebUI probes the Hermes Agent gateway at `http://127.0.0.1:8642` (default). If the gateway isn't found, you'll see a **connection overlay** prompting you to start it.

**Two-tier capability model:**
- **Core** (always needed): health check, chat completions, models list, streaming
- **Enhanced** (optional): sessions, skills, memory, config, jobs

If only core is available, some UI areas (Jobs, Memory browser) show graceful degradation placeholders instead of errors.

### 2.3 Authentication

If you've set a password via `CLAUDE_PASSWORD` in the environment, you'll see a login screen on first load. The password is checked against a bcrypt hash server-side.

### 2.4 Accessing Remotely via Tailscale

The WebUI supports Tailscale for zero-config remote access. See the [desktop update docs](https://github.com/outsourc-e/hermes-workspace/blob/main/docs/desktop-update-system.md) for Tailscale integration setup. With Tailscale, you can access the WebUI from anywhere without port forwarding.

---

## 3. The Chat Screen

**Route:** `/chat` or `/chat/$sessionKey`

The main interaction surface. Supports real-time SSE streaming, tool call rendering, multi-session management, and markdown + syntax highlighting.

### 3.1 Chat Modes

The WebUI has two backend modes (switchable in the UI):

**Enhanced Claude** — Full session API via Hermes Agent gateway. Supports persistent session history, skill loading, full tool access, and all Hermes features. **Recommended.**

**Portable** — OpenAI-compatible `/v1/chat/completions`. Works with any OpenAI-compatible endpoint (Ollama, LM Studio, vLLM, etc.). No session persistence, no skills.

### 3.2 Chat Features

- **Real-time streaming** — text appears as it's generated, no waiting for complete response
- **Tool call rendering** — see each tool execution as it happens with timing info
- **Markdown rendering** — code blocks, tables, LaTeX math all render properly
- **Session branching** — fork any session to experiment without losing the original
- **Context indicator** — shows current session ID and model in use

### 3.3 Slash Commands (in chat)

Type these directly in the chat input:

| Command | Action |
|---------|--------|
| `/new` | Fresh session |
| `/skill name` | Load a skill |
| `/goal text` | Set a persistent goal |
| `/fork` | Branch session |
| `/resume name` | Resume a session |
| `/status` | Show session info |
| `/verbose` | Cycle verbose modes |
| `/yolo` | Toggle approval bypass |

---

## 4. Files & Code Editor

**Route:** `/workspace` or `/files`

A file browser and code editor integrated into the WebUI. Navigate the container filesystem, view and edit code, and see changes in real time.

### Features

- **Directory tree** — navigate `/opt/data` (container's `~/.hermes`) and mounted volumes
- **Syntax highlighting** — supports all major languages (Python, JS, TypeScript, Go, Rust, etc.)
- **Double-click to rename** — rename files and folders directly in the UI
- **Breadcrumb navigation** — always know where you are in the tree

### Screenshots Available

The WebUI ships with annotated screenshots showing:
- `ui-workspace.png` — the full workspace layout
- `ui-sessions.png` — the session management panel

---

## 5. Terminal

**Route:** `/terminal` or embedded in the workspace

Built-in PTY terminal backed by Python's pty module. Run shell commands, interact with processes, and use familiar tools (vim, nano, tmux) — all without leaving the browser.

### Using the Terminal

- Opens at `/opt/data` by default (container's `~/.hermes`)
- Full PTY support — interactive commands work (top, htop, vim)
- Scrollback buffer — previous output preserved
- Copy/paste support

### Key Commands

```bash
cd /opt/data                           # Navigate to hermes home
ls -la                                 # List files
hermes config edit                     # Edit config
gh auth status                         # Check GitHub auth
htop                                   # Monitor processes
tmux ls                                # List tmux sessions
```

---

## 6. Memory Browser

**Route:** `/memory`

Browse, search, and edit Hermes's memory files directly.

### What You Can Do

- **View SOUL.md** — the agent's personality and core principles
- **View/edit MEMORY.md** — curated long-term facts about you and your setup
- **Browse daily logs** — `memory/YYYY-MM-DD.md` files from each session
- **Search** — find anything across all memory files

### Key Memory Files

| File | Purpose |
|------|---------|
| `SOUL.md` | Agent identity and personality |
| `MEMORY.md` | Curated long-term knowledge |
| `memory/YYYY-MM-DD.md` | Daily session logs |

### Updating MEMORY.md

The agent updates MEMORY.md after significant events. You can also edit it manually — the agent reads it fresh on each main session start. Good things to put there:
- Preferences (concise output, preferred workflows)
- Environment details (custom paths, specific configs)
- Lessons learned (don't do X, always use Y instead)

---

## 7. Skills Browser

**Route:** `/skills` or `/skills/$skillName`

Browse, install, and manage all 146 available skills.

### Three Tabs

1. **Installed** — skills currently active in your environment
2. **Marketplace** — all 146 catalog skills with descriptions
3. **Featured** — curated skill groups by category

### Filtering & Search

- **Search bar** — filter by name, description, or tags in real time
- **Category filter** — narrow down by devops, agents, creative, mlops, etc.
- **Security risk badges** — each skill shows a risk level (review high-risk before install)

### Loading Skills

- **Auto-match** — the skill-selector scores skills against your message every turn and loads relevant ones silently (score ≥3.0) or with notification (score 1.5–3.0)
- **Explicit load** — `/skill github-pr-workflow` in chat
- **Startup preload** — `hermes -s github-auth,coder` to preload at launch

### Core Skills to Know

| Skill | Purpose |
|-------|---------|
| `hermes-agent` | Complete CLI/config reference — load when anything isn't working |
| `autonomous-cron-pipeline` | Multi-phase projects via chained cron jobs |
| `github-pr-workflow` | Full PR lifecycle |
| `morning-briefing` | Daily news + weather + calendar |
| `coder` + `reviewer` | Code generation and QA — use together |
| `repo-transmute` | Vision-driven codebase migration |

---

## 8. Jobs (Cron)

**Route:** `/jobs`

Create, monitor, pause, and resume cron jobs. Each job has its own output viewer so you can see what it produced last time it ran.

### Creating a Job

1. Click **New Job** or the **+** button
2. Set the **schedule** (30m, hourly, daily, or cron syntax)
3. Write the **prompt** (what the job should do)
4. Attach **skills** (optional but recommended)
5. Set **delivery** (where to send output)
6. Save

### Job States

| State | Meaning |
|-------|---------|
| `active` | Running on schedule |
| `paused` | Temporarily disabled |
| `disabled` | Manually turned off |
| `ok` | Last run succeeded |
| `error` | Last run failed |

### Monitoring

- **Last run** — timestamp of most recent execution
- **Output** — what the job produced (viewable in the output panel)
- **Error log** — if it failed, see why
- **Pause/resume** — control without deleting

---

## 9. Dashboard

**Route:** `/dashboard` or `/insights`

Session count, model usage mix, token cost ledger, and system health card.

### What's Tracked

- **Sessions** — total active sessions, session history over time
- **Model mix** — which models you've used and how much
- **Token usage** — daily/weekly/monthly token counts
- **Cost ledger** — estimated spend per provider
- **System health** — component status (gateway, tools, memory)

### Screenshots Available

- `insights-daily-tokens-models.png` — token usage breakdown by model
- `system-health-panel.png` — system health card with component status

---

## 10. Swarm Mode

**Route:** `/swarm` (when enabled)

Multi-agent control plane for orchestrating unlimited Hermes workers.

### How It Works

- **1 Orchestrator** — your primary Hermes session that plans and dispatches
- **N Workers** — persistent tmux sessions running Hermes agents with specific roles
- **Role-based dispatch** — builders, reviewers, QA, ops — each worker has a specialty
- **Kanban board** — backlog → in review → done lanes
- **Byte-verified review gates** — each handoff is verified correct before the next agent takes over

### Best For

- Complex multi-phase projects (architecture work, large refactors)
- Parallel development (backend + frontend simultaneously)
- Automated code review pipelines
- Multi-repository management

---

## 11. Keyboard Shortcuts

| Key | Action |
|-----|--------|
| `Space` | Next slide |
| `Shift+Space` | Previous slide |
| `←` `→` | Navigate slides (when not editing) |
| `F` | Toggle fullscreen (hides all UI chrome) |
| `E` | Toggle edit mode (inline text editing) |
| `Esc` | Exit fullscreen |
| `Ctrl+Z` | Undo last edit |
| Swipe | Navigate on touch devices |

### Editable Content

Click any text element (title, body, subtitle) to edit it inline. Changes auto-save to localStorage. Use the **↺ Reset** button to restore default slides.

---

## 12. Themes

The WebUI ships with multiple color schemes:

### Current Default: Solid Dark (#191919)

All regions within ΔE<8 of the reference values:

| Region | Reference Color |
|--------|----------------|
| Top bar | `#131313` |
| Left sidebar | `#191919` |
| Center content | `#1a1a1a` |
| Right panel | `#1d1d1d` |
| Dock | `#191919` |

### Previous: Glassmorphism

Translucent panels with backdrop blur (removed in current version for performance).

### Changing Themes

Settings → Display → Skin. The `THEMES.md` in your workspace covers the full theme system with ΔE color difference measurements.

---

## 13. Mobile & PWA

### PWA Installation

The WebUI is a Progressive Web App. On mobile:
1. Open the WebUI in your browser
2. Tap **Share** → **Add to Home Screen**
3. Opens as a standalone app without browser chrome

### Mobile Features

- **Responsive layout** — adapts to phone screens
- **Touch swipe** — swipe left/right to navigate slides
- **Fullscreen** — tap the fullscreen button for distraction-free reading

### Remote Access via Tailscale

No port forwarding needed. Install Tailscale on both ends, connect to the same tailnet, and access the WebUI at its Tailscale address from anywhere.

---

## 14. How It Connects to Hermes Agent

```
┌────────────────┐       HTTP/WebSocket       ┌──────────────────┐
│   Your Browser  │ ◄─────────────────────────► │  Hermes Gateway  │
│   (WebUI :9119) │        SSE Streaming         │   (:8642)        │
└────────────────┘                             └────────┬─────────┘
                                                         │
                                                 ┌────────┴─────────┐
                                                 │  Hermes Agent     │
                                                 │  (Docker :8642)   │
                                                 │  + Tool Executor  │
                                                 │  + LLM Provider   │
                                                 └────────────────────┘
```

- **WebUI on :9119** — the React frontend (static files)
- **Gateway on :8642** — the FastAPI backend that WebUI connects to
- **Agent** — handles the actual tool execution and LLM calls

---

## 15. Known Limitations

### Session Branching Behavior

- Branching while a session is actively running may not preserve all state
- Long-running sessions can hold locks that block new messages (use `/stop` to release)

### Tool Changes Require /reset

Editing config.yaml or enabling/disabling tools mid-session does not reload them. You must `/reset` for changes to take effect.

### Memory Not in Cron Contexts

`MEMORY.md` is only loaded in main (interactive) sessions. Cron jobs run in a different context and do not have access to your curated memory. Skills attached to cron jobs also run in this limited context.

### Portal Mode (OpenClaw legacy)

Some UI elements reference "Portal mode" — this is legacy terminology from the OpenClaw migration. Functionality is the same under the new naming.

---

## Screenshots Reference

Real screenshots from the Hermes WebUI repository (`/home/sean/.hermes/hermes-webui/docs/`):

| Screenshot | Shows |
|------------|-------|
| `ui-workspace.png` | Full workspace layout with chat, files, terminal |
| `ui-sessions.png` | Session management panel |
| `workspace-rename.png` | Double-click rename in file browser |
| `goal-status.png` | Goal evaluation status panel |
| `system-health.png` | System health panel with component status |
| `sidebar-fix.png` | Session sidebar behavior fix |
| `insights-tokens.png` | Token usage by model over time |
| `dashboard-nav.png` | Dashboard navigation link |
| `compression-toast.png` | Context compression toast notification |
| `claude-code-import.png` | Claude Code import (readonly) |

Access them in the WebUI docs at `~/.hermes/hermes-webui/docs/pr-media/` or view them in the slideshow at `https://chonsong.github.io/hermes-guide/`.