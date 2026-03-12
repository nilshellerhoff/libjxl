# True ROI decode design for `djxl` / libjxl

## Goal

Support decoding only a requested region of interest (ROI) instead of decoding the full image and cropping afterwards.

Current behavior in `djxl` is output crop after full decode.

## Current pipeline (where full decode is enforced)

- Public decode API (`lib/include/jxl/decode.h`) has no ROI/window setter.
- `JxlDecoderProcessCodestream` always drives per-frame section processing through `JxlDecoderProcessSections` in `lib/jxl/decode.cc`.
- `JxlDecoderProcessSections` builds bitreaders for all frame sections from TOC and passes them to `FrameDecoder::ProcessSections`.
- `FrameDecoder::ProcessSections` schedules AC work per group (`ProcessACGroup`) over all groups with available section data.
- Output dimensions come from `GetCurrentDimensions` in `lib/jxl/decode.cc`, then are fed into `FrameDecoder::SetImageOutput`.

So today, there is no control point that asks core decode to only materialize an ROI.

## Constraints and implications

1. **No stable API for ROI in decoder**
   - Must add a new decoder option/API first.

2. **Group-based entropy + post-filters**
   - VarDCT decode is group-oriented; post-processing (EPF/gaborish, borders) uses neighboring data.
   - ROI decode cannot naively decode only exact ROI groups without halo.

3. **Modular full-image mode**
   - `modular_frame_decoder_.UsesFullImage()` path may require broader/full state.
   - Initial ROI implementation should fall back to full decode for unsupported frame modes.

4. **Animation/coalescing references**
   - Coalesced animation may depend on off-ROI reference content.
   - First implementation should target still images and simple frames.

## Proposed API changes (phase 1)

Add a decode API window setter (name bikeshed):

- `JxlDecoderSetDecodeRegion(JxlDecoder* dec, uint32_t x0, uint32_t y0, uint32_t xsize, uint32_t ysize)`

Behavior:

- Must be called before decoding starts (same pattern as `SetCoalescing`, etc.).
- Coordinates are in oriented output pixel space (consistent with current output path).
- Region is clamped/validated against frame dimensions when known.
- Buffer size queries (`JxlDecoderImageOutBufferSize`, `JxlDecoderExtraChannelBufferSize`) return ROI-sized output.
- Image callback coordinates are ROI-relative.

Compatibility:

- `JxlBasicInfo` remains full image size.
- Existing users unaffected unless they opt into region decode.

## Decoder architecture changes (phase 2: real partial decode)

### A. Track ROI in decoder state

In `lib/jxl/decode.cc` `JxlDecoder` struct:

- Add optional ROI rectangle and enabled flag.
- Add per-frame derived ROI info (after orientation/frame dims).

### B. Compute required group set for frame

At frame setup time (after `FrameHeader` known):

- Convert ROI rectangle to frame coordinates.
- Expand by required filter halo.
- Convert to AC/DC group ranges.

For first implementation:

- VarDCT + non-preview + coalescing + non-animation only.
- If unsupported mode is detected, fall back to full decode for correctness.

### C. Section scheduling in `JxlDecoderProcessSections`

Currently all sections are queued.

Change to:

- Always include required global sections (DC global, AC global).
- Include only required DC/AC group sections for needed groups.
- Mark skipped non-required sections as consumed in codestream progression bookkeeping so input advances without decoding their payload.

### D. FrameDecoder handling for skipped groups

`FrameDecoder` currently assumes complete group progression for full frame completion.

Need one of:

1. explicit skip API to mark AC/DC group sections as intentionally skipped, OR
2. synthetic zero-fill path for skipped groups that still satisfies invariants.

Preferred:

- Add explicit “group skipped” bookkeeping for non-output groups to avoid fake decode work.

### E. Render pipeline and borders

Because borders are exchanged between neighbor groups:

- Required group set must include at least 1 neighbor ring (or exact halo from pipeline padding data).
- Run pipeline only for required groups.

For output:

- Final write/callback clips to ROI rectangle.

## `djxl` integration

Once API exists:

- Replace post-crop logic in `tools/djxl_main.cc` with:
  - `JxlDecoderSetDecodeRegion(...)` via extras path (or direct decoder path where needed).
- Keep post-crop fallback only for unsupported frame modes if decoder reports region unsupported.

## Testing plan

1. API unit tests
   - Valid/invalid region setting order.
   - Buffer size reflects ROI.

2. Decoder correctness
   - ROI output equals full-decode+crop for still VarDCT images (multiple formats, bit depths, alpha).

3. Edge cases
   - ROI touching borders.
   - Tiny ROIs (1x1, narrow strips).
   - Orientation-on/off consistency.

4. Fallback behavior
   - Modular/full-image, animation, non-coalesced paths either:
     - produce correct ROI with full decode fallback, or
     - return clear unsupported status (policy decision).

5. Performance
   - CPU and memory reduction on large images for small ROI.

## Phased delivery recommendation

- **Phase A (already done in `djxl`)**: full decode + output crop (`--region`).
- **Phase B**: new decoder API + ROI-sized output surfaces; still full decode fallback.
- **Phase C**: true partial group decode for VarDCT still images.
- **Phase D**: broaden support (modular/animation) as feasible.

## Expected payoff

For large still VarDCT images and small ROI, Phase C should reduce:

- entropy decode work (skip non-required group sections),
- render pipeline work outside ROI+halo,
- output memory footprint.

Actual speedup depends on ROI area, filter halo, and image mode.
