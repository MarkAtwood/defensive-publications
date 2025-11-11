# Defensive Publication: In‑Band Vector Scene Metadata for AV1 - Rev 2

**Publication Date:** 2025‑08‑12

**Author:** Mark Atwood (Review Commit, LLC)

**Purpose:** This document serves as a defensive publication to establish prior art and prevent future patent claims on the methods, systems, and techniques described herein.

**License:** CC0


---

## 1. Abstract

This disclosure places into the public domain a full, enabling description of **Vector Scene Metadata (VSM)** carried *in‑band* with AV1 elementary streams. VSM transports a compact vector scene model—paths, paints, strokes, transforms, text runs, glyph outlines, and per‑frame deltas—alongside conventional video frames. Unmodified AV1 decoders ignore VSM. VSM‑aware tooling exploits it to: (a) guide the encoder (ROI maps, motion‑vector seeds, skip decisions, transform hints) for higher compression and temporal stability on vector animation; (b) optionally re‑rasterize at playback for resolution‑independent sharpness; and/or (c) expose the vector scene for accessibility and graphics interop.

This rev includes: true miter joins with miter‑limit, Porter–Duff compositing operators (src‑over, dst‑over, plus, xor), blend modes (multiply, screen, overlay), glyph shaping/import via HarfBuzz/FreeType, 8× supersample conformance raster, limited‑range YCbCr signaling, and containerization helpers for AV1 OBU metadata, MP4 `emsg`, and Matroska `BlockAdditions`.

---

## 2. Problem and Rationale

Modern 2D animation pipelines originate as vectors (shapes, gradients, text, transforms). Conventional delivery rasterizes these to full frames prior to compression, discarding structure the encoder would exploit. AV1 encoders infer motion and region importance from pixels, which is wasteful for scenes already described by affine transforms, occlusion order, and sharp edges. VSM in‑band side data exposes this structure to encoders and VSM‑aware decoders without breaking compatibility.

**Goals**

- Preserve 2D scene semantics over time (objects, transforms, styles).
- Drive encoder decisions deterministically from scene truth.
- Permit resolution‑independent playback and super‑resolution rasterization.
- Remain strictly optional and non‑normative to AV1 decoding.

---

## 3. Overview of the Technique

A VSM stream defines:

1. **Canvas** and colorimetry (size, tiles, bit‑depth, limited/full range).
2. **Resources** (glyph outlines table; optional external source metadata).
3. **Objects**: each has a path (cubic), paint (solid/linear/radial), stroke (width, join, cap, miter‑limit), blend mode, Porter–Duff composite operator, affine transform, visibility, and optional text run referencing glyph IDs with advances.
4. **Keyframe** message initializes the entire scene.
5. **Delta** message per frame provides object transforms/visibility updates and a tile‑granular dirty bitmap. Optional style deltas are allowed.

Carriage is via **CBOR** (canonical, definite length). In AV1 elementary streams, VSM is embedded as **OBU\_METADATA** payloads. In MP4, VSM appears in `emsg` boxes; in Matroska/WebM, as `BlockAdditional` with a registered ID.

Unaware decoders ignore all VSM data. Aware encoders/players consume it to:

- Seed motion estimation with affine candidates per object.
- Generate ROI/QP maps from dirty tiles and object saliency (stroke/fill edges).
- Decide inter/intra reuse based on unchanged objects, enabling skip chains.
- Optionally re‑rasterize objects at the display resolution using the normative raster below.

---

## 4. Data Model (Normative)

### 4.1 Fixed‑Point and Color

- Coordinates, radii, and affine entries use **Q16.16** signed fixed‑point.
- Gradient stop positions use **Q0.12**.
- Colors use 10…16‑bit **linear‑light** RGBA per channel; storage is 16‑bit lanes with the active bit depth signaled at the canvas.
- Canvas signals `limited_range` (studio range) or full range for Y′CbCr write‑out.

### 4.2 Objects

Each object record contains:

- `id: u32`, `z: s32` (painter’s order; stable across frames).
- `path: cubic[]` where each segment is `[x0,y0, cx1,cy1, cx2,cy2, x3,y3]` in Q16.16.
- `paint`: `solid{r,g,b,a}` or `linear{x0,y0,x1,y1, stops[]}` or `radial{cx0,cy0,r0,cx1,cy1,r1, stops[]}`. Each stop: `{pos(Q0.12), color}`.
- `stroke`: `{width(Q16.16), join∈{miter,bevel,round}, cap∈{butt,square,round}, miter(Q8.8), present:bool}`.
- `blend`: `{normal,multiply,screen,overlay}` in linear light.
- `comp`: Porter–Duff `{src-over,dst-over,plus,xor}` on pre‑multiplied colors.
- `affine`: `[a,b,c,d,e,f]` (2×3 matrix).
- `visible: bool`.
- **Optional text**: `text.run[]` sequence of `{gid, dx(Q16.16), dy(Q16.16)}`; path may be omitted when text present.
- **Optional mask**: `mask_id: u32` referencing another object’s fill coverage.

### 4.3 Resources

- `glyphs[]`: `{gid, outline:path}` pre‑extracted outlines (cubic). The file also permits embedding of the originating vector source for provenance.

### 4.4 Canvas and Tiling

- `w,h`: canvas in pixels; `tile_w,tile_h` define encoder/renderer tile size.
- `bitdepth`: 10–16; `primaries`/`matrix` strings for informational tagging.
- Dirty tiles map is width `ceil(w/tile_w) × ceil(h/tile_h)` bits, row‑major.

### 4.5 Messages

- **Init** `vsm_bundle_init`: `{scene_id, profile, canvas, keyframe{objects[], resources}, src_present?, src_meta?}`.
- **Delta** `vsm_delta`: `{scene_id, frame_idx, xforms[{id, affine, vis}], style[], dirty: bytes}`.

**Wire format:** CBOR (RFC 8949), canonical, definite length. Keys are lower‑case ASCII. Unknown keys are ignored.

---

## 5. Carriage in Containers (Normative)

### 5.1 AV1 Elementary Stream (Required)

- Encapsulate each VSM message in an **OBU\_METADATA**. The payload layout: `metadata_type: uint8` followed by the CBOR message. `metadata_type` value allocation uses AOM reserved metadata space.
- Size uses LEB128 per AV1 OBU rules. VSM messages are interleaved at access‑unit boundaries (before the coded frame they describe).

### 5.2 MP4 (Optional)

- Use `emsg` version 0. `scheme_id_uri = "urn:org.aomedia:vsm:1"`, `value` selects message type (e.g., `vsm_delta`). `message_data` is the raw CBOR payload.

### 5.3 Matroska/WebM (Optional)

- Carry raw CBOR in `BlockAdditional` with registered `BlockAddID` assigned to VSM.

Reference helper packers for each carriage exist in the accompanying code.

---

## 6. Decoder/Encoder Behavior (Normative)

### 6.1 Compatibility

- A standard AV1 decoder SHALL ignore all VSM data. No coded‑video semantics change.

### 6.2 VSM‑Aware Encoder

Given `vsm_bundle_init` and subsequent `vsm_delta`:

1. **ROI/QP Map from Dirty Tiles.** For every coded frame, mark dirty tiles (1 → ROI). Recommended QP deltas: ROI=−6, neighbors=−3, others=0. Encoders propagate deltas through their segmentation/ROI API.
2. **Motion‑Vector Seeds from Affine.** For each object intersecting a block, derive candidate MV `(vx,vy)` by sampling the affine transform on block centers over Δt and quantizing to MV units. Install seeds for ME and merge candidates.
3. **Skip/Golden Reuse.** Where an object’s coverage and style remain unchanged over N frames, prefer skip/no‑recode for those blocks.
4. **Transform/Prediction Hints.** Prefer global/warp models matching object affines.

### 6.3 VSM‑Aware Renderer/Player (Optional)

A player MAY ignore VSM or use it to re‑rasterize at the display resolution following Section 7, compositing over the decoded luma/chroma or replacing it entirely.

---

## 7. Normative Raster (Resolution‑Independent)

This section defines a deterministic CPU raster used by the reference and for conformance.

### 7.1 Geometry

- **Flattening:** Each cubic segment subdivides recursively until the control‑polygon flatness `max(||3c1−2p0−p3||, ||3c2−2p3−p0||) < τ`. Default τ=0.25 px.
- **Fill Rule:** Even‑odd.
- **Antialiasing:** Supersample on an `S×S` grid per pixel. `S=4` default. Conformance mode `S=8` when requested.
- **Stroke:** Width `w`. Joins/caps per object. **Miter join** length `L = w/(2·sin(θ/2))`. If `L > miter_limit·w/2`, bevel fallback. Otherwise, add a wedge triangle from the offset edges to the miter apex.

### 7.2 Paints

- **Solid:** Constant linear RGBA.
- **Linear Gradient:** Parameter `t` from projection of sample point onto the line segment; clamp to `[0,1]`; interpolate stop colors linearly in RGBA.
- **Two‑Circle Radial:** Centers `(C0,r0)`→`(C1,r1)`. Solve `||P−C(t)|| = r(t)` with bisection on `[0,1]` (24 iterations). Interpolate stops at that `t`.

### 7.3 Compositing

- Compute source color in linear light (pre‑multiplied by coverage and source alpha). Apply **blend mode** to the destination color component‑wise: `normal`, `multiply`, `screen`, `overlay`.
- Apply **Porter–Duff** composite: `src‑over` (default), `dst‑over`, `plus` (additive, alpha saturates), `xor`. Equations use pre‑multiplied RGBA.
- Optional `mask_id` multiplies source alpha by mask coverage.

### 7.4 Y′CbCr Write‑Out

- Convert linear RGB to Y′CbCr(709) with coefficients `(Kr,Kb)=(0.2126,0.0722)`. Full range writes `[0..2^n−1]` per component. Limited range maps to `[16..235]` luma and `[16..240]` chroma in 8‑bit space scaled to `n` bits.

---

## 8. Text and Glyphs

- **Shaping:** The ingest pipeline shapes UTF‑8 text into glyph IDs and advances using HarfBuzz with language/script/features as needed. The run encodes `{gid, dx, dy}` in canvas units Q16.16.
- **Outlines:** Glyph outlines derive from the font via FreeType; quadratics are converted to cubics for the path representation.
- The renderer composites glyph outlines like any other object. Hinting is out of scope.

---

## 9. Wire Format (CBOR)

**Canonical, definite‑length CBOR** per RFC 8949. Minimal schema:

```
init := {
  "type": "vsm_bundle_init",
  "scene_id": uint,
  "profile": text,                # e.g., "vsm-lite-1", "vsm-full-1"
  "canvas": {
    "w": uint, "h": uint,
    "tile_w": uint, "tile_h": uint,
    "color": {"bitdepth": uint, "primaries": text, "matrix": text, "limited_range": bool}
  },
  "keyframe": {
    "objects": [object, ...],
    "resources": {"glyphs": [{"gid": uint, "outline": path}, ...]}
  },
  "src_present": bool?,
  "src_meta": {"mime": text, "encoding": text, "uncompressed": uint, "sha256": text}?
}

delta := {
  "type": "vsm_delta",
  "scene_id": uint,
  "frame_idx": uint,
  "xforms": [{"id": uint, "affine": [int,...×6], "vis": bool}],
  "style": [...],
  "dirty": bstr                        # tile bitmap, row‑major
}

object := {
  "id": uint, "z": int,
  "path": path?,                       # omitted if text‑only
  "paint": paint,
  "stroke": {"w": int, "join": text, "cap": text, "miter": uint}?,
  "blend": text,                       # normal|multiply|screen|overlay
  "comp": text,                        # src-over|dst-over|plus|xor
  "affine": [int,...×6],
  "visible": bool,
  "mask_id": uint?,
  "text": {"run": [{"gid": uint, "dx": int, "dy": int}, ...]}?
}

path := {"kind":"cubic", "pts": [int, ...]}   # Q16.16 pairs making cubic segments
paint := solid | linear | radial
solid := {"type":"solid", "r":uint, "g":uint, "b":uint, "a":uint}
linear := {"type":"linear", "x0":int, "y0":int, "x1":int, "y1":int,
           "stops":[{"pos":uint, "r":uint, "g":uint, "b":uint, "a":uint}, ...]}
radial := {"type":"radial", "cx0":int, "cy0":int, "r0":int, "cx1":int, "cy1":int, "r1":int,
           "stops":[{"pos":uint, "r":uint, "g":uint, "b":uint, "a":uint}, ...]}
```

**Encoding rules**

- All integers big‑endian per CBOR. No indefinite‑length items. Keys sorted lexicographically for canonical form.
- Unknown keys ignored. Unknown `profile` accepted.

---

## 10. Reference Algorithms (Enabling Disclosure)

### 10.1 MV Seeding from Affine

Given object transform `A = [[a,b],[c,d]]` and translation `(e,f)` in Q16.16, and block center `(x,y)`:

```
(x',y') = (a·x + b·y + e,  c·x + d·y + f)
MV = round(((x'−x)/Δt, (y'−y)/Δt) / MV_unit)
```

Install MV as a candidate for the block and its neighbors.

### 10.2 ROI/QP from Dirty Map

Let `T` be the dirty bitmap. For each tile `t∈T`, assign QP delta −6. For 1‑tile neighborhood, assign −3. Others 0. Clip to encoder’s segmentation limits.

### 10.3 Miter Limit Test

Let vectors `u = normalize(B−A)` and `v = normalize(C−B)`. Interior angle `θ = acos(u·v)`. Miter length `L = w/(2·sin(θ/2))`. If `L > miter_limit·w/2`, bevel fallback. Else form wedge triangle from offset edges to apex at `B + L·normalize(n0+n1)` with `n0 = left(u)`, `n1 = left(v)`.

### 10.4 Porter–Duff (pre‑multiplied)

```
src-over:   C = Cs + Cd·(1−As),   A = As + Ad·(1−As)
dst-over:   C = Cd + Cs·(1−Ad),   A = Ad + As·(1−Ad)
plus:       C = min(1, Cs + Cd),   A = min(1, As + Ad)
xor:        C = Cs·(1−Ad) + Cd·(1−As),  A = As·(1−Ad) + Ad·(1−As)
```

### 10.5 Two‑Circle Radial Parameter

Solve for `t∈[0,1]` where `||P−(C0 + t(C1−C0))|| − (r0 + t(r1−r0)) = 0` by bisection (24 iterations). Interpolate stops at that `t`.

### 10.6 Supersampling

Default `S=4` sample grid per pixel; conformance `S=8`. Sample positions are center‑of‑cell on a uniform lattice. Average coverage and composite in linear light.

---

## 11. Interoperability and Backward Compatibility

- VSM never changes AV1 bitstream semantics. Streams remain compliant when stripping all VSM OBUs.
- Multiple VSM messages per access unit are ordered; the last for a frame wins.
- Profiles define floors:
  - **vsm‑lite‑1**: solids + linear gradients, fills only, text runs allowed, 4× SS, full‑range Y′CbCr.
  - **vsm‑full‑1**: adds radial gradients, stroke with miter‑limit, masks, blend modes, Porter–Duff, 8× SS, limited‑range signaling, glyph resources.

---

## 12. Security and Privacy

- Payloads contain graphics primitives and optional text glyph IDs. No executable code. Reference implementations perform strict bounds checking and reject indefinite‑length CBOR.
- Optional source embedding is treated as opaque data with MIME/length/hash metadata.

---

## 13. Reference Implementation (Availability)

A permissively licensed reference implementation (**libvsm**) accompanies this disclosure, including:

- CBOR loaders (`vsm_cbor_*`), deterministic CPU raster with fills, strokes, masks; blend modes and Porter–Duff; two‑circle radial; 4×/8× SS; linear‑light Y′CbCr with limited/full range; text runs and glyph outlines.
- Container packers for AV1 OBU, MP4 `emsg`, and Matroska `BlockAdditional`.
- Demo app that renders example payloads to YUV44416/PPM and illustrates tile iteration. All code is CC0. Any compatible implementation is free to diverge.

---

## 14. Test Material

### 14.1 Minimal Init (JSON)

```json
{
  "type":"vsm_bundle_init","scene_id":1,"profile":"vsm-full-1",
  "canvas":{"w":1920,"h":1080,"tile_w":240,"tile_h":135,
            "color":{"bitdepth":10,"primaries":"bt709","matrix":"bt709","limited_range":false}},
  "keyframe":{
    "objects":[{
      "id":10,"z":0,
      "path":{"kind":"cubic","pts":[0,0,41943040,0,41943040,0, 83886080,0,83886080,23592960,83886080,23592960, 83886080,47185920,41943040,47185920,41943040,47185920, 0,47185920,0,23592960,0,23592960]},
      "paint":{"type":"solid","r":64,"g":64,"b":64,"a":1023},
      "stroke":{"w":131072,"join":"miter","cap":"round","miter":1024},
      "blend":"normal","comp":"src-over",
      "affine":[65536,0,0,65536,20971520,11796480],
      "visible":true
    }],
    "resources":{"glyphs":[]}
  }
}
```

### 14.2 Delta (JSON)

```json
{
  "type":"vsm_delta","scene_id":1,"frame_idx":1,
  "xforms":[{"id":10,"affine":[65536,0,0,65536,22282240,11796480],"vis":true}],
  "dirty":"…bytes…"
}
```

---

## 15. Variations and Extensions (Placed in the Public Domain)

To broaden prior art coverage, the following variants are expressly included:

- Other vector segment bases (quadratic, lines, arcs) with equivalent cubic canonicalization.
- Additional blends and Porter–Duff operators.
- Additional gradient types (conical, diamond), mesh gradients.
- Alternative shaping engines or inline glyph path embedding for text.
- Alternative fixed‑point precisions and sampling lattices.
- Alternative ROI/QP mapping heuristics and object‑level saliency metrics.
- Alternative container bindings (e.g., CMAF events, timed ID3 in HLS/DASH) carrying the same CBOR.
- Hardware or GPU rasterizers that implement the same compositing semantics.

---

## 16. Claim‑Style Enumeration (For Searchability; Not Legal Claims)

1. In‑band carriage of a vector scene model synchronized to frames in an AV1 elementary stream such that unaware decoders ignore the data, while aware encoders/decoders use it to guide compression and/or rendering.
2. A message set including an initialization bundle and per‑frame deltas comprising affine transforms and dirty‑tile maps.
3. A per‑object description including a cubic path, a paint (solid/linear/radial), a stroke with miter‑limited joins, a blend mode, and a Porter–Duff composite operator.
4. Text runs expressed as glyph IDs and advances, with glyph outlines embedded as resources.
5. A deterministic raster using even‑odd fills, supersampling antialiasing, two‑circle radial interpolation, miter‑limit wedges, and linear‑light compositing.
6. Encoder guidance derived from object transforms and dirty tiles to generate ROI maps, motion‑vector seeds, skip decisions, and transform hints.
7. Container bindings for AV1 OBU metadata, MP4 `emsg`, and Matroska `BlockAdditional` carrying CBOR‑encoded messages.

---

## 17. IPR and Dedication

The author dedicates all subject matter herein—including variations, algorithms, data structures, formats, examples, and described implementations—to the public domain under CC0‑1.0. The intent is to constitute prior art and prevent the grant of exclusive rights on these techniques or obvious variants.

---

## 18. Change Log

- **Rev 2 (2025‑08‑12):** Added miter‑limit joins; Porter–Duff compositing; multiply/screen/overlay blends; glyph shaping and outline import; 8× SS conformance raster; limited‑range Y′CbCr; container mux helpers; expanded test material and normative sections.
- **Rev 1:** Initial disclosure with object model, deltas, dirty tiles, linear/radial gradients, encoder guidance, and AV1 OBU carriage.

