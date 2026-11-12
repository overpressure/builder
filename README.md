# overpressure builder

**TL;DR:** the container image every [overpressure](https://github.com/overpressure/overpressure) build runs
in: the userspace, the system image, Valve's kernel ([overpressure/linux](https://github.com/overpressure/linux))
and packages. Published as `ghcr.io/overpressure/builder` for amd64 and arm64. Consumers pin it by digest, and
that pin is what makes a build's tools the same on every computer and in a year.

## What it holds

Ubuntu 24.04 (the base pinned by digest) with every tool of the builds:

- the kernel's toolchain: `gcc-aarch64-linux-gnu` (on amd64; on arm64 the native `gcc`), binutils, `pahole`
  (dwarves), `device-tree-compiler`, `flex`, `bison`, `bc`, `libssl-dev`, `libelf-dev`;
- LLVM 20's `clang-20` and `lld-20`, which compile natively inside the userspace build's aarch64 root;
- `qemu-user-static`, whose `qemu-aarch64-static` runs that root's programs on an x86-64 host;
- `python3`, `git`, `make`, `e2fsprogs`, `util-linux`, `zstd`, `xz-utils`, `cpio`, `curl`, `shellcheck`.

The packages come from Ubuntu's archive as it was when the image was built. Every installed package and
version is in `/usr/share/overpressure-packages.txt` and attached to the image's release.

## Using it

Pin the image with its tag and digest, as each [release](https://github.com/overpressure/builder/releases)
names it:

```
ghcr.io/overpressure/builder:2026.11.12@sha256:<digest>
```

The digest names a multi-arch index; the engine takes the variant of its host. Reproducible kernel releases
are built with the amd64 variant only: on arm64 the compiler is the native one, which gives other bytes.

## The contract

What overpressure's tooling (`overpressure/linux.py`) relies on. A change to it is a new image that the
consumers pin on purpose.

- **How it runs.** Privileged (`--privileged`): the builds mount `/proc`, `/sys` and overlayfs and chroot.
  The checkout is bound at `/work`, the working directory; build roots live in a volume at `/var/tmp`.
- **The entrypoint**, `/usr/local/bin/overpressure-entrypoint`, then runs the command:
  1. it mounts the host kernel's `binfmt_misc` at `/proc/sys/fs/binfmt_misc` when it is not there;
  2. with `OVERPRESSURE_BINFMT=1` (a build) it runs `overpressure-binfmt`;
  3. with `OVERPRESSURE_UID` and `OVERPRESSURE_GID` (a Linux host's user, not 0) on a rootful engine,
     what root created or changed under `/work` during the command goes back to that user
     (`overpressure-hand-back`), whatever the command's exit status, which stays the container's.
- **`overpressure-binfmt [--report]`** registers this image's `qemu-aarch64-static` as the host kernel's
  aarch64 handler `overpressure-aarch64` with the flags `POF`, unless an enabled handler with `F` is there,
  or the host is arm64. `--report` says which. `BINFMT_MISC` and `BINFMT_QEMU` name other places, for tests.
- **`overpressure-hand-back DIR MARKER UID GID`** gives UID:GID what root owns under DIR and created or
  changed after MARKER was made, on DIR's own file system, links themselves; nothing else.
- **git** trusts every directory (`safe.directory *`): the checkout is the host user's, git here runs as root.

## Publishing

Every push builds both variants, checks their tools and runs the scripts' tests in them
([image.yml](.github/workflows/image.yml)). A tag `YYYY.MM.DD` (or `YYYY.MM.DD.N`) publishes them as one
multi-arch image under that tag, with a release that names its digest and lists each variant's packages.
Published images are kept: an old pin keeps working.

```sh
python3 -m unittest -v            # the recipe test; the script tests need x86-64 Linux, root and bash
docker build -t builder:local image
```

## License

[The Unlicense](LICENSE). The image itself contains Ubuntu's packages under their own licenses.
