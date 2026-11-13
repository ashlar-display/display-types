# Testing strategy

display-types holds data types, their constructors and a few pure computations (timing
formulas, lookup tables, color capability derivation). Every test is a small deterministic
unit test or a doctest; there is no I/O to simulate and no input parser to fuzz.

## Test structure

### Unit tests

Unit tests live next to the code they test, in a `#[cfg(test)]` module at the end of each
file:

| File | What the tests check |
|---|---|
| `src/timing.rs` | Pixel clock estimation (exact clocks returned unchanged, saturation on oversized geometry) and the CVT reduced-blanking formulas, v1 and v2, against known modes and degenerate input |
| `src/cea861/vic_table.rs` | Lookups of individual VICs (including interlaced and aliased entries), the codes that have no mode, and that every entry carries its horizontal blanking from CTA-861 |
| `src/cea861/dmt_table.rs` | Lookups by DMT ID, the fractional NTSC rate, every entry's horizontal blanking, and the rows corrected against VESA DMT |
| `src/cea861/hdmi_forum.rs` | Lane count and order for every FRL rate; `HdmiForumDsc::new` keeps each flag in its field |
| `src/cea861/hdr.rs` | `HdrStaticMetadata::new` keeps each luminance value in its field |
| `src/cea861/vesa_dddb.rs` | `VesaDisplayDeviceBlock::new` keeps each number and each flag in its field |
| `src/color.rs` | Color capability derivation: bit depths per color format from the base block, the HDMI VSDB deep-color flags and the HF-SCDB |
| `src/displayid/mod.rs` | DisplayID field decoding: chromaticity normalisation, display technology and scan orientation codes (including reserved and unknown values), color depth bit layouts |
| `src/capabilities.rs` | `RefreshRate` arithmetic (reduction, canonical equality, ordering), extension data storage, the preferred timing, and the serde round trips |
| `src/manufacture.rs` | `ManufacturerId` invariants, `MonitorString` trimming, and their serde wire format |
| `src/screen.rs`, `src/resolved.rs` | Aspect ratio formulas; `ResolvedDisplayConfig` construction |
| `src/panel.rs` | `PowerSequencing::new` keeps each delay in its field |

### Doctests

Examples in the rustdoc comments run as doctests. The README's Rust examples run as
doctests too (`ReadmeDoctests` in `src/lib.rs`, compiled only for `cargo test`). An
example that needs a feature-gated type is wrapped in a hidden
`# #[cfg(feature = "alloc")] { … # }` block, so it compiles to nothing in builds without
that feature instead of failing.

## Feature sets

Tests that need heap types are gated on `alloc` or `std`, and the serde tests on `serde`,
like the code they test. CI runs the tests with the default features, with `serde`, with
`alloc` only and with no default features, so each gate is exercised; the same sets run
on the MSRV. The bare `no_std` and `alloc`-only builds are compiled for
`thumbv7em-none-eabi`, which has no `std`.

## Coverage

Coverage is measured with `cargo-llvm-cov` over the default and `serde` feature sets and
kept from falling by a ratchet: CI fails when line coverage drops below
`.coverage-baseline` (with a 0.1 point tolerance), and opens a PR raising the baseline when
a push improves it. See [setup.md](setup.md#coverage) for the commands.

## Philosophy

- **Check computed values against the standard's own numbers.** Timing formulas and
  tables are tested against modes the VESA and CTA documents list, not against values
  the code produced once.
- **Test the edges.** Zero and unreachable inputs, reserved and unknown codes, and
  saturation are tested explicitly, so a type never panics or wraps on unusual data.
- **Every build is tested, not only compiled.** A feature gate that hides a test or an
  example is exercised by the feature set that turns it off.
