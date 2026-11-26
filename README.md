# 🌐 CactZstd-x86_64

<p align="center">
  <img src="https://img.shields.io/badge/version-1.5.7-green.svg?style=for-the-badge" alt="Version: 1.5.7">
  <img src="https://img.shields.io/badge/license-BSD--3--Clause%2FGPLv2-blue.svg?style=for-the-badge" alt="License: BSD-3-Clause / GPLv2">
  <img src="https://img.shields.io/badge/arch-x86_64-red.svg?style=for-the-badge" alt="Arch: x86_64">
  <img src="https://img.shields.io/badge/output-zstd-orange.svg?style=for-the-badge" alt="Output: zstd">
</p>

<p align="center">
  A <a href="https://github.com/meta/zstd">zstd</a> <strong>fork</strong> for
  CactOS: the tree at the root is upstream zstd pinned at <strong>v1.5.7</strong>,
  and the CactOS build that links the CLI against CactLibc is the
  <code>cact</code> branch stacked on that tag.
</p>

---

## 📦 What this is

`zstd` is the standard compression tool.  On CactOS it replaces the bespoke
`firmextr`: the boot modules (`firmrepo`, `netdrvrepo`, `drvrepo`,
`localbinrepo`) are ustar images wrapped in one zstd frame, so a member is
reached with the ordinary pipeline

```sh
zstd -dc /dev/firmrepo | tar -xf - usr/lib/firmware/rt2870.bin
```

The installer and the shell use the same tool, so no CactOS-specific extractor
is needed.

## 🔨 Building

```sh
meson setup build-meson --cross-file cross/x86_64-cact-clang.ini
ninja -C build-meson            # build-meson/zstd
ninja -C build-meson stage      # → -Dlr_bin (/usr/bin)
```

Override paths if needed:

```sh
meson configure build-meson -Dcactlib=/path/to/CactLibc-x86_64
```

The CLI is built **multithreaded** (`-DZSTD_MULTITHREAD`): CactLibc's pthreads
back `threading.c`, the thread pool and `fileio_asyncio.c`, so `-T#` works.
Compression, decompression, `-l`, `--train`, the bench (`-b`), the legacy
(v0.5+) decoders and the trace hook are all compiled in.  The amd64 BMI2
assembly is left out (see `meson.build`): zstd gates it on
`__linux__`/`__APPLE__`/Windows, and advertising `__linux__` to the whole front
end would change unrelated platform behaviour for a decompress-only speedup.

## 🗂️ Layout

The root is upstream zstd — every `Makefile`, `build/`, `tests/`, `doc/`, …
file there is upstream's.  The `cact` branch adds only:

```
CactZstd-x86_64/
├── cross/            ← Meson cross file (CactOS userspace)
├── link.ld           ← PIE layout @ 0x08000000, shared with the other tools
├── meson.build       ← the Cact addition: builds the CLI against CactLibc
├── meson_options.txt
├── sync.conf         ← upstream sync manifest (see ../.cactsync)
├── sync.sh
├── UPSTREAM.md       ← provenance
└── PORTING-x86_64.md
```

`lib/` and `programs/` are never edited — every environment difference is a
compile flag in `meson.build`.

## 🔄 Syncing with upstream

`upstream` is `meta/zstd` with `pushurl=no_push`; `origin` is this repository, so
a push can never reach upstream by accident.  To move to a newer release:

```sh
./sync.sh --list                 # candidate tags, current pin marked
./sync.sh --to v1.5.8 --build    # rebase the Cact commits onto the tag and build
```

## 🔗 Ecosystem

| Piece | Role |
|-------|------|
| **[CactLibc-x86_64](https://github.com/QwaYer/CactLibc-x86_64)** | C library the tool links against |
| **[CactKernel-x86_64](https://github.com/QwaYer/CactKernel-x86_64)** | Serves the boot archives as `/dev/firmrepo` and friends |
| **[LocalRepoCactOS-x86_64](https://github.com/QwaYer/LocalRepoCactOS-x86_64)** | Receives the staged `zstd` for the localbinrepo image |
| **[Cact-install-x86_64](https://github.com/QwaYer/Cact-install-x86_64)** | Uses `zstd` to read the boot archives |
