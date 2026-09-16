# OpenWrt HA Cluster feed - containerised build environment
#
# Copyright (C) 2025-2026 Pierre Gaufillet <pierre.gaufillet@bergamote.eu>
#
# Builds the ha-cluster feed packages using the official OpenWrt SDK.
# The SDK ships a prebuilt cross-toolchain, so packages compile in minutes
# without rebuilding the whole toolchain (unlike a full buildroot clone).
#
# Build the image for the default target (x86-64):
#   docker build -t ha-feed-build docker/
#
# Build for a different target (e.g. MediaTek Filogic / aarch64):
#   docker build -t ha-feed-build-filogic \
#     --build-arg OPENWRT_TARGET=mediatek/filogic docker/
#
# Compile all packages, dropping .apk files into ./output on the host:
#   docker run --rm -v "$PWD/output:/output" ha-feed-build
#
# See docker/README.md for the full guide (signing, custom feed source, etc.).

FROM debian:bookworm-slim

# --- Build arguments ---------------------------------------------------------
# OpenWrt release the SDK is taken from. Must match the firmware on the router.
ARG OPENWRT_VERSION=25.12.0
# Target/subtarget as used in the OpenWrt download tree (target/subtarget).
# Examples: x86/64, mediatek/filogic, ath79/generic, ramips/mt7621
ARG OPENWRT_TARGET=x86/64
# Toolchain suffix embedded in the SDK tarball name. Bump when OpenWrt does.
ARG OPENWRT_GCC=gcc-14.3.0_musl
# Feed source. Defaults to the public repository so the image is self-contained.
# Override with a fork/branch, or bind-mount a local checkout at /feed and pass
# HA_FEED_SRC="src-link ha_feed /feed".
ARG HA_FEED_SRC="src-git ha_feed https://github.com/pgaufillet/openwrt-ha-feed.git"

# --- Host build dependencies -------------------------------------------------
# Package list follows the OpenWrt "build system setup" docs (Debian), plus
# zstd for the .tar.zst SDK archive.
RUN apt-get update && \
    DEBIAN_FRONTEND=noninteractive apt-get install -y --no-install-recommends \
        build-essential \
        ca-certificates \
        file \
        gawk \
        gettext \
        git \
        libncurses-dev \
        libssl-dev \
        python3 \
        python3-distutils-extra \
        rsync \
        unzip \
        wget \
        xsltproc \
        zlib1g-dev \
        zstd \
    && rm -rf /var/lib/apt/lists/*

# The OpenWrt SDK refuses to run as root. Create an unprivileged build user.
RUN useradd -m -s /bin/bash build
USER build
WORKDIR /home/build

# --- Download and unpack the SDK ---------------------------------------------
# Derived name: mediatek/filogic -> mediatek-filogic (slashes become dashes).
RUN set -eux; \
    target_flat="$(echo "${OPENWRT_TARGET}" | tr '/' '-')"; \
    sdk="openwrt-sdk-${OPENWRT_VERSION}-${target_flat}_${OPENWRT_GCC}.Linux-x86_64"; \
    url="https://downloads.openwrt.org/releases/${OPENWRT_VERSION}/targets/${OPENWRT_TARGET}/${sdk}.tar.zst"; \
    echo "Fetching ${url}"; \
    wget -q "${url}" -O sdk.tar.zst; \
    tar --use-compress-program=unzstd -xf sdk.tar.zst; \
    rm sdk.tar.zst; \
    mv "${sdk}" openwrt-sdk

WORKDIR /home/build/openwrt-sdk

# --- Register the ha_feed feed -----------------------------------------------
# Keep the SDK's default upstream feeds, then append ha_feed.
RUN set -eux; \
    cp feeds.conf.default feeds.conf; \
    echo "${HA_FEED_SRC}" >> feeds.conf

# --- Build entrypoint --------------------------------------------------------
COPY --chown=build:build build-packages.sh /home/build/build-packages.sh
ENTRYPOINT ["/home/build/build-packages.sh"]
