---
name: comfy-local-authoring
description: "Author a NEW ComfyUI workflow graph when no shipped workflow fits the user's intent — img2img, ControlNet, inpaint, upscale, or any custom node graph. Identifies the required nodes/models (e.g. a ControlNet model, a preprocessor node), installs what's missing, writes an API-format graph JSON with override points into comfy_local_mcp/workflows/, runs it via the client, and iterates on the exposed knobs. Use when: 'turn this image into…', 'use my drawing as a control/reference', 'build a workflow that…', 'img2img / controlnet / inpaint with ComfyUI', or any local generation the built-in workflows (list_workflows) don't cover. Distinct from the comfy-local agent, which only PICKS among existing workflows."
version: 0.1.0
---

# Authoring ComfyUI workflows

When the user's intent isn't covered by an existing workflow (`list_workflows`), you build a new one. A workflow is an **API-format graph JSON** in `comfy_local_mcp/workflows/<name>.json` that the client loads, parameterizes, and submits. Your job: interpret intent → design the graph → make sure its nodes/models exist → author the JSON → run it → expose the right knobs and iterate.

The node-recipe cookbook (base Flux chain, img2img/ControlNet/inpaint inserts, the `_overrides` format, VRAM math) lives in `references/graph-cookbook.md` — **read it before authoring**. Two worked examples already ship in `workflows/`: `scene_image_flux_img2img.json` and `scene_logo_flux_controlnet.json` — copy their structure.

## Protocol

1. **Clarify intent + inputs.** What's the *operation* (restyle an image? lock structure from a sketch? inpaint? upscale?) and what are the inputs (a source image? a control type — canny/depth/pose? faithful vs. creative)? These decide the topology. Ask only what you can't infer.

2. **Pick a base chain, then mutate it.** Don't author from scratch — start from a known-good workflow for the model family (for Flux: `scene_image_flux.json`) and splice in the operation:
   - **img2img** = replace `EmptySD3LatentImage` with `LoadImage → ImageScale → VAEEncode`, and set KSampler `denoise` < 1.
   - **ControlNet** = `LoadImage → (preprocessor, e.g. Canny) → ControlNetLoader → Set*ControlNetType → ControlNetApplyAdvanced` rewiring the KSampler's positive/negative.
   See the cookbook for exact node wiring.

3. **Discover nodes; never guess class names or fields.** Query ComfyUI's object_info to confirm a node exists and to read its exact inputs/outputs:
   `GET http://127.0.0.1:8188/object_info/<NodeClass>` (or `list_models(node_class)` for a loader's installed files). An **empty combo list = the model/node is NOT installed.**

4. **Identify and install what's missing.** If a required model isn't in the loader's list (ControlNet model, VAE, encoder) or a node class is absent:
   - Find it (Hugging Face MCP if registered) and download to the right ComfyUI dir: ControlNets → `models/controlnet/`, encoders → `models/clip/`, etc.
   - **Send downloads to a roomy drive, not the default C: cache** — set `HF_HUB_CACHE` / `cache_dir` to a path with space, then copy into ComfyUI. (A full system drive crashes ComfyUI.)
   - Custom nodes → ComfyUI-Manager or `git clone` into `custom_nodes/`. Then re-query object_info (ComfyUI rescans models on an object_info GET; new nodes need a restart).

5. **Budget VRAM before running.** `device_report` for free VRAM; sum the model footprints. On a 16 GB card the Flux unet (Q4 ~6.4) + **fp8** T5 (~4.9) + a ControlNet (~4.3) ≈ 15.6 GB fits, but fp16 T5 (9.2) pushes it OOM. Levers when over budget: swap the fp8 text encoder, lower working resolution, or accept sequential offload (slower). Don't launch a job that will thrash or OOM-crash.

6. **Author the JSON.** Write `comfy_local_mcp/workflows/<name>.json` with `_meta` (name, description, required_models, required_nodes, notes), an `_overrides` map (logical name → `"<node_id>.inputs.<field>"`), and the `graph`. Expose as overrides the knobs you'll want to iterate (prompt, seed, denoise/strength, the model-role names). Validate it loads: `python -c "from comfy_local_mcp.workflows import load_workflow; load_workflow('<name>', {})"`.

7. **Run + iterate.** Submit through the client (`ComfyClient().submit(name, overrides)` → `wait_for_result` → `download`) or the MCP (`submit_workflow` + `get_result`). Then sweep the knobs **one generation at a time** (single GPU). Change one parameter per run, keep the seed fixed while tuning structure, and log what each does. Vectorize/post-process rasters locally rather than re-generating when the fix is geometric.

## Principles
- **Mutate a proven graph; don't hand-write loaders.** The text-encoder/VAE/guidance wiring is fiddly and model-specific — reuse what works.
- **object_info is ground truth.** Confirm every node class and model file before submitting; an empty combo list means "not installed."
- **Honest about hardware.** Check VRAM and disk first; a clear "needs fp8 / won't fit" beats a 10-minute thrash or a crash.
- **One heavy job at a time.** Iterate sequentially; expose knobs so iteration is a parameter change, not a rebuild.
