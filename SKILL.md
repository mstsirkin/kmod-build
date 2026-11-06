---
name: kmod-build
description: Fast in-tree kernel module rebuild against installed kernel-devel. Use when rebuilding a kernel module (KVM, drivers, etc.) without a full kernel build. Handles GCC version mismatches, BTF tooling issues, multi-directory modules with M=, and trace header fixups.
allowed-tools: Shell
metadata:
  author: Michael S. Tsirkin <mst@kernel.org>
  version: 0.1.0
  category: kernel
---

# Fast in-tree kernel module rebuild

Rebuild individual kernel modules against an installed kernel-devel
tree without a full kernel build. Seconds instead of 10+ minutes.

## When to use

- You patched a kernel module (KVM, a driver, etc.) and want to
  test it against a running kernel
- The module source spans multiple directories (e.g. KVM uses
  both arch/x86/kvm/ and virt/kvm/)
- Your host GCC is newer than the kernel was built with

## When NOT to use

- Your changes touch code outside the module (core kernel,
  headers used by other subsystems, vmlinux built-in code).
  In that case you need a full kernel build - the module
  links against vmlinux symbols and a stale bzImage won't
  have your changes.
- You changed kernel headers that affect struct layouts
  visible to the module. The module will compile against the
  new headers but the running kernel has the old layout -
  instant corruption at runtime.

## Setup

### 1. Locate or install kernel-devel

The kernel-devel tree provides headers, Makefiles, scripts, and
Module.symvers matching the target kernel.

```bash
# If installed as a package:
KDEV=/usr/src/kernels/$(uname -r)

# If extracted from RPM into a custom root:
rpm2cpio kernel-devel-<version>.rpm | (cd $ROOT && cpio -idm)
KDEV=$ROOT/usr/src/kernels/<version>
```

Verify it has what we need:

```bash
ls "$KDEV/Makefile" "$KDEV/Module.symvers" "$KDEV/scripts/mod/modpost"
```

### 2. Fix resolve_btfids (if GCC mismatch)

If `make prepare` fails building resolve_btfids because libbpf
triggers `-Werror` with your GCC, two options:

**Option A** - copy the tool from a working kernel-devel on the same host:

```bash
cp /usr/src/kernels/$(uname -r)/tools/bpf/resolve_btfids/resolve_btfids \
   "$KDEV/tools/bpf/resolve_btfids/resolve_btfids"
```

**Option B** - suppress the warning during build:

```bash
make -C "$KDEV" prepare HOSTCFLAGS="-Wno-error=discarded-qualifiers"
```

The resolve_btfids Makefile (tools/bpf/resolve_btfids/Makefile)
forwards HOSTCFLAGS as EXTRA_CFLAGS to the libbpf build.

### 3. Fix trace headers

Kernel tracepoint headers use TRACE_INCLUDE_PATH which resolves
relative to `include/trace/` in the kernel-devel tree. If your
module defines tracepoints, symlink its trace header:

```bash
# For KVM:
ln -sf "$SRCTREE/arch/x86/kvm/trace.h" "$KDEV/arch/x86/kvm/trace.h"

# General pattern - find TRACE_INCLUDE_PATH in your module's trace header:
#   #define TRACE_INCLUDE_PATH ../../path/to/module
# Then symlink trace.h into that path under $KDEV
```

## Build

```bash
KDEV=/path/to/kernel-devel
SRCTREE=/path/to/patched/kernel/source
MODULE_DIR=arch/x86/kvm   # directory containing the module Makefile

make -C "$KDEV" \
  M="$SRCTREE/$MODULE_DIR" \
  KCFLAGS="-I$SRCTREE/$MODULE_DIR" \
  KBUILD_MODPOST_WARN=1 \
  -j$(nproc) modules
```

Output .ko files appear in `$SRCTREE/$MODULE_DIR/`.

### What the flags do

- `M=` - build only the module in this directory as an external module
- `KCFLAGS="-I..."` - the module Makefile uses `ccflags-y += -I $(srctree)/...`
  for relative includes; $(srctree) points to kernel-devel (no source),
  so we add the real source path
- `KBUILD_MODPOST_WARN=1` - demote unresolved symbol errors to warnings;
  symbols exist in the running kernel and resolve at load time
- BTF generation is skipped (no vmlinux) but struct module ABI matches
  because the config matches the original kernel

### Multi-directory modules

Some modules span multiple directories. KVM uses:
- `arch/x86/kvm/` - main module directory with Makefile
- `virt/kvm/` - shared KVM core (kvm_main.c, eventfd.c, etc.)

The KVM Makefile includes virt/kvm/Makefile.kvm which sets:
```
KVM ?= ../../../virt/kvm
```

This relative path resolves from the M= directory, so it finds
the source files in the patched tree automatically. No extra
symlinks needed for source files - only for trace headers.

## Testing the module

```bash
# Load into a test VM (e.g. via vmtest):
KVMKO=$SRCTREE/arch/x86/kvm/kvm.ko \
KVMINTEL=$SRCTREE/arch/x86/kvm/kvm-intel.ko \
bash run-test.sh
```

## Gotchas

- **Do NOT disable CONFIG_DEBUG_INFO_BTF** to work around tool
  build failures. It adds fields to struct module - modules built
  without it won't load against a kernel built with it:
  `module: .gnu.linkonce.this_module section size must match`

- **modversions CRC mismatch** can occur if the module exports
  symbols whose signatures changed between kernel versions or
  compilers. KBUILD_MODPOST_WARN=1 lets the build succeed;
  if the module fails to load with "disagrees about version of
  symbol", you need a full kernel build with matching compiler.

- **GCC version differences** in generated code are usually safe
  for modules (struct layouts are determined by config + source,
  not compiler). But if you see crashes in module code, rebuild
  with the original compiler.
