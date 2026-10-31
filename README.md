# hermes-env-doctor

A skill for [Hermes Agent](https://github.com/NousResearch/hermes-agent) that turns
Hermes into its own mechanic: when Hermes is stuck starting up, an update broke
something, a dependency is missing, or an MCP server hangs the boot, Hermes runs
this playbook to update itself, isolate the fault (usually one bad MCP entry),
repair it in small verified steps, and report what changed — with backups before
every edit and a two-strikes stop rule.

## Install

Copy the folder into your Hermes skills directory:

```
~/.hermes/skills/hermes-env-doctor/SKILL.md
```

From WSL/Linux:

```bash
mkdir -p ~/.hermes/skills/hermes-env-doctor
cp SKILL.md ~/.hermes/skills/hermes-env-doctor/
```

Then restart Hermes (or run `/reload-skills` in a session) and invoke it:

```
/hermes-env-doctor
```

or just tell Hermes: *"Run the hermes-env-doctor skill — startup is stuck."*

## If Hermes won't start at all

A skill can't run inside a Hermes that never boots. Open `SKILL.md` and follow
**Level 0** by hand (back up the config, comment out `mcp_servers`, start clean,
re-enable one server at a time). Once Hermes boots, let the skill take over.

## What it covers

- `hermes update` + `hermes doctor --fix` workflow
- The MCP isolation loop: disable all → re-enable one by one → pre-warm
  first-run downloads (`uvx`/`npx`) → generous `connect_timeout`
- PATH and command-not-found repairs, including WSL → Windows interop
  (`.exe` names, full paths, the PowerShell-bridge launch pattern)
- The right-interpreter rule for Python dependencies (incl. pywin32/COM
  servers such as CAD-MCP)
- Config YAML validation and backup/restore
- A final verification suite and a plain-language repair report

## Sharing

The skill is a single self-contained `SKILL.md` (MIT). Share the file, or put it
in a GitHub repo so others can install it with `hermes skills install`.
Improvements welcome — especially new entries for the failure catalogue.
