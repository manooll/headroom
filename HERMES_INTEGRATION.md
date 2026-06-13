# Wiring the local-only Headroom build into the Hermes runtime

Findings + recipe from putting this fork's proxy in front of the live Hermes agent
(OpenRouter backbone, `google/gemini-3.5-flash`). Goal: cut the ~$30/day Gemini
token bill via compression, with no telemetry and full reversibility.

## TL;DR

- **It works.** Every OpenRouter call from Hermes (DM + cron + aux models) now flows
  through a local Headroom proxy; nothing phones home; Hermes behaves normally.
- **Measured potential: ~90% reduction** on real historical traffic (large cron /
  research sessions). **Early-conversation live savings are ~10–12%** because small
  requests have little to compress — savings scale with context size (big tool
  outputs, long history, web-research dumps are where the 90% lives).
- The non-obvious blocker: Hermes's `openrouter` provider **ignores `model.base_url`**.
  See "Routing lever" below.

## Components installed (all reversible, none phone home)

1. **Build:** this fork (`local-only-no-telemetry`) built via maturin into a venv at
   `~/hermes-workspaces/headroom/.venv` (Python 3.12, Rust `_core` compiled).
2. **Keepalive proxy:** governable launchd agent `com.hermes.headroom-proxy`
   (owned by Jay), `--backend openrouter --port 8788`, telemetry off,
   `HEADROOM_EXCLUDE_TOOLS=read_file,headroom_retrieve`.
   - Launcher: `~/.hermes/headroom-proxy.sh` (reads `OPENROUTER_API_KEY` from
     `~/.hermes/.env`; **no secret embedded in the plist**).
   - Plist: `~/Library/LaunchAgents/com.hermes.headroom-proxy.plist` (KeepAlive).
   - Logs: `~/.hermes/logs/headroom-proxy.log`.
   - Stop: `launchctl bootout gui/$(id -u)/com.hermes.headroom-proxy`.
3. **Retrieve plugin:** `plugins/hermes/headroom_retrieve` → `~/.hermes/plugins/`,
   `_PROXY_URL` edited to `:8788`; enabled in `config.yaml`
   (`toolsets: + headroom`, `plugins.enabled: [headroom_retrieve, hermes-memory-store]`).
   Lets the agent expand CCR compression markers instead of treating them as opaque.

## Routing lever (the hard-won part)

Hermes's `openrouter` provider does **not** read `model.base_url` from `config.yaml`
(only the `anthropic` provider does). The URL is resolved in
`hermes_cli/runtime_provider.py`:
- **env/config path** (`resolve_runtime_provider`): `base_url = ... or OPENROUTER_BASE_URL env or OPENROUTER_BASE_URL constant`.
- **pool path** (`_resolve_runtime_from_pool_entry`, used by cron): `base_url = entry.base_url or OPENROUTER_BASE_URL constant`.

Both paths fall back to the `OPENROUTER_BASE_URL` **constant** in `hermes_constants.py`.
So the reliable universal lever is a 2-line local patch to that constant **plus** the
env var (the env var also fixes key-selection so `OPENROUTER_API_KEY` is sent, though
the proxy injects its own key when the client sends none):

```python
# hermes_constants.py  (on branch local/stability-patches-*)
OPENROUTER_BASE_URL  = "http://127.0.0.1:8788/v1"        # inference -> proxy
OPENROUTER_MODELS_URL = "https://openrouter.ai/api/v1/models"  # discovery -> DIRECT
```

**Why the split:** the proxy mis-routes `/v1/models` to OpenAI, and
`OPENROUTER_MODELS_URL` (derived from the constant) is used at runtime by
`agent/model_metadata.py`. Keep model discovery pointed at openrouter.ai.

```bash
# ~/.hermes/.env
OPENROUTER_BASE_URL=http://127.0.0.1:8788/v1
```

Apply with: `launchctl kickstart -k gui/$(id -u)/ai.hermes.gateway`
(verify `pgrep -f "gateway run" | wc -l` == 1).

Verify routing: `curl -s 127.0.0.1:8788/stats` (requests>0) and
`~/.hermes/logs/agent.log` should show `base_url=http://127.0.0.1:8788/v1`.

## What did NOT work

- Editing `model.base_url` in `config.yaml` → **inert** for openrouter (ignored;
  agent kept calling openrouter.ai directly — no harm, no compression).
- `OPENROUTER_BASE_URL` env var **alone** → covers the env/config path but the cron
  **pool path ignores env** (uses the constant), so cron would bypass the proxy.

## Measured savings

| Source | Reduction |
|---|---|
| Real historical sessions via `compress()` (large cron/research, 240-msg) | **90–94%** |
| Mid-size sessions | 40–75% |
| Live early-conversation DMs (small input) | ~0–11% (little to compress) |
| Live cumulative so far (`savings.per_project`) | ~10.8% / ~49.8k tokens |

Savings scale with per-request context size. The bill is dominated by large-context
calls (web research, long history), which compress ~90% — that's where the win is.

## Reversibility (full rollback)

```bash
# Hermes constant patch
cp /tmp/hermes_constants.py.bak_headroom /Users/sage/hermes_local/hermes/hermes_constants.py
# env var: remove the OPENROUTER_BASE_URL line (backup: ~/.hermes/.env.bak_headroom_*)
# config.yaml: backup at ~/.hermes/config.yaml.bak_headroom_*
launchctl kickstart -k gui/$(id -u)/ai.hermes.gateway   # verify single gateway
# proxy agent can stay up (idle) or: launchctl bootout gui/$(id -u)/com.hermes.headroom-proxy
```

## Telemetry status

Confirmed `Anonymous telemetry: DISABLED` in the live proxy log even with
`HEADROOM_TELEMETRY=on` forced. No vendor egress. See [FORK_CHANGES.md](FORK_CHANGES.md).

## Not yet done

- **Dev-loop Codex lane** (originally selected): Codex runs on a fixed monthly
  subscription, so compression saves rate-limit headroom, not dollars. Would need a
  second codex-backend proxy + scoped `[model_providers.headroom]` config. Deferred.
- **Push** of `local-only-no-telemetry` to `manooll/headroom`: blocked on GitHub
  write auth (no `gh`, no SSH key, no stored HTTPS credential).
