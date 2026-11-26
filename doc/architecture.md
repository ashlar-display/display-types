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
| `cea861` | Types of the CTA-861 extension blocks (audio, colorimetry, HDR, HDMI and HDMI Forum VSDBs, speaker allocation, video capability, VESA blocks, video timing blocks), the VIC and DMT tables (`vic_to_mode`, `dmt_to_mode`), the reverse lookup (`mode_to_vic`, `mode_to_vic_with`, `VicMatch`, `VicMatchOptions`), the HDMI_VIC table (`hdmi_vic_to_vic`, `hdmi_vic_to_mode`), well-known OUIs |
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

## Mode → VIC lookup

`vic_to_mode` turns a VIC into its mode; `mode_to_vic` goes the other way, for code that
builds InfoFrames from a `ResolvedDisplayConfig`: which VIC, and which HDMI 1.4 HDMI_VIC,
carry a given mode. How a mode is taken to be a VIC is behaviour of HDMI sources, so the
rules come from public source implementations, read at the commits under
[Sources](#sources); where they disagree, the majority is the default and the others are
settings.

**What it answers, and what it leaves.** The lookup says which VIC and HDMI_VIC carry the
mode's timing. Which one to send (the AVI VIC or the HDMI_VIC, under the sink's
capabilities, 3D or an HF-VSIF), and which member of an aspect pair, is the InfoFrame
builder's decision, not this crate's ("No judgement").

**Shape.** `mode_to_vic(&VideoMode)` uses `VicMatchOptions::default()`;
`mode_to_vic_with(&VideoMode, &VicMatchOptions)` takes the settings. Both return
`Option<VicMatch>`:

| Field | Meaning |
|---|---|
| `vic` | The VIC carrying the timing |
| `alternate` | The other member of `vic`'s aspect pair (4:3/16:9 or 16:9/64:27), **only when the mode does not determine the aspect**; `None` when it does, or when `vic` has no pair |
| `hdmi_vic` | The HDMI_VIC carrying the same timing, if any |

`VicMatch` and `VicMatchOptions` are `#[non_exhaustive]`. `hdmi_vic_to_vic` and
`hdmi_vic_to_mode` give an HDMI_VIC's VIC and mode.

**How a mode matches a VIC.**
- **A mode decoded from an SVD** (`ModeSource::Vic(n)`) is VIC `n`, when its timing matches
  `n`'s by the rules below; its aspect is then known and `alternate` is `None`.
- **Timing.** Width, height, horizontal front porch, sync width and blanking, vertical
  front porch and sync width, and interlace are compared exactly; the lowest VIC that
  matches is returned [1][2][3][8]. `VideoMode` has no vertical blanking or total, so the
  vertical comparison stops at front porch and sync width. A mode without detailed timing
  (no `h_blank`) matches only through its `ModeSource::Vic`.
- **Sync polarities are not compared** by default [2][3]; comparing them [1] is a setting.
- **Pixel repetition** is matched at full width and clock, the convention of `vic_to_mode`.
- **Rate.** With `pixel_clock_khz`, the clock must be within ±5 kHz (a DTD's 10 kHz
  resolution) of the VIC's clock or, when the VIC's rate is a multiple of 6, of its
  ×1000/1001 clock [1][4][5]. Without it, `refresh_rate` must equal the VIC's rate or, for a
  multiple of 6, its ×1000/1001 rate, exactly. Accepting any rate within ±0.5% [2][3] is a
  setting. Both rates match the same VIC; the VIC does not say which is sent.
- **Aspect.** `VideoMode` has no picture aspect field, and width and height do not give
  it: the pairs share a timing, and in the 720×480 and 64:27 formats the pixels are not
  square. Without `ModeSource::Vic`, the match returns the lower member as `vic` [1][2][3]
  and the other as `alternate`. Because `alternate` is defined by whether the mode
  determines the aspect, an aspect carried by `VideoMode` would make it `None` more often
  without changing the signature or the meaning.
- **640×480p60 is VIC 1** [1].

**HDMI_VIC.** HDMI_VIC 1, 2, 3, 4 carry the timings of VIC 95, 94, 93 and 98 in every source
examined [1][4][5][6][9]. HDMI_VIC 1–3 also cover their ×1000/1001 rate. HDMI_VIC 4 does
too by default [1][4][7]; with 24 Hz only [2][6], a 4096×2160 timing at 23.976 Hz is VIC 98
with no HDMI_VIC.

**`VicMatchOptions`**, `Default` giving the defaults above:

| Field | Default | Other value |
|---|---|---|
| `compare_sync_polarity` | `false` [2][3] | `true` [1] |
| `fractional_rates` | `FractionalRates::MultiplesOfSix` [1][4][5] | `FractionalRates::WithinHalfPercent` [2][3] |
| `hdmi_vic_4_fractional` | `true` [1][4][7] | `false`, 24 Hz only [2][6] |

## Versioning

Several crates re-export display-types' types in their own public APIs. A breaking release
of display-types is therefore a breaking release of each crate that re-exports the
changed types, and is released together with them.

## Sources

Source read at these commits; line numbers refer to them.

- Linux, github.com/torvalds/linux `4dce97c6c265` (2026-11-24)
- NVIDIA, github.com/NVIDIA/open-gpu-kernel-modules `a4b95f6a2db0` (2026-11-18)
- Qualcomm, github.com/LineageOS/android_kernel_qcom_msm8998 `d152ec623891` (2026-08-14)
- Amlogic, github.com/CoreELEC/common_drivers `44c281f03228` (2026-11-24)
- edid-decode, git.linuxtv.org/v4l-utils.git `d2e89246cefc` (2026-10-30)
- HiSilicon, gitee.com/openharmony/device_soc_hisilicon `1ee91262fff0` (2025-07-07)
- NVIDIA Tegra, github.com/OE4T/linux-tegra-4.9 `ea4853708c52` (2024-06-29)
- Xilinx embeddedsw, github.com/Xilinx/embeddedsw `5aebf0defbcc` (2026-11-18)

|     | Source | Location |
|-----|--------|----------|
| [1] | Linux | `drivers/gpu/drm/drm_edid.c:4320–4338` (×1000/1001 clock, multiples of 6), `4417–4449` (first VIC whose clock, timings and flags match), `4472–4480` (the same rule for HDMI_VICs), `1624–1652` (HDMI_VIC timings); `drivers/gpu/drm/drm_modes.c:1491–1504`, `1519–1524` (timings and flags, polarities included) |
| [2] | NVIDIA | `src/common/modeset/timing/nvt_edidext_861.c:2214–2278` (first match), `428–431` (HDMI_VIC timings; HDMI_VIC 4 at 24.000 Hz), `155` (VIC 98 at 23.976/24 Hz); `nvt_util.c:417–438` (polarities skipped), `nvt_edid.c:607` (rounded rate) |
| [3] | Qualcomm | `drivers/video/fbdev/msm/mdss_hdmi_util.c:713–763` (first match; `725` polarities and interlace not compared; `732` ±0.5% clock and rate) |
| [4] | Amlogic | `drivers/media/vout/hdmitx_common/hdmitx_mode.c:599–609` (×1000/1001 for multiples of 6); `hdmitx_edid_parse.c:359–371` (HDMI_VIC → VIC); `hdmitx_packet.c:30` (HDMI_VIC 4 for VIC 98 at either rate) |
| [5] | edid-decode | `utils/edid-decode/parse-cta-block.cpp:269` (HDMI_VIC → VIC); `edid-decode.cpp:563–573` (reduced rate for multiples of 6, HDMI_VICs included) |
| [6] | HiSilicon | `hi3516dv300/sdk_linux/drv/mpp/component/hdmi/src/mkp/drv_hdmi_common.c:178–188` (HDMI_VIC table; HDMI_VIC 1 and 3 at the ×1000/1001 clock, 2 and 4 at the integer clock) |
| [7] | NVIDIA Tegra | `nvidia/drivers/video/tegra/dc/hdmi2.0.c:2238–2264` (×1000/1001 removed before the HDMI_VIC lookup) |
| [8] | Xilinx | `XilinxProcessorIPLib/drivers/v_hdmi_common/src/xv_hdmic.c:50–61` (table order, lower VIC first) |
| [9] | Linux, amdgpu DC | `drivers/gpu/drm/amd/display/dc/core/dc_resource.c:5188–5203` (HDMI_VIC → VIC) |

[`DisplayCapabilities`]: https://docs.rs/display-types/latest/display_types/capabilities/struct.DisplayCapabilities.html
[`ResolvedDisplayConfig`]: https://docs.rs/display-types/latest/display_types/resolved/struct.ResolvedDisplayConfig.html
