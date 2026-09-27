# MAGDOX in AI coding tools

When explicitly configured in a supported host, the MAGDOX plugin runs the installed `magdox` CLI on regular files named by write/edit events and returns findings to the agent. A supported Stop hook rescans files recorded for that workspace and session, requesting a block for critical/high findings or incomplete checks. Hosts can omit events or ignore hook responses; this is not automatic enforcement in every editor. The [MCP bridge](mcp.md) provides separate, on-demand tools.

The v1.2 launcher installs the signed private plugin bundle after product authorization. There is no new public plugin package; `@magdox/cli` and `@magdox/mcp` retain their existing names.

The plugin ships no scanner or rules and makes no network requests of its own. Its CLI scans do not request uploads or remote AI review. The installed CLI can still contact MAGDOX for authorization or rule updates unless offline mode is enabled.

## Requirements

- MAGDOX launcher 1.2.0 or later, the authorized private engine, rules, and compatible signed private components. Sign in separately in a terminal with `magdox login`; hooks do not install software or sign in for you.
- Node.js 18 or later on a POSIX system with ownership/mode checks.
- Install the private MCP bridge and plugin with `magdox mcp install` using authenticated product access. The public npm `@magdox/mcp` package is only an optional launcher shim; it includes no private bridge, hooks, or skills and downloads no binary.

Native Windows is unsupported for state-backed hooks and the staged gate: they fail closed without POSIX ownership checks. Existing Windows host templates do not establish Windows support.

## What the agent sees

| Moment | What happens |
|---|---|
| Session start | Instructions and a notice if the CLI version probe fails; this does **not** verify sign-in or product access |
| After a write or edit | Up to five findings with severity, title, `path:line` and the recommended fix, plus a count of the rest; a note if the scan was incomplete |
| Before the agent stops | A rescan of recorded files; critical/high findings or incomplete checks request a block where the host supports it |

Finding summaries select public guidance and location fields rather than rule-ID or source-snippet fields. Scanner stderr and process failures are converted to fixed diagnostic messages, never copied into hook context. Findings still contain paths and authored guidance; do not treat them as anonymized or as instructions to execute.

Successful unchanged-file checks are debounced for two seconds. Each file is a separate CLI target within a shared 20-second scan budget. Missing/malformed coverage, zero scanned files, failures, and incomplete analysis remain visibly unchecked, even with no findings. Failed scans can be retried. Stop uses high-severity remediation even when ordinary display is filtered to critical. An active Stop retry does not request another block, avoiding a loop.

## Hosts

Review the files in the installed, verified private plugin directory and configure your host explicitly. Only the Claude Code hook integration has been tested end to end. Other host setups remain experimental; this validation does not establish that every feature or host version works.

| Host | Setup status |
|---|---|
| Claude Code | Local-session setup: `claude --plugin-dir "<plugin>"`; only hook integration tested end to end |
| Codex | Experimental manual `AGENTS.md` / hook template; no `.codex-plugin/plugin.json` or native Codex plugin registration |
| Gemini CLI, Copilot CLI, Cursor, Windsurf, OpenCode | Experimental templates/instructions requiring review against the installed host and explicit trusted paths |
| No supported editor hooks | Optional staged-content gate in an existing Git hook chain; installation is not automatic |

The plugin's `.mcp.json` is intentionally empty. Configure MCP separately with explicit absolute roots:

```sh
magdox mcp config generic --root /absolute/project
```

Select the appropriate client instead of `generic`, then review and merge the generated configuration. It uses the absolute installed public launcher and separate `mcp serve --root` arguments, not a shell. Repeat `--root` for additional trusted directories (maximum 32); relative tool paths use the first root. Do not assume automatic project-root variable expansion or omit the root.

For Cursor's POSIX-shell template, set `MAGDOX_PLUGIN_DIR` to the trusted absolute installed plugin directory and ensure the editor inherits it. Copilot's Bash template similarly requires `PLUGIN_ROOT`. Keep full script paths quoted; do not paste untrusted repository text into shell commands. Consult the installed plugin README for template-specific path handling and limits.

## Configuration

`.magdox-plugin.json` at the repository root:

```json
{ "minSeverity": "high", "ignorePaths": ["vendor/", "generated/", "*.min.js"] }
```

Only the event workspace root is consulted; parent settings are not inherited. Invalid, oversized, or symlinked configuration leaves the check unchecked. Ignore patterns apply during write/edit checks; already recorded paths are still checked at Stop.

`MAGDOX_PLUGIN_DISABLE=1` explicitly bypasses checks. `MAGDOX_BIN` selects a trusted installed binary. `MAGDOX_PLUGIN_ARGS` accepts whitespace-separated `--rules`, `--vulndb`, `--endpoint`, `--offline`, and `--include-tests`; upload, code-inclusion, AI, and other flags are dropped. Shell quoting is not parsed, so values containing whitespace are unsupported there. Filenames remain separate absolute arguments after `--`.

For offline editor checks, set `MAGDOX_PLUGIN_ARGS=--offline`; cached access/rules or explicit local rules must already be available. Findings still reach the editor and potentially its model. These controls do not make all output nonsensitive.

## Optional staged-content gate

After review, add `node "<plugin>/pre-commit/magdox-staged.js"` to your existing Git hook chain. No hooks are installed or replaced automatically. The gate scans changed staged regular-file blobs in a fresh private temporary snapshot, not unstaged working-tree bytes. It forces offline mode and includes tests, then removes the snapshot on normal success or failure.

Missing access/CLI, malformed reports, incomplete or insufficient coverage, configured-threshold findings, and changes to selected staged blobs during the scan block the commit. Symlinks, submodules, unresolved merges, and ambiguous paths are rejected. Unsupported file types can therefore block even a documentation-only commit. This bounded check is not a whole-project or dependency scan.

## Limits

A hook is agent guidance, not a remote merge gate: the agent can still finish with findings open, and a host that skips hooks gets no check. Filesystem confinement is not an OS sandbox against concurrent changes by another trusted-user process. The staged gate can be explicitly bypassed and does not enforce remote policy.

Keep the CLI in CI as a separate enforced gate (`magdox scan --full --format json --fail-on high -- .`), with exit 3 (incomplete coverage) and exit 4 (threshold findings) both failing the job. Provision an authorized private engine and rules before scanning; keep credentials and private artifacts out of shared caches and uploaded artifacts. Run a full-project scan before release and report incomplete checks honestly.
