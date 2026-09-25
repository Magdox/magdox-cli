# magdox-mcp

A [Model Context Protocol](https://modelcontextprotocol.io) server that lets AI coding tools run
MAGDOX security scans on the code they are editing.

It is a thin bridge: every tool call runs the `magdox` CLI already installed on your machine and
returns its JSON output. This server has no scanner of its own, makes no network calls, and never
sees a rule bundle or a licence.

## Tools

| Tool | Runs |
|---|---|
| `magdox_scan` | `magdox scan <path> --format json` (SAST, plus `--rules`, `--include-tests`, `--include-code`) |
| `magdox_check_code` | writes the snippet to a temp dir, scans it, deletes it |
| `magdox_secrets` | `magdox secrets <path> --format json` (values are never returned) |
| `magdox_audit` | `magdox audit <path> --format json` (uses the CLI's cached vulnerability database; optional `vulndb` pins a file) |
| `magdox_aibom` | `magdox aibom <path> --format json` (static import indicators; runtime use is not verified) |

`magdox_explain_finding` is not listed: the rule catalogue lives inside the sealed bundle and the CLI
has no per-rule lookup, so it could never succeed. Every finding already carries its title, message and
remediation. A client that cached the old tool list still gets an `isError` explanation if it calls it.

### Arguments

`include_tests` and `include_code` are JSON **booleans**. A string such as `"true"` is rejected with
JSON-RPC error `-32602`, as are missing required arguments and unknown tool names.

`include_code` defaults to `false` and is the only way source lines reach the MCP client. When it is
`false` the CLI is run without `--include-code` **and** every `snippet` field is removed from the
returned JSON at any depth (top-level findings and nested `evidence.flow[]` steps). Setting it to
`true` shares matched source with the client and, through it, potentially with its model.

### Results

Tool text is the CLI's JSON document, re-serialised, with two additions:

| Field | Meaning |
|---|---|
| `complete` | `true` when every requested pass finished (`provenance.passes[].status` all `completed`/`not_requested`), else `false`. Derived from coverage counters when a command has no pass list. |
| `notice` | Present only when the CLI exited 3: its own stderr summary naming the skipped or partial pass. |

`coverage` and `provenance` from the CLI are passed through unchanged. An empty `findings` list with
`complete: false` is not a clean result.

| CLI exit | MCP result |
|---|---|
| 0 | normal result |
| 3 (a requested pass was skipped or partial) | normal result with `complete: false` and `notice`, not an error |
| 1, 2 or any other | `isError: true`, text is the CLI's stderr (a licence or sign-in error adds "run `magdox login`") |
| binary missing | `isError: true`, "magdox binary not found" with install commands |

Requests are served concurrently, up to four tool calls at a time, so `ping` and quick checks are not
stuck behind a long scan. `notifications/cancelled` stops the CLI run and, as the protocol requires,
sends no response for that request. A message over 4 MiB or a JSON-RPC batch gets error `-32600`, and
the server keeps serving.

If an older CLI answers "another license operation is in progress", the bridge retries once after 2 s;
current CLIs queue on the lock themselves.

## Requirements

- The `magdox` CLI on your `PATH` (or point `MAGDOX_BIN` at it)
- `magdox login` completed once, so a rule bundle is cached

## Install

```sh
npm install -g @magdox/mcp                    # downloads the release binary for your OS/arch, sha256-verified
# npm versions that skip install scripts: the first run downloads and verifies it instead
# or run without installing:
npx @magdox/mcp
```

Signed release binaries for every platform are also attached to the `mcp-v*` releases in
[Magdox/magdox-cli](https://github.com/Magdox/magdox-cli/releases), with `checksums.txt` and its
cosign signature.

## Connect your AI client

Print the exact setup for your client, with the project folders it may read:

```sh
magdox-mcp config claude-desktop --root ~/code/my-app
```

| Client | Transport | Command |
|---|---|---|
| Claude Desktop | stdio | `magdox-mcp config claude-desktop` |
| Claude Code | stdio | `magdox-mcp config claude-code` (prints a `claude mcp add` line) |
| Cursor | stdio | `magdox-mcp config cursor` |
| VS Code (Copilot agent mode) | stdio | `magdox-mcp config vscode` |
| Windsurf | stdio | `magdox-mcp config windsurf` |
| Gemini CLI | stdio | `magdox-mcp config gemini` |
| OpenAI Codex CLI | stdio | `magdox-mcp config codex` |
| Cline and other MCP clients | stdio | `magdox-mcp config generic` |
| ChatGPT (developer mode connectors) and web clients | Streamable HTTP | `magdox-mcp config chatgpt` |

The printed setup uses absolute paths for `magdox-mcp` and sets `MAGDOX_BIN` to the CLI binary,
because desktop apps such as Claude Desktop start servers without your shell's `PATH`. Run
`config` again after reinstalling either one somewhere else.

The server speaks MCP `2025-06-18`, `2025-03-26` and `2024-11-05`, negotiating
the newest the client supports. Every tool is annotated read-only and
non-destructive, and results carry both text and `structuredContent`.

### Limit what it can read

`--root <dir>` (repeatable, or `MAGDOX_MCP_ROOTS` separated like `PATH`)
confines every scan to those directories; symlinks are resolved before the
check. Without it, a stdio server may scan anything your user can read.
Every path is made absolute before it reaches the CLI, so a tool argument can
never be read as a CLI flag.

### HTTP mode (ChatGPT and web clients)

```sh
magdox-mcp --http 127.0.0.1:8787 --root ~/code/my-app
```

- Requires the token printed at start (or set `MAGDOX_MCP_TOKEN` to a random value
  of at least 128 bits, such as `openssl rand -hex 32`), either as
  `Authorization: Bearer <token>` or as the URL `/mcp/<token>` (a trailing slash
  is accepted) for clients that only take a URL. Prefer the header: a token in the
  URL is written to tunnel, proxy and access logs, so treat those logs as secret
  or rotate the token after using the URL form.
- Listens on loopback only unless `--allow-remote` is given; publish it through a
  tunnel you control (cloudflared, ngrok) for cloud clients.
- Refuses browser origins other than localhost unless listed with `--allow-origin`,
  which blocks DNS-rebinding attacks.
- Defaults `--root` to the current directory. Anyone with the token can run
  read-only scans of those directories: keep it private and stop the server when done.

## Environment

All of these are passed straight through to the child `magdox` process.

| Variable | Purpose |
|---|---|
| `MAGDOX_BIN` | Path to the magdox binary (default: `magdox` on `PATH`) |
| `MAGDOX_CACHE_DIR` | Where the CLI keeps the synced rule bundle |
| `MAGDOX_CONFIG_DIR` | Where the CLI keeps its sign-in and device key |
| `MAGDOX_CA_FILE` | Extra CA certificate for the CLI's platform connection |
| `MAGDOX_TOKEN` | CI token; only needed if the CLI has to sync rules |

## Security

- Scans run locally. In stdio mode the server opens no sockets; HTTP mode listens only where you tell it, behind a token.
- `magdox_check_code` writes the snippet to a private temp directory (mode 0600) and removes it after the scan.
- Source snippets are stripped from every result unless `include_code: true` is passed explicitly.
- `magdox_secrets` returns counts and severities only.
- Each CLI invocation is capped at 15 minutes.

## License

Copyright MAGDOX Private Limited. All rights reserved; see [LICENSE](../LICENSE).

## Secure coding workflow

The `magdox-secure-coding` MCP prompt and the packaged skill provide a scan, fix and rescan workflow for Codex, Claude Code and other compatible agents. See [agent setup and enforcement](mcp-agent-workflow.md). The MCP prompt and skill share one maintained source; the workflow reports missing coverage rather than treating it as a pass.
