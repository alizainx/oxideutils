# OxideUtils — Master Development Plan

## Overview

OxideUtils is a production-grade, memory-safe rewrite of GNU binutils in pure Rust. It is not a hobby project or a learning exercise. It is designed to be a real replacement for GNU tools in build systems, CI pipelines, and developer workflows. Every architectural decision in this plan was made after studying comparable projects; the reasoning is documented in full in CASE_STUDY.md.

The development is organized into three sequential phases. Each phase has a single clear goal, a defined set of deliverables, and hard exit criteria that must be satisfied before the next phase begins. The phases are not negotiable timelines — they are quality gates.

**The core principle: add tools without ever touching the core.** After Phase 2, `oxideutils-core` is functionally frozen. Adding `oxide-nm` in week 3 of Phase 3 means creating a new crate, writing a thin `main.rs`, and importing from core. It does not mean modifying `oxideutils-core`. If a tool needs something from core that does not exist, development stops, the missing functionality is added to core as a standalone change with tests, and then the tool is written. This invariant is non-negotiable.

---

## Final Project Structure

This structure is locked. Creating a file that does not appear here requires a documented justification.

```
oxideutils/
├── Cargo.toml                        # workspace manifest with [workspace.dependencies]
├── Cargo.lock
├── README.md
├── LICENSE                           # MIT OR Apache-2.0 dual
├── CONTRIBUTING.md
├── CODE_OF_CONDUCT.md
├── CHANGELOG.md
├── PLAN.md                           # this file
├── CASE_STUDY.md
├── .gitignore
│
├── .github/
│   ├── workflows/
│   │   ├── ci.yml                    # fmt, clippy, test, cross-compile
│   │   ├── release.yml
│   │   └── bench.yml
│   └── ISSUE_TEMPLATE/
│       ├── bug_report.md
│       └── feature_request.md
│
├── docs/
│   ├── architecture.md
│   ├── gnu-compatibility.md
│   └── man/
│
├── crates/
│   ├── oxideutils-core/              # ← THE ENTIRE FOUNDATION. Built in Phase 1 and 2.
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── lib.rs                # public re-exports only
│   │       ├── error.rs              # OxideError (thiserror)
│   │       ├── utils.rs              # hex, alignment, byte display helpers
│   │       ├── symbols.rs            # symbol type classification, demangling
│   │       ├── archive.rs            # archive parsing helpers
│   │       │
│   │       ├── cli/                  # PHASE 1 — complete CLI infrastructure
│   │       │   ├── mod.rs
│   │       │   ├── parser.rs         # common clap setup
│   │       │   ├── command.rs        # tool enum and dispatch
│   │       │   ├── aliases.rs        # short-form aliases, GNU compat mapping
│   │       │   ├── multicall.rs      # BusyBox-style argv[0] dispatch
│   │       │   ├── config.rs         # GlobalConfig: color, json, quiet, verbose
│   │       │   ├── help.rs           # shared help formatting
│   │       │   └── utils.rs          # output helpers, terminal detection
│   │       │
│   │       └── format/               # PHASE 2 — complete format parsing layer
│   │           ├── mod.rs
│   │           ├── object.rs         # OxideObject wrapper over `object` crate
│   │           ├── traits.rs         # ObjectFile, SymbolProvider, SectionProvider, etc.
│   │           ├── utils.rs          # format-level utilities
│   │           ├── archive.rs        # .a archive member iteration
│   │           ├── coff.rs           # COFF header and symbol table
│   │           ├── elf/
│   │           │   ├── mod.rs
│   │           │   ├── header.rs
│   │           │   ├── section.rs
│   │           │   ├── segment.rs
│   │           │   ├── symbol.rs
│   │           │   ├── relocation.rs
│   │           │   ├── dynamic.rs
│   │           │   ├── note.rs
│   │           │   └── dwarf.rs
│   │           ├── pe/
│   │           │   ├── mod.rs
│   │           │   ├── header.rs
│   │           │   ├── section.rs
│   │           │   ├── symbol.rs
│   │           │   ├── relocation.rs
│   │           │   └── import.rs
│   │           ├── macho/
│   │           │   ├── mod.rs
│   │           │   ├── header.rs
│   │           │   ├── section.rs
│   │           │   ├── symbol.rs
│   │           │   ├── relocation.rs
│   │           │   ├── fat.rs
│   │           │   └── dyld.rs
│   │           └── wasm/
│   │               ├── mod.rs
│   │               ├── header.rs
│   │               ├── section.rs
│   │               └── custom.rs
│   │
│   ├── oxide-objdump/                # skeleton in Phase 1, full impl in Phase 3 week 1–2
│   │   ├── Cargo.toml
│   │   └── src/main.rs
│   ├── oxide-nm/                     # Phase 3 week 3
│   ├── oxide-readelf/                # Phase 3 week 4
│   ├── oxide-size/                   # Phase 3 week 5
│   ├── oxide-strings/                # Phase 3 week 6
│   ├── oxide-strip/                  # Phase 3 week 7
│   ├── oxide-objcopy/                # Phase 3 week 8
│   ├── oxide-ar/                     # Phase 3 week 9
│   └── oxide-addr2line/              # Phase 3 week 10
│
├── bin/
│   └── oxideutils.rs                 # unified multicall binary
│
├── tests/
│   └── integration/
│       ├── gnu-compat/               # diffs against GNU output
│       ├── fixtures/                 # real binary test inputs
│       └── snapshots/                # insta snapshot files
│
└── benches/
    └── comparison.rs                 # criterion benchmarks vs GNU
```

---

## Philosophy

These principles govern every decision made during development. They are ordered by priority.

**Memory safety first.** There is no acceptable use of `unsafe` without a documented justification and a corresponding test that exercises the unsafe block. The bar for justifying `unsafe` is high: it must be provably impossible to achieve the same result in safe Rust, and the gain must be material. In practice, this means `unsafe` appears zero or near-zero times in this codebase.

**Thin binaries, fat core.** Every tool's `main.rs` contains exactly two things: CLI argument parsing and a call into `oxideutils-core`. All logic — parsing, formatting, analysis, symbol resolution — lives in the core library. The test surface is in the core. The documentation is in the core.

**Correctness before performance.** We match GNU output exactly in `--gnu-compat` mode before we optimize anything. A tool that is fast but wrong is worse than no tool. The benchmark workflow (`bench.yml`) is run only after the compatibility test suite passes.

**Interfaces before implementations.** `format/traits.rs` is written and reviewed before any parser is written. No parser is merged until it satisfies the trait interface. This is the lesson from the lld case study: format-specific code that escapes its abstraction boundary creates permanent technical debt.

**GNU compat by default, modern UX by choice.** Every output tool supports both modes. In default mode, output is colored and structured for human readability. With `--gnu-compat`, output matches GNU character-for-character for use in scripts and existing tooling. With `--json`, output is machine-readable newline-delimited JSON.

---

## Phase 1 — Core CLI Infrastructure and Workspace Foundation

**Goal:** A complete, tested, documented `oxideutils-core` CLI layer and a workspace that compiles cleanly with a skeleton `oxide-objdump` tool.

**Duration:** 2–3 weeks.

**Why this comes first:** Every tool built in Phase 3 depends on the CLI layer being correct, consistent, and complete. If `cli/config.rs` is added or changed after tools are written, every tool must be updated. Building it first, completely, means it never needs to be revisited.

### Deliverables

**Root `Cargo.toml`** — workspace manifest with `[workspace.package]` (edition = "2021", version = "0.1.0", license = "MIT OR Apache-2.0"), `[workspace.dependencies]` declaring all shared dependencies with pinned versions, and `[workspace.members]` listing all current and planned crates. resolver = "2" is mandatory.

**`oxideutils-core/Cargo.toml`** — the core library manifest. Declares workspace dependencies. No tool-specific dependencies enter this file.

**`error.rs`** — `OxideError` enum using `thiserror`. Variants cover: `Io(std::io::Error)`, `Parse(String)`, `UnsupportedFormat(String)`, `UnsupportedArchitecture(String)`, `InvalidArgument(String)`, `Archive(String)`, `Dwarf(String)`. All variants implement `std::error::Error` via the derive. The `Result<T>` alias is defined here as `pub type Result<T> = std::result::Result<T, OxideError>`.

**`utils.rs`** — shared utility functions: hex formatting with configurable prefix and width, byte-to-human-readable size formatting, alignment calculation, address display (with and without symbol offset annotation). All functions are pure, all have unit tests, all have doc comments.

**`symbols.rs`** — symbol classification utilities: determining symbol type from ELF `st_type`, symbol binding classification, C++ demangling via the `rustc-demangle` crate (with a fallback that returns the mangled name on failure), symbol visibility classification. These are used by `oxide-nm`, `oxide-objdump`, and `oxide-readelf`.

**`archive.rs`** — archive-level utilities: detecting whether a byte slice is an archive, extracting member names, reading the symbol index. This is distinct from `format/archive.rs`, which provides the full parsed abstraction; this file provides the low-level detection logic.

**`cli/config.rs`** — `GlobalConfig` struct with fields: `color: ColorMode` (enum: Auto, Always, Never), `output_format: OutputFormat` (enum: Human, Json, GnuCompat), `quiet: bool`, `verbose: bool`. Constructor `GlobalConfig::from_env()` detects terminal attachment and sets color accordingly. `OutputFormat::is_gnu_compat()` and `OutputFormat::is_json()` convenience methods. This struct is passed by reference into every core function that produces output.

**`cli/parser.rs`** — common clap setup: the `--color`, `--json`, `--gnu-compat`, `--quiet`, `--verbose` arguments that appear on every tool. A `CommonArgs` struct derived from clap that every tool's own `Args` struct embeds via `#[clap(flatten)]`.

**`cli/command.rs`** — `OxideTool` enum with a variant for each tool. A `dispatch` function that takes `OxideTool` and `&[OsString]` and calls the appropriate tool entrypoint. Used by the multicall binary.

**`cli/multicall.rs`** — reads `argv[0]`, strips path prefix, maps the binary name to an `OxideTool` variant, and calls `dispatch`. Handles both `oxide-nm` and `nm` style names (the latter when symlinked).

**`cli/aliases.rs`** — short-form argument aliases (e.g., `-D` for `--demangle`, `-C` for `--no-demangle`) and GNU-to-modern flag mapping. Centralized here so tool-specific arg parsers stay clean.

**`cli/help.rs`** — shared help text formatting utilities: header with tool name and version, common flags section, exit codes section. Ensures consistent help output appearance across all tools.

**`cli/utils.rs`** — output writing utilities: `write_human`, `write_json_line`, `write_error`. These are the only functions that should ever call `println!`, `eprintln!`, or write to stdout/stderr in the entire codebase. All core functions return data; these functions emit it.

**`lib.rs`** — public re-exports. Everything the tool crates need is re-exported here. No implementation code lives in `lib.rs`.

**`oxide-objdump/Cargo.toml`** and **`oxide-objdump/src/main.rs`** — the skeleton tool. `main.rs` is approximately 30 lines: parse arguments with clap, construct `GlobalConfig`, call `oxideutils_core::objdump::run(&args, &config)` (which returns a stub "not yet implemented" message for now), handle the returned `OxideError` with a clean exit. This verifies the entire workspace compiles and the CLI layer works end-to-end.

**`.github/workflows/ci.yml`** — runs on every push and pull request: `cargo fmt --check`, `cargo clippy --workspace -- -D warnings`, `cargo test --workspace`, and cross-compilation checks for `x86_64-unknown-linux-musl`, `x86_64-pc-windows-gnu`, and `aarch64-apple-darwin`.

**`.gitignore`**, **`README.md`** (see Appendix A), **`LICENSE`**, **`CONTRIBUTING.md`**, **`CODE_OF_CONDUCT.md`**, **`CHANGELOG.md`**.

### Exit Criteria

All of the following must pass before Phase 2 begins:

- `cargo build --workspace` completes without errors or warnings.
- `cargo clippy --workspace -- -D warnings` produces zero diagnostics.
- `cargo test --workspace` passes all tests. The test count for `oxideutils-core` must be at least 40 unit tests at this point, covering error formatting, config construction, utility functions, CLI argument parsing, and symbol classification.
- `oxide-objdump --help` renders correctly and lists all common flags.
- `oxide-objdump --version` outputs a correctly formatted version string.
- Code review confirms that no implementation logic exists outside of `oxideutils-core`.

---

## Phase 2 — Format Parsing Layer

**Goal:** A complete, tested, fuzzed `format/` module in `oxideutils-core` that can parse any ELF, PE, Mach-O, WASM, or archive binary and expose its metadata through the trait interface.

**Duration:** 3–4 weeks.

**Why this is a separate phase:** The format parsers are the most technically complex component of the entire project. They must be correct before tools are built against them. A bug in the ELF parser discovered after five tools are built against it affects all five tools simultaneously. Discovering it during Phase 2, when no tools depend on it yet, means fixing it in one place.

### Deliverables

**`format/traits.rs`** — this is the most important file in Phase 2. It defines the interfaces that every parser implements and that every tool consumes. Traits include:

- `ObjectFile`: provides format detection, architecture, endianness, bitwidth, entry point, and access to the other providers.
- `SectionProvider`: iterates sections, looks up sections by name or index, returns `SectionInfo` structs (name, type, flags, address, offset, size, alignment, data).
- `SymbolProvider`: iterates symbols, looks up symbols by name or value, returns `SymbolInfo` structs (name, demangled name, type, binding, visibility, section index, value, size).
- `RelocationProvider`: iterates relocations per section, returns `RelocationInfo` structs (offset, type, symbol, addend).
- `DynamicProvider`: iterates dynamic section entries, returns `DynamicEntry` structs (tag, value or string).
- `NoteProvider`: iterates note sections, returns `NoteInfo` structs (name, type, description bytes).
- `DebugProvider`: provides DWARF line information lookup by address, returning `DebugLocation` structs (file, line, column, function name).

All trait methods return `crate::error::Result<_>`. No trait method panics. No trait method is allowed to return a format-specific type; all return types are defined in `format/traits.rs` or `format/mod.rs`.

**`format/object.rs`** — `OxideObject` struct. Holds a `Box<dyn ObjectFile>` internally. Constructor `OxideObject::from_bytes(data: &[u8]) -> Result<Self>` uses the `object` crate for format detection, then constructs the appropriate parser. The `object` crate is used for detection and for populating the generic data structures; format-specific parsing uses the dedicated submodules. No tool ever imports from the `object` crate directly; they always go through `OxideObject`.

**`format/elf/`** — complete ELF parser. `header.rs` handles both ELF32 and ELF64, little-endian and big-endian. `section.rs` parses the section header table and all standard section types (`SHT_PROGBITS`, `SHT_SYMTAB`, `SHT_DYNSYM`, `SHT_STRTAB`, `SHT_RELA`, `SHT_REL`, `SHT_DYNAMIC`, `SHT_NOTE`, `SHT_GNU_HASH`, `SHT_GNU_VERSYM`, `SHT_GNU_VERNEED`, `SHT_GNU_VERDEF`). `symbol.rs` implements `SymbolProvider` including GNU symbol version annotations. `relocation.rs` handles REL and RELA relocation tables for x86-64, ARM, AArch64, RISC-V, and MIPS. `dynamic.rs` parses the `.dynamic` section including `DT_NEEDED` library names. `note.rs` parses `.note.ABI-tag`, `.note.gnu.build-id`, and `.note.gnu.property`. `dwarf.rs` uses the `gimli` crate to provide line information lookup and function name resolution.

**`format/pe/`** — PE/COFF parser for Windows binaries. Covers PE32 and PE32+ (64-bit). `import.rs` parses the import directory table and produces `ImportEntry` structs listing DLL name and imported function names.

**`format/macho/`** — Mach-O parser. `fat.rs` handles Universal Binary (fat) containers, returning an iterator over architectures and their offsets. `dyld.rs` parses the dyld info opcodes for rebase and bind information.

**`format/wasm/`** — WebAssembly binary parser. Covers the standard section types: type, import, function, table, memory, global, export, element, code, data. `custom.rs` handles known custom sections including the name section (for function names) and the linking section.

**`format/archive.rs`** — complete `.a` archive implementation: GNU/SysV format, BSD format, and thin archives. Member iteration with name resolution (using the long names table). Symbol index reading (both GNU and BSD formats). The `ArchiveMember` struct holds the member name, header data, and a reference to the member bytes.

**`format/coff.rs`** — standalone COFF parser for Windows object files (`.obj`). Used internally by the PE parser and directly when a COFF object file is opened without a PE wrapper.

**`format/utils.rs`** — format-level utilities: reading null-terminated strings from byte slices, reading LEB128 encoded integers (for WASM), reading ULEB128, address alignment helpers, machine type to human-readable string conversion.

**Unit tests** — every parser module has tests in a `#[cfg(test)]` block. Tests parse real binary fixtures checked into `tests/integration/fixtures/`. The fixture set must include at minimum: a Linux ELF64 shared library, a Linux ELF32 static executable, a Windows PE32+ DLL, a macOS Mach-O universal binary, a WASM module, and a `.a` static library.

**Fuzz targets** — fuzz targets using `cargo-fuzz` (libfuzzer) for each major parser entry point: `fuzz_elf`, `fuzz_pe`, `fuzz_macho`, `fuzz_wasm`, `fuzz_archive`. These are in `fuzz/fuzz_targets/` and are run in CI with a corpus derived from the fixture files.

### Exit Criteria

All of the following must pass before Phase 3 begins:

- `cargo test --workspace` passes, and the test count for `oxideutils-core` is at least 150 tests.
- The fixture test suite parses every fixture file without returning an error.
- No test file in `format/` calls `unwrap()` or `expect()` in production code paths.
- Fuzz targets have run for a minimum of 30 minutes per target without finding a panic or abort.
- A manual review of `format/traits.rs` confirms that no format-specific types are visible through any trait method signature.
- `oxide-objdump` can be called on a real ELF binary and produce a non-empty section listing (even if the output format is preliminary).

---

## Phase 3 — Tool Implementation: Weekly Schedule

**Goal:** Implement each tool one per week (or two weeks for complex tools), releasing weekly, with the core never modified.

**Duration:** Ongoing, starting immediately after Phase 2 exit criteria are met.

**The invariant:** Adding a tool is adding a crate. It requires creating `crates/oxide-{tool}/Cargo.toml` and `crates/oxide-{tool}/src/main.rs`. If the tool needs something from core that does not yet exist, development stops and a core-only PR is opened first. The tool PR follows only after the core PR is merged and CI passes.

Each tool is delivered with: full GNU flag compatibility in `--gnu-compat` mode verified by diff tests, `--json` output, colored human-readable output in default mode, a snapshot test suite using `insta`, and a man page in `docs/man/`.

### Week 1–2 — `oxide-objdump` (full implementation)

`oxide-objdump` is the most complex tool because it covers the widest range of functionality. Full implementation includes: section header display (`-h`), full contents dump in hex+ASCII (`-s`), disassembly of code sections (`-d`, `-D`) using the `capstone` crate, relocation entries (`-r`), dynamic symbol table (`--dynamic`), DWARF line information (`--dwarf=line`), and all-together output mode (`-x`). GNU compat mode produces output byte-for-byte identical to `objdump` on the same binary.

### Week 3 — `oxide-nm`

Symbol listing with all standard flags: `-a` (all symbols including debug), `-D` (dynamic symbols), `-g` (external symbols only), `-u` (undefined symbols), `-C` (demangle), `--size-sort`, `--radix`, `--format` (bsd/sysv/posix). Output columns: value, type character, name. GNU compat mode matches `nm` output exactly.

### Week 4 — `oxide-readelf`

ELF-specific inspection: `-h` (ELF header), `-l` (program headers), `-S` (section headers), `-s` (symbol table), `-r` (relocations), `-d` (dynamic section), `-n` (notes), `-V` (version information), `--wide` (no line truncation). This tool is ELF-only by design, matching GNU `readelf` semantics.

### Week 5 — `oxide-size`

Section size reporting. Berkeley format (default): text, data, bss, decimal total, hex total. SysV format (`-A`): section-by-section listing. Common mode (`--common`): includes common symbols in BSS count. Supports multiple input files with per-file and total lines. Format detection: silently handles non-ELF inputs with appropriate error.

### Week 6 — `oxide-strings`

Printable string extraction. Options: `-n` (minimum length, default 4), `-t` (radix for offset display: d/o/x), `-e` (encoding: s=single-byte, S=7-bit, b/l=16-bit big/little, B/L=32-bit big/little), `-a` (scan entire file, not just initialized sections). Default output format: offset and string. GNU compat matches `strings` output exactly.

### Week 7 — `oxide-strip`

Binary stripping: remove debug symbols (`--strip-debug`), remove all local symbols (`--strip-unneeded`), remove all symbols (`-s`). Section removal by name (`--remove-section`). Preserves sections required for correct binary execution. This is a write tool — it modifies binary content — and therefore has the most extensive testing of any tool: every strip operation is followed by reading the output with `oxide-readelf` and verifying the expected sections are absent and the binary is still valid.

### Week 8 — `oxide-objcopy`

Binary transformation: format conversion (`--input-target`, `--output-target`), section addition (`--add-section`), section removal (`--remove-section`), section renaming (`--rename-section`), symbol stripping options. This is the second write tool and carries the same testing discipline as `oxide-strip`.

### Week 9 — `oxide-ar`

Archive operations: create (`-c`), append (`-q`), delete (`-d`), extract (`-x`), list (`-t`), print member (`-p`), update (`-u`), create symbol index (`-s`). Verbose mode lists members with permissions, ownership, size, and timestamp. GNU compat matches `ar` output format exactly.

### Week 10+ — `oxide-addr2line` and future tools

Address-to-source-location translation using DWARF debug information. Options: `-e` (input file), `-f` (output function names), `-i` (inline function information), `-C` (demangle), `-s` (strip directory components from file names). Subsequent tools are added following the same weekly pattern.

---

## Dependency Manifest

All dependencies are declared in the root `[workspace.dependencies]` table. Tool crates inherit from this table; they do not specify their own versions.

| Crate | Purpose | Used In |
|---|---|---|
| `object` | Binary format parsing | `oxideutils-core/format/` |
| `gimli` | DWARF debug info | `oxideutils-core/format/elf/dwarf.rs` |
| `capstone` | Disassembly | `oxide-objdump` only |
| `clap` (derive) | CLI argument parsing | All tool crates, `oxideutils-core/cli/` |
| `thiserror` | Error derive | `oxideutils-core/error.rs` |
| `anyhow` | Error propagation in tool `main.rs` | Tool crates only |
| `rayon` | Parallel iteration | `oxideutils-core/format/`, heavy-lifting tools |
| `tracing` | Structured logging | `oxideutils-core` |
| `tracing-subscriber` | Log output | Tool crates |
| `colored` | Terminal color | `oxideutils-core/cli/utils.rs` |
| `serde` + `serde_json` | JSON serialization | `oxideutils-core/cli/utils.rs` |
| `rustc-demangle` | C++ / Rust demangling | `oxideutils-core/symbols.rs` |
| `insta` | Snapshot testing | All `#[cfg(test)]` |
| `criterion` | Benchmarking | `benches/` |
| `cargo-fuzz` | Fuzz testing | `fuzz/` |

---

## Quality Standards

These standards apply to every line of code in every crate. They are enforced by CI; a PR that fails any of them is not merged.

**No `unwrap()` or `expect()` in production code.** `unwrap()` is permitted in test code only, and only where the test is specifically verifying a condition that must be true for the test to be meaningful. Every other call site uses `?` or maps errors to `OxideError`.

**No `println!` or `eprintln!` in `oxideutils-core`.** All output goes through `cli/utils.rs`. This is enforced by a clippy lint configured in `.cargo/config.toml`.

**Every public function has a doc comment.** This includes functions, structs, enums, and their variants. The doc comment must include at least a one-sentence description and, for non-trivial functions, an example or a note about error conditions.

**Every module has a module-level doc comment.** The first line of every `mod.rs`, `lib.rs`, and single-file module begins with `//!` documentation.

**Clippy pedantic is enabled for `oxideutils-core`.** The core library's `Cargo.toml` includes `#![deny(clippy::pedantic)]` in `lib.rs`. Tool crates use `#![warn(clippy::pedantic)]`.

**Snapshot tests for all output.** Every tool function that produces user-visible output has a corresponding snapshot test using `insta`. The snapshots are committed to the repository and reviewed in PRs.

**GNU compat tests run in CI.** The `tests/integration/gnu-compat/` suite is run against the fixture binaries and diffs tool output against recorded GNU output. Any divergence in `--gnu-compat` mode fails CI.

---

## Appendix A — README.md Content

```markdown
# OxideUtils

Memory-safe, high-performance rewrite of GNU binutils in pure Rust.

Drop-in compatible. Better defaults. Production quality.

## Tools

oxide-objdump · oxide-nm · oxide-readelf · oxide-size · oxide-strings
oxide-strip · oxide-objcopy · oxide-ar · oxide-addr2line

## Quick Start

    git clone https://github.com/your-org/oxideutils
    cd oxideutils
    cargo build --release --workspace

    ./target/release/oxide-nm path/to/binary
    ./target/release/oxide-objdump --json path/to/binary

## GNU Compatibility

Every output tool supports --gnu-compat for byte-identical output with GNU equivalents.
Use as drop-in replacements by symlinking: ln -s oxide-nm /usr/local/bin/nm

## Supported Formats

ELF (32/64-bit, LE/BE) — PE/COFF (Windows) — Mach-O (macOS, fat) — WebAssembly — Archive (.a)

## Development

See PLAN.md for the three-phase development roadmap.
See CASE_STUDY.md for the architectural decisions and their justifications.
See CONTRIBUTING.md before submitting a pull request.

## License

MIT OR Apache-2.0
```

---

## Appendix B — AI Session Prompt

Paste this at the start of every development session to enforce context:

```
You are a world-class Rust systems engineer building OxideUtils:
a memory-safe, production-grade rewrite of GNU binutils.

Project layout: oxideutils/ workspace monorepo.
Core library: crates/oxideutils-core/ (Phase 1 = cli/, Phase 2 = format/)
Tools: crates/oxide-{name}/src/main.rs (thin wrapper only)

Strict rules — never violate:
1. main.rs = argument parsing + one call into core + error handling. Nothing else.
2. All logic lives in oxideutils-core. No exceptions.
3. No unwrap() or expect() in library code. Use ? and OxideError.
4. No println!/eprintln! in oxideutils-core. Use cli/utils.rs functions.
5. No format-specific types in trait method signatures (format/traits.rs).
6. Every public fn has a doc comment. Every module has a module-level doc comment.
7. cargo clippy --workspace -- -D warnings must pass before any commit.
8. New tools never modify oxideutils-core — they import it only.
9. Priority: Correctness → GNU Compatibility → Performance → UX.
10. All output tools must support --gnu-compat and --json.

Current phase: [PHASE]
Current task: [TASK]
Current file: [FILE_PATH]

Implement with production-grade quality. No shortcuts.
```

---

*This plan is version 1.0 and is considered final for the purposes of Phase 1 and Phase 2. Phase 3 tool ordering may be adjusted based on user demand, but the architectural constraints are permanent.*
