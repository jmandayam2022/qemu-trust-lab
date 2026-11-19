# OP-TEE build environment (Ubuntu 22.04, linux/arm64)
# Package list from https://optee.readthedocs.io/en/latest/building/prerequisites.html
# Differences from the docs Dockerfile (intentional):
#   - No `repo init/sync` or toolchain build in the image; source lives on a Docker volume.
#   - Added tmux, vim, less, sudo, ca-certificates for interactive use (Day 9 console panes).
FROM ubuntu:22.04
ARG DEBIAN_FRONTEND=noninteractive
ENV FORCE_UNSAFE_CONFIGURE=1
RUN apt-get update && apt-get upgrade -y
RUN apt-get install -y \
    adb \
    acpica-tools \
    autoconf \
    automake \
    bc \
    bison \
    build-essential \
    ccache \
    cpio \
    cscope \
    curl \
    device-tree-compiler \
    e2tools \
    expect \
    fastboot \
    flex \
    ftp-upload \
    gdisk \
    git \
    libgnutls28-dev \
    libattr1-dev \
    libcap-ng-dev \
    libfdt-dev \
    libftdi-dev \
    libglib2.0-dev \
    libgmp3-dev \
    libhidapi-dev \
    libmpc-dev \
    libncurses5-dev \
    libpixman-1-dev \
    libslirp-dev \
    libssl-dev \
    libtool \
    libusb-1.0-0-dev \
    make \
    mtools \
    netcat \
    ninja-build \
    python3-cryptography \
    python3-pip \
    python3-pyelftools \
    python3-serial \
    python3-tomli \
    python-is-python3 \
    rsync \
    swig \
    unzip \
    uuid-dev \
    wget \
    xdg-utils \
    xsltproc \
    xterm \
    xz-utils \
    zlib1g-dev \
    ca-certificates \
    less \
    sudo \
    tmux \
    vim
RUN curl https://storage.googleapis.com/git-repo-downloads/repo > /bin/repo && chmod a+x /bin/repo
# repo and git need an identity for sync/commits inside the container.
RUN git config --system user.name "qemu-trust-lab" \
 && git config --system user.email "lab@example.invalid" \
 && git config --system color.ui false
WORKDIR /optee
