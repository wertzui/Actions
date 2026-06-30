# Actions

Reusable GitHub Actions and Workflows for .NET projects — covering version checking, NuGet publishing, and full CI pipelines.

---

## Contents

| Name | Type | Description |
| ---- | ---- | ----------- |
| [`nuget-build`](#nuget-build-workflow) | Reusable Workflow | Build, test, pack, and publish .NET NuGet packages end-to-end |
| [`npm-build`](#npm-build-workflow) | Reusable Workflow | Build, test, pack, and publish npm packages end-to-end |
| [`install-dotnet-version`](#install-dotnet-version) | Composite Action | Install the `dotnet-version` CLI tool from NuGet or a local build |
| [`check-dotnet-version`](#check-dotnet-version) | Composite Action | Verify all packaged projects have version bumps since a base ref |
| [`read-dotnet-version`](#read-dotnet-version) | Composite Action | Read version metadata and generate git tags and release notes |
| [`publish-dotnet-packages`](#publish-dotnet-packages) | Composite Action | Collect `.nupkg` files, log in via OIDC, and push to NuGet.org |
| [`publish-npm-packages`](#publish-npm-packages) | Composite Action | Discover, pack, and publish npm packages via OIDC trusted publishing or a token |

---

## Reusable Workflow

### `nuget-build` Workflow

A complete end-to-end CI/CD pipeline: restore → build → test → version-check → pack → publish → GitHub release.

#### Usage

```yaml
jobs:
  release:
    uses: wertzui/Actions/.github/workflows/nuget-build.yml@main
    with:
      solution_file: MySolution.slnx   # required — path to your .sln or .slnx
      # dotnet_version: '10.0.x'       # optional — default: '10.0.x'
      # filter: 'GeneratePackageOnBuild=true'  # optional — selects which projects to publish
      # base: 'main'                   # optional — base ref for version-bump check
    secrets:
      nuget_user: ${{ secrets.NUGET_USER }}
```

#### Required permissions on the calling job

```yaml
permissions:
  contents: write   # push git tags and create GitHub releases
  id-token: write   # NuGet OIDC login
```

#### Inputs

| Input | Required | Default | Description |
| ----- | -------- | ------- | ----------- |
| `solution_file` | ✅ | — | Path to the `.sln` or `.slnx` solution file |
| `dotnet_version` | ❌ | `10.0.x` | .NET SDK version to set up |
| `filter` | ❌ | `GeneratePackageOnBuild=true` | `dotnet-version` filter to select packaged projects |
| `tool_version` | ❌ | *(latest)* | Version of `DotnetVersionReader` to install; leave empty for latest |
| `base` | ❌ | `main` | Git ref to compare versions against for the version-bump check |

#### Secrets

| Secret | Required | Description |
| ------ | -------- | ----------- |
| `nuget_user` | ✅ | NuGet username for OIDC login |

---

### `npm-build` Workflow

A complete end-to-end CI/CD pipeline for npm packages:
install → build → test → version-check → pack → publish → GitHub release.

Authentication uses **npm OIDC trusted publishing by default** — no token and **no encrypt/decrypt key
hand-off** (unlike `nuget-build`). npm validates OIDC against the **calling (top-level) workflow**, not this
reusable workflow, so you add your **real caller workflow** as the Trusted Publisher on npmjs.com and this
reusable workflow is never trusted. See
[Using inside a reusable workflow](#using-inside-a-reusable-workflow-oidc) for the full rationale.

#### Usage (OIDC — recommended)

```yaml
jobs:
  release:
    permissions:
      contents: write   # push git tags and create GitHub releases
      id-token: write   # npm OIDC trusted publishing + provenance
    uses: wertzui/Actions/.github/workflows/npm-build.yml@main
    with:
      working_directory: .       # optional — default: '.'
      # node_version: '22.x'     # optional — default: '22.x'
      # access: 'public'         # optional — default: 'public'
```

#### Usage (token-based auth)

```yaml
jobs:
  release:
    permissions:
      contents: write
      id-token: write            # still needed for provenance
    uses: wertzui/Actions/.github/workflows/npm-build.yml@main
    secrets:
      npm_token: ${{ secrets.NPM_TOKEN }}
```

#### Required permissions on the calling job

```yaml
permissions:
  contents: write   # push git tags and create GitHub releases
  id-token: write   # npm OIDC trusted publishing and/or provenance
```

#### Inputs

| Input | Required | Default | Description |
| ----- | -------- | ------- | ----------- |
| `node_version` | ❌ | `22.x` | Node.js version to set up |
| `working_directory` | ❌ | `.` | Directory to install/build/test in and to search for packages |
| `registry` | ❌ | `https://registry.npmjs.org` | Registry URL to publish to |
| `access` | ❌ | `public` | Access for scoped packages: `public` or `restricted` |
| `provenance` | ❌ | `true` | Publish with provenance (requires `id-token: write`) |
| `base` | ❌ | `main` | Git ref to compare versions against (pull-request version-bump check) |
| `install_command` | ❌ | `npm ci` | Command used to install dependencies |
| `build_command` | ❌ | `npm run build --if-present` | Command used to build the package(s) |
| `test_command` | ❌ | `npm test` | Command used to run tests |

#### Secrets

| Secret | Required | Description |
| ------ | -------- | ----------- |
| `npm_token` | ❌ | npm token. Omit to use OIDC trusted publishing |

> The version-bump check runs on `pull_request` events; tagging, publishing, and the GitHub release run on
> pushes to `main`.

---

## Composite Actions

### `install-dotnet-version`

Makes the [`dotnet-version`](https://github.com/wertzui/DotnetVersionReader) CLI tool available on `PATH` by installing it from NuGet.org or by locating a locally-built binary.

#### Usage

```yaml
- uses: wertzui/Actions/.github/actions/install-dotnet-version@main
  with:
    version: ''             # optional — leave empty for latest stable
    use_local_build: false  # optional — use a binary already built in the workspace
```

#### Inputs

| Input | Required | Default | Description |
| ----- | -------- | ------- | ----------- |
| `version` | ❌ | *(latest)* | Version to install from NuGet.org. Ignored when `use_local_build` is `true` |
| `use_local_build` | ❌ | `false` | Skip NuGet install; find the built executable in the workspace and add it to `PATH` |

---

### `check-dotnet-version`

Compares the version numbers of all packaged projects against a base git ref. Fails the step and writes a job summary if any project has not been bumped.

#### Usage

```yaml
- uses: wertzui/Actions/.github/actions/check-dotnet-version@main
  with:
    input: MySolution.slnx          # required
    base: main                      # required — tag, branch, or SHA to compare against
    filter: 'GeneratePackageOnBuild=true'  # optional
    output: table                   # optional — table | json | csv
```

#### Inputs

| Input | Required | Default | Description |
| ----- | -------- | ------- | ----------- |
| `input` | ✅ | — | Path to the `.slnx` or `.csproj` file |
| `base` | ✅ | — | Git ref to compare versions against |
| `filter` | ❌ | *(none)* | Property filter expression (e.g. `GeneratePackageOnBuild=true`) |
| `output` | ❌ | `table` | Output format: `table`, `json`, or `csv` |

---

### `read-dotnet-version`

Reads version metadata from a solution or project file. Optionally git-tags the current commit and produces structured outputs ready for use in a GitHub release step.

#### Usage

```yaml
- uses: wertzui/Actions/.github/actions/read-dotnet-version@main
  id: read-versions
  with:
    input: MySolution.slnx
    filter: 'GeneratePackageOnBuild=true'
    create_tags: 'true'
```

#### Inputs

| Input | Required | Default | Description |
| ----- | -------- | ------- | ----------- |
| `input` | ✅ | — | Path to the `.slnx` or `.csproj` file |
| `filter` | ❌ | *(none)* | Property filter expression |
| `create_tags` | ❌ | `false` | When `true`, git-tags the current commit for each package and creates a combined release tag |

#### Outputs

| Output | Description |
| ------ | ----------- |
| `json` | JSON array of `{ Name, Version }` objects for all matched projects |
| `tags` | Newline-separated list of individual package tags (`Name-vVersion`) |
| `release_tag` | Combined release tag, e.g. `release-MyPkg-v1.2.3` |
| `release_name` | Human-readable release name, e.g. `Release: MyPkg 1.2.3` |
| `release_body` | Markdown-formatted release body listing all packages and their versions |

---

### `publish-dotnet-packages`

Collects `.nupkg` and `.snupkg` files from the workspace, authenticates to NuGet.org via OIDC, and pushes all packages. Outputs the file list for use in a subsequent GitHub release step.

#### Usage

```yaml
- uses: wertzui/Actions/.github/actions/publish-dotnet-packages@main
  id: publish
  with:
    nuget_user: ${{ secrets.NUGET_USER }}
```

#### Inputs

| Input | Required | Description |
| ----- | -------- | ----------- |
| `nuget_user` | ✅ | NuGet username for OIDC login (typically a repository secret) |

#### Outputs

| Output | Description |
| ------ | ----------- |
| `package_files` | Newline-separated list of all `.nupkg` and `.snupkg` file paths that were published |

---

### `publish-npm-packages`

Discovers publishable `package.json` files in the workspace (recursively, skipping `node_modules` and any
package marked `"private": true`), packs each into a `.tgz` tarball, and publishes them to the npm registry.
Outputs the tarball file list for use in a subsequent GitHub release step.

Authentication uses **npm OIDC trusted publishing by default** — no secret required. Unlike the NuGet flow,
npm needs **no `encrypt-value`/`decrypt-value` plumbing**: the npm CLI performs the OIDC token exchange inline
at publish time (nothing crosses a job boundary), so there is no short-lived key to smuggle between jobs.
Provenance is generated automatically. Alternatively, supply an `npm_token` for classic token-based auth.

#### Usage (OIDC trusted publishing — recommended)

```yaml
jobs:
  publish:
    runs-on: ubuntu-latest
    permissions:
      contents: write   # create GitHub releases
      id-token: write   # npm OIDC trusted publishing + provenance
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '22.x'
          registry-url: 'https://registry.npmjs.org'
      - run: npm ci
      - run: npm run build --if-present
      - uses: wertzui/Actions/.github/actions/publish-npm-packages@main
        id: publish
      - uses: softprops/action-gh-release@v2
        with:
          files: ${{ steps.publish.outputs.package_files }}
```

> A Trusted Publisher must be configured for each package on
> **npmjs.com → Package → Settings → Trusted Publishing** (provider: GitHub Actions, pointing at this
> repository and workflow). No secret is stored anywhere.

#### Usage (token-based auth)

```yaml
- uses: wertzui/Actions/.github/actions/publish-npm-packages@main
  id: publish
  with:
    npm_token: ${{ secrets.NPM_TOKEN }}
```

#### Inputs

| Input | Required | Default | Description |
| ----- | -------- | ------- | ----------- |
| `npm_token` | ❌ | *(empty)* | npm automation/granular token. Leave empty to use OIDC trusted publishing |
| `registry` | ❌ | `https://registry.npmjs.org` | Registry URL to publish to |
| `access` | ❌ | `public` | Access for scoped packages: `public` or `restricted` |
| `provenance` | ❌ | `true` | Publish with `--provenance` (requires `id-token: write`; ignored on dry runs) |
| `dry_run` | ❌ | `false` | Run `npm publish --dry-run` without actually publishing |
| `working_directory` | ❌ | `.` | Root directory to search recursively for publishable packages |

#### Outputs

| Output | Description |
| ------ | ----------- |
| `package_files` | Newline-separated list of all packed `.tgz` tarball paths that were published |
| `published_packages` | Newline-separated list of published packages in `name@version` format |

#### Required permissions on the calling job

```yaml
permissions:
  contents: write   # if creating a GitHub release with the tarballs
  id-token: write   # OIDC trusted publishing and/or provenance
```

#### Using inside a reusable workflow (OIDC)

Unlike NuGet, npm trusted publishing does **not** require any encrypt/decrypt token hand-off when this
action runs inside a reusable workflow. npm validates OIDC against the **calling (top-level) workflow**, not
the reusable workflow that actually runs `npm publish`
([npm docs](https://docs.npmjs.com/trusted-publishers/)). NuGet's `NuGet/login` instead binds to the
reusable workflow (`job_workflow_ref`), which is why the NuGet flow must mint the key in the caller and pass
it across via `encrypt-value`/`decrypt-value`.

Practical consequences:

- Configure the **Trusted Publisher** on npmjs.com with your **real caller workflow** (the caller repository
  and its entry workflow filename, e.g. `release.yml`). The reusable workflow is **never** added as trusted.
- Grant `id-token: write` in **both** the caller job that invokes the reusable workflow **and** the reusable
  workflow's publish job — both are required.
- `package.json`'s `repository.url` must match the caller's GitHub repository.

```yaml
# Caller workflow (.github/workflows/release.yml) — this is the one you add as a Trusted Publisher on npm.
jobs:
  release:
    permissions:
      contents: write
      id-token: write          # propagated to the reusable workflow's OIDC request
    uses: wertzui/Actions/.github/workflows/npm-build.yml@main
    # ...
```

---

## License

[MIT](LICENSE)
