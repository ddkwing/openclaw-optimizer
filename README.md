# The OpenClaw Optimizer

> Fork of [jacob-bd/the-openclaw-optimizer](https://github.com/jacob-bd/the-openclaw-optimizer), trimmed to focus on context discipline and agent identity optimization.

**v26.3.17** | Aligned with OpenClaw v2026.3.11 | CLI-first advisor

An AI skill focused on two high-impact areas: **context discipline** (bootstrap file sizes, compaction tuning, session management) and **agent identity** (SOUL.md, IDENTITY.md, AGENTS.md, USER.md audit). Audits what you have, proposes exact changes, shows the rollback command before anything is applied.

> Advisory only. Recommendations are AI-generated — inspect and confirm every action before applying. Use at your own risk.

---

## Why This Exists

Most OpenClaw users are running with bloated bootstrap files (burning tokens every turn), MEMORY.md growing unchecked, and no idea that a single duplicate directive in SOUL.md and AGENTS.md is doubling their context load on every call.

This skill finds all of that. Then it fixes it — but only when you say so.

---

## What It Does

### Bootstrap File Discipline

Every bootstrap file has a token target. SOUL.md should be under 1K tokens. AGENTS.md under 2K. MEMORY.md under 5K. The skill audits each file's actual size, flags content that's in the wrong file, and catches silent cross-file duplication.

### Compaction Tuning

Uncapped context leads to cascade timeouts: the conversation grows until every provider in the fallback chain fails sequentially. The skill identifies the root cause and gives you the exact config to fix it — `contextTokens: 100000`, `reserveTokensFloor: 32000`, cheaper compaction model — with rollback commands for each.

### 36-Check Agent Identity Audit

Your agent's personality is scattered across SOUL.md, IDENTITY.md, AGENTS.md, USER.md, TOOLS.md, HEARTBEAT.md, and MEMORY.md. Content ends up in the wrong file, directives conflict, and duplicated instructions burn tokens silently on every turn.

The identity audit catches all of it: structural issues, misplaced content, overlapping directives, bloat, best practice violations, and USER.md gaps. Then walks you through each fix one at a time — approve, modify, or skip — with diffs and dated backups.

### Deployment Memory

Every session, the skill reads a deployment profile before it starts working. That profile contains your topology, context config, past issues, and lessons learned. When you fix a problem, the fix gets logged. Next time — even months later, even from a different AI tool — the skill already knows the answer.

Profiles support a **directory format** that splits the monolith into topic files (`topology.md`, `routing.md`, `lessons.md`, `issues/`). Only a lightweight `INDEX.md` (~1K tokens) loads at session start — topic files load on-demand. This cuts session-start context cost by 90%+ for mature deployments.

---

## Installation

Pick your tool. Each command downloads and installs the skill in one step.

**Claude Code:**
```bash
curl -fsSL https://raw.githubusercontent.com/ddkwing/openclaw-optimizer/main/install.sh | sh -s -- --tools claude
```

**OpenClaw:**
```bash
curl -fsSL https://raw.githubusercontent.com/ddkwing/openclaw-optimizer/main/install.sh | sh -s -- --tools openclaw
```

**Cursor:**
```bash
curl -fsSL https://raw.githubusercontent.com/ddkwing/openclaw-optimizer/main/install.sh | sh -s -- --tools cursor
```

**Gemini CLI:**
```bash
curl -fsSL https://raw.githubusercontent.com/ddkwing/openclaw-optimizer/main/install.sh | sh -s -- --tools gemini
```

**Multiple tools at once:**
```bash
curl -fsSL https://raw.githubusercontent.com/ddkwing/openclaw-optimizer/main/install.sh | sh -s -- --tools claude,openclaw,cursor
```

**All detected tools (auto-detects every supported AI tool on your system):**
```bash
curl -fsSL https://raw.githubusercontent.com/ddkwing/openclaw-optimizer/main/install.sh | sh
```

**Git clone (manual):**
```bash
git clone https://github.com/ddkwing/openclaw-optimizer.git
cd openclaw-optimizer
cp -r openclaw-optimizer ~/.claude/skills/             # or your tool's skills path
```

---

## Quick Start

**Full audit (safe, no changes):**
> Audit my OpenClaw agent's context usage and bootstrap files. Check for bloat, misplaced content, and compaction issues. Prioritized plan with rollback. Do NOT apply changes.

**Troubleshoot a specific problem:**
> [Describe your symptom or paste the error message]. Diagnose it and give me the exact fix.

**Audit agent personality & identity:**
> Audit my agent's personality and identity files. Check for conflicts, bloat, and bad practices. Walk me through improvements.

---

## Safety

The skill is **advisory by default**. It never touches your config or bootstrap files without explicit approval. Before every change: the exact CLI command, the expected impact, and the rollback command.

---

## What's Inside

| # | Section | What It Covers |
|---|---------|---------------|
| 3 | Context Management | Bootstrap file sizes, compaction config, prompt modes, light bootstrap, session pruning |
| 7 | High-ROI Levers | 11 levers ranked by impact — prompt caching, bootstrap discipline, compaction tuning |
| 8 | CLI Reference | Common commands, in-chat commands, env vars, safe gateway restart |
| 9 | Ops Hygiene | Weekly MEMORY.md curation, backup workflow, breaking changes, security |
| 10 | Troubleshooting | Triage sequence, symptom table, context bloat cascade deep-dive |
| 11 | System Learning | Deployment profiles (directory + legacy), on-demand topic loading, issue lifecycle |
| 12 | Continuous Improvement | Self-update triggers, versioning, self-audit checklist |
| 13 | Agent Identity Optimizer | 36-check audit, file role definitions, interactive walkthrough with diffs and backups |

**Reference file:**
- `references/identity-optimizer.md` — 36-check audit checklist, file roles, walkthrough workflow

---

## Repository Structure

```
README.md                       # This file
install.sh                      # One-liner installer script
CHANGELOG.md                    # Version history
docs/
  workflows.md                  # Workflow diagrams and architecture overview

openclaw-optimizer/             # The skill (copy this directory to install)
  SKILL.md                      # Main skill definition (8 sections)
  .gitignore                    # Excludes system profiles
  systems/
    TEMPLATE.md                 # Template for new deployments (copied on first run)
  references/
    identity-optimizer.md       # Agent identity/personality audit (36 checks)
  scripts/
    version-check.py            # Runtime version check (cached, once per session)
    update-skill.sh             # Drift report + changelog + version bump

~/.openclaw-optimizer/          # CENTRALIZED (not in git)
  systems/
    TEMPLATE.md                 # Deployment profile template (directory format)
    <deployment-id>/            # Directory format (preferred)
      INDEX.md                  # Always-loaded summary (~1K tokens)
      topology.md               # Machines, network, paired devices
      providers.md              # Active and removed providers
      routing.md                # Model routing, fallbacks, context config
      channels.md               # Telegram, WhatsApp, delivery queue
      cron.md                   # Full cron job inventory
      lessons.md                # Permanent lessons learned
      issues/
        YYYY-MM.md              # Monthly issue files
        archive.md              # Compressed summaries (14+ days old)
    <deployment-id>.md          # Legacy single-file format (still supported)
```

---

## Field-Tested Learnings

Everything below came from real production failures, not documentation.

**Context bloat cascade:** When `contextTokens` is unset (defaults to unlimited), the main session accumulates conversation history until no provider can respond in time. Each provider in the fallback chain gets the same oversized payload and times out in sequence. Fix: `contextTokens: 100000`, `timeoutSeconds: 180`, `reserveTokensFloor: 32000`.

**MEMORY.md silent growth:** MEMORY.md is auto-injected on every turn — not on-demand. Unlike other bootstrap files that stay stable, the agent appends to MEMORY.md over time without user awareness. A 50K-char MEMORY.md burning 12K tokens on every turn is common and invisible until you check `/context detail`.

**Cross-file duplication:** The same protocol in both SOUL.md and AGENTS.md is injected twice per turn. Users who write "be concise" in SOUL.md and "respond briefly" in AGENTS.md are paying double for the same instruction.

**Gateway restart:** Never use `openclaw gateway restart` on macOS — it races with the LaunchAgent's `KeepAlive: true` and spawns duplicate processes. Use `launchctl kickstart -k gui/$(id -u)/ai.openclaw.gateway` instead.

**compactionSections survival:** Instructions in AGENTS.md that aren't under a `postCompactionSections` heading get wiped when the context compacts. If your agent "forgets" key behaviors mid-session, this is why.

---

## Make It Yours

Fork this repo and maintain your own version. After every session where the skill helps you fix a context or identity issue, ask your AI tool:

> Did we learn anything this session that should be updated in the skill? Any CLI commands that didn't work as documented, troubleshooting steps that were missing, or advice that was wrong?

The skill has a built-in continuous improvement workflow (Section 12 in SKILL.md). It tracks when to update, what to update, and how to version bump.

---

## Version History

See [`CHANGELOG.md`](CHANGELOG.md) for the full release history.

## License

MIT
