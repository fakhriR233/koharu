# Understanding Koharu (this fork)

> Written by Muse after exploring the repository at `fakhriR233/koharu @ 4a13353` (v0.83.4).
> This is a reading of the codebase, not official documentation.

## What it is

**Koharu is a local-first, ML-powered manga translator, written in Rust.**
You open manga pages (images, archives, PDFs), and it automatically detects text regions,
reads the Japanese/Chinese/Korean source text (OCR), translates it with an LLM,
erases the original text (inpainting), and typesets the translation back onto the page —
all with models running **locally on your machine** so your data stays private.

It's a full desktop application, not a script or a web service.

## Fork status

This fork (`fakhriR233/koharu`) is a **clean fork of `koharu-rs/koharu` at the v0.83.4
release commit** (`4a13353`). There are **no custom commits** on `main` — it is byte-identical
to upstream at that tag.

Upstream has since moved on: at the time of writing, upstream `main` is **14 commits ahead**
(mostly release chores, Tauri/CEF dependency bumps, and one feature: *"Scale pen size with
image resolution"*). So this fork is slightly behind but otherwise pristine — a good
starting point if you want to experiment or contribute.

## High-level architecture

```
┌─────────────────────────────────────────────────────────────┐
│  Desktop shell (Tauri v3 + CEF webview)  crates/koharu       │
│  ┌──────────────────────┐   ┌────────────────────────────┐  │
│  │ Frontend (Next.js)   │   │ Canvas (WASM + WebGPU)     │  │
│  │ packages/koharu      │◄─►│ crates/koharu-canvas       │  │
│  │ React UI, tools,     │   │ browser viewport, camera,  │  │
│  │ dialogs, preferences │   │ transient previews        │  │
│  └──────────┬───────────┘   └─────────────▲──────────────┘  │
│             │ Tauri commands (specta-typed IPC)              │
│  ┌──────────▼───────────────────────────────────────────┐   │
│  │ Backend (Rust)  crates/koharu-app                     │   │
│  │  commands: project, processing, editing, canvas,      │   │
│  │  import, output, preferences, agent, fonts, lifecycle │   │
│  └──────────┬───────────────────────────────────────────┘   │
│  ┌──────────▼───────────────────────────────────────────┐   │
│  │ Core: scene → pipeline → renderer → storage          │   │
│  │  koharu-scene     project/document model (ECS-like)  │   │
│  │  koharu-pipeline  ML orchestration & scheduling      │   │
│  │  koharu-ml        vision + diffusion models          │   │
│  │  koharu-translator local LLM / hosted providers      │   │
│  │  koharu-renderer  Vello-based page renderer          │   │
│  │  koharu-storage   .khrproj on-disk format            │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

Key design split (from `AGENTS.md`): **durable work stays native, transient interaction
stays in the browser.** The browser never discovers fonts, decodes assets, or commits
project state — React owns tools/gestures/selection, Rust owns the scene, undo,
persistence, and rendering.

## The translation pipeline

`crates/koharu-pipeline` is the heart of the product. The workflow is small and explicit —
no runtime graph library:

```
detection ─┬─► OCR ─► translation
           └─► inpainting
```

- **Detection** finds text regions, speech bubbles, and segmentation masks on the page.
- **OCR** reads source text from each detected region.
- **Translation** renders the translated text for each region (a normal pipeline processor,
  configured under `[pipeline.translation]`).
- **Inpainting** (parallel branch) reconstructs the artwork behind the source text so the
  translation can be placed on a clean background.

Execution details worth knowing:

- The unit of work is **one stage on one page**. Each stage result is committed immediately,
  so the UI refreshes per page and completed work survives if the user stops the run or a
  later model fails.
- Outputs are **optimistically rebased** onto the latest scene snapshot with conflict
  validation — a changed input or overlapping write fails instead of publishing stale output.
- The scheduler keeps a small rolling window of pages in project order; there is no global
  wave barrier. On an accelerator, ready jobs share **one execution lane** — the team
  benchmarked that overlapping heterogeneous models *increased* makespan 2.5–4× through
  GPU kernel/memory contention, so serial model execution is deliberately faster.
- Models are lazy-loaded and stay resident across pages; config changes publish a new
  immutable "stage generation" via `ArcSwap` so a running execution keeps its config.

## The ML stack

`crates/koharu-ml` contains one module per model, each with a consistent shape:
`model.rs` (network + weights), `processor.rs` (pre/post-processing), `config.rs`,
plus a `bin/` for standalone testing. Model inventory at this version:

| Task | Models |
|---|---|
| Layout / detection | Koharu Layout RF-DETR Seg 2XL, Comic Layout YOLO26s, Comic Text Detector, Comic Text Bubble Detector, Speech Bubble YOLO11n / YOLOv8m, PP-DocLayout V3 |
| OCR | PaddleOCR VL 1.6, Manga OCR, Baberu OCR, Hayai OCR, PP-OCRv6, Comic Onomatopoeia recognizer |
| Inpainting | FLUX.2 Klein, RORem Mixed, LaMa, AOT-GAN |
| Translation (local) | GGUF LLMs via llama.cpp: LFM 2.5, Ministral 3, Gemma 4 (+uncensored), Qwen 3.5/3.6/3.8 |
| Misc | Font detector, Manga text mask |

Backends (per `AGENTS.md` ML rules — consistent lifecycle, device abstraction at the
model boundary, no gradient tracking at inference):

- `koharu-torch` / `koharu-torch-sys` — LibTorch backend
- `koharu-llama` / `koharu-llama-sys` — llama.cpp backend for local GGUF inference
- `koharu-diffusion` / `koharu-diffusion-sys` — diffusion backend for generative inpainting
- `koharu-runtime` — hardware/device abstraction: **CUDA, ROCm/HIP, Metal, Vulkan, CPU**,
  plus model downloading (`download.rs`) from Hugging Face

## Translation providers

`crates/koharu-translator` exposes one segment-preserving translation interface over two
kinds of backends:

- **Local**: GGUF models run through the llama backend; the selected model stays resident.
- **Hosted**: OpenAI, Claude, Gemini, DeepSeek, Grok, MiniMax, OpenRouter, plus MT
  providers DeepL, Google Cloud Translation, Caiyun, and any OpenAI-compatible endpoint
  (incl. LM Studio).

Each provider module owns its connection config, defaults, and model catalog. Connection
settings live under `[providers]` in config; **credentials stay in the OS keychain**
(`koharu-secrets`), never in config files.

## Data model: scene + storage

- **`koharu-scene`** is the in-memory project model: an ECS-flavored document of entities
  with components (`text`, `spatial`, `layers`, `assets`, `analysis`, `provenance`,
  `groups`, `structure`). Edits are `patch`es applied to immutable `snapshot`s inside a
  `session`; the pipeline commits stage outputs as patches. Undo groups one pipeline
  invocation as a single undo step.
- **`koharu-storage`** is the on-disk format (`project.khrproj/`): two alternating complete
  state slots (`state-a.khr` / `state-b.khr`) for crash safety plus immutable
  content-addressed blobs named by **BLAKE3** hash under `blobs/`. No database, no op log —
  deliberately "document persistence, not version control."

## Rendering

- **`koharu-renderer`** turns one scene page into retained **Vello** content (and pixels on
  demand). The editor canvas, flat export, and PSD export all consume the same result —
  one source of truth for what a page looks like.
- **`koharu-canvas`** is the browser-side WebGPU/WASM viewport: it installs versioned
  frame manifests prepared natively, caches GPU-safe raster tiles, and handles camera and
  transient previews. Multilingual typesetting (auto-fit, font fallback, vertical CJK,
  RTL) and canvas compositing live here/around here.
- **`koharu-psd`** writes layered PSD export; `koharu-rasterizer` is the shared software
  rasterizer bound to the canvas.

## Frontend

- **`packages/koharu`** — Next.js + React app (the whole UI: editor, start screen,
  preferences, controls). Communicates with Rust via Tauri commands with
  **specta-generated TypeScript bindings** (types can't drift).
- **`packages/bridge`** — the `canvas.ts` / `protocol.ts` contract between React and the
  WASM canvas module.
- **`packages/ui`** — shared component library; **`packages/docs`** — the documentation
  site (EN/JA/ZH) at koharu.rs.

## Agent

`crates/koharu-agent` implements an **agent-based workflow**: project inspection, editing,
and pipeline control driven by an agent loop, with a `codex` submodule (auth, protocol,
streaming, token store) for Codex integration. This is how "AI operates Koharu" features
are built.

## Configuration & platform notes

- `koharu-config`: layered configuration (`[pipeline.translation]`, `[providers]`, …).
- Desktop shells per-OS in `crates/koharu` (`tauri.conf.json` + per-platform overrides);
  release via WinGet (Windows) and Homebrew (macOS); Linux builds too.
- Strict repo rules in `AGENTS.md`: no backward-compat shims (update all callers),
  ports stay traceable to pinned upstream implementations, comments explain *why* not
  *what*, debug profile for iteration, CEF remote debugging at `127.0.0.1:4000`.

## My take

This is a **serious, well-architected native ML application** — closer to a pro creative
tool (think Photoshop-meets-translator) than to a demo. A few things stand out:

1. **The pipeline scheduler is the cleverest part.** Serial accelerator execution beating
   parallel by 2.5–4× is the kind of counter-intuitive, measured decision that signals
   real engineering.
2. **The scene/storage split is disciplined.** Immutable snapshots + content-addressed
   blobs + crash-safe dual slots is a robust foundation for an editor.
3. **Local-first is the product thesis**, not a feature: vision models and LLMs run on
   your GPU so manga data never leaves the machine — with hosted providers as an option.
4. **It's big.** ~24 Rust crates + 4 TS packages, custom sys crates (torch, llama,
   diffusion), Tauri v3 alpha, WebGPU canvas. Building it needs Rust 1.97+, Bun, LLVM,
   and a GPU for any real ML work.

If you want to hack on it, the natural entry points are: `crates/koharu-pipeline`
(behavior), `crates/koharu-ml` (models), `packages/koharu/components/editor` (UI) —
and `git pull` upstream first, since you're 14 commits behind.
