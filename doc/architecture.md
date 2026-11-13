# Architecture

## Role

display-types is the vocabulary the Ashlar crates share. A display data parser
(piaf) produces its types; a negotiation engine (concordance) consumes them; the layers
below (InfoFrames, SCDC, link training, the hardware traits) use the same mode, color and
link types. Because every crate speaks these types, a parser, a negotiation engine or a
driver can be replaced without converting between private models.

The crate holds data and the small pure computations that belong to the data: the CTA-861
VIC and VESA DMT tables, the CVT reduced-blanking timing formulas, pixel clock estimation,
and color capability derivation. It parses nothing and decides nothing.

## The two models

- **[`DisplayCapabilities`]** — what a sink can do, as decoded from its EDID and
  DisplayID data: identity (manufacturer, product, serial, dates), the video input and
  feature flags, color characteristics, screen size, the preferred timing and supported
  modes, and the decoded CTA-861 and DisplayID blocks. A parser fills it; it is the input
  of negotiation.
- **[`ResolvedDisplayConfig`]** — one configuration chosen to drive the sink: the video
  mode, color encoding and bit depth, the FRL rate (or TMDS), and whether DSC and VRR
  apply. A negotiation engine produces it; drivers and InfoFrame encoders consume it.

Each model lives here rather than in the crate that produces it, so a consumer does not
depend on the producer: negotiation does not depend on the parser, and a driver does not
depend on negotiation.

## Module map

| Module | Contents |
|---|---|
| `capabilities` | `DisplayCapabilities`, `VideoMode` and its signal fields, `ModeSource` (which descriptor or table a mode came from), `RefreshRate` (an exact fraction), `ParseWarning`, `ExtensionData` |
| `cea861` | Types of the CTA-861 extension blocks (audio, colorimetry, HDR, HDMI and HDMI Forum VSDBs, speaker allocation, video capability, VESA blocks, video timing blocks), the VIC and DMT tables (`vic_to_mode`, `dmt_to_mode`), well-known OUIs |
| `displayid` | Types of the DisplayID 1.x and 2.x data blocks, the block tags and product types |
| `color` | Chromaticity, gamma, color formats and bit depths, and `color_capabilities_from_edid` |
| `timing` | The CVT reduced-blanking formulas, v1 and v2 (`compute_type_ix_timing`), `pixel_clock_khz`, and the GTF and CVT parameters a display declares |
| `panel`, `screen`, `input`, `features`, `manufacture`, `transfer` | Panel characteristics, size and aspect ratio, input interface, feature flags, identity, luminance transfer curves |
| `resolved` | `ResolvedDisplayConfig` |
| `prelude` | `Vec`, `String`, `Box` and `Arc` from `alloc` or `std`, whichever the build has |

## Builds

| Features | What is available |
|---|---|
| `std` (default) | Everything |
| `alloc` | Everything; heap types come from `alloc` |
| neither | The types that hold no lists (enums, flags and plain structs such as `VideoMode`, `RefreshRate` and `ResolvedDisplayConfig`), `DisplayCapabilities` without its list fields, the tables and the formulas |

Without `alloc`, `DisplayCapabilities` has no supported modes, warnings, extension data or
DisplayID transfer characteristic, and the block types that hold lists do not exist. A
parser without `alloc` keeps those in fixed-capacity storage of its own. `serde` derives
`Serialize` and `Deserialize` on the public types. It takes serde with its default
features, so today it needs `std`; making it build without `std` is on the roadmap.

## Design rules

- **The standard, complete.** Every field the relevant standard defines is decoded and
  exposed; none is left out because it looks obscure. What is useful is the consumer's
  decision.
- **Nothing invented.** A field is `Option` where the data may be absent or undecodable,
  and `None` means exactly that; the types never carry a default in place of data the
  display did not give.
- **No judgement.** The types describe what the display declares. Choosing among modes,
  correcting quirks and resolving contradictions belong to the parser or the negotiation
  engine.
- **Stable for consumers.** Structs that may gain fields are `#[non_exhaustive]`, so a
  field can be added without a breaking release; small values that are complete by
  definition (a chromaticity point, a refresh rate, a manufacturer ID) are not. Crates
  outside this one (parsers, tests) build the non-exhaustive structs with a constructor,
  or from `Default` and the public fields.
- **Features are additive.** Enabling `alloc` or `std` should only add types and methods,
  never change a type that exists without it, because Cargo enables a feature for every
  crate in a build that one of them asks for. `DisplayCapabilities` does not meet this yet:
  its supported modes, warnings, extension data and DisplayID transfer characteristic are
  public fields that exist only with `alloc`. Moving them behind accessors is on the
  roadmap.
- **Extensible without forking.** A parser's custom handler stores its own typed data in
  `DisplayCapabilities::extension_data`, keyed by extension tag, and reports its own
  warning types as `ParseWarning` (any `core::error::Error`); consumers recover them by
  downcasting.

## Versioning

Several crates re-export display-types' types in their own public APIs. A breaking release
of display-types is therefore a breaking release of each crate that re-exports the
changed types, and is released together with them.

[`DisplayCapabilities`]: https://docs.rs/display-types/latest/display_types/capabilities/struct.DisplayCapabilities.html
[`ResolvedDisplayConfig`]: https://docs.rs/display-types/latest/display_types/resolved/struct.ResolvedDisplayConfig.html
