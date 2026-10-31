---
name: hermes-env-doctor
description: Diagnose and repair Hermes Agent's own environment — stuck or failed startup, broken updates, missing or wrong-interpreter dependencies, invalid config, stale processes, and MCP servers that hang or fail to connect (including Windows-hosted MCP servers reached from WSL). Use whenever Hermes won't start cleanly, tools are missing after startup, or right after adding a new MCP server, skill, or dependency.
version: 1.1.0
author: Smarty & Max
license: MIT
platforms: [linux, windows, macos]
metadata:
  hermes:
    tags: [troubleshooting, repair, environment, dependencies, mcp, wsl, maintenance]
    category: maintenance
    related_skills: []
---

# Hermes Env Doctor

You are Hermes Agent, and something in your own environment is broken. This skill
turns you into your own mechanic: update yourself, find what is failing, repair it
in small verified steps, and report exactly what changed.

Work through the phases in order. Most startup failures are caused by one bad
MCP server entry, one missing dependency, or one stale process — find it by
isolation, not by reinstalling everything.

## Ground rules

1. **Back up before you touch.** Before editing any config, copy it with a
   timestamp: `cp ~/.hermes/config.yaml ~/.hermes/config.yaml.bak-$(date +%Y%m%d-%H%M%S)`.
   Same for `.env` and any file you are about to change. Never edit the only copy.
2. **One change at a time, then verify.** Change one thing, restart or reload,
   check the result. If you change three things at once you will never know
   which one fixed it — or broke it.
3. **Never delete user data.** Memory, sessions, skills, and config are the
   user's. Repair around them. A full reinstall is the last resort and needs
   the user's explicit approval first.
4. **Never print secrets.** API keys and tokens may exist in config and `.env`.
   Check whether a key is *set*, never echo its value into the conversation
   or a report.
5. **Two strikes rule.** If the same repair fails twice, stop. Report what you
   tried, what the error says, and what you think the options are. Do not
   thrash — a stuck repair reported clearly beats ten silent retries.
6. **Report as you go.** If a step will take a while (first-run downloads can
   take minutes), say so before starting it, not after the user asks.
7. **The report is an output, not an input.** Never ask the user to run a
   command and paste its output back to you when you can run the command
   yourself — `hermes doctor`, version checks, log reads, config reads, and
   server test launches are all yours to run in your own terminal. The user
   is never a courier for diagnostics. The only report in this workflow is
   the one you write at the end (Phase 9): what was wrong, and what you did.
   If something genuinely needs the user's hands, eyes, or a password prompt,
   say exactly that — it is the exception, and it should feel like one.

## Level 0 — if Hermes cannot start at all (for the human reading this file)

A skill runs inside Hermes, so if Hermes itself will not start, the user must
run this rescue by hand in a plain terminal first. Keep it short:

```bash
cp ~/.hermes/config.yaml ~/.hermes/config.yaml.bak-manual   # backup
# Temporarily comment out the whole mcp_servers: block in config.yaml, then:
hermes            # if it starts now, an MCP entry was the blockage — do Phase 4
hermes doctor     # built-in health check
hermes update     # bring Hermes itself current
```

If Hermes starts with MCP disabled, re-enable **one** server at a time with a
full restart between each (Phase 4). The last one added before the hang is the
prime suspect.

## Phase 1 — Snapshot (touch nothing yet)

Collect the facts first; half of all "mystery" failures are visible here.

- `hermes doctor` — the built-in health check. Run it yourself, now, in your
  own terminal (Ground Rule 7) and keep its full output: it is your baseline,
  your work list, and your final comparison. Work the findings yourself,
  top to bottom; do not hand the list back to the user as homework.
- OS and shell: `uname -a`, and on WSL confirm it: `grep -i microsoft /proc/version`.
- Runtimes: `python3 --version`, `node --version`, `uv --version` (whichever exist).
- Disk and memory headroom: `df -h ~`, `free -h`. A full disk imitates every
  other failure.
- Leftover processes: `ps aux | grep -i hermes` — a zombie Hermes or gateway
  from a crashed session can hold ports and locks. Note them; kill only ones
  you are sure are stale, and tell the user you did.
- What changed recently? Ask (or infer from file timestamps): a just-added MCP
  server, a config edit, an OS update, a moved folder. The newest change is
  the first suspect.

## Phase 2 — Update everything, in order

1. **Hermes itself:** `hermes update`. Afterwards `source ~/.bashrc` (or open a
   fresh terminal) so the shell picks up any PATH changes, then `hermes doctor`.
2. **The launchers:** `uv self update` if `uv` exists. Node/`npx`-based servers
   fetch fresh copies on their own.
3. **MCP server packages:** stdio servers started via `uvx <package>` or
   `npx -y <package>` normally resolve the latest version per launch — but a
   *stuck* startup is usually the first-run download, not an old version.
   Pre-warm each one by hand (Phase 4, step 3) so downloads finish outside
   Hermes' connection timeout.
4. Re-run `hermes doctor` and compare with the Phase 1 baseline.

## Phase 3 — Built-in diagnostics

- `hermes doctor --fix` — let it auto-repair what it knows how to repair
  (missing plugins, missing keys prompts, stale state). Read every line it
  prints; note anything it could not fix for Phase 4–7.
- If the `computer_use` toolset is in play: `hermes computer-use status` and
  `hermes computer-use doctor`.
- Read the startup logs from the failed run, not just the live error. Look for
  the *first* error in the log, not the last — later errors are usually echoes.

## Phase 4 — The MCP isolation loop (the core procedure)

MCP servers are launched by Hermes at startup and discovered over stdio or
HTTP. A server that is slow, missing, or crashing can hang the whole startup.
Isolate them:

1. **Disable all.** Comment out the entire `mcp_servers:` block in
   `~/.hermes/config.yaml` (you have a backup from the Ground Rules). Start
   Hermes. If it starts cleanly, the core is healthy and the fault is in one
   server entry. If it *still* fails, skip to Phase 7 (config/core problems).
2. **Re-enable one server at a time**, with a **full Hermes restart** after
   each — a reload is not always equivalent, and a skipped restart is a
   classic false diagnosis. After each start, confirm the server's tools
   actually appeared (they are prefixed `mcp_<server-name>_*`) before adding
   the next server.
3. **Pre-warm the failing server by hand.** Copy its exact `command` + `args`
   from the config and run them in a terminal yourself:
   - A healthy stdio server prints little or nothing and then *waits silently*
     — that silence is success. Let it sit for ~15 seconds, then Ctrl+C.
   - First run via `uvx`/`npx` downloads the package (and possibly a Python
     runtime). This can take minutes. Let it finish once, by hand, so Hermes
     never has to wait on it.
   - If it crashes, the error is now on *your* screen instead of buried in a
     log. Fix that error (usually Phase 5 or 6), then re-test by hand.
4. **Give first-run servers room.** For any server launched via `uvx`/`npx`,
   set `connect_timeout: 300` in its config entry. The default timeout is
   sized for warm starts, not downloads.
5. Only declare a server fixed when Hermes starts cleanly with it enabled
   **and** its prefixed tools are listed **and** one read-only tool call
   succeeds (list something, take a screenshot, read a status — no side
   effects).

## Phase 5 — Command-not-found and PATH repairs

The single most common MCP failure: the config names a command the launching
environment cannot find.

- Check resolution in the *same* shell Hermes runs in: `which <command>`
  (Linux/macOS) — an interactive shell finding it is not proof; PATH differs
  by context.
- **WSL → Windows binaries:** WSL interop runs Windows programs, but the
  name usually needs its extension: `uvx.exe`, `python.exe`, `powershell.exe`.
  Verify interop itself works first: `powershell.exe -NoProfile -Command "echo ok"`.
  If that fails, check `/etc/wsl.conf` has `[interop] enabled=true`, and from
  Windows run `wsl --shutdown`, then reopen the terminal.
- Windows PATH entries reach WSL only if `appendWindowsPath` is on, and only
  after a WSL restart. When in doubt, stop depending on PATH: use the full
  path in the config (find it on the Windows side with `where.exe <name>` in
  PowerShell — e.g. uv lands in `C:\Users\<name>\.local\bin\uvx.exe`).
- **The PowerShell bridge (known-good pattern for Windows-hosted servers
  launched from WSL):** if a direct `.exe` entry misbehaves, launch through
  PowerShell with the full Windows path instead:

  ```yaml
  mcp_servers:
    windows:
      command: "powershell.exe"
      args: ["-Command", "C:\\Users\\<name>\\.local\\bin\\uvx.exe windows-mcp serve"]
      connect_timeout: 300
  ```

  Note the `serve` argument — omitting the server's run subcommand is an
  easy, silent way to launch nothing at all.
- Keep Windows-hosted code on a Windows drive (`C:\...`). Do not point a
  Windows interpreter at paths inside the WSL filesystem (`\\wsl$\...`) —
  it is slow and fragile.

## Phase 6 — Dependency repairs (the right interpreter rule)

A dependency only counts as installed if it is installed **in the interpreter
that launches the server**.

- For each Python server, identify its launcher (system `python`, `python.exe`,
  a `uvx` package, a repo's `server.py`) and test imports with that exact
  interpreter, e.g.:
  - `python.exe -c "import win32com.client"` (COM-bridge servers on Windows)
  - `python.exe -c "import mcp, pydantic"`
- If an import fails, install into that interpreter: `python.exe -m pip install <packages>`
  (use `-m pip` so the packages land in the interpreter you just tested).
- COM/automation servers (CAD, Office): the target application usually must
  be installed, registered (launched at least once as the user), and often
  already open before the server can connect. Check the server's own config
  file too (e.g. CAD-MCP's `src/config.json` must name the installed product).
- Version floors are real: check the server's README for minimum Python/Node
  versions before assuming corruption — an old interpreter fails politely
  and looks like a bug.

## Phase 7 — Config and core repairs

- **Validate the YAML** before blaming anything else:
  `python3 -c "import yaml,sys; yaml.safe_load(open('$HOME/.hermes/config.yaml'))"`.
  Indentation errors in `mcp_servers` are the top cause of "Hermes ignores my
  server" *and* of startup crashes. Fix indentation; do not rewrite the file.
- If the config is mangled beyond a small fix, restore the newest backup from
  the Ground Rules, restart, and re-apply the user's intended changes one at
  a time.
- **Stale state:** if startup hangs with MCP disabled, look for leftover
  processes (Phase 1) and port conflicts if the gateway is used; kill only
  confirmed-stale Hermes processes.
- **Reinstall (last resort, needs user approval):** re-run the official
  installer over the existing install — it repairs the runtime without
  wiping `~/.hermes` data. Say explicitly what will and will not be touched
  before running it.

## Phase 8 — Verification suite (all must pass)

1. Hermes starts cleanly from a fresh terminal, with all intended MCP
   servers enabled, in reasonable time.
2. `hermes doctor` output is no worse than the Phase 1 baseline — ideally
   better — and `--fix` has nothing left it can repair.
3. Every MCP server's prefixed tools are visible.
4. One **read-only** call per server succeeds (a screenshot, a list, a
   status). Only after that, one small reversible real action if the user
   wants it (e.g. draw one small shape in a scratch document).
5. A second restart also succeeds — a fix that only works once is not a fix.

## Phase 9 — Report and rollback

Finish with a short report the user can keep:

- **Symptom:** what was failing, in one line.
- **Cause:** what you found, with the evidence (the log line, the failed
  import, the missing path).
- **Fixes applied:** each change, one line each, in the order applied.
- **Backups:** the exact backup filenames, and the one command that undoes
  everything: restore the backup config and restart.
- **Leftovers:** anything still broken or untested, stated plainly.

## When to stop and ask the user

- Before any reinstall, deletion, or killing a process you cannot identify.
- Before changing the model/provider or any credential.
- When a repair needs something only the user has (a password prompt, a UAC
  dialog, a physical look at the screen).
- When the same step has failed twice (Ground Rule 5).

A clear "I'm stuck here, and here's exactly where" is a successful outcome of
this skill. A silently half-repaired environment is not.
