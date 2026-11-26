# Upstream provenance — zstd v1.5.7

`CactZstd-x86_64` is a **fork** of the upstream zstd repository: the tree at the
root *is* upstream zstd, and every CactOS-specific file is a commit on the `cact`
branch stacked on the pinned release tag.

| | |
|---|---|
| Upstream | <https://github.com/meta/zstd> (the older `facebook/zstd` URL redirects here) |
| Pin | tag `v1.5.7` — commit `f8745da6ff1ad1e7bab384bd1f9d742439278e99` |
| Branch | `cact` — the Cact commits on top of the pin |
| Licence | BSD-3-Clause (library) / GPLv2 (programs) — see `LICENSE` and `COPYING` |

Nothing upstream is edited.  What the `cact` branch adds on top of `v1.5.7`:

| Path | |
|---|---|
| `meson.build`, `meson_options.txt`, `cross/`, `link.ld` | the CactOS build |
| `sync.conf`, `sync.sh` | upstream sync (see `../.cactsync`) |
| `UPSTREAM.md`, `PORTING-x86_64.md` | provenance and the port plan |
| `README.md`, `.gitignore` | the CactOS front page and the build-tree ignore |

`lib/` and `programs/` are untouched: every environment difference lives in
`meson.build`'s `zstd_c_args` (`-D__CACT__`, `-DPLATFORM_POSIX_VERSION=200809L`,
`-DZSTD_LEGACY_SUPPORT=5`, `-DZSTD_DISABLE_ASM`) and in the object lists there.
The kernel tree vendors the *same* release (`1.5.7`) under
`CactKernel-x86_64/Cact/drivers/zstd`, so the in-kernel and userspace
decompressors stay in step.

## Sync

`upstream` is <https://github.com/meta/zstd> (`pushurl=no_push`, so a push can
never reach it); `origin` is the published repo `QwaYer/CactZstd-x86_64`.  Moving
to a newer release is a rebase of the Cact commits, driven by `sync.conf`:

```sh
./sync.sh --list                 # candidate tags, current pin marked
./sync.sh --to v1.5.8 --build    # rebase onto the tag and build
```

`lib/libzstd.mk` and `programs/Makefile` upstream are the source lists that
`meson.build` mirrors — re-check them after a rebase.
