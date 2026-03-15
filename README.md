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

7. Every subsequent `run_in_terminal` attempt in the same session also fails.
   The user's own terminal and file-reading Copilot tools continue working normally.

---

## Expected behaviour

Both calls succeed; all subsequent terminal tool calls also succeed.

---

## Actual behaviour

Call 1 succeeds. Calls 2+ fail with `ENOPRO` on the workspace root path. The error
persists for the lifetime of the chat session; opening a new session resets it so
that exactly one call succeeds again before failing.

---

## Root cause

`ENOPRO` means the VS Code renderer process cannot find a `FileSystemProvider`
registered for the `file:` scheme at the workspace root.

In VS Code Web / Codespaces, this provider is owned by the Remote extension running
in the renderer. **Shell integration** (`terminal.integrated.shellIntegration.enabled`,
which defaults to `true`) injects activation scripts whose URIs are anchored to
`file://` VFS paths. When the first Copilot Agent terminal session is closed or
recycled by the tool after call 1, the shell-integration teardown path releases its
reference to the VFS provider. If the provider reaches a zero reference count, VS
Code Web de-registers it. The next `run_in_terminal` call tries to resolve
`file:///workspaces/<repo>` to open a new terminal and gets `ENOPRO`.

Key evidence:
- Setting `terminal.integrated.shellIntegration.enabled: false` **completely resolves
  the bug** — no other change needed.
- The bug reproduces with only `github.copilot-chat` installed (no Python extension,
  no project-specific configuration required).
- The bug does not reproduce in VS Code Desktop, where `file:` is handled by the OS
  and does not go through a renderer-side VFS provider.

---

## The fix

Add this to your devcontainer's `"settings"` block:

```jsonc
// .devcontainer/devcontainer.json  →  customizations.vscode.settings
"terminal.integrated.shellIntegration.enabled": false
```

This disables shell integration script injection, preventing the VFS provider
reference from being dropped between Copilot Agent terminal calls.

**Trade-off:** Shell integration provides helpful terminal features (command
decorations, working-directory detection, command history). Disabling it loses
those features inside the Codespace. There is no known way to keep shell
integration enabled and also have `run_in_terminal` work reliably in VS Code Web.

---

## Verifying the fix

1. Add `"terminal.integrated.shellIntegration.enabled": false` to `.devcontainer/devcontainer.json`.
2. Rebuild the container (`F1 → Dev Containers: Rebuild Container`).
3. Open Copilot Agent and send 5+ consecutive messages that each invoke
   `run_in_terminal`.
4. All calls should succeed.

---

## Additional notes

- **Reloading the VS Code window** does not fix an active session — the next
  `run_in_terminal` call re-triggers shell integration teardown.
- **VS Code Desktop** is not affected; the `file:` scheme is handled by the native
  OS filesystem and is not subject to renderer-side provider lifecycle management.
- The `webOnly: true` setting in `devcontainer.json` ensures the Codespace always
  opens in VS Code Web so the bug is consistently exercised.
