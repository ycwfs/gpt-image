# GPT Image

这是一个面向 Claude Code 与 GitHub Copilot CLI 的跨运行时图片技能，保留原有的双模式设计，但将优先使用的“宿主原生”路径重新定义为 **Codex CLI 桥接**。

## 功能特性

- **通过 Codex CLI 进行宿主原生生成** - 在 Claude/Copilot 中借助 Codex 内建图像工作流生成图片，无需 `OPENAI_API_KEY`
- **显式 GPT Image CLI 回退模式** - 当用户明确要求脚本/API 模式时，使用内置的 `scripts/image_gen.py`
- **图像编辑** - 支持直接编辑、基于参考图生成，以及合成类流程
- **显式 CLI 模式下的批量生成** - 仅在用户明确选择回退模式时使用 `generate-batch` 处理多条提示词
- **共享提示词体系** - 两种模式复用同一套提示词结构

## 两种模式

| 模式 | 适用场景 | 后端 | 是否需要 `OPENAI_API_KEY` |
|---|---|---|---|
| 宿主原生（推荐） | 来自 Claude 或 Copilot 的常规生成/编辑请求 | `codex exec` -> Codex 内建图像工作流 | 否 |
| 显式 CLI | 用户明确要求 CLI/脚本/API 模式，或需要 CLI 专属控制项 | `scripts/image_gen.py` | 是 |

## 前置要求

### 宿主原生模式

1. **已安装 Codex CLI**，可通过 `npm i -g @openai/codex` 安装
2. **已完成 Codex 登录认证**，执行 `codex login`
3. **当前运行时已能加载此技能**，即该技能位于 Claude/Copilot 的自定义技能目录中

快速检查：

```bash
codex --help
codex exec --help
```

### 显式 CLI 模式

1. **可用的 Python 环境**，且能访问内置脚本
2. **已安装 OpenAI Python 包**
3. **已在本地导出 `OPENAI_API_KEY`**

如有需要，安装依赖：

```bash
pip install openai # or use uv pip install openai
pip install pillow  # optional, for downscaling only
```

## 安装

### Claude Code

将本目录复制到 Claude 的技能目录：

```bash
mkdir -p ~/.claude/skills/gpt-image
cp -R /path/to/gpt-image/. ~/.claude/skills/gpt-image/
```

### Copilot / 项目本地技能配置

将 `gpt-image` 文件夹放在你的 Copilot 风格运行时可通过 `skill` 工具发现的位置。具体路径取决于宿主集成方式，但文件夹中至少应包含：

- `SKILL.md`
- `README.md`
- `references/`
- `scripts/image_gen.py`

## 工作方式

### Claude/Copilot 中的宿主原生模式

Claude 或 Copilot **不应假装自己能直接调用图像工具**。正确做法是调用 Codex CLI，再由 Codex 使用其内建图像工作流完成生成或编辑。

推荐模式：

```bash
codex exec -C "$PWD" --skip-git-repo-check \
  "Generate a blog header image for a post about AI agents. Save the selected final to output/gpt-image/blog-header.png and report the final path. Use Codex's built-in image workflow, not scripts/image_gen.py."
```

如果是本地图像编辑目标或参考图，可通过 `-i` 传入文件：

```bash
codex exec -C "$PWD" --skip-git-repo-check -i input.png -i style-ref.png \
  "Image 1 is the edit target. Image 2 is a style reference. Replace only the background, keep the subject unchanged, and save the final to output/gpt-image/input-edited.png."
```

说明：

- 使用 `-C "$PWD"` 让 Codex 在当前工作区中执行
- 若当前目录不是 Git 仓库，加上 `--skip-git-repo-check`
- 如果图片是项目资产，要求 Codex 在结束前将选中的最终结果移动或复制到工作区
- Codex 可能先把图片写到 `$CODEX_HOME/generated_images/...`；不要把项目最终资产长期留在那里

### 显式 CLI 模式

仅当用户明确要求以下内容时使用：

- CLI 模式
- 脚本模式
- 直接控制 GPT Image API
- 需要 CLI 专属参数，例如 `quality`、`input_fidelity`、mask、`background`、`generate-batch`、`model` 或输出格式控制

设置稳定路径：

```bash
export GPT_IMAGE_SKILL="/absolute/path/to/gpt-image"
export IMAGE_GEN="$GPT_IMAGE_SKILL/scripts/image_gen.py"
```

生成：

```bash
python "$IMAGE_GEN" generate \
  --prompt "A minimal modern blog header about AI agents" \
  --size 1536x1024 \
  --out output/gpt-image/blog-header.png
```

编辑：

```bash
python "$IMAGE_GEN" edit \
  --image input.png \
  --prompt "Replace only the background with a soft blue gradient; keep the subject unchanged" \
  --out output/gpt-image/input-edited.png
```

批量生成：

```bash
python "$IMAGE_GEN" generate-batch \
  --input tmp/gpt-image/prompts.jsonl \
  --out-dir output/gpt-image/batch
```

## 使用方式

安装完成后，这个技能应能自动匹配如下请求：

- “给我做一个博客头图”
- “给这个瓶子做一个产品 mockup”
- “编辑这张图，去掉背景”
- “生成三个 landing-page hero 的视觉方向”
- “用 CLI 模式生成一张透明 PNG”

### 模式选择规则

1. 默认使用通过 Codex CLI 实现的 **宿主原生模式**
2. 只有在用户明确选择后，才切换到 **显式 CLI 模式**
3. 永远不要要求用户把 API key 粘贴到聊天中
4. 对于项目资产，最终结果应保存在 `output/gpt-image/` 下

## 提示词结构

对于较复杂的请求，建议使用简短的标签化规格：

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

参考：

- `references/prompting.md`
- `references/sample-prompts.md`

## 输出约定

### 宿主原生模式

- Codex 可能先将图片保存到 `$CODEX_HOME/generated_images/...`
- 在结束前应将选中的最终结果移动或复制到工作区
- 对项目资产优先使用 `output/gpt-image/`

### 显式 CLI 模式

- 最终输出：`output/gpt-image/`
- 临时文件：`tmp/gpt-image/`

## 故障排查

| 问题 | 处理方式 |
|---|---|
| 找不到 `codex` | 安装 Codex CLI，并确认 `codex --help` 可用 |
| Codex 未登录 | 执行 `codex login` |
| 缺少 `OPENAI_API_KEY` | 在本地 shell 中导出，不要发到聊天里 |
| 需要通过 `OPENAI_BASE_URL` 选择第三方 API 提供商 | 在本地 shell 中导出，不要发到聊天里 |
| 缺少 `openai` 包 | 运行 `uv pip install openai` |
| 需要 mask / `quality` / `input_fidelity` / `background` 控制 | 使用显式 CLI 模式 |
| 需要可预测的工作区输出路径 | 在提示中要求 Codex 将最终结果保存到 `output/gpt-image/` |

## 参考映射

- `SKILL.md` - 面向 Claude/Copilot/Codex 运行时的说明
- `references/prompting.md` - 共享提示词指导
- `references/sample-prompts.md` - 可直接复用的提示词示例
- `references/cli.md` - 显式 CLI 回退模式说明
- `references/image-api.md` - 显式 CLI/API 参数参考
- `scripts/image_gen.py` - 内置的显式 CLI 实现
