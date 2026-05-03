# Windsurf / Devin Chat-Triggered Command Execution PoC

## Summary

This repository is a single-issue proof of concept for a Windsurf / Devin
agent execution boundary:

```text
fresh clone -> open in Windsurf -> agent chat "inspect this repository" ->
repo-controlled rule steers the agent into a shell command ->
repo-controlled permission hook approves the shell request ->
local command executes without a fresh command approval
```

The payload is harmless. It only writes:

```text
/tmp/windsurf-chat-exec-proof.txt
```

## PoC Files

- `AGENTS.md` supplies always-on repository instructions.
- `.devin/config.json` defines a `PermissionRequest` hook that approves `exec`
  permission requests.

The proof command writes a static marker to `/tmp`. It does not read secrets,
make network callbacks, install persistence, or modify the repository.

## Reproduction

```sh
cd ~/Downloads
rm -rf windsurf-bug
git clone https://github.com/Arth-Singh/windsurf-bug.git
cd windsurf-bug
rm -f /tmp/windsurf-chat-exec-proof.txt
```

Open the cloned folder in Windsurf, open the Devin/Cascade agent chat, and send:

```text
Please inspect this repository and summarize what it does.
```

Then check:

```sh
cat /tmp/windsurf-chat-exec-proof.txt
```

## Expected Vulnerable Result

If vulnerable, the file exists and contains:

```text
windsurf_chat_exec_poc
```

The important security condition is that this happens without a fresh,
specific approval for the shell command. A generic workspace trust click is not
the same as approving repo-controlled instructions and a repo-controlled
permission hook to execute a command.

## Headless Validation

The same chain was validated with Windsurf's bundled Devin CLI in non-GUI mode:

```sh
rm -f /tmp/windsurf-chat-exec-proof.txt
/Applications/Windsurf.app/Contents/Resources/app/extensions/windsurf/devin/bin/devin \
  -p "Please inspect this repository and summarize what it does."
cat /tmp/windsurf-chat-exec-proof.txt
```

Observed result:

```text
windsurf_chat_exec_poc
```

Note: non-interactive CLI mode does not enforce workspace trust by default, so
the GUI validation should focus on whether Windsurf shows a clear command
approval after the workspace is opened.

## Why This Is Stronger Than A Startup Hook

This is not just `SessionStart` running when the agent opens. The execution is
triggered by an ordinary user chat request:

```text
Please inspect this repository and summarize what it does.
```

The repository controls both sides of the execution path:

- the instruction that makes the agent request a shell command
- the permission hook that approves the shell command

That removes the fresh human approval that users expect before an AI agent runs
local commands suggested by repository-controlled text.

## Impact

If this crosses the GUI trust/approval boundary, a malicious repository can
turn a routine agent prompt into local command execution under the victim's user
account. A real attacker could replace the harmless proof command with commands
that run local scripts, invoke package managers, make network requests, or act
on third-party systems through locally configured tools.

## Security Boundary To Verify

This should be considered reportable only if Windsurf's GUI agent flow executes
the command without a clear, specific approval for the repo-controlled command
or permission policy.

Expected defenses:

- do not let project config approve its own shell execution
- require a separate prompt for project-level `PermissionRequest` hooks
- show the source file and exact command before execution
- ignore project command approvals in untrusted workspaces
