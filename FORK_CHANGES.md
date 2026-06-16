# Fork changes — local-only / no-telemetry hardening

This fork strips out all "phone-home" network egress to vendor servers and
disables installation of persistent background services. Headroom still works
fully as a **local** library / proxy / MCP server / agent-wrap, and all
**local** stats (the `headroom perf` / `/stats` savings numbers) are preserved —
they never leave the machine.

## What was removed / neutered

### 1. Anonymous telemetry beacon (was ON by default)
- **File:** `headroom/telemetry/beacon.py`
- Sent anonymous aggregate stats to a Supabase endpoint every 5 min.
- `is_telemetry_enabled()` is now hardwired to `return False` — the beacon
  never starts.
- The Supabase URL + embedded anon key/table constants were deleted; the
  `_report()` HTTP POST is a hard no-op (defense in depth).

### 2. License validation + usage reporting (enterprise phone-home)
- **File:** `headroom/telemetry/reporter.py`
- Validated a license key and reported usage to `app.headroomlabs.ai`.
- `validate_license()` now returns a permissive local "active" license without
  any network call; `_report_usage()` is a no-op. Compression always works
  (fail-open). Original methods kept as `*_DISABLED` for reference.

### 3. "Headroom Cloud" mode (sent message *content* out, opt-in)
- **Files:** `headroom/integrations/asgi.py`, `headroom/integrations/litellm_callback.py`
- These could POST your messages to `api.headroomlabs.ai` for managed
  compression. `cloud_mode` is now hardwired `False`, so they always compress
  locally and can never transmit content, regardless of any api_key.

### 4. Third-party binary downloads (difft, scc) — now opt-in
- **File:** `headroom/binaries.py`
- GitHub release downloads are now OFF by default. Set
  `HEADROOM_BINARIES_ALLOW_DOWNLOAD=1` to re-enable. `ensure_tools()` already
  degrades gracefully when a tool can't be fetched.

### 5. Persistent service installation — disabled
- **Files:** `headroom/install/supervisors.py`, `headroom/install/runtime.py`
- `install_supervisor()` refuses to register any persistent OS service
  (systemd unit / cron / launchd plist / Windows service or scheduled task).
- `start_persistent_docker()` refuses to start an auto-restarting container.
- The teardown paths (`stop_supervisor` / `remove_supervisor`) are kept intact
  so any previously-installed daemon can still be removed.
- Removed `examples/deployment/macos-launchagent/` (launchd install example).

### 6. Local stats collector — preserved (decoupled from the dead beacon)
- **File:** `headroom/telemetry/collector.py`
- `TelemetryCollector` does **no network I/O**; it powers the local `/stats`
  endpoint and dashboard. It was previously gated by the same flag as the
  beacon, so disabling the beacon would have disabled local stats too. It is
  now decoupled: ON by default, turned off only by the explicit legacy opt-out
  `HEADROOM_TELEMETRY_DISABLED=1`.

## Left in place (off by default, user-controlled, not vendor phone-home)
- **Langfuse OTLP tracing** (`headroom/observability/tracing.py`) — `enabled=False`
  by default; only sends to *your* Langfuse if you set `HEADROOM_LANGFUSE_ENABLED`
  + your own keys.
- **OpenTelemetry metrics** (`headroom/observability/metrics.py`) — exports only
  to an endpoint you configure.
- The proxy forwarding requests to LLM providers (Anthropic/OpenAI/Bedrock/…) —
  that is the proxy's normal job, not telemetry.

## Verifying no egress remains
```bash
grep -rn "supabase.co\|headroomlabs.ai" headroom --include="*.py"   # only dead/disabled refs
# Run the proxy and confirm the log says: "Anonymous telemetry: DISABLED"
```

## Local integrations on this machine (how the proxy is actually wired)

These docs record the live, reversible wiring of the local-only proxy in front
of real agents — read them before changing or refreshing a setup:

- **[CLAUDE_CODE_INTEGRATION.md](CLAUDE_CODE_INTEGRATION.md)** — Claude Code
  (subscription/OAuth) → local proxy on `:8787` in `cache` mode, via launchd
  `com.headroom.proxy` + `ANTHROPIC_BASE_URL`. Includes the Python-3.12 / editable
  `.pth` / `[proxy]` extras gotchas and the update-from-git procedure.
- **[HERMES_INTEGRATION.md](HERMES_INTEGRATION.md)** — Hermes / OpenRouter agent
  → local proxy, with the `OPENROUTER_BASE_URL` routing lever.
