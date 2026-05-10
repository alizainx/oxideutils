# OxideUtils

Memory-safe, high-performance rewrite of GNU binutils in pure Rust.

Drop-in compatible. Better defaults. Production quality.

[![CI](https://github.com/your-org/oxideutils/actions/workflows/ci.yml/badge.svg)](https://github.com/your-org/oxideutils/actions/workflows/ci.yml)
[![License: MIT OR Apache-2.0](https://img.shields.io/badge/license-MIT%20OR%20Apache--2.0-blue)](LICENSE)
[![Rust: 2021 Edition](https://img.shields.io/badge/rust-2021%20edition-orange)](https://doc.rust-lang.org/edition-guide/rust-2021/)

---

## What is OxideUtils?

OxideUtils replaces the GNU binutils tools (`objdump`, `nm`, `readelf`,
`strip`, `objcopy`, `ar`, `size`, `addr2line`, `strings`) with Rust
equivalents that provide:

- **Zero undefined behavior** — memory safety by construction
- **Colored, structured output** by default; plain text with `--gnu-compat`
- **JSON output** (`--json`) for scripting and tooling integration
- **Parallel processing** via rayon where applicable
- **GNU compatibility mode** (`--gnu-compat`) for byte-identical output
- **Clear error messages** — not cryptic C-style one-liners

---

## Quick Start

```bash
git clone https://github.com/your-org/oxideutils
cd oxideutils
cargo build --release --workspace

# Use individual tools
./target/release/oxide-nm path/to/binary
./target/release/oxide-objdump --help

# Or use the unified multicall binary
./target/release/oxideutils nm path/to/binary
./target/release/oxideutils objdump --json path/to/binary
```

## Drop-in GNU Replacement

```bash
# Symlink into PATH as GNU tool names
ln -s $(pwd)/target/release/oxide-nm      /usr/local/bin/nm
ln -s $(pwd)/target/release/oxide-objdump /usr/local/bin/objdump
ln -s $(pwd)/target/release/oxide-readelf /usr/local/bin/readelf

# Or symlink the multicall binary
ln -s $(pwd)/target/release/oxideutils     /usr/local/bin/nm
ln -s $(pwd)/target/release/oxideutils     /usr/local/bin/objdump
```

---

## Tools

| Tool              | GNU Equivalent | Phase | Status      |
|-------------------|----------------|-------|-------------|
| `oxide-objdump`   | `objdump`      | 3     | Skeleton    |
| `oxide-nm`        | `nm`           | 3     | Planned     |
| `oxide-readelf`   | `readelf`      | 3     | Planned     |
| `oxide-size`      | `size`         | 3     | Planned     |
| `oxide-strings`   | `strings`      | 3     | Planned     |
| `oxide-strip`     | `strip`        | 3     | Planned     |
| `oxide-objcopy`   | `objcopy`      | 3     | Planned     |
| `oxide-ar`        | `ar`           | 3     | Planned     |
| `oxide-addr2line` | `addr2line`    | 3     | Planned     |

---

## Supported Formats

- **ELF** — 32-bit and 64-bit, little-endian and big-endian
- **PE/COFF** — Windows executables and object files
- **Mach-O** — macOS executables, fat/universal binaries
- **WebAssembly** — `.wasm` modules
- **Archive** — `.a` static libraries (GNU, BSD, thin formats)
- **DWARF** — debug information via `gimli`

---

## Architecture

All shared logic lives in `crates/oxideutils-core`. Tool crates contain only
a thin `main.rs` that parses CLI arguments and calls into core.

```
oxideutils-core/src/
├── error.rs          # OxideError — single error type for the project
├── utils.rs          # Hex formatting, alignment, address display
├── symbols.rs        # Symbol classification, nm type chars, demangling
├── archive.rs        # Archive magic detection
├── cli/              # Shared CLI layer (Phase 1)
│   ├── config.rs     # GlobalConfig: color, json, gnu-compat, quiet, verbose
│   ├── parser.rs     # CommonArgs — shared clap args for every tool
│   ├── command.rs    # OxideTool enum and name dispatch
│   ├── multicall.rs  # BusyBox-style argv[0] dispatch
│   ├── aliases.rs    # Short-form and GNU-compat alias expansion
│   ├── help.rs       # Consistent help header and footer
│   └── utils.rs      # Sole stdout/stderr boundary for all output
└── format/           # Object file parsers (Phase 2)
    ├── traits.rs     # ObjectFile, SymbolProvider, SectionProvider, …
    ├── elf/          # ELF 32/64, all endianness
    ├── pe/           # PE32 and PE32+
    ├── macho/        # Mach-O and fat binaries
    └── wasm/         # WebAssembly
```

See [`docs/architecture.md`](docs/architecture.md) and
[`PLAN.md`](PLAN.md) for the full three-phase development roadmap.

---

## Development Phases

| Phase | Scope                              | Status      |
|-------|------------------------------------|-------------|
| 1     | `oxideutils-core` CLI + foundation | Complete    |
| 2     | `format/` — all parsers + traits   | Planned     |
| 3     | Tools — one per week               | Planned     |

---

## Dependencies

| Crate               | Purpose                              |
|---------------------|--------------------------------------|
| `object`            | Binary format reading (Phase 2)      |
| `gimli`             | DWARF debug info (Phase 2)           |
| `capstone`          | Disassembly (oxide-objdump)          |
| `clap` (derive)     | CLI argument parsing                 |
| `thiserror`         | Error type derivation                |
| `anyhow`            | Top-level error propagation in tools |
| `rayon`             | Parallelism                          |
| `tracing`           | Structured logging                   |
| `colored`           | Terminal color detection + ANSI      |
| `serde` + `serde_json` | JSON output                       |
| `rustc-demangle`    | Rust and C++ symbol demangling       |
| `insta`             | Snapshot testing                     |
| `criterion`         | Benchmarking                         |

---

## Quality Standards

- `cargo clippy --workspace -- -D warnings` must produce zero diagnostics
- No `unwrap()` or `expect()` in library code (enforced by lint)
- No `println!` / `eprintln!` outside of `cli/utils.rs` (enforced by lint)
- Every public function has a doc comment
- Every module has a module-level doc comment
- All output tools support `--gnu-compat` and `--json`
- GNU compat tests diff output against recorded GNU tool output on the same binaries

---

## Contributing

See [`CONTRIBUTING.md`](CONTRIBUTING.md). All contributions require:

1. Tests (unit tests + snapshot tests for output functions)
2. Documentation on all public items
3. `cargo clippy` clean
4. GNU compatibility verification for any output-producing tool

---

## License

Licensed under either of:

- [Apache License, Version 2.0](LICENSE-APACHE)
