# cicd

Reusable GitHub Actions workflows for every Bulmer Kiln repository: build and test, static and
dependency analysis, and releases with signing and SLSA provenance. This repository is public, so
anyone can see how a release was built and check its provenance.

It stops at publishing artifacts. Deployment lives in a private repository and pulls released
versions from there; nothing here holds deployment credentials.

## Workflows

| Workflow | What it does | Caller permissions |
|---|---|---|
| [`gradle-build.yml`](.github/workflows/gradle-build.yml) | `./gradlew build` with every gate it runs; optionally PIT mutation testing as a second job. Reports are uploaded when something fails | `contents: read` |
| [`codeql.yml`](.github/workflows/codeql.yml) | CodeQL with the security-extended queries, one job per language. `java-kotlin` is compiled with Gradle; other languages (`javascript-typescript`, `actions`, ...) are analyzed from source | `actions: read`, `contents: read`, `security-events: write` |
| [`semgrep.yml`](.github/workflows/semgrep.yml) | Semgrep's community rules; any finding fails. Results go to code scanning | `contents: read`, `security-events: write` |
| [`dependencies.yml`](.github/workflows/dependencies.yml) | Submits the Gradle dependency graph, which drives Dependabot's CVE alerts, and fails a pull request that adds a dependency with a high or critical advisory | `contents: write`, `pull-requests: write` |
| [`gradle-release.yml`](.github/workflows/gradle-release.yml) | On a published GitHub release: checks the tag, builds from scratch in the `release` environment, signs and uploads the bundle to Maven Central, attaches SLSA provenance and the bundle to the release, and rebuilds to check the artifacts are reproducible | `actions: read`, `contents: write`, `id-token: write` |

Each workflow's inputs are documented at the top of the file.

## Usage

```yaml
jobs:
  build:
    permissions:
      contents: read
    uses: BulmerKiln/cicd/.github/workflows/gradle-build.yml@v1
```

A web repository passes its own languages to CodeQL:

```yaml
  codeql:
    permissions:
      actions: read
      contents: read
      security-events: write
    uses: BulmerKiln/cicd/.github/workflows/codeql.yml@v1
    with:
      languages: '["javascript-typescript", "actions"]'
```

## Releasing a Gradle repository

The calling repository needs:

1. A `release` environment with a required reviewer, limited to tags, holding these secrets:
   - `KILN_SIGNING_KEY` and `KILN_SIGNING_PASSWORD`: the project's armored PGP key and its passphrase
   - `MAVEN_CENTRAL_USERNAME` and `MAVEN_CENTRAL_PASSWORD`: a Central Portal user token for the
     `com.bulmerkiln` namespace
2. `version=` in `gradle.properties`. The release replaces it with the tag, so the tag is the only
   place a version is decided.
3. A task that stages every publication in Maven repository layout (`stageRelease`, writing to
   `build/staging-deploy`, by default).

Then publish a GitHub release whose tag is the version, such as `1.4.0` or `1.4.0-rc.1`.

## Conventions

- Actions are referenced by major tag (`@v7`). Semgrep's rule asking for commit SHAs is turned off
  to match. Actions and images without a major tag use an exact version.
- Every job declares its own permissions, and checkouts don't keep credentials.
- Releases build with Gradle's cache off, so a poisoned cache can't reach a release.

## Versioning

Callers use the major tag. Each release gets an exact tag (`v1.2.0`), and `v1` moves to the latest
`v1.x.y`. A breaking change to a workflow's inputs or behaviour starts `v2`.
