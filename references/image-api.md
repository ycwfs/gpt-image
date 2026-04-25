# Image API quick reference

This file documents the default CLI mode. Use it when the skill is running `scripts/image_gen.py`.

These parameters describe the Image API and bundled CLI surface. Do not assume they are normal arguments on the Codex/native path.

## Scope
- This CLI is intended for GPT Image models (`gpt-image-2`, `gpt-image-1.5`, and `gpt-image-1`, `gpt-image-1-mini`).
- The Codex/native path and the CLI path do not expose the same controls.

## Endpoints
- Generate: `POST /v1/images/generations` (`client.images.generate(...)`)
- Edit: `POST /v1/images/edits` (`client.images.edit(...)`)

## Core parameters for GPT Image models
- `prompt`: text prompt
- `model`: image model
- `n`: number of images (1-10)
- `size`: `1024x1024`, `1536x1024`, `1024x1536`, `2048x2048`, `2048x1152`, `3840x2160`, `2160x3840` or `auto`
- `quality`: `low`, `medium`, `high`, or `auto`
- `background`: output transparency behavior (`transparent`, `opaque`, or `auto`) for generated output; this is not the same thing as the prompt's visual scene/backdrop
- `output_format`: `png` (default), `jpeg`, `webp`
- `output_compression`: 0-100 (jpeg/webp only)
- `moderation`: `auto` (default) or `low`

## Size Constrain	
- Maximum edge length must be less than or equal to 3840px
- Both edges must be multiples of 16px
- Long edge to short edge ratio must not exceed 3:1
- Total pixels must be at least 655,360 and no more than 8,294,400

## Edit-specific parameters
- `image`: one or more input images. For GPT Image models, you can provide up to 16 images.
- `mask`: optional mask image
- `input_fidelity`: `low` (default) or `high`

Model-specific note for `input_fidelity`:
- `gpt-image-1` and `gpt-image-1-mini` preserve all input images, but the first image gets richer textures and finer details.
- `gpt-image-1.5` preserves the first 5 input images with higher fidelity.

## Output
- `data[]` list with `b64_json` per image
- The bundled `scripts/image_gen.py` CLI decodes `b64_json` and writes output files for you.

## Limits and notes
- Input images and masks must be under 50MB.
- Use the edits endpoint when the user requests changes to an existing image.
- Masking is prompt-guided; exact shapes are not guaranteed.
- Large sizes and high quality increase latency and cost.
- High `input_fidelity` can materially increase input token usage.
- If a request fails because a specific option is unsupported by the selected GPT Image model, retry manually without that option.

## Important boundary
- `quality`, `input_fidelity`, explicit masks, `background`, `output_format`, and related parameters are CLI-only execution controls.
- Do not assume they are arguments on the Codex/native path.
