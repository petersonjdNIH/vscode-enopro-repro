# Bug Reproducer: `ENOPRO: No file system provider found` — Copilot Agent `run_in_terminal`

> **Affects:** GitHub Copilot Chat (Agent mode) · GitHub Codespaces · VS Code Web  
> **Related closed issue:** [microsoft/vscode#283806](https://github.com/microsoft/vscode/issues/283806) (closed as *not-reproducible*, but issue persists)

---

## Summary

When GitHub Copilot Agent mode attempts to run any terminal command via its internal
`run_in_terminal` tool inside a GitHub Codespace, the call consistently fails with:

```
ENOPRO: No file system provider found for resource 'file:///workspaces/<repo-name>'
```

The VS Code terminal works fine when the user types commands directly. Only the
Copilot Agent's programmatic terminal tool is affected. The error survives window
reloads and full container rebuilds.

---

## Environment

| Field | Value |
|---|---|
| Platform | GitHub Codespaces (VS Code Web, browser) |
| Devcontainer base image | `mcr.microsoft.com/devcontainers/base:ubuntu-24.04` |
| VS Code version | (see bottom of Codespace — any recent version) |
| Copilot Chat version | (any version with Agent mode / tool calls) |
| Workspace path | `/workspaces/vscode-enopro-repro` |

---

## Steps to reproduce

1. Click **"Open in Codespace"** on this repository (or open it via codespaces.new).

2. Wait for the container to fully start (the terminal prompt appears).

3. Open **GitHub Copilot Chat** and switch to **Agent mode**.

4. Send any message that causes Copilot to invoke the terminal tool, e.g.:

   > Run `echo hello` in the terminal

5. Observe the tool call attempt and the resulting error in the chat panel.

---

## Expected behaviour

Copilot Agent executes `echo hello` in the integrated terminal and returns the output
`hello`.

---

## Actual behaviour

The tool call fails immediately with:

```
ERROR while calling tool: ENOPRO: No file system provider found for resource 'file:///workspaces/vscode-enopro-repro'
Please check your input and try again.
```

The chat session then becomes partially non-functional: subsequent tool calls that
interact with the file system (file reads, file writes, linting tasks) also begin
failing with the same error, eventually making the entire Copilot Chat session
unresponsive until the browser tab is refreshed — at which point the cycle repeats.

---

## Additional observations

- The error occurs on the **very first** terminal tool call after opening a fresh Codespace.
- **Reloading the VS Code window** (`F1 → Developer: Reload Window`) does not fix it —
  the error reappears immediately on the next tool call.
- **Rebuilding the container** (`F1 → Dev Containers: Rebuild Container`) does not fix
  it either. The issue returns as soon as Copilot Agent tries to use the terminal.
- Typing commands directly into the VS Code terminal panel works perfectly.
- No custom extensions, tasks, or workspace settings are present in this repo —
  this is the minimal possible devcontainer configuration.
- The error message `ENOPRO` corresponds to `vscode.FileSystemError.NoPermissions` /
  "No file system provider" — indicating the Codespaces VFS (Virtual File System)
  provider is either not registered or has been de-registered for the workspace root
  by the time the Copilot tool makes its call.

---

## Workaround

There is currently no reliable workaround. The user must type all terminal commands
manually; Copilot Agent cannot automate terminal operations in this environment.

---

## Open this repo in a Codespace

[![Open in GitHub Codespaces](https://github.com/codespaces/badge.svg)](https://codespaces.new/petersonjdNIH/vscode-enopro-repro)
