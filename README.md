# mc-watcher

A lightweight Minecraft server log watcher daemon in a single POSIX shell script. Tails every active `mc-*` tmux session's `logs/latest.log` and fires notifications for joins, leaves, deaths, kicks, gamemode switches, and boot/stop events.

> **yep this is vibecoded aswell**

> **Built for one homelab.** This is written for and tested only on `laptoo`
> (Alpine Linux, OpenRC, `~/mc/<server>`, my own paths). It can be adapted to
> other systems — see [Portability](#portability) — but it is not portable
> as-is, and it has no CI, no test suite and no security audit. Read the source
> before running it anywhere that matters.

> ### ⚠️ Deliberately not made configurable
>
> Unlike [`mc-go`](https://github.com/laptooxz/mc-go), this project **will not
> get a config file, a `mc-watcher config` command, or a round of portability
> work.** I looked at doing it and decided against it: the script is one big
> pile of interleaved state — per-server tail pidfiles, byte-offset cursors,
> dedup windows, log-parsing branches — and threading a config layer through
> all of it would have touched nearly every line for no benefit to me. It was
> too bulky and too complex to be worth optimising for portability or
> customizability.
>
> **What that means in practice:** the handful of paths at the top of the
> script are meant to be edited by hand (or patched with `sed`), not
> configured. If you need it somewhere else, fork it and change the variables —
> don't expect upstream to grow options. `mc-go` is the one that got the
> proper treatment because its layout was small enough to factor out cleanly.

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

No config file — by design (see the notice at the top). Paths are variables at
the top of the script (`LOG_DIR`, `SOUNDS_DIR`, `EVENTS_FILE`, `STATE_DIR`).

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

**`hostenv` and `ntfy-push` are not part of this repo.** They live in my
dotfiles: `hostenv` wraps a command with the desktop session's environment,
and `ntfy-push <topic> <title> <message> [priority] [markdown]` publishes to
my local ntfy. Either provide equivalents, or drop them — notifications fail
silently (both are `&`/`command -v` guarded), so the watcher still runs.

## Portability

> **This section is a map, not an invitation.** See the notice at the top:
> portability was explicitly descoped for this project. The variables below are
> where a fork would start, not a supported configuration surface.

Paths and assumptions are variables at the top of `mc-watcher`
(`LOG_DIR`, `WATCHER_LOG`, `SOUNDS_DIR`, `EVENTS_FILE`, `STATE_DIR`), but
`HOME` and `PATH` are also hardcoded (`export HOME=/home/main`). The script
additionally assumes:

- servers live in `~/mc/<server>` and tmux sessions are named `mc-<server>`
- logs are at `<server>/logs/latest.log`
- Minecraft sounds are in `~/mc/library/sounds`
- the events log goes to `/srv/logs/mc_events.jsonl`
- `openrc/mc-watcher` is an OpenRC `supervise-daemon` wrapper

Change those variables (and write a unit for your init system) to run it
elsewhere. The log parsers themselves are Paper/Fabric-specific but not
distro-specific.

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
