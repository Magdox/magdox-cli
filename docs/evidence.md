# Scan evidence and release packages

This describes the implementation in this working tree. Deploy the updated API
before distributing the CLI that sends payload version 2.

## Review locally, then preview an optional upload

```sh
magdox scan . --format json --out scan.json
magdox scan . --show-payload --project my-project > upload-preview.json
magdox scan . --upload --project my-project
```

Scanning still requires the existing product entitlement and rules. A preview
does not upload findings, including when `--upload` is also supplied.
Authentication and rule renewal have their own network behavior. AIBOM uses a
separate upload path; this preview does not cover it.

Local JSON retains source context. Uploads retain finding location, confidence,
limits, captured rule guidance and ordered same-file flow evidence. Source
snippets, including nested flow snippets, require `--include-code` and a project
that permits snippets. The upload builder removes customer function names from
generated interprocedural flow labels. Detailed parser/OS failure text stays
local; uploaded failure entries contain relative paths and a generic reason.

Dependencies retain positive EPSS scores, KEV flags, aliases, all reported fixed
versions and advisory guidance. Legacy advisory bundles cannot distinguish a
missing EPSS value from zero; zero is therefore represented as unreported. A
missing KEV flag does not establish safety. These are local bundle snapshots,
not live intelligence queries.

## Read completeness before finding counts

JSON scan output contains `provenance`; uploads carry the same data under
`coverage.provenance`. It records execution start/end, decoded rule-content
digest, decoded advisory-content digest/build time when used, test scope and
the status of each pass. Server receipt time is separate.

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
severity gates use exit **4**. Upload policy failures can also return **4**.
Fatal errors still stop the command; no successful run is fabricated or uploaded.

The server conservatively avoids closing findings when coverage is incomplete
or when rule content, advisory content, test scope or pass statuses differ from
the previous run on that branch. It still records positive observations. A
subsequent comparable complete scan can close absent findings. Existing branch,
chronology, collection-scope and rule-count guards continue to apply. Older
uploads have unknown provenance; their missing fields are not backfilled with
invented evidence.

## Package selected reports without network access

```sh
magdox evidence pack \
  --project my-project --commit <reviewed-revision> --branch main \
  --scan scan.json --sbom sbom.json --out release-evidence.zip

magdox evidence verify release-evidence.zip
```

Optional inputs are `--sarif`, `--cbom`, `--aibom`, `--vex` and `--decisions`.
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

## Packaged MCP

`magdox_scan` scans source/configuration only. Dependency, secrets and AIBOM
tools remain separate. `include_tests` and `include_code` accept actual JSON
booleans. Scan results include complete coverage, confidence, limitations and
flow metadata. Snippets are omitted unless `include_code: true` is supplied;
that option shares source with the MCP client and potentially its model.
Tool execution failures are marked `isError: true`.

## Server rollout

1. Back up the database using the existing operational process.
2. Start the updated API; the existing migration runner applies
   `0026_scan_evidence.sql` (additive JSON evidence, line range and coverage columns).
3. Deploy the updated web console.
4. Distribute the v2 CLI and packaged MCP build.

The API accepts v1 and v2 uploads. An older API rejects v2, deliberately avoiding
silent evidence loss. Roll back CLI distribution to v1 before rolling back the
API; retain the additive columns to preserve stored evidence. Existing findings
do not gain historical traces until rescanned. No live database or deployment
was changed as part of this implementation.

## Customer upload choice

Metadata-only remains the default. An organisation owner/admin may allow matched source for a repository in Settings > Organisation & members > Repository upload privacy; the scan operator must also pass `--include-code` to upload snippets. This applies to both finding and nested flow snippets. Turning permission off blocks future uploads but does not purge historical copies.

The CLI redacts credentials and query strings in metadata URLs before serializing uploads and upload previews, with server-side redaction retained as a second check. This does not anonymize repository paths or make arbitrary metadata nonsensitive. CI templates omit source sharing. MCP source sharing is a separate client choice.
