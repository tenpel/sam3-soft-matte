# SAM3 Soft Matte Mask — ComfyUI workflow

A small ComfyUI workflow for pulling a **soft, compositable mask** of a detail
(an eye, a highlight, a logo, a reflection) out of video footage, built on
text-prompted SAM3 segmentation with three optional ways of softening the
result: alpha matting, a luma-derived matte, and a second wider detection.

Long footage is processed **in chunks**, so memory usage stays flat regardless
of clip length. There are two outputs, both numbered PNG sequences:

- **the matte** — the mask on its own, to import as an image sequence into
  After Effects, Nuke, Resolve or TouchDesigner
- **the RGBA cut-out** — the source frames with that matte as their alpha
  channel, so this workflow can feed a compositing graph directly (see
  [As a prep step](#as-a-prep-step))

```
                    SAM3 detect (detail) ─┐          ┌ grow + blur ──┐
 A  chunked input ─┐                      ├[4c: off]─┤     [5c: off] ├─┐
                   ├─ C                   │          └ F VITMatte ───┘ ├┬ E matte
 B  model+prompts ─┤  SAM3 detect (subject)┘                           ││
                   ├──── same, as an extra shape ────────── [9b: off] ─┘│
                   └─ D  luma soft matte ─────────────────── [7b: off] ─┴ G cut-out
```

**The base mode is the detail prompt alone** — one SAM3 pass, feathered with
grow + blur. Everything else sits behind lazy switches and is switched on only
when you need it.

---

## Why

Raw SAM3 masks are binary and edge-jittery. Segmentation is a per-pixel
classification — there is no alpha in the output, only a decision — which is
fine for inpainting and bad for compositing. The base pass papers over the
worst of it with grow + blur; three optional branches are there for when that
isn't enough:

1. **VITMatte refine** (`5c`) replaces grow + blur with actual alpha matting:
   a trimap is built around the SAM3 edge and a continuous alpha is regressed
   inside it. This is the only branch that produces *real* transparency — hair,
   fur, glass, smoke — rather than a feathered hard edge. Costs an extra node
   pack, a model, and a lot of time per frame.
2. **The luma matte** (`7b`) is built from the source frames themselves and
   reintroduces a gradient into the mask. Model-free, but it only works if the
   subject separates from the background in luminance.
3. **The subject mask** (`9b`) is a second SAM3 pass on a wider prompt
   (`fish` rather than `fish eye`), curve-shaped before it is combined. Switch
   it on when you need the whole silhouette as well — typically as the second
   operand of a mask operation downstream.

All three are off by default, and because the switches are lazy, an off branch
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
- [ComfyUI_LayerStyle](https://github.com/chflame163/ComfyUI_LayerStyle) — `LayerMask: MaskEdgeUltraDetail V2`
  — **only needed for the `5c` VITMatte branch**

### The VITMatte model (only for switch `5c`)

The `5c` branch needs one model that does not ship with ComfyUI:

1. Download from
   **<https://huggingface.co/hustvl/vitmatte-small-composition-1k/tree/main>**
   — `config.json`, `model.safetensors`, `preprocessor_config.json`
   (≈100 MB; `pytorch_model.bin` is the same weights in the old format and can
   be skipped).
2. Copy them into **`ComfyUI/models/vitmatte/`**.

The node's `method` widget then decides where it loads from:

| `method` | needs the download above | notes |
|---|---|---|
| `VITMatte` | no | fetches the model from Hugging Face on first run |
| `VITMatte(local)` | **yes** | loads from `ComfyUI/models/vitmatte/`, works offline |
| `PyMatting` | no | classical matting, no model, slower on large frames |
| `GuidedFilter` | no | fast edge-aware refinement — the principled version of what group D does by hand |

The workflow ships with `VITMatte`, so the branch works without any manual
download; switch it to `VITMatte(local)` once you've done step 2.

---

## Install

Drop the `.json` files into `ComfyUI/user/default/workflows/` (or just drag one
onto the ComfyUI canvas).

Two workflows ship here:

| file | for |
|---|---|
| `sam3_soft_matte_mask.json` | the general case — segmentation-driven, described below |
| `sam3_soft_Garbage_mask.json` | a translucent subject on a black plate — luma-driven, see [the second workflow](#the-second-workflow--sam3_soft_garbage_maskjson) |

---

## Usage

1. **Load your footage** — node `3. Load video chunk`. Put the file in
   `ComfyUI/input/` and pick it in the `video` widget.
2. **Write the prompts** — `2a` is the detail, `2b` the whole subject. Switch
   `4c` picks which one you're masking; short, concrete noun phrases work best.
3. **Pick the mode** — all three switches off is the base mode; see below.
4. **Set the chunk size** — see below. Test first with a single chunk.
5. **Run once** and check the mask preview / the first PNGs.
6. **Enable Auto Queue** and let it walk through the whole clip.
7. **Reset** the chunk index to `0` and turn Auto Queue off when it's done.

Output lands in `ComfyUI/output/` as two sequences:

- `mask_out/mask_soft__<batch>.<frame>.png` — the matte
- `cutout_rgba/cutout__<batch>.<frame>.png` — the RGBA cut-out

Don't need one of them? `Ctrl+M` mutes that save node and it stops being
written.

### As a prep step

The `G` group exists so this workflow can hand off directly to a compositing
graph — dropping a cut-out object into a plate and relighting it, for example.
`13. Source + alpha → RGBA` puts the final matte into the alpha channel of the
source frames, and `14` writes transparent PNGs.

Three things matter when you pick those up downstream:

1. **The alpha is straight, not premultiplied.** ComfyUI does not multiply RGB
   by alpha. Import as *straight / unmatted*; interpreting it as premultiplied
   is exactly what puts dark fringes on soft edges — and the softer the matte
   (`5c`, `7b`), the more visible that mistake gets.
2. **`InvertMask` in front of `JoinImageWithAlpha` is not optional.** That core
   node does `alpha = 1.0 - mask` internally, because a ComfyUI MASK marks a
   *region* and is not an alpha channel. Without node `12` you get a
   transparent object on an opaque background.
3. **8-bit will band** if the matte gets graded hard later. Switch node `14` to
   `video/16bit-png` (rgba64) for that, or to `ProRes` with `profile: 4444` if
   you'd rather have one `.mov` with alpha than a sequence — all three formats
   carry the alpha through.

This is also where the `5c` switch earns its keep: a grow + blur edge is
obvious the moment the cut-out is composited over a different plate, while a
VITMatte alpha holds up.

**The hard case.** A subject that is translucent across its whole body, or has
thin fins, fur or hair, is not a soft-*edge* problem at all — it is a large
area of partial transparency, and a binary mask has no way to represent it. No
amount of grow + blur helps there; rotoscoping it by hand is worse. That case
is what `4c` on + `5c` on is for, with the cut-out from group G as the thing
you actually judge. Worth trying `7b` on top too: for a translucent body the
luminance roughly *is* the opacity, which is the one situation where that
branch stops being a hack.

### Modes

Four `LazySwitchKJ` nodes, all **off by default**:

| switch | off (default) | on |
|---|---|---|
| `4c` target | the **detail** — prompt `2a` | the **whole subject** — prompt `2b` |
| `5c` VITMatte refine | `5a` grow + blur — feathered hard edge, fast | `5v` VITMatte — real alpha, slow, needs the model above |
| `7b` luma soft matte | — | adds the group D matte for a graded edge |
| `9b` subject / full shape | — | adds the second SAM3 pass (prompt B, curve-shaped) |

With all four off you get the **base mode**: one SAM3 detection per frame on
the detail prompt, feathered with grow + blur.

`4c` decides *what* everything downstream operates on — the refine, the luma
matte and the cut-out all follow it. Two presets worth starting from:

| | `4c` | `5c` | `7b` | `9b` |
|---|:---:|:---:|:---:|:---:|
| a small, hard-edged detail | off | off | off | off |
| a translucent subject (fins, fur, glass) | **on** | **on** | try it | off |

With `4c` on, `9b` is pointless — it would add the subject to itself.

The switch inputs are *lazy*, so whatever is off is **not executed** — not the
VITMatte node, not group D and `7a`, not the second SAM3 pass and the curves
subgraph. The base mode really is a single detection per frame, not a full
graph with branches thrown away at the end.

`5c` and `7b` are alternative answers to the same problem — a binary mask edge
— and they are not exclusive, but turning both on is usually redundant: if
VITMatte solved the edge properly, adding a luma matte on top mostly widens
it again.

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
Two `SAM3_Detect` passes against the same frames. Switch `4c` picks which one
drives the rest of the graph; the other only runs if `9b` needs it.

| | detail branch | subject branch |
|---|---|---|
| `expand` | 10 px | 0 px |
| `blur_radius` | 10 px | 1 px |

`threshold` (default `0.5`) is the knob to reach for first: lower it if the
detection drops out on some frames, raise it if it starts grabbing background.
`individual_masks = false` merges everything found into one mask per frame.

### F — VITMatte refine *(switchable, extra install)*
`LayerMask: MaskEdgeUltraDetail V2` takes the **raw** SAM3 mask (not the
grown/blurred one — it needs the un-feathered edge) plus the source frames,
builds a trimap band around the edge and regresses a continuous alpha inside
it.

| parameter | what it does |
|---|---|
| `edge_erode` / `edte_dilate` | width of the trimap band in px. Wider = more room to solve, slower, more bleeding risk. `6 / 6` is a sane start. |
| `black_point` / `white_point` | alpha clipping. `0.01 / 0.99` keeps it soft; pull them in to harden the matte. |
| `max_megapixels` | VITMatte downscales above this. The node's own default of `2.0` is *below* HD — the workflow ships at `4.0` for 1920×1080; raise it for 4K at the cost of VRAM. |
| `device` | `cpu` works, just slowly. |

This is by far the slowest node in the graph — it runs per frame. Test on one
short chunk before queueing a sequence.

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

### G — RGBA cut-out
`InvertMask` → `JoinImageWithAlpha` → `VHS_VideoCombine`. Takes the same final
mask group E saves and the source frames, and writes transparent PNGs. See
[As a prep step](#as-a-prep-step) for the caveats.

---

## Tuning cheat sheet

| Symptom | Try |
|---|---|
| Mask flickers between frames | raise `refine_iterations`, increase `expand` + `blur_radius` on the detail branch |
| Detection misses frames | lower `threshold` on `SAM3_Detect` |
| Background gets picked up | raise `threshold`, make the prompt more specific |
| Edge too hard | raise `blur_radius`, or open up the Levels mid value in group D |
| Subject is translucent / has hair, fur, fine strands | `4c` on + `5c` on — grow + blur cannot invent alpha that isn't there |
| Fins / strands disappear entirely | widen `edge_erode` / `edte_dilate` so the trimap actually covers them, and lower `black_point` |
| `5c` bleeds into the background | narrow `edge_erode` / `edte_dilate`, or raise `black_point` |
| Too much of the body in the matte | flatten the RGB master curve in `8b`, switch `9a` to `multiply`, or turn `9b` off |
| Luma matte fights the footage (noise, low contrast, busy background) | turn `7b` off — pure SAM3 mask |
| Slow, and you only need the detail | both switches off; the other branches then never run |
| Out of memory | lower `frame_load_cap` **and** the multiplier in the expression |
| Cut-out has a dark halo when composited | your compositor is treating straight alpha as premultiplied |
| Cut-out is inverted (background opaque) | node `12. InvertMask` got removed or bypassed |

---

## Notes

- The workflow saves a PNG *sequence*, not a video — `VHS_VideoCombine` with
  `format: video/8bit-png`. Each chunk writes its own numbered batch, so the
  frames stay in order across chunks.
- The prompts shipped in the file (`translucent fish` / `translucent fish's eye`)
  are just an example; replace them with your own.
- Everything except the `5c` branch runs on the four core custom node packs. If
  you don't want the LayerStyle dependency, delete the `5v` node and the `5c`
  switch — the rest of the graph is unaffected.

## The second workflow — `sam3_soft_Garbage_mask.json`

A different answer to the same problem, for one specific but very common case:
**a translucent subject on a pure black plate** (glass, smoke, sparks, a
rendered element, an x-ray-looking fish).

Here the segmentation model is *not* the right tool for the matte, and the
main workflow's `5c` VITMatte branch will not save you either. The reason is
simple: an element on black is **already premultiplied** — `RGB = alpha ×
colour` — so the luminance *is* the alpha, to a good approximation. That is
how smoke and fire have always been keyed. A model guessing at transparency
cannot beat a plate that states it directly.

So the roles are inverted:

```
luma  ──────────────── the alpha              (group D)
  × dilated SAM3 body   garbage matte         (group F1)
  + eroded body core    body mass, ~15%       (group F2)
  + SAM3 eye            semantic hold-out     (group F3)
```

**What SAM3 is for here** — two narrow jobs, neither of them the matte:

- **garbage matte** (the body prompt) — *where is there any subject at all*,
  multiplied in to kill reflections and noise. Dilated hard on purpose: a
  garbage matte that clips the fins is worse than no garbage matte.
- **semantic hold-out** (the eye prompt) — *what must stay opaque even though
  it is dark*. The pupil is black, but it is part of the fish. Luminance
  cannot know that; a prompt can. This is the part a pure key can never do.

**Body mass** is the third piece, and the one that makes the result read as an
object. A fully translucent body luma-keys down to its own internal structure —
you get the skeleton and the organs and lose the animal. Two `erode + blur`
passes approximate a distance transform (value by distance from the edge),
remapped to `0 … 0.15` by `RemapMaskRange`. That is a 15% soft brush that
follows the subject on every frame. A true medial-axis skeleton would give a
line drawing instead — the centreline is not what you want, "more opaque
towards the middle" is.

Read as a Photoshop recipe: luma-key the layer, mask off the junk, paint mass
back with a 15% soft brush, paint the pupil back to 100%.

### Order matters

`multiply` the garbage matte **first**, on the raw luma — anything outside the
body dies there and cannot come back. Then the additions. `MaskComposite`
clamps to `0…1` after each step, so `add` behaves as a soft union.

> Only `multiply` / `add` / `subtract` are safe on a soft matte.
> `and` / `or` / `xor` **binarise** their inputs and will destroy it.

### Output

16-bit RGBA PNG, and **interpret it as premultiplied (black)** in After
Effects. The plate is black, so the RGB is already multiplied by its alpha;
importing as straight darkens every semi-transparent pixel a second time. On a
subject that is transparent all over, that is not a fringe — it is the whole
subject. The mass added in group E raises alpha above the luminance, so the
body lifts slightly under premultiplied interpretation; that is intentional.

8-bit was not enough here: this matte bands after the Levels curve.

### Switch

One, deliberately: `F2b` turns the body mass on and off, so you can A/B it
against the plain luma key.

---

## License

MIT — see [LICENSE](LICENSE).
