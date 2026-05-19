# Changelog

All notable changes to `openproteo-core` are documented here. The format
follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/) and the
crate adheres to [Semantic Versioning](https://semver.org/).

## [Unreleased]

- _No unreleased changes yet._

## [0.1.0]

Initial published shape of the crate. This release defines the
vendor-neutral foundation the vendor parsers
(`opentfraw`, `opentimstdf`, `openwraw`) build on.

### Added

- Vendor-neutral record types: `SpectrumRecord`, `PrecursorInfo`,
  `ChromatogramRecord`, `RunMetadata`, `CvTerm`.
- Standard enumerations: `Polarity`, `Analyzer`, `ScanMode`, `MsPower`,
  `Activation`, `MobilityArrayKind`.
- `SpectrumSource` trait that every vendor parser implements; default
  empty `iter_chromatograms` and `spectrum_count`.
- Canonical mzML 1.1.0 writer (`write_mzml`) and indexed-mzML writer
  (`write_indexed_mzml`) with `<indexList>` and SHA-1 footer.
- Conformance harness (`assert_source_invariants` /
  `assert_iter_invariants`) with structured `ConformanceError`
  variants (peak-array length, mobility-array length, retention-time
  monotonicity, MS-level / polarity, precursor presence, index
  sequence, empty spectrum).
- Optional `arrow` feature: zero-copy `SpectrumBatchBuilder` and the
  canonical `spectrum_record_schema()` for downstream Arrow / Parquet
  / Lance consumers.
- Aggregate `Error` enum (`thiserror`-based) covering I/O, decode, and
  conformance failures.

### Policy

- MSRV pinned at Rust 1.75.
- `#![forbid(unsafe_code)]` crate-wide.
- License: Apache-2.0.

[Unreleased]: https://github.com/Sigilweaver/OpenProteoCore/compare/v0.1.0...HEAD
[0.1.0]: https://github.com/Sigilweaver/OpenProteoCore/releases/tag/v0.1.0
