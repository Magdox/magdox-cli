# Scan evidence and release packages

MAGDOX v1.3.2 sends version 2 findings uploads to a compatible API and creates
local report packages. Scanning requires an authorized private engine and rules.

## Review locally, then preview an optional upload

```sh
magdox scan . --format json --out scan.json
magdox scan . --show-payload --repository my-project > upload-preview.json
magdox scan . --upload --repository my-project
```

Scanning still requires the existing product entitlement and rules. A preview
does not upload findings, including when `--upload` is also supplied.
Authentication and rule renewal have their own network behavior. AIBOM uses a
separate upload path; this preview does not cover it.

Local JSON retains source context but omits private rule metadata. The
`--show-payload` preview also omits private rule metadata; it is not the exact
authenticated upload body. Uploads retain finding location, confidence,
limits, captured rule guidance and ordered same-file flow evidence. Source
snippets, including nested flow snippets, require `--include-code` and a repository
that permits snippets. The upload builder removes customer function names from
generated interprocedural flow labels. Detailed parser/OS failure text stays
local; uploaded failure entries contain relative paths and a generic reason.

Dependencies retain positive EPSS scores, KEV flags, aliases, all reported fixed
versions and advisory guidance. Legacy advisory bundles cannot distinguish a
missing EPSS value from zero; zero is therefore represented as unreported. A
missing KEV flag does not establish safety. These are local bundle snapshots,
not live intelligence queries.

## Read completeness before finding counts

Customer JSON includes coverage and available provenance while omitting private
rule identifiers and rule digests. Authenticated uploads retain internal metadata
under `coverage.provenance`, including execution start/end, decoded rule-content
digest, decoded advisory-content digest/build time when used, test scope and
the status of each pass. Server receipt time is separate. Presentation redaction
does not change finding fingerprints or the authenticated upload contract.

The content digests identify the inputs the engine decoded; they are not archive
signatures or proof of publisher identity. SAST and IaC currently share the
engine's aggregate coverage. `completed` means a pass finished within its
supported scope, not that every file format or framework was analyzed.

`--full` requests source/configuration, secrets, SBOM, CBOM and dependency audit.
The dependency audit uses `--vulndb`, `MAGDOX_VULNDB`, the file pinned with
`magdox vulndb use`, or the platform's database cached locally, in that order.
If none is available (for example `--offline` with nothing cached), SCA is
explicitly `skipped`. Without `--full`, `scan` audits dependencies only when
`--vulndb` is given. AIBOM and malicious-package checking remain separate commands.
Unrequested passes are `not_requested`, not clean.

`scan` now exits **3** when a requested pass is partial or skipped, including
zero useful coverage, unresolved dependency versions and no usable vulnerability
database for `--full`. Review JSON coverage even when no findings were emitted. Existing
severity gates use exit **4**, which takes precedence when a scan is both
incomplete and over the threshold (the incomplete notice is still printed).
Upload policy failures also return **4**. A scan or upload that fails outright
exits **1**; no successful run is fabricated or uploaded.

The server conservatively avoids closing findings when coverage is incomplete
or when rule content, advisory content, test scope or pass statuses differ from
the previous run on that branch. It still records positive observations. A
subsequent comparable complete scan can close absent findings. Existing branch,
chronology, collection-scope and rule-count guards continue to apply. Older
uploads have unknown provenance; their missing fields are not backfilled with
invented evidence.

## Package selected reports without network access

Packing and verification process existing local reports without scanning. The
public launcher still checks product authorization and may renew it online.
To keep these commands offline, use `MAGDOX_OFFLINE=1` with a valid local grant
and an already-installed verified private engine. Offline mode does not bypass
access checks. Generate the required reports before running these commands.

```sh
MAGDOX_OFFLINE=1 magdox evidence pack \
  --repository my-project --commit '<reviewed-revision>' --branch main \
  --scan scan.json --sbom sbom.json --out release-evidence.zip

MAGDOX_OFFLINE=1 magdox evidence verify release-evidence.zip
```

Optional inputs are `--sarif`, `--cbom`, `--aibom`, `--vex` and `--decisions`.
An explicitly supplied `--risk risk.json` can also package a local risk report.
Each must be an existing JSON report. The command does not generate missing
reports, infer a clean result from their absence, or combine their formats.
`--decisions` includes an explicitly supplied JSON record; it does not fetch
dashboard decisions or synchronize triage.

The ZIP contains those exact bytes and a `manifest.json` with artifact types,
sizes, SHA-256 hashes, CLI version, creation time and operator-supplied revision
labels. No source-tree traversal occurs. Inputs can already contain sensitive
source or metadata, so review them before sharing the package. New ZIP files
use owner-only permissions and existing files are not overwritten.

Limits: 32 MiB per input, 128 MiB total. Verification checks hashes, declared
membership and sizes without extracting files. It rejects missing, duplicate,
undeclared and altered artifacts. It does not validate every report's external
schema, authenticate the publisher, verify Git provenance, or sign the package.

## Offline risk assessment and feature boundaries

The local source now supports an opt-in risk layer. Ordinary scan output and
severity gates remain unchanged when these options are not used. Analysis uses
the same versioned, dependency-free priority core as the MAGDOX service; each
side vendors the identical core so it can build independently, and both check
the copy and golden scores. Private scanner rules and their references are not part
of risk reports.

```sh
magdox scan . --full --offline --risk --risk-context context.json \
  --format json --out scan-with-risk.json
magdox scan . --full --offline --risk-context context.json \
  --risk-baseline scan-with-risk.json --format json --out next-scan.json
magdox risk assess --scan next-scan.json --context context.json \
  --decisions local-decisions.json --offline --format html --out risk.html
magdox risk assess --scan next-scan.json --context context.json \
  --format json --out risk.json --fail-on high --fail-priority p1
```

`risk assess` always runs offline, even if `--offline=false` is passed. It reads
existing JSON; it does not scan, renew access, download intelligence or upload.
Normal signed product entitlement is required by both launcher and engine.
`scan --risk` keeps the scan's existing network policy; use `--offline` to stop
renewal and provider calls. Customer perpetual air-gapped licences and renewable
seven-day access retain their existing rules. Risk features grant no new access.

Example local context, with no credentials:

```json
{
  "schema": 1,
  "project": "my-project",
  "environment": "production",
  "businessCriticality": "high",
  "dataSensitivity": "confidential",
  "internetExposed": "unknown",
  "owner": "local-reviewer",
  "team": "local-team"
}
```

Each enum may be `unknown`. Environments are production/staging/development;
criticality is critical/high/medium/low; sensitivity is regulated/confidential/
internal/public; internet exposure is yes/no/unknown. Owner/team are local
accountability labels, not membership, permission or authenticated identity.
With no context file, `scan --risk` uses unknown business context and a stable
opaque local repository label (or the supplied `--repository`). This supports local
baseline comparisons without setup. Re-assessing a scan already containing a
risk snapshot retains its recorded local context unless a new context file is
explicitly supplied. It does not automatically import decision records.

Decisions have `schema: 1`, the same `project`, and an `items` array. Each item
requires a current report's `fingerprint`, a state, `recordedBy` and RFC3339
`recordedAt`. States are untriaged/confirmed/to_fix/false_positive/accepted_risk.
False-positive and accepted-risk declarations require a reason; acceptance also
requires compensating controls and ordered `reviewAt`/`expiresAt`. A `deadline`
is optional. Attribution is local, not verified against an organisation.

Optional decision `evidence` records exposure/reachability (yes/no/unknown),
source, note, recordedBy, observedAt and recordedAt. Future timestamps, evidence
over 30 days old, or a scan newer than the recording stop its ranking points.
Future-dated decisions do not count as verification. Review-due and expired
exceptions remain visible; even current local exceptions never remove findings.

| Feature | Available locally | Authority and limits |
| --- | --- | --- |
| Priority and explanation | Severity, eligible intelligence, local context, human review, deadlines and age | P1–P4 and a capped 0–100 ordering aid, not exploitation probability or monetary impact. Severity/confidence never change. |
| Threat intelligence | Cached CVSS, CVE-only EPSS, known-exploited evidence and snapshot dates | Missing/future/older-than-48-hour enrichment adds no intelligence priority points; historical values remain labelled. Missing legacy zero/false values are unknown. |
| Local ownership, triage and exceptions | Explicit repository-scoped JSON declarations | Not organisation approvals; never uploaded or automatically synchronised. |
| Scan movement | New, persistent and no longer observed | Requires matching repository, policy, opaque scan-scope key, chronology and complete coverage on both sides. Absence is not verified remediation. Legacy reports without scope identity cannot establish closure. |
| Offline reports | Compact text; complete JSON/CSV/XML; filterable, paginated HTML | HTML uses no external assets or network calls. No source snippets or secret-value digests in risk reports. |
| CI gates | Existing `--fail-on`, plus explicitly selected `--fail-priority` | Gates are additive. Local exceptions do not waive them; exit 4 still takes precedence over incomplete exit 3. |
| Evidence packaging | Explicit `--risk` artifact with content checksums | Internal consistency, not signed publisher authenticity or tamper-proof local history. |
| MCP | Read-only `magdox_risk` over existing files | Every input is confined to one configured root; the child anchors reads too. Entitlement remains in the CLI. |
| Teams and governed approvals | Console/API only | Requires current tenant membership and independent approval; offline labels cannot grant authority. |
| Organisation campaigns, reminders and shared metrics | Console/API only | Live recipients, durable delivery outcomes, shared history and tenant scopes require server state. |
| Multi-branch organisation coverage policies | Console/API only | A single local scan cannot certify every required organisation branch. |

Risk outputs created with `--out` are new owner-only files and refuse overwrite.
`scan --risk --format json` retains the ordinary local scan document and adds
the `risk` projection; the original scan portion can still contain source and
local secret digests, as before. Use `risk assess --format json` when sharing a
standalone risk projection without that source data. Neither option uploads
local risk context or decision declarations.
Final input files must be bounded regular files, not links or special devices.
Root-confined agent reads also reject linked parent directories; a supplied
`--root` must be a canonical absolute path without symlink ancestors. Typed
arrays are bounded before allocation. Duplicate JSON properties, aliases for
the same schema field, excessive nesting and malformed schemas fail closed. Local data and
the machine clock remain user-controlled: this is not a tamper-proof audit,
trusted approval record or clock-tamper-proof expiry mechanism.

## Packaged MCP

`magdox_scan` scans source/configuration only. Dependency, secrets and AIBOM
tools remain separate. `include_tests` and `include_code` accept actual JSON
booleans. Scan results include complete coverage, confidence, limitations and
flow metadata. Snippets are omitted unless `include_code: true` is supplied;
that option shares source with the MCP client and potentially its model.
Tool execution failures are marked `isError: true`.

`magdox_risk` takes a required `scan` JSON path and optional `context`, `decisions`
and `baseline` JSON paths. It invokes `risk assess` offline with an anchored
repository root. It cannot write files, fetch rules, upload, mutate governance or
override licence checks. Incomplete and malformed completion remain explicit;
output limits can reject oversized results rather than truncate findings.

## API compatibility

The API accepts payload versions 1 and 2. An older API rejects version 2,
deliberately avoiding silent evidence loss; use an API compatible with your CLI's
payload version. Payload versions are separate from the v1.3.2 release namespace
and the 1.3.2 package version. Existing findings do not gain historical traces
until rescanned.

## Customer upload choice

Metadata-only remains the default. An organisation owner/admin may allow matched source for a repository in Settings > Organisation & members > Repository upload privacy; the scan operator must also pass `--include-code` to upload snippets. This applies to both finding and nested flow snippets. Turning permission off blocks future uploads but does not purge historical copies.

The CLI redacts credentials and query strings in metadata URLs before serializing uploads and upload previews, with server-side redaction retained as a second check. This does not anonymize repository paths or make arbitrary metadata nonsensitive. CI templates omit source sharing. MCP source sharing is a separate client choice.
