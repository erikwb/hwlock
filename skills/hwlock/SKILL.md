---
name: hwlock
description: Coordinate exclusive access to shared physical test machines using the `hwlock` CLI. Use whenever running SSH commands, deploying builds, rebooting, benchmarking, or checking status on any shared lab/test host managed by hwlock.
---

# Shared Hardware Leases (`hwlock`)

All access to shared test machines goes through `hwlock`. Never bypass it with raw `ssh`, `scp`, or `rsync`.

## Rules

1. **Check status first:** Run `hwlock status` to see which machines are free, held, or blocked.
2. **Acquire before changing anything:**
   ```bash
   TOKEN=$(hwlock acquire MACHINE --for "<concise task description>")
   ```
   - Pass `--quiet` when running benchmarks, timing, or power measurements so nobody else can even `peek`.
   - Keep the printed token (`MACHINE.xxxxxxxx`); it is your lease handle.
3. **Run all work through `hwlock`:**
   - Remote command: `hwlock ssh "$TOKEN" -- <cmd>`
   - Local script/rsync: `hwlock run "$TOKEN" -- rsync -av -e "$HWLOCK_SSH" ./out/ "$HWLOCK_HOST:/tmp/out/"`
   - Reboot wait: `hwlock wait-up "$TOKEN" --down-first` (do not write your own `sleep`/SSH polling loop)
   - These auto-renew the lease every 60s while running.
4. **Read-only inspection without a lease:** Use `hwlock peek MACHINE -- <read-only cmd>`. Never mutate state via `peek`. If it exits `2`, a `--quiet` measurement is running—wait.
5. **Release promptly—even on failure:**
   ```bash
   hwlock release "$TOKEN" --state ok|trial-boot-pending|down|needs-human --note "<exact state left behind>"
   ```
   - Never end your turn while holding a lease unless you are intentionally waiting on the user; if you are, run `hwlock renew "$TOKEN" --idle-ok` first.
   - If the machine needs physical intervention (power cycle, console recovery, hung boot), release with `--state needs-human` and explain what happened in `--note`.
6. **If the machine is busy (exit `2`):** Do not touch it.
   - In a T3 Code thread (`$T3CODE_THREAD_ID` set): queue asynchronously with `hwlock acquire MACHINE --for "..." --wait 1h --notify` (exits `6`), then end your turn or do other work until the token arrives as a message. Run `hwlock dequeue MACHINE` if you no longer need it.
   - Outside T3: use `hwlock acquire MACHINE --for "..." --wait 1h` in a background task rather than polling in a loop.
7. **Inspect dirty handoffs before changing anything:** If `acquire` warns that you took over an expired lease, or the state is `unknown` / `trial-boot-pending`, check `uname -a`, running processes, and boot state before flashing or modifying the machine.
8. **Human-only overrides:** Never run `hwlock break` or `hwlock clear` unless the user explicitly tells you to in the current conversation. When authorized, pass their words via `--user-said "<their words>"`.

## Exit Codes

- `0` — Success
- `2` — Busy (held or queued; do not touch the machine)
- `3` — `needs-human` (stop and tell the user; do not `clear` on your own)
- `4` — Unreachable or timed out (lease is kept; diagnose or release as `down` / `needs-human`)
- `5` — Lease lost or invalid token (**stop modifying the machine immediately**)
- `6` — Queued with `--notify` (not an error; wait for the grant message)
