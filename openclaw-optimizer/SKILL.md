---
name: openclaw-optimizer
slug: openclaw-optimizer
version: 26.3.17
description: "Optimize OpenClaw agent context (bootstrap discipline, compaction, session management) and audit identity files (SOUL.md, IDENTITY.md, AGENTS.md, USER.md, MEMORY.md). Advisory by default — audit first, propose exact changes, apply only on approval."
triggers:
  - optimize agent
  - optimizing agent
  - improve OpenClaw setup
  - agent best practices
  - OpenClaw optimization
  - context bloat
  - context management
  - bootstrap audit
  - bootstrap files
  - token usage
  - compaction
  - identity audit
  - personality audit
  - agent personality
  - agent identity
  - optimize personality
  - SOUL.md
  - IDENTITY.md
  - USER.md
  - MEMORY.md
  - troubleshoot
  - not working
  - error
metadata:
  openclaw:
    emoji: "🧰"
---

# OpenClaw Optimizer

**Aligned with: OpenClaw v2026.3.11** | v26.3.17 | Updated: 2026-03-17 | CLI-first advisor

Optimize OpenClaw agent context usage and identity files: bootstrap file discipline, compaction tuning, session management, and agent personality/identity audit.

**Reference files (load when needed):**
- `references/identity-optimizer.md` — agent identity/personality audit checklist, file roles, walkthrough workflow

---

## Version Awareness

This skill tracks OpenClaw releases via two mechanisms:

1. **GitHub Actions** — daily workflow checks for new releases, opens an issue on drift, auto-closes when resolved
2. **Runtime check** — lightweight cached version comparison at session start

### Runtime Check (once per session)

```bash
python3 ~/.claude/skills/openclaw-optimizer/scripts/version-check.py --status
```

- **`CURRENT`** → note the version and proceed.
- **`STALE`** → inform the user: "OpenClaw v`<new>` is available (skill is at v`<current>`). Run `update-skill.sh` to review what changed."
- **`UNCHECKED`** → note "Version check unavailable (offline)" and proceed.

### Update Workflow (user-initiated, never automatic)

```bash
# Show drift report, changelog, and affected sections
bash ~/.claude/skills/openclaw-optimizer/scripts/update-skill.sh

# After updating content in SKILL.md and references/:
bash ~/.claude/skills/openclaw-optimizer/scripts/update-skill.sh --apply    # bump versions
bash ~/.claude/skills/openclaw-optimizer/scripts/update-skill.sh --commit   # bump + commit + push
```

Updates are deliberate — this skill never auto-modifies its own content or pushes to git without explicit user action.

---

## Quick Start (copy/paste prompts)

**Full audit (safe, no changes):**
> Audit my OpenClaw agent's context usage and bootstrap files. Check for bloat, misplaced content, and compaction issues. Prioritized plan with rollback. Do NOT apply changes.

**Troubleshoot a specific problem:**
> [Describe your symptom or paste the error message]. Diagnose it and give me the exact fix.

**Audit agent personality & identity:**
> Audit my agent's personality and identity files. Check for conflicts, bloat, and bad practices. Walk me through improvements.

---

## Safety Contract (non-negotiable)

- This skill is **advisory by default** — not an autonomous control-plane.
- **Never** mutate config (`config.apply`, `config.patch`), cron jobs, or persistent settings without explicit user approval.
- Before any approved change: show (1) exact CLI command or config patch, (2) expected impact, (3) rollback command.
- If an optimization reduces monitoring coverage, present Options A/B/C and require the user to choose.

---

## Backup Strategy

Four backup layers exist — don't stack manual backups on top unnecessarily:

| Layer | What | Retention | When It's Enough |
|---|---|---|---|
| **CLI rolling `.bak`** | Auto-created on every `config set`, `models set`, `cron edit` | Rolling (overwritten each write) | Single-command undo |
| **Nightly GitHub backup** | Full config committed by cron job (3 AM) | Git history (unlimited) | Any rollback to a previous day's state |
| **`openclaw backup create`** | Local state archive with manifest verification (v2026.3.8+) | Until manually deleted | Pre-upgrade safety net; use `openclaw backup verify` to validate |
| **Manual dated backup** | `cp <file> <file>.YYYY-MM-DD-<reason>` | Until next nightly covers it, then delete | Major upgrades, multi-file restructuring, direct JSON edits |

**Rule:** For routine CLI changes (model swaps, cron edits, config sets), do NOT create manual backups. The CLI `.bak` + nightly GitHub backup are sufficient. Only create a manual backup when: (1) upgrading OpenClaw versions, (2) editing multiple config files simultaneously (identity audits), or (3) editing JSON directly without the CLI. For upgrades, prefer `openclaw backup create` over manual copies.

---

## 3. Context Management

**What burns tokens:** System prompt (5–10K tokens/call) + bootstrap files + conversation history. Bootstrap files injected on every turn (source: `docs.openclaw.ai/concepts/system-prompt`): `AGENTS.md`, `SOUL.md`, `TOOLS.md`, `IDENTITY.md`, `USER.md`, `HEARTBEAT.md`, `BOOTSTRAP.md` (first-run only), plus `MEMORY.md` and/or `memory.md` **when present**. Daily `memory/*.md` files are NOT auto-injected (on-demand via memory tools). Bootstrap cap: 150K chars total, 20K per file (both configurable).

> **MEMORY.md warning (from docs):** *"Keep them concise — especially MEMORY.md, which can grow over time and lead to unexpectedly high context usage and more frequent compaction."* MEMORY.md is the most common source of bootstrap bloat. Unlike AGENTS.md or SOUL.md which users actively edit, MEMORY.md tends to grow unchecked as the agent appends to it.

**Memory search embeddings (v2026.3.11+):** `gemini-embedding-2-preview` supports multimodal image and audio indexing for memory search. Configure via `memorySearch.provider`, `memorySearch.model`, and `memorySearch.extraPaths`.

**Check context:** `/status` · `/context list` · `/context detail` · `/usage tokens` · `/usage cost`

### Prompt Modes

| Mode | Bootstrap Files Loaded | Use Case |
|---|---|---|
| `full` (default) | All — AGENTS, SOUL, TOOLS, IDENTITY, USER, HEARTBEAT, MEMORY | Main interactive sessions |
| `minimal` (sub-agents) | AGENTS.md + TOOLS.md only | Sub-agent spawns — no SOUL, IDENTITY, USER, HEARTBEAT, MEMORY |
| `none` | Base identity line only | Bare-minimum sessions |

### Light Bootstrap (v2026.3.1+)

Skip all workspace bootstrap files for automated runs:

```bash
openclaw cron add --light-context --cron "*/30 * * * *" --message "Quick check"
```

```json5
{
  agents: { defaults: { heartbeat: {
    lightContext: true,     // only loads HEARTBEAT.md, skips all other bootstrap files
  } } },
}
```

Massive token savings for heartbeats and cron — eliminates 5-10K tokens/call of bootstrap overhead.

### Bootstrap Truncation Warning (v2026.3.7+)

```bash
openclaw config set agents.defaults.bootstrapPromptTruncationWarning once   # off | once | always
```

When a bootstrap file exceeds `bootstrapMaxChars` (default 20K), the agent receives a warning. Set to `always` during identity audits to catch truncated files.

### Compaction Config

```bash
# Manual: /compact [focus instructions]
# Auto: triggers near context limit — count visible in /status

openclaw config set agents.defaults.compaction.mode safeguard
openclaw config set agents.defaults.compaction.reserveTokensFloor 32000
openclaw config set agents.defaults.contextTokens 100000
openclaw config set agents.defaults.compaction.model google/gemini-3-flash-preview   # cheaper compaction (v2026.3.7+)
openclaw config set agents.defaults.compaction.recentTurnsPreserve 4                 # quality-guard (v2026.3.7+)
```

```json5
{
  agents: { defaults: { compaction: {
    mode: "safeguard",
    model: "google/gemini-3-flash-preview",    // route compaction through a cheaper model
    reserveTokensFloor: 32000,
    recentTurnsPreserve: 4,                    // keep last N turns intact during compaction
    postCompactionSections: ["Session Startup", "Red Lines"],  // AGENTS.md sections re-injected after compaction
    memoryFlush: {
      enabled: true,
      prompt: "Write lasting notes to memory/YYYY-MM-DD.md; reply NO_REPLY if nothing to store.",
    },
  }, contextTokens: 100000 } },
}
```

**Known bug — memory flush threshold gap (Issue #25880):** Set `reserveTokensFloor` equal to `reserveTokens` (both `62500`) to fix compaction firing before flush completes.

**Known bug — compaction timeout (Issue #38233):** Both `/compact` and auto compaction can timeout at ~300s with `openai-codex/gpt-5.3-codex`, freezing the session. Fix: override compaction model to `google/gemini-3-flash-preview` with `thinking: "off"`. Tune: `maxHistoryShare: 0.6`, `reserveTokensFloor: 40000`, `maxAttempts: 3`.

### Context Engine Plugin (v2026.3.7+)

Replace the built-in context assembly pipeline with a custom plugin:

```json5
{
  plugins: { slots: { contextEngine: "lossless-claw" } },   // default: "legacy" (built-in)
}
```

Context Engine plugins get full lifecycle hooks: `bootstrap`, `ingest`, `assemble`, `compact`, `afterTurn`, `prepareSubagentSpawn`, `onSubagentEnded`. This enables alternative context management strategies (lossless context, semantic chunking, etc.) without modifying OpenClaw core.

### Bootstrap File Size Targets (optimization recommendations)

These are optimization targets for keeping context lean, not hard limits. All files are subject to `bootstrapMaxChars` (default 20K) and `bootstrapTotalMaxChars` (default 150K).

| File | Target Size | Purpose | Injected? |
|---|---|---|---|
| `SOUL.md` | < 1K tokens (~4K chars) | Personality + absolute constraints | Always (main + full prompt mode) |
| `AGENTS.md` | < 2K tokens (~8K chars) | Workflows, rules, operating procedures | Always (main + sub-agents) |
| `TOOLS.md` | < 2K tokens (~8K chars) | Tool-specific notes, local conventions | Always (main + sub-agents) |
| `IDENTITY.md` | < 500 tokens (~2K chars) | Name, vibe, emoji, presentation | Always (main only) |
| `USER.md` | < 1K tokens (~4K chars) | User profile, preferences, context | Always (main only) |
| `HEARTBEAT.md` | < 200 tokens (~800 chars) | Heartbeat checklist (keep minimal) | Always (main only); skipped with `lightContext` |
| `MEMORY.md` | < 5K tokens (~20K chars) | **Curated long-term facts ONLY** | **Always in main sessions (auto-injected when present)** |

**Critical:** MEMORY.md is auto-injected on every turn in main sessions, NOT loaded on-demand. It burns tokens continuously. Keep it as small as possible with only curated facts. Operational protocols belong in AGENTS.md. Tool notes belong in TOOLS.md.

### Bootstrap Content Placement (What Goes Where)

Users commonly dump all content into SOUL.md because it feels like "who the agent is." This bloats the file (burns tokens every turn) and confuses lighter models that can't prioritize across a noisy instruction set. Place content in the correct file:

| Content Type | Correct File | Common Mistake |
|---|---|---|
| Personality, voice, humor, constraints | SOUL.md | - |
| Protocols, workflows, checklists, operational rules | AGENTS.md | Dumping in SOUL.md |
| User bio, preferences, working hours, communication style | USER.md | Duplicating in SOUL.md |
| Tool configs, API templates, channel IDs, env vars | TOOLS.md | Scattering in AGENTS.md |
| Curated long-term facts (lean) | MEMORY.md | Growing unchecked |
| Proactivity rules, initiative behavior | AGENTS.md | Putting in SOUL.md |

**Cross-file duplication burns tokens silently.** If the same protocol appears in both SOUL.md and AGENTS.md, it's injected twice on every turn. Deduplicate aggressively — pick one canonical location.

**Stale model references are silent saboteurs.** When you change models via CLI (`openclaw models set`), update any AGENTS.md sections that reference specific model names (e.g., Model Selection, Sub-Agent defaults). The agent follows bootstrap instructions and may try to use models that are no longer configured.

**Persistence stack:** `SOUL.md` → `AGENTS.md` → `TOOLS.md` → `IDENTITY.md` → `USER.md` → `MEMORY.md` (all auto-injected in main sessions) → `memory/YYYY-MM-DD.md` (on-demand via memory tools) → `conversation-state.md` → `ACTIVE-TASK.md`

### Session Maintenance

```bash
openclaw config set session.maintenance.mode enforce
openclaw config set session.maintenance.maxDiskBytes 500mb
openclaw sessions cleanup --dry-run      # preview
openclaw sessions cleanup --enforce     # apply
openclaw sessions cleanup --fix-missing # prune store entries whose transcript files are missing (v2026.2.26+)
```

---

## 7. High-ROI Optimization Levers

| Lever | Impact | How |
|---|---|---|
| **Prompt caching** | 60–90% input token reduction | Keep system prompt stable; use `anthropic` direct |
| **Bootstrap file discipline** | 2K–10K tokens/call saved | SOUL.md <1K, AGENTS.md <2K, MEMORY.md <5K |
| **Light bootstrap for automated runs** | 5-10K tokens/call saved | `lightContext: true` on heartbeat; `--light-context` on cron |
| **Adaptive thinking** | Auto-scales token use | `thinkingDefault: adaptive` for Claude 4.6; `minimal` for routine |
| **Session pruning** | Reclaims stale context | `contextPruning.mode: cache-ttl` with Anthropic |
| **Compaction tuning** | Prevents overflow disasters | `safeguard` mode, `reserveTokensFloor: 32000` |
| **Cheaper compaction model** | Reduces compaction cost | Route compaction through `gemini-3-flash-preview` |
| **Session maintenance** | Prevents disk/perf degradation | `mode: enforce`, `maxDiskBytes: 500mb` |
| **Gateway security** | Prevents exposure | `gateway.bind: loopback`; Tailscale for remote |
| **Never switch mid-session** | Preserves prompt cache | Only switch model at `/new` boundaries |
| **Backup before upgrades** | Pre-change safety net | `openclaw backup create` before `openclaw update` |

---

## 8. CLI Reference

**Best practice (v2026.2.25+):** Before editing config or asking config-field questions, have the agent call the `config.schema` tool in-chat. This returns the current schema with valid keys, types, and defaults — avoids guessing or using stale field names. Note: this is an agent in-chat tool, NOT a CLI command.

**Most common commands:**
```bash
openclaw doctor --fix               # auto-fix config issues
openclaw gateway status             # check runtime + RPC probe
openclaw models set <provider/model>
openclaw models status --probe
openclaw sessions cleanup --dry-run
openclaw sessions cleanup --fix-missing  # prune entries with missing transcripts (v2026.2.26+)
openclaw config validate [--json]        # validate config against schema (v2026.3.2+)
openclaw config file                     # print active config file path (v2026.3.1+)
openclaw backup create [--only-config]   # local state archive (v2026.3.8+)
openclaw backup verify                   # validate backup integrity (v2026.3.8+)
openclaw update
openclaw security audit             # post-upgrade check
openclaw secrets audit              # scan bootstrap files for hardcoded secrets (v2026.2.26+)
openclaw secrets configure          # configure external secrets (v2026.2.26+)
openclaw secrets apply              # apply secrets with strict target-path validation (v2026.2.26+)
```

> **`openclaw onboard --reset` scope change (v2026.2.26):** Default reset scope is now `config+creds+sessions`. Workspace deletion (bootstrap files, skills, memory) now requires `--reset-scope full`. Do NOT run `openclaw onboard --reset` without specifying `--reset-scope` explicitly — the default no longer wipes the workspace.

### In-Chat Commands (v2026.3.x)

```
/usage cost                       local cost summary from session logs
/usage tokens                     show per-reply token usage
/model <provider/model>           switch model without restart
/compact [instructions]           manual compaction with optional focus
/context detail                   per-file, per-tool, per-skill token breakdown
/think <level>                    off | minimal | low | medium | high | xhigh | adaptive
/check-updates                    quick update summary
```

### Environment Variables (v2026.3.x)

```bash
OPENCLAW_LOG_LEVEL=<level>         # override log level: silent|fatal|error|warn|info|debug|trace
OPENCLAW_DIAGNOSTICS=<pattern>     # targeted debug logs (e.g., "telegram.*" or "*" for all)
OPENCLAW_SHELL=<runtime>           # set across shell-like runtimes (exec, acp, tui-local)
OPENCLAW_THEME=light|dark          # TUI theme override (v2026.3.8+)
```

**Gateway restart (macOS LaunchAgent):**
```bash
# SAFE restart — single atomic operation, no duplicate processes
launchctl kickstart -k gui/$(id -u)/ai.openclaw.gateway

# DO NOT use `openclaw gateway restart` — it races with KeepAlive and spawns
# duplicate processes that loop "Port already in use" every ~10s at 100%+ CPU.

# Recovery if duplicates already exist:
launchctl bootout gui/$(id -u)/ai.openclaw.gateway    # stop launchd service + kill managed process
kill <any-remaining-pids>                              # kill orphans
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/ai.openclaw.gateway.plist  # re-register + start clean
```

---

## 9. Ops Hygiene Checklist

**Daily:**
- `openclaw health --json` via cron (→ HEARTBEAT_OK if clean)

**Weekly:**
- `openclaw update --dry-run` → review → `openclaw update`
- Curate MEMORY.md — archive old daily logs, promote key insights
- `openclaw sessions cleanup --dry-run` → `openclaw sessions cleanup`
- Clean stale backup files: `find ~/.openclaw -name "*.bak.*" -mtime +7 -not -name "*.bak" | xargs rm -v` (preserves CLI's rolling `.bak` files, removes old named/dated backups)

**Quarterly:**
- Review custom scripts (`scripts/`) for redundancy with built-in OpenClaw features. Users often build custom solutions (RAG pipelines, session archivers, memory indexers) that become redundant when OpenClaw adds equivalent built-in functionality. Check whether each script and its associated cron job still serves a purpose that the platform doesn't already handle.

**Before/After Updates:**
- Before update: `openclaw backup create` (pre-change safety net — v2026.3.8+)
- After update: `openclaw doctor --fix` (handles config migrations automatically)
- After update: `openclaw config validate --json` (catch fail-closed config errors — v2026.3.2+)

**v2026.3.x Breaking Changes:**
- **Config fail-closed (v2026.3.2+):** Invalid configs cause gateway startup failure instead of silently falling back to permissive defaults.
- **`gateway.auth.mode` required (v2026.3.7):** When both `gateway.auth.token` AND `gateway.auth.password` are configured, you must set `gateway.auth.mode` to `"token"` or `"password"`. Gateway will not start without this.
- **WebSocket origin validation (v2026.3.11):** Browser origin validation enforced for all browser-originated connections even with proxy headers. Closes cross-site WebSocket hijacking in `trusted-proxy` mode (GHSA-5wcw-8jjv-m286).
- **Node.js v22.12+ enforced:** Attempting to run on Node 18/20 causes immediate failure.

**Security:**
- `openclaw config get gateway.bind` → must be `loopback`
- No public port exposure — use Tailscale for remote
- API keys not in skill files or version control
- CVE-2026-25253 (ClawJacked): WebSocket authentication bypass allowing one-click RCE. Patched in v2026.1.29+. Verify you are on v2026.2.26+ minimum.
- GHSA-5wcw-8jjv-m286 (v2026.3.11): Cross-site WebSocket hijacking in `trusted-proxy` mode. Browser origin validation now enforced for all browser-originated connections.
- `openclaw security audit --deep` for live Gateway probe

---

## 10. Troubleshooting

**Log file paths (macOS):**
- **Error log:** `~/.openclaw/logs/gateway.err.log` — primary source for errors, 502s, plugin failures, tool errors
- **Main log:** `/tmp/openclaw/openclaw-YYYY-MM-DD.log` — verbose debug output (lane events, session activity)

Always check `gateway.err.log` first when troubleshooting — it contains only errors and warnings, making root cause identification much faster than grepping the main log.

**First — always run this triage sequence:**
```bash
openclaw status
openclaw gateway status            # must show "Runtime: running" + "RPC probe: ok"
openclaw doctor
openclaw config validate --json    # catch config errors before restart (v2026.3.2+)
tail -50 ~/.openclaw/logs/gateway.err.log | grep -v DEP0040   # skip Node deprecation noise
```

**Quick fix by symptom:**

| Symptom | First Command | Most Likely Fix |
|---|---|---|
| No response from agent | `openclaw gateway status` | Gateway not running or pairing pending |
| Gateway won't start | `openclaw logs --follow` | `EADDRINUSE` or `gateway.mode` not set to `local` |
| "Port already in use" loop | `ps aux \| grep openclaw-gateway` | Duplicate processes from CLI restart vs LaunchAgent `KeepAlive`. Fix: `launchctl bootout` → kill orphans → `launchctl bootstrap` (see Section 8) |
| "Gateway start blocked: set gateway.auth.mode" | `openclaw config get gateway.auth` | Both token and password set but `gateway.auth.mode` missing. Fix: `openclaw config set gateway.auth.mode token` (v2026.3.7 breaking change) |
| "unauthorized" on Control UI | `launchctl getenv OPENCLAW_GATEWAY_TOKEN` | Remove stale launchctl env override |
| Config file wiped on restart | Back up config first | Known bug #40410 — gateway restart can wipe `openclaw.json`. Use `openclaw backup create` before restarts. |
| Bootstrap file truncated silently | `openclaw config set agents.defaults.bootstrapPromptTruncationWarning always` | File exceeds `bootstrapMaxChars` (default 20K). Enable warning to catch it, then reduce file size. |
| Post-upgrade breakage | `openclaw doctor --fix` | Automatic config migration |
| Compaction freezes session | Override compaction model | Known bug #38233 — `/compact` times out at ~300s with Codex models. Use `compaction.model: google/gemini-3-flash-preview` |
| ALL providers timeout simultaneously | `tail -50 ~/.openclaw/logs/gateway.err.log` | Context bloat cascade — see Section 10a below |

### 10a. Context Bloat Cascade Timeouts

**Symptom:** ALL providers in the fallback chain timeout simultaneously on the same request. The same `runId` appears across multiple providers in 90-second intervals. Looks like a massive outage but providers are actually fine.

**Pattern in logs:**
```
14:17:24 Profile openai-codex:default timed out. Trying next account...
14:18:54 Profile kimi-coding:default timed out. Trying next account...
14:20:25 [diagnostic] lane task error ... FailoverError: LLM request timed out.
14:21:55 Profile anthropic:manual timed out. Trying next account...
```

**Root cause:** `contextTokens` is unset (defaults to unlimited). The main session accumulates conversation history until the payload is so large that no provider can respond within `timeoutSeconds`. Each provider in the fallback chain gets the same oversized payload, times out, and passes to the next one — creating a cascade that takes `timeoutSeconds × number_of_providers` to fully fail.

**The deadly trio:**
1. **Unlimited `contextTokens`** — payload grows unchecked
2. **Short `timeoutSeconds` (e.g., 90)** — not enough time for large payloads
3. **Long fallback chain (4-5 providers)** — each one gets a full timeout cycle before failing

**Fix — recommended baseline for any mixed-provider fallback chain:**
```bash
openclaw config set agents.defaults.contextTokens 100000
openclaw config set agents.defaults.timeoutSeconds 180
openclaw config set agents.defaults.compaction.reserveTokensFloor 32000
openclaw config set agents.defaults.compaction.mode safeguard
```

**How this works together:**
- `contextTokens: 100000` — caps context so all providers can handle it
- Compaction triggers at ~68K tokens (100K minus 32K reserve)
- Memory flush runs first (if enabled), then compaction compresses history
- `timeoutSeconds: 180` — gives providers 3 minutes per attempt (vs 90s)
- The cap ensures every provider in the chain can respond in time

**Tradeoff:** Models with large context windows (Gemini: 1M, GPT-5.4: 1.05M) are capped at 100K. This is intentional — the cap must match the weakest provider in the fallback chain. For dedicated large-context sessions, temporarily increase `contextTokens`.

---

## 11. System Learning

This skill maintains **system profiles** — persistent knowledge files that capture everything learned about specific OpenClaw deployments. Each deployment gets a unique profile that grows over time, turning the skill into an expert on that particular system.

### How It Works

**Directory:** `~/.openclaw-optimizer/systems/` — one profile per deployment, plus `TEMPLATE.md` for new deployments. This is a **centralized location outside the skill directory** so that: (1) system profiles are never accidentally pushed to git, (2) multiple AI tools (Claude Code, OpenClaw, Gemini CLI, etc.) on the same machine can read/write the same profiles without drift. Cross-machine sync is still manual via SCP.

**Deployment ID:** Each deployment has a unique slug (e.g., `home-gateway`, `prod-cluster-east`, `dev-standalone`).

**Profile formats (two supported):**
- **Directory format (preferred):** `~/.openclaw-optimizer/systems/<deployment-id>/` — directory containing `INDEX.md` (always-loaded summary, ~1-4K tokens) plus topic files loaded on-demand. Dramatically reduces session-start context cost.
- **Single-file format (legacy):** `~/.openclaw-optimizer/systems/<deployment-id>.md` — monolith file containing everything. Still supported for backwards compatibility.

**Topology types:**
| Type | Description |
|---|---|
| `gateway-only` | Single gateway, no remote nodes |
| `hub-spoke` | One gateway, one or more client nodes connecting to it |
| `multi-gateway` | Multiple gateways, nodes may connect to different ones |
| `mesh` | Nodes interconnected, multiple gateways with cross-routing |

### Session Workflow

**First-run setup (once per machine):**
1. Check if `~/.openclaw-optimizer/systems/` exists
2. If not: inform the user that this skill stores deployment profiles in `~/.openclaw-optimizer/systems/` (centralized, outside git, shared across AI tools), confirm they're OK with creating it, then: `mkdir -p ~/.openclaw-optimizer/systems/` and copy `TEMPLATE.md` from the skill's `systems/` directory into it
3. If the directory exists but is empty (no TEMPLATE.md): copy `TEMPLATE.md` from the skill's `systems/` directory

**At session start (identify the deployment):**
1. Ask which deployment the user is working on, or identify it from context (SSH target, hostnames, IPs)
2. Check if `~/.openclaw-optimizer/systems/<deployment-id>/` **directory** exists
3. If directory found: read `INDEX.md` only (~1-4K tokens). Use the **File Manifest** table at the bottom to load topic files on-demand during the session — do NOT read all files upfront.
4. If directory NOT found but `<deployment-id>.md` **file** exists: read the monolith (legacy mode). Consider migrating to directory format.
5. If neither found: create a new profile from `~/.openclaw-optimizer/systems/TEMPLATE.md` during the session

**On any system assessment or audit (mandatory — run before making recommendations):**
1. `openclaw config get agents.defaults.model` — capture model routing (primary + fallbacks)
2. `ls ~/.openclaw/delivery-queue/*.json 2>/dev/null | wc -l` — check for stuck delivery entries
3. `openclaw nodes list` — check paired nodes and connection status
4. Document ALL findings in the system profile before making recommendations
5. **Without this data, recommendations will miss hidden drains.**

**During the session (on-demand file loading):**
- Reference INDEX.md for SSH access, IPs, routing, and cron status
- **When diagnosing any issue:** read `lessons.md` FIRST (check if it's already solved), then the relevant topic file
- **When troubleshooting context/compaction:** read `routing.md` for context config
- **When reviewing history:** read `issues/YYYY-MM.md` for the relevant month
- Apply lessons learned to avoid repeating mistakes

**At session end (update the profile):**

*For directory-based profiles:*
1. Update the specific **topic file(s)** that changed (e.g., `routing.md` if context settings were tuned)
2. Update `INDEX.md` only if **summary-level data changed**
3. Add new issues to `issues/YYYY-MM.md` (current month file, newest first) with: symptom, root cause, fix, rollback, lesson
4. Add new lessons to `lessons.md` (permanent, never archived)
5. Update the `Last updated` date in INDEX.md
6. Sync **only changed files** to the gateway: `scp ~/.openclaw-optimizer/systems/<deployment-id>/<changed-file> <user>@<host>:~/.openclaw-optimizer/systems/<deployment-id>/`
7. Note: system profiles live in `~/.openclaw-optimizer/systems/`, NOT in the skill directory. Do not commit them to git.

*For legacy single-file profiles:*
1. Add any new issues to the **Issue Log** (newest first) with: symptom, root cause, fix, rollback, lesson
2. Update **Lessons Learned** with new patterns discovered
3. Update machine details if anything changed (IPs, versions, config)
4. Update the `Last updated` date
5. Sync the profile to the gateway: `scp ~/.openclaw-optimizer/systems/<deployment-id>.md <user>@<host>:~/.openclaw-optimizer/systems/`

### What Gets Captured

| Topic | File (directory format) | Purpose |
|---|---|---|
| **Machines, Network, Paired Devices** | `topology.md` | Every machine: role, SSH, IPs, OS, paths, config. Tailnet, auth, connectivity. Device entries from paired.json. |
| **Providers** | `providers.md` | Active model providers with slugs, auth details, notes. Removed providers with context. |
| **Model Routing** | `routing.md` | Tiered routing table, fallback chain, heartbeat config |
| **Channels, Delivery Queue** | `channels.md` | Messaging channels, Telegram group mapping, stuck delivery entries |
| **Cron Jobs** | `cron.md` | Full inventory: job ID, name, schedule, model, status, observations |
| **Issues** | `issues/YYYY-MM.md` | Every problem encountered: symptom → root cause → fix → rollback → lesson |
| **Lessons Learned** | `lessons.md` | Accumulated patterns and gotchas specific to this deployment (permanent) |
| **Summary** | `INDEX.md` | Always-loaded overview with key tables and file manifest |

### Issue Lifecycle (directory format)

1. **New issues** go into `issues/YYYY-MM.md` (current month file, newest first)
2. **After 14 days:** full detail stays in the monthly file, a one-liner is added to `issues/archive.md`
3. **Monthly files are never deleted** — they're the permanent record
4. **Lessons** extracted from issues go to `lessons.md` (permanent, never archived)

### Rules

- **Never store full secrets** in profiles — use first 12 chars + `...` for tokens, never store full API keys
- **Always read the profile before troubleshooting** — don't rediscover what's already known
- **Always update the profile after fixes** — future sessions depend on accurate knowledge
- **One profile per gateway** — nodes are documented within the gateway's profile
- **Keep lessons actionable** — not "TLS was broken" but "macOS app rejects `ws://` for remote gateways — always use `wss://`"
- **Rely on built-in backup layers — don't create manual backups for routine changes.** OpenClaw's CLI creates rolling `.bak` files on every config write, and the nightly GitHub backup cron captures the full config in git history. Manual dated backups (`cp <file> <file>.YYYY-MM-DD-<reason>`) are only needed for: (1) major version upgrades, (2) multi-file restructuring (identity audits), (3) direct JSON edits where the CLI isn't used. For routine CLI changes (model swaps, cron edits, config sets), the CLI `.bak` + GitHub nightly are sufficient. Clean up old manual backups after they're covered by the nightly backup.

---


## 12. Continuous Improvement

This skill is a living document. Every troubleshooting session, every CLI interaction, and every failure is an opportunity to make it more accurate. Future sessions must actively update the skill based on real-world experience.

### When to Update SKILL.md

| Trigger | Action |
|---|---|
| A CLI command in the skill doesn't work as documented | Fix the command, add a note about what changed |
| A troubleshooting step is missing or incomplete | Add it to Section 10's symptom table |
| A workaround is discovered that isn't documented | Add it to the relevant section |
| Advice in the skill caused a failure | Correct the advice and add a warning |
| A new `openclaw` flag or subcommand is discovered during use | Update Section 8 (CLI Reference) |
| A new known bug or GitHub issue is found | Add it to the relevant section with issue number |
| A config key is renamed, deprecated, or new | Update the relevant config examples |

### What to Update

**Two targets — always update both when applicable:**

1. **SKILL.md** — general knowledge that applies to ALL deployments (CLI commands, config patterns, troubleshooting steps, known bugs, process workflows)
2. **System profile** (`systems/<deployment-id>.md`) — deployment-specific knowledge (IPs, paths, credentials, topology, issue log, lessons learned)

### How to Update

1. **During the session:** When you discover something new, update the relevant section immediately — don't wait until the end. Corrections to bad advice are urgent.
2. **Be specific:** Don't write "TLS can be tricky." Write "macOS app rejects `ws://` for remote gateways — always use `wss://`. The `ws://` scheme is only valid for loopback connections."
3. **Include the why:** Don't just say "use X instead of Y." Explain what goes wrong when you use Y.
4. **Preserve what works:** Only change what's actually wrong. Don't rewrite sections that are accurate.
5. **Sync to remote:** After updating, sync the skill and system profiles to any remote OpenClaw instances:
   ```bash
   # Sync SKILL.md (skill code — lives in the skill directory)
   scp ~/.claude/skills/openclaw-optimizer/SKILL.md <user>@<host>:~/.openclaw/workspace/skills/openclaw-optimizer/SKILL.md
   # Sync system profiles — directory format (sync only changed files)
   scp ~/.openclaw-optimizer/systems/<deployment-id>/<changed-file> <user>@<host>:~/.openclaw-optimizer/systems/<deployment-id>/
   # Sync system profiles — legacy single-file format
   scp ~/.openclaw-optimizer/systems/<deployment-id>.md <user>@<host>:~/.openclaw-optimizer/systems/
   ```

### Versioning

The skill uses semver (`MAJOR.MINOR.PATCH`) independent of OpenClaw's version:
- **PATCH** (1.3.0 → 1.3.1): Fixing a typo, correcting a command, small clarification
- **MINOR** (1.3.0 → 1.4.0): Adding a new section, new troubleshooting entries, new workflow
- **MAJOR** (1.3.0 → 2.0.0): Restructuring sections, breaking changes to the skill's own workflow

**On every commit that changes SKILL.md:**
1. Bump `version:` in the YAML frontmatter
2. Update the `Updated:` date in the header
3. Update the `Skill v` tag in the header

### Self-Audit Checklist (run mentally at session end)

- Did I discover any CLI behavior that contradicts the skill? → Fix Section 8 or 10
- Did I find a workaround that future sessions would need? → Add to relevant section
- Did I hit an error that's not in the troubleshooting table? → Add to Section 10
- Did I learn something deployment-specific? → Update the system profile
- Did any advice in this skill lead me astray? → Correct it with a warning
- Did I change SKILL.md? → Bump version, update date, commit, push, sync to gateway

---

## 13. Agent Identity Optimizer

Audit and optimize OpenClaw bootstrap/identity files for conflicts, bloat, misplaced content, and best practice violations. Interactive issue-by-issue walkthrough with preview diffs.

**Core files audited:** SOUL.md, IDENTITY.md, AGENTS.md, USER.md
**Supporting files (if present):** TOOLS.md, HEARTBEAT.md, MEMORY.md, BOOT.md

**What it checks (36 items):** Structural issues (truncation risk, bloat), content in the wrong file, conflicting/overlapping directives, best practice violations (official AGENTS.md template), USER.md completeness gaps, token efficiency.

**Workflow:** Collect files (local or SSH) → run checklist → present findings by severity → walk through each issue (approve/modify/skip) → apply changes → report token savings.

**Context-aware (v2026.3.7+):** When auditing, consider `lightContext` and `postCompactionSections` — files used only in `lightContext` mode (HEARTBEAT.md) or re-injected after compaction (`postCompactionSections` headings in AGENTS.md) have different optimization priorities. Ensure critical instructions appear under `postCompactionSections` headings (default: `Session Startup`, `Red Lines`) so they survive compaction.

**Full audit checklist, file role definitions, and detailed workflow:** Read `references/identity-optimizer.md`

---

## Output Shape

- **Executive summary** — what matters most + why
- **Top offenders** — context drivers, token bloat, reliability risks
- **Options A/B/C** — tradeoffs made explicit
- **Recommended plan** — smallest change first
- **Exact change proposals** — CLI commands or config patches, all with rollback
- **Rollback** — exact command to undo every change

Sources: docs.openclaw.ai, github.com/openclaw/openclaw, r/openclaw, r/myclaw, r/OpenClawUseCases
