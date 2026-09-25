# Magdox in coding agents

This integration helps an agent scan generated code, fix detected problems, and rescan before handoff. It reuses the local Magdox CLI through MCP. It does not guarantee vulnerability-free output or intercept every token an agent generates.

## Install and connect

Install the Magdox CLI, complete `magdox login`, and install this MCP bridge with `npm install -g @magdox/mcp` (0.1.2 or later), which puts `magdox-mcp` on your `PATH`. The skill ships in the npm package at `skills/magdox-secure-coding/SKILL.md` (run `npm root -g` to find it).

| Client | Connect the MCP server | Load the workflow |
| --- | --- | --- |
| Codex | `codex mcp add magdox -- magdox-mcp` | Copy the skill directory into the target repository's `.agents/skills/`, then invoke `$magdox-secure-coding` with the coding task. |
| Claude Code | `claude mcp add --transport stdio --scope project magdox -- magdox-mcp` | Copy the same directory into `.claude/skills/`, then invoke `/magdox-secure-coding` with the task. |
| Other clients with MCP prompts | Register the same binary as a local stdio server using that client's configuration. | Discover `prompts/list`, then get `magdox-secure-coding` through `prompts/get`. The prompt text is the same maintained skill. |
| Clients with tools but no skills/prompts | Register the local stdio server. | Add the skill body to that client's project instructions; ask the agent to follow it for code changes. |

Do not overwrite an existing skill or MCP configuration. Merge the new entry. Every developer/CI environment needs its own permitted CLI access and rule bundle. The bridge does not bypass Magdox licences. Remote/cloud agents need the CLI in their own execution environment: a local stdio path is not a hosted MCP endpoint.

Sources: [Codex MCP](https://developers.openai.com/codex/mcp/), [Codex skills](https://developers.openai.com/codex/skills/), [Claude Code MCP](https://code.claude.com/docs/en/mcp), [Claude Code skills](https://code.claude.com/docs/en/skills). Configuration guidance checked 2026-09-23. Other clients require their own MCP/skill support; they have not all been exercised interactively.

## Workflow and boundaries

1. Baseline scan identifies existing findings.
2. Snippet checks provide feedback during generation.
3. Final repository scan checks the integrated code; secrets and changed dependencies receive their separate checks.
4. The agent fixes issues in scope, runs project tests, and rescans after the final edit.
5. The handoff reports unresolved issues and incomplete/missing coverage. Missing tools, rules, advisories, or supported language coverage are not a pass.

The bridge exposes `magdox_scan`, `magdox_check_code`, `magdox_secrets`, `magdox_audit`, and `magdox_aibom`. `magdox_explain_finding` is not offered by this CLI bridge; findings carry remediation guidance instead. Dependency auditing needs a local vulnerability database. No automatic upload or organisation mutation is added. Findings, paths and messages reach the MCP client/model; matched source snippets are stripped unless explicitly opted in. Snippet-check inputs are already visible to the coding agent and are written to a temporary local file for scanning, then removed.

## Enforceable gate

Skills and prompts guide agent behavior; they do not enforce merge policy. For enforcement, configure the existing CLI in required CI checks, for example `magdox scan . --full --format json --fail-on high`, with the required rule/advisory material available. Keep partial-scan failures blocking as well as severity failures; do not use `|| true`. The CLI's exit 3 means incomplete coverage; exit 4 means findings reached the selected threshold. Test the policy on an intentionally vulnerable fixture and on an unsupported/unscanned fixture before requiring it on protected branches. Existing enterprise policy takes precedence over this example.

This change does not publish the package, install it into user clients, or change branch protection. A separate plugin marketplace package, remote authenticated MCP service, and client-specific hard hooks would require additional implementation.
