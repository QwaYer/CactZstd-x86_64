# SPDX-License-Identifier: GPL-3.0-or-later
# Copyright (C) 2026 CactOS Project

# ==============================================================================
# CactZstd-x86_64 — the zstd command-line tool (and library) for CactOS
# ==============================================================================
#
# The tree in the root is upstream zstd (a fork of meta/zstd, see UPSTREAM.md);
# this build is the Cact addition.  It compiles the library objects and the
# command-line front end against CactLibc and links one PIE:
#
#   build-meson/zstd      the CLI (fixed base 0x08000000, link.ld)
#
#   meson setup build-meson --cross-file cross/x86_64-cact-clang.ini
#   ninja -C build-meson            # build-meson/zstd
#   ninja -C build-meson stage      # zstd into -Dlr_bin (/usr/bin)
#
# The CLI is built multithreaded (-DZSTD_MULTITHREAD): CactLibc's pthreads back
# threading.c, the thread pool and fileio_asyncio.c, so -T# works.  Every other
# upstream feature — compression, decompression, listing, --train, the bench and
# the legacy (v0.5+) decoders — is compiled in.
# ==============================================================================

project('cact-zstd', 'c',
  version: '1.5.7',
  meson_version: '>=1.0.0',
  default_options: [
    'c_std=gnu99',
    'warning_level=0',            # zstd is vendored; -w below
    'buildtype=plain',
    'b_staticpic=false',
  ])

fs = import('fs')
bash = find_program('bash')
ld = find_program('ld')

src_root = meson.project_source_root()

# Path options are relative to this repo's root unless absolute.
_cactlib = get_option('cactlib')
cactlib = _cactlib.startswith('/') ? _cactlib : src_root / _cactlib
_lr_bin = get_option('lr_bin')
lr_bin = _lr_bin.startswith('/') ? _lr_bin : src_root / _lr_bin

cactlib_inc = cactlib / 'include'
start_o = cactlib / 'build-meson' / 'start.o'
clibc_so = cactlib / 'build-meson' / 'clibc.so'

if not fs.exists(start_o) or not fs.exists(clibc_so)
  error('CactLibc is not built: expected ' + start_o +
        '  —  build it first:  ninja -C ' + cactlib / 'build-meson')
endif

# Every object is position independent: the CLI is a PIE.
add_project_arguments(['-fPIE', '-ffunction-sections', '-fdata-sections'],
                      language: 'c')

# zstd is vendored, so its warnings stay out of the CactOS build log (-w).
#   -D__CACT__                marks a CactLibc target (what <syscall.h> keys on)
#   -DPLATFORM_POSIX_VERSION  platform.h would otherwise read _POSIX_VERSION,
#                             which a freestanding libc does not publish; the
#                             value turns on the POSIX code paths (file lists,
#                             console detection, signal handling) verbatim.
#   -DZSTD_LEGACY_SUPPORT=5   the upstream default: decode legacy (v0.5+) frames
#   -DZSTD_MULTITHREAD        worker threads (-T#): CactLibc's pthreads back
#                             threading.c, the thread pool and fileio_asyncio.c
#   -DZSTD_DISABLE_ASM        keep the amd64 BMI2 assembly out: zstd gates it on
#                             __linux__/__APPLE__/Windows (portability_macros.h),
#                             so enabling it would mean advertising __linux__ to
#                             the whole front end for a decompress-only speedup
zstd_c_args = [
  '-ffreestanding', '-fno-stack-protector', '-nostdlib', '-nostdlibinc',
  '-D__CACT__', '-DPLATFORM_POSIX_VERSION=200809L',
  '-DZSTD_LEGACY_SUPPORT=5', '-DZSTD_MULTITHREAD', '-DZSTD_DISABLE_ASM',
  '-w',
  '-I' + (src_root / 'lib'),
  '-isystem', cactlib_inc,
]

# The library objects: libzstd.mk's ZSTD_COMMON/COMPRESS/DECOMPRESS/DICTBUILDER
# lists plus the v0.5+ legacy decoders (ZSTD_LEGACY_SUPPORT=5).
zstd_lib = static_library('cact-zstd-lib',
  files(
    'lib/common/debug.c',
    'lib/common/entropy_common.c',
    'lib/common/error_private.c',
    'lib/common/fse_decompress.c',
    'lib/common/pool.c',
    'lib/common/threading.c',
    'lib/common/xxhash.c',
    'lib/common/zstd_common.c',
    'lib/compress/fse_compress.c',
    'lib/compress/hist.c',
    'lib/compress/huf_compress.c',
    'lib/compress/zstd_compress.c',
    'lib/compress/zstd_compress_literals.c',
    'lib/compress/zstd_compress_sequences.c',
    'lib/compress/zstd_compress_superblock.c',
    'lib/compress/zstd_double_fast.c',
    'lib/compress/zstd_fast.c',
    'lib/compress/zstd_lazy.c',
    'lib/compress/zstd_ldm.c',
    'lib/compress/zstdmt_compress.c',
    'lib/compress/zstd_opt.c',
    'lib/compress/zstd_preSplit.c',
    'lib/decompress/huf_decompress.c',
    'lib/decompress/zstd_ddict.c',
    'lib/decompress/zstd_decompress.c',
    'lib/decompress/zstd_decompress_block.c',
    'lib/dictBuilder/cover.c',
    'lib/dictBuilder/divsufsort.c',
    'lib/dictBuilder/fastcover.c',
    'lib/dictBuilder/zdict.c',
    'lib/legacy/zstd_v05.c',
    'lib/legacy/zstd_v06.c',
    'lib/legacy/zstd_v07.c',
  ),
  c_args: zstd_c_args)

# The front end: the upstream default `zstd` program's objects.  fileio_asyncio.c
# is the async-I/O pool layer, built here against CactLibc's pthreads.
zstd_prog = static_library('cact-zstd-prog',
  files(
    'programs/benchfn.c',
    'programs/benchzstd.c',
    'programs/datagen.c',
    'programs/dibio.c',
    'programs/fileio.c',
    'programs/fileio_asyncio.c',
    'programs/lorem.c',
    'programs/timefn.c',
    'programs/util.c',
    'programs/zstdcli.c',
    'programs/zstdcli_trace.c',
  ),
  c_args: zstd_c_args)

zstd_bin = custom_target('zstd',
  output: 'zstd',
  input: [zstd_prog, zstd_lib],
  command: [
    ld, '-m', 'elf_x86_64', '-pie', '--dynamic-linker=/lib/ld.so',
    '--hash-style=both', '-nostdlib', '--gc-sections',
    '-T', src_root / 'link.ld', '-o', '@OUTPUT@',
    start_o,
    '--whole-archive', '@INPUT0@', '@INPUT1@', '--no-whole-archive',
    clibc_so,
  ],
  build_by_default: true)

run_target('stage',
  command: [bash, '-c', 'set -e; mkdir -p "$1"; cp -f "$2" "$1/zstd"',
            'bash', lr_bin, zstd_bin])
