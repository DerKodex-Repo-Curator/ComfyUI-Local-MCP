# /comfy-build - Author a new ComfyUI workflow for a custom operation

Build (not just pick) a ComfyUI workflow when no shipped one fits — img2img, ControlNet, inpaint, upscale, or any custom node graph — then run and iterate on it.

## Process
Follow the **comfy-local-authoring** skill:
1. Clarify the operation + inputs (source image? control type? faithful vs. creative).
2. Start from a known-good base graph (e.g. `scene_image_flux.json`) and splice in the operation (see `skills/authoring/references/graph-cookbook.md`).
3. Confirm node classes/fields and installed models via `/object_info` & `list_models`; install anything missing (download to the right ComfyUI dir, set the HF cache to a roomy drive).
4. Budget VRAM (`device_report`; prefer the fp8 T5 on a 16 GB card).
5. Write `comfy_local_mcp/workflows/<name>.json` (`_meta` + `_overrides` + `graph`); validate it loads.
6. Submit (`submit_workflow`/`get_result` or `ComfyClient`), then sweep the exposed knobs one generation at a time.

Usage: `/comfy-build use my sketch as a canny ControlNet and render a clean logo`
