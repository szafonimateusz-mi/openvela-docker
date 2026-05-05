# OpenVela CI Docker-First Implementation Plan

## 1. Goal

Create a standalone `openvela-ci` project that starts with a Docker development
container and grows into CI tooling for OpenVela.

The first deliverable is only the Docker container and local helper workflow.
The container must let a developer initialize a complete OpenVela workspace,
sync OpenVela prebuilts through the official manifest, build a selected config,
and optionally run emulator or test commands.

Later stages will add simplified local CI and then Apache NuttX-style CI
compatibility. The scripts should be written so they can eventually be
upstreamed into OpenVela community repositories, most likely:

- Docker and GitHub Actions helpers into `open-vela/public-actions`
- build/testlist integration into `open-vela/nuttx`
- OpenVela-specific target lists into `open-vela/nuttx/tools/ci/testlist`

## 2. Background

OpenVela development is documented for Ubuntu 22.04. The official quick start
explicitly initializes source through the `repo` tool and the
`open-vela/manifests` repository.

OpenVela differs from upstream Apache NuttX in an important way: OpenVela uses a
larger manifest-managed source tree and ships many prebuilts as manifest
projects. The Docker image should not duplicate those prebuilts by downloading
toolchains independently.

OpenVela manifest-provided prebuilts include:

- GCC bare-metal toolchains for Linux x86_64 and other hosts
- QEMU prebuilts
- CMake prebuilts
- build tools such as Ninja and kconfig helpers
- Python helper packages under `prebuilts/tools`
- emulator binaries and emulator helper scripts
- WASM/Clang prebuilts

Apache NuttX CI remains the reference model for the future full CI design:

- `tools/ci/testlist/*.dat` defines build target sets
- `tools/testbuild.sh` interprets the testlist files
- `tools/ci/cibuild.sh` is the CI entrypoint
- normal testlist entries build with Makefile flow
- `CMake,<target>` entries select CMake flow when `-N` is passed
- CI uploads build artifacts and can run `run.sh` or NTFC/citest jobs

OpenVela CI must preserve the same general testlist-driven model later, but the
execution has to call OpenVela's `build.sh` flow:

```bash
./build.sh <target> -e <flags> -j<n>
./build.sh <target> --cmake -e <flags> -j<n>
```

## 3. Upstream References

- OpenVela Ubuntu quick start:
  <https://github.com/open-vela/docs/blob/dev/en/quickstart/openvela_ubuntu_quick_start.md>
- OpenVela CMake quick start:
  <https://github.com/open-vela/docs/blob/dev/en/device_dev_guide/build/CMake_quick_start.md>
- OpenVela manifest:
  <https://github.com/open-vela/manifests/blob/dev/openvela.xml>
- OpenVela public CI:
  <https://github.com/open-vela/public-actions/blob/dev/.github/workflows/ci-real.yml>
- Apache NuttX CI testlists:
  <https://github.com/apache/nuttx/tree/master/tools/ci/testlist>
- Apache NuttX `testbuild.sh`:
  <https://github.com/apache/nuttx/blob/master/tools/testbuild.sh>
- Apache NuttX `cibuild.sh`:
  <https://github.com/apache/nuttx/blob/master/tools/ci/cibuild.sh>
- Apache NuttX build workflow:
  <https://github.com/apache/nuttx/blob/master/.github/workflows/build.yml>

## 4. Stage 1: Create The Docker Development Container

Stage 1 creates a separate `openvela-ci` repo layout and implements only the
container plus local helper commands. It does not implement full CI.

Recommended repository layout:

```text
openvela-ci/
├── README.md
├── docker/
│   └── openvela-dev/
│       └── Dockerfile
└── scripts/
    ├── openvela-init
    ├── openvela-build
    ├── openvela-emulator
    ├── openvela-ci-quick
    └── openvela-clean-docker
```

### 4.1 Docker Base

Use Ubuntu 22.04:

```dockerfile
FROM ubuntu:22.04
```

Set noninteractive defaults:

```dockerfile
ENV DEBIAN_FRONTEND=noninteractive
ENV PIP_DISABLE_PIP_VERSION_CHECK=true
ENV PIP_NO_CACHE_DIR=1
```

### 4.2 Host Packages

Install host dependencies only. Do not install OpenVela compiler prebuilts in
Docker.

Required base tools:

```text
ca-certificates
curl
git
git-lfs
gnupg
jq
less
locales
sudo
unzip
wget
xz-utils
zip
```

OpenVela build host packages from the docs and `nuttx/tools/build.sh`:

```text
autoconf
automake
bison
build-essential
cmake
dfu-util
flex
genromfs
gettext
gperf
make
mtools
nasm
net-tools
ninja-build
nodejs
npm
pkgconf
protobuf-c-compiler
protobuf-compiler
python-is-python3
python3
python3-pip
xxd
yasm
```

Simulator and emulator libraries:

```text
libasound2-dev
libasound2-plugins
libc++-dev
libc++abi-dev
libc6-dev
libdivsufsort-dev
libncurses5
libprotobuf-dev
libpulse-dev
libusb-1.0-0-dev
libuv1-dev
libv4l-dev
libx11-dev
libxext-dev
zlib1g-dev
```

QEMU packages useful for local emulator and future NTFC work:

```text
qemu-efi-aarch64
qemu-system-arm
qemu-system-misc
qemu-utils
```

Install Git LFS system-wide:

```dockerfile
RUN git lfs install --system
```

Install `gnupg` too. The `repo` launcher can proceed without it, but fresh
containers warn when release signing keys are updated.

### 4.3 Python Packages

Install Python packages needed by OpenVela/NuttX tooling and future NTFC tests:

```text
coloredlogs
construct
cxxfilt
jinja2
kconfiglib
lark
pexpect==4.8.0
pyelftools
pyserial==3.5
pytest==6.2.5
pytest-json==0.4.0
pytest-ordering==0.6
pytest-repeat==0.9.1
stringcase
```

Use a single `pip3 install` line to keep the Dockerfile simple and cacheable.

### 4.4 Repo Tool

Install Google's `repo` tool in the image:

```dockerfile
RUN curl -fsSL https://storage.googleapis.com/git-repo-downloads/repo \
      -o /usr/local/bin/repo \
    && chmod +x /usr/local/bin/repo
```

### 4.5 What Must Not Be In The Image

Do not bake these into Docker:

- OpenVela source tree
- OpenVela `prebuilts/gcc/*`
- OpenVela `prebuilts/qemu/*`
- OpenVela `prebuilts/cmake/*`
- OpenVela `prebuilts/build-tools/*`
- OpenVela emulator prebuilts
- Apache-downloaded GCC or Clang toolchains

Those components must be synced from the manifest so they match the OpenVela
branch under test.

## 5. Stage 2: Local Developer Workflow

Stage 2 provides simple helper scripts. These scripts are convenience wrappers;
developers must still be able to run raw OpenVela commands manually.

### 5.1 Build The Docker Image

From the future `openvela-ci` repo:

```bash
docker build -t openvela-dev -f docker/openvela-dev/Dockerfile .
```

### 5.2 Start A Persistent Workspace

```bash
mkdir -p workspace
docker run --rm -it \
  -v "$PWD/workspace:/workspace" \
  -w /workspace \
  openvela-dev
```

### 5.3 `openvela-init`

Purpose: initialize and sync a full OpenVela workspace.

Defaults:

- manifest repository: `https://github.com/open-vela/manifests.git`
- branch: `dev`
- manifest file: `openvela.xml`
- groups: `default,platform-linux`
- jobs: `$(nproc)`

Default command:

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

The script must run without user interaction. Fresh containers usually have no
Git identity, and `repo init` asks Git for `GIT_COMMITTER_IDENT` while preparing
local manifest metadata. Set anonymous local-only defaults if missing:

```bash
git config --global user.name "OpenVela CI"
git config --global user.email "openvela-ci@example.invalid"
```

This is not GitHub authentication and does not publish anything. Also disable
the interactive repo color prompt:

```bash
git config --global color.ui false
```

Required options:

```text
--branch <name>       default: dev
--manifest <file>     default: openvela.xml
--groups <groups>     default: default,platform-linux
--jobs <n>            default: nproc
--no-depth            optional full-history sync
```

### 5.4 `openvela-build`

Purpose: build one selected OpenVela config.

Default target:

```text
vendor/openvela/boards/vela/configs/goldfish-arm64-v8a-ap/
```

Default command:

```bash
./build.sh vendor/openvela/boards/vela/configs/goldfish-arm64-v8a-ap/ \
  --cmake \
  -e -Werror \
  -j"$(nproc)"
```

Required behavior:

- accepts target path or NuttX-style target as first positional argument
- supports `--cmake` and `--make`
- defaults to CMake
- accepts `--jobs <n>`
- accepts extra flags after `--extra-flags`

Examples:

```bash
openvela-build
openvela-build vendor/openvela/boards/vela/configs/goldfish-x86_64-ap/ --cmake
openvela-build vendor/openvela/boards/sil/configs/ksil_arm32r_nsh_core0 --cmake --jobs 8
openvela-build sim:nsh --make --jobs 8
```

### 5.5 `openvela-emulator`

Purpose: run a built CMake output through OpenVela's emulator wrapper.

Default command shape:

```bash
./emulator.sh cmake_out/vela_goldfish-arm64-v8a-ap/
```

Behavior:

- verifies `./emulator.sh` exists
- verifies the selected output directory exists
- passes remaining arguments through to `emulator.sh`
- documents that graphical or KVM behavior may require extra Docker runtime
  flags on some hosts

Example:

```bash
openvela-emulator cmake_out/vela_goldfish-arm64-v8a-ap/
```

### 5.6 `openvela-clean-docker`

Purpose: remove large local Docker/OpenVela development data from the host.

The cleanup helper must be dry-run by default:

```bash
scripts/openvela-clean-docker --remove-workspace
```

Actually remove the default mounted workspace:

```bash
scripts/openvela-clean-docker --remove-workspace --force
```

If files were created by a root container and cleanup reports permission
errors:

```bash
scripts/openvela-clean-docker --remove-workspace --sudo --force
```

Remove workspace, local image, and dangling Docker data:

```bash
scripts/openvela-clean-docker --all --sudo --force
```

The Docker image now runs as a non-root `openvela` user by default. Build the
image with the host UID/GID so files created in a mounted workspace can be
removed by the host user without sudo:

```bash
docker build -t openvela-dev \
  --build-arg OPENVELA_UID="$(id -u)" \
  --build-arg OPENVELA_GID="$(id -g)" \
  -f docker/openvela-dev/Dockerfile .
```

## 6. Stage 3: Simplified Local CI

Stage 3 adds `scripts/openvela-ci-quick`.

The goal is fast local validation. Apache NuttX CI runs a massive number of
builds and can take hours. That is not appropriate for every local Docker or
script change.

Add a quick mode controlled by one or both of:

```bash
openvela-ci-quick --quick
OPENVELA_CI_QUICK=1 openvela-ci-quick
```

Initial quick set:

```text
vendor/openvela/boards/vela/configs/goldfish-arm64-v8a-ap/ --cmake
vendor/openvela/boards/vela/configs/goldfish-x86_64-ap/ --cmake
```

Later quick set:

```text
one goldfish CMake target
one OpenVela vendor/SIL target
one OpenVela-specific citest/NTFC target
```

`openvela-ci-quick` should:

- run from the OpenVela workspace root
- build only the configured smoke targets
- clean `cmake_out` between targets unless `--keep-output` is passed
- write logs under `ci-artifacts/quick/<target>/`
- return nonzero on the first failed target by default
- provide `--keep-going` for development

## 7. Stage 4: Apache NuttX-Style CI Compatibility

This stage comes after the Docker container works.

OpenVela should support the same CI scheme as Apache NuttX:

- list files under `tools/ci/testlist/*.dat`
- `tools/testbuild.sh`
- `tools/ci/cibuild.sh`
- matrix jobs that pass selected testlist files

### 7.1 Preserve Testlist Grammar

Normal entries build with Makefile mode:

```text
sim:nsh
arm_board:config
/sim/*/*/*/[a-b]*
```

Blacklist entries skip matching host/target combinations:

```text
-Darwin,sim:alsa
```

CMake entries select CMake builds when `-N` is passed:

```text
CMake,sim:alsa
CMake,vendor/openvela/boards/vela/configs/goldfish-arm64-v8a-ap
```

Optional toolchain override syntax should remain available where it is still
meaningful:

```text
board:config,CONFIG_ARM_TOOLCHAIN_GNU_EABI
```

### 7.2 Adapt `testbuild.sh`

The OpenVela version should preserve Apache behavior where possible:

- `-p` prints resolved target list without building
- `-N` enables CMake entries
- `-A` stores artifacts
- `-R` runs target `run.sh` where present
- `-S` enables third-party package store behavior if still relevant
- `--codechecker` remains optional

Execution changes:

```bash
# Makefile mode
./build.sh <target> -e "<extra flags>" -j<n>

# CMake mode
./build.sh <target> --cmake -e "<extra flags>" -j<n>
```

OpenVela vendor config paths must be accepted directly:

```text
vendor/openvela/boards/vela/configs/goldfish-arm64-v8a-ap
vendor/openvela/boards/sil/configs/ksil_arm32r_nsh_core0
vendor/flagchip/boards/fc7300/fc7300f8m-evb/configs/core0
```

### 7.3 Adapt `cibuild.sh`

The OpenVela `cibuild.sh` should remain the outer CI entrypoint:

```bash
tools/ci/cibuild.sh -c -A -N -R tools/ci/testlist/openvela-quick.dat
```

It should:

- detect CPU count
- set ccache if requested
- pass options through to `testbuild.sh`
- default to `-e "-Wno-cpp -Werror"` or the OpenVela-agreed equivalent
- accept multiple testlist files

### 7.4 Add OpenVela Testlists

Initial new testlists:

```text
openvela-goldfish.dat
openvela-sil.dat
openvela-citest.dat
openvela-quick.dat
```

`openvela-quick.dat` should stay intentionally small and be suitable for local
development.

Full CI lists can grow later and may mirror Apache's arch grouping:

```text
arm-01.dat
risc-v-01.dat
sim-01.dat
x86_64-01.dat
openvela-01.dat
openvela-02.dat
```

## 8. Stage 5: NTFC And OpenVela Citests

The future full CI should build targets and then run NTFC/citest jobs for
selected configs.

The model should follow Apache NuttX:

1. build target
2. collect artifacts
3. run QEMU/emulator test where supported
4. collect logs and pytest JSON
5. upload artifacts

OpenVela-specific behavior:

- OpenVela-only citest configs are added as normal testlist entries.
- Test execution can use OpenVela's `tests/scripts/script` pytest layout.
- The runtime test command should be generated from the built target metadata
  instead of hardcoding all targets in workflow YAML.

Example command shape, based on current OpenVela public CI:

```bash
cd tests/scripts/script
pytest -m "common or <target_model>" ./ \
  -B <standard_config> \
  -L /workspace \
  -P /workspace \
  -F /tmp \
  -R qemu \
  -v \
  --disable-warnings \
  --count=1 \
  --json=/workspace/autotest.json \
  --maxfail=10
```

## 9. Stage 6: GitHub Actions

Only after local Docker and quick CI work should GitHub Actions be added.

### 9.1 Docker Publish Workflow

Add `.github/workflows/docker.yml` in `openvela-ci`:

- build `docker/openvela-dev`
- publish to `ghcr.io/<owner>/openvela-dev`
- run on push to main and on manual dispatch
- optionally run on PR without pushing

### 9.2 Quick CI Workflow

Add `.github/workflows/quick-ci.yml`:

- checkout `openvela-ci`
- run Docker image
- initialize OpenVela with `repo`
- run `openvela-ci-quick --quick`
- upload `ci-artifacts`

### 9.3 Full CI Workflow

Future full CI should:

- initialize OpenVela source through manifest
- preserve manifest snapshot artifacts
- support dependent PRs across OpenVela repos
- run matrix jobs by testlist file
- use OpenVela-adapted `cibuild.sh`
- upload build and test artifacts

## 10. Validation Plan

### 10.1 Docker Smoke Test

Inside the container:

```bash
repo --version
git lfs --version
cmake --version
ninja --version
python3 -m pytest --version
```

### 10.2 Full Initialization Test

In a clean mounted workspace:

```bash
openvela-init
```

Expected result:

- `.repo/` exists
- `nuttx/`, `apps/`, `vendor/`, and `prebuilts/` exist
- Linux platform prebuilts are synced
- Git LFS files are materialized, not pointer-only files

### 10.3 Build Test

```bash
openvela-build vendor/openvela/boards/vela/configs/goldfish-arm64-v8a-ap/ --cmake
```

Expected result:

- build exits zero
- `cmake_out/vela_goldfish-arm64-v8a-ap/` exists
- `nuttx` artifact exists under the output directory
- `.config` exists under the output directory

### 10.4 Emulator Test

```bash
openvela-emulator cmake_out/vela_goldfish-arm64-v8a-ap/
```

Expected result:

- emulator starts
- OpenVela shell prompt appears

### 10.5 Quick CI Test

```bash
openvela-ci-quick --quick
```

Expected result:

- only the smoke targets run
- logs are written under `ci-artifacts/quick`
- command exits nonzero if any target fails

### 10.6 Future Testlist Parser Test

After Stage 4:

```bash
tools/testbuild.sh -p -N tools/ci/testlist/openvela-quick.dat
```

Expected result:

- wildcard entries expand correctly
- blacklist entries are respected
- `CMake,<target>` entries are selected only with `-N`
- OpenVela vendor config paths are accepted

## 11. Risks And Decisions

### Docker vs Official Docs

OpenVela docs state that Docker is not supported for normal development. This
project intentionally creates a Docker environment for CI and development
reproducibility. Any incompatibilities found during build or emulator testing
must be documented in `README.md`.

### Prebuilts Source Of Truth

OpenVela manifest prebuilts are authoritative. The Docker image must not drift
by installing separate compiler or QEMU versions unless a missing host package
is proven necessary.

### Local Runtime Tests

Emulator and QEMU tests may require host-specific Docker flags, KVM access, or
display/headless configuration. Stage 1 should make build reproducible first;
runtime tests can be improved after build success.

### Full CI Runtime

Apache-scale CI can take hours. Local development must use `openvela-ci-quick`
or a similar flag-controlled smoke mode. Full matrix CI should remain available
for GitHub Actions and scheduled validation.

## 12. Initial Implementation Checklist

1. Create `openvela-ci` repo skeleton.
2. Add `docker/openvela-dev/Dockerfile`.
3. Add `scripts/openvela-init`.
4. Add `scripts/openvela-build`.
5. Add `scripts/openvela-emulator`.
6. Add `scripts/openvela-clean-docker`.
7. Add placeholder `scripts/openvela-ci-quick`.
8. Add `README.md` with Docker build, init, build, emulator, and cleanup examples.
9. Build Docker image locally.
10. Run Docker smoke checks.
11. Run full OpenVela `repo` initialization.
12. Build the default goldfish CMake target.
13. Record any missing packages or Docker runtime constraints.
14. Only then start Stage 3 quick CI scripting.

## 13. Current Status And Next Step

Current state:

- Docker image builds on Ubuntu 22.04 and installs only host-side dependencies.
- The image runs as non-root `openvela` by default, with configurable UID/GID.
- `openvela-init` performs noninteractive `repo init` by setting default Git
  identity and disabling the repo color prompt.
- `openvela-build` builds the default goldfish CMake target through OpenVela
  `build.sh`.
- `openvela-emulator` exists, but graphical/runtime execution still needs
  host-specific Docker flags and is not required for the first Docker milestone.
- `openvela-clean-docker` can dry-run cleanup, remove the large mounted
  workspace, remove the local image, and use `--sudo` for older root-owned
  workspaces.
- `openvela-ci-quick` supports default smoke targets, custom `--target`,
  `--testlist`, `--max-targets`, `--print-list`, `--keep-going`, and per-target
  logs with a summary file.
- `testlists/quick.dat` defines the first small CMake smoke matrix.

Next step:

Run `openvela-ci-quick --testlist testlists/quick.dat --max-targets 1` inside a
fresh initialized OpenVela workspace and record the first real build/runtime
gap. After that, add an OpenVela-specific NTFC wrapper for emulator-capable
targets and expand the testlist mapping for OpenVela-only board configs.
