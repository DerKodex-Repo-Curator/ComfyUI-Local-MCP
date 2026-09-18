# ComfyUI graph cookbook (for authoring workflows)

Concrete recipes for building `comfy_local_mcp/workflows/<name>.json`. Worked examples in `workflows/`: `scene_image_flux.json` (base), `scene_image_flux_img2img.json`, `scene_logo_flux_controlnet.json`.

## File format

```jsonc
{
  "_meta": { "name", "description", "required_models": [...], "required_nodes": [...], "notes" },
  "_overrides": { "<logical_name>": "<node_id>.inputs.<field>", ... },
  "graph": { "<node_id>": { "class_type": "...", "inputs": { ... } }, ... }
}
```
- `graph` is ComfyUI **API format** (the "Save (API Format)" export, not the UI graph). Node inputs are literals or `["<src_node_id>", <output_index>]` links.
- `_overrides` lets callers pass logical names; `load_workflow(name, {logical: value})` resolves them to `node.inputs.field`. Model-role names that match the user's `config` are auto-filled. Expose every knob you'll iterate on.
- Validate: `python -c "from comfy_local_mcp.workflows import load_workflow; load_workflow('<name>', {})"`.

## Base Flux txt2img chain (node ids from scene_image_flux.json)

```
12 UnetLoaderGGUF(flux1-dev-Q4_K_S.gguf) ─┐
11 DualCLIPLoader(clip_l, t5xxl_fp8_e4m3fn, type=flux) ─┐
10 VAELoader(flux_vae.safetensors)
 6 CLIPTextEncode(text, clip=[11,0])
17 FluxGuidance(conditioning=[6,0], guidance=3.5)      # positive
16 ConditioningZeroOut(conditioning=[6,0])              # negative (Flux cfg=1 → no real negative)
 5 EmptySD3LatentImage(width,height,1)                  # Flux uses EmptySD3LatentImage, NOT EmptyLatentImage
13 KSampler(model=[12,0], positive=[17,0], negative=[16,0], latent_image=[5,0],
            cfg=1.0, sampler=euler, scheduler=simple, steps=20, denoise=1.0)
 8 VAEDecode(samples=[13,0], vae=[10,0])
 9 SaveImage(images=[8,0])
```
Keep `cfg=1.0` for Flux and zero the negative. `guidance` (FluxGuidance) is the prompt-adherence dial.

## Mutation: img2img (see scene_image_flux_img2img.json)

Replace the empty latent (node 5) with an encoded source image, and denoise < 1:
```
 4 LoadImage(image="<filename in ComfyUI/input>")
20 ImageScale(image=[4,0], width,height, lanczos, crop=disabled)   # keep source aspect, multiples of 16
 5 VAEEncode(pixels=[20,0], vae=[10,0])                            # now node 5 is the latent
13 KSampler(... latent_image=[5,0], denoise=0.45..0.7)
```
`denoise` = strength: ~0.45 faithful restyle, ~0.7 heavy. Copy the source into `ComfyUI/input/` first.

## Mutation: ControlNet (see scene_logo_flux_controlnet.json)

Lock structure from a source/sketch. Insert a control branch and rewire the KSampler's conditioning through `ControlNetApplyAdvanced`:
```
 4 LoadImage → 20 ImageScale → 30 Canny(low,high)                 # preprocessor → control image
31 ControlNetLoader(control_net_name="flux-union-pro-2.0.safetensors")
32 SetShakkerLabsUnionControlNetType(control_net=[31,0], type="canny")   # Union model needs the type set
33 ControlNetApplyAdvanced(positive=[17,0], negative=[16,0], control_net=[32,0],
        image=[30,0], strength, start_percent=0.0, end_percent, vae=[10,0])
13 KSampler(... positive=[33,0], negative=[33,1] ...)             # take BOTH outputs of node 33
```
- Union ControlNets require `Set(Union|ShakkerLabsUnion)ControlNetType`; types: auto/canny/tile/depth/blur/pose/gray/low quality.
- Use full-denoise txt2img (EmptySD3LatentImage) when you want reinterpretation; the control image carries the structure.
- Other preprocessors (depth/pose/lineart) usually need ComfyUI-controlnet-aux installed; built-in `Canny` is always available.

## Other mutations (discover exact fields via object_info)
- **Inpaint:** `LoadImage` + `LoadImageMask` → `VAEEncodeForInpaint(pixels, vae, mask)` → KSampler denoise<1.
- **Upscale (latent):** `LatentUpscaleBy` between two KSamplers, or **(model)** `UpscaleModelLoader → ImageUpscaleWithModel` on the decoded image.

## Tuning knobs — what actually moves the result
- **ControlNet `strength` holds structure; FluxGuidance does NOT.** Raising guidance pulls *away* from the control and jumbles/doubles the composition. To stay faithful: `strength` ~0.8–0.85, `end_percent` 1.0, guidance ~3.5. Dropping strength below ~0.75 → hallucinated background artifacts, not tasteful redesign.
- **`end_percent` (cn_end) is the gentle "step down from faithful" lever.** Hold strength ~0.8 and lower end_percent (~0.55–0.7): ControlNet sets structure early then releases so the model refines linework — without inventing junk.
- **Prompt biases the tone.** "pure white background / minimalist / flat vector" washes ink out to near-white (darkest pixel ~219/255). For solid output prompt "bold solid jet-black ink / high contrast monochrome" (→ ~43/255).
- **denoise** (img2img) = how far from the source. Fixed seed while tuning structure.

## VRAM (16 GB card, RTX 4080)
| Component | ~GB |
|---|---|
| Flux Q4 GGUF unet | 6.4 |
| t5xxl **fp16** | 9.2 |
| t5xxl **fp8_e4m3fn** | 4.9 |
| Union ControlNet | 4.3 |
| Flux VAE + clip_l | ~0.6 |

Flux + fp16 T5 alone ≈ 15.6 GB; adding a ControlNet OOMs → **use the fp8 T5**. `device_report` for live free VRAM. Over budget → fp8 encoder, lower resolution, or sequential offload (slower).

## Pitfalls
- **EmptySD3LatentImage** for Flux (not EmptyLatentImage). Keep `cfg=1.0` + ConditioningZeroOut negative.
- After downloading a model, ComfyUI rescans on an `object_info` GET; **new custom nodes need a restart**. Confirm the file appears in the loader's combo before submitting.
- HF downloads default to the C: cache — redirect to a roomy drive (`HF_HUB_CACHE`); a full system drive can crash ComfyUI mid-render.
- `ComfyClient.fetch_result` now re-polls `/history` briefly (fast/cached graphs used to race and falsely report "no outputs").
- One heavy job per GPU; sweeping two back-to-back under VRAM pressure can thrash/crash.
