# MAGDOX MCP quick setup

Set up MAGDOX v1.3.2 with launcher 1.3.0 or later, product authorization and
compatible signed private engine, MCP and plugin components.

Use your approved installed MAGDOX launcher. Complete authentication and private
component installation in a terminal:

```sh
magdox login
magdox mcp install
magdox mcp config generic --root "/absolute/path/to/repository"
```

Login prepares device authorization, the private engine and rules. MCP installation
requires online authorization and verifies the signed private MCP and plugin
components. It does not automatically install host settings or hooks.

Review and merge the generated configuration into your client's local MCP
settings. It uses the absolute launcher path, separate `mcp serve --root ABS`
arguments and no shell. Select an existing absolute repository directory. Do not
put credentials in the configuration or overwrite unrelated settings.

For direct stdio operation:

```sh
magdox mcp serve --root "/absolute/path/to/repository"
```

Use `--offline` only after preparing components and caches online, while offline
authorization remains valid. Missing access, rules, advisory data or coverage
is unchecked, not clean.

## First check

Ask the agent to run `magdox_scan` with path `.`, the repository root. The
server's MCP instructions name the root it reads; relative paths resolve
against it.

- Results are JSON. `complete: false` means part of the code was not checked; it
  is not a clean result. Each pass in `provenance.passes` says why it is partial.
- A result is kept within 64 KiB so a model can read it. A larger report returns
  its most severe findings and a `truncated` object with the total; scan a
  narrower path to see the rest.
- `magdox_check_code` needs a `filename` (preferred) or a `language`, such as
  `python` or `tsx`.
- Errors say what to change. An argument error names the argument and the
  repository root. A missing sign-in, vulnerability database or rule set names
  the command that fixes it: `magdox login`, `magdox vulndb fetch` or
  `magdox rules sync`. Run it in a terminal, then retry.
- A check is stopped after 15 minutes and reported as timed out; scan a narrower
  path instead.

The optional public npm package is a forwarding shim only: it downloads no
binary and includes no private engine, rules, hooks or skill files. It cannot
replace `magdox mcp install`. The generated absolute-launcher configuration
does not require Node or a shell on a desktop client's PATH.

Templates require host-specific review; no native plugin registration or live
host/schema verification is claimed. There is no hosted endpoint or public
tunnel setup. Snippet checks use a private temporary file, not an in-memory
scanner, and report findings under the snippet's file name. Only the Claude
Code hook integration has been tested end to end; other host setups remain
experimental. Automatic hooks are POSIX-only (macOS and Linux); Windows hooks
are unsupported.

See [full MCP reference](mcp.md) and [agent workflow](mcp-agent-workflow.md).
