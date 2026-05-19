# OpenProteo Roadmap

`openproteo-core` is the shared foundation for the open vendor parsers
(`opentfraw`, `opentimstdf`, `openwraw`). Each vendor crate stays
standalone (a user can pull in `opentfraw` alone and get mzML export);
the core crate defines the shared vocabulary, the `SpectrumSource`
trait, the canonical mzML writer, an Arrow bridge, and the cross-vendor
conformance harness.

`openproteo-io` is the umbrella that ties the three vendor parsers
together behind a uniform `detect_format` + `convert_to_mzml` API and
ships the `vendor2mzml` CLI.

This file is tracked in git and ships with the crate so external
contributors can see where the project is heading. Edit freely; the
plan evolves with each phase.

Status legend: `[x]` shipped, `[~]` in progress, `[ ]` planned.

## Phase 0 - Foundation - SHIPPED

- [x] Core types, `SpectrumSource` trait, canonical mzML 1.1.0 writer

## Phase 1 - Vendor alignment - SHIPPED

- [x] opentfraw, opentimstdf, openwraw all emit mzML through
      `openproteo-core::write_mzml` / `write_indexed_mzml`

## Phase 2 - SDK consolidation - SHIPPED

- [x] **2a Waters parity** (`OpenWRaw`): polarity from `_extern.inf`,
      MS-level from `FunctionMode` (MSe split into MS1/MS2 by index),
      IMS drift pass-through via `inv_mobility_per_peak`.
- [x] **2b Arrow bridge** (`openproteo-core`, `arrow` feature):
      `spectrum_record_schema()` + `SpectrumBatchBuilder` with
      `LargeList<Float64/Float32>` peak columns and a flat precursor
      block. Default builds remain zero-dep.
- [x] **2c Conformance harness** (`openproteo-core::conformance`):
      `assert_source_invariants` checks peak-array length parity,
      TIC + base-peak consistency, MS2 precursor presence, RT
      monotonicity per native-ID stream, and index sequencing. Wired
      into all three vendor crates via `tests/conformance.rs`.
      Validated on 3047 (Thermo), 24425 (Bruker TIMS), 590 (Waters
      IMS) spectra.
- [x] **2d Umbrella + CLI** (`OpenProteo`): `openproteo-io` library
      with feature-gated `thermo / bruker / waters / all` vendor
      re-exports, runtime `detect_format()` probe (Thermo Finnigan
      magic at offset 2, Bruker `analysis.tdf+_bin` in `.d/`, Waters
      `_HEADER.TXT` in `.raw/`), `convert_to_mzml()` dispatch. CLI
      `vendor2mzml <input> <output.mzML> [--indexed]`. End-to-end
      smoke test covers all three vendors (12 MB / 564 MB / 26 MB
      mzML).

### Carry-overs

- `openproteo-io-py` (PyO3 bindings) - deferred into Phase 3.

## Phase 3 - Distribution + ergonomics (PROPOSED WORK PACKAGE)

The Rust + mzML pipeline is now complete and validated. Phase 3 is
about making it usable from outside Rust and shippable to end users.

### 3a. `openproteo-io-py` (Python bindings)

Single PyO3 module that exposes the umbrella, not three separate
vendor modules. PEP 503 distribution name `openproteo-io`
(import name `openproteo_io`).

- [ ] `openproteo_io.detect(path: str) -> VendorFormat`
- [ ] `openproteo_io.to_mzml(input: str, output: str, *, indexed: bool=True)`
- [ ] `openproteo_io.iter_spectra(path: str) -> Iterator[Spectrum]`
      yielding objects with **zero-copy NumPy views** over the m/z,
      intensity, and (optional) inverse-mobility arrays via
      `numpy::PyArray1::from_slice_bound`.
- [ ] `openproteo_io.read_arrow(path: str) -> pyarrow.RecordBatchReader`
      backed by `SpectrumBatchBuilder` (requires the `arrow` feature).
- [ ] maturin build, abi3-py39 wheels, manylinux2014.
- [ ] pytest smoke (Thermo / Bruker / Waters round-trips).

### 3b. CLI polish (SHIPPED)

`vendor2mzml` is now a clap-derive binary with `convert` + `info`
sub-commands, validation, and gzip output.

- [x] Replace ad-hoc arg parsing with `clap` derive.
- [x] `--validate` post-pass running the conformance harness on the
      source before writing (skip-write on failure, exit 3).
- [x] `--profile json|text` emitting timing + record counts to stderr.
- [x] Auto-detect output format from extension (`.mzML.gz` -> gzip via
      flate2).
- [x] `vendor2mzml info <input>` sub-command: prints vendor, instrument,
      polarity, MS-level breakdown, spectrum count, RT range; `--json`
      for machine-readable output.
- [x] `--version` reports `vendor2mzml` + `openproteo-core` versions.

### 3c. Release engineering (SHIPPED workflows; publish deferred)

GitHub Actions workflows are in place for both repos. Tag a release
with `vX.Y.Z` and the workflows attach `vendor2mzml` archives + Python
wheels to the GitHub Release. Publish to crates.io / PyPI is
intentionally deferred per Phase 3 decision.

- [ ] Publish `openproteo-core` 0.1.0 to crates.io. *(deferred)*
- [ ] Publish `openproteo-io` 0.1.0 to crates.io once the vendor
      crates' new tags are out. *(deferred)*
- [x] GitHub Actions: per-repo CI (rustfmt + clippy + test + msrv 1.75),
      release workflow uploading `vendor2mzml` binaries for linux-x64,
      linux-aarch64, macos-x64, macos-arm64, windows-x64.
- [x] Python wheel workflow (maturin, abi3-py39) covering the same
      five targets plus an sdist.
- [ ] Re-validate byte-identical mzML diff on the PXD058812 Waters
      baseline after 2a landed (sanity check). *(open)*
- [ ] CHANGELOGs synced across the five repos. *(open)*

### 3d. Documentation (SHIPPED)

`OpenProteo/docs/` is an mdBook with chapters for install, three
quickstarts (CLI / Rust / Python), the format-detection rules, the
conformance contract, the Arrow schema, per-vendor coverage notes,
and the architecture / crate-layout / pure-Rust design rationale.

- [x] `OpenProteo/docs/` mdBook: quickstart for Rust + Python +
      `vendor2mzml`, format-detection notes, conformance contract,
      design rationale.
- [x] Per-vendor chapter listing covered MassLynx / Finnigan / TDF
      features and known gaps.

## Phase 4 - Performance + scale (later)

- Criterion bench suite gated on PRs.
- Rayon intra-file ingest for indexed writes.
- Async / object-store readers (S3, GCS) behind `openproteo-core::io`.
- Streaming Arrow writes to Parquet (DataFusion compatibility).
- Lance corpus exporter via `openproteo-core::arrow` + lance-rs.

## Phase 5+ - Search-engine bridge (later)

- `ProLance` integration via one `SpectrumSource`-consuming ingester.
- Sage / Comet-rs adapter behind `openproteo-io::search`.
- Spectral library export (BLIB, MSP).

## Design invariants

1. Each vendor crate is self-contained. `openproteo-core` is a leaf
   dependency, never a runtime hub.
2. `SpectrumRecord` is the only handshake.
3. RT in seconds; mzML writes minutes at the boundary.
4. `native_id` is verbatim in the vendor's canonical format.
5. No `unsafe` anywhere in the suite (`unsafe_code = "forbid"`).
