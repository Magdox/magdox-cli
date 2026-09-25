# MAGDOX MCP Server

The `magdox-mcp` server exposes MAGDOX scanning as MCP tools for AI coding editors.
Full reference: [mcp.md](mcp.md). Signed binaries are on the `mcp-v*` releases of this repository. Install it separately:

```sh
npm install -g @magdox/mcp   # or: npx @magdox/mcp
```

It runs the `magdox` CLI on this machine; it ships no rules and no licence.

## Tools

- `magdox_scan`: scan a directory for security findings
- `magdox_check_code`: check a code snippet inline without writing to disk
- `magdox_aibom`: inventory AI SDKs in a project
- `magdox_audit`: audit dependencies for vulnerabilities
- `magdox_secrets`: scan for committed secrets

All tools are read-only. `magdox_explain_finding` is not offered: the rule
catalogue is sealed inside the bundle, so findings carry their own title,
message and remediation instead.

## Setup

Let the server print the setup for your client, with the absolute path to
`magdox-mcp` and `MAGDOX_BIN` pointing at the `magdox` binary. Desktop apps
start servers without your shell's PATH, so a bare `magdox-mcp` often fails.

```
magdox-mcp config <client> --root /path/to/project
```

Clients: `claude-desktop`, `claude-code`, `cursor`, `vscode`, `windsurf`,
`gemini`, `codex`, `chatgpt`, `cline`, `generic`. `--root` limits the folders
the server may read and can be repeated.

ChatGPT and other web clients use `magdox-mcp --http 127.0.0.1:8787 --root <project>`
behind a tunnel you control; the token it prints is the credential.

## Requirements

- `@magdox/mcp` installed (`npm install -g @magdox/mcp`) and the `magdox` CLI installed with npm or Homebrew
- `magdox login` completed in a terminal (the server runs your signed-in CLI; it ships no rules and no licence)

Full details in the MCP repository README.
