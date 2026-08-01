# 🔒 Hermes Incognito Mode v2.5.9

[中文版 (Chinese)](README_CN.md) | [English](README.md)

> Browser incognito mode — upgraded for AI agents. Defense-in-depth, zero trace left.

## What is this?

A skill for [Hermes Agent](https://hermes-agent.nousresearch.com/) that ensures complete session privacy through a **four-layer defense-in-depth architecture**:

1. **Skill Policy** — Capability matrix blocking persistent writes (memory, skills, cron, config)
2. **Runtime Guardrails** — Shell history suppression, compound command wrapping (`bash -c`), timestamp TTL sandbox
3. **Framework Support** — Session-level isolation markers, subagent inheritance protocol
4. **Post-Session Audit** — 10+ step reverse audit pipeline with secure wipe (Python `os.urandom` overwrite → `fsync` → `truncate` → `unlink`), including `/tmp/` root and `hermes-snap-*.sh` terminal snapshots

## Architecture

```
Phase 1: Idempotency check + timestamp TTL sandbox init + orphan cleanup
   ↓
Phase 2: Isolated execution (pre-hoc defense)
   ↓
Phase 3: User confirmation gate / 15min TTL
   ↓
Phase 4: Full reverse audit (10+ steps) + secure wipe + session destruction
   ↓
Phase 5: Audit report + final receipt
```

### What it protects against

| Layer | Protection |
|-------|-----------|
| Filesystem | All writes confined to timestamp TTL sandbox; `/tmp/` root scan detects bypasses; non-sandbox writes detected and wiped |
| Shell History | Every command wrapped with `HISTFILE=/dev/null HISTSIZE=0`; compound statements (`if`/`for`/`while`) via `bash -c '...'` |
| Terminal Snapshots | `hermes-snap-*.sh` files (plaintext command history + env vars) included in forced secure wipe |
| Web Cache | 7 user-data dirs under `~/.hermes/cache/` (web/screenshots/videos/images/audio/documents/vision) symlinked to sandbox — crash-safe pre-hoc isolation |
| Log Redaction | `agent.log` queries/user-message plaintext scrubbed to `[REDACTED_INCOGNITO_QUERY]` by **timestamp window**, covering 7 search providers; v2.5.6+ the **Logging RedactingFilter plugin** intercepts in memory pre-write (plaintext never hits disk), 4.6b post-scrub as backstop |
| Vector Index | Sentinel file (`/tmp/.hermes-incognito-active`) fine-grained-skips `active_sessions` source — incognito content never enters the persistent vector store; 4.9 session delete implicitly releases |
| Memory | SHA-256 hash diffing against baseline snapshot |
| Skills/Cron | Detects unauthorized skill/cron creation during session |
| Processes | Snapshot diffing to detect orphan processes; `.python_history` included |
| Subagents | Residual subagent sandboxes + live transcripts + `async_delegations` table audited during cleanup |
| Crash Residue | `interrupted_turns.json` (desktop auto-continue markers) cleaned by sid |
| Session | Container destruction as final line of defense |

### Known blind spots (informed consent)

- OS-level: swap, core dumps, syslog/journald, filesystem atime
- Network-level: corporate proxy/firewall logs, DNS queries, LLM API provider logs (30-day retention by default)
- Third-party: IDE file watchers, existing service logs

## Installation

```bash
# Clone into your Hermes skills directory
git clone https://github.com/GenmetsuWenxuePress/hermes-incognito-mode.git ~/.hermes/skills/incognito-mode

# Or copy manually
cp SKILL.md ~/.hermes/skills/incognito-mode/
```

## Dependencies (v2.5.6+)

Pre-write log interception relies on two optional components (absent → falls back to 4.6b post-scrub, functionality intact):

1. **incognito-log-filter plugin** (recommended) — redacts queries/URLs/user-message previews in memory while the sentinel is active, so plaintext never reaches disk:
   ```bash
   # Copy the plugin into your Hermes user plugins dir
   cp -r incognito-log-filter ~/.hermes/plugins/
   hermes plugins enable incognito-log-filter
   ```
   > The plugin's `register()` runs at Hermes process startup — restart Hermes after enabling. Phase 1 step 3.6 auto-checks plugin status (soft warning, non-blocking).

2. **index_all.py sentinel support** (optional) — if you run the vector-index cron (`~/.hermes/scripts/index_all.py`), upgrade its `incognito_active()` to v2.5.5 semantics (fine-grained skip + session existence check). The incognito session creates/releases the sentinel automatically; vector jobs are never blocked.

## Usage

Activate in any Hermes session:

```
/incognito start
```

Mid-session commands:

| Command | Action |
|---------|--------|
| `/incognito status` | Show sandbox path, PID lock, file inventory |
| `/incognito audit` | Status + recent terminal/web activity |
| `/incognito export <path>` | Export results to persistent storage |
| `/incognito abort` | Emergency skip to secure destruction |

## Ending an incognito session

> ⚠️ **Manual step required.** The agent does NOT auto-destroy — you must explicitly end the session.

When you're done, tell the agent:

```
确认销毁
```

Or for emergency skip straight to destruction:

```
/incognito abort
```

The agent will then:
1. Run the 10+ step reverse audit (Phase 4)
2. Securely wipe all temp files including terminal snapshots (`hermes-snap-*.sh`)
3. Present a final audit report (Phase 5)
4. Destroy the session container

> 💡 **15-minute TTL reminder**: if you walk away mid-session, the agent will prompt you to end it after 15 minutes of inactivity. Auto-destruction requires `[Framework L2]` support (see §Limitations in [README_CN.md](README_CN.md)).

Then start a **new session** (`/new`) to ensure the old session container is fully purged.

### v2.6.0
- **4.9r recovery protocol** — abnormal-close recovery: sentinel residue + session deleted → re-run 4.6b idempotently (field-tested: 16 plaintext lines scrubbed)
- **Env-aware plugin check (3.6)** — serve/gateway backends never load user plugins (`_plugin_cli_discovery_needed` skips built-ins); check now detects the runtime env
- **Hermes core patch** — cmd_dashboard explicitly calls `discover_plugins()` so desktop/serve loads user plugins (16 lines, pending upstream merge)

## Changelog

### v2.5.9
- **P0 fix: bash single-quote safety** — bare `'` inside 4.6b's `python3 -c '...'` (comments/`(["'])` regex) broke the bash wrapper and silently failed on real terminal runs — now chr()-concatenated; 30/30 bash blocks verified
- **Release compliance** — description ≤60 chars, `platforms` field, generalized example wording

### v2.5.8
- **async_delegations table audit** — delegate_task briefs/results persist into state.db (4.9 doesn't clear them); now deleted by sid
- **unset timing fix** — moved after 4.8 so 4.6b/4.7b/4.8 never lose env vars silently

### v2.5.7
- **Plugin dependency check (step 3.6)** — Phase 1 explicitly verifies incognito-log-filter is enabled, preventing silent loss of the pre-write defense

### v2.5.6
- **Logging RedactingFilter plugin** — sentinel-gated, in-memory redaction; pre-write interception + 4.6b post-scrub double defense
- **interrupted_turns.json audit (4.1c)** — crash-residue prompt plaintext cleaned by sid

### v2.5.5
- **agent.log dual-blindspot fix (R6 field-tested)** — covers 7 search-provider log formats + `conversation turn` user-message plaintext (repr double-quote compatible) + URL lines
- **Sentinel semantics rework** — fine-grained skip (only `active_sessions` source) + 4.9 session delete implicitly releases

### v2.5.4
- **profile-safe completion** — 4 runtime paths now use `$INCOGNITO_HERMES_HOME`; recovery block restores the variable

### v2.5.3
- **R4 deep-audit fixes** — 4.6b timestamp-window scrub (old regex 0-hit silent failure), 4.1a delegation live-transcript audit, vector-index sentinel, Phase 4 recovery split, 7-dir cache redirect, 4.1 exclusion list, capability matrix (gbrain/agy/execute_code), 4.6c snap re-wipe

### v2.5.2
- **Skill audit noise reduction** — 4.4 find excludes `.curator_state`/`.bundled_manifest`/`.usage.json` metadata

### v2.5.1
- **Export persistence fix** — Phase 1 split into two `terminal()` calls: simple command chain for `export` (persists across calls), then `bash -c` for idempotency logic (subshell-safe). Fixes broken `$INCOGNITO_TMP_DIR` in long sessions.

### v2.5.0
- **Web cache symlink to sandbox** — `~/.hermes/cache/web/`, `screenshots/`, `videos/` symlinked into sandbox. Pre-hoc isolation replaces post-hoc wipe — crash-safe, no cache to clean up after a crash.
- **Agent log keyword redaction** — `web_search` query plaintext in `agent.log` redacted to `[REDACTED]` by session ID on cleanup.
- **Terminal snapshot auto-wipe** — `hermes-snap-*.sh` files (contain plaintext command history + env vars) now included in forced secure wipe

### v2.4.0
- **Cache + tmp + subagent blind spots closed** — `~/.hermes/cache/`, `/tmp/` root, subagent residual directories now audited and wiped
- **Python history audit** — `.python_history` added to process/filesystem audit scope
- **Process audit degraded** — acknowledged inherent limitation of Hermes `terminal()` model
- **Phase 4 environment recovery** — auto-restore `$INCOGNITO_TMP_DIR` via `pid.lock` session matching if variable lost during long sessions
- **Expanded pre-exclusion paths** — `.hermes/cache/`, `.local/state/tirith/` added to noise filter
- **Deduplicated session delete** — removed duplicate Phase 4.9 / Phase 5 `hermes sessions delete` instructions
- **Phase 1 numbering fix** — corrected step numbering after v2.3.0 additions
- **Generalized pre-exclusion paths** — broader path filtering in filesystem audit

### v2.3.0
- **Timestamp TTL orphan cleanup** — replaces broken PID lock check (Hermes `terminal()` spawns independent bash, PID dies immediately)
- **Compound shell command wrapping** — `if`/`for`/`while` wrapped in `bash -c '...'` to avoid syntax errors from direct `HISTFILE=/dev/null` prefix
- **Security scanner compatibility** — reduced false positives from `skills-guard-v1`
- **System path noise filtering** — narrowed filesystem audit scope
- **Quote escaping standardization** — consistent quoting patterns across all shell blocks

### v2.2.1 (initial release)
- PID-locked sandbox with Python `os.urandom` secure wipe
- 10-step reverse audit pipeline
- Subagent inheritance protocol
- 15min TTL with user confirmation gate

## License

MIT — see [LICENSE](LICENSE)

## Author

幻灭文学出版社 + Hermes (17-round cross-audited)
