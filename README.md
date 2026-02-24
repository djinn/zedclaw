# zedclaw

An AI agent that runs as a desktop command-line tool — written in Zig, containerised with Podman.

Talk to it in plain language. It remembers things, runs scheduled tasks, and lets you build custom tools — all from your terminal or via Telegram.

```
> remind me every weekday at 9am to review open pull requests
Created daily schedule 1 (at 09:00): review open pull requests

> what do I have scheduled
[{"id":1,"type":"daily","time":"09:00","action":"review open pull requests","enabled":true,"timezone":"America/New_York"}]

> remember that the staging server is at staging.example.com
Stored: u_staging_server = staging.example.com
```

---

## Contents

- [Quick Start](#quick-start)
- [Configuration](#configuration)
- [Running with Podman](#running-with-podman)
- [Security Model](#security-model)
- [Features](#features)
- [Tools Reference](#tools-reference)
- [Building from Source](#building-from-source)
- [Changelog](#changelog)
- [License](#license)

---

## Quick Start

### Native (requires Zig 0.13.0+)

```bash
git clone https://github.com/tnm/zedclaw
cd zedclaw

cp .env.example .env
# edit .env — set ZEDCLAW_LLM_BACKEND and ZEDCLAW_LLM_API_KEY

zig build run
```

### Podman (recommended — no Zig install required)

```bash
git clone https://github.com/tnm/zedclaw
cd zedclaw

cp .env.example .env
# edit .env — set ZEDCLAW_LLM_BACKEND and ZEDCLAW_LLM_API_KEY

podman build -t zedclaw .
podman run -it --rm \
  --env-file .env \
  -v ./data:/data:z \
  zedclaw
```

Or with Compose:

```bash
podman-compose run zedclaw
```

---

## Configuration

All configuration is via environment variables. Copy `.env.example` to `.env` and fill in your values. The `.env` file is loaded automatically from the current directory at startup — it is never committed to git.

### Required

| Variable | Description |
|---|---|
| `ZEDCLAW_LLM_BACKEND` | `anthropic` \| `openai` \| `openrouter` \| `ollama` |
| `ZEDCLAW_LLM_API_KEY` | API key for the selected backend (not required for Ollama) |

### Optional

| Variable | Default | Description |
|---|---|---|
| `ZEDCLAW_LLM_MODEL` | backend default | Override model name |
| `ZEDCLAW_LLM_API_URL` | backend default | Override API endpoint (useful for Ollama) |
| `ZEDCLAW_TELEGRAM_TOKEN` | _(none)_ | Enable Telegram bot integration |
| `ZEDCLAW_DATA_DIR` | `~/.config/zedclaw` | Where to persist memories, schedules, tools |
| `ZEDCLAW_TIMEZONE` | `UTC` | Timezone for `get_time` and daily schedules |

### Backend defaults

| Backend | Default model | Default endpoint |
|---|---|---|
| `anthropic` | `claude-opus-4-5` | `https://api.anthropic.com/v1/messages` |
| `openai` | `gpt-4o` | `https://api.openai.com/v1/chat/completions` |
| `openrouter` | `anthropic/claude-opus-4-5` | `https://openrouter.ai/api/v1/chat/completions` |
| `ollama` | `qwen3:8b` | `http://127.0.0.1:11434/v1/chat/completions` |

---

## Running with Podman

Podman is the recommended way to run zedclaw on both macOS and Linux. It provides strong process isolation, runs rootless by default, and maps cleanly to OCI container standards without requiring a daemon.

### Build the image

```bash
podman build -t zedclaw .
```

The `Containerfile` uses a multi-stage build:

1. **Builder stage** — downloads the Zig toolchain, compiles a `ReleaseSafe` binary, and strips it.
2. **Runtime stage** — copies only the final binary into a minimal Debian slim image with CA certificates. No compiler, no source, no build tools.

### Interactive use (stdin/stdout)

```bash
podman run -it --rm \
  --env-file .env \
  -v ./data:/data:z \
  zedclaw
```

Flag notes:
- `-it` — allocates a TTY and keeps stdin attached for interactive chat
- `--rm` — removes the container after exit (state persists in the volume)
- `--env-file .env` — injects your secrets as environment variables, not baked into the image
- `-v ./data:/data:z` — mounts the data directory; `:z` applies the correct SELinux label on Linux
- No ports exposed — zedclaw makes outbound HTTPS connections only

### Telegram / background mode

If `ZEDCLAW_TELEGRAM_TOKEN` is set, the Telegram polling thread starts automatically. You can run headlessly:

```bash
podman run -d \
  --name zedclaw \
  --restart unless-stopped \
  --env-file .env \
  -v ./data:/data:z \
  zedclaw
```

### Using Compose

```bash
podman-compose run zedclaw    # interactive
podman-compose up -d zedclaw  # background (Telegram mode)
podman-compose logs -f zedclaw
```

### Connecting to a local Ollama instance

When Ollama runs on the host, the container needs to reach `host.docker.internal` (macOS/Windows) or use `--network host` (Linux):

```bash
# macOS / Windows
ZEDCLAW_LLM_API_URL=http://host.docker.internal:11434/v1/chat/completions

# Linux
podman run -it --rm \
  --network host \
  --env-file .env \
  -v ./data:/data:z \
  zedclaw
```

---

## Security Model

zedclaw handles API keys, persistent memories, and optional Telegram credentials. This section explains exactly how each is protected.

### Secrets: never in the image

API keys are passed at runtime via `--env-file` or `-e` flags. They are:

- Not written into the `Containerfile` or any source file
- Not present in the built image layer
- Not logged at any log level
- Redacted in all log output for known sensitive keys (`llm_api_key`, `telegram_token`, etc.)

The `.env` file itself is listed in `.gitignore` and should never be committed.

### Data at rest

Persistent state (memories, schedules, user tools) is stored as JSON files in `ZEDCLAW_DATA_DIR` (default `~/.config/zedclaw`). Each namespace is a separate file:

```
~/.config/zedclaw/
  zclaw.json         # user memories and rate limit counters
  zc_cron.json       # scheduled tasks
  zc_tools.json      # user-defined custom tools
  zc_config.json     # runtime config persisted by tools
```

Files are written atomically via a temp-file-and-rename pattern to prevent partial writes on crash.

**At-rest encryption** is not built in — if you need encrypted storage, use a LUKS volume (Linux) or encrypted APFS (macOS) as the underlying store, then mount it to `ZEDCLAW_DATA_DIR`.

### Container isolation

When run with Podman:

- **Rootless** — the container process runs as `uid=zedclaw` (non-root) inside an unprivileged user namespace by default with Podman
- **Read-only root filesystem** — the image contains only the binary and CA certs; nothing writable outside the mounted volume
- **No capabilities** — no `CAP_NET_ADMIN`, `CAP_SYS_ADMIN`, or other elevated capabilities are requested
- **No device access** — no `/dev` devices are mounted (unlike the original ESP32 firmware which required serial port access)
- **Outbound only** — no `EXPOSE` directives; the container makes outbound HTTPS calls to LLM APIs and Telegram only
- **SELinux/AppArmor** — the `:z` volume flag on Linux applies the correct SELinux context to the data volume

### Network trust

All LLM API calls use HTTPS with the system CA bundle (`ca-certificates` package in the runtime image). The TLS implementation is Zig's built-in `std.crypto.tls` — no OpenSSL fork dependency.

Ollama connections over `http://` (unencrypted) are only used when `ZEDCLAW_LLM_API_URL` is explicitly set to a local address. Remote Ollama endpoints should use a TLS-terminating proxy.

### Tool safety

User-defined tools (`create_tool`) store an **action string** — a plain-language instruction that the agent executes on the next invocation. No shell commands are run; the action is sent back to the LLM as a task description, which then uses built-in tools (memory, cron) to carry it out.

Built-in tools that write persistent state (`memory_set`, `cron_set`, `create_tool`) validate inputs and enforce key naming rules (`u_` prefix for user memory keys) before writing.

### Rate limiting

Built-in sliding-window rate limits protect against runaway LLM spend:

| Limit | Default |
|---|---|
| Requests per hour | 100 |
| Requests per day | 1000 |

Counters persist across restarts in the data directory. The agent returns a clear error message when a limit is reached rather than silently dropping messages.

### Telegram access control

When Telegram is enabled, all messages received by the bot are forwarded to the agent. You can restrict which chat IDs the agent responds to by using Telegram's bot privacy settings or by adding a chat ID allowlist to your deployment configuration.

---

## Features

- **Multi-backend LLM** — Anthropic, OpenAI, OpenRouter, Ollama with automatic format switching (Anthropic vs OpenAI-compatible API)
- **Persistent memory** — key-value store that survives restarts; agent uses it proactively via `memory_set`/`memory_get`/`memory_list`
- **Scheduler** — `periodic` (every N minutes), `daily` (at a specific time), and `once` (one-shot delay) schedule types
- **Custom tools** — define reusable named tools with `create_tool`; call them by name in natural language
- **Persona modes** — `neutral` (default), `friendly`, `technical`, `witty`; persisted across restarts
- **Telegram bot** — optional background integration; same agent handles both CLI and Telegram in parallel
- **Rate limiting** — configurable hourly and daily caps with persistent counters
- **Timezone support** — `set_timezone`/`get_timezone` controls daily schedule times and `get_time` output

---

## Tools Reference

### Memory

| Tool | Description |
|---|---|
| `memory_set` | Store a value. Key must start with `u_`. |
| `memory_get` | Retrieve a stored value. |
| `memory_list` | List all `u_*` keys. |
| `memory_delete` | Delete a stored key. |

### Scheduler

| Tool | Description |
|---|---|
| `cron_set` | Create a schedule: `periodic`, `daily`, or `once`. |
| `cron_list` | List all active schedules as JSON. |
| `cron_delete` | Delete a schedule by ID. |
| `get_time` | Get current date and time. |
| `set_timezone` | Set timezone for schedules and `get_time`. |
| `get_timezone` | Get current timezone setting. |

### Persona

| Tool | Description |
|---|---|
| `set_persona` | Set tone: `neutral`, `friendly`, `technical`, `witty`. |
| `get_persona` | Get current persona. |
| `reset_persona` | Reset to `neutral`. |

### System

| Tool | Description |
|---|---|
| `get_version` | Current zedclaw version. |
| `get_health` | Rate limit status, current time, version. |

### User Tools

| Tool | Description |
|---|---|
| `create_tool` | Define a named tool with an action description. |
| `list_user_tools` | List all user-defined tools. |
| `delete_user_tool` | Delete a user-defined tool by name. |

### Slash Commands (CLI / Telegram)

| Command | Description |
|---|---|
| `/help` | Show help message. |
| `/settings` | Show current status (persona, intake state). |
| `/stop` | Pause message intake. |
| `/resume` | Resume after `/stop`. |
| `/exit` | Quit the CLI process. |

---

## Building from Source

Requires [Zig 0.13.0](https://ziglang.org/download/).

```bash
# Debug build
zig build

# Optimised release build
zig build -Doptimize=ReleaseSafe

# Run directly
zig build run

# Run tests
zig build test
```

The binary is written to `zig-out/bin/zedclaw`. No external libraries are required — only Zig's standard library.

---

## Changelog

See [CHANGELOG.md](CHANGELOG.md) for the full history.

---

## License

MIT
