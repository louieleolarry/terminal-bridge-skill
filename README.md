# Terminal Bridge Skill

OpenClaw/Claude skill instructions for using Terminal Bridge: a local, file-based command courier that lets a sandboxed assistant ask the user's real macOS shell to run allowlisted commands.

This repo contains the skill documentation only. The bridge runtime itself lives separately at:

```text
~/Sites/Claude/Projects/terminal-bridge
```

## Install

Copy `SKILL.md` into the OpenClaw skills directory:

```bash
mkdir -p ~/.openclaw/skills/terminal-bridge
cp SKILL.md ~/.openclaw/skills/terminal-bridge/SKILL.md
```

## Safety Notes

- The bridge is file-based, not a network shell.
- Commands are checked against `allowlist.json`.
- Denied commands should stop and ask the user before expanding the allowlist.
- Runtime folders such as `queue/`, `results/`, `logs/`, and local auth/config files do not belong in this repo.
