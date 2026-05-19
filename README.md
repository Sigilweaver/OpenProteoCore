# openproteo-core

> Part of the
> [OpenProteo](https://github.com/Sigilweaver/OpenProteo) stack for
> proteomics raw-file access. `openproteo-core` is the
> vendor-neutral foundation: records, the `SpectrumSource` trait,
> a canonical mzML 1.1.0 writer, an Arrow bridge, and the
> cross-vendor conformance suite. It is consumed by every reader in
> the stack -
> [OpenTFRaw](https://github.com/Sigilweaver/OpenTFRaw) (Thermo),
> [OpenTimsTDF](https://github.com/Sigilweaver/OpenTimsTDF) (Bruker),
> [OpenWRaw](https://github.com/Sigilweaver/OpenWRaw) (Waters) -
> and by the
> [openproteo-io](https://github.com/Sigilweaver/OpenProteo)
> umbrella.

Shared foundation for the open Rust mass-spec parsers
([opentfraw](https://github.com/Sigilweaver/OpenTFRaw),
[opentimstdf](https://github.com/Sigilweaver/OpenTimsTDF),
[openwraw](https://github.com/Sigilweaver/OpenWRaw)).

Defines the vendor-neutral records, the `SpectrumSource` trait every
parser implements, a canonical mzML 1.1.0 writer (with optional indexed
mzML output), an optional Apache Arrow bridge, and a cross-vendor
conformance harness.

- MSRV: 1.75
- License: Apache-2.0
- `#![forbid(unsafe_code)]`

Each vendor crate stays a complete standalone tool: a user pulls in
`opentfraw` alone and gets parsing **and** mzML export. This crate is
the shared vocabulary that keeps the three parsers in lock-step.

## Install

```toml
[dependencies]
openproteo-core = "0.1"

# Optional: zero-copy Arrow RecordBatch builder for spectra.
openproteo-core = { version = "0.1", features = ["arrow"] }
```

## Quick example

Implement `SpectrumSource` and write a valid mzML document:

```rust
use openproteo_core::{
    write_indexed_mzml, RunMetadata, SpectrumRecord, SpectrumSource,
};

struct MySource {
    spectra: Vec<SpectrumRecord>,
}

impl SpectrumSource for MySource {
    fn run_metadata(&self) -> RunMetadata {
        RunMetadata::default()
    }
    fn iter_spectra<'a>(&'a mut self) -> Box<dyn Iterator<Item = SpectrumRecord> + 'a> {
        Box::new(self.spectra.drain(..))
    }
    fn spectrum_count(&self) -> Option<usize> {
        Some(self.spectra.len())
    }
}

let mut src = MySource { spectra: vec![/* SpectrumRecord { .. } */] };
let mut out = std::fs::File::create("run.mzML")?;
write_indexed_mzml(&mut src, &mut out)?;
# Ok::<(), Box<dyn std::error::Error>>(())
```

## Public API

| Symbol                                                      | Module          | Purpose                                                                  |
| ----------------------------------------------------------- | --------------- | ------------------------------------------------------------------------ |
| `SpectrumRecord`                                            | `types`         | Decoded spectrum: id, ms level, polarity, rt, peaks, precursor.          |
| `PrecursorInfo`                                             | `types`         | Selected / isolated precursor, charge, activation, scan window.          |
| `ChromatogramRecord`                                        | `types`         | TIC / BPC / SRM trace.                                                   |
| `RunMetadata`                                               | `types`         | Run-level CV terms: instrument, source format, native id format.         |
| `CvTerm`                                                    | `types`         | A PSI-MS controlled-vocabulary term.                                     |
| `Polarity`, `Analyzer`, `ScanMode`, `MsPower`, `Activation` | `enums`         | Standard enumerations.                                                   |
| `MobilityArrayKind`                                         | `enums`         | Per-peak inverse-mobility / drift-time array kind.                       |
| `SpectrumSource`                                            | `source`        | Trait every parser implements.                                           |
| `write_mzml`                                                | `mzml`          | Stream a `SpectrumSource` to a plain mzML 1.1.0 document.                |
| `write_indexed_mzml`                                        | `mzml`          | Same, with `<indexList>` + SHA-1 footer for byte-offset indexing.        |
| `conformance::assert_source_invariants`                     | `conformance`   | Check a live `SpectrumSource` for cross-vendor invariants.               |
| `conformance::assert_iter_invariants`                       | `conformance`   | Same, but from any `IntoIterator<Item = SpectrumRecord>`.                |
| `arrow::SpectrumBatchBuilder`                               | `arrow` (feat)  | Zero-copy builder for `arrow_array::RecordBatch` from a spectrum stream. |
| `arrow::spectrum_record_schema`                             | `arrow` (feat)  | The canonical Arrow schema.                                              |
| `Error`                                                     | `error`         | Aggregate `thiserror`-based error type.                                  |

## Conformance harness

The conformance module enforces the cross-vendor invariants every parser
must satisfy:

- monotonic spectrum indices,
- non-negative, non-decreasing retention times,
- equal-length m/z and intensity arrays,
- equal-length mobility arrays (when present),
- MS-level / polarity sanity,
- precursor presence on MSn spectra.

Failures surface as `ConformanceError` variants
(`PeakArrayLengthMismatch`, `MobilityArrayLengthMismatch`,
`RetentionTimeNonMonotonic`, and others).

The `vendor2mzml validate` subcommand in
[OpenProteo](https://github.com/Sigilweaver/OpenProteo) runs this
harness on any vendor input or pre-existing mzML.

To assemble a local corpus for running the harness across vendors,
see the shared corpus schema and fetcher described in
[OpenProteo/docs/CORPUS.md](https://github.com/Sigilweaver/OpenProteo/blob/main/docs/CORPUS.md).

## Feature flags

| Flag    | Default | Effect                                                  |
| ------- | :-----: | ------------------------------------------------------- |
| `arrow` |    no   | Enables `arrow_array::RecordBatch` building from spectra. |

## Ecosystem

```text
              openproteo-core   (this crate: types + trait + mzML writer)
                     ^
        +------------+------------+
        |            |            |
   opentfraw    opentimstdf    openwraw       (vendor parsers)
        |            |            |
        +------------+------------+
                     v
               openproteo-io      (umbrella: detect_format, collect, to_mzml)
                     |
        +------------+------------+
        |                         |
  vendor2mzml CLI            openproteo (Python metapackage)
```

- Unified docs hub: https://github.com/Sigilweaver/OpenProteo
- Umbrella crate (workspace): https://github.com/Sigilweaver/OpenProteo
- Vendor parsers:
  [opentfraw](https://github.com/Sigilweaver/OpenTFRaw),
  [opentimstdf](https://github.com/Sigilweaver/OpenTimsTDF),
  [openwraw](https://github.com/Sigilweaver/OpenWRaw).

## Roadmap

See [ROADMAP.md](ROADMAP.md) for the multi-phase plan.

## Changelog

See [CHANGELOG.md](CHANGELOG.md).

## License

Apache-2.0
