# AGENTS.md

Guidance for AI agents working on this repository.

## What this is

`hc-notify` is a single-file Python CLI (`hc-notify`) that watches `hc` terminals through `hc events --json` and, when each watched terminal finishes, sends a templated message to Zulip and Raft. Configuration is a single TOML file shared by both channels.

Keep it a single script with no third-party Python dependencies. `tomllib` is the only non-trivial stdlib requirement (Python 3.11+).

## Files

- `hc-notify` — the entire implementation (argparse, config, event loop, template rendering, senders).
- `config.example.toml` — the config template users copy into `~/.config/herdr-core/herdr-notify.toml`.
- `README.md` — user-facing docs.

## Key design decisions

- **`hc events --json`, not `hc connect` and not N `hc wait` processes.** One process handles any number of targets and the event records already carry `state`, `reason`, `exit_code`, `created_ms`, `finished_ms`. `hc connect` is a full terminal-I/O protocol and is overkill here.
- **Two-stage event stream.** The first line of `hc events --json` is a full snapshot (`{channel, seq, terminals}`). Subsequent lines are JSON-RPC notifications (`{jsonrpc, method:"event", params:{terminal, type, data, seq}}`). `created` carries a full terminal record; `finished` carries only `{exit_code, finished_ms, reason, signal}`, so the watcher merges it into its cached record before rendering. The snapshot establishes the baseline (already-finished targets are skipped); `finished` events trigger notifications, deduplicated by `finished_ms`.
- **Target matching** prefers exact `address`, falling back to `name`. Names can collide across terminals, so addresses are safer.
- **Self-backgrounding.** Unless `--raw` is set (or env `HERDR_NOTIFY_SPAWNED=1` is present), the script re-executes itself as `hc run --bg --outlive-parent <python> <script> <args>`. The env var prevents infinite recursion. `--outlive-parent` keeps the watcher alive when the invoking terminal exits.
- **Two independent senders.** `send_zulip` posts to `{site}/api/v1/messages` with Basic auth; `send_raft` posts to `{server_url}/internal/agent-api/send` with `Authorization: Bearer <credential>` and a fresh `idempotencyKey`. Either can fail without blocking the other. Both return a dict with an optional `url` used by `--verbose`.
- **`{output}` is lazy.** Only templates containing `{output}` call `hc read <address>` (last 100 lines), so successful runs do not pay the cost.
- **Failures are aggregated.** Each failed channel is reported on stderr and the final exit code is 1 if any notification failed.

## Templates

`SafeFormatter` leaves unknown placeholders untouched instead of raising `KeyError`. Available variables are listed in `TEMPLATE_VARS` and in the README. Keep new variables in sync across the script, `--help` epilog, and README.

## Making changes

- Edit only `hc-notify` for behavior changes.
- Update `README.md` when adding or changing options, template variables, or behavior.
- Update `config.example.toml` if the config schema changes.
- Keep the docstring at the top of the script current — it is used as the `--help` epilog.

## Testing

```sh
python3 -m py_compile hc-notify
python3 hc-notify --help
```

To exercise the watcher without sending real messages, point `--config` at a TOML with placeholder credentials and watch a terminal that was already finished before start (it will be skipped and the watcher will exit cleanly). Do not use real credentials in tests — sending a message has side effects in Zulip and Raft.

Avoid running the script without `--raw` in tests: that starts a real background hc terminal.

## Raft API reference

Derived from `@botiverse/raft-sdk` (installed in `/x/try/raft/node_modules`):

- Endpoint: `POST {server_url}/internal/agent-api/send`
- Headers: `Authorization: Bearer <credential>`, `Content-Type: application/json`
- Body: `{"target": "#general", "content": "...", "idempotencyKey": "<uuid>"}`
- Channel join (`/internal/agent-api/channels/:channelId/join`) is omitted deliberately: the external-agent credential is typically already a member; the SDK also treats join as best-effort.
