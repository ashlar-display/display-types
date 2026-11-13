# Roadmap

What comes next for display-types. What has shipped is in [CHANGELOG.md](../CHANGELOG.md).

## Additive

- **`DisplayCapabilities` at the crate root in every build.** The type exists without
  `alloc` (only some of its fields need it), but the root re-export is `alloc`-only, so a
  public path depends on a feature. Re-export it unconditionally; meanwhile it is
  reachable as `display_types::capabilities::DisplayCapabilities`.
- **`serde` without `std`.** The `serde` feature takes serde with its default features,
  which need `std`, so `--features serde` does not build on a bare-metal target. Take
  serde without default features (plus `serde/alloc` from `alloc`) and build it for a
  bare target in CI.
- **Every declaration of a mode.** When the same mode is declared twice (an established
  timing and a detailed timing, a VIC and a DTD), only one source survives today. Keep
  each declaration with its source.
- **Image size per detailed timing.** `preferred_image_size_mm` comes from the preferred
  timing, so an EDID 1.3 or earlier without the preferred-timing bit reports no size even
  when its first timing descriptor gives one. Each descriptor's own size closes the gap.
- **Native pixel count per source.** DisplayID 1.x Display Parameters, the 1.x display
  device data and DisplayID 2.x Display Parameters all state the native pixel count;
  today one field holds whichever was decoded last. Give each source its own field.
- **The sink's native mode.** A field for the CTA-861 SVD native flag that links it to its
  mode, and a helper that combines every source of the native resolution (the SVD flag,
  the Native Video Resolution block, DisplayID, the VESA display device block, EDID 1.4's
  "first DTD is native") into one answer that names its source and reports
  contradictions.
- **Capability summaries.** Normalized views over the decoded blocks: color (formats and
  maximum bit depth per format), HDR (HDR10, HLG, Dolby Vision, luminance) and VRR
  (supported, refresh range), so a consumer does not walk the CTA-861 blocks itself.
- **Mode helpers.** Filters over the supported modes (progressive only, high refresh,
  within a pixel clock limit) and the pixel density of a mode from the screen size.
- **Capability diff.** Compare two `DisplayCapabilities` and report what changed, for
  tools that track a display's data across firmware updates or captures.

## Breaking

- **Group `DisplayCapabilities`** into modes, color, HDR, VRR and physical parts, instead
  of one flat struct.
- **Link-agnostic types for DisplayPort.** `ResolvedDisplayConfig::frl_rate` is an HDMI
  type; DisplayPort support (after HDMI is complete) needs a link description that covers
  both.
