# Idealogic shared GitHub Actions workflows

Reusable workflows for the account's repositories: Maven and Angular builds, tests, Docker and
Helm publishing, ArgoCD manifest updates, GitFlow releases, and the software factory's
pull-request lifecycle route (Copilot review request, merge-eligibility label, Resolved at merge,
release marking in Azure DevOps). They moved here from `christhonie/event` on 2026-10-28 so that
public repositories can call them too: a public repository cannot call a reusable workflow that
lives in a private one.

Call one with

```yaml
jobs:
  package:
    uses: christhonie/idealogic-workflows/.github/workflows/maven-package.yml@main
    with:
      maven-profiles: prod
    secrets:
      MAVEN_PASSWORD: ${{ secrets.EVENT_PACKAGE_REPO_TOKEN }}
```

`christhonie/event` keeps a thin wrapper per workflow that calls the copy here, so callers that
still reference `christhonie/event/.github/workflows/<file>@main` keep working and move when
they are next touched.

## Account constants

One account uses these workflows, so the values that would differ for another account are not
inputs yet. Each sits in an `env:` block at the top of the workflow that uses it, with a comment
saying so; turning one into a `workflow_call` input with that value as its default is the whole
change. They are: the Maven server id and user for GitHub Packages (`MAVEN_SERVER_ID`,
`MAVEN_USERNAME`), the npm scope (`NPM_SCOPE`), the Docker Hub user and the Helm plugin's Docker
Hub server id (`DOCKER_HUB_USER`, `DOCKER_SERVER_ID`), and the factory dispatcher's Entra
application and tenant ids (`AZURE_CLIENT_ID`, `AZURE_TENANT_ID`). The ArgoCD repository is
already an input with a default (`argocd-repo`), as are the Azure DevOps organisation and project
of the route workflows (`ado-organisation`, `ado-project`). None of these is a secret.

## Workflows

| File | Purpose |
|------|---------|
| `test-be.yml` | Run back-end tests (enforcer, JavaDoc, verify); with `sonar: true`, publish results and coverage to SonarQube |
| `test-fe.yml` | Run front-end tests (npm test) |
| `check-format.yml` | Fail a pull request whose changes the pre-commit formatting (lint-staged) would rewrite |
| `maven-package.yml` | Build JAR with embedded dependencies |
| `maven-deploy.yml` | Publish artifacts to GitHub Packages |
| `docker-build.yml` | Build and push Docker image via Jib |
| `helm-build.yml` | Package and push Helm chart |
| `publish.yml` | Upload JAR as GitHub artifact |
| `dependencygraph.yml` | Submit dependency snapshot to GitHub |
| `argocd-update.yml` | Update ArgoCD manifest with new image tag |
| `gitflow-release-finish.yml` | Complete GitFlow release (tag, merge, bump version) |
| `manual-release-start.yml` | Start GitFlow release branch (manual dispatch) |
| `copilot-review.yml` | Request a Copilot code review on a new pull request |
| `merge-eligibility.yml` | Keep the `ready-to-merge` label true to reviews, answered threads and green checks |
| `resolve-merged.yml` | On a merge into develop, move the Azure DevOps items the PR names to Resolved and create their Testing tasks |
| `mark-released.yml` | On a release, write `Released In`, the release comment and the on-environment comments on the items it shipped |

## Data Flow and Caching

The workflows use two distinct caching mechanisms to share data:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           WORKFLOW EXECUTION                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌──────────────┐     ┌──────────────────┐     ┌─────────────────────────┐  │
│  │  test-be.yml │     │ maven-package.yml│     │  Downstream Workflows   │  │
│  │              │     │                  │     │                         │  │
│  │  Downloads   │     │  Downloads deps  │     │  docker-build.yml       │  │
│  │  Maven deps  │     │  Builds JAR      │     │  maven-deploy.yml       │  │
│  │              │     │  Saves target/   │────▶│  publish.yml            │  │
│  └──────────────┘     └──────────────────┘     │  dependencygraph.yml    │  │
│         │                      │               │  helm-build.yml         │  │
│         │                      │               │                         │  │
│         ▼                      ▼               │  Restores target/       │  │
│  ┌─────────────────────────────────────┐       │  from cache             │  │
│  │      Maven Dependencies Cache       │       └─────────────────────────┘  │
│  │                                     │                                    │
│  │  Key: {os}-maven-{pom-hash}-{run_id}│                                    │
│  │  Path: ~/.m2/repository             │                                    │
│  └─────────────────────────────────────┘                                    │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 1. Maven Dependencies Cache

**Purpose:** Cache Maven dependencies (`~/.m2/repository`) across workflow runs.

**Cache Key Strategy:**
```yaml
key: ${{ runner.os }}-maven-${{ hashFiles('**/pom.xml') }}-${{ github.run_id }}
restore-keys: |
  ${{ runner.os }}-maven-${{ hashFiles('**/pom.xml') }}-
  ${{ runner.os }}-maven-
```

**How it works:**
- The primary key includes `run_id`, making it unique per workflow run (never hits cache)
- The `restore-keys` fallback restores the most recent cache with matching prefix
- Combined with SNAPSHOT purge step, this ensures fresh SNAPSHOT dependencies while caching stable ones

**Why this approach:**
The built-in `setup-java` cache uses only `pom.xml` hash as the key. This causes stale SNAPSHOT dependencies because the cache is never invalidated when remote SNAPSHOTs are updated. By including `run_id`, we force a cache miss on the primary key, falling back to restore stable dependencies, then re-downloading SNAPSHOTs.

**Workflows using this cache:**
- `test-be.yml`
- `maven-package.yml`
- `docker-build.yml`
- `maven-deploy.yml`
- `publish.yml`
- `dependencygraph.yml`
- `helm-build.yml`
- `gitflow-release-finish.yml`
- `manual-release-start.yml`

### 2. Target Directory Cache

**Purpose:** Share the built JAR and compiled classes between workflow jobs within the same run.

**Cache Key:**
```yaml
key: ${{ runner.os }}-jdk${{ inputs.jdk-version }}-target-${{ github.run_id }}
```

**How it works:**
- `maven-package.yml` builds the JAR and saves `target/` to cache
- Downstream workflows restore `target/` to avoid rebuilding
- Key includes `run_id` to ensure jobs in the same run share the exact same artifact

**Producer workflow:**
- `maven-package.yml` (uses `actions/cache/save`)

**Consumer workflows:**
- `docker-build.yml` - Uses pre-built JAR for Docker image
- `maven-deploy.yml` - Deploys pre-built JAR to GitHub Packages
- `publish.yml` - Uploads pre-built JAR as artifact
- `dependencygraph.yml` - Analyzes pre-built dependencies
- `helm-build.yml` - Packages Helm chart with application

### 3. NPM Dependencies Cache

**Purpose:** Cache Node.js dependencies for front-end workflows.

**Cache Key:**
```yaml
key: ${{ runner.os }}-node${{ inputs.node-version }}-${{ hashFiles('**/package-lock.json') }}
```

**Workflows using this cache:**
- `maven-package.yml` (when `require-npm: true`)
- `test-fe.yml`
- `check-format.yml`

## Typical Workflow Chain

### Development Branch Push

```
push to develop
      │
      ▼
┌─────────────┐    ┌──────────────────┐    ┌────────────────┐
│  test-be    │───▶│  maven-package   │───▶│  docker-build  │
│             │    │                  │    │                │
│  Run tests  │    │  Build JAR       │    │  Build image   │
│             │    │  Save target/    │    │  Push to Hub   │
└─────────────┘    └──────────────────┘    └────────────────┘
                            │                      │
                            ▼                      ▼
                   ┌─────────────────┐     ┌────────────────┐
                   │  publish        │     │ argocd-update  │
                   │                 │     │                │
                   │  Upload JAR     │     │  Deploy dev    │
                   └─────────────────┘     └────────────────┘
```

### Main Branch Push (Release)

```
push to main
      │
      ▼
┌──────────────────┐    ┌────────────────┐    ┌─────────────────────┐
│  maven-package   │───▶│  maven-deploy  │───▶│ gitflow-release-    │
│                  │    │                │    │ finish              │
│  Build JAR       │    │  Publish to    │    │                     │
│  (prod profile)  │    │  GitHub Pkgs   │    │  Tag, merge, bump   │
└──────────────────┘    └────────────────┘    └─────────────────────┘
```

## Common Inputs

Most reusable workflows accept these inputs:

| Input | Default | Description |
|-------|---------|-------------|
| `maven-profiles` | `dev` | Maven profiles to activate. Pass `''` for projects that define no profiles — the `-P` flag is then omitted entirely rather than naming a profile that does not exist |
| `jdk-version` | `17` | JDK version for builds |
| `node-version` | `22` | Node.js version (when applicable) |
| `require-npm` | `false` | Whether to set up Node.js |

## Secrets

| Secret | Used By | Purpose |
|--------|---------|---------|
| `MAVEN_PASSWORD` | Most workflows | GitHub Packages authentication |
| `DOCKER_PAT` | `docker-build.yml`, `helm-build.yml` | Docker Hub authentication |
| `ARGOCD_REPO_TOKEN` | `argocd-update.yml` | ArgoCD repository access |

`MAVEN_PASSWORD` is **optional** on `docker-build.yml` and `helm-build.yml`, but
callers should pass it. Both run Maven goals that resolve the project (and its
parent POM) from GitHub Packages; without credentials they succeed only while the
restored `~/.m2` cache already holds those artifacts, and fail with `401` on a
cache miss.

## Troubleshooting

### Stale SNAPSHOT Dependencies

SNAPSHOT freshness is only guaranteed in workflows that explicitly purge cached
SNAPSHOTs and run Maven with `-U`. Currently this applies to `test-be.yml` and
`maven-package.yml`:
1. The Maven cache key includes `run_id`, but the `restore-keys` fallback can
   still restore stale SNAPSHOTs from prior runs
2. SNAPSHOT directories are purged before each build (`Remove cached SNAPSHOT
   dependencies` step)
3. Maven runs with `-U` to force update checks

Other workflows (`docker-build.yml`, `helm-build.yml`, `maven-deploy.yml`,
`publish.yml`, `dependencygraph.yml`, `gitflow-release-finish.yml`,
`manual-release-start.yml`) cache `~/.m2/repository` for speed but may serve
stale SNAPSHOTs. Add the purge + `-U` pattern where SNAPSHOT freshness matters.

### Deploy fails with `409 Conflict`

`maven-deploy.yml` runs on every push to `main`, but `main`'s version only changes
when a release lands. A maintenance push to `main` therefore re-attempts a deploy
of a version that is already published, and GitHub Packages refuses to overwrite a
release version:

```
Failed to deploy artifacts: Could not transfer artifact ...:pom:1.3.3
from/to github-christhonie: status code: 409, reason phrase: Conflict (409)
```

The workflow now probes for the published `.pom` first and skips the deploy when
the version already exists, emitting a `Deploy skipped` notice in the run summary.
SNAPSHOTs bypass the probe because they are meant to be overwritten. If the probe
cannot reach the registry it deploys anyway rather than silently skipping a real
release, so a genuine `409` can still surface.

### Cache Not Restoring

Check that:
- The producing job completed successfully
- Cache keys match between save and restore
- The `run_id` is the same (for target directory cache)

### Target Directory Missing

Ensure `maven-package.yml` runs and completes before dependent workflows. The cache is keyed by `run_id`, so it only works within the same workflow run.
