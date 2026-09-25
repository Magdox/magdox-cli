# Dependency scanning

`magdox deps`, `magdox audit`, `magdox sbom`, `magdox vex` and `magdox scan --full`
read the same inventory. Each file below is found anywhere in the tree, except
under `node_modules`, `vendor`, virtual environments, build output (`dist`,
`build`, `target`, `obj`, `.build`, `Pods`) and dot-directories.

| Ecosystem | Files, most exact first | Package URL |
|-----------|-------------------------|-------------|
| npm | `package-lock.json`, `yarn.lock` (classic and Berry), `pnpm-lock.yaml` (v5 to v9), `bun.lock` | `pkg:npm` |
| PyPI | `poetry.lock`, `uv.lock`, `Pipfile.lock`, `requirements.txt`, `pyproject.toml` (PEP 621 and Poetry) | `pkg:pypi` |
| Go | `go.sum` with `go.mod` | `pkg:golang` |
| Maven | `gradle.lockfile`, `pom.xml`, `build.gradle`, `build.gradle.kts` | `pkg:maven` |
| NuGet | `packages.lock.json`, `*.csproj` / `*.fsproj` / `*.vbproj` with `Directory.Packages.props`, `packages.config` | `pkg:nuget` |
| crates.io | `Cargo.lock` | `pkg:cargo` |
| Packagist | `composer.lock` | `pkg:composer` |
| RubyGems | `Gemfile.lock` | `pkg:gem` |
| Swift | `Package.resolved` (v1, v2, v3) | `pkg:swift` |

A lockfile wins over the manifest beside it: `pyproject.toml` is not read when
`poetry.lock`, `uv.lock` or `Pipfile.lock` sits next to it, a project file when
`packages.lock.json` does, and `build.gradle` when `gradle.lockfile` does. The
same package read twice from one directory is reported once.

**Direct or transitive.** A dependency is marked direct when its format says so:
the lockfile itself (`package-lock.json`, `pnpm-lock.yaml`, `bun.lock`,
`packages.lock.json`, `Gemfile.lock`) or the manifest beside it (`package.json`
for `yarn.lock`, `pyproject.toml` for `poetry.lock` and `uv.lock`, `Pipfile` for
`Pipfile.lock`, `composer.json` for `composer.lock`, `go.mod` for `go.sum`).
`gradle.lockfile`, `Cargo.lock`, `packages.config` and `Package.resolved` do not
say, so their packages are reported as transitive.

**Unresolved.** Only an exact version is matched against advisories. A range,
a floating version (`1.*`, `latest.release`), a version held in a variable the
file does not define, or a Maven version inherited from an external parent or
BOM is listed under `unresolved` instead of guessed. `pom.xml`, `build.gradle`
and project files without a lockfile also say that their transitive
dependencies were not resolved. Local, workspace, path and git sources are the
project's own code or not on a registry, so they are skipped. A file that
cannot be parsed is listed under `unresolved` too, and the rest of the
inventory is still read.

**Version order** follows each ecosystem's own rules: PEP 440 for PyPI, Maven's
ComparableVersion (`1.0-SNAPSHOT < 1.0 < 1.0-sp`, `5.3.18.RELEASE = 5.3.18`),
RubyGems' `Gem::Version` (`1.0.0.rc1 < 1.0.0`), and semantic versioning with
four-part NuGet versions for the rest.

Not read today: GitHub Actions `uses:` references, Gradle version catalogs,
`Podfile.lock`, and ecosystems the vulnerability database does not index.
