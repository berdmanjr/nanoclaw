# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

# NanoClaw

Personal Claude assistant. See [README.md](README.md) for philosophy and setup. See [docs/REQUIREMENTS.md](docs/REQUIREMENTS.md) for architecture decisions.

## Quick Context

Single Node.js process with skill-based channel system. Channels (WhatsApp, Telegram, Slack, Discord, Gmail) are skills that self-register at startup. Messages route to Claude Agent SDK running in containers (Linux VMs). Each group has isolated filesystem and memory.

## Key Files

| File | Purpose |
|------|---------|
| `src/index.ts` | Orchestrator: state, message loop, agent invocation |
| `src/channels/registry.ts` | Channel registry (self-registration at startup) |
| `src/ipc.ts` | IPC watcher and task processing |
| `src/router.ts` | Message formatting and outbound routing |
| `src/config.ts` | Trigger pattern, paths, intervals |
| `src/container-runner.ts` | Spawns agent containers with mounts |
| `src/task-scheduler.ts` | Runs scheduled tasks |
| `src/db.ts` | SQLite operations |
| `groups/{name}/CLAUDE.md` | Per-group memory (isolated) |
| `container/skills/` | Skills loaded inside agent containers (browser, status, formatting) |

## Skills

Four types of skills exist in NanoClaw. See [CONTRIBUTING.md](CONTRIBUTING.md) for the full taxonomy and guidelines.

- **Feature skills** — merge a `skill/*` branch to add capabilities (e.g. `/add-telegram`, `/add-slack`)
- **Utility skills** — ship code files alongside SKILL.md (e.g. `/claw`)
- **Operational skills** — instruction-only workflows, always on `main` (e.g. `/setup`, `/debug`)
- **Container skills** — loaded inside agent containers at runtime (`container/skills/`)

| Skill | When to Use |
|-------|-------------|
| `/setup` | First-time installation, authentication, service configuration |
| `/customize` | Adding channels, integrations, changing behavior |
| `/debug` | Container issues, logs, troubleshooting |
| `/update-nanoclaw` | Bring upstream NanoClaw updates into a customized install |
| `/qodo-pr-resolver` | Fetch and fix Qodo PR review issues interactively or in batch |
| `/get-qodo-rules` | Load org- and repo-level coding rules from Qodo before code tasks |

## Contributing

Before creating a PR, adding a skill, or preparing any contribution, you MUST read [CONTRIBUTING.md](CONTRIBUTING.md). It covers accepted change types, the four skill types and their guidelines, SKILL.md format rules, PR requirements, and the pre-submission checklist (searching for existing PRs/issues, testing, description format).

## Development

Run commands directly—don't tell the user to run them.

```bash
npm run dev          # Run with hot reload
npm run build        # Compile TypeScript
npm test             # Run all tests (vitest)
npm test -- src/db.test.ts          # Run a single test file
npm test -- -t "test name pattern"  # Run tests matching a pattern
npm run lint         # ESLint
npm run format       # Prettier
./container/build.sh # Rebuild agent container
```

Service management:
```bash
# macOS (launchd)
launchctl load ~/Library/LaunchAgents/com.nanoclaw.plist
launchctl unload ~/Library/LaunchAgents/com.nanoclaw.plist
launchctl kickstart -k gui/$(id -u)/com.nanoclaw  # restart

# Linux (systemd)
systemctl --user start nanoclaw
systemctl --user stop nanoclaw
systemctl --user restart nanoclaw
```

## Architecture

### Message Flow

1. Channel receives message → stored in SQLite
2. Message loop (every 2s) polls `getNewMessages()` per group
3. If trigger word present (or `isMain`): `formatMessages()` → XML `<message>` blocks
4. `GroupQueue` enforces `MAX_CONCURRENT_CONTAINERS` (default 5) concurrency
5. `runContainerAgent()` spawns a Docker container, writes JSON to stdin
6. Container streams output between `---NANOCLAW_OUTPUT_START---` / `---NANOCLAW_OUTPUT_END---` markers
7. `onOutput` callback strips `<internal>` tags and sends via `channel.sendMessage()`
8. Container stays idle up to `IDLE_TIMEOUT` waiting for IPC follow-ups

### IPC (Host ↔ Container)

Containers communicate back to the host by writing JSON files to `/workspace/ipc/` (mounted from `data/ipc/{folder}/`). The IPC watcher (`src/ipc.ts`) polls this every 1s and processes:
- **messages/**: forward a message to another group (main-only to cross-group)
- **tasks/**: create/pause/resume/cancel/update scheduled tasks or register groups

### Container Mounts

| Mount | Container path | Access |
|-------|---------------|--------|
| Group folder | `/workspace/group` | read-write |
| Project root | `/workspace/project` | read-only (main only) |
| Global memory | `/workspace/global` | read-only (non-main) |
| IPC namespace | `/workspace/ipc` | read-write |
| Claude sessions | `/home/node/.claude` | read-write |

### Credential Proxy

`src/credential-proxy.ts` runs a local HTTP proxy that injects the Anthropic API key into container requests. Containers receive a dummy `ANTHROPIC_API_KEY` and route through `ANTHROPIC_BASE_URL` — real credentials never reach containers.

### Channel Registry

Channels self-register at startup via `registerChannel(name, factory)`. Each factory receives options and returns a `Channel` instance or `null` if credentials are missing. Channels implement `connect()`, `sendMessage()`, `isConnected()`, `ownsJid()`, `disconnect()` (and optionally `setTyping()`, `syncGroups()`).

### Data Directories

```
groups/{folder}/         # Group workspace (mounted into container)
data/
  ipc/{folder}/          # IPC file drop (messages/, tasks/)
  sessions/{folder}/     # Per-group Claude settings and synced skills
  store/messages.db      # SQLite: messages, chats, tasks, sessions, groups
```

## Troubleshooting

**WhatsApp not connecting after upgrade:** WhatsApp is now a separate skill, not bundled in core. Run `/add-whatsapp` (or `npx tsx scripts/apply-skill.ts .claude/skills/add-whatsapp && npm run build`) to install it. Existing auth credentials and groups are preserved.

## Container Build Cache

The container buildkit caches the build context aggressively. `--no-cache` alone does NOT invalidate COPY steps — the builder's volume retains stale files. To force a truly clean rebuild, prune the builder then re-run `./container/build.sh`.
