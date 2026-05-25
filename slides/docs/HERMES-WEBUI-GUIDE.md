# Hermes WebUI — Complete User Guide

> **Version:** 2.x | **URL:** `http://localhost:9119` (when running) | **Backend:** Requires Hermes Agent gateway running
>
> Hermes WebUI is the browser-based command center for Hermes Agent. Chat, files, terminal, memory, skills, scheduled jobs, and agent management — all in one responsive web app. It can also run as a PWA (installable) and via Tailscale for remote access.

![Hermes Workspace — Chat](./public/screenshots/chat-v3.png)
![Hermes Workspace — Dashboard](./public/screenshots/dashboard-v3.png)

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
10. [Settings & Providers](#10-settings--providers)
11. [Keyboard Shortcuts](#11-keyboard-shortcuts)
12. [Themes](#12-themes)
13. [Swarm Mode](#13-swarm-mode)
14. [Mobile & PWA](#14-mobile--pwa)
15. [How It Connects to Hermes Agent](#15-how-it-connects-to-hermes-agent)
16. [Known Limitations](#16-known-limitations)

---

## 1. What Is Hermes WebUI?

Hermes WebUI (also called **Hermes Workspace** or **hermes-workspace**) is the React-based web interface that talks to your Hermes Agent gateway. It's not a wrapper around an LLM API — it's a full workspace UI that:
- Sends messages to your Hermes Agent gateway
- Renders tool calls, streaming text, and markdown in real time
- Manages sessions, skills, memory, cron jobs, files, and terminal
- Provides a built-in PTY terminal backed by Python's pty module
- Supports multi-agent Swarm mode

**WebUI vs TUI vs CLI:**

| Interface | Best for | Access |
|-----------|---------|--------|
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

The WebUI supports Tailscale for zero-config remote access. See the [desktop update docs](https://github.com/outsourc-e/hermes-workspace/blob/main/docs/desktop-update-system.md) for Tailscale integration setup.

---

## 3. The Chat Screen

**Route:** `/chat` or `/chat/$sessionKey`

The main interaction surface. Supports real-time SSE streaming, tool call rendering, multi-session management, and markdown + syntax highlighting.

### 3.1 Chat Modes

The WebUI has two backend modes (switchable in the UI):

**Enhanced Claude** — Full session API via Hermes Agent gateway. Supports persistent session history, skill loading, full tool access, and all Hermes features. **Recommended.**

**Portable** — OpenAI-compatible `/v1/chat/completions`. Works with any OpenAI-compatible endpoint (Ollama, LM Studio, vLLM, etc.). No session persistence, no skills.

### 3.2 Message Rendering

Messages render with:
- Full markdown (GFM: tables, task lists, strikethrough)
- Syntax-highlighted code blocks (Shiki)
- Collapsible tool call pills with arguments and results
- Thinking/reasoning blocks (configurable: show or hide via `/reasoning show|hide`)
- Timestamps on each message
- Copy button on code blocks and tool calls

### 3.3 Chat Composer

The input area at the bottom supports:
- **Multi-line input** — Shift+Enter for newlines, Enter to send
- **`/` slash commands** — autocomplete command menu: `/new`, `/clear`, `/model`, `/save`, `/skills`, `/skin`, `/help`, `/btw`, `/branch`
- **File attachments** — drag-and-drop or click the attachment button (images base64-encoded for multimodal models)
- **Voice input** — Web Speech API microphone button
- **Context meter** — token usage percentage bar, warns at 80% of context limit

### 3.4 Session Management

**Sidebar (left):**
- Session list with search
- Pin/unpin sessions
- Rename sessions (click title or `/title name`)
- Delete with confirmation
- Fork/branch sessions

**Session forking:** Creates a copy of the current session as a new branch. Useful for exploring alternatives without losing the original.

**Session tombtones:** Deleted sessions are marked as tombstones rather than immediately removed, allowing recovery of accidentally deleted sessions.

### 3.5 Inspector Panel

The right sidebar shows:
- **Session activity** — recent tool calls and their results
- **Memory** — what's loaded in the current session
- **Skills** — which skills are active

Toggle it with the inspector icon button.

### 3.6 Slash Command Reference

Typed in the chat input (after `/`):

| Command | What it does |
|---------|-------------|
| `/new` | Fresh session |
| `/clear` | Clear screen + new session |
| `/retry` | Resend last message |
| `/undo` | Remove last exchange |
| `/title [name]` | Name the session |
| `/compress` | Manually compress context |
| `/stop` | Kill background processes |
| `/rollback [N]` | Restore checkpoint |
| `/background <prompt>` | Run in background |
| `/queue <prompt>` | Queue for next turn |
| `/resume [name]` | Resume a named session |
| `/goal [text]` | Set a standing goal |
| `/model [name]` | Show/change model |
| `/reasoning [level]` | Set reasoning effort |
| `/voice [on\|off\|tts]` | Voice mode |
| `/yolo` | Toggle approval bypass |
| `/skin [name]` | Change theme |
| `/skill <name>` | Load skill |
| `/btw` | Ephemeral side question |
| `/branch` | Fork session |
| `/save` | Save conversation to file |
| `/image` | Attach image |
| `/usage` | Token usage stats |
| `/insights [days]` | Usage analytics |
| `/help` | Show commands |

### 3.7 Provider Selection

Click the model name in the chat header to open the **model picker dialog** inline in the chat. Choose from available providers and models. The change applies to the current session.

---

## 4. Files & Code Editor

**Route:** `/files`

Full workspace file browser with directory tree navigation, Monaco Editor integration, and file preview.

### 4.1 File Browser

- Navigate the workspace directory tree
- Create, read, rename, delete, mkdir operations
- File upload (multipart)
- Image preview (base64 rendering)
- Path traversal is sandboxed — can't escape workspace root
- Ignored directories: `node_modules`, `.git`, `.next`, `.turbo`, `.cache`, `__pycache__`, `.venv`, `dist`
- Configurable depth (default 3 levels) and entry limits (max 20,000)

### 4.2 Monaco Editor

Click any file to open it in the Monaco Editor panel:
- Full syntax highlighting for 50+ languages
- Word wrap toggle
- Minimap toggle
- Find and replace
- Font size adjustable in Settings

### 4.3 File Preview Dialog

For quick viewing without opening the editor, use the file preview dialog. Shows file content with syntax highlighting, read-only.

### 4.4 Supported File Operations

| Operation | How |
|-----------|-----|
| Read | Click file in browser or use `/api/files?action=read` |
| Write | Use Monaco Editor or write via API |
| Create | New file button in file browser |
| Rename | Right-click → Rename |
| Delete | Right-click → Delete (with confirmation) |
| Mkdir | New folder button |
| Download | `/api/files?action=download` |

---

## 5. Terminal

**Route:** `/terminal`

A full cross-platform PTY terminal backed by Python's `pty-helper.py` (no native `node-pty` dependency). Uses **xterm.js** with fit, search, and web-links addons. 256-color support (`TERM=xterm-256color`, `COLORTERM=truecolor`).

### 5.1 Features

- Create, input, resize, close shell sessions
- Real-time SSE streaming of terminal output
- Keepalive pings every 8 seconds
- Persistent shell sessions across WebUI navigation
- Platform-aware default shell: zsh on macOS, bash on Linux, PowerShell on Windows

### 5.2 Keyboard Shortcuts

| Shortcut | Action |
|----------|--------|
| `⌘K` / `Ctrl+K` | Open command palette |
| `Ctrl+L` | Clear terminal |
| `Ctrl+C` | Send interrupt (SIGINT) |
| `Ctrl+D` | Send EOF |
| `Ctrl+Z` | Suspend process (SIGTSTP) |

### 5.3 Mobile

The terminal has adapted touch input for mobile devices. A virtual keyboard with common shortcuts is shown above the terminal input on mobile.

### 5.4 Debug Panel

The terminal workspace component includes a debug panel showing session state, buffer size, and connection health.

---

## 6. Memory Browser

**Route:** `/memory`

Browse, search, and edit Hermes Agent's persistent memory files (`~/.hermes/`). Shows MEMORY.md, daily memory files, and any memories stored in `memories/`.

### 6.1 What You Can Do

- **List** all memory files sorted by date
- **Search** across all memory entries — text search with line-level results (max 200 matches)
- **Preview** memory files with markdown rendering
- **Edit** files directly with the MemoryEditor component

### 6.2 Memory Files Shown

- `MEMORY.md` — always first in the list
- `memory/YYYY-MM-DD.md` — daily logs sorted by date descending
- `memories/` — additional curated memory files
- `USER.md` — user profile

### 6.3 How Memory Works

Hermes reads these files at the start of each new session. You can edit them in the WebUI and the changes will be picked up by Hermes on its next session start. **Edits within an active session require `/reset` to take effect** (starts a new session with updated memory).

### 6.4 Path

Files are stored in `~/.hermes/` on the host (mapped to `/opt/data/` inside the container). The WebUI accesses them via the Hermes Agent gateway's memory API.

---

## 7. Skills Browser

**Route:** `/skills`

Browse and manage the 2,000+ skill registry. Three tabs:

| Tab | What it shows |
|-----|---------------|
| **Installed** | Skills currently installed in your `~/.hermes/skills/` |
| **Marketplace** | All available skills in the catalog |
| **Featured** | Curated skill groups (Most Popular, New This Week, Developer Tools, Productivity) |

### 7.1 Categories

All, Web & Frontend, Coding Agents, Git & GitHub, DevOps & Cloud, Browser & Automation, Image & Video, Search & Research, AI & LLMs, Productivity, Marketing & Sales, Communication, Data & Analytics, Finance & Crypto.

### 7.2 Search & Filter

Filter skills by name, description, author, tags, and triggers. Sort by name or category. Full-text search with category badges and origin indicators.

### 7.3 Security Risk Display

Each skill shows a risk level: **safe**, **low**, **medium**, **high**, with specific flags and scores. Review before installing high-risk skills — they may execute arbitrary code.

### 7.4 Installing Skills

1. Browse or search for a skill
2. Click **Install** — downloads to `~/.hermes/skills/`
3. Load in session: `/skill skill-name` or pre-load with `hermes -s skill-name`

### 7.5 Workspace Skills

The skills browser also has a **workspace skills screen** for per-session skill management — configure which skills load for which context.

---

## 8. Jobs (Cron)

**Route:** `/jobs`

Manage scheduled autonomous tasks. Create, edit, pause, resume, trigger, and view output for all cron jobs.

### 8.1 Job States

Each job shows:
- **Enabled/Disabled** state
- **Next run** and **last run** timestamps
- **Last status** — ok, error, or running
- **Output** — click to view full execution results

### 8.2 Creating a Job

Click **New Job** to open the create dialog:
- **Name** — descriptive label
- **Schedule** — cron expression or natural language (`every 30m`, `0 9 * * MON-FRI`)
- **Prompt** — the task description (self-contained — include all context needed)
- **Skills** — attach skills to load before execution
- **Delivery** — where to send output (`origin`, `local`, `all`, or platform:chat_id)
- **Repeat** — one-shot or recurring

### 8.3 Delivery Gotcha

**`deliver: local` saves output to the filesystem — you never see it.** `deliver: discord` does NOT work via the delivery field — the scheduler has no Discord delivery code path. For reliable Discord delivery, include `send_message` tool calls in the job prompt itself.

**Always test delivery immediately** by clicking **Run Now** after creating the job. `last_status: ok` does NOT mean output reached you.

### 8.4 Viewing Output

Click any job to expand its output view. Shows the full execution transcript, any error messages, and delivery status. Useful for auditing what the cron job actually did.

---

## 9. Dashboard

**Route:** `/dashboard`

Aggregated overview showing:
- **Sessions** — active sessions, session count, recent activity
- **Model mix** — which models you've been using
- **Cost ledger** — token usage across providers
- **Attention card** — what the agent is currently focused on
- **Ops strip** — quick actions and status indicators

The dashboard overflow panel provides expanded views for deeper inspection.

---

## 10. Settings & Providers

**Route:** `/settings` and `/settings/providers`

### 10.1 Settings

Centralized configuration panel for:
- **Theme** — dark/light mode, accent color
- **Editor** — font size, word wrap, minimap
- **Notifications** — enable/disable sounds
- **Usage threshold** — when to show context warnings
- **Smart suggestions** — model recommendations
- **Mobile nav** — dock, integrated, or scroll-hide modes

### 10.2 Providers Screen

Manage AI provider connections. Each provider shows:
- Auth type (API key, OAuth, etc.)
- Connection status
- Masked API key display
- Quick actions (re-auth, test, remove)

**Supported providers:**
| Provider | Auth | Notes |
|----------|------|-------|
| Anthropic | API key, CLI Token | Claude Haiku/Sonnet/Opus |
| OpenAI | API key | GPT models |
| Google | API key, OAuth | Gemini models |
| OpenRouter | API key | Multi-provider unified |
| Nous Portal | OAuth device code | Main provider in this setup |
| Ollama | Local (no auth) | Local models |
| MiniMax | API key | Main model in this setup |
| Custom | API key | Any OpenAI-compatible endpoint |

### 10.3 Provider Wizard

Guided step-by-step setup for new providers. Select provider → authenticate → test → save. Handles OAuth flows, API key input, and connection verification.

### 10.4 Config Management

The WebUI can read and write `~/.hermes/config.yaml` and `~/.hermes/.env` directly through the Settings UI. Changes take effect on the next session (or restart the gateway for immediate effect).

---

## 11. Keyboard Shortcuts

| Shortcut | Action |
|----------|--------|
| `⌘K` / `Ctrl+K` | Open command palette |
| `Ctrl+L` | Clear chat |
| `Escape` | Close modal/dialog |
| `Shift+Enter` | New line in composer |
| `Enter` | Send message |
| `/` | Open slash command menu |

### 11.1 Terminal Shortcuts

| Shortcut | Action |
|----------|--------|
| `Ctrl+C` | Interrupt |
| `Ctrl+D` | EOF |
| `Ctrl+Z` | Suspend |
| `Ctrl+L` | Clear |

### 11.2 Command Palette

Press `⌘K` anywhere in the app to open the command palette. Search and execute any command: navigation, settings, session management, skills.

---

## 12. Themes

### 12.1 Available Themes

Hermes WebUI ships with **8 built-in themes** (4 dark + 4 light variants):

| Theme | Style |
|-------|-------|
| **Claude Official** | Navy and indigo flagship |
| **Claude Official Light** | Soft indigo light palette |
| **Claude Classic** | Bronze accents on dark charcoal |
| **Classic Light** | Warm parchment with bronze |
| **Slate** | Cool blue developer theme |
| **Slate Light** | GitHub-light with blue accents |
| **Mono** | Clean monochrome grayscale |
| **Mono Light** | Bright monochrome grayscale |

Plus **SciFi theme** — a full dark/light palette remap with different visual identity.

### 12.2 Switching Themes

**In chat:** `/skin slate` (or any theme name)
**In Settings:** Theme picker in the Settings panel
**CSS Variables:** Theme tokens use CSS custom properties, so themes can be overridden via browser dev tools.

### 12.3 System Theme

Set `theme: system` in settings to follow your OS light/dark preference automatically.

---

## 13. Swarm Mode

**Route:** See `/swarm` routes

Swarm mode turns the WebUI into a **live multi-agent control plane**. It's an advanced feature for running multiple autonomous Hermes Agents simultaneously with role-based dispatch.

### 13.1 How It Works

- Unlimited Hermes Agent workers with **1 orchestrator**
- Workers are **persistent tmux sessions** that maintain context across tasks
- Roles: builders, reviewers, docs, research, ops, triage, QA, lab
- Role-based dispatch routes tasks to the right worker without manual assignment
- **Byte-verified review gate** — checkpoints verified before handoff
- Autonomous PR/issue lanes, lab experiments, repair playbook

### 13.2 What's in the Swarm UI

- **Orchestrator Chat** — ask the control plane for a task, decomposed mission, or full broadcast
- **Live Agent Panel** — see all workers, their roles, state, and runtime status
- **Kanban TaskBoard** — backlog → ready → running → review → blocked → done lanes
- **Reports + Inbox** — review checkpoints, blockers, handoffs, decisions needing human judgment
- **TUI View** — attach to tmux-backed workers or fall back to live shell/log stream

### 13.3 Starting Swarm

See `docs/swarm/` in the hermes-workspace repo. Swarm mode requires tmux on the host and the Hermes Agent gateway running.

### 13.4 Conductor (Advanced)

Conductor is a mission dispatch + decomposition feature. It requires the upstream dashboard plugin (not yet in upstream Hermes Agent — see [issue #262](https://github.com/outsourc-e/hermes-workspace/issues/262)). When the plugin isn't available, the UI shows a clear placeholder instead of failing.

---

## 14. Mobile & PWA

### 14.1 Mobile Layout

Hermes WebUI is responsive. On mobile:
- Bottom tab bar for navigation (`dock` mode) or integrated navigation (`integrated` mode)
- Hamburger menu for secondary navigation
- Mobile-optimized chat composer
- Mobile terminal with touch-adapted input
- Mobile session panel

### 14.2 PWA Installation

Hermes Workspace can be installed as a Progressive Web App:
1. Open in Chrome/Safari on mobile
2. "Add to Home Screen" / "Install App"
3. Runs as a native-feeling app with offline caching

### 14.3 Tailscale Access

With Tailscale integration configured, you can access the WebUI from any device on your tailnet — no port forwarding needed. See `docs/desktop-update-system.md` for setup.

---

## 15. How It Connects to Hermes Agent

```
Browser (WebUI) ←→ Hermes Agent Gateway (port 8642) ←→ Hermes Agent
                      ↓
              SQLite (sessions, state)
              Skills, memory files
              Cron scheduler
```

The WebUI is a **client** to the Hermes Agent gateway, not a separate agent. All chat, file, terminal, and memory operations go through the gateway.

### 15.1 Two Backend Modes

**Enhanced Claude mode** — WebUI sends messages to the Hermes Agent gateway (`/api/send-stream`). The gateway routes to the AIAgent. Sessions are stored in SQLite via Hermes Agent. Full features.

**Portable mode** — WebUI sends directly to an OpenAI-compatible endpoint (`/v1/chat/completions`). No session persistence, no skill loading, no Hermes features. Connects to Ollama, LM Studio, etc.

### 15.2 WebUI Auto-Start

The WebUI can auto-detect a sibling `hermes-agent/` directory and spawn the gateway. It reads the Hermes Agent Python venv configuration and starts uvicorn with health polling.

### 15.3 Key API Endpoints

| Endpoint | What it does |
|----------|-------------|
| `POST /api/send-stream` | Main streaming chat (enhanced Claude mode) |
| `POST /api/send` | Non-streaming chat send |
| `GET /api/chat-events` | SSE event stream |
| `GET/POST/PATCH/DELETE /api/sessions` | Session CRUD |
| `GET/POST /api/files` | File operations |
| `GET/POST/PATCH/DELETE /api/claude-jobs` | Cron job management |
| `POST /api/terminal-stream` | PTY session creation |
| `GET /api/memory` | Read agent memory |
| `GET /api/skills` | List skills |
| `GET /api/claude-config` | Read config.yaml + .env |
| `PATCH /api/claude-config` | Write config |
| `GET /api/gateway-status` | Gateway capabilities |
| `POST /api/start-claude` | Auto-start gateway |

---

## 16. Known Limitations

### 16.1 Conductor Requires Upstream Plugin

Conductor (mission dispatch + decomposition) needs a dashboard plugin not yet in the Hermes Agent upstream. The UI shows a clear placeholder when that endpoint isn't available ([#262](https://github.com/outsourc-e/hermes-workspace/issues/262)). All other features work without it.

### 16.2 Session Storage Location

Sessions are stored by Hermes Agent in `~/.hermes/sessions/` on the host (`/opt/data/sessions/` in container). The WebUI reads from this directory via the gateway. If permissions are wrong (files owned by `root`), sessions won't load in the WebUI.

### 16.3 WebUI Itself Has No Authentication Backend

The WebUI relies on Hermes Agent gateway's auth. If no `CLAUDE_PASSWORD` is set, the WebUI is open to anyone who can reach it. **Always run behind a reverse proxy with auth** if exposing remotely.

### 16.4 Tool Reload Requires New Session

Changes to enabled toolsets in `config.yaml` don't take effect mid-session. `/reset` starts a new session with the updated tool list.

### 16.5 Memory Edits Require New Session

Editing memory files in the WebUI Memory browser doesn't automatically reload them into an active session. `/reset` starts a new session that picks up the changes.

### 16.6 Portable Mode Limitations

When using Portable mode (OpenAI-compatible API):
- No session persistence — each page load is a fresh context
- No skills loading
- No memory persistence
- No cron jobs
- Full tool access depends on the backend server

### 16.7 Hermes World (3D Environment)

The 3D "Hermes World" environment is an experimental feature documented in `docs/hermesworld/`. It uses Three.js + React Three Fiber + Rapier physics. Not covered in this guide — see the dedicated docs in the workspace repository.

---

*For the Hermes Agent backend guide, see [HERMES-AGENT-GUIDE.md](./HERMES-AGENT-GUIDE.md). For the skills guide, see [HERMES-SKILLS-GUIDE.md](./HERMES-SKILLS-GUIDE.md). For container quick reference, see [HERMES_CHEATSHEET.md](./HERMES_CHEATSHEET.md).*