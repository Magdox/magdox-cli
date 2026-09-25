# magdox

Command line scanner for [MAGDOX Code Security](https://magdox.io).
It analyses source code, infrastructure definitions, dependencies and secrets
on your own machine. Findings and inventories upload only when requested;
authentication and rule renewal have separate network paths. Source snippets
require explicit sharing consent.

See [scan evidence and release packages](docs/evidence.md) for the working-tree
v2 upload contract, completeness behavior, local packaging and rollout order.

## Install

```sh
brew tap magdox/tap && brew install magdox   # macOS, Linux
npm install -g @magdox/cli            # macOS, Linux, Windows; Node 18+
curl -fsSL https://magdox.io/install.sh | sh   # macOS, Linux
```

On Windows, use npm or the `windows_amd64` zip from the releases page.

The npm package downloads the binary in a postinstall script. If npm skips
install scripts, the first `magdox` run performs the same verified download.
Install with only one of brew or npm: both place a `magdox` on the PATH and
the one found first wins.

Release archives and checksums are on the [releases page](https://github.com/Magdox/magdox-cli/releases).
Every checksum file is signed with Sigstore cosign. The npm installer checks
that signature against the release workflow's identity, then the SHA-256
checksum. The shell installer checks the SHA-256 checksum obtained over HTTPS;
it needs curl, tar and sha256sum or shasum, installs to `/usr/local/bin` (using
sudo only when that directory is not writable), and honours
`MAGDOX_INSTALL_DIR` and `MAGDOX_VERSION`. The CLI itself never needs root:
scan as your ordinary user.

## Use

```sh
magdox login                 # sign in once; downloads the signed rule bundle
magdox scan .                # scan the current project
magdox scan --full .         # code, secrets, dependencies (platform database), SBOM, CBOM
magdox scan --secrets --format json .
magdox scan --format html --out report.html .
magdox scan --project my-app --upload .
magdox aibom --upload --project my-service .
```

Scan supports `--format text|json|csv|xml|html` (default text) and
`--out <file>` to write the report to a file instead of stdout.
CSV includes a BOM header and escapes formula-injection characters.
XML and HTML are self-contained documents suitable for offline archival.

Other commands: `deps`, `audit`, `secrets`, `cbom`, `aibom`, `sarif`, `sbom`,
`vex`, `evidence`, `ai`, `vulndb`, `config`, `license`, `rules`, `token`, `whoami`. Run `magdox help` or `magdox <command> -h`.

Dependency scanning reads lockfiles and manifests for npm (npm, yarn, pnpm,
bun), PyPI (poetry, uv, Pipenv, requirements, pyproject), Go, Maven and
Gradle, NuGet, Cargo, Composer, Bundler and SwiftPM. See
[docs/dependencies.md](docs/dependencies.md) for the files, precedence and
what is reported as unresolved.

Code scans and SARIF exports exit 3 when files fail or rules are unsupported,
while retaining partial findings and coverage. JSON coverage includes up to
128 relative failed paths and reasons. A short report from an incomplete scan
does not establish that the other code is clean.

Scanning uses at most four workers, further bounded by `GOMAXPROCS`. For a
shared machine, `GOMAXPROCS=2 magdox scan ...` limits concurrent native analysis.
The 30-second file budget cancels parsing and checks between queries; this
tree-sitter binding cannot interrupt a single native query while it is running.
Timed-out work stays in its worker and cannot accumulate detached goroutines.

## AI review with your own model

```sh
magdox ai add groq               # or anthropic, openai, gemini, bedrock, azure, ollama, custom
magdox ai models groq && magdox ai use groq --model <id>
magdox ai allow .                # approve this project; nothing is sent before this
magdox scan --ai --ai-fix .      # verdict per finding, fixes re-checked by the engine
```

The engine still decides what is a finding. The model adds an advisory
verdict (likely real, likely false positive, needs review) and a proposed
patch that is marked verified only when a re-scan no longer reports the
finding. Keys stay in your config or environment; code excerpts go only to
the provider you chose, with secrets redacted. See [docs/ai.md](docs/ai.md).

## Offline and configuration

```sh
magdox vulndb status             # which vulnerability database would be used, and how fresh
magdox vulndb fetch              # refresh it from the platform now
magdox vulndb export magdox-vulns.bundle   # write it to a file to carry to an air-gapped machine
magdox vulndb use ./magdox-vulns.bundle    # pin a local file (air-gapped machines); --clear to undo
magdox scan --offline .          # cached rules and database only, no network at all
magdox config show               # every effective setting and where it comes from
```

Environment equivalents: `MAGDOX_ENDPOINT`, `MAGDOX_OFFLINE=1`, `MAGDOX_VULNDB`,
`MAGDOX_TOKEN`, `MAGDOX_CONFIG_DIR`, `MAGDOX_CACHE_DIR`, `MAGDOX_AI_PROVIDER`,
`MAGDOX_AI_CONSENT=1`, `MAGDOX_AI_DEBUG=1`. See [docs/offline.md](docs/offline.md). The CLI never needs
an NVD API key: the platform builds the database.

## What leaves your machine

Source analysis runs locally. Rule syncing may contact the configured platform.
With `--upload`, a findings payload contains rule id, relative file path, line,
severity and fingerprint, plus requested inventories. Code snippets require
`--include-code`; secret findings omit their values. URL dependency versions
are replaced with `[redacted-url]` before reporting, including query credentials.
`--show-payload` prints one JSON document and always prevents upload, even when
combined with `--upload`.
The schema is in `schema/`.

## How rules arrive

Detection rules are downloaded as a bundle signed with ed25519. The CLI holds
the public key and refuses any bundle whose signature or digest does not
verify. Rules are not part of this repository.
Cached archives are reverified on every load. Older caches without a signed
manifest must be refreshed with `magdox rules sync`. Credentials and device keys
must be private regular files; linked or group/world-readable files are rejected.

Repository traversal is anchored to an opened directory and does not follow
repository symlinks. Reads are limited to 2 MiB per source/certificate file and
16 MiB per recognized dependency manifest. Traversal stops with an error above
200,000 entries; a directory nested deeper than 128 levels is skipped and the
scan is marked incomplete. Missing or unreadable targets fail explicitly.

The MCP server for AI coding assistants is a separate package: see
[docs/mcp-setup.md](docs/mcp-setup.md) and [docs/mcp.md](docs/mcp.md).

## Licence

Apache-2.0. See `LICENSE` and `THIRD_PARTY_NOTICES.md`.

Security reports: security@magdox.io (see `SECURITY.md`).
