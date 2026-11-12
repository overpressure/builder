# SPDX-License-Identifier: Unlicense
# overpressure's build image: the Linux side of every build (userspace, system image, Valve's kernel,
# packages), on any host with Docker or Podman. It runs privileged: the builds mount /proc, /sys
# and overlayfs and chroot into an aarch64 root, whose programs the host kernel's binfmt handler
# runs through QEMU (binfmt.sh registers this image's qemu-aarch64-static when an x86-64 engine has
# no handler). The x86-64 clang and lld (LLVM 20) compile natively inside that root.
#
# The base is pinned by digest; the packages come from Ubuntu's archive as it is when the image is
# built (ports.ubuntu.com on arm64). What pins a build's tools is the published image, by digest
# (README.md): consumers pull it, nobody needs to rebuild it to get the same tools.
FROM ubuntu:24.04@sha256:24737cef17f4e22b1977df7d9a05e77c4e28b7da5b8911f8d8ccda136bf3573c
ARG DEBIAN_FRONTEND=noninteractive
# The kernel's cross compiler only where the host is no arm64 (there the native gcc has its name).
RUN set -eu; \
    arch=$(dpkg --print-architecture); \
    apt="apt-get -o Acquire::Retries=5"; \
    cross=$([ "$arch" = arm64 ] || echo gcc-aarch64-linux-gnu); \
    $apt update; \
    $apt install -y --no-install-recommends \
      bash ca-certificates coreutils curl diffutils findutils tar zstd xz-utils python3 e2fsprogs util-linux \
      clang-20 lld-20 qemu-user-static \
      make gcc libc6-dev $cross flex bison bc git perl cpio libssl-dev libelf-dev dwarves device-tree-compiler \
      shellcheck; \
    rm -rf /var/lib/apt/lists/*; \
    dpkg-query -W -f '${Package} ${Version} ${Architecture}\n' | sort > /usr/share/overpressure-packages.txt; \
    git config --system --add safe.directory '*'  # the checkout (/work, a worktree's /git) is the host user's, git here runs as root
COPY entrypoint.sh /usr/local/bin/overpressure-entrypoint
COPY binfmt.sh /usr/local/bin/overpressure-binfmt
COPY hand-back.sh /usr/local/bin/overpressure-hand-back
RUN chmod 0755 /usr/local/bin/overpressure-entrypoint /usr/local/bin/overpressure-binfmt \
      /usr/local/bin/overpressure-hand-back
LABEL org.opencontainers.image.source="https://github.com/overpressure/builder" \
      org.opencontainers.image.description="overpressure's build image: the Linux side of the headsets' builds" \
      org.opencontainers.image.licenses="Unlicense"
WORKDIR /work
ENTRYPOINT ["/usr/local/bin/overpressure-entrypoint"]
CMD ["bash"]
