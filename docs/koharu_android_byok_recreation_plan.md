# Android Manga Translator Recreation Plan
## Koharu-inspired workflow, local vision/inpainting, remote BYOK translation

**Updated:** 2026-10-05

## Executive decision

The Android project should **re-create Koharu's workflow rather than try to port the entire Koharu desktop runtime**.

The target product is:

```text
Import manga page / CBZ
        ↓
Local page layout + text/bubble detection
        ↓
Local crop OCR
        ↓
Remote translation through a user-supplied OpenAI-compatible API
        ↓
Local text-mask cleanup / inpainting
        ↓
Local typesetting + manual correction
        ↓
Export PNG / WEBP / CBZ
```

Koharu remains useful as a behavioral reference: its separation between detection, OCR, translation, inpainting and rendering is good. But the Android app should be designed around mobile constraints from the beginning instead of carrying desktop Torch, llama.cpp, diffusion, Tauri or CEF assumptions into the phone build.

The most important revision from the previous plan is:

> **Use ONNX Runtime as the Android inference platform, but do not treat Koharu's old ONNX bundle as the current model stack.**

The old Koharu ONNX artifacts are mostly 2025-era exports. ONNX itself is absolutely not abandoned, and newer ONNX exports now exist for the current Koharu layout model, Hayai Nova, AOT manga inpainting and other relevant models.

---

# 1. What to copy from Koharu, and what not to copy

## Keep the ideas

Keep these concepts:

- independent processing stages
- stable text-region IDs
- text/bubble masks stored separately from translated text
- OCR text editable before translation
- translation editable before rendering
- local inpainting independent from translation
- page/project history
- selective processing of a page or selected regions
- re-running one stage without destroying unrelated work

## Do not make Koharu internals a requirement

For the first Android version, I would **not** make Rust, Koharu crates, Tauri, Torch, llama.cpp or stable-diffusion.cpp mandatory.

A much simpler first implementation is:

```text
Kotlin + Jetpack Compose
        │
        ├─ Android project/database layer
        ├─ ONNX Runtime Android
        ├─ image/mask processing
        ├─ OpenAI-compatible HTTP client
        └─ Android Canvas / Skia rendering
```

If a specific Koharu algorithm later turns out to be worth reusing directly, it can be moved behind JNI/UniFFI then. Do not introduce JNI on day one unless it buys something measurable.

This makes the app a **functional Android clone / reimplementation**, not a brittle desktop source port.

---

# 2. Is the Koharu ONNX path abandoned?

## Short answer

**No. ONNX Runtime is current and viable on Android. The old Koharu ONNX model repository is stale. Those are different questions.**

### Old Koharu ONNX bundle

`mayocream/koharu` contains:

- `comictextdetector.onnx`
- `manga-ocr.onnx`
- `lama-manga.onnx`

The comic detector and MangaOCR files were uploaded in **April 2025**. Separate MangaOCR and LaMa ONNX repositories were created around **May 2025**.

They are useful as proof that those networks can run under ONNX Runtime, but I would **not freeze the Android application around those exact files**.

### Current ONNX evidence

A community conversion now exists for Koharu's current layout detector:

`ShiniShiho/koharu-layout-rfdetr-seg-2xl-1152-onnx`

It is an FP32, opset-17 ONNX export of the current Koharu RF-DETR Seg 2XL layout model and reports parity against the source graph:

- 100% top-class agreement on its validation check
- maximum confidence drift around `2.25e-5`
- thresholded mask IoU around `0.99979`

It outputs the same useful classes as current Koharu:

- text
- onomatopoeia / SFX
- speech bubble
- panel

This is very important: **modern Koharu-style detection is already ONNX-exportable.**

### ONNX Runtime Android itself

ONNX Runtime still ships a normal Android package and has active Android hardware paths. Current documentation includes:

- standard `onnxruntime-android`
- CPU fallback
- NNAPI support
- Qualcomm QNN support
- an Android ARM64 Maven package for QNN

Current QNN documentation pairs the Android package with ONNX Runtime 1.29.0 / QNN EP 2.7.0.

So the runtime is not the problem. The real work is choosing/exporting models that make sense on mobile and validating operator support/performance.

---

# 3. Recommended Android inference strategy

Use **one runtime abstraction** in the app:

```kotlin
interface LocalModel<I, O> {
    suspend fun load()
    suspend fun run(input: I): O
    suspend fun unload()
}
```

Then have implementations such as:

```text
LayoutDetectorOnnx
HayaiOcrOnnx
AotInpainterOnnx
LamaInpainterOnnx
```

The rest of the app must not care which model is behind a stage.

## Runtime order

Start with:

1. **ONNX Runtime CPU** for correctness.
2. Test **NNAPI** where it actually improves the graph.
3. Add **QNN** on supported Snapdragon devices if profiling justifies it.
4. Consider ExecuTorch/LiteRT only for a specific model where ONNX Runtime is a proven bottleneck.

Do not make the application architecture depend on one vendor NPU.

---

# 4. Detection: two realistic Android profiles

## Profile A — Koharu parity

Use the current **Koharu Layout RF-DETR Seg 2XL ONNX**.

Advantages:

- closest to what current Koharu desktop is trying to do
- one model produces text, SFX, bubble and panel detections
- instance masks are available
- current export has already been parity-checked against the PyTorch graph

Disadvantages:

- 1152 × 1152 FP32 input
- 2XL detector is not designed primarily around phone efficiency
- current model card explicitly recommends CUDA on desktop
- it may be slow and memory-heavy on some phones

This should be the **Quality / Koharu-compatible** detector profile, not necessarily the only detector.

## Profile B — Mobile fast path

Use smaller ONNX components:

```text
Comic Text Detector ONNX
        +
small bubble detector / segmentation model
        +
optional panel detector
```

A current third-party manga translator model repository exposes a Comic Text Detector ONNX around 77 MB, producing both YOLO boxes and a pixel text mask.

This is less elegant than the single RF-DETR model, but it is easier to optimize for phones.

## Product recommendation

Ship the detector behind a profile selector:

```text
Detection mode
(*) Balanced / Mobile
( ) Koharu Quality
```

Do not expose the underlying architecture names to normal users unless they open Advanced settings.

## Long-term improvement

If RF-DETR quality is clearly better but 2XL is too expensive, train or distill a **smaller detector on the same four-class task** instead of permanently falling back to old Comic Text Detector behavior.

RF-DETR upstream now supports ONNX plus multiple mobile/edge export formats, so a smaller mobile-specific model is a realistic later project.

---

# 5. OCR: Hayai should become the main mobile target

The previous plan treated MangaOCR as the main OCR candidate. I would change that.

## Koharu's current Hayai link is already behind

Koharu currently links to `hayai-ocr-v2`, but that model card now explicitly says it is outdated and points to **Hayai OCR v2.5 Nova**.

So an Android reimplementation should not blindly mirror Koharu's pinned OCR choice.

## Why Hayai Nova fits this project

Hayai OCR v2.5 Nova is about 150M parameters and is specifically designed for **crop-level OCR** across Japanese, Chinese, Korean and English.

That is exactly what this pipeline needs:

```text
full manga page
    ↓ detector
text/bubble crop
    ↓ Hayai
recognized text
```

The current model card reports, on its Manga benchmark at the 512-patch setting:

- about 3.10% CER
- about 80.68% exact match

It also reports better crop-level results than MangaOCR on that benchmark.

Most importantly for mobile, v2.5 Nova reduces decoder vision tokens by 4× compared with the earlier architecture, reducing prefill/KV cost.

## Current ONNX export exists

A community ONNX conversion was updated in October 2026:

`bixii/hayai-ocr-v2.5-nova-onnx`

It splits inference into:

- vision encoder ONNX
- autoregressive decoder ONNX
- tokenizer JSON

The converter reports that greedy output matched the source PyTorch model on all 64 crops in its check set.

That makes Hayai a **real Android candidate now**, rather than a hypothetical future conversion.

## Important caveat

The current ONNX export is still FP32 and large:

- vision graph: roughly 343 MB
- decoder graph: roughly 256 MB

So the application should not assume that "ONNX" means "small".

The first optimization work should be:

1. get the FP32 graph correct on Android CPU;
2. profile peak RSS and crop latency;
3. test FP16 where the target backend actually benefits;
4. test INT8/QDQ carefully;
5. compare OCR accuracy after quantization;
6. keep an optional smaller OCR profile if needed.

Hayai also requires application-side NaFlex preprocessing, mRoPE construction, greedy decoding and KV-cache management. The ONNX export makes this feasible, but it is not a one-call image classifier.

## OCR profiles I would ship

```text
OCR mode
(*) Hayai Nova — recommended
( ) MangaOCR — compatibility fallback
( ) Lite OCR — later, for very low-memory devices
```

For a modern high-end Android phone, I would make Hayai the first serious target.

---

# 6. Inpainting: change the default from LaMa to AOT

This is another place where the Android app can improve on a literal Koharu clone.

## AOT is much more attractive on mobile

Koharu's current AOT model is only about **5.68M parameters**.

A current ONNX manga-translator model repository provides an AOT ONNX graph around **23 MB**, with dynamic height/width support (multiple-of-8 requirement).

That is dramatically easier to deploy on Android than a 200+ MB LaMa model.

## Recommended inpainting profiles

### Default: AOT

Use for:

- ordinary speech bubbles
- captions
- simple screentone
- most text removal
- interactive/manual remove brush

### Quality fallback: LaMa Manga

Keep LaMa as an optional downloaded quality model for difficult cases:

- line art crossing text
- complicated texture
- larger removal areas
- cases where AOT visibly smears the background

The old LaMa ONNX proves exportability, but the safer approach is to **re-export the current Koharu LaMa SafeTensors weights and parity-test the result** rather than relying permanently on a 2025 artifact.

## Do not put generative diffusion in v1

Do not make FLUX/RORem-class inpainting part of the Android MVP.

The phone app should optimize for:

- instant region cleanup
- predictable memory
- offline behavior
- easy undo

AOT + optional LaMa is enough for the first serious version.

---

# 7. The resulting local model stack

My current preferred stack is:

| Stage | Default | Quality/alternate | Why |
|---|---|---|---|
| Layout/text detection | mobile detector profile | current Koharu RF-DETR Seg 2XL ONNX | speed vs Koharu parity |
| OCR | **Hayai OCR v2.5 Nova ONNX** | MangaOCR ONNX | Hayai is current and crop-oriented |
| Text mask | detector mask + local morphology | RF-DETR instance mask | no cloud required |
| Inpainting | **AOT ONNX** | LaMa Manga ONNX | AOT is far smaller |
| Translation | remote OpenAI-compatible API | same | no local LLM |
| Rendering | Android Canvas/Skia | same | native mobile UI |

All image data remains local by default.

Only OCR text is sent to the translation provider unless the user explicitly enables multimodal image context later.

---

# 8. Translation provider design: OpenAI-compatible first

The app should stop thinking in terms of hard-coded provider brands.

The primary abstraction should be:

```text
OpenAI-compatible endpoint
```

Provider presets such as OpenCode Go, ClinePass or NanoGPT can simply pre-fill known values where appropriate.

## Core provider settings

```text
Profile name
Base URL
API key
Protocol
Model discovery mode
Model ID
Reasoning mode
Reasoning effort
Extra headers (advanced)
Extra request JSON (advanced)
```

Suggested model:

```kotlin
data class ProviderProfile(
    val id: String,
    val name: String,
    val baseUrl: String,
    val protocol: ApiProtocol,
    val secretAlias: String,
    val modelDiscovery: ModelDiscovery,
    val manualModelId: String?,
    val reasoning: ReasoningConfig,
    val extraHeaders: Map<String, String>,
    val extraBodyJson: String?
)
```

---

# 9. Do not require `/models`

This is important.

Many OpenAI-compatible services expose something equivalent to:

```text
GET /v1/models
```

but not all do, and some gateways intentionally expose only inference endpoints.

Therefore model discovery must be optional.

## Connection flow

When a user adds a provider:

```text
1. Enter Base URL
2. Enter API key
3. Try model discovery if enabled
4. If model listing works -> show picker
5. If it fails -> show manual Model ID text field
6. Test the selected/manual model with a tiny request
7. Save profile
```

The UI should never dead-end because `/models` is unavailable.

## Model discovery setting

```text
Model source
(*) Auto — try model endpoint, allow manual fallback
( ) Manual model ID
```

Provider presets can optionally specify their known model-list path, but the generic provider must not assume one.

## Manual model IDs are first-class

A manually entered model must behave exactly like a discovered model.

Do not make it an unsupported "custom" corner case.

This is what allows the app to work with gateways such as:

- OpenCode Go
- ClinePass-compatible endpoints
- NanoGPT
- self-hosted vLLM
- LiteLLM gateways
- custom OpenAI-compatible proxies
- future providers that do not implement model listing

---

# 10. Reasoning / effort support

Reasoning support varies by API and model, so it must be **optional and capability-driven**.

Normal UI:

```text
Reasoning
(*) Auto / provider default
( ) Off / omit parameter
( ) Minimal
( ) Low
( ) Medium
( ) High
( ) Extra High
```

Do not send a reasoning parameter unless:

- the provider preset says it is supported, or
- the user explicitly enables it for a generic provider.

Why: OpenAI-compatible gateways are not perfectly identical. Sending an unsupported reasoning field can produce a 400 even if the actual chat endpoint otherwise works.

## Protocol mapping

The request builder should map the user's semantic setting onto the selected wire protocol.

Conceptually:

```text
OpenAI Chat-style
    reasoning_effort: "high"       # where supported

OpenAI Responses-style
    reasoning: { effort: "high" }

Provider-specific compatible gateway
    mapped by profile/adapter
```

For unknown providers, default to **omit**.

## Capability cache

After a successful test request, cache capabilities per model/profile:

```text
supportsStructuredOutput
supportsReasoning
supportedReasoningLevels
supportsResponsesApi
supportsChatCompletions
supportsVision
```

Do not guess from the model name alone when a real test can establish behavior.

---

# 11. Temperature, top-p and other Koharu generation settings

For this Android product, **remove temperature/top-p from the normal translation settings UI**.

That is the right simplification.

Why:

- translation should be stable and reproducible
- some reasoning models do not accept the same sampling controls
- different OpenAI-compatible providers interpret or reject parameters differently
- users care much more about model, prompt and reasoning effort than temperature for this workflow

However, I would **not delete support from the network layer completely**.

Keep one Advanced field:

```text
Extra request parameters (JSON)
```

Example:

```json
{
  "temperature": 0.2,
  "top_p": 0.95
}
```

Normal users never need to see it. Power users can still work around unusual gateways without waiting for an app update.

Default behavior should be:

> **Do not send temperature or top-p at all unless a provider preset or advanced override explicitly requests them.**

This is more compatible than sending Koharu's desktop defaults to every model.

---

# 12. Translation request format

The translator should batch ordered regions and preserve IDs.

Example internal object:

```json
{
  "source_language": "ja",
  "target_language": "en",
  "page": 12,
  "segments": [
    {"id": "p12-r04", "text": "..."},
    {"id": "p12-r05", "text": "..."}
  ],
  "glossary": [],
  "context": []
}
```

Expected output:

```json
{
  "translations": [
    {"id": "p12-r04", "text": "..."},
    {"id": "p12-r05", "text": "..."}
  ]
}
```

Use native JSON schema/structured output when the provider supports it.

Otherwise:

1. request plain JSON;
2. parse locally;
3. validate every region ID;
4. run one repair request if malformed;
5. never silently shift translation N onto bubble N+1.

For manga translation, correctness of segment mapping matters more than streaming token UX.

Use non-streaming requests first. Streaming can be added later.

---

# 13. Improve translation beyond desktop-style "send text to LLM"

This Android clone can improve the workflow significantly here.

## Project glossary

Store:

- character names
- honorific rules
- recurring terminology
- spellings
- gender/pronoun notes entered by the user
- translation style notes

Inject only the relevant subset into each page request.

## Previous-page context

Keep a small rolling context window containing:

- previous source lines
- accepted translations
- character names currently active

This helps pronouns, sentence continuation and recurring jokes without sending the entire manga every time.

## Translation versions

Never overwrite a user-approved translation immediately.

Store versions:

```text
OCR source
Translation A — provider/model/time
Translation B — provider/model/time
Manual final
```

This makes A/B testing OpenCode Go vs NanoGPT vs another endpoint much nicer on mobile.

## Optional proofread pass

Add a button:

```text
Proofread current page
```

It can use the same remote BYOK provider and compare:

- OCR source
- first translation
- surrounding lines

This gives many of Koharu's agent/proofreading benefits without running a local LLM on the phone.

---

# 14. API-key security

API keys are remote credentials, but they should still be treated as secrets.

Use:

- Android Keystore-backed encryption
- one alias per provider profile
- no key in Room/database rows
- no key in project export
- no Authorization header in logs
- no key in crash telemetry
- optional biometric/device-credential protection for viewing/replacing a key

The provider profile stores only a reference such as:

```text
secretAlias = manga_provider_7
```

---

# 15. Android architecture I would build now

```text
app/
├─ ui/
│  ├─ project/
│  ├─ editor/
│  ├─ processing/
│  ├─ provider/
│  └─ settings/
│
├─ domain/
│  ├─ Project.kt
│  ├─ Page.kt
│  ├─ TextRegion.kt
│  ├─ TranslationVersion.kt
│  └─ PipelineState.kt
│
├─ pipeline/
│  ├─ DetectStage.kt
│  ├─ OcrStage.kt
│  ├─ TranslateStage.kt
│  ├─ InpaintStage.kt
│  └─ RenderStage.kt
│
├─ inference/
│  ├─ OnnxSessionManager.kt
│  ├─ detector/
│  ├─ hayai/
│  └─ inpainting/
│
├─ providers/
│  ├─ OpenAiCompatibleClient.kt
│  ├─ ChatCompletionsProtocol.kt
│  ├─ ResponsesProtocol.kt
│  ├─ ProviderCapabilityProbe.kt
│  └─ ProviderProfile.kt
│
├─ storage/
│  ├─ Room database
│  ├─ Project files
│  └─ ModelStore
│
└─ rendering/
   ├─ PageCanvas.kt
   ├─ TextLayout.kt
   └─ ExportRenderer.kt
```

This is easier to debug and ship than a Kotlin + Rust + webview + JNI stack.

Rust can still be introduced later for a proven hotspot.

---

# 16. Pipeline should be a cached dependency graph

This is one of the biggest improvements I would make over a naive clone.

Each stage stores its input fingerprint.

```text
Detection depends on source image + detector settings
OCR depends on detection crops + OCR model/settings
Translation depends on accepted OCR + glossary + provider/model/prompt
Inpainting depends on source image + removal mask + inpainter
Rendering depends on cleanup image + accepted translation + layout
```

If the user edits a translation, the app should **not rerun OCR or inpainting**.

If the user corrects OCR, only translation and rendering become stale.

If the user adjusts a mask, only inpainting/rendering become stale.

This saves battery and makes Android use feel much faster.

---

# 17. Region-level processing should be first-class

Desktop tools often assume page/project processing.

On a phone, the most useful interaction is frequently:

```text
Tap one bubble
 -> Redetect region
 -> OCR again
 -> Translate again
 -> Clean again
```

Every pipeline command should support:

```text
selected region
selected page
selected pages
whole project
```

This is especially useful when only one stylized SFX or difficult bubble failed.

---

# 18. Mobile UI improvements

## Editor layout

```text
Top bar
  Project | page | undo | redo | export

Canvas
  pinch zoom / pan
  source
  cleanup layer
  text overlays
  detector overlays

Bottom modes
  Select | OCR | Text | Remove | Brush | Erase

Inspector bottom sheet
  Source OCR
  Translation
  Translation version
  Region type
  Reading order
  Font / alignment / size
```

## One-tap processing

A normal user should see:

```text
Translate Page
```

not four mandatory technical buttons.

Internally it runs:

```text
Detect if stale
OCR if stale
Translate if stale
Inpaint if stale
Render
```

Advanced users can open the pipeline sheet and run stages manually.

---

# 19. Manual painting / cleanup

Manual image editing should stay completely local.

Tools:

- remove/inpaint brush
- ordinary paint brush
- eraser
- clone/stamp later
- color picker
- mask grow/shrink
- undo/redo

For the remove brush:

1. create/update a binary mask;
2. compute the smallest padded crop around the dirty region;
3. run AOT/LaMa only on that crop;
4. merge the output into the cleanup layer;
5. keep the source image immutable.

Do not rerun full-page inpainting for a tiny correction.

---

# 20. Model manager

Do not bundle hundreds of megabytes of models into the base APK.

The APK should contain only the app and possibly a tiny bootstrap model if absolutely necessary.

Downloaded model manifest example:

```json
{
  "id": "hayai-nova-onnx-fp32",
  "stage": "ocr",
  "version": 1,
  "files": [
    {"name": "vision.onnx", "sha256": "..."},
    {"name": "decoder.onnx", "sha256": "..."},
    {"name": "tokenizer.json", "sha256": "..."}
  ],
  "runtime": "onnxruntime",
  "requiredMemoryTier": "high"
}
```

Model manager features:

- download on demand
- exact download size before starting
- SHA-256 verification
- resume download
- remove model
- show model version
- allow experimental model channel

This allows detector/OCR/inpainting models to evolve independently of Play Store APK releases.

---

# 21. Memory, thermal and Android lifecycle handling

Do not keep every model resident.

Initial policy:

```text
load detector
process requested pages
unload detector

load Hayai
OCR requested regions
unload Hayai

remote translation

load AOT
inpaint requested regions
unload AOT
```

For interactive editing, retain a model only if there is sufficient memory headroom.

## Long-running jobs

For multi-page CBZ processing:

- use a foreground service while actively inferencing
- persist stage progress after every page
- allow pause/cancel
- recover after process death
- do not trust one in-memory task for a 100-page manga

## Thermal behavior

Provide:

```text
Performance mode
(*) Balanced
( ) Fast
( ) Battery saver
```

Balanced can pause briefly between heavy pages when the device reports severe thermal status.

---

# 22. Model validation before shipping

Never trust a conversion just because ONNX Runtime can load it.

Create a fixed golden manga set and compare against the source model.

## Detector

Compare:

- class
- confidence
- boxes
- instance mask IoU

## Hayai

Compare:

- exact greedy output
- normalized output
- CER
- punctuation
- vertical text
- furigana
- SFX

## Inpainting

Compare:

- residual text pixels
- line-art damage
- screentone continuity
- seam visibility

Every quantized model should have an accuracy regression budget.

Example policy:

> Do not ship a mobile quantization that saves 40% latency but noticeably damages furigana or stylized SFX recognition.

---

# 23. Development sequence I recommend

## Phase 1 — Android shell + project model

Build:

- image import
- page list
- zoomable canvas
- region overlay format
- project persistence
- undo/redo

No ML yet.

## Phase 2 — exact current detector proof

Integrate ONNX Runtime Android and run the current RF-DETR Koharu-layout ONNX.

Goal:

> prove that modern Koharu-style layout analysis can run locally on the target phone.

Measure:

- load time
- per-page latency
- peak RSS
- thermal behavior

Do not guess whether 2XL is too slow; benchmark it.

## Phase 3 — Hayai Nova ONNX

Implement:

- crop generation
- NaFlex preprocessing
- vision encoder
- decoder loop
- KV cache
- tokenizer
- normalized OCR output

Goal:

> tap a detected region and get editable Japanese text entirely offline.

## Phase 4 — OpenAI-compatible BYOK translation

Build only the generic provider first.

Required settings:

```text
Base URL
API key
Auto/manual model ID
Chat Completions or Responses protocol
Reasoning Auto/Off/Effort
Test connection
```

Then add presets for services such as OpenCode Go, ClinePass and NanoGPT as convenience configuration, not separate translation engines.

## Phase 5 — AOT inpainting

Use the small ONNX AOT model.

Goal:

> selected region can be cleaned locally and interactively.

## Phase 6 — typesetting/export

Add:

- automatic fit
- stroke/outline
- alignment
- rotation
- font selection
- PNG/WEBP export

## Phase 7 — mobile optimization

Only now optimize:

- FP16
- INT8/QDQ
- NNAPI
- QNN
- smaller detector
- model caching
- batch scheduling

## Phase 8 — advanced workflow

Add:

- CBZ import/export
- glossary
- translation memory
- provider A/B comparison
- proofread pass
- region relationship / bubble association
- more automatic typesetting

---

# 24. First acceptance test

The first real build is successful when this works on a physical Android phone:

1. Enable airplane mode.
2. Import a Japanese manga page.
3. Tap **Analyze**.
4. Text, bubbles and optionally panels are detected locally.
5. Tap a bubble.
6. Hayai produces editable Japanese OCR locally.
7. Turn network back on.
8. Configure a generic OpenAI-compatible endpoint.
9. If model discovery works, choose a model.
10. If it does not, manually enter the model ID.
11. Choose reasoning effort if that endpoint/model supports it.
12. Tap Translate.
13. Translation comes back mapped to the correct region ID.
14. Disable network again.
15. Run AOT cleanup locally.
16. Render translated text.
17. Manually correct text position/mask if needed.
18. Export PNG.

If those 18 steps work, the core product is proven.

---

# 25. What I would improve versus simply cloning Koharu

If the goal is "Koharu on Android," I would deliberately make these improvements instead of chasing desktop parity first:

1. **OpenAI-compatible BYOK is the default translation architecture**, not one provider among many local/cloud backends.
2. **Manual model ID is first-class**, so providers without `/models` still work normally.
3. **Reasoning effort is capability-aware** and optional.
4. **Temperature/top-p disappear from the normal UI**.
5. **Hayai Nova becomes the preferred OCR target** instead of preserving an older pinned OCR model.
6. **AOT becomes the default mobile inpainter**, with LaMa as quality mode.
7. **Pipeline caching prevents unnecessary reruns**.
8. **Region-level processing is as important as whole-page processing**.
9. **Translation version history enables provider/model comparison**.
10. **Project glossary + prior-page context are built in from the start**.
11. **Models update independently from the APK** through a signed/hashed model manifest.
12. **All images remain local by default**; the network boundary is explicit and limited to translation text.
13. **Android-native UI and rendering** replace a desktop webview architecture.
14. **Performance profiles** make the same app usable across different phone tiers.

The end product should feel less like "desktop Koharu squeezed onto a phone" and more like a manga translation editor designed for touch from the beginning.

---

# 26. Recommended v1 stack

```text
Android app
  Kotlin
  Jetpack Compose
  Room / app-private project files
  Android Canvas / Skia

Local inference
  ONNX Runtime Android

Detection
  Balanced profile: lighter detector stack
  Quality profile: current Koharu Layout RF-DETR Seg 2XL ONNX

OCR
  Hayai OCR v2.5 Nova ONNX
  MangaOCR fallback

Inpainting
  AOT ONNX default
  LaMa Manga optional quality pack

Remote translation
  Generic OpenAI-compatible client
  Chat Completions + Responses protocol support
  automatic / manual model ID
  optional reasoning effort
  no default temperature/top-p

Security
  Android Keystore-backed API-key storage

Network boundary
  OCR text only by default
```

That is the architecture I would implement today.

---

# Sources checked for this revision

## Koharu

- Official repository: https://github.com/koharu-rs/koharu
- Current Koharu release page: https://github.com/koharu-rs/koharu/releases
- Current Koharu Layout RF-DETR model: https://huggingface.co/mayocream/koharu-layout-rfdetr-seg-2xl-1152
- Validated ONNX conversion of current layout model: https://huggingface.co/ShiniShiho/koharu-layout-rfdetr-seg-2xl-1152-onnx
- Older Koharu ONNX bundle: https://huggingface.co/mayocream/koharu
- MangaOCR ONNX repository: https://huggingface.co/mayocream/manga-ocr-onnx
- LaMa Manga current weights: https://huggingface.co/mayocream/lama-manga
- AOT Manga current weights: https://huggingface.co/mayocream/aot-inpainting

## Hayai

- Hayai v2 model currently linked by Koharu; model card marks it outdated: https://huggingface.co/JustANormalTinkerer/hayai-ocr-v2
- Current Hayai OCR v2.5 Nova: https://huggingface.co/JustANormalTinkerer/hayai-ocr-v2.5-nova
- Current community ONNX conversion: https://huggingface.co/bixii/hayai-ocr-v2.5-nova-onnx

## ONNX / mobile

- ONNX Runtime Android install: https://onnxruntime.ai/docs/install/
- ONNX Runtime Android build / NNAPI / QNN: https://onnxruntime.ai/docs/build/android.html
- ONNX Runtime QNN provider: https://github.com/onnxruntime/onnxruntime-qnn
- Current QNN provider docs: https://github.com/onnxruntime/onnxruntime-qnn/blob/main/docs/execution_providers/QNN-ExecutionProvider.md
- RF-DETR export formats/mobile support: https://rfdetr.roboflow.com/latest/learn/export/

## Other ONNX manga deployment evidence

- Lemon manga translator ONNX CTD/AOT model repository: https://huggingface.co/lemondouble/lemon-manga-translator

## API behavior reference

- OpenAI API reasoning configuration reference: https://platform.openai.com/docs/api-reference/responses

---

## Final note on third-party model exports

Some current ONNX files referenced above are **community conversions rather than official Koharu releases**.

Treat them as proof of feasibility and useful prototype artifacts, not as unquestioned production dependencies.

For a real release:

1. pin the exact upstream model revision;
2. reproduce the export script in this project's repository;
3. generate the ONNX files in a controlled build environment;
4. parity-test them against upstream;
5. publish SHA-256 hashes;
6. review every model/training-data license separately.

That keeps the Android project reproducible even if a third-party model repository later disappears.
