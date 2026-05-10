# OxideUtils — Case Study

## Purpose

Before writing a single line of OxideUtils code, this case study was conducted to understand what the existing ecosystem has built, where each project succeeded, and critically, where each one created technical debt, inconsistency, or architectural regret. Every lesson recorded here is directly reflected in a concrete decision in the OxideUtils design.

This is not an academic survey. It is a pre-mortems-and-post-mortems document. We read the source, the issues, the changelogs, and the contributor complaints. We extracted what matters for building a real production-quality binutils replacement.

---

## Project 1 — uutils/coreutils

### What It Is

The most ambitious Rust rewrite project currently in existence: a full reimplementation of GNU coreutils (ls, cp, mv, cat, sort, etc.) in pure Rust, structured as a Cargo workspace monorepo with a multicall binary. Active since 2013, it has hundreds of contributors and ships as a component in some Linux distributions.

### What They Got Right

The workspace structure is sound. Each tool lives in its own crate. The multicall binary works. The project proved that a large-scale Rust rewrite of a GNU toolkit is achievable and maintainable long-term. The use of `clap` for argument parsing across all tools established a consistent baseline, and the CI pipeline is comprehensive, testing against real GNU test suites on multiple platforms.

### Where It Went Wrong

The CLI layer is the biggest pain point. Because there was no shared CLI abstraction layer built at the start, each tool author implemented argument handling independently. Over time this created inconsistency: some tools support `--color=auto`, others do not; some handle `--quiet` and `--verbose` differently; some output to stderr on error, others to stdout. There is no unified config struct that propagates global settings like verbosity or output format. Fixing this now would require touching every tool crate simultaneously, which is why the inconsistency persists years into the project.

The second issue is that the project has no shared output abstraction. Each tool handles its own formatting. Adding JSON output to a tool, for example, requires modifying that tool's internals, not changing a centralized output layer.

### Lesson Applied to OxideUtils

`cli/config.rs` in `oxideutils-core` is the single source of truth for all global settings: color mode, JSON output, quiet mode, GNU compat mode, verbosity level. This struct is constructed once in `main.rs` and passed into every core function. No tool author can diverge from this. The `cli/` module is built and frozen in Phase 1, before a single tool implementation is written.

---

## Project 2 — ripgrep (BurntSushi)

### What It Is

A line-oriented search tool written in Rust that is faster than GNU grep on nearly all workloads. Widely regarded as the gold standard for a Rust CLI tool that gets UX right. Not a GNU rewrite in the strict sense, but the most instructive example of how to do output, color, JSON, and error handling in a Rust command-line tool.

### What They Got Right

Nearly everything on the UX side. Color output is on by default when stdout is a terminal and off when piped — detected automatically, with no flag required from the user. JSON output mode is a first-class feature, not an afterthought; the `--json` flag produces newline-delimited JSON that downstream tools can parse reliably. Error messages are structured, specific, and actionable. Performance profiling was done from the beginning, not bolted on at the end.

The internal architecture separates concerns cleanly: the search engine core has no awareness of output format. The printer layer receives results and formats them however the config dictates. This is the correct model.

### Where It Is Not Directly Applicable

ripgrep is a single tool. The modularity challenges of a multi-tool workspace — shared dependencies, consistent behavior across crates, multicall binary dispatch — were never faced. Its lessons are primarily about output quality and UX defaults, not about workspace architecture.

### Lesson Applied to OxideUtils

The output abstraction in `oxideutils-core` follows ripgrep's printer model. The core functions return data structures; a separate formatting layer converts them to either human-readable colored text or JSON, depending on the config. No core function ever calls `println!` directly. The `--json` flag is planned from day one for every output tool, not added later.

Auto-detection of color: if stdout is a terminal, color is on. If piped, it is off. This is implemented in `cli/config.rs` via the `colored` crate's `control::set_override`, not with a manual `--color` flag required from the user.

---

## Project 3 — fd (sharkdp)

### What It Is

A modern replacement for `find`, written in Rust. Simpler syntax, sensible defaults (ignores `.git` and hidden files), color output by default, and parallelism for speed. One of the cleanest single-tool Rust CLI projects in the ecosystem.

### What They Got Right

Dependency discipline. fd's `Cargo.toml` is carefully curated. Every dependency is justified and feature-flagged where possible. The result is a fast compile time and a small binary. The project demonstrates that "batteries included" defaults (ignore hidden files, colorize output, use regex by default) can dramatically improve UX without sacrificing compatibility, because the compatibility escape hatches (show hidden files, plain string, classic find syntax) are always available.

### Where It Is Not Directly Applicable

Single tool, no shared core. No format parsing. The lessons are about dependency hygiene and default behavior design, not about multi-crate architecture.

### Lesson Applied to OxideUtils

The root `Cargo.toml` uses `[workspace.dependencies]` with `optional` and feature flags where applicable. Every dependency added to `oxideutils-core` is explicitly justified in a code comment. No tool crate pulls in a dependency that the core does not need — tool-specific dependencies go in that tool's `Cargo.toml` only.

Default behavior follows fd's philosophy: useful by default, compatible when asked. `oxide-nm` shows demangled symbols by default. `oxide-objdump` colorizes output by default. `--gnu-compat` switches to GNU behavior everywhere.

---

## Project 4 — mold linker (Rui Ueyama)

### What It Is

The fastest linker currently available for Linux, written in C++ by the original author of lld. It achieves 5–10× faster link times than GNU ld and 2–3× faster than lld by parallelizing every phase of linking. It is the best available example of how to design a systems tool from a performance-first, architecture-first mindset.

### What They Got Right

The design was frozen before implementation. Rui defined the full data model — input sections, output sections, symbol tables, relocation entries — as C++ structs before writing a single line of parsing code. This meant that when the ELF parser was written, it had a clear destination: populate these structs. When the layout engine was written, it operated on those same structs. No phase of the tool needed to know the format specifics of another phase.

Parallelism was designed in from the beginning, not added later. Every phase that could be parallelized was identified upfront and given appropriate synchronization primitives. The result is that mold's parallelism is deep and correct, not the shallow "just add rayon" approach that produces false speedups.

### Where It Went Wrong (lld comparison)

lld started without a clean abstraction between ELF, COFF, and Mach-O support. Each format got its own directory as it was added, but the abstractions between them leaked. There is duplicated logic across `ELF/`, `COFF/`, and `MachO/` directories in lld because the core interfaces were not designed to span all formats from the start. Adding WebAssembly support required significant refactoring.

### Lesson Applied to OxideUtils

Phase 2 is dedicated entirely to `format/traits.rs` and the parser implementations. The traits — `ObjectFile`, `SymbolProvider`, `SectionProvider`, `RelocationProvider`, `Disassembler` — are designed and stabilized before any tool implementation begins. Every format parser implements these traits. No tool ever reaches into a format-specific module directly; it always goes through the trait interface.

This is the mold lesson applied to Rust: define the data model and the interfaces first. The parsers are the implementation detail. The tools are consumers of the abstraction.

---

## Project 5 — object crate + gimli (bytecodealliance / gimli-rs)

### What It Is

`object` is a Rust library for reading and writing object file formats (ELF, PE, Mach-O, WASM, XCOFF, archive files). `gimli` is a library for reading and writing DWARF debug information. Together they form the foundation that the Rust compiler's own debugging infrastructure is built on. Both are zero-copy, production-grade, and extensively fuzzed.

### What They Got Right

Zero-copy design. The `object` crate returns views into the original byte slice wherever possible, avoiding allocations. This is critical for performance when processing large binaries. The API is well-typed — section types, symbol kinds, relocation encodings are all enums, not raw integers. Error handling is comprehensive.

Both crates are used in anger by the Rust compiler toolchain, which means their correctness is tested against every Rust binary ever compiled. The fuzz corpus is extensive.

### What They Do Not Provide

They are libraries, not tools. They provide no CLI, no output formatting, no error messages suitable for end users, no progress reporting, no color. Using them directly in a tool produces a usable but rough experience. The error types are designed for library consumers, not end users.

The `object` crate's API also exposes format-specific types in some paths, which means tool code that uses it directly can accumulate format-specific branches over time.

### Lesson Applied to OxideUtils

We do not use the `object` crate directly in tool code. `format/object.rs` defines `OxideObject`, a wrapper that takes a `&[u8]` byte slice, uses `object` crate internally for format detection and basic parsing, and exposes only the `ObjectFile` trait to the rest of the codebase. All format-specific logic is inside the `format/` module. All error messages that reach the user go through `OxideError`, which has human-readable descriptions.

This also means that if we ever need to replace or augment the `object` crate — with a custom parser for a specific format, or with `goblin` for a specific use case — we change `format/object.rs` only, and no tool code is affected.

---

## Project 6 — GNU binutils (original)

### What It Is

The reference implementation. `objdump`, `nm`, `readelf`, `strip`, `objcopy`, `ar`, `size`, `addr2line`, `strings` — the tools that have shipped with every Linux distribution for decades. The C source code is the specification for what each tool should do. Their output format, flag behavior, and edge case handling define what "correct" means for OxideUtils.

### What They Got Right

Correctness and completeness. GNU binutils handles every edge case in every binary format that exists. The test suite is enormous. The documentation (man pages, info pages) is thorough. `readelf`'s output is the de facto standard format for inspecting ELF files.

### Where They Fell Short

The codebase is approximately 800,000 lines of C with no memory safety guarantees. BFD (Binary File Descriptor library), which underpins most binutils tools, is a monolithic C library that is notoriously difficult to contribute to. Build system is autotools. Cross-compilation is painful. Adding a new tool requires understanding decades of accumulated macro-based C idioms.

Output is not colored by default on most systems. There is no JSON output mode. Error messages are often terse to the point of being unusable. Parallelism is absent in most tools.

### Lesson Applied to OxideUtils

GNU binutils is the target for compatibility testing, not for code inspiration. The integration test suite in `tests/integration/gnu-compat/` runs both the GNU tool and the OxideUtils equivalent on the same binary and diffs the output in `--gnu-compat` mode. Any divergence is a bug.

The man pages in `docs/man/` are written by reading the GNU man pages and documenting every flag, then adding documentation for OxideUtils-specific flags (`--json`, `--color`, etc.).

---

## Summary: Architectural Decisions Derived from This Study

| Decision | Source lesson |
|---|---|
| `cli/config.rs` as a shared global config struct | uutils: lack of shared CLI config caused inconsistency across tools |
| Output abstraction: core returns data, formatter prints it | ripgrep: clean separation between search engine and printer |
| `--json` and color as first-class, day-one features | ripgrep: retrofitting these later is much harder than designing for them |
| `[workspace.dependencies]` with justification comments | fd: dependency discipline keeps compile times fast and binaries small |
| `format/traits.rs` designed and frozen before parsers | mold: data model first, implementation second |
| `OxideObject` wrapper around `object` crate | object crate lesson: never let library types leak into tool code |
| GNU compat diff testing in CI | GNU binutils: correctness is defined by existing behavior, not by assumptions |
| Phase gate: no tool code until Phase 2 traits are stable | lld: adding formats without a shared abstraction caused duplicated logic |

---

## What No Existing Project Combines

No current project combines all four of the following:

1. Multi-tool workspace architecture with a shared core library (uutils does this)
2. ripgrep-level output quality with JSON and color as first-class features
3. object+gimli-level format coverage with a clean abstraction layer above it
4. mold-level design discipline: interfaces before implementations

OxideUtils is designed to be the first project that combines all four. This is the gap in the ecosystem it fills.

---

*This document is updated when a new lesson is learned from a dependency, contributor report, or post-implementation review. Version history is tracked in CHANGELOG.md.*
