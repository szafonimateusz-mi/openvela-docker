# SPDX-License-Identifier: Apache-2.0
#
# OpenVela development and CI base image.
#
# This image intentionally installs host dependencies only. OpenVela source,
# compiler toolchains, QEMU prebuilts, CMake prebuilts, build-tools, and
# emulator prebuilts are synced by repo from open-vela/manifests.

FROM ubuntu:22.04

ARG OPENVELA_UID=1000
ARG OPENVELA_GID=1000

ENV DEBIAN_FRONTEND=noninteractive
ENV PIP_DISABLE_PIP_VERSION_CHECK=true
ENV PIP_NO_CACHE_DIR=1
ENV LANG=C.UTF-8
ENV LC_ALL=C.UTF-8
ENV HOME=/home/openvela

SHELL ["/bin/bash", "-o", "pipefail", "-c"]

RUN apt-get update -qq \
  && apt-get install -y -qq --no-install-recommends \
    autoconf \
    automake \
    bison \
    build-essential \
    ca-certificates \
    cmake \
    curl \
    dfu-util \
    flex \
    genromfs \
    gettext \
    git \
    git-lfs \
    gnupg \
    gperf \
    jq \
    less \
    libasound2-dev \
    libasound2-plugins \
    libc++-dev \
    libc++abi-dev \
    libc6-dev \
    libdivsufsort-dev \
    libncurses5 \
    libprotobuf-dev \
    libpulse-dev \
    libusb-1.0-0-dev \
    libuv1-dev \
    libv4l-dev \
    libx11-dev \
    libxext-dev \
    locales \
    make \
    mtools \
    nasm \
    net-tools \
    ninja-build \
    nodejs \
    npm \
    pkgconf \
    protobuf-c-compiler \
    protobuf-compiler \
    python-is-python3 \
    python3 \
    python3-pip \
    qemu-efi-aarch64 \
    qemu-system-arm \
    qemu-system-misc \
    qemu-utils \
    rsync \
    sudo \
    unzip \
    wget \
    xxd \
    xz-utils \
    yasm \
    zip \
  && locale-gen C.UTF-8 \
  && git lfs install --system \
  && rm -rf /var/lib/apt/lists/*

RUN python3 -m pip install \
    coloredlogs \
    construct \
    cxxfilt \
    jinja2 \
    kconfiglib \
    lark \
    pexpect==4.8.0 \
    pyelftools \
    pyserial==3.5 \
    pytest==6.2.5 \
    pytest-json==0.4.0 \
    pytest-ordering==0.6 \
    pytest-repeat==0.9.1 \
    stringcase

RUN curl -fsSL https://storage.googleapis.com/git-repo-downloads/repo \
      -o /usr/local/bin/repo \
  && chmod +x /usr/local/bin/repo

COPY scripts/openvela-* /usr/local/bin/
COPY testlists/ /opt/openvela-ci/testlists/

ENV OPENVELA_CI_TESTLIST_DIR=/opt/openvela-ci/testlists

RUN groupadd --gid "${OPENVELA_GID}" openvela \
  && useradd --uid "${OPENVELA_UID}" --gid "${OPENVELA_GID}" \
    --create-home --shell /bin/bash openvela \
  && mkdir -p /workspace \
  && chown openvela:openvela /workspace \
  && echo "openvela ALL=(ALL) NOPASSWD:ALL" >/etc/sudoers.d/openvela \
  && chmod 0440 /etc/sudoers.d/openvela

WORKDIR /workspace

USER openvela

CMD ["/bin/bash"]
