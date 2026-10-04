# MAGDOX MCP quick setup

Set up MAGDOX v1.3 with launcher 1.3.0 or later, product authorization and
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

The optional public npm package is a forwarding shim only: it downloads no
binary and includes no private engine, rules, hooks or skill files. It cannot
replace `magdox mcp install`. The generated absolute-launcher configuration
does not require Node or a shell on a desktop client's PATH.

Templates require host-specific review; no native plugin registration or live
host/schema verification is claimed. There is no hosted endpoint or public
tunnel setup. Snippet checks use a private temporary file, not an in-memory
scanner. Only the Claude Code hook integration has been tested end to end;
other host setups remain experimental. Automatic hooks are POSIX-only
(macOS and Linux); Windows hooks are unsupported.

See [full MCP reference](mcp.md) and [agent workflow](mcp-agent-workflow.md).
