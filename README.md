# hwlock

A single-file, zero-dependency Python CLI for coordinating exclusive leases on shared physical test machines so multiple coding agents (or humans) don't flash, reboot, or benchmark on top of each other.

## Design

- **No daemon.** Lease state lives as one JSON file per machine in `~/.local/state/hwlock/` (plus an append-only `log.jsonl`), serialized with `flock` and atomic `os.replace`.
- **Self-renewing leases.** `hwlock ssh`, `hwlock run`, and `hwlock wait-up` heartbeat-renew your lease every 60s while running. If a process dies or forgets to release, the lease expires cleanly after its TTL (default `30m`, hard cap `8h`).
- **Host-key-pinned address probing.** Each machine can list multiple addresses (e.g. Ethernet, Wi-Fi, mDNS, VPN). `hwlock` tries the last working address first with `StrictHostKeyChecking=yes` and `HostKeyAlias` so a reassigned IP can never send commands to the wrong machine.
- **Explicit hardware state.** Releases record machine health (`ok`, `trial-boot-pending`, `down`, `needs-human`, `unknown`). `needs-human` blocks further checkouts until a human runs `hwlock clear`.
- **Quiet mode.** `acquire --quiet` blocks even read-only `hwlock peek` calls while a benchmark or power measurement is running.
- **Optional T3 Code integration.** When invoked from a [T3 Code](https://t3.codes) thread (`$T3CODE_THREAD_ID`), `--wait --notify` queues asynchronously and messages the thread once the lease is granted, and `hwlock sweep` (run on a 1-minute timer) nudges idle holders and auto-releases leases from settled or archived threads.

## Setup

1. Put `hwlock` on your `PATH`:

   ```bash
   install -m 755 hwlock ~/.local/bin/hwlock
   ```

2. Create `~/.config/hwlock/machines.json` based on [`machines.example.json`](machines.example.json), and ensure each machine's `host_key_alias` is in `~/.ssh/known_hosts`.

3. *(Optional)* Run `hwlock sweep` every minute (e.g. via a systemd user timer or cron) if you use `--notify` or idle-thread cleanup.

## Usage

```bash
# See who holds what
hwlock status

# Check out a machine (prints a lease token on stdout)
TOKEN=$(hwlock acquire lab-box-a --for "run kernel smoke test")

# Run commands under the lease (auto-renews every 60s)
hwlock ssh "$TOKEN" -- uname -a
hwlock run "$TOKEN" -- rsync -av -e "$HWLOCK_SSH" ./build/ "$HWLOCK_HOST:/tmp/build/"
hwlock wait-up "$TOKEN" --down-first

# Release when done
hwlock release "$TOKEN" --state ok --note "back on baseline build"

# Read-only check without a lease (blocked if holder passed --quiet)
hwlock peek lab-box-a -- dmesg | tail -n 50

# Human override commands (agents must pass --user-said "<words>")
hwlock break lab-box-a --note "reclaiming for urgent test"
hwlock clear lab-box-a --note "power-cycled"
```

## Environment variables

| Variable | Default | Purpose |
| --- | --- | --- |
| `HWLOCK_CONFIG` | `~/.config/hwlock/machines.json` | Machine inventory file |
| `HWLOCK_STATE` | `~/.local/state/hwlock` | Lease JSON files and `log.jsonl` |
| `HWLOCK_T3` | `~/.local/bin/t3` | Optional T3 CLI path for `sweep` / `--notify` |
| `HWLOCK_NUDGE_IDLE_S` | `300` | Seconds a T3 thread sits idle before a reminder |
| `HWLOCK_NUDGE_GRACE_S` | `600` | Grace period after a reminder before auto-release |
