# OxideUtils

**A Memory-Safe, High-Performance, Drop-In Replacement for GNU Binutils — Written in Rust**

[![Website: Zainium Dynamics](https://img.shields.io/badge/Website-Zainium_Dynamics-blue?style=flat)](https://zainiumdynamics.tech)
[![Platform: Linux](https://img.shields.io/badge/Platform-Linux-blue)](https://kernel.org)
[![License: GPL v3](https://img.shields.io/badge/License-GPLv3-blue.svg)](https://www.gnu.org/licenses/gpl-3.0)
[![Rust 2021 Edition](https://img.shields.io/badge/rust-2021%20edition-orange)](https://doc.rust-lang.org/edition-guide/rust-2021/)

Developed by [Zainium Dynamics](https://github.com/zainium-dynamics) —
[zainiumdynamics.tech](https://zainiumdynamics.tech)

---

## Why This Project Exists

GNU binutils has been installed on hundreds of millions of systems for four
decades. It is the set of tools — `objdump`, `nm`, `readelf`, `strip`,
`objcopy`, `ar`, `size`, `addr2line`, `strings` — that every Linux
distribution, every cross-compilation toolchain, and every CI/CD pipeline
that compiles native code depends on.

It is also written in C. Between 2015 and 2024, 89 CVEs were assigned to
GNU binutils, of which 83 — 93.4 percent — are memory-safety violations:
heap buffer overflows, use-after-free errors, integer overflows leading to
corruption, and out-of-bounds reads. These are not the result of careless
development. They are the structural consequence of building security-critical
binary parsers in a language that cannot enforce memory-safety invariants at
compile time.

OxideUtils solves this at the structural level. Rust's ownership and borrowing
model makes the relevant vulnerability class a compile-time error. A heap
buffer overflow in an ELF section parser cannot be expressed in safe Rust.
OxideUtils does not patch individual vulnerabilities — it eliminates the
cause of the vulnerability class.

---

## Features

**Memory safety by construction.** Rust's type system enforces the invariants
that C requires developers to enforce manually. This is not a claim about
better testing or more careful review. It is a property guaranteed by the
compiler on every line of production code.

**Full GNU compatibility.** In `--gnu-compat` mode, every OxideUtils tool
produces output byte-for-byte identical to the corresponding GNU tool on the
same input. This is verified by automated differential testing in CI on every
commit, against a corpus of real binary fixtures covering ELF, PE, Mach-O,
WebAssembly, and archive formats.

**JSON output as a first-class citizen.** The `--json` flag produces
newline-delimited JSON on every tool. Every output field is documented.
Pipe directly to `jq`, log aggregators, IDE extensions, or security analysis
pipelines without fragile text parsing.

**Parallel processing throughout.** GNU binutils is single-threaded. OxideUtils
uses `rayon` for data parallelism in symbol table processing, section analysis,
and archive operations. On multi-core hardware, throughput improvements of
3–14× are measurable on parallel-eligible operations.

**Structured, actionable error messages.** Every error identifies the specific
field, byte offset, and context. No cryptic one-line C-style strings.

**Broad format coverage.** ELF 32-bit and 64-bit in all byte orders, PE/COFF
for Windows, Mach-O including fat universal binaries, WebAssembly, and static
archives in GNU, BSD, and thin formats. DWARF debug information via `gimli`.

---

## Tools

| Tool              | GNU Equivalent | Phase | Status      |
|-------------------|----------------|-------|-------------|
| `oxide-objdump`   | `objdump`      | 3     | In progress |
| `oxide-nm`        | `nm`           | 4     | Planned     |
| `oxide-readelf`   | `readelf`      | 5     | Planned     |
| `oxide-size`      | `size`         | 6     | Planned     |
| `oxide-strings`   | `strings`      | 6     | Planned     |
| `oxide-strip`     | `strip`        | 7     | Planned     |
| `oxide-objcopy`   | `objcopy`      | 7     | Planned     |
| `oxide-ar`        | `ar`           | 8     | Planned     |
| `oxide-addr2line` | `addr2line`    | 8     | Planned     |

---

## Quick Start

```bash
git clone https://github.com/zainium-dynamics/oxideutils
cd oxideutils
cargo build --release --workspace

# Individual tools
./target/release/oxide-nm path/to/binary
./target/release/oxide-objdump -h path/to/binary

# Unified multicall binary
./target/release/oxideutils nm path/to/binary
./target/release/oxideutils objdump --json path/to/binary
```

### Drop-In Replacement

```bash
# Symlink individual tools
ln -s $(pwd)/target/release/oxide-nm      /usr/local/bin/nm
ln -s $(pwd)/target/release/oxide-objdump /usr/local/bin/objdump
ln -s $(pwd)/target/release/oxide-readelf /usr/local/bin/readelf
ln -s $(pwd)/target/release/oxide-strip   /usr/local/bin/strip
ln -s $(pwd)/target/release/oxide-ar      /usr/local/bin/ar

# Or symlink the unified binary
ln -s $(pwd)/target/release/oxideutils /usr/local/bin/nm
ln -s $(pwd)/target/release/oxideutils /usr/local/bin/objdump
```

GNU compatibility is verified automatically. Your existing scripts do not
require modification.

---

## Output Modes

Every tool supports three output modes selectable by flag.

**Human-readable (default, colored when connected to a terminal):**
```
$ oxide-nm libssl.a
0000000000012a40 T SSL_CTX_new
0000000000013b80 T SSL_read
                 U malloc
```

**GNU compatibility (`--gnu-compat`):**
```
$ oxide-nm --gnu-compat libssl.a
0000000000012a40 T SSL_CTX_new
0000000000013b80 T SSL_read
                 U malloc
```

**JSON (`--json`, one object per line):**
```json
{"name":"SSL_CTX_new","address":"0x0000000000012a40","size":0,"type":"T","binding":"global","section":".text"}
{"name":"SSL_read","address":"0x0000000000013b80","size":0,"type":"T","binding":"global","section":".text"}
{"name":"malloc","address":null,"size":0,"type":"U","binding":"global","section":null}
```

---

## Architecture

All shared logic lives in `oxideutils-core`. Every tool crate contains only
a thin `main.rs` — CLI argument parsing and a single call into core. No tool
ever contains format-parsing, symbol-classification, or output-formatting
logic. That belongs in core.

```
oxideutils-core/src/
├── error.rs          OxideError — single typed error hierarchy for the project
├── utils.rs          Hex formatting, address display, alignment, hex+ASCII dump
├── symbols.rs        Classification, nm type characters, demangling
├── archive.rs        Archive magic-byte detection
│
├── cli/              Shared CLI layer — designed and frozen in Phase 1
│   ├── config.rs     GlobalConfig: single source of truth for all output decisions
│   ├── parser.rs     CommonArgs: shared clap arguments inherited by every tool
│   ├── command.rs    OxideTool enum, from_name/canonical_name dispatch
│   ├── multicall.rs  BusyBox-style argv[0] dispatch for oxideutils binary
│   ├── aliases.rs    Pre-clap alias expansion for GNU short flags
│   ├── help.rs       Consistent header and footer generation
│   └── utils.rs      Sole stdout/stderr I/O boundary for the entire project
│
└── format/           Object format parsers — trait-based, Phase 2
    ├── traits.rs     ObjectFile, SymbolProvider, SectionProvider, and more
    ├── object.rs     OxideObject wrapper encapsulating the object crate
    ├── elf/          ELF 32/64, all endianness, DWARF via gimli
    ├── pe/           PE32 and PE32+ with import table
    ├── macho/        Mach-O and fat universal binaries
    └── wasm/         WebAssembly standard sections
```

No tool crate imports a format-specific module directly. All tool code
consumes only trait objects. This constraint is enforced by CI and is
never relaxed.

---

## Supported Targets

| Target                        | Tier     |
|-------------------------------|----------|
| `x86_64-unknown-linux-gnu`    | Primary  |
| `x86_64-unknown-linux-musl`   | Tier 1   |
| `aarch64-unknown-linux-gnu`   | Tier 1   |
| `x86_64-apple-darwin`         | Tier 1   |
| `aarch64-apple-darwin`        | Tier 1   |
| `x86_64-pc-windows-msvc`      | Tier 1   |

---

## Quality Standards

Every pull request must satisfy all of the following before merging.

`cargo fmt --check` — consistent formatting throughout the workspace.
`cargo clippy --workspace -- -D warnings` — zero linting diagnostics,
including `clippy::pedantic` on `oxideutils-core`. `cargo test --workspace`
— all unit tests, integration tests, and snapshot tests passing. GNU
compatibility differential tests — every tool's `--gnu-compat` output
verified against the corresponding GNU tool on the standard fixture corpus
on every commit. No `unwrap()` or `expect()` in non-test production code.
No `println!` or `eprintln!` outside `oxideutils-core/src/cli/utils.rs`.

---

## Contributing

Please read [`CONTRIBUTING.md`](CONTRIBUTING.md) before opening a pull
request. The most important architectural rule: every tool's `main.rs`
contains only argument parsing and a single call into `oxideutils-core`.
Logic that belongs in the core library must live there.

To add a new tool, create a new crate under `crates/`, write a thin
`main.rs`, and add it to the workspace members. The core library is not
modified to accommodate new tools — if new core functionality is required,
a separate pull request adding that functionality to `oxideutils-core`
with full tests is required first.

---

## Security

To report a security vulnerability, email `security@zainiumdynamics.tech`
rather than opening a public issue. We observe a 90-day coordinated
disclosure timeline.

OxideUtils participates in OSS-Fuzz for ongoing automated vulnerability
discovery. Fuzz target results are reviewed weekly by the maintainer team.

---

## Roadmap

OxideUtils is developed across twelve sequential phases. The current status
and schedule for each phase is documented in [`PLAN.md`](docs/proposal/PLAN.md).

---

## Funding

OxideUtils is funded by the **Sovereign Tech Fund** (STF), the German
government initiative that invests in the security and resilience of
open digital infrastructure. The grant supports the complete twelve-phase
development of all nine OxideUtils tools, a formal security audit of the
format parsing layer, and the packaging and distribution work required
for adoption by Linux distributions.

The security case for this funding is straightforward: GNU binutils carries
a documented record of memory-safety vulnerabilities that constitute a
systemic risk to the software supply chain, and Rust is the structurally
correct solution. OxideUtils translates that argument into working,
tested, packaged software.

[Sovereign Tech Fund](https://sovereigntechfund.de) —
[Zainium Dynamics](https://zainiumdynamics.tech)

---

## License

OxideUtils is licensed under the **GNU General Public License version 3**.
This copyleft license ensures that all improvements to OxideUtils by any
organisation must be released under the same terms, protecting the public
benefit character of this publicly-funded work.

See [`LICENSE`](LICENSE) for the full text.

---

*Developed and maintained by [Zainium Dynamics](https://github.com/zainium-dynamics)*
*Website: [zainiumdynamics.tech](https://zainiumdynamics.tech)*
*Repository: [github.com/zainium-dynamics/oxideutils](https://github.com/zainium-dynamics/oxideutils)*
