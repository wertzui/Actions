# Actions

Reusable GitHub Actions and Workflows for .NET projects — covering version checking, NuGet publishing, and full CI pipelines.

---

## Contents

| Name | Type | Description |
| ---- | ---- | ----------- |
| [`nuget-build`](#nuget-build-workflow) | Reusable Workflow | Build, test, pack, and publish .NET NuGet packages end-to-end |
| [`install-dotnet-version`](#install-dotnet-version) | Composite Action | Install the `dotnet-version` CLI tool from NuGet or a local build |
| [`check-dotnet-version`](#check-dotnet-version) | Composite Action | Verify all packaged projects have version bumps since a base ref |
| [`read-dotnet-version`](#read-dotnet-version) | Composite Action | Read version metadata and generate git tags and release notes |
| [`publish-dotnet-packages`](#publish-dotnet-packages) | Composite Action | Collect `.nupkg` files, log in via OIDC, and push to NuGet.org |

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

## License

[MIT](LICENSE)
