# hc-notify

Watch one or more [hc](https://github.com/) terminals and send a notification to **both Zulip and Raft** as soon as each terminal finishes.

By default the watcher re-launches itself inside a background hc terminal (`hc run --bg`). Use `--raw` to run it in the foreground.

## Requirements

- Python 3.11+ (`tomllib`)
- The `hc` CLI on `PATH`

## Install

```sh
git clone https://github.com/e06/hc-notify.git
cd hc-notify
chmod +x hc-notify
ln -s "$PWD/hc-notify" /usr/local/bin/hc-notify   # optional
```

## Configuration

Config lives in one TOML file, default:

```
~/.config/herdr-core/herdr-notify.toml
```

Copy the example and fill in the credentials:

```sh
mkdir -p ~/.config/herdr-core
cp config.example.toml ~/.config/herdr-core/herdr-notify.toml
$EDITOR ~/.config/herdr-core/herdr-notify.toml
```

```toml
[zulip]
site = "https://iso.zulipchat.com"
email = "seei-bot@iso.zulipchat.com"
key = "<your-zulip-api-key>"
channel = "so"
topic = "hc-notify"

[raft]
credential = "<your-sk_agent-credential>"
server_url = "https://api.raft.build"
channel = "#general"
```

Use `--config PATH` to point at a different file. Environment variables override the TOML values:

- Zulip: `ZULIP_SITE`, `ZULIP_EMAIL`, `ZULIP_API_KEY`, `ZULIP_CHANNEL`, `ZULIP_TOPIC`
- Raft: `RAFT_AGENT_CREDENTIAL`, `RAFT_SERVER_URL`, `RAFT_CHANNEL`

## Usage

```sh
# Watch one terminal; default template on success, error template on failure
hc-notify t5

# Watch several; each finishing terminal produces its own notification
hc-notify t5 t6 t7

# Custom success template
hc-notify t5 -m "job {name} finished in {duration} (exit {exit_code})"

# Custom error template; {output} is filled from `hc read <terminal>`
hc-notify t7 -e "job {address} FAILED:\n{output}"

# Bound the runtime, print message URLs, and run in the foreground
hc-notify t5 --timeout 1h30m --verbose --raw
```

### Options

| Option | Meaning |
| --- | --- |
| `terminals` | One or more terminal addresses (`t5`) or names. Prefer addresses: names may collide. |
| `-m`, `--message` | Template for a successful finish (`exit_code` 0). |
| `-e`, `--error-message` | Template for a failed finish. `{output}` triggers an `hc read`. |
| `--config` | Config file path (default above). |
| `--timeout` | Max runtime: `30s`, `5m30s`, `1h30m20s`. On timeout the watcher exits. |
| `--raw` | Run in the foreground instead of `hc run --bg`. |
| `--verbose` | Print the URL returned by each channel after a successful send. |

### Template variables

| Variable | Value |
| --- | --- |
| `{address}` | Terminal address, e.g. `t5` |
| `{name}` | Terminal name, if any |
| `{cwd}` | Terminal working directory |
| `{argv}` | Terminal command, space-joined |
| `{exit_code}` | Process exit code |
| `{reason}` | Finish reason, e.g. `exited` |
| `{output}` | Last 100 lines from `hc read <address>` (read only when the template uses it) |
| `{start_time}` | Terminal creation time (`YYYY-MM-DD HH:MM:SS`) |
| `{end_time}` | Terminal finish time |
| `{duration}` | Actual runtime, e.g. `5m30s` / `1h30m20s` |
| `{hostname}` | Local hostname |

Unknown placeholders are left as-is (`{typo}` stays `{typo}`) rather than crashing the watcher.

## Behavior

- Notifications go to **both** Zulip and Raft. A failure in one channel does not prevent the other; failures are reported on stderr and make the process exit non-zero when it finishes.
- The watcher uses `hc events --json` (one process, any number of terminals), not a polling loop of `hc wait` processes.
- A terminal that was already finished when the watcher started is recorded but **not** notified, to avoid duplicate alerts.
- `--timeout` limits the watcher itself; `{duration}` reports the watched terminal's actual runtime. These are independent.
- In background mode the watcher is started with `hc run --bg --outlive-parent` and an internal env var prevents it from re-spawning itself.
