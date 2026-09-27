# MAGDOX MCP quick setup

Upcoming v1.2 workflow (internal version 1.2.0). Publication and private component
availability are separate release gates; these instructions are not a release announcement.

Use your approved installed MAGDOX launcher. Complete authentication and private
component installation in a terminal:

```sh
magdox login
magdox mcp install
magdox mcp config generic --root "/absolute/path/to/project"
```

Login prepares device authorization, the private engine and rules. MCP installation
requires online authorization and verifies the signed private MCP and plugin
components. It does not automatically install host settings or hooks.

Review and merge the generated configuration into your client's local MCP
settings. It uses the absolute launcher path, separate `mcp serve --root ABS`
arguments and no shell. Select an existing absolute project directory. Do not
put credentials in the configuration or overwrite unrelated settings.

For direct stdio operation:

```sh
magdox mcp serve --root "/absolute/path/to/project"
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
scanner.

See [full MCP reference](mcp.md) and [agent workflow](mcp-agent-workflow.md).
