# JFrog Shared Actions

Reusable CI/CD templates and composite actions for [JFrog Frogbot](https://docs.jfrog.com/applications/frogbot) and JFrog Platform integrations across **GitHub Actions**, **Azure DevOps**, and **GitLab CI**.

This repository centralizes hardened, opinionated pipeline patterns so teams can adopt Frogbot and Artifactory workflows consistently — including air-gapped environments where binaries and dependencies must be resolved from internal Artifactory rather than the public internet.

## Goals

- **Cross-platform reuse** — One source of truth for Frogbot and JFrog CLI patterns on GitHub Actions, Azure DevOps, and GitLab runners.
- **Frogbot v2 and v3** — Templates and actions that work with both Frogbot major versions, with version pinned via parameters or inputs.
- **Air-gap ready** — Redirect analyzer downloads (`JF_RELEASES_REPO`), dependency resolution (`JF_DEPS_REPO`), and package restores through Artifactory.
- **Audit-friendly** — Documented environment variables, explicit scan behaviour, and no reliance on undocumented Frogbot flags.

## Repository layout

```
jfrog-shared-actions/
├── azure-devops/          # Azure Pipelines step templates
│   └── frogbot-repo-scan-NuGet.yml
├── github-actions/        # (planned) GitHub Actions workflows & reusable jobs
├── gitlab/                # (planned) GitLab CI includes & templates
└── maven/                 # GitHub composite actions for Maven + JFrog CLI
    ├── setup/
    │   └── action.yaml
    └── publish/
        └── action.yaml
```

> **Note:** GitHub composite actions currently live under `maven/` at the repo root. A `github-actions/` directory will be added as Frogbot and other ecosystem templates are migrated here.

---

## Available templates

### Azure DevOps — Frogbot NuGet repository scan

**File:** [`azure-devops/frogbot-repo-scan-NuGet.yml`](azure-devops/frogbot-repo-scan-NuGet.yml)

Reusable step template for scanning .NET / NuGet repositories with Frogbot on Azure Repos. Designed for air-gapped networks and the **Frogbot v2** binary distribution model (download `frogbot.exe` from Artifactory, then run `scan-repository`).

| Parameter | Default | Description |
|-----------|---------|-------------|
| `frogbotVersion` | `2.31.0` | Frogbot release to download from Artifactory |
| `solutionName` | `netcore-template.sln` | Solution file passed to `dotnet restore` |
| `sdkVersion` | `8.0.x` | .NET SDK version installed via `UseDotNet@2` |

**Required variable group:** `Jfrog-FrogBot`

| Variable | Description |
|----------|-------------|
| `JF_URL` | JFrog Platform base URL |
| `JF_ACCESS_TOKEN` | JFrog access token (secret) |
| `JF_GIT_TOKEN` | Azure Repos PAT for Frogbot PRs (secret) |
| `JF_RELEASES_REPO` | Artifactory repo proxying `releases.jfrog.io` (for analyzer binaries) |

**Usage:**

```yaml
# azure-pipelines.yml
trigger:
  - main

variables:
  - group: Jfrog-FrogBot

steps:
  - template: azure-devops/frogbot-repo-scan-NuGet.yml@jfrog-shared-actions
    parameters:
      frogbotVersion: "2.31.0"
      solutionName: "MyApp.sln"
      sdkVersion: "8.0.x"
```

**What the template does:**

1. Installs the .NET SDK
2. Restores NuGet packages via `JFrogNuget@1` (Artifactory)
3. Downloads the Frogbot binary from internal Artifactory (`JFrogGenericArtifacts@1`)
4. Runs `frogbot scan-repository` with Azure Repos and air-gap environment variables

Key environment variables set by the template:

| Variable | Purpose |
|----------|---------|
| `JF_GIT_PROVIDER` | `azureRepos` |
| `JF_INSTALL_DEPS_CMD` | `dotnet restore <solution>` — required for NuGet/.NET scans |
| `JF_DEPS_REPO` | Artifactory NuGet repo for dependency resolution |
| `JF_RELEASES_REPO` | Internal proxy for Frogbot analyzer binaries |
| `JF_FAIL` | `"false"` — scan continues on findings (string, not boolean) |

---

### GitHub Actions — Maven setup

**Path:** [`maven/setup/action.yaml`](maven/setup/action.yaml)

Composite action that installs JFrog CLI (OIDC), configures Maven resolution against Artifactory, and optionally exports build-info environment variables.

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `jfrog-url` | yes | — | JFrog Platform URL |
| `jfrog-user` | yes | — | Artifactory username (for Maven credentials) |
| `jfrog-oidc-provider-name` | yes | — | OIDC provider configured in JFrog Platform |
| `jfrog-access-token` | no | — | Access token (alternative to OIDC for Maven auth) |
| `java-version` | no | `25` | Java version for `setup-java` |
| `maven-version` | no | `3.8.8` | Maven version |
| `resolve-repo` | no | `""` | Artifactory repo for Maven resolve/deploy |
| `build-name` | no | — | Build name for JFrog build-info |
| `build-number` | no | — | Build number for JFrog build-info |
| `cache-maven` | no | `true` | Enable Maven dependency cache |

**Usage:**

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      id-token: write   # required for OIDC
      contents: read
    steps:
      - uses: actions/checkout@v4

      - name: JFrog Maven setup
        uses: chaitanyagovande/jfrog-shared-actions/maven/setup@main
        with:
          jfrog-url: ${{ vars.JFROG_URL }}
          jfrog-user: ${{ vars.JFROG_USER }}
          jfrog-oidc-provider-name: ${{ vars.JFROG_OIDC_PROVIDER }}
          resolve-repo: my-maven-virtual
          build-name: my-app
          build-number: ${{ github.run_number }}

      - name: Build
        run: mvn -B clean package
```

---

### GitHub Actions — Maven publish

**Path:** [`maven/publish/action.yaml`](maven/publish/action.yaml)

Publishes build-info to Artifactory and optionally runs an Xray build scan. Use after `maven/setup` and a Maven build step.

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `jfrog-url` | yes | — | JFrog Platform URL |
| `build-name` | yes | — | Build name |
| `build-number` | yes | — | Build number |
| `jfrog-oidc-provider-name` | no | — | OIDC provider name |
| `jfrog-access-token` | no | — | Access token |
| `xray-scan` | no | `false` | Run `jf build-scan` after publish |
| `xray-fail-on-violation` | no | `false` | Fail the step on Xray violations |
| `capture-maven-sbom` | no | `true` | Capture Maven dependency tree as SBOM |

**Usage:**

```yaml
      - name: Publish to Artifactory
        uses: chaitanyagovande/jfrog-shared-actions/maven/publish@main
        with:
          jfrog-url: ${{ vars.JFROG_URL }}
          build-name: my-app
          build-number: ${{ github.run_number }}
          xray-scan: "true"
          xray-fail-on-violation: "false"
```

---

## Frogbot v2 vs v3

| | Frogbot v2 | Frogbot v3 |
|---|-----------|-----------|
| **Distribution** | Standalone binary (`frogbot.exe` / `frogbot`) downloaded from Artifactory or `releases.jfrog.io` | Typically invoked via JFrog CLI (`jfrog frogbot …`) |
| **Azure DevOps** | [`frogbot-repo-scan-NuGet.yml`](azure-devops/frogbot-repo-scan-NuGet.yml) — pin version via `frogbotVersion` parameter | Planned |
| **GitHub Actions** | Planned reusable workflow | Planned composite action / workflow |
| **GitLab CI** | Planned `.gitlab-ci.yml` include | Planned |

When adding new templates, pin the Frogbot version explicitly and document which JFrog Platform and CLI versions were validated.

---

## Air-gapped / enterprise setup

Templates in this repo assume the following Artifactory configuration:

1. **`JF_RELEASES_REPO`** — Remote repository proxying `https://releases.jfrog.io` with **Store Artifacts Locally** unchecked, so Frogbot analyzer binaries are fetched through Artifactory.
2. **`JF_DEPS_REPO`** — Virtual or remote NuGet/Maven/npm repo for resolving project dependencies without reaching the public internet.
3. **Service connection / OIDC** — Azure DevOps service connection (`Artifactory-V2`) or GitHub OIDC integration with JFrog Platform.

See [JFrog Azure DevOps documentation](https://docs.jfrog.com/applications/frogbot/frogbot-for-azure-devops) and [Frogbot environment variables](https://docs.jfrog.com/applications/frogbot/frogbot-environment-variables) for full reference.

---

## Roadmap

- [ ] GitHub Actions — Frogbot v2 repository scan (reusable workflow)
- [ ] GitHub Actions — Frogbot v3 via JFrog CLI
- [ ] GitLab CI — Frogbot v2 / v3 includes for NuGet, Maven, npm
- [ ] Azure DevOps — Frogbot v3 template
- [ ] Additional ecosystems: Maven, npm, Gradle, Docker

Contributions and issue reports are welcome. When opening a PR, include the Frogbot version, CI platform, and ecosystem tested.

## References

- [Frogbot documentation](https://docs.jfrog.com/applications/frogbot)
- [Frogbot environment variables](https://docs.jfrog.com/applications/frogbot/frogbot-environment-variables)
- [JFrog CLI setup action](https://github.com/jfrog/setup-jfrog-cli)
- [Frogbot for Azure DevOps](https://docs.jfrog.com/applications/frogbot/frogbot-for-azure-devops)
