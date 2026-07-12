# Building m1n1 on FreeBSD

Tested on FreeBSD 15.0-RELEASE-p4 amd64 (2026-07-11).

## Toolchain

Two packages needed from the FreeBSD ports tree:

```sh
sudo pkg install -y \
    aarch64-none-elf-gcc \
    rust
```

- `aarch64-none-elf-gcc` provides the bare-metal cross-compiler chain
  under `/usr/local/bin/aarch64-none-elf-*`.
- `rust` (currently 1.94.0) ships the compiler *plus* the
  `/usr/local/lib/rustlib/src/rust/library/` tree, which is what
  `-Z build-std` needs to compile `core` + `alloc` for the
  `aarch64-unknown-none-softfloat` target.  There is no `rustup` in the
  FreeBSD ports tree — the pkg install is self-sufficient.

No `aarch64-linux-gnu-*` toolchain is needed.  m1n1's Makefile default
`ARCH ?= aarch64-linux-gnu-` must be overridden with the FreeBSD-native
`aarch64-none-elf-` prefix.

## Build

```sh
env ARCH=aarch64-none-elf- BUILDSTD=1 gmake -j4
```

- `ARCH=aarch64-none-elf-` steers the C toolchain to the pkg-installed
  cross-compiler.
- `BUILDSTD=1` enables `CARGO_FLAGS := -Z build-std=alloc,core`
  (see `Makefile:77-81`).  Without it, cargo tries to link against a
  pre-built `core` crate for `aarch64-unknown-none-softfloat` — which
  the FreeBSD pkg rust doesn't ship, since no `rustup` target-add step
  is available.  `RUSTC_BOOTSTRAP=1` (already exported by the
  Makefile) is what lets stable Rust use the nightly `-Z build-std`
  flag.

FreeBSD uses `gmake`, not `make`, for GNU Make compatibility.

## Expected build outputs (~25 s on a modern amd64 host)

`build/m1n1.bin` — flat binary for the raw-mode `kmutil configure-boot`
install path (see main README §Usage).  ~1.07 MiB.

`build/m1n1.macho` — Mach-O for the legacy install path.  ~880 KiB.

`build/m1n1.elf` — debug ELF with symbols.  Useful when running the
hypervisor tracer scripts under `proxyclient/hv/` and correlating
addresses back to source.

## Docker / podman alternative

If you would rather not install Rust on the host, the container flow in
the top-level README (`podman-compose run m1n1 make`) works on FreeBSD
too — the container ships all build dependencies.  For iterating on
m1n1 source, the native FreeBSD build above is roughly 3× faster than
the container round-trip.

## Common issues

**"cannot find `core`" / `E0463`**

Cargo is trying to link a prebuilt standard library.  Set `BUILDSTD=1`
in the build environment.

**"'stdint.h' file not found"**

The build has picked up `clang --target=…` without freestanding headers.
Explicitly set `ARCH=aarch64-none-elf-` so the pkg-installed
`aarch64-none-elf-gcc` is used instead.

**"gmake: cargo: No such file or directory"**

FreeBSD pkg rust doesn't add itself to root's PATH by default.  Either
run `make` from a shell with `/usr/local/bin` in `PATH` (usually the
default) or invoke as `env PATH=/usr/local/bin:$PATH gmake ...`.
