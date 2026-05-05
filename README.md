# openvela-docker

Docker-first development and CI tooling for OpenVela.

This repository is intentionally separate from OpenVela source while the CI
workflow is being developed. The scripts are written so they can later be
upstreamed into OpenVela community repositories.

## Scope

The first deliverable is a Docker container that can:

1. Initialize OpenVela with `repo`.
2. Sync OpenVela source and prebuilts from `open-vela/manifests`.
3. Build one selected target with OpenVela `build.sh`.
4. Optionally run emulator output.
5. Run a small local smoke build set for development.

Full Apache NuttX-style CI is a later stage. It should preserve the
`tools/ci/testlist/*.dat`, `tools/testbuild.sh`, and `tools/ci/cibuild.sh`
scheme, but execute builds through OpenVela `build.sh`.

## Build The Image

```bash
docker build -t openvela-dev -f docker/openvela-dev/Dockerfile .
```

If your host UID/GID are not `1000:1000`, build the image with matching IDs so
mounted workspace files are owned by your host user:

```bash
docker build -t openvela-dev \
  --build-arg OPENVELA_UID="$(id -u)" \
  --build-arg OPENVELA_GID="$(id -g)" \
  -f docker/openvela-dev/Dockerfile .
```

## Start A Workspace

```bash
mkdir -p workspace
docker run --rm -it \
  -v "$PWD/workspace:/workspace" \
  -w /workspace \
  openvela-dev
```

The image runs as the non-root `openvela` user by default. This avoids
root-owned files in the mounted `workspace/` directory when the image UID/GID
matches your host user.

## Initialize OpenVela

Inside the container:

```bash
openvela-init
```

Equivalent raw commands:

```bash
repo init \
  -u https://github.com/open-vela/manifests.git \
  -b dev \
  -m openvela.xml \
  --group=default,platform-linux \
  --depth=1 \
  --git-lfs

repo sync -c -d --no-tags -j"$(nproc)"
```

## Build A Target

Default target:

```bash
openvela-build
```

Explicit target:

```bash
openvela-build vendor/openvela/boards/vela/configs/goldfish-arm64-v8a-ap/ --cmake
```

Makefile mode remains available for future NuttX-style testlist parity:

```bash
openvela-build sim:nsh --make --jobs 8
```

## Run Emulator

After a successful goldfish build:

```bash
openvela-emulator cmake_out/vela_goldfish-arm64-v8a-ap/
```

Some hosts may need extra Docker runtime flags for graphical output, KVM, or
device access. The build flow should be validated before emulator-specific host
integration is tuned.

## Quick Local CI

Run the small smoke set:

```bash
openvela-ci-quick --quick
```

Run the checked-in quick testlist:

```bash
openvela-ci-quick --testlist "$OPENVELA_CI_TESTLIST_DIR/quick.dat"
```

Add a one-off target:

```bash
openvela-ci-quick \
  --target "vendor/openvela/boards/vela/configs/goldfish-arm64-v8a-ap/ --cmake"
```

Useful development controls:

```bash
openvela-ci-quick --testlist "$OPENVELA_CI_TESTLIST_DIR/quick.dat" --print-list
openvela-ci-quick --testlist "$OPENVELA_CI_TESTLIST_DIR/quick.dat" --max-targets 1
openvela-ci-quick --testlist "$OPENVELA_CI_TESTLIST_DIR/quick.dat" --keep-going
```

Logs are written under `ci-artifacts/quick`, with a tab-separated summary at
`ci-artifacts/quick/summary.tsv`.

Supported testlist lines:

```text
CMake,vendor/openvela/boards/vela/configs/goldfish-arm64-v8a-ap/
Make,board:config
board:config
vendor/openvela/boards/vela/configs/goldfish-x86_64-ap/ --cmake
```

For `board:config` entries the runner searches the OpenVela workspace for
`*/boards/<board>/configs/<config>` and passes the resolved path to
`openvela-build`. This keeps the file format close to Apache NuttX while still
using OpenVela's `build.sh` workflow.

## Clean Local Data

The OpenVela workspace can be large. The cleanup helper is dry-run by default:

```bash
scripts/openvela-clean-docker --remove-workspace
```

Actually remove the default `workspace/` directory:

```bash
scripts/openvela-clean-docker --remove-workspace --force
```

If the workspace was created by a root container and cleanup reports permission
errors:

```bash
scripts/openvela-clean-docker --remove-workspace --sudo --force
```

Remove workspace, local `openvela-dev` image, and dangling Docker data:

```bash
scripts/openvela-clean-docker --all --force
```

If you are using an older image that still runs as root, start the container
with your host UID/GID:

```bash
docker run --rm -it \
  --user "$(id -u):$(id -g)" \
  -v "$PWD/workspace:/workspace" \
  -w /workspace \
  openvela-dev
```

## Smoke Checks

Inside the container:

```bash
repo --version
git lfs --version
cmake --version
ninja --version
python3 -m pytest --version
```

## Design Notes

The Docker image installs host dependencies only. It does not bake OpenVela
source, compiler toolchains, QEMU prebuilts, CMake prebuilts, build-tools, or
emulator binaries into the image. Those are manifest-controlled and synced by
`repo`, so they match the branch under test.
