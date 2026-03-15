# Bug Reproducer: `ENOPRO: No file system provider found` — Copilot Agent `run_in_terminal`

> **Affects:** GitHub Copilot Chat (Agent mode) · GitHub Codespaces · VS Code Web  
> **Related issue:** [microsoft/vscode#283806](https://github.com/microsoft/vscode/issues/283806)

[![Open in GitHub Codespaces](https://github.com/codespaces/badge.svg)](https://codespaces.new/petersonjdNIH/vscode-enopro-repro)

---

## Summary

In a GitHub Codespace opened in VS Code Web (browser), the Copilot Agent
`run_in_terminal` tool succeeds on the **first** call but fails on every
**subsequent** call with:

```
ENOPRO: No file system provider found for resource 'file:///workspaces/<repo-name>'
```

The user's own integrated terminal is unaffected. File read/write Copilot tools are
unaffected. Only the programmatic `run_in_terminal` tool call is broken after the
first invocation.

---

## Environment

| Field | Value |
|---|---|
| Platform | GitHub Codespaces (VS Code Web, browser) |
| Devcontainer base image | `mcr.microsoft.com/devcontainers/base:ubuntu-24.04` |
| VS Code version | Any recent version |
| Copilot Chat version | Any version with Agent mode / tool calls |
| Workspace path | `/workspaces/vscode-enopro-repro` |

---

## Steps to reproduce

1. **Open the Codespace** — click the badge above or go to `codespaces.new`.  
   Open it in **VS Code Web** (default; do not switch to VS Code Desktop).

2. **Wait** for the container to fully start and the terminal prompt to appear.

3. **Open GitHub Copilot Chat** and switch to **Agent mode**.

4. **First call — will succeed.** Send:
   > Run `echo first call` in the terminal

   Confirm that Copilot executes the command and returns `first call`.

5. **Second call — will fail with ENOPRO.** In the same chat session, send:
   > Now run `echo second call` in the terminal

6. Observe the error in the chat panel. It will look like:
   ```
   ERROR while calling tool run_in_terminal: ENOPRO: No file system provider
   found for resource 'file:///workspaces/vscode-enopro-repro'
   ```

7. Every subsequent `run_in_terminal` attempt in the same session will also fail.
   The user's own terminal and file-reading Copilot tools continue to work normally.

---

## Expected behaviour

Both `echo first call` and `echo second call` succeed; all subsequent terminal
tool calls also succeed.

---

## Actual behaviour

Call 1 succeeds. Calls 2+ fail with `ENOPRO` on the workspace root path. The error
persists for the lifetime of the chat session; opening a new chat session resets it
so that exactly one call succeeds again before it fails.

---

## Root cause analysis

`ENOPRO` (`FileSystemError.NoPermissions` / "no provider") on a `file://` URI means
VS Code's renderer process cannot find a `FileSystemProvider` registered for the
`file:` scheme at the workspace root.

In VS Code Web / Codespaces the `file:` scheme provider is registered by the
**Remote - Codespaces** extension in the renderer process. It is registered once
per workspace session. It can be *temporarily de-registered* when any extension
calls `vscode.workspace.updateWorkspaceFolders()`, because that API triggers VS Code
to re-initialise the workspace folder list — tearing down existing providers and
re-registering them after the update completes.

**What triggers the update in this repo:**

1. The first `run_in_terminal` call opens a new terminal.
2. Opening a terminal causes the **`ms-python.python`** extension to activate and
   evaluate `python.defaultInterpreterPath`.
3. `python.defaultInterpreterPath` is set to `${workspaceFolder}/.venv/bin/python`,
   which does **not exist** on disk (no `postCreateCommand` creates it).
4. Detecting a missing interpreter, the Python extension starts an
   interpreter-discovery cycle. Internally this calls
   `workspace.updateWorkspaceFolders()` to refresh the list of known environments.
5. That call briefly de-registers the `file:` VFS provider for
   `file:///workspaces/vscode-enopro-repro`.
6. Copilot Agent's second `run_in_terminal` call — which was queued almost
   immediately after the first — arrives during this brief window and fails.

**Why this doesn't reproduce without extensions:** A fresh Codespace with no
extensions has no code that calls `updateWorkspaceFolders()` between Copilot tool
calls, so the provider is never de-registered.

**Why `python.terminal.activateEnvironment: false` doesn't help:** That setting
suppresses *terminal activation* (sourcing the venv on every new shell), but does
not suppress the *interpreter-discovery* code path that triggers the workspace
folder update.

---

## Isolation matrix

Tests performed across multiple full container rebuilds on the affected project,
removing extensions one at a time:

| Removed | Bug still present? |
|---|---|
| `ms-azuretools.vscode-azurefunctions` | Yes |
| `ms-azuretools.vscode-azureresourcegroups` | Yes |
| `ms-vscode.azurecli` | Yes |
| `ms-python.vscode-python-envs` | Yes |
| `ms-python.vscode-pylance` | Yes |
| `bradlc.vscode-tailwindcss` | Yes |
| `hashicorp.terraform` | Yes |
| `github.vscode-pull-request-github` | Yes |
| `github.vscode-github-actions` | Yes |
| `ms-python.autopep8` | Yes |
| `ms-python.debugpy` | Yes |
| All of the above; remaining: `github.copilot-chat` + `ms-python.python` + `ms-python.black-formatter` | **Yes** |

Settings also ruled out:
- `python.terminal.activateEnvironment: false` — no effect
- Removing all `azureFunctions.*` settings — no effect
- Removing `runOn: folderOpen` tasks — no effect

---

## The fix

The root cause is that `python.defaultInterpreterPath` points to a path that does
not exist when extensions first activate. There are three equivalent fixes; apply
the one that fits your project:

### Fix A — Point to the system Python (always exists)

```jsonc
// .devcontainer/devcontainer.json  →  "settings":
"python.defaultInterpreterPath": "/usr/local/bin/python3"
```

Use this when you want the Python extension to work immediately without requiring
the project venv to be set up first.

### Fix B — Create the venv before the container is usable

```jsonc
// .devcontainer/devcontainer.json
"postCreateCommand": "python3 -m venv /workspaces/<repo>/.venv"
```

Ensures the path exists by the time extensions activate. If `postCreateCommand`
already runs a script that creates the venv, verify that the venv creation step
completes *before* the script exits (i.e., it is not backgrounded).

### Fix C — Remove `python.defaultInterpreterPath` entirely

```jsonc
// .devcontainer/devcontainer.json  →  "settings":
// (delete the python.defaultInterpreterPath line)
```

Without this setting, the Python extension uses its built-in discovery logic and
does not trigger `updateWorkspaceFolders()` during activation.

---

## Verifying the fix

1. Apply one of the fixes above to `.devcontainer/devcontainer.json`.
2. Rebuild the container (`F1 → Dev Containers: Rebuild Container`).
3. Open Copilot Agent and send 5+ consecutive messages that each invoke
   `run_in_terminal`.
4. All 5 calls should succeed.

---

## Additional notes

- **Reloading the VS Code window** does not fix an active session — the second call
  after reload will still fail because the Python extension re-activates on terminal
  open.
- **VS Code Desktop** (non-browser) is not affected because the `file:` scheme
  provider is handled by the OS file system directly and cannot be de-registered.
- The `webOnly: true` setting in this repo's `devcontainer.json` ensures the
  Codespace always opens in VS Code Web so the bug is always exercised.
