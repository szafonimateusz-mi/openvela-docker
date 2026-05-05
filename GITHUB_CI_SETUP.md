# GitHub CI Setup For OpenVela Repositories

This guide explains the GitHub-side steps needed to publish the OpenVela CI
Docker image and run initial CI builds in OpenVela repository forks such as
`nuttx` and `nuttx-apps`.

The first CI version is intentionally build-only. It initializes a full
OpenVela workspace from the manifest, replaces only the repository that
triggered CI, then runs a small `openvela-ci-quick` build matrix.
Nested manifest projects inside that repository path are preserved during the
overlay, for example `nuttx/fs/fatfs/fatfs` under `nuttx`.

## Branch Policy

OpenVela uses one manifest branch to select a compatible set of repositories.
The repository being tested is the only checkout that CI replaces.

Use the OpenVela branch policy when choosing the manifest branch:

- `dev`: active development branch. Use this for normal contribution CI.
- `trunk`: stable branch. Use this only when the tested branch is based on
  OpenVela `trunk`.
- release tags: stable production snapshots. Use only for release validation.

Do not mix a tested repository branch based on `trunk` with a `dev` manifest,
or the reverse. The result is a source tree that does not match any OpenVela
repo workspace and can fail in build scripts or CMake before compilation.

## 1. Publish `openvela-docker` To GitHub

Create a GitHub repository for this CI project, for example:

```text
<owner>/openvela-docker
```

Push this repository content to GitHub and enable GitHub Actions for the repo.

## 2. Publish The Docker Image To GHCR

The Docker workflow publishes:

```text
ghcr.io/<owner>/openvela-dev:latest
ghcr.io/<owner>/openvela-dev:<commit-sha>
```

Run the workflow manually:

1. Open `https://github.com/<owner>/openvela-docker/actions`.
2. Select the `Docker` workflow.
3. Click `Run workflow`.
4. Wait for it to pass.

After the workflow finishes, verify the package exists:

```text
https://github.com/<owner>?tab=packages
```

Package name:

```text
openvela-dev
```

If your OpenVela forks cannot pull private GHCR packages, make the package
public:

1. Open the `openvela-dev` package page.
2. Open package settings.
3. Change visibility to public.

## 3. Smoke-Test The Published Image

From any machine with Docker:

```bash
docker run --rm ghcr.io/<owner>/openvela-dev:latest \
  /bin/bash -lc 'openvela-ci-quick --testlist "$OPENVELA_CI_TESTLIST_DIR/quick.dat" --print-list'
```

Expected output:

```text
vendor/openvela/boards/vela/configs/goldfish-arm64-v8a-ap/ --cmake
vendor/openvela/boards/vela/configs/goldfish-x86_64-ap/ --cmake
```

## 4. Add CI To Your `nuttx` Fork

Create this file in your OpenVela `nuttx` fork:

```text
.github/workflows/openvela-ci.yml
```

Use this content, replacing `<owner>` with the GitHub owner that hosts
`openvela-docker`:

```yaml
name: OpenVela CI

on:
  pull_request:
  push:
    branches: [dev, main]
  workflow_dispatch:

permissions:
  contents: read
  packages: read

jobs:
  openvela:
    uses: <owner>/openvela-docker/.github/workflows/openvela-repo-ci.yml@main
    with:
      image: ghcr.io/<owner>/openvela-dev:latest
      manifest-branch: dev
      repo-path: nuttx
      max-targets: "1"
      jobs: "4"
```

Open a small test PR in the `nuttx` fork. The workflow should:

1. Pull `ghcr.io/<owner>/openvela-dev:latest`.
2. Initialize OpenVela from `open-vela/manifests` on `manifest-branch`.
3. Replace `openvela/nuttx` with the PR checkout.
4. Build one target from `/opt/openvela-ci/testlists/quick.dat`.
5. Upload `openvela-ci-artifacts`.

## 5. Add CI To Your `nuttx-apps` Fork

Create the same file in your OpenVela `nuttx-apps` fork:

```text
.github/workflows/openvela-ci.yml
```

Use this content:

```yaml
name: OpenVela CI

on:
  pull_request:
  push:
    branches: [dev, main]
  workflow_dispatch:

permissions:
  contents: read
  packages: read

jobs:
  openvela:
    uses: <owner>/openvela-docker/.github/workflows/openvela-repo-ci.yml@main
    with:
      image: ghcr.io/<owner>/openvela-dev:latest
      manifest-branch: dev
      repo-path: apps
      max-targets: "1"
      jobs: "4"
```

Open a small test PR in the `nuttx-apps` fork. The workflow should replace the
manifest checkout at `apps` with the PR checkout.

## 6. Read CI Results

Open the workflow run and download the `openvela-ci-artifacts` artifact.

Important files:

```text
ci-artifacts/quick/summary.tsv
ci-artifacts/quick/<target>/build.log
```

`summary.tsv` contains one row per target:

```text
status  target  mode  log
```

If a build fails, inspect the matching `build.log`.

## 7. Increase Coverage After The First Passes

Keep the first CI run small:

```yaml
max-targets: "1"
jobs: "4"
```

After both `nuttx` and `nuttx-apps` pass, increase to:

```yaml
max-targets: "2"
```

Only add emulator or NTFC runtime tests after build-only CI is reliable.

## 8. Enable NTFC Runtime Tests

The reusable workflow has opt-in NTFC support based on Apache NuttX CI:

- installs and uses `ntfc==0.0.1` from the Docker image
- clones `apache/nuttx-ntfc-testing` into `$NTFCDIR/external/nuttx-testing`
- runs target config `run.sh` after a successful build
- uploads NTFC output with the normal `openvela-ci-artifacts`

Example caller configuration:

```yaml
jobs:
  openvela:
    uses: <owner>/openvela-docker/.github/workflows/openvela-repo-ci.yml@main
    with:
      image: ghcr.io/<owner>/openvela-dev:latest
      manifest-branch: dev
      repo-path: nuttx
      testlist: /opt/openvela-ci/testlists/ntfc-smoke.dat
      max-targets: "1"
      jobs: "4"
      run-runtime-tests: true
```

After the OpenVela target configs provide executable `run.sh` files, make the
runtime test mandatory:

```yaml
      require-runtime-tests: true
```

Each runtime-enabled target should provide:

```text
<config>/run.sh
<config>/config.yaml
<config>/session.json
```

`run.sh` receives the same environment variables as Apache NuttX `testbuild.sh`
runtime hooks:

```text
CURRENTCONFDIR
ARTIFACTCONFDIR
NTFCDIR
```

The target script should call `ntfc test` and move logs/results into
`$ARTIFACTCONFDIR/ntfc`.

## Notes

- The reusable workflow also supports auto-resolving the repo path from
  `openvela.xml`, but the initial fork workflows use explicit `repo-path` to
  avoid ambiguity.
- The Docker image contains only host tools. OpenVela source, prebuilts,
  compilers, build tools, and emulator files still come from the manifest.
- The reusable workflow defaults to `manifest-branch: dev`. If a tested branch
  is intentionally based on `trunk`, set `manifest-branch: trunk` in the caller
  workflow and keep all other repositories on that same branch through the
  manifest.
- If `repo sync` hits memory pressure in GitHub Actions, lower `jobs` to `"2"`.
