# Claude Code permission setup for splinter

This directory holds the project-scoped Claude Code configuration. The notes
below record which permission-mode settings apply at which scope, because two
of the three common patterns silently do nothing when put in the wrong file.
Each claim was checked against the Claude Code docs
([permission modes](https://code.claude.com/docs/en/permission-modes),
[headless mode](https://code.claude.com/docs/en/headless),
[settings](https://code.claude.com/docs/en/settings)) and against
`claude 2.1.294` on 2026-10-08.

## What this repository ships

| File | Purpose |
| --- | --- |
| `.claude/settings.json` | Shared `permissions.allow` rules for the repo's deterministic checks. Applies in every mode except `bypassPermissions`, where allow rules are ignored. |
| `bin/claude-check` | Non-interactive runner: `claude -p` in `dontAsk` mode with an exact allowlist. Use it in CI or scripts. |
| `.claude/skills/new-lint/` | The `/new-lint` skill (unchanged). |

`.claude/settings.json` deliberately does **not** set `permissions.defaultMode`.
See pattern 1 for why.

## Pattern 1: start interactive sessions in auto mode

```json
// ~/.claude/settings.json (per user) or managed settings (per organization)
{
  "permissions": {
    "defaultMode": "auto"
  }
}
```

Verified facts:

- `"auto"` and `"bypassPermissions"` only take effect from `~/.claude/settings.json`,
  `--settings`, or managed settings. Set in a project `.claude/settings.json` or
  `.claude/settings.local.json`, the value is ignored and the session falls back to
  the built-in default. That is why this repo's settings file carries no `defaultMode`.
- On Claude Code v2.1.283 or later, auto mode is already the built-in starting mode
  for interactive terminal and VS Code sessions, so pattern 1 mostly matters on older
  versions or when another file overrides it.
- `defaultMode` does not lock the mode. The user can still press `Shift+Tab`; from
  `auto` the cycle runs `default` -> `acceptEdits` -> `plan` -> back to `auto`.
- The only official locks are in managed settings: `permissions.disableAutoMode:
  "disable"` removes `auto` from the cycle, and
  `permissions.disableBypassPermissionsMode: "disable"` removes `bypassPermissions`.
  There is no setting that pins a session to a single mode.

## Pattern 2: CI and scripts, approved tools only (`dontAsk`)

```bash
claude -p "run the test suite" \
  --permission-mode dontAsk \
  --allowedTools "Bash(npm test)" "Read"
```

`bin/claude-check` is this pattern adapted to splinter. It adds `--bare` (no
hooks, plugins, MCP servers, or `CLAUDE.md`, so the run is identical on every
machine) and `--permission-prompts none` (never wait for an answer nobody will
give).

Verified facts, including two that are easy to get wrong:

- `dontAsk` is never in the `Shift+Tab` cycle. It can only be set with
  `--permission-mode dontAsk`, and `-p` has no mode-switching UI at all.
- Anything that would prompt is denied without waiting. Confirmed locally:
  `rm -rf test/expected` with only `Read` allowed was denied and nothing was removed.
- Allow rules without a trailing ` *` match the command **verbatim**. Confirmed
  locally: with `Bash(python bin/check_lints.py)` allowed, Claude's first attempt
  `python bin/check_lints.py; echo "exit=$?"` was denied. The script's prompt
  therefore tells Claude to run each command exactly as written.
- `dontAsk` does **not** deny the built-in read-only command set (`ls`, `echo`,
  `cat`, `git status`, and similar). Those run in every mode. Do not rely on
  `dontAsk` to block read-only commands.
- Cloud sessions ignore `dontAsk` from settings files. Pass it on the command line.
- `--permission-prompts` requires Claude Code v2.1.259 or later.

## Pattern 3: fully unattended inside an isolated container or VM

```bash
# Isolated container or VM only. Never on a workstation or a shared network.
claude -p "<prompt>" --dangerously-skip-permissions
```

This pattern is documented here but intentionally **not** shipped as a script,
because nothing in this repo guarantees it runs isolated.

Verified facts:

- It skips every permission check, and `permissions.allow` rules have no effect.
  `permissions.deny` rules still block in every mode, including this one.
- Safeguards that remain: `rm` and `rmdir` on a critical path (for example
  `rm -rf /` or `rm -rf ~`) are never auto-approved by any allow rule or hook. In a
  `-p` run, the few calls that would still prompt are denied instead.
- The docs require a container, VM, or the sandbox runtime, and on Linux and macOS
  a non-root user.
- Cloud sessions ignore this mode from settings files, and it is not offered in
  the cloud mode selector at all.

## Scope summary

| Setting | `~/.claude/settings.json` | project `.claude/settings.json` | managed settings | CLI flag |
| --- | --- | --- | --- | --- |
| `defaultMode: auto` | yes | ignored | yes | `--permission-mode auto` |
| `defaultMode: bypassPermissions` | yes | ignored | yes | `--dangerously-skip-permissions` |
| `defaultMode: default/acceptEdits/plan/dontAsk` | yes | yes | yes | `--permission-mode <mode>` |
| `permissions.allow` rules | yes | yes | yes | `--allowedTools` |
| lock the mode in an interactive session | no | no | partial (`disableAutoMode`, `disableBypassPermissionsMode`) | no |
