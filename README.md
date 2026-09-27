# cargo-build-sbpf

Build an SBPF program with upstream nightly Rust.

## CLI syntax

```text
cargo build-sbpf

USAGE:
    cargo build-sbpf [OPTIONS]

OPTIONS:
        --arch <ARCH>    SBPF architecture to build for [possible values: v0, v3]
        --dump <DIR>     Dump the linked LLVM module and control-flow graphs into this directory
    -v, --verbose        Show the Cargo command and enable Cargo's verbose output
    -h, --help           Print help
    -V, --version        Print version
```

The architecture comes from `--arch`, then Cargo config, and defaults to V3
when neither specifies it. V0 can be selected with the only supported
option:

```sh
cargo build-sbpf --arch v0
```

Dump the linked LLVM module and its control-flow graphs into one directory:

```sh
cargo build-sbpf --dump target/sbpf-dump
```

This writes the LLVM `.ll` dumps and CFG `.dot` files to
`target/sbpf-dump`.

Use `-v` or `--verbose` to print the underlying Cargo command and enable
Cargo's verbose build output.

Before building, the command checks the toolchain and project dependencies.
It asks for permission before fixing anything:

- A nightly toolchain and `sbpf-linker` 0.2.1 or newer are required. The build
  stops if either is unavailable and its installation is declined.
- LLVM 23 is recommended because LLVM 22 generates less optimal SBPF code. If
  updating nightly is declined, the command warns and continues.
- `solana-compiler-builtins` is recommended because its compiler builtins are
  optimized for the SVM. If adding it is declined, the command warns and
  continues.

These checks do not read or modify Cargo config. A linker installed in
`$CARGO_HOME/bin` is used even when that directory is not already on `PATH`.

Build configuration is applied as follows:

1. When Cargo finds no `.cargo/config.toml` or legacy `.cargo/config` in its
   configuration hierarchy, including `$CARGO_HOME`, the SBPF rustflags are
   supplied through `CARGO_TARGET_BPFEL_UNKNOWN_NONE_RUSTFLAGS`.
2. When a config exists, Cargo reads its rustflags and `cargo-build-sbpf`
   supplies `sbpf-linker` through Cargo's command-line config.
3. The config's architecture must not conflict with `--arch`. If it does not
   specify an architecture, the selected architecture is passed to Cargo
   without modifying the config file.
4. The config's BPF stack size must match the selected policy: V0 uses 8192
   bytes before SIMD-0460 and 4096 bytes with SIMD-0460; V3 uses 4096 bytes. If
   the stack size is missing or mismatched, the command asks permission to
   update the Cargo config.
5. The injected rustflags select `target_os="solana"` and the `static-syscalls`
   target feature so SBPF crates compile their on-chain code paths. The explicit
   built-in cfg lint is allowed for these target cfgs.
