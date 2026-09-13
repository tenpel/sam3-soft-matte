# SAM3 Soft Matte Mask — ComfyUI workflow

A small ComfyUI workflow for pulling a **soft, compositable mask** of a detail
(an eye, a highlight, a logo, a reflection) out of video footage, using
text-prompted SAM3 segmentation blended with a luma-derived matte.

Long footage is processed **in chunks**, so memory usage stays flat regardless
of clip length. The result is written out as a numbered 8-bit PNG sequence,
ready to be imported as an image sequence into After Effects, Nuke, Resolve or
TouchDesigner.

```
 A  chunked input ─┐
                   ├─ C  SAM3 detect (detail) ─ grow+blur ──────────┐
 B  model+prompts ─┤                                                ├─ save PNG seq
                   ├─ C  SAM3 detect (subject) ─ curves ─[9b: off]──┤
                   └─ D  luma soft matte ───────────────[7b: off]───┘
```

**The base mode is the detail prompt alone** — one SAM3 pass, nothing else.
The other two branches sit behind lazy switches and are switched on only when
you need them.

---

## Why

Raw SAM3 masks are binary and edge-jittery — fine for inpainting, bad for
compositing. The base pass already fixes the worst of it with grow + blur, and
two optional branches are there for when that isn't enough:

1. **The luma matte** (`7b`) is built from the source frames themselves and
   reintroduces a real gradient into the mask instead of a hard cutout.
2. **The subject mask** (`9b`) is a second SAM3 pass on a wider prompt
   (`fish` rather than `fish eye`), curve-shaped before it is combined. Switch
   it on when you need the whole silhouette as well — typically as the second
   operand of a mask operation downstream.

Both are off by default, and because the switches are lazy, an off branch
costs nothing at all.

---

## Requirements

| | |
|---|---|
| ComfyUI | recent build with SAM3 core nodes (`SAM3_Detect`, `CurveEditor`, `GLSLShader`, subgraph support) |
| Model | a SAM3 checkpoint in `ComfyUI/models/checkpoints/` — the workflow ships pointing at `sam3.1_multiplex_fp16.safetensors` |

Custom nodes:

- [ComfyUI-VideoHelperSuite](https://github.com/Kosinkadink/ComfyUI-VideoHelperSuite) — `VHS_LoadVideo`, `VHS_VideoCombine`
- [ComfyUI-KJNodes](https://github.com/kijai/ComfyUI-KJNodes) — `GrowMaskWithBlur`, `LazySwitchKJ`
- [WAS Node Suite](https://github.com/WASasquatch/was-node-suite-comfyui) — `Image Levels Adjustment`
- [pythongosssss / ComfyUI-Custom-Scripts](https://github.com/pythongosssss/ComfyUI-Custom-Scripts) — `MathExpression`

---

## Install

Drop `sam3_soft_matte_mask.json` into
`ComfyUI/user/default/workflows/` (or just drag it onto the ComfyUI canvas).

---

## Usage

1. **Load your footage** — node `3. Load video chunk`. Put the file in
   `ComfyUI/input/` and pick it in the `video` widget.
2. **Write the detail prompt** — node `2a`. Short, concrete noun phrases work
   best. (`2b`, the subject prompt, only matters if you switch `9b` on.)
3. **Pick the mode** — both switches off is the base mode; see below.
4. **Set the chunk size** — see below. Test first with a single chunk.
5. **Run once** and check the mask preview / the first PNGs.
6. **Enable Auto Queue** and let it walk through the whole clip.
7. **Reset** the chunk index to `0` and turn Auto Queue off when it's done.

Output lands in `ComfyUI/output/mask_out/` as `mask_soft__<batch>.<frame>.png`.

### Modes

Two `LazySwitchKJ` nodes, both **off by default**:

| `7b` luma | `9b` subject | result |
|:---:|:---:|---|
| off | off | **base** — pure SAM3 mask of the detail prompt. One SAM3 pass, clean hard edge. |
| **on** | off | detail mask + luma soft matte — soft, graded edge |
| off | **on** | detail mask + full subject silhouette |
| **on** | **on** | everything (the original behaviour of this graph) |

The switch inputs are *lazy*: whatever is switched off is **not executed** —
group D and node `7a` for `7b`, the second SAM3 pass and the curves subgraph
for `9b`. So the base mode really is a single detection per frame, not a full
graph with branches thrown away at the end.

Turn `7b` on when the edge needs to be graded rather than binary — it only
helps if the subject actually separates from the background in luminance.
Turn `9b` on when you need the whole shape, usually because you are going to
add / subtract / intersect it with something downstream.

### Chunked processing

Three nodes drive it:

```
Chunk index  (PrimitiveInt, mode: increment)   0, 1, 2, 3 ...
      |
Chunk offset (MathExpression)  a * 48   ->  skip_first_frames
      |
Load video   frame_load_cap = 48
```

> **The number in the expression must match `frame_load_cap`.**
> Change both if you want longer or shorter chunks. 48 frames is a safe
> starting point on 24 GB VRAM at HD; drop to 16–24 for 4K or a smaller card.

Because `PrimitiveInt` is set to `increment`, every queued run advances one
chunk automatically — that is the whole trick. Auto Queue then processes the
clip start to finish unattended.

**Caveat:** the chunk index keeps incrementing. Always reset it to `0` before
the next job, otherwise the next run starts somewhere in the middle of the
footage.

---

## The graph, group by group

### A — Chunked input
`Chunk index` → `Chunk offset` → `Load video`. Nothing else lives here.

### B — Model + prompts
SAM3 checkpoint loader, plus two `CLIPTextEncode` nodes. Prompt A is the
detail — this is the one that matters in the base mode. Prompt B is the
subject that contains it, and is only used when `9b` is on.

### C — SAM3 segmentation
Two `SAM3_Detect` passes against the same frames, each followed by
`GrowMaskWithBlur`. The subject pass only runs when `9b` is on.

| | detail branch | subject branch |
|---|---|---|
| `expand` | 10 px | 0 px |
| `blur_radius` | 10 px | 1 px |

`threshold` (default `0.5`) is the knob to reach for first: lower it if the
detection drops out on some frames, raise it if it starts grabbing background.
`individual_masks = false` merges everything found into one mask per frame.

### D — Luma soft matte *(switchable)*
`source → R channel as mask → back to image → Levels → mask again`.

The `Image Levels Adjustment` values (`6 / 96 / 255`) decide how much of the
bright interior survives. This is the part that gives the final mask a
gradient. `6e. Grow 1px` is **bypassed** (mode 4) by default — enable it if the
edge comes out too thin.

Off by default — switch `7b` turns it on. See [Modes](#modes).

### E — Combine + output

The tail of the graph is a chain of two optional additions, each gated by a
`LazySwitchKJ`:

```
detail mask ──┬─ 7a add luma matte ──┐
              └──────────────────────┴─[7b]─┬─ 9a add subject ──┐
                                            └───────────────────┴─[9b]─ save
```

The subject mask goes through a **Color Curves** subgraph (a GLSL curve editor
with GIMP-compatible interpolation) before being added in — flatten the RGB
master curve to reduce its influence, push it up to let more of the subject
bleed into the matte.

Both `MaskComposite` nodes use `add` (union). Switch `9a` to `multiply` if you
want the intersection — i.e. "the detail, but only where it overlaps the
subject" — which is often what you actually want for tight isolation.

---

## Tuning cheat sheet

| Symptom | Try |
|---|---|
| Mask flickers between frames | raise `refine_iterations`, increase `expand` + `blur_radius` on the detail branch |
| Detection misses frames | lower `threshold` on `SAM3_Detect` |
| Background gets picked up | raise `threshold`, make the prompt more specific |
| Edge too hard | raise `blur_radius`, or open up the Levels mid value in group D |
| Too much of the body in the matte | flatten the RGB master curve in `8b`, switch `9a` to `multiply`, or turn `9b` off |
| Luma matte fights the footage (noise, low contrast, busy background) | turn `7b` off — pure SAM3 mask |
| Slow, and you only need the detail | both switches off; the other branches then never run |
| Out of memory | lower `frame_load_cap` **and** the multiplier in the expression |

---

## Notes

- The workflow saves a PNG *sequence*, not a video — `VHS_VideoCombine` with
  `format: video/8bit-png`. Each chunk writes its own numbered batch, so the
  frames stay in order across chunks.
- The prompts shipped in the file (`translucent fish` / `translucent fish's eye`)
  are just an example; replace them with your own.

## License

MIT — see [LICENSE](LICENSE).
