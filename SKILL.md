---
name: gpt-image
description: This skill should be used when the user asks to "generate an image", "edit this image", "make a hero banner", "create a product mockup", "remove the background", "make bitmap variations", or needs help creating raster visuals with a Codex-backed host-native path and an explicit GPT Image CLI fallback.
---

# GPT Image Skill

Generate or edit raster images from Claude Code, GitHub Copilot CLI, or Codex-style environments while preserving a consistent two-mode workflow.

## Goal

- Make one image skill work in Claude and Copilot without assuming those hosts can call an image tool directly.
- Prefer a Codex-backed host-native path for ordinary generation and editing.
- Preserve a bundled GPT Image CLI fallback for explicit script or API workflows.
- Keep raster image work separate from vector or code-native asset work.

## Two top-level modes

This skill has exactly two top-level modes:

- **Host-native mode (preferred):** in Claude Code and Copilot-style runtimes, invoke the local Codex CLI (`codex exec`) and let Codex handle the actual built-in image workflow. This path does not require `OPENAI_API_KEY`, but it does require a working `codex` installation and auth.
- **Explicit CLI fallback mode:** use `scripts/image_gen.py` only when the user explicitly asks for CLI mode, script mode, direct OpenAI/GPT Image API control, or CLI-only parameters. This path requires network access and `OPENAI_API_KEY`.

If this skill is used from Codex itself, host-native mode maps to Codex's built-in image workflow directly. If this skill is used from Claude or Copilot, host-native mode means "delegate to Codex CLI" rather than "call a host-native image tool directly."

Within the explicit CLI fallback only, the bundled CLI exposes three subcommands:

- `generate`
- `edit`
- `generate-batch`

Rules:

- In Claude Code and Copilot-style runtimes, do **not** claim that a direct host-native image tool exists unless the runtime actually exposes one.
- Default to host-native mode through Codex CLI for normal image generation and editing.
- Never switch to CLI fallback automatically.
- If Codex CLI is unavailable or cannot satisfy the request, tell the user the explicit CLI fallback exists and that it requires explicit opt-in plus `OPENAI_API_KEY`.
- Never modify `scripts/image_gen.py` unless the user explicitly asks to change the bundled CLI.

## Host-native mode in Claude/Copilot

When this skill runs in Claude Code or GitHub Copilot CLI, the preferred path is to shell out to Codex CLI.

Use these rules:

1. Use `codex exec` for non-interactive runs.
2. Pass `-C <workspace>` so Codex works in the current project.
3. If the current directory is not a Git repository, add `--skip-git-repo-check`.
4. For local input images, attach them with repeated `-i` flags and label each image's role in the prompt.
5. Ask Codex to save project-bound finals into the workspace before finishing, usually under `output/gpt-image/`.
6. Do not use fallback-only controls such as `quality`, `input_fidelity`, masks, `background`, output format, or direct API parameters unless the user explicitly chooses explicit CLI mode.

Recommended host-native command shapes:

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

Host-native output policy:

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

1. **Mode:** Codex-backed host-native default or explicit CLI fallback?
2. **Intent:** generate a new image or edit an existing one?
3. **Execution strategy:** one asset, a few targeted variants, or many jobs (`generate-batch` only in explicit CLI mode)?

Interpret inputs conservatively:

- If the user wants to preserve an existing image while changing part of it, treat the request as an **edit**.
- If provided images are only style, composition, or mood references, treat the request as **generate with references**.
- Label every image by role: `edit target`, `style reference`, `composition reference`, or `insert/compositing input`.

## Default workflow

1. Confirm the task should produce a raster image rather than a vector or code-native asset.
2. Choose the top-level mode. In Claude Code and Copilot-style runtimes, stay in host-native mode through Codex CLI unless the user explicitly chooses CLI, script, or API mode.
3. Choose `generate` vs `edit`.
4. Collect prompt, exact text, constraints, avoid list, and all input images.
5. Use `references/prompting.md` for shared prompt structure and iteration rules. Use `references/sample-prompts.md` as recipes, not as mandatory augmentation.
6. In host-native mode from Claude/Copilot, use `codex exec` and instruct Codex to save the selected final into the workspace when the asset is project-bound. If local direct file-path control or CLI-only parameter control is required, explain the explicit CLI fallback and wait for opt-in.
7. In explicit CLI mode, use the bundled `scripts/image_gen.py` workflow and the CLI-only docs in `references/cli.md` and `references/image-api.md`.
8. Save project-bound finals into the workspace. Do not leave a project asset only in a tool-owned scratch location.
9. Do not overwrite an existing asset unless the user explicitly asked for replacement. Otherwise create a versioned sibling such as `hero-v2.png`.
10. Report the final saved path, the final prompt, and whether host-native (Codex CLI) or explicit CLI mode was used.

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

## CLI fallback conventions

These conventions apply only in explicit CLI mode:

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
- `references/cli.md` - explicit CLI fallback usage for `scripts/image_gen.py`
- `references/image-api.md` - GPT Image API and CLI-only parameter reference
- `scripts/image_gen.py` - bundled CLI fallback implementation
