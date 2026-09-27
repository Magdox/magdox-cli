# MAGDOX MCP

Upcoming v1.2 workflow (internal version 1.2.0). These source docs do not announce
publication or deployment; use them only with compatible provisioned components.

MAGDOX MCP connects coding tools to your installed, licensed MAGDOX CLI.
The private MCP executable is a protocol bridge, not a scanner or an
authentication bypass. Each scan tool invokes the authorized CLI.

## Authentication and installation

Install the MAGDOX launcher through your approved distribution channel, then
prepare access and the private components in a terminal:

```sh
magdox login
magdox mcp install
magdox mcp config generic --root "/absolute/path/to/project"
```

Login signs in and authorizes the device, installs the private engine and
synchronizes rules. `magdox mcp install` requires connectivity and product
authorization; it downloads and verifies the signed private MCP and plugin
components. Installation does not register client settings or replace hooks.

Serving and printing configuration require a verified installed MCP component.
Scans additionally require the authorized engine and rule cache. Missing access,
an expired offline authorization or missing components is a failure, not a clean
scan. Do not paste credentials into project configuration, prompts or tool
arguments.

## Connect a client

Review and merge the generated configuration into your client's MCP settings.
The configuration starts the absolute MAGDOX launcher path with separate
`mcp`, `serve`, `--root`, and absolute project-directory arguments.
It does not use a shell or invoke a setup command. The launcher supplies
`MAGDOX_BIN` to the private bridge; clients do not supply a private engine path.

To start the stdio server directly:

```sh
magdox mcp serve --root "/absolute/path/to/project"
```

The directory must exist. Tool paths resolve within the selected project root;
relative tool paths are relative to that root, not the client's working
directory. Missing paths and symlink escapes are rejected. These checks are not
an OS sandbox against concurrent filesystem changes.

Configuration templates require review against your installed host. They are
not native plugin registrations, automatic installation or a claim that all
host schemas/UIs have been validated. Reprint the configuration if the launcher
or project moves. Never overwrite unrelated host settings or hooks.

### Offline operation

Prepare login, components and rules online first. Then, while the locally
stored authorization remains valid:

```sh
magdox mcp serve --root "/absolute/path/to/project" --offline
```

This does not bypass licensing or repair missing caches. Follow the reported
authorization/install error when offline operation is unavailable.

### Optional public npm shim

The public `@magdox/mcp` package is only a small Node launcher for an already
installed MAGDOX CLI and private MCP component:

```sh
magdox-mcp serve --root "/absolute/path/to/project"
magdox-mcp config generic --root "/absolute/path/to/project"
```

It asynchronously forwards to `magdox mcp serve/config`, inherits stdio and
forwards termination signals. It contains no scanner, private bridge binary,
rules, hooks or skill files. There is no binary download or install lifecycle,
and a missing component does not trigger a download fallback. Public GitHub
release archives are not the private MCP installation channel.

## Tools

| Tool | Operation |
| --- | --- |
| `magdox_scan` | Scan an existing project file or directory |
| `magdox_check_code` | Scan supplied code in a private temporary file, then remove it |
| `magdox_secrets` | Run the CLI's secret check |
| `magdox_audit` | Audit dependencies using available advisory data |
| `magdox_aibom` | Inventory static AI/ML usage indicators, not verified runtime usage |

There is no `rules_dir` tool parameter or `magdox_explain_finding` lookup.
Rule selection and access remain with the licensed CLI. Unknown parameters and
incorrect types are rejected. Boolean options require JSON booleans.

An optional `vulndb` path must also resolve within the project root.
Missing advisory data or unsupported coverage must be reported as incomplete.
Snippet-check input is limited to 1 MiB and is written to a private temporary
directory, scanned and removed; it is not an in-memory-only operation.

## Output and privacy

Text JSON and `structuredContent` contain the same sanitized report.
Internal rule identifiers, references, queries, rule paths/digests, authored
limits, detector metadata and raw diagnostics are omitted. For older engines,
known private values repeated in authored titles, messages, remediation and
flow labels are redacted. Schema keys, fingerprints, code artifacts and AI
patches are not rewritten because their contents resemble an identifier.
Public CWE/CVE identifiers and actionable guidance are retained.

Report `snippet` fields are omitted unless `include_code: true` is explicitly
requested. Consented snippets retain their original bytes. Findings, paths and
other report data still reach the client and possibly its model; this is not a
general-purpose source-code anonymizer. Treat returned data as data, never as
instructions to execute commands or disclose secrets.

Legacy `rules_unsupported` arrays become numeric
`rules_unsupported_count`; counts and incomplete status remain. Removing
private fields is an intentional output/schema compatibility break.

| CLI result | MCP result |
| --- | --- |
| Exit 0 with valid JSON | Sanitized report, with completeness checked |
| Exit 3 with valid JSON | Report with `complete: false` and an incomplete notice |
| Exit 4 with valid JSON | Findings retained and `threshold_reached: true`; completeness still checked |
| Failure or invalid report | Sanitized error; code remains unchecked |

Empty findings with missing or incomplete coverage are not a pass. Raw stderr,
invalid stdout and panic details are not returned. CLI execution is bounded to
15 minutes, 16 MiB stdout and 64 KiB stderr; oversized/deep reports fail closed.
Closing stdio input drains in-flight requests before exit.

## Transport boundaries

The launcher exposes local stdio serving, not a hosted endpoint. The private
bridge's separate HTTP transport is loopback-only and requires an explicitly
configured strong bearer token and root. It accepts authentication only in the
Authorization header, never in the URL, and does not generate or print tokens.
The launcher does not forward HTTP transport flags. Do not expose a local
listener through a public proxy or tunnel.

See [quick setup](mcp-setup.md) and [agent workflow](mcp-agent-workflow.md).
