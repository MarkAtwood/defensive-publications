# Defensive Publication: In‑Band Vector Scene Metadata for AV1

** Publication Date:** 2025‑08‑12

**Author:** Mark Atwood (mark@reviewcommit.com)

**Purpose:** This document serves as a defensive publication to establish prior art and prevent future patent claims on the methods, systems, and techniques described herein.

**License:** CC0

---

## Abstract

This document discloses a method and data format for carrying an analytic, vector‑based scene description alongside an otherwise standard AV1 video bitstream, with deterministic rasterization rules and per‑frame transform deltas. The side channel is transported in AV1 Metadata OBUs and/or container‑level timed metadata. Legacy decoders ignore the metadata. Enhanced decoders either re‑render vector layers for presentation or feed the vector field into encoder/decoder guidance (e.g., motion candidate seeding, warped‑motion hints, region‑of‑interest maps, reference synthesis) while producing an AV1‑conformant decoded picture. The disclosure includes on‑wire structure, numeric conventions, resource bounds, error handling, conformance points, and reference raster semantics sufficient for a person skilled in the art to implement interoperable encoders and decoders.

## 1. Motivation and Scope

AV1 is a block‑based hybrid video codec designed for pixels. Many modern sources—UI captures, cel/limited animation, titles, lower thirds, vector cartoons—originate as deterministic vector scenes with layer transforms and simple shading. Traditional pipelines rasterize to full frames and let the encoder rediscover structure through motion estimation and intra prediction. This wastes bitrate and compute, and it produces avoidable artifacts (e.g., chroma fringing on colored text, gradient banding).

The disclosed method introduces a compact, deterministic side channel describing the vector scene and its evolution over time. The base AV1 bitstream remains unchanged and fully standards‑compliant. The side channel enables three behaviors: (a) being ignored entirely by legacy decoders; (b) being used at presentation time to draw vector layers at display resolution as an overlay or replacement; and (c) being used internally by encoders/decoders/transcoders as guidance to reduce search, improve prediction, and bound memory. The scope explicitly includes a dual‑track option that embeds the original authoring source (e.g., Lottie JSON) as documentation/provenance, without making it part of the normative decode path.

## 2. Prior Art Context (Non‑Exhaustive)

Block codecs have long carried side metadata (e.g., film grain synthesis parameters, HDR dynamic metadata). Container formats support timed metadata tracks. MPEG‑4 BIFS/LASeR and similar systems defined full scene‑graph codecs separate from block‑based video. None of these, as used with AV1, combine a deterministic vector description with per‑frame affine deltas, a tile‑aligned dirty map, and explicit encoder/decoder guidance hooks while preserving baseline AV1 conformance. This disclosure establishes an AV1‑specific mapping and the concrete rules required to implement it.

## 3. Architectural Overview

The system consists of a conventional AV1 elementary stream and a synchronized Vector Scene Metadata (VSM) stream. The AV1 elementary stream carries pixels and decodes unchanged on all devices. The VSM stream is transported in AV1 Metadata OBUs (and optionally mirrored as a container metadata track). An enhanced player obtains both streams; when only the AV1 stream is present, behavior is identical to legacy playback.

At scene start, the encoder emits a VSM keyframe describing the static object table (paths, paints, initial transforms, z‑order, optional glyph resources) and fixed rasterization rules. Each subsequent coded frame emits a VSM delta containing only the changes necessary to advance the scene: typically affine matrices and visibility flags for objects, and occasionally paint deltas. A tile‑aligned dirty‑region map accompanies each delta to bound raster work and to enable encoder ROI steering. The enhanced renderer uses the VSM to draw vector layers into Y′CbCr tiles on demand, aligned with AV1’s tile grid, and composites them over or in place of the decoded pixels according to the chosen conformance profile.

## 4. Deterministic Rasterization Semantics

Interoperability requires that a given VSM payload produce the same pixels on any conforming implementation. The following rules are normative for Tier‑B/C renderers:

**Coordinate system and numerics.** The canvas coordinate space is device‑space pixels with origin at the upper‑left, x increasing rightward, y downward. All path vertices and affine matrices are represented in fixed‑point Q16.16 unless otherwise stated. When converting to device pixels, implementations must use round‑to‑nearest with ties to even. Affine transforms multiply in parent‑to‑child order; the canonical 2×3 matrix [a b c; d e f] maps (x, y, 1) to (a·x + b·y + c, d·x + e·y + f).

**Paths and fills.** Paths consist of moveTo, lineTo, and cubic Bezier segments. Arcs are flattened to cubic segments during compilation and do not appear on‑wire. The fill rule is even‑odd. Rasterization is edge‑inclusive with half‑pixel coverage conventions matching a supersample‑then‑box filter model with an effective 1‑pixel box. Implementations may use analytic coverage, but results must match reference supersampling to within ±1 LSB in the final bit depth after quantization.

**Stroking.** Strokes are centered on the path with fixed cap styles (butt, round, square) and join styles (miter with miter‑limit, bevel, round). The stroke width is given in canvas pixels (Q16.16). All stroking is performed in post‑transform space; that is, the path is transformed by the current affine, then strokes are computed in device space.

**Color and transfer.** All colors are specified in the video primaries signaled for the AV1 stream (e.g., BT.709 or BT.2020) with a linear segment/sRGB‑like transfer. Gradient interpolation must occur in linear light in the specified color space. After rasterization, values are quantized to the internal working depth (at least 10 bits per component). If the AV1 stream’s luma/chroma signal is 8‑bit 4:2:0, the renderer must dither deterministically when reducing precision.

**Chroma placement and subsampling.** When rendering directly into 4:2:0, chroma samples represent the average of a 2×2 luma sample region in co‑sited convention. Implementations may render in 4:4:4 and downsample using a fixed 6‑tap separable filter defined in this document, but the result must be bit‑identical to the reference filter. The reference downsampler uses taps {1, 4, 6, 4, 1}/16 applied in linear light before quantization.

**Gradients.** Linear and radial gradients are supported. Stop positions are fixed‑point Q0.12 along the gradient parameter. Colors at stops are defined as above. Interpolation between stops is linear in parameter t with no implicit premultiplication. Radial gradients define inner and outer circles; conic/mesh gradients are not included in the baseline.

**Masks and compositing.** The only required Porter–Duff operator is source‑over. Masks may be applied per object via an alpha mask object, which is itself a fill of a path hierarchy rendered to a single‑channel coverage plane at the same internal precision. The mask is multiplied into the source alpha before compositing.

**Dithering.** When reducing bit depth, implementations must apply ordered dithering using the fixed 8×8 Bayer matrix scaled to the target LSB. The dither phase is anchored to canvas coordinates so that identical content yields identical pixels independent of tile partitioning.

These rules are sufficient to guarantee pixel stability across conforming renderers while remaining implementable atop common vector stacks such as Skia or rlottie.

## 5. Data Model and Profiles

The VSM stream has two levels: a compiled, deterministic core that all enhanced decoders must understand, and an optional embedded authoring source that smart renderers may use for higher fidelity text or author‑intended effects. The compiled core is the normative source of truth for encoder/decoder guidance and for baseline vector re‑rendering. The authoring source is advisory.

### 5.1 Compiled Core (Normative)

At scene start, a keyframe defines the canvas, colorimetry, and a fixed object table. Each object specifies a stable identifier, z‑order, a path tree, a paint (solid or gradient), optional stroke parameters, an initial affine, and an optional mask reference. The keyframe also defines global resource tables, such as gradient definitions and optional glyph outlines. Subsequent deltas carry only changes: primarily new affine matrices and visibility flags, plus occasional paint or stroke adjustments. Every delta includes a tile‑aligned dirty‑region map. All numerics are fixed‑point, and the grammar is closed and bounded.

### 5.2 Embedded Authoring Source (Advisory)

The bundle may optionally include the original authoring asset, for example, a Lottie JSON file. This is carried as length‑prefixed chunks within the same metadata stream, compressed with a general‑purpose compressor (e.g., Brotli). The presence of the source does not change conformance. Smart renderers may prefer to render from the authoring source for improved font shaping, hinting, or subpixel accuracy, but they must honor safe areas and produce pixels that are visually consistent with the compiled core.

## 6. On‑Wire Structure

The VSM stream is transported in AV1 Metadata OBUs with a dedicated metadata type assigned by the AV1 specification authority or, alternatively, encapsulated in an ITU‑T T.35 user data payload with a registered provider code. The payload itself is encoded using CBOR for compactness and schema stability. Lengths use LEB128 or CBOR’s native length prefixes.

A VSM bundle consists of three message kinds:

• **Bundle‑Init** appears once at the start of a scene and contains the compiled keyframe and optional source metadata preview (size, hash, compression).
• **Source‑Chunk** appears zero or more times to carry the compressed authoring asset in sequence‑numbered chunks.
• **Delta** appears for each coded frame in decode order and carries transforms, style deltas, and the dirty‑tile map.

A scene is identified by a monotonically increasing `scene_id`. Each message includes the `scene_id` and a CRC for transport integrity. The delta includes the decode order frame index and, optionally, the presentation timestamp for containers that require it.

## 7. CBOR Schema (CDDL)

For clarity, the following CDDL describes the CBOR structures. This schema is informative; any representation producing equivalent fields and semantics is acceptable.

```
vsm-bundle = vsm-init / vsm-src-chunk / vsm-delta

vsm-init = {
  type:          "vsm_bundle_init",
  scene-id:      uint,
  profile:       tstr,              ; e.g., "vsm-lite-1"
  canvas:        canvas,
  color:         color,
  keyframe:      vsm-keyframe,
  src-present:   bool,
  ? src-meta:    src-meta
}

canvas = { w: uint, h: uint, tile-w: uint, tile-h: uint }
color  = { primaries: tstr, matrix: tstr, bitdepth: uint }

vsm-keyframe = {
  objects:       [* vsm-object],
  resources:     resources
}

vsm-object = {
  id:            uint,
  z:             int,
  path:          path,
  paint:         paint,
  ? stroke:      stroke,
  affine:        affine,
  ? mask-id:     uint,
  visible:       bool
}

affine = [ six-int ]
six-int = int, int, int, int, int, int   ; Q16.16 fixed-point

path = { kind: "cubic", pts: [* int] }  ; canonical cubic-only encoding

paint = solid / linear-grad / radial-grad
solid = { type: "solid", r: uint, g: uint, b: uint, a: uint }

linear-grad = {
  type: "linear",
  x0: int, y0: int, x1: int, y1: int,    ; Q16.16
  stops: [* grad-stop]
}
radial-grad = {
  type: "radial",
  cx0: int, cy0: int, r0: int,
  cx1: int, cy1: int, r1: int,
  stops: [* grad-stop]
}

grad-stop = { pos: uint, r: uint, g: uint, b: uint, a: uint } ; pos Q0.12

stroke = { w: int, join: tstr, cap: tstr, miter: uint }

resources = {
  ? glyphs: [* glyph],
  ? symbols: [* symbol]
}

glyph = { gid: uint, outline: path }   ; simplified
symbol = { sid: uint, path: path }

src-meta = {
  mime:       tstr,          ; e.g., "application/lottie+json"
  encoding:   tstr,          ; "br", "zst", or "raw"
  uncompressed: uint,
  sha256:     bstr
}

vsm-src-chunk = {
  type:       "vsm_bundle_src_chunk",
  scene-id:   uint,
  seq:        uint,
  crc16:      uint,
  payload:    bstr
}

vsm-delta = {
  type:       "vsm_delta",
  scene-id:   uint,
  frame-idx:  uint,
  ? pts:      uint,
  xforms:     [* xform],
  style:      [* style-delta],
  dirty:      bstr            ; RLE or bitmap of tiles
}

xform = { id: uint, affine: affine, vis: bool }

style-delta = {
  id: uint,
  ? paint: paint,
  ? stroke: stroke
}
```

## 8. Transport Mapping in AV1 and Containers

In AV1 elementary streams, VSM messages are carried in Metadata OBUs. Each VSM message forms one metadata OBU payload. The metadata type is a dedicated value assigned to "VSM\_BUNDLE"; until assignment, implementations may use the ITU‑T T.35 metadata type with an agreed provider code and sub‑payload identifier of "VSM". The OBU payload is the CBOR‑encoded message. Bundle‑Init must appear before the first coded frame of the scene. Source‑Chunk messages may appear at any point before the first frame that requires the embedded source; they are purely advisory and may be omitted entirely. Delta messages must appear with the coded frame they describe and are identified by `frame-idx` in decode order.

In ISO BMFF/MP4, a mirror track of timed metadata is recommended. Each sample carries one VSM message in the same CBOR form. The track is synchronized to the video via composition time. In Matroska/WebM, the metadata may be carried as BlockAdditions or as an additional track with lacing disabled. In DASH/HLS, the VSM can also be transported using `emsg` events or sidecar segments.

## 9. Conformance Tiers

Two independent axes of conformance are defined. **Decode conformance** requires that the decoder produce the normative AV1 decoded picture independent of VSM; all existing AV1 decoders meet this. **Presentation conformance** defines three tiers for VSM‑aware implementations.

Tier A produces the decoded AV1 picture and ignores VSM. Tier B consumes the VSM and re‑renders vector layers into Y′CbCr tiles according to the rasterization rules and composites them over or in place of the decoded pixels at presentation time. Tier C consumes the VSM for presentation as in Tier B and additionally uses the data to guide predictive tools when encoding or transcoding: it seeds motion candidates, supplies warped‑motion fields, sets region‑of‑interest maps, and synthesizes forward references. Tier C still outputs bitstreams and decoded pictures that conform to AV1; guidance influences encoder decisions but not the normative decoder state.

## 10. Encoder and Transcoder Guidance

Encoders benefit from VSM even when the decoder ignores it. A conforming encoder extracts a block‑wise motion field from the per‑object affine transforms and installs these vectors as first‑class candidates in motion estimation. Where an affine model better explains the local motion than pure translation, the encoder promotes warped‑motion candidates pre‑initialized from the VSM. Rate‑distortion costs are biased to prefer candidates closer to the VSM field. The dirty‑tile bitmap is mapped to the encoder’s segmentation or ROI mechanism so that blocks outside the active region are skipped or encoded with coarser QP. Palette and Intra Block Copy modes are promoted within dirty tiles whose paints and masks indicate limited color variety and repeat structures.

When the encoder maintains a short lookahead, it uses the VSM transforms to synthesize a forward reference by warping the last reconstructed picture into the next frame’s predicted pose. This forward reference functions like an alt‑ref image in temporal filtering pipelines and typically reduces residual energy for vector motion. All of these behaviors are internal to the encoder and do not affect conformance; they simply improve compression and reduce search.

## 11. Tile Alignment and Memory Discipline

The VSM canvas declares a tile grid aligned with the AV1 tile grid. The delta message carries a dirty‑tile map in that grid. Enhanced renderers render only those tiles whose bits are set for a given frame; other tiles are drawn from the decoded AV1 picture. Implementations may further subdivide tiles internally for parallelism, but the observable dither phase and chroma placement must remain anchored to the declared canvas to ensure seamless joins at tile boundaries. This approach bounds working memory to a small number of tiles plus reference rows, avoids full‑frame staging, and enables zero‑copy paths to hardware encoders via DMABUF when present.

## 12. Error Handling and Fallback

If a VSM message is missing, corrupted, or fails integrity checks, the implementation must ignore the VSM for that frame and display the decoded AV1 picture alone. If the Bundle‑Init is missing or malformed, the entire VSM stream for the scene is ignored. If the embedded authoring source chunks are incomplete, smart renderers fall back to the compiled core. These rules guarantee forward progress and avoid visible corruption.

## 13. Resource Limits and Security Considerations

To prevent denial‑of‑service, this profile imposes hard bounds that a decoder may enforce without negotiation. A scene may declare at most 4096 objects and at most 200,000 path segments total. A gradient may have at most eight stops. The cumulative uncompressed size of the authoring source must not exceed a fixed cap such as 512 KiB per scene. The rasterizer must validate all integer ranges on input and avoid dynamic allocation proportional to untrusted values. Dithering and downsampling filters are fixed and do not depend on content.

Because VSM is declarative and non‑programmatic, there is no script execution. Glyph shaping, if supported via embedded fonts, must be performed with a deterministic shaping engine and must be reduced to outlines before rasterization. Implementations should sandbox the metadata parser consistent with general media‑parsing best practices.

## 14. Interoperability With Authoring Formats

The VSM core is deliberately narrower than common authoring formats. Implementations are expected to provide an importer that compiles a subset of authoring formats such as Lottie into the VSM keyframe and deltas. The importer eliminates expressions, effects, and non‑deterministic features, flattens precompositions into a single object table, converts text to outlines or embeds a minimal font subset, canonicalizes paths to cubic Beziers, and quantizes numerics to fixed‑point. The resulting VSM is stable and codec‑friendly. The original source may be carried as advisory chunks for documentation, provenance, and enhanced presentation on devices that choose to parse it.

## 15. Reference Raster Interface

A minimal reference rasterizer exposes a tile‑centric procedure. Given time `t`, a tile rectangle `(x, y, w, h)`, and pointers to destination Y, Cb, and Cr planes with strides, the procedure evaluates the object table in z‑order, computes each object’s device‑space geometry by applying the current affine, calculates coverage for filled paths and strokes, evaluates paint and masks in linear light, composites with source‑over, and writes luma and chroma samples consistent with the specified subsampling and downsampling rules. The function returns without touching pixels outside the tile rectangle. A pseudocode signature:

```
void vsm_render_tile(const VsmScene* s,
                     int64_t t,                 // decode-order frame index or PTS ticks
                     int x, int y, int w, int h,
                     uint16_t* Y, int y_stride,
                     uint16_t* Cb, int cb_stride,
                     uint16_t* Cr, int cr_stride);
```

An encoder integrates this renderer by issuing calls for tiles flagged dirty in the VSM delta immediately prior to coding those tiles. A transcoder capable of GPU acceleration renders tiles into GPU surfaces and passes them by handle to a hardware encoder, avoiding system memory copies.

## 16. Worked Example (Illustrative)

Consider a 1920×1080 canvas with 8×8 tile grid (240×135 tiles). The keyframe declares two objects: a rounded rectangle with a solid fill at z=0 and a text outline path at z=1 with a linear gradient stroke. The initial affine for the text places it at the left edge of the screen. The first delta sets the text visible and supplies an affine that translates it rightward by 4 pixels. The dirty‑tile bitmap contains only the tiles intersecting the text’s bounding box. An enhanced encoder maps those tiles to active regions, seeds translational vectors in blocks overlapping the text, and attempts Intra Block Copy where the text outline self‑overlaps. A Tier‑B renderer re‑rasterizes only the text tiles per frame, composites them over the decoded background, and outputs 4:2:0 with ordered dithering.

## 17. Versioning and Extensibility

The VSM profile string identifies a closed set of features. Unknown profiles are ignored. Within a profile, extensions are negotiated by adding optional fields with default behavior that preserves determinism. New paint types (e.g., image patterns) and blend modes can be added in future profiles with clear fallback rules. The metadata type identifier in AV1 distinguishes VSM from other metadata and allows multiple independent metadata streams to co‑exist.

## 18. Conformance Testing

A conformance suite consists of VSM/AV1 pairs with golden Y′CbCr outputs for several representative scenes: flat UI elements, moving vector characters, animated gradients, masked compositions, and text. Implementations must match golden pixels to within ±1 LSB at the declared bit depth after the required dithering and downsampling. For presentation conformance, a pixel map indicates which regions are subject to exact match and which are subject to perceptual thresholds when fonts are rendered from source rather than outlines. The suite also includes pathological but bounded inputs to validate resource limits and error handling.

## 19. Implementation Sketch

A minimal open‑source reference comprises a `libvsm` parser and renderer, an FFmpeg filter to burn VSM into pixels for legacy targets, and patches to a reference AV1 encoder to accept VSM‑derived active maps and motion seeds. The parser validates CBOR, enforces resource limits, builds a compact object table, and maintains per‑frame affine state. The renderer implements the deterministic rules above and provides both CPU and GPU backends. The encoder plumbing is straightforward: the tile‑aligned dirty bitmap maps to the encoder’s region control, and the per‑block motion field derived from object affines is introduced as ordered candidates with small negative biases in the mode decision cost function. A forward reference is synthesized by warping the last reconstructed frame using the scene’s transforms where applicable; occlusions are handled by standard compositor ordering.

## 20. Legal Notice and License

This document is a public, enabling defensive disclosure. It dedicates the disclosed ideas and their combinations to the public domain under Creative Commons CC0 1.0. Anyone may implement the techniques herein. The author asserts no proprietary claim and intends that this publication constitute prior art against any later‑filed patent applications covering the same or obvious variations of the described methods.

**CC0 1.0 Universal (CC0 1.0) Public Domain Dedication**
To the extent possible under law, the author has waived all copyright and related or neighboring rights to this work. A copy of the dedication may be found at [https://creativecommons.org/publicdomain/zero/1.0/](https://creativecommons.org/publicdomain/zero/1.0/) .

## 21. Submission Guidance (Informative)

For maximum defensive effect, submit this disclosure to multiple indexed repositories: an arXiv or TechRxiv preprint with the abstract and keywords explicitly mentioning "AV1" and "Vector Scene Metadata"; a public Git repository containing the schema and example payloads; and a message to the AOM public mailing list linking the preprint and attaching the CBOR schema and examples. Optionally, create an IETF Internet‑Draft in the `informational` stream to place the disclosure in the IETF archive. Mirror the content on an IP.com prior‑art record.

## Appendix A: Example Messages (JSON Readable Form)

**Bundle‑Init**

```json
{
  "type": "vsm_bundle_init",
  "scene_id": 1,
  "profile": "vsm-lite-1",
  "canvas": {"w": 1920, "h": 1080, "tile_w": 240, "tile_h": 135},
  "color": {"primaries": "bt709", "matrix": "bt709", "bitdepth": 10},
  "keyframe": {
    "objects": [
      {
        "id": 10,
        "z": 0,
        "path": {"kind":"cubic","pts":[/* Q16.16 ints */]},
        "paint": {"type":"solid","r":64,"g":64,"b":64,"a":1023},
        "stroke": {"w": 131072, "join":"miter","cap":"butt","miter": 4096},
        "affine": [65536,0,0,65536,0,0],
        "visible": true
      }
    ],
    "resources": {}
  },
  "src_present": true,
  "src_meta": {
    "mime": "application/lottie+json",
    "encoding": "br",
    "uncompressed": 148532,
    "sha256": "Oe8…base16…=="
  }
}
```

**Source‑Chunk**

```json
{
  "type": "vsm_bundle_src_chunk",
  "scene_id": 1,
  "seq": 0,
  "crc16": 52341,
  "payload": "<bytes>"
}
```

**Delta**

```json
{
  "type": "vsm_delta",
  "scene_id": 1,
  "frame_idx": 42,
  "xforms": [{"id": 10, "affine": [65536,0,0,65536,2048,0], "vis": true}],
  "style": [],
  "dirty": "RLE:AAECAwQF…"
}
```

## Appendix B: Reference Downsampler and Dither

The reference downsampler from 4:4:4 linear light to 4:2:0 co‑sited chroma applies a separable 1‑4‑6‑4‑1 filter horizontally and vertically. The ordered dither uses the standard 8×8 Bayer matrix scaled to the target LSB and applied independently to Y, Cb, and Cr with the same phase anchored at (0,0). Implementations may vectorize these operations, but observable pixels after quantization must match the reference to within ±1 LSB.

## Appendix C: Security Checklist

All integer inputs are range‑checked prior to allocation. Path command sequences are validated for structural correctness. Gradient stop counts are bounded and strictly increasing in position. CRCs are validated for each metadata message. Failure of any check results in ignoring the VSM for that frame without aborting the overall decode. The authoring source, if present, is decompressed into a bounded buffer with a streaming API that enforces the declared size.

## Appendix D: Glossary

**AV1:** AOMedia Video 1, a royalty‑free video codec.
**Metadata OBU:** AV1 bitstream unit carrying non‑pixel metadata.
**VSM:** Vector Scene Metadata, the compiled, deterministic side channel disclosed here.
**Dirty‑tile map:** A bitmap of tiles that changed this frame.
**Tier A/B/C:** Presentation conformance levels described in Section 9.
