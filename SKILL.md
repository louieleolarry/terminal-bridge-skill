---
name: terminal-bridge
description: Run real shell commands on the user's macOS from the sandbox via the Terminal Bridge courier at ~/Sites/Claude/Projects/terminal-bridge. Use when the task requires reading actual files outside the sandbox mount, inspecting git state in a real repo, running pytest, executing a python3/node script under ~/Sites/, or any action where the sandbox's own Bash tool is insufficient because it can't see the user's filesystem. Triggers include phrases like "run on my mac", "check my real repo", "run the test suite", "what's in my Sites folder", "git status on the actual repo", "use the bridge", "courier this", or any request to execute a command whose effects must happen on the user's machine rather than in the ephemeral sandbox.
---

# Terminal Bridge

A file-based courier that lets Claude execute commands on Louie Larry's real Mac
shell from the sandbox. Claude writes a JSON command file into `queue/`; a
Python LaunchAgent (`com.terminalbridge.courier`) on the Mac watches that
directory, executes against `allowlist.json`, and writes the result into
`results/`. No network, no remote shell — just a shared folder.

Built and verified end-to-end on **2026-05-09**. This skill documents the
contract; the implementation lives at
`/Users/louieleolarry/Sites/Claude/Projects/terminal-bridge/`.

## When to reach for the bridge

Use it when the task **requires** the user's real filesystem or process state:

- Reading or writing files under `~/Sites/` that aren't on the sandbox mount.
- Inspecting git state of a real repo (`git status`, `git log`, `git diff`).
- Running tests (`pytest`) or scripts (`python3 ~/Sites/.../foo.py`,
  `node ~/Sites/.../foo.js`) against the user's actual code.
- Verifying a LaunchAgent is loaded, checking what's in a config the user
  edited outside the session, etc.

Do **not** use it for:

- Anything the sandbox's own Bash tool can already do (don't go through the
  bridge to run arithmetic, parse JSON, hit a public URL, etc.).
- Sensitive actions outside the allowlist (network, package installs,
  `~/.ssh`, keychain). Ask the user first; widening the allowlist is a
  major system change.

## Canonical invocation

The bridge entrypoint exists at the same path on the Mac and inside the
sandbox mount, so a single command works from either side:

```
python3 "/sessions/<session>/mnt/terminal-bridge/bridge.py" --json -- <argv...>
```

Common forms:

```
# Run a command, get full result JSON (preferred — you see status + matched_rule)
python3 "<bridge>/bridge.py" --json -- git status

# Run inside a specific repo
python3 "<bridge>/bridge.py" --cwd /Users/louieleolarry/Sites/<repo> --json -- git log --oneline -5

# Hard timeout override (default 60s)
python3 "<bridge>/bridge.py" --timeout 300 --json -- pytest -x

# Force per-command approval (courier waits for queue/<id>.approve)
python3 "<bridge>/bridge.py" --require-approval --json -- <command>
```

`--` is required to separate bridge flags from the command's own argv.

`--json` is strongly preferred over the default text output: you get the full
result envelope (`status`, `exit_code`, `stdout`, `stderr`, `truncated`,
`matched_rule`, `error`) so you can reason about failure modes instead of
guessing from a non-zero exit.

## Result envelope — what to check

Every result file at `results/<id>.result.json` has a `status` field. Decide
what to do based on it:

| status      | meaning                                   | what to do                                          |
|-------------|-------------------------------------------|-----------------------------------------------------|
| `completed` | process exited; `exit_code` is the real code | Treat like any normal command — non-zero is a real failure |
| `denied`    | allowlist refused the command             | **Stop. Ask the user before widening the allowlist.** |
| `timeout`   | killed at `timeout_sec`                   | Show partial output, ask whether to retry with longer `--timeout` |
| `error`     | courier could not execute                 | Show `error` field, check whether courier is running |

Also check `truncated`: if `true`, output exceeded `max_output_bytes` (1 MB
default). Re-run with a narrower scope or ask the user to bump the cap.

## What the default allowlist permits

Read this as a ceiling, not a floor — anything outside it returns `denied`
and pops a macOS notification banner.

- **Read-only inspection:** `ls`, `cat`, `head`, `tail`, `pwd`, `echo`,
  `date`, `which`, `file`, `stat`, `wc`, `grep`, `find`, `tree`, `diff`,
  `sort`, `uniq`, `awk`, `sed -n`.
- **Git read-only:** `status`, `log`, `diff`, `branch`, `show`, `fetch`,
  `remote`, `rev-parse`, `ls-files`, `blame`, `config --list`, `describe`,
  `tag`/`tag -l`, `stash list`, `reflog`, `shortlog`, `whatchanged`, `grep`.
- **Git writes (history-safe):** `add`, `commit`, `stash` / `stash push`,
  `checkout`, `switch`, `merge --no-commit`, `merge --abort`, `tag <name>`,
  `cherry-pick`, `revert`, `restore` / `restore --staged`, `mv`, `rm`.
  **NOT allowed:** `push`, `pull`, `rebase`, `reset`, `--force` of any kind.
- **Script runtimes (paths under `~/Sites/` only):** `python3 -m <module>`
  or `python3 <abs path under Sites>.py`; `node <abs path under Sites>.{js,mjs,cjs}`.
- **Package managers (read/test only):** `npm run|test|ci|ls|view|outdated|exec|init|version`;
  `pip3 list|show|freeze|check|index`; `pytest` (full).
  **NOT allowed:** `npm install`/`uninstall`/`publish`, `pip3 install`.
- **Limited file management (paths under `~/Sites/`, `~/.openclaw/skills/`, or `~/.openclaw/workspace/`):**
  `mkdir -p`, `touch`, `mv` (both ends under one of those three trees),
  `cp` / `cp -r` (both ends under one of those three trees).

The allowlist file (`allowlist.json`) supports JSONC comments and the
courier reloads it on every command — edits take effect immediately, no
restart.

## Codex Sandbox Fallback

Use Terminal Bridge when Codex's shell cannot reach the real host
environment, even though the user's Mac can. Common signals include:

- `ssh: connect ... Operation not permitted`
- `Could not resolve host`
- Files or repos visible on the Mac but missing from the sandbox mount.
- Commands that must run from the user's real `~/Sites` checkout.

Treat the bridge result as host-shell evidence, and treat the sandbox error as
sandbox evidence. Do not infer that the remote host, repo, DNS, or SSH config is
broken until the same check has been run through the bridge or the user's real
terminal.

Recommended flow:

1. Run the cheap sandbox check first when useful.
2. If it fails with a sandbox-denial or sandbox-network symptom, retry the same
   narrow command through Terminal Bridge.
3. Compare the two results explicitly in the final answer.
4. Keep all bridge commands allowlisted and ask before widening the allowlist.

## Failure handling

Decision tree when something goes wrong:

1. **Result times out at the bridge level** (no `results/<id>.result.json`
   appears within `--wait-timeout`, default 180s):
   - The courier likely isn't running. Tell the user.
   - Recovery: `launchctl list | grep terminalbridge` on the Mac, then
     `launchctl unload && launchctl load ~/Library/LaunchAgents/com.terminalbridge.courier.plist`
     or rerun `install_launchagent.sh`.
   - Do NOT try to invoke the courier yourself — it's a per-user LaunchAgent
     and the user is the one who owns its lifecycle.

2. **`status: "denied"`:**
   - Show the user the `error` field (it includes which rule was sought).
   - Ask before editing `allowlist.json`. The exclusions
     (`git push/pull/rebase/reset`, `npm install`, `curl`, wildcards,
     anything outside `~/Sites/`) were deliberate.
   - Never add a wildcard rule (`{"name": "*"}`) — that turns the bridge
     into a remote shell and nullifies the safety model.

3. **`status: "timeout"`:**
   - Show partial stdout/stderr.
   - Ask whether to retry with a longer `--timeout`. Don't silently bump it.

4. **`status: "error"`:**
   - Show the `error` string. Common causes: malformed payload, missing
     binary, bad `cwd`. Fix and retry; if cause is unclear, surface to the
     user.

## When to ask the user vs proceed autonomously

Standing user preference:
> "Figure things out and provide updates rather than requests. Always ask
> before performing or making any major system or company site changes."

For the bridge that means:

- **Proceed without asking:** any command that's already in the allowlist
  and confined to `~/Sites/`. That's the whole point of the allowlist —
  it's the user's pre-authorization in machine-readable form.
- **Ask first:** anything that returns `denied`, anything that would write
  outside `~/Sites/`, anything that touches the network, any package
  install, anything that rewrites git history, and any change to
  `allowlist.json` itself or to `com.terminalbridge.courier.plist`.
- **Use `--require-approval` when in doubt:** the courier will block
  until the user `touch queue/<id>.approve`s the command. That's the
  middle ground when an action is in-allowlist but high-impact.

## Hard limits — falls back to ask-first

Even with an allowlist match, stop and ask before:

- Editing `allowlist.json` to add a new rule (especially network, package
  install, or wildcard rules).
- Running anything that mutates state outside the three permitted trees
  (`~/Sites/`, `~/.openclaw/skills/`, `~/.openclaw/workspace/`) — paths
  under `~/.ssh/`, `~/.claude/`, the rest of `~/.openclaw/` (auth files
  live there: `openclaw.json`, `agents/main/agent/auth-profiles.json`),
  system Library, anything in `/etc`.
- Modifying or unloading `com.terminalbridge.courier.plist`.
- Issuing a long-running command that could lock a repo (e.g. an interactive
  rebase or a test suite that holds a port). Use `--timeout` aggressively.
- Chaining multiple write commands without showing the user the plan first.

## Verifying it's healthy

Cheap pre-flight when starting a new bridge-heavy session:

```
python3 "<bridge>/bridge.py" --json -- echo ping
```

A healthy courier returns `status: "completed"`, `stdout: "ping\n"` in
~10ms. Anything else (timeout, denied, error) means the bridge isn't ready
for the real task.

## Pointers

- Bridge code & docs: `/Users/louieleolarry/Sites/Claude/Projects/terminal-bridge/`
- Protocol spec: `<bridge>/PROTOCOL.md`
- Audit log: `<bridge>/logs/courier.log` (append-only; every picked-up,
  denied, completed, and timed-out command is recorded)
- LaunchAgent plist: `~/Library/LaunchAgents/com.terminalbridge.courier.plist`
- Origin session (built and verified end-to-end): `9100dbeb-3659-4ccc-85d0-11272ea8f032`
