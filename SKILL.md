---
name: gpt-image
description: This skill should be used when the user asks to "generate an image", "edit this image", "make a hero banner", "create a product mockup", "remove the background", "make bitmap variations", or needs help creating raster visuals with a CLI-first GPT Image workflow plus an explicit Codex path when requested.
---

# GPT Image Skill

Generate or edit raster images from Claude Code, GitHub Copilot CLI, or Codex-style environments while preserving a consistent two-mode workflow.

## Goal

- Make one image skill work in Claude and Copilot without assuming those hosts can call an image tool directly.
- Default to the bundled GPT Image CLI workflow for ordinary generation and editing.
- Preserve an explicit Codex-backed path for users who specifically want Codex/native generation.
- Keep raster image work separate from vector or code-native asset work.

## Two top-level modes

This skill has exactly two top-level modes:

- **Explicit CLI mode (default):** use `scripts/image_gen.py` for normal generation and editing requests unless the user explicitly asks for Codex/native generation. This path requires network access and `OPENAI_API_KEY`.
- **Codex mode (explicit opt-in):** in Claude Code and Copilot-style runtimes, invoke the local Codex CLI (`codex exec`) only when the user explicitly asks for Codex, native generation, or a Codex-built-in image workflow. This path does not require `OPENAI_API_KEY`, but it does require a working `codex` installation and auth.

If this skill is used from Codex itself, Codex mode maps to Codex's built-in image workflow directly. If this skill is used from Claude or Copilot, Codex mode means "delegate to Codex CLI" rather than "call a host-native image tool directly."

Within the default CLI mode, the bundled CLI exposes three subcommands:

- `generate`
- `edit`
- `generate-batch`

Rules:

- In Claude Code and Copilot-style runtimes, do **not** claim that a direct host-native image tool exists unless the runtime actually exposes one.
- Default to the bundled CLI mode for normal image generation and editing.
- Use Codex mode only when the user explicitly asks for Codex/native generation.
- Before running either mode, ask for any missing image parameters that materially affect the result or export.
- If the user explicitly says to use defaults, you may proceed with the documented CLI defaults instead of asking one-by-one.
- If CLI mode is unavailable because `OPENAI_API_KEY` is missing, tell the user how to set it locally and mention that Codex mode exists only as an explicit opt-in path.
- Never modify `scripts/image_gen.py` unless the user explicitly asks to change the bundled CLI.

## Codex mode in Claude/Copilot

When this skill runs in Claude Code or GitHub Copilot CLI, Codex mode means shelling out to Codex CLI only after explicit user opt-in.

Use these rules:

1. Use `codex exec` for non-interactive runs.
2. Pass `-C <workspace>` so Codex works in the current project.
3. If the current directory is not a Git repository, add `--skip-git-repo-check`.
4. For local input images, attach them with repeated `-i` flags and label each image's role in the prompt.
5. Ask Codex to save project-bound finals into the workspace before finishing, usually under `output/gpt-image/`.
6. Do not use CLI-only controls such as `quality`, `input_fidelity`, masks, `background`, output format, or direct API parameters in Codex mode.

Recommended Codex command shapes:

Generate:

```bash
codex exec -C "$PWD" --skip-git-repo-check \
  "Generate a blog header image for a post about AI agents. Save the selected final to output/gpt-image/blog-header.png and report the final path. Use Codex's built-in image workflow, not scripts/image_gen.py."
```

Edit:

```bash
codex exec -C "$PWD" --skip-git-repo-check -i input.png \
  "Image 1 is the edit target. Replace only the background with a soft blue gradient. Keep the subject unchanged. Save the final to output/gpt-image/input-edited.png and report the final path. Use Codex's built-in image workflow, not scripts/image_gen.py."
```

Codex output policy:

- Codex may initially place generated images under `$CODEX_HOME/generated_images/...`.
- If the image is meant for the current project, move or copy the selected final into the workspace before finishing.
- Never leave a project-referenced asset only under `$CODEX_HOME/generated_images/...`.
- Do not overwrite an existing asset unless the user explicitly asked for replacement; otherwise create a versioned sibling such as `hero-v2.png`.

## When to use

- Generate new raster images such as hero art, blog headers, product shots, game assets, textures, photoreal scenes, or bitmap infographics.
- Edit existing raster images such as background extraction, object replacement, text replacement, lighting changes, compositing, or style transfer.
- Produce multiple image variants when the end result should still be a bitmap asset.

## When not to use

- Extending or matching an existing repo-native SVG, icon, logo, or illustration system.
- Simple diagrams, wireframes, icons, or production vector/logo assets that are better produced directly in SVG, HTML/CSS, canvas, or draw.io.
- Small edits to assets that already exist in an editable native format and should stay there.
- Any task where the user clearly wants deterministic code-native output instead of generated bitmap output.

## Decision points

Think about three separate questions:

1. **Mode:** default CLI mode or explicit Codex mode?
2. **Intent:** generate a new image or edit an existing one?
3. **Execution strategy:** one asset, a few targeted variants, or many jobs (`generate-batch` only in explicit CLI mode)?

Interpret inputs conservatively:

- If the user wants to preserve an existing image while changing part of it, treat the request as an **edit**.
- If provided images are only style, composition, or mood references, treat the request as **generate with references**.
- Label every image by role: `edit target`, `style reference`, `composition reference`, or `insert/compositing input`.

## Default workflow

1. Confirm the task should produce a raster image rather than a vector or code-native asset.
2. Choose the top-level mode. Stay in explicit CLI mode unless the user explicitly chooses Codex, native generation, or Codex CLI.
3. Choose `generate` vs `edit`.
4. Collect prompt, exact text, constraints, avoid list, all input images, and execution parameters before running.
5. Use `references/prompting.md` for shared prompt structure and iteration rules. Use `references/sample-prompts.md` as recipes, not as mandatory augmentation.
6. In default CLI mode, use the bundled `scripts/image_gen.py` workflow and the CLI-only docs in `references/cli.md` and `references/image-api.md`.
7. In Codex mode from Claude/Copilot, use `codex exec` and instruct Codex to save the selected final into the workspace when the asset is project-bound.
8. Save project-bound finals into the workspace. Do not leave a project asset only in a tool-owned scratch location.
9. Do not overwrite an existing asset unless the user explicitly asked for replacement. Otherwise create a versioned sibling such as `hero-v2.png`.
10. Report the final saved path, the final prompt, and whether CLI mode or Codex mode was used.

## Ask for missing parameters

Do not silently choose user-facing image parameters when they are missing.

For **generate**, ask for any missing items that materially affect output:

- model
- size or aspect ratio
- quality
- whether the image should include text
- exact text, if text is required
- output format or transparency/background behavior when relevant
- style/use case if the request is too vague to execute reliably

For **edit**, ask for any missing items that materially affect output:

- model
- which image is the edit target and the role of any additional images
- exact edit scope (`change only X`)
- invariants (`keep Y unchanged`)
- quality
- `input_fidelity`
- whether a mask is required and the mask path if applicable
- output format or transparency/background behavior when relevant

If the user already supplied a parameter, do not re-ask it. If the user says "use defaults", you may use the CLI defaults documented in `references/cli.md`.

## Prompt shaping

Use short labeled specs when the request is non-trivial. Favor normalization over invention.

Recommended labels:

```text
Use case: <taxonomy slug>
Asset type: <where the image will be used>
Primary request: <main instruction>
Input images: <Image 1: role; Image 2: role> (optional)
Scene/backdrop: <environment>
Subject: <main subject>
Style/medium: <photo/illustration/3D/etc.>
Composition/framing: <wide/close/top-down; placement>
Lighting/mood: <lighting + mood>
Color palette: <palette notes>
Materials/textures: <surface details>
Text (verbatim): "<exact text>"
Constraints: <must keep / must avoid>
Avoid: <negative constraints>
```

Taxonomy slugs:

- Generate: `photorealistic-natural`, `product-mockup`, `ui-mockup`, `infographic-diagram`, `logo-brand`, `illustration-story`, `stylized-concept`, `historical-scene`
- Edit: `text-localization`, `identity-preserve`, `precise-object-edit`, `lighting-weather`, `background-extraction`, `style-transfer`, `compositing`, `sketch-to-render`

## CLI conventions

These conventions apply in the default CLI mode:

- Preferred package manager: `uv`
- Final outputs: `output/gpt-image/`
- Temporary JSONL or scratch files: `tmp/gpt-image/`
- `OPENAI_API_KEY` must be set locally
- Do not ask the user to paste the full API key into chat

Install dependency if needed:

```bash
uv pip install openai
pip install openai
```

Optional downscaling support:

```bash
uv pip install pillow
pip install pillow
```

## Resources

- `references/prompting.md` - shared prompt structure, specificity, and iteration rules
- `references/sample-prompts.md` - copy/paste prompt recipes shared by both modes
- `references/cli.md` - default CLI usage for `scripts/image_gen.py`
- `references/image-api.md` - GPT Image API and CLI-only parameter reference
- `scripts/image_gen.py` - bundled CLI implementation
