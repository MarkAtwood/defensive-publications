# Defensive Publication: In‑Band Vector Scene Metadata for AV1 - Rev 3

**Publication Date:** 2025‑08‑12

**Author:** Mark Atwood (Review Commit, LLC)

**Purpose:** This document serves as a defensive publication to establish prior art and prevent future patent claims on the methods, systems, and techniques described herein.

**License:** CC0

---

## 1. Abstract

This defensive publication discloses **Vector Scene Metadata (VSM)**: a compact, deterministic, vector‑graphics scene description carried **in‑band** with an otherwise standard AV1 elementary stream. VSM co‑travels with coded video frames as metadata, describing paths, paints, strokes (with miter‑limit), affine transforms, masks, text runs and glyph outlines, and per‑frame deltas (affine/visibility plus a tile‑aligned dirty map). Legacy AV1 decoders ignore VSM and remain fully conformant. VSM‑aware systems may: (a) use the scene truth to **guide encoders** (ROI maps, MV seeds, warped‑motion hints, skip/reuse), (b) **re‑rasterize** vector layers at presentation for resolution‑independent sharpness, and/or (c) expose the scene for accessibility and graphics interop.

This superset revision harmonizes and consolidates prior write‑ups into a single normative deliverable and incorporates: true miter joins with miter‑limit wedges; **Porter–Duff** compositing operators (`src-over`, `dst-over`, `plus`, `xor`); blend modes (`normal`, `multiply`, `screen`, `overlay`); HarfBuzz/FreeType **text shaping** and glyph‑outline import; **8× supersample** conformance raster; **limited‑range Y′CbCr** signaling; explicit **chroma downsampling** (1‑4‑6‑4‑1) and **ordered dithering** (anchored 8×8 Bayer); and container helpers for AV1 OBU metadata, MP4 `emsg`, and Matroska `BlockAdditions`.

---

## 2. Motivation and Scope

Most 2D sources (UI, titles, cel animation, vector cartoons) originate as **vectors** with simple paint and affine transforms. Rasterizing fully before compression discards structure the encoder could exploit, wasting bits and compute and degrading text/edges. VSM exposes the original structure **without** altering AV1 decoding semantics.

**Objectives**

- Preserve object identity, order, transforms, and paint over time.
- Drive encoder decisions deterministically from scene truth.
- Enable resolution‑independent, tile‑local re‑rasterization at playback.
- Remain strictly optional and non‑normative to AV1 decoding.

---

## 3. Architecture

A standard AV1 bitstream is accompanied by a synchronized **VSM metadata stream** carried in AV1 **OBU\_METADATA** units (and optionally mirrored at the container level). A **Bundle‑Init** message fixes the canvas, colorimetry, object table, and resources (glyphs). Each coded frame carries a **Delta** with per‑object affine/visibility updates and a **tile‑aligned dirty bitmap**. Optionally, the original authoring asset (e.g., **Lottie**) is chunked as advisory side data; the **compiled core** is normative.

---

## 4. Deterministic Rasterization (Normative)

The following rules guarantee pixel‑stable results across conforming implementations.

### 4.1 Coordinates and Numerics

- Canvas space is pixel‑aligned, origin top‑left, x→right, y→down.
- Path vertices and affine entries are **Q16.16** signed fixed‑point; gradient stop positions are **Q0.12**.
- Round‑to‑nearest‑even when converting to device integer pixels.
- Affine `[a b c; d e f]` maps `(x,y,1) → (a·x + b·y + c, d·x + e·y + f)`; apply parent‑to‑child.

### 4.2 Paths, Fills, Strokes

- Paths: moveTo/lineTo/**cubic** Bezier; arcs/elliptics are pre‑canonicalized to cubics.
- Fill rule: **even‑odd**.
- Stroke: device‑space width `w` (Q16.16); caps `{butt,round,square}`; joins `{miter,bevel,round}` with **miter‑limit**. For join angle θ, miter length `L = w/(2·sin(θ/2))`. If `L > miter_limit·w/2` → **bevel**; else fill the **wedge** from offset edges to apex.
- Antialiasing: **supersample** `S×S` per pixel; default `S=4`, *conformance* `S=8`.

### 4.3 Paint and Gradients

- Color is **linear‑light** in the signaled video primaries (e.g., BT.709/BT.2020). Gradient interpolation occurs in linear light.
- Paints: `solid` RGBA; `linear`(x0,y0→x1,y1) with stop list; `radial` two‑circle (C0,r0→C1,r1) solved by bisection over t∈[0,1] (24 iterations).

### 4.4 Compositing

- Compute source pre‑multiplied color (coverage × alpha × paint).
- Apply **blend** to destination (`normal`,`multiply`,`screen`,`overlay`) component‑wise in linear light.
- Then apply **Porter–Duff**: `src-over` (default), `dst-over`, `plus` (additive; alpha saturates), `xor`. Math uses pre‑multiplied RGBA.
- Optional `mask_id` multiplies source alpha by mask coverage prior to compositing.

### 4.5 Y′CbCr, Subsampling, Dither

- Convert linear RGB→**Y′CbCr(709)**. Quantize to the working bit depth (≥10). If output is **limited‑range**, map to [16..235] (Y) and [16..240] (Cb/Cr) at 8‑bit equivalence.
- For 4:2:0 rendering, either render 4:4:4 and downsample with the **separable 1‑4‑6‑4‑1** filter (linear light), or render natively into co‑sited chroma with the same effect. Results must match the reference filter within ±1 LSB after quantization.
- Apply **ordered dithering** with fixed **8×8 Bayer** matrix, phase anchored at (0,0) in canvas space (same phase for Y, Cb, Cr) before final quantization.

---

## 5. Data Model (Normative)

### 5.1 Canvas

`{ w, h, tile_w, tile_h, color{bitdepth, primaries, matrix, limited_range} }`

### 5.2 Objects

Each object has: `id:u32`, `z:s32`, `path:cubic?`, `paint`, `stroke?{w,join,cap,miter}`, `blend`, `comp`, `affine[6]`, `visible:bool`, `mask_id?`, and optional **text run**: `text.run[{gid, dx, dy}]` (Q16.16). Paths encode as cubic segments `[x0,y0, cx1,cy1, cx2,cy2, x3,y3]*`.

### 5.3 Resources

`resources.glyphs[{gid, outline:path}]` (cubic outlines). Optionally, embed original authoring source as advisory chunks with `{mime, encoding, uncompressed, sha256}` metadata.

### 5.4 Messages

- **Init** `vsm_bundle_init`: `{scene_id, profile, canvas, keyframe{objects[], resources}, src_present?, src_meta?}`.
- **Delta** `vsm_delta`: `{scene_id, frame_idx, xforms[{id,affine,vis}], style[], dirty: bytes}` where `dirty` is tile bitmap (row‑major) or RLE.

### 5.5 Encoding

**CBOR** (RFC 8949), canonical, definite‑length; ASCII lower‑case keys; unknown keys ignored.

---

## 6. Transport (Normative)

### 6.1 AV1 Elementary Stream

- Each VSM message appears in an **OBU\_METADATA** with a dedicated `metadata_type` (AOM allocation). Payload = CBOR message; size = LEB128. Messages are interleaved at access‑unit boundaries **before** the coded frame they describe.

### 6.2 ISO BMFF/MP4

- Timed metadata track samples; or `emsg` v0 with `scheme_id_uri="urn:org.aomedia:vsm:1"` and `value` selecting message type; `message_data` = CBOR.

### 6.3 Matroska/WebM

- `BlockAdditional` or a dedicated track with no lacing; payload = CBOR; registered `BlockAddID` for VSM.

---

## 7. Conformance and Profiles

- **Decode conformance**: AV1 decoding is unaffected; all existing decoders conform by ignoring VSM.
- **Presentation conformance** tiers:
  - **Tier A**: ignore VSM; display decoded AV1.
  - **Tier B**: consume VSM; re‑rasterize tiles per Section 4; composite over/in place of decoded pixels.
  - **Tier C**: Tier B + **encoder guidance** (ROI, MV seeds, warped‑motion hints, forward‑ref synthesis) while preserving AV1 conformance.

**Profiles**:

- ``: fills, solids + linear gradients, text runs allowed, 4× SS, full‑range Y′CbCr.
- ``: adds radial gradients, strokes with miter‑limit, masks, blend modes, Porter–Duff ops, 8× SS, limited‑range signaling, glyph resources.

---

## 8. Encoder/Transcoder Guidance (Normative for Tier C)

1. **ROI/QP maps** from dirty tiles: ROI ΔQP≈−6; one‑tile ring ΔQP≈−3; others 0 (clip to encoder limits).
2. **MV seeds** from object affines: for each block center `(x,y)` in an object’s coverage, compute `(x′,y′)` under the affine and seed `MV = round(((x′−x, y′−y)/Δt)/MV_unit)` into translational ME and merge lists. Promote local **warped‑motion** where fit is good.
3. **Skip/reuse**: prefer skip for unchanged object coverage/style; bias RD to reuse.
4. **Forward reference**: synthesize by warping the last reconstructed frame under the declared transforms for lookahead frames; treat like alt‑ref.
5. **Palette/IBC**: within ROI tiles, promote palette mode and Intra Block Copy guided by paint/mask homogeneity.

---

## 9. Error Handling, Limits, Security

- Missing/malformed VSM for a frame → ignore VSM, display AV1 picture; malformed **Init** → ignore VSM for the scene.
- Hard bounds (enforce without negotiation): ≤4096 objects, ≤200k path segments, ≤8 gradient stops per gradient, ≤512 KiB advisory source per scene.
- All integer inputs range‑checked; paths validated; gradient stops strictly increasing. No scripts; shaping results reduced to outlines for raster. Parser sandboxing recommended.

---

## 10. Interop With Authoring Formats (Informative)

Importers compile a subset of Lottie/other vector sources to the compiled core: eliminate expressions/effects, flatten pre‑comps, canonicalize to **cubic** segments, quantize to fixed‑point, shape text via HarfBuzz, and either embed a minimal font subset (for shaping at ingest) or convert to outlines (for the core). The original source may be embedded as advisory chunks for provenance and smarter renderers.

---

## 11. Wire Grammar (Readable CBOR Schema)

```
init := {
  "type":"vsm_bundle_init",
  "scene_id": uint,
  "profile": text,
  "canvas": {"w":uint,"h":uint,"tile_w":uint,"tile_h":uint,
              "color":{"bitdepth":uint,"primaries":text,"matrix":text,"limited_range":bool}},
  "keyframe": {"objects":[object,...], "resources":{"glyphs":[{"gid":uint,"outline":path},...]}},
  "src_present": bool?,
  "src_meta": {"mime":text,"encoding":text,"uncompressed":uint,"sha256":text}?
}

delta := {
  "type":"vsm_delta", "scene_id":uint, "frame_idx":uint,
  "xforms":[{"id":uint,"affine":[int×6],"vis":bool}],
  "style": [...],
  "dirty": bstr
}

object := {"id":uint,"z":int, "path":path?, "paint":paint,
  "stroke": {"w":int,"join":text,"cap":text,"miter":uint}?,
  "blend": text,           # normal|multiply|screen|overlay
  "comp": text,            # src-over|dst-over|plus|xor
  "affine": [int×6], "visible":bool, "mask_id":uint?,
  "text": {"run":[{"gid":uint,"dx":int,"dy":int},...]}? }

path := {"kind":"cubic","pts":[int,...]}    # Q16.16 pairs per cubic segment
paint := solid | linear | radial
solid := {"type":"solid","r":uint,"g":uint,"b":uint,"a":uint}
linear := {"type":"linear","x0":int,"y0":int,"x1":int,"y1":int,
           "stops":[{"pos":uint,"r":uint,"g":uint,"b":uint,"a":uint},...]}
radial := {"type":"radial","cx0":int,"cy0":int,"r0":int,
           "cx1":int,"cy1":int,"r1":int,
           "stops":[{"pos":uint,"r":uint,"g":uint,"b":uint,"a":uint},...]}
```

Encoding: canonical CBOR; definite lengths; lexicographic keys; unknown keys ignored.

---

## 12. Reference Raster Interface (Informative)

```
void vsm_render_tile(const VsmScene* s,
                     int64_t t,
                     int x,int y,int w,int h,
                     uint16_t* Y,int y_stride,
                     uint16_t* Cb,int cb_stride,
                     uint16_t* Cr,int cr_stride);
```

Tile‑centric; renders only dirty tiles; preserves anchored dither/chroma phase at tile boundaries. CPU or GPU back‑ends acceptable so long as Section 4 pixels match to ±1 LSB after quantization.

---

## 13. Conformance Testing (Informative)

Provide AV1/VSM pairs with golden Y′CbCr for: (1) flat UI elements, (2) moving vector characters, (3) animated gradients, (4) masked comps, (5) text (shaped + outlines). Pixel match within **±1 LSB** at declared bit depth after downsample/dither. For source‑driven text, designate perceptual‑tolerance regions.

---

## 14. Worked Example (Informative)

A 1920×1080 canvas with 240×135 tiles. Keyframe: z=0 rounded rect fill; z=1 text outlines with a linear‑gradient stroke. Delta frame translates text by +4 px in X; the **dirty bitmap** sets only intersecting tiles. Encoder maps ROI/QP, seeds translational MVs for blocks overlapping the text, promotes IBC where outlines repeat. Tier‑B renderer re‑rasterizes just the text tiles per frame and composites over decoded background.

---

## 15. Variations (Placed in Public Domain)

- Alternative vector segment bases (lines/quadratics/arcs) with cubic canonicalization.
- Additional blends/Porter–Duff ops and gradient types (conic, diamond, mesh).
- Alternate fixed‑point precisions, sampling lattices, and ROI heuristics.
- Alternate container bindings (CMAF events; timed ID3) with identical CBOR.
- GPU rasterizers or hardware blocks implementing the same compositing semantics.

---

## 16. Claim‑Style Enumeration (Searchable, Non‑legal)

1. In‑band carriage of a vector scene model synchronized to AV1 frames such that unaware decoders ignore it while aware systems use it for guidance and/or presentation.
2. An initialization bundle and per‑frame deltas with object affines/visibility and a tile‑aligned dirty map.
3. Per‑object description including cubic path, paint (solid/linear/radial), stroke with miter‑limit, blend mode, and Porter–Duff operator.
4. Text runs as glyph IDs with advances, plus embedded glyph outlines as resources and optional advisory authoring source.
5. A deterministic raster with even‑odd fills, **S×S** supersampling, two‑circle radial solver, miter‑limit wedges, linear‑light blending, fixed Bayer dithering, and fixed 1‑4‑6‑4‑1 chroma downsampling.
6. Encoder guidance from transforms and dirty tiles to produce ROI/QP maps, motion‑vector seeds, skip decisions, warped‑motion hints, and forward‑reference synthesis.
7. Transport via AV1 **OBU\_METADATA**, MP4 `emsg`, and Matroska `BlockAdditions` carrying canonical CBOR.

---

## 17. Reference Implementation (Availability)

A permissive **libvsm** reference accompanies this disclosure (CC0): CBOR parser, deterministic raster (fills/strokes/masks; blends + Porter–Duff; two‑circle radial; 4×/8× SS; linear‑light Y′CbCr with limited/full range; text shaping and glyph outlines), and container packers (OBU/MP4/Matroska). A demo renders examples to YUV44416/PPM and iterates tiles.

---

## 18. Error Handling & Fallback (Normative)

Missing/corrupt per‑frame VSM → ignore VSM for that frame; missing/malformed **Init** → ignore VSM for the scene; incomplete advisory source → fall back to compiled core; never block decode or present corrupted overlays.

---

## 19. Legal Notice and License

This work is a public, enabling defensive disclosure. It dedicates the disclosed ideas and combinations to the public domain under **CC0 1.0**. The intent is to constitute **prior art** against later applications on the same or obvious variants.

---

## 20. Submission Guidance (Informative)

Publish to multiple indexed venues (e.g., TechRxiv/arXiv; public Git with schema/examples; AOM public list referencing this text). Optionally mirror as an IETF Informational Internet‑Draft and on IP.com. Record exact submission timestamps.

---

## 21. Change Log

- **Superset Rev 3 (2025‑08‑12):** Consolidated prior documents; added explicit chroma downsampling and ordered dithering; confirmed Porter–Duff ops, blend modes, HarfBuzz shaping, 8× SS, limited‑range signaling; harmonized transport and encoder‑guidance sections; integrated worked example and conformance text into one deliverable.
- **Rev 2:** Added miter‑limit joins; Porter–Duff compositing; multiply/screen/overlay blends; glyph shaping and outline import; 8× SS conformance raster; limited‑range Y′CbCr; container mux helpers; expanded test and normative sections.
- **Rev 1:** Initial disclosure with object model, deltas, dirty tiles, linear/radial gradients, encoder guidance, and AV1 OBU carriage.

