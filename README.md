# GPT Image

A cross-runtime image skill for Claude Code and GitHub Copilot CLI that uses the bundled GPT Image CLI as the default path, with an explicit Codex mode when the user asks for it.

[中文说明 / Chinese README](./README_ZH.md)

## Features

- **CLI-first generation** - Use the bundled `scripts/image_gen.py` by default for generation and editing
- **Explicit Codex mode** - Use Codex's built-in image workflow only when the user explicitly wants Codex/native generation
- **Image editing** - Support direct edits, reference-guided generation, and compositing flows
- **Batch generation in CLI mode** - Use `generate-batch` for many prompts in the default CLI workflow
- **Shared prompting system** - Reuse the same prompt structure across both modes

## Two Modes

| Mode | When to use | Backend | Needs `OPENAI_API_KEY` |
|---|---|---|---|
| Explicit CLI (default) | Normal generation/edit requests unless the user explicitly asks for Codex/native generation | `scripts/image_gen.py` | Yes |
| Codex mode | User explicitly asks for Codex, native generation, or Codex CLI | `codex exec` -> Codex built-in image workflow | No |

## Prerequisites

### Explicit CLI mode

1. **Python environment** with the bundled script available
2. **OpenAI Python package**
3. **`OPENAI_API_KEY` exported locally**

Install dependencies if needed:

```bash
pip install openai # or use uv pip install openai
pip install pillow  # optional, for downscaling only
```

### Codex mode

1. **Codex CLI installed** via `npm i -g @openai/codex`
2. **Codex authenticated** via `codex login`
3. **This skill available to Claude/Copilot** in the runtime's custom skill path

Quick checks:

```bash
codex --help
codex exec --help
```

## Installation

### Claude Code

Copy this directory into your Claude skills directory:

```bash
mkdir -p ~/.claude/skills/gpt-image
cp -R /path/to/gpt-image/. ~/.claude/skills/gpt-image/
```

### Copilot / project-local skill setup

Keep the `gpt-image` folder in the location your Copilot-style runtime exposes through the `skill` tool. The exact path depends on the host integration, but the folder must include:

- `SKILL.md`
- `README.md`
- `references/`
- `scripts/image_gen.py`

## How It Works

### Default CLI mode

Unless the user explicitly asks for Codex/native generation, use the bundled CLI directly.

Set stable paths:

```bash
export GPT_IMAGE_SKILL="/absolute/path/to/gpt-image"
export IMAGE_GEN="$GPT_IMAGE_SKILL/scripts/image_gen.py"
```

Generate:

```bash
python "$IMAGE_GEN" generate \
  --prompt "A minimal modern blog header about AI agents" \
  --size 1536x1024 \
  --out output/gpt-image/blog-header.png
```

Edit:

```bash
python "$IMAGE_GEN" edit \
  --image input.png \
  --prompt "Replace only the background with a soft blue gradient; keep the subject unchanged" \
  --out output/gpt-image/input-edited.png
```

Batch:

```bash
python "$IMAGE_GEN" generate-batch \
  --input tmp/gpt-image/prompts.jsonl \
  --out-dir output/gpt-image/batch
```

Ask for missing parameters before running. At minimum, do not silently assume `model`, `size`, `quality`, text inclusion/exact copy, or edit fidelity/mask behavior unless the user explicitly says to use defaults.

### Codex mode in Claude/Copilot

Use this only when the user explicitly asks for:

- Codex mode
- native generation
- Codex CLI
- Codex's built-in image workflow

Recommended pattern:

```bash
codex exec -C "$PWD" --skip-git-repo-check \
  "Generate a blog header image for a post about AI agents. Save the selected final to output/gpt-image/blog-header.png and report the final path. Use Codex's built-in image workflow, not scripts/image_gen.py."
```

For local edit targets or reference images, attach files with `-i`:

```bash
codex exec -C "$PWD" --skip-git-repo-check -i input.png -i style-ref.png \
  "Image 1 is the edit target. Image 2 is a style reference. Replace only the background, keep the subject unchanged, and save the final to output/gpt-image/input-edited.png."
```

Notes:

- Use `-C "$PWD"` so Codex works in the current workspace.
- Add `--skip-git-repo-check` when you are outside a Git repo.
- Ask Codex to move or copy the selected final into your workspace if the image is project-bound.
- Codex may first write images under `$CODEX_HOME/generated_images/...`; do not leave project assets there permanently.

## Usage

Once installed, the skill should activate for requests like:

- "Make me a blog header"
- "Create a product mockup for this bottle"
- "Edit this image to remove the background"
- "Generate three visual directions for a landing-page hero"
- "Use CLI mode and make a transparent PNG"

### Mode selection rules

1. Default to **explicit CLI mode**.
2. Switch to **Codex mode** only after explicit user opt-in.
3. Never ask the user to paste API keys into chat.
4. Ask about missing image parameters before execution, unless the user explicitly says to use defaults.
5. For project assets, save finals under `output/gpt-image/`.

## Prompt Structure

For non-trivial requests, use a short labeled spec:

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

See:

- `references/prompting.md`
- `references/sample-prompts.md`

## Output Conventions

### Explicit CLI mode

- Finals: `output/gpt-image/`
- Temporary scratch files: `tmp/gpt-image/`

### Codex mode

- Codex may initially save images under `$CODEX_HOME/generated_images/...`
- Move or copy selected finals into the workspace before finishing
- Prefer `output/gpt-image/` for project assets

## Troubleshooting

| Problem | What to do |
|---|---|
| `codex` not found | Install Codex CLI and verify `codex --help` works |
| Codex not authenticated | Run `codex login` |
| `OPENAI_API_KEY` missing | Export it locally in your shell, not in chat |
| Use `OPENAI_BASE_URL` to choice third part api provider | Export it locally in your shell, not in chat |
| `openai` package missing | Run `uv pip install openai` |
| Missing model / size / quality / text settings | Ask the user before running, or use defaults only if they explicitly allow it |
| Need masks / `quality` / `input_fidelity` / `background` control | Use explicit CLI mode |
| Need a predictable workspace output path in Codex mode | Tell Codex to save the selected final into `output/gpt-image/` |

## Reference Map

- `SKILL.md` - runtime-facing instructions for CLI-first mode selection and Codex opt-in
- `references/prompting.md` - shared prompting guidance
- `references/sample-prompts.md` - copy/paste prompt recipes
- `references/cli.md` - default CLI usage
- `references/image-api.md` - explicit CLI/API parameter reference
- `scripts/image_gen.py` - bundled CLI implementation
