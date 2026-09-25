# Offline and air-gapped use

Matching never needs the network. The CLI needs two local files: the signed
rule bundle and the vulnerability database. Online, both are fetched and
cached automatically; offline, the cached copies are used.

## Flags and variables

| Setting | Flag | Environment | Default |
|---|---|---|---|
| Platform endpoint | `--endpoint` | `MAGDOX_ENDPOINT` | `https://api.magdox.io` |
| Offline mode | `--offline` (scan, audit, vex) | `MAGDOX_OFFLINE=1` | off |
| Vulnerability database | `--vulndb <file>` | `MAGDOX_VULNDB` | pinned file, else the platform's cached copy |
| Rules | `--rules <dir>` | | the synced signed bundle |
| CI token | | `MAGDOX_TOKEN` | stored sign-in |
| Config directory | | `MAGDOX_CONFIG_DIR` | the OS config directory |
| Cache directory | | `MAGDOX_CACHE_DIR` | the OS cache directory |
| AI provider for one run | | `MAGDOX_AI_PROVIDER` | the active provider |
| AI consent in CI | | `MAGDOX_AI_CONSENT=1` | per-project approval |

`magdox config show` prints every effective value and its source.

## Offline mode

`--offline` never contacts the platform or a remote AI provider:

- rules come from the cache; with no usable cache the command fails and
  names the cache directory (run `magdox rules sync` once online, or pass
  `--rules`);
- the vulnerability database comes from `--vulndb`, `MAGDOX_VULNDB`, the file
  pinned with `magdox vulndb use`, or the cached copy, in that order;
- `--upload` is refused;
- `--ai` works only with a local provider (for example `magdox ai add ollama`).

## Air-gapped machines

1. On a connected machine: `magdox login`, `magdox rules sync`,
   `magdox vulndb fetch`, then `magdox vulndb export magdox-vulns.bundle`
   to write the current bundle to a file you can carry across. Export also
   writes `magdox-vulns.bundle.sig`; carry both files.
2. On the air-gapped machine: `magdox vulndb use <file>` with the `.sig` in the
   same folder, and a perpetual,
   air-gap eligible licence (`magdox license import`).
3. Run with `--offline`. Every scan records the database's build date and
   digest in its provenance, so an old copy shows as old in the console rather
   than passing silently.

## Signed databases

The platform signs each bundle with the same MAGDOX key that signs rule
bundles. The CLI verifies the signature when it downloads the bundle and every
time it opens one: a cached copy that was fetched signed is refused if its
signature is missing or does not match, and a pinned file with a `.sig` beside
it must verify. A bundle with no signature at all (an older platform, or one
you built yourself) still works, and `magdox audit` and `magdox vulndb status`
say plainly that it cannot be shown to be unaltered.

## Building your own database

Customers do not need an NVD API key: the platform builds and serves the
database. Self-hosters who run the `vulndb` builder themselves may set
`NVD_API_KEY` for it to sync faster than NVD's public rate limit.
