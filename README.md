# mc-watcher

A lightweight Minecraft server log watcher daemon in a single POSIX shell script. Tails every active `mc-*` tmux session's `logs/latest.log` and fires notifications for joins, leaves, deaths, kicks, gamemode switches, and boot/stop events.

Borrowed from the [LAPTOO homelab](https://laptoo.xyz) (Alpine Linux / OpenRC).

> **yep this is vibecoded aswell**

## Requirements

- POSIX `sh` (dash, busybox ash, bash — anything)
- `tmux`, `dd`, `awk`, `sed`, `wc`, `tail`
- A `~/mc/<server>/logs/latest.log` layout with tmux sessions named `mc-<server>`
- Optional for desktop notifications: `notify-send` + `paplay` and a `hostenv`-style wrapper (see below)
- Optional for push: an `ntfy-push` wrapper (topic `mc`)

## Install

```sh
sudo install -m 755 mc-watcher /usr/local/bin/mc-watcher
sudo install -m 755 openrc/mc-watcher /etc/init.d/mc-watcher
sudo rc-update add mc-watcher default
sudo rc-service mc-watcher start
```

No config file — paths are variables at the top of the script (`LOG_DIR`, `SOUNDS_DIR`, `EVENTS_FILE`, `STATE_DIR`).

## How it works

- The main loop scans `tmux list-sessions` for `mc-*` and spawns **one background tail per server** (pidfiles in `/tmp/mc-watcher/watch-<srv>.pid`).
- Each tail tracks its log **by exact byte range** (`dd skip/count`), so it survives log rotation and never over-reads; a partial last line is held back for the next poll.
- Every event is appended to a JSONL events log (`{"ts","server","event",…}`) for dashboards to read.

### Notification triple

`notify()` fan-outs to, in parallel:

1. **Desktop** — `hostenv notify-send …` (GUI session)
2. **Sound** — `hostenv paplay <minecraft ogg>` (per-event sounds: levelup, pling, wither…)
3. **ntfy** — publish to the `mc` topic (markdown)

### `hostenv`

Desktop/sound delivery goes through a `hostenv` helper that locates the login session's `DBUS_SESSION_BUS_ADDRESS` and `PULSE_SERVER`, so the daemon can reach the user's GUI. If you don't need desktop notifications, just make sure `notify-send` isn't on `PATH`.

## Parsing notes (Paper + Fabric)

Server implementations log the same events in different orders and formats — all verified against real logs:

| Case | Paper | Fabric |
|---|---|---|
| Join | `joined the game` **then** `logged in with entity id … at ([world]x, y, z)` | `logged in … at (x, y, z)` → `Player [x] joined.` → `joined the game` |
| Leave | `lost connection: …` + `left the game` | `name (uuid) lost connection: …` + `left the game` |
| Stop | `Stopping the server` → `Stopping server` | same |
| Gamemode | `[name: Set own game mode to X Mode]` (Essentials) | `Set name's game mode to X Mode` (vanilla) |
| Named-mob death | `Villager Villager['Cleric'/7650, uuid='…', l='ServerLevel[world]', x=…, y=…, z=…, …] died, message: '…'` | — |

- **Join dedup is order-independent** (20s window): exactly one ping either way. On Paper the follow-up login only enriches the event log with the IP; on Fabric the login ping carries it.
- **Leave echo-suppress** (10s window): the `lost connection` + `left the game` pair yields one ping.
- **Names are cleaned** (`[ip]`/`(uuid)` suffixes stripped, whitespace trimmed) and empty names never notify.
- **Ready fires once per boot** (Fabric prints `Done` twice: vanilla + Geyser).
- **Death bodies are the death message only** (the title already says the server). Player chat containing death verbs (`<name> I fell off lol`) is explicitly skipped.
- **Named-mob deaths** are parsed into an organized multi-line body: death message / `uuid = …` / `x = …, y = …, z = …` / `level = …`.
- **Graceful-stop detection**: `Stopping the server` sets a flag (no ping); on session death you get `Server stopped`, or `Server stopped unexpectedly` when the session vanished with no stop line (crash/kill/reboot).
- AdvancedBan tempbans/kicks and GrimAC flags have their own handlers with a 5-min per-player dedup (self-pruning).
