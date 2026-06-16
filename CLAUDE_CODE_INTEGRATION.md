# Wiring Headroom compression into Claude Code (this machine)

How this fork's Python compressor is wired in front of **Claude Code** on the
local machine, why each choice was made, and how to refresh it after a `git`
update. Companion to [HERMES_INTEGRATION.md](HERMES_INTEGRATION.md) (which does
the same for the Hermes/OpenRouter agent) and [FORK_CHANGES.md](FORK_CHANGES.md)
(the local-only / no-telemetry hardening).

## TL;DR

- **Scope: Claude Code only.** Claude **Desktop was deliberately skipped** — the
  desktop app talks to claude.ai with no configurable API base URL, so there is
  no supported/safe way to compress its traffic (would require TLS-MITM of
  claude.ai → breaks cert pinning + ToS + account risk). Untouched.
- **Auth: Claude subscription (OAuth)**, not an API key. This drove the
  `--mode cache` choice (below). With a subscription the win is **rate-limit
  headroom, not dollars** — savings scale with per-request context size.
- **Mechanism:** a local Headroom proxy (`127.0.0.1:8787`, Anthropic backend) +
  `ANTHROPIC_BASE_URL` pointed at it in Claude Code settings. Fully reversible.
- **Compressor used: the Python package, not the Rust crate.** See gotcha #5.

## Components installed (all reversible, none phone home)

1. **Runtime venv:** `~/.headroom/venv`, built with **Python 3.12**
   (`python3.12 -m venv`). Installed the prebuilt wheel
   `target/wheels/headroom_ai-0.25.0-cp310-abi3-macosx_11_0_arm64.whl`, then the
   `[proxy]` extra deps (fastapi, uvicorn, httpx[http2], openai, mcp, magika,
   zstandard, websockets, onnxruntime, transformers, watchdog, sqlite-vec).
2. **Editable `.pth` repointed to this repo:**
   `~/.headroom/venv/lib/python3.12/site-packages/headroom_ai.pth` →
   `/Users/pramana/Projects/OSS/headroom`. The wheel was built as an *editable*
   install baked to the original author's path (`/Users/sage/...`); the real
   `headroom/` package **and the prebuilt `headroom/_core.abi3.so`** live in this
   repo, so the venv resolves the package straight out of the working tree.
   (Upshot: `git pull` updates the Python package automatically — see "Updating".)
3. **Launcher:** `~/.headroom/proxy.sh` — runs
   `headroom proxy --host 127.0.0.1 --port 8787 --mode cache --backend anthropic --no-telemetry --log-file ~/.headroom/logs/proxy.jsonl`.
4. **Keepalive agent:** launchd `com.headroom.proxy`
   (`~/Library/LaunchAgents/com.headroom.proxy.plist`, `KeepAlive` + `RunAtLoad`).
   Logs: `~/.headroom/logs/proxy.{out,err}.log`; JSONL request log: `proxy.jsonl`.
5. **Claude Code routing:** `~/.claude/settings.json` →
   `env.ANTHROPIC_BASE_URL = "http://127.0.0.1:8787"`. Loads at Claude Code
   startup (restart Claude Code to activate).

## Why `--mode cache` (the hard-won decision)

The proxy has two modes:
- `token` (default): rewrites prior turns for maximum compression. **Busts
  Anthropic's prefix cache** on every compression event.
- `cache`: freezes prior turns; compresses only the **live zone** (latest user
  message + latest tool outputs). Preserves the prefix cache.

A **subscription/OAuth** client is prefix-cache sensitive, so `cache` is correct
here. This mirrors the entire `REALIGNMENT/` thesis: *"passthrough is sacred;
compress only the live zone, hash-keyed, position-preserving; never touch the
cache hot zone (system, tools, old turns, thinking)."*

**We did NOT use `headroom init claude`** (the built-in durable installer) for
two reasons: (a) its supervisor launches the proxy in `token` mode with no
override, and (b) this fork **disables `install_supervisor()`** anyway
(FORK_CHANGES.md §5). The manual launchd agent is the supported path on this fork.

## Gotchas discovered (read before debugging)

1. **System Python is 3.14; the wheel needs <3.14.** `litellm` caps at
   `>=3.10,<3.14`, so the venv MUST be built with `python3.12` (or 3.11/3.13).
   Building with 3.14 fails at `pip install` with "No matching distribution for
   litellm".
2. **The wheel is an editable stub.** `pip show -f headroom-ai` lists only a
   `headroom_ai.pth` — no package files. If `headroom` won't import, the `.pth`
   is pointing at a stale/foreign path; repoint it at this repo (component #2).
3. **Proxy needs the `[proxy]` extra.** A bare wheel install boots the CLI but
   `headroom proxy` dies with `No module named 'fastapi'`. Install the extra deps.
4. **PyTorch is absent → Kompress ML compressor is skipped** (harmless warning at
   boot). Structural compressors (SmartCrusher / log / diff / search) still run.
   Install `torch` only if you want the ML text compressor.
5. **The Rust `crates/headroom-proxy` is passthrough today.** It's mid-rewrite
   (`REALIGNMENT/`): `compress_anthropic_request` returns `NoCompression` for
   every well-formed body. Do **not** wire the Rust binary expecting compression
   — the working compressor is the **Python** package, which is what's wired here.
6. **This fork can't phone home.** Telemetry beacon / license / cloud-mode are
   hardwired off (FORK_CHANGES.md §1–3). `--no-telemetry` is belt-and-suspenders.

## Verify

```bash
curl -s 127.0.0.1:8787/livez                 # {"status":"healthy",...}
headroom-stats                               # shell helper: mode/requests/avg_pct/tokens_removed
headroom-stats-full                          # full /stats JSON
```
In a real Claude Code session, `summary.compression.requests_compressed` and
`avg_compression_pct` climb. Engine smoke test (no Anthropic auth needed):
a 13,315-token `tool_result` compresses to ~7,602 (~43%) via
`headroom.compress(messages, model="claude-sonnet-4-5")`.

Shell helpers (in `~/.zshrc`): `headroom-stats`, `headroom-stats-full`,
`headroom-proxy-log`, `headroom-proxy-restart`.

## Updating from git

The venv `.pth` points at this working tree, so:

```bash
cd ~/Projects/OSS/headroom
git pull
# If the Rust core (crates/) changed, rebuild the extension the Python pkg loads:
#   maturin develop --release    (or: make build) — only needed when _core.abi3.so is stale
# If pyproject's [proxy] deps changed, reinstall them into the venv:
#   ~/.headroom/venv/bin/pip install -U fastapi uvicorn 'httpx[http2]' openai mcp magika zstandard websockets onnxruntime transformers watchdog sqlite-vec
headroom-proxy-restart        # launchctl kickstart -k gui/$(id -u)/com.headroom.proxy
curl -s 127.0.0.1:8787/livez
```

Pure-Python changes need only `headroom-proxy-restart`. A symptom of a stale
`_core.abi3.so` after a Rust change is an `ImportError`/ABI mismatch in
`proxy.err.log` — rebuild with maturin then restart.

## Rollback (full)

```bash
# 1. Remove the ANTHROPIC_BASE_URL line from ~/.claude/settings.json
# 2. Stop + remove the keepalive agent
launchctl bootout gui/$(id -u)/com.headroom.proxy
rm ~/Library/LaunchAgents/com.headroom.proxy.plist
# 3. (optional) remove the runtime + shell helpers
rm -rf ~/.headroom
#    and delete the "--- Headroom ---" block from ~/.zshrc
```
Removing only the `ANTHROPIC_BASE_URL` line (step 1) + restarting Claude Code is
enough to instantly bypass the proxy while leaving everything installed.
