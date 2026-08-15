# Edge QuickJS Linux-on-WebAssembly port

| | | Remarks |
| --- | --- | --- |
| **Status** | ✅ | The embedded QuickJS Edge runtime builds and passes its focused Linux-wasm VM proof. |
| **Severity** | High | The target cannot build or run Edge until its static Linux platform constraints are represented. |

## Goal

Build the Edge.js CLI with `EDGE_NAPI_PROVIDER=quickjs` as a normal Linux
userspace WebAssembly executable for the Lowland WASM kernel, package the same
binary as both `edge` and `node`, and prove this exact guest command:

```sh
node -e 'console.log(process.version)'
```

The target is Linux and uses musl plus the kernel's ordinary syscall ABI. It is
not WASI or WASIX, and it must not import Wasmer's WASIX interfaces.

## Source baseline

- Edge.js: `1ca99ab4ff3d74bf5940177eb007457612d398e3`
- N-API submodule: `c5b66fb9f5b1b997d5bdd463dc1a80bb174d4730`
- QuickJS submodule: `9d5513a65693e4fc16f48975df59f6fa62f6a9b0`
- Linux target triple: `wasm32-unknown-linux-musl`

The Edge checkout is persistent at `/home/alex/linux-wasm/edgejs`; the distro
package reconstructs the same source tree from commit-pinned archives until a
project fork becomes the canonical source.

## Initial platform gaps

1. Edge's native CMake path supports vendored OpenSSL only for x86, x86-64,
   and arm64, while its system OpenSSL option requires shared libraries. The
   WASM Linux platform is static-only and already provides ported static
   `libssl.a` and `libcrypto.a` packages.
2. `EDGE_QUICKJS_WEBASSEMBLY` must be disabled. That option is the nested
   WebAssembly API exposed inside Edge via Wasmer's C API, not the fact that the
   Edge executable itself is WebAssembly.
3. The ordinary Linux dependency paths (libuv, c-ares, ICU, zlib, and the
   Node-compatible runtime) must compile without assuming ELF, `fork`, `mmap`,
   dynamic loading, or a native machine architecture.
4. The installed `node` must be a real file rather than an executable symlink,
   because the WASM kernel cannot execute through such symlinks.

## Action plan

1. Add an explicit CMake mode for static system OpenSSL, preserving the
   existing shared-system and vendored modes.
2. Configure the distro package against its target OpenSSL and build only the
   embedded-QuickJS `edge` executable.
3. Fix each compiler or linker failure at the narrowest truthful platform
   boundary, preferring reusable upstream portability changes over package-only
   source rewrites.
4. Add a VM installed test that boots the real WASM Linux kernel and asserts a
   non-empty `v*` `process.version` from the `node -e` command.
5. Include Edge.js in the published repository and runner rootfs only after the
   focused runtime proof passes.
6. Record final build/runtime evidence and any retained limitations here.

## Implemented platform adaptations

- Added a static-system OpenSSL mode and made it reject non-static imported
  targets, so cross builds cannot silently pick an unusable shared library.
- Cleared the wasm host-import decorations on both public N-API surfaces when
  QuickJS is embedded, so this Linux executable provides those symbols itself
  instead of importing Wasmer's `napi` modules.
- Kept llhttp's standalone bare-wasm callback shim out of wasm Linux builds;
  the normal Node/Edge callback API is linked instead.
- Narrowed WASIX-only runtime fallbacks to `__wasi__`, allowing wasm Linux to
  use libuv's native memory, load, interface, TTY, and HTTP/2 behavior.
- Ported Edge's compatibility-command launchers from `fork`/`exec` to
  `posix_spawn` on wasm. PATH lookup uses the actual child environment rather
  than musl's parent-environment `posix_spawnp` behavior.
- Ported libuv's process launcher to its full posix-spawn flow on wasm,
  including stdio actions, cwd, signal state, detached sessions, and child
  environment PATH resolution. Arbitrary uid/gid replacement remains
  unsupported because POSIX spawn has no portable equivalent.
- Disabled libuv's optional io_uring mmap fast path on wasm. Its ordinary
  epoll and worker-thread paths remain active.
- Selected ICU's built-in stdio data loader on wasm while retaining the
  embedded ICU common-data archive.

The distro carries these source changes as `linux-wasm-port.patch` against the
commit-pinned archive. They remain staged in the persistent checkout as the
candidate content for the future project fork.

## Toolchain adaptations

The port also exposed two target-runtime gaps in the distro toolchain:

- The wasm Clang wrapper does not yet discover the sysroot's libc++ headers or
  select libc++ for C++ links. The Edge package supplies those C++-only flags
  explicitly so they cannot affect C compilation.
- libc++abi requires an unwinder for real C++ exceptions. The target runtimes
  now build static LLVM libunwind, use it from libc++abi, and avoid treating
  wasm as a Windows-style export platform. The cross-build also forces
  `LIBCXXABI_HAS_CXA_THREAD_ATEXIT_IMPL=OFF`: static-library CMake probes cannot
  determine that musl lacks glibc's `__cxa_thread_atexit_impl`, while libc++abi's
  pthread-key fallback is valid here.

## Verification

The packaged CLI built successfully as a 55 MiB WebAssembly executable. Both
`$out/bin/edge` and `$out/bin/node` are real files with identical contents.
An undefined-symbol audit found only the Linux-wasm kernel ABI imports:

```text
__get_tp
__wasm_copy_siginfo
__wasm_syscall
```

There are no Wasmer N-API, wasm-c-api, or WASI imports. The focused derivation
`/nix/store/i491slinbpyvdax6fhprk3fwnhpppr2d-vm-test-edgejs-node-version`
booted Linux 7.1.5 and ran the exact requested command. Its guest console ended
with:

```text
::edgejs::process.version=v24.13.2
::vm-test::pass
```

After that proof passed, Edge.js was added to the published APK repository and
the default runner root filesystem.
