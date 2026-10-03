# MiniMax_H3_R2V_MV_Studio_FaceRefine_LatentUpscale

ComfyUI workflow for **MiniMax H3 Reference-to-Video music video generation**.

This workflow is designed for creating longer music videos by generating up to **7 seamless clips per Part**, with Motion Context continuity, FaceRefine, automatic FaceRefine bypass, Final Clip Auto Fit for the last Part, multiple MiniMax H3 acceleration configurations, audio synchronization, sparse-attention optimization, and RTX Video Super Resolution 2x upscaling.

![Workflow](workflow.png)

---

## Features

- MiniMax H3 Reference-to-Video
- Up to 7 clips per Part
- Selectable number of clips
- Seamless Motion Context between clips
- Audio synchronization
- FaceRefine for face detail enhancement
- Automatic FaceRefine bypass when no face/person is detected
- PDD Acc 8-step support
- Fused Turbo 4-step support
- TaoMate 3-step LoRA support
- Model Sparse Attention optimization
- Model Attention Backend support (`comfy kitchen attention` / `pytorch attention`)
- Part-based long video generation
- **Final Clip Auto Fit** for automatically extending only the last active clip when the final Part is shorter than the remaining song duration
- **Song Duration Probe** for reading the full audio duration used by Final Clip Auto Fit
- RTX Video Super Resolution 2x upscaling
- Designed for practical use on approximately 12GB–16GB VRAM GPUs

---

# Workflow Overview

```text
Load Audio (Upload)
   +
Song Duration Probe (Full Audio)
   ↓
Duration-1 ～ Duration-7
   ↓
Use Clip Total
   ↓
Final Clip Auto Fit
(only when enabled for the final Part)
   ↓
Clip 1 Pass1
   ↓
Motion Context
   ↓
Clip 2 Pass1
   ↓
Motion Context
   ↓
...
   ↓
Clip 7 Pass1
   ↓
VAE Decode
   ↓
FaceRefine / Auto Bypass
   ↓
Motion Context overlap trim
   ↓
Selected clips concatenated
   ↓
RTX Video Super Resolution 2x
   ↓
Save Part
```

Each generated group of clips is treated as one **Part**.

For ordinary Parts, Final Clip Auto Fit remains disabled and the configured clip durations are used as-is.

For the **final Part**, Final Clip Auto Fit can calculate the remaining song duration and, only when necessary, extend the **last active clip** before MiniMax H3 generation. It does not stretch an already generated video or duplicate output frames.

For longer music videos:

```text
Part 1
↓
Part 2
↓
Part 3
↓
...
↓
Final Part
↓
Merge Parts externally
↓
Final Music Video
```

---

# Workflow File

The workflow JSON included in this repository:

```text
last_complete_MiniMax_H3_R2V_FinalClipAutoFit.json
```

Download the JSON and load it into ComfyUI.

---

# Required Custom Nodes

## Install with ComfyUI Manager

### rgthree-comfy

Repository:

```text
https://github.com/rgthree/rgthree-comfy
```

Used nodes:

- Any Switch
- Fast Groups Bypasser

---

### ComfyUI-KJNodes

Repository:

```text
https://github.com/kijai/ComfyUI-KJNodes
```

Used nodes include:

- GetNode / SetNode
- MiniMax H3 Chunk FeedForward
- Patch SageAttention *(alternative / comparison path)*
- MiniMax H3 Memory Efficient SageAttention Patch *(alternative / comparison path)*
- MiniMax H3 Low VRAM Attention *(alternative / comparison path)*

---

### ComfyUI Core Optimization Nodes

The workflow also uses ComfyUI core model-optimization nodes:

- Model Sparse Attention
- Model Attention Backend

Recommended current configuration:

```text
Model Attention Backend
backend = comfy kitchen attention

↓

Model Sparse Attention
method = sol-attn
tau = 1.30
start_percent = 0.20
end_percent = 1.00
min_tokens = 12288
extra_tokens = 256
sink_conditioning = exact_kv_and_rows

↓

MiniMax H3 Chunk FeedForward
chunks = 2
seq_threshold = 4096

↓

MiniMaxH3SigmaShift
shift_video = 12
shift_audio = 3
```

The older SageAttention / Low-VRAM patch path is retained in the workflow for comparison and fallback use.

---

### NVIDIA RTX Nodes for ComfyUI

Repository:

```text
https://github.com/Comfy-Org/Nvidia_RTX_Nodes_ComfyUI
```

Used node:

- RTX Video Super Resolution

---

### ComfyUI-VideoHelperSuite

Repository:

```text
https://github.com/Kosinkadink/ComfyUI-VideoHelperSuite
```

Used node:

- VHS_LoadAudioUpload

---

# Manual Installation Custom Nodes

## ComfyUI-H3-Motion-Context-MultiRef

Repository:

```text
https://github.com/seitanism/ComfyUI-H3-Motion-Context-MultiRef
```

Installation:

```bash
cd ComfyUI/custom_nodes
git clone https://github.com/seitanism/ComfyUI-H3-Motion-Context-MultiRef.git
```

Used nodes:

- MiniMaxH3SongMaskedAVContext
- MiniMaxH3MotionContextTrim

This node set is used for seamless Motion Context continuity between clips.

---

## ComfyUI-MiniMax-H3-PDD-Acc

Repository:

```text
https://github.com/Jalen-Brunson/ComfyUI-MiniMax-H3-PDD-Acc
```

Installation:

```bash
cd ComfyUI/custom_nodes
git clone https://github.com/Jalen-Brunson/ComfyUI-MiniMax-H3-PDD-Acc.git
```

Used node:

- MiniMaxH3PDDAccApply

The node creates the following model folder when required:

```text
ComfyUI/models/pdd_acc/
```

---

## ComfyUI-H3-FaceRefine

Repository:

```text
https://github.com/Carasibana/ComfyUI-H3-FaceRefine
```

Installation:

```bash
cd ComfyUI/custom_nodes
git clone https://github.com/Carasibana/ComfyUI-H3-FaceRefine.git
```

Used nodes:

- H3FaceTrackCrop
- H3InjectVideoLatent
- H3PerFrameDenoise
- H3FaceStitch

FaceRefine processes detected faces after the initial MiniMax H3 generation.

---

## ComfyUI-H3-FaceAutoBypass

Repository:

```text
https://github.com/fukkun2705-commits/ComfyUI-H3-FaceAutoBypass
```

Installation:

1. Clone the repository:

```bash
cd ComfyUI/custom_nodes
git clone https://github.com/fukkun2705-commits/ComfyUI-H3-FaceAutoBypass.git
```

2. Install the required dependencies in the ComfyUI Python virtual environment:

```bash
python -m pip install -r custom_nodes/ComfyUI-H3-FaceAutoBypass/requirements.txt
```

Used nodes:

- H3FacePresenceGate
- H3LazyImageSwitch

This custom node automatically determines whether FaceRefine should be used.

If a face/person is detected:

```text
Face detected
→ FaceRefine enabled
```

If no suitable face/person is detected:

```text
No face/person
→ FaceRefine bypassed
```

This prevents unnecessary FaceRefine processing on clips that do not require it.

---

## ComfyUI-H3-FinalClipAutoFit

This workflow also uses a custom node for fitting the final generated Part to the actual song duration.

Used node:

- `H3FinalClipAutoFit`

Purpose:

```text
Actual full song duration
-
Final Part start time
↓
Remaining song duration
↓
Extend only the last active clip when required
```

The node changes the **generation duration of the final active clip before generation**. It does not stretch the finished video, repeat frames, or perform frame interpolation.

Manual installation:

```text
ComfyUI/custom_nodes/
└── ComfyUI-H3-FinalClipAutoFit/
```

If the custom node is published as a Git repository later, it can also be installed with `git clone` into `ComfyUI/custom_nodes/`.

---

# Required Models

## diffusion_models

### MiniMax H3 Ref2VA

```text
minimax_h3_ref2va_pruned_int8_convrot.safetensors
```

Download:

```text
https://huggingface.co/Comfy-Org/MiniMax-H3/blob/main/diffusion_models/minimax_h3_ref2va_pruned_int8_convrot.safetensors
```

---

### MiniMax H3 Fused Turbo

```text
minimax_h3_fused_refdelta_r1024_turbo8_mystic07_int8_convrot.safetensors
```

Download:

```text
https://huggingface.co/MATLOWAI/minimax-h3-fused-turbo-int8-convrot/blob/main/diffusion_models/minimax_h3_fused_refdelta_r1024_turbo8_mystic07_int8_convrot.safetensors
```

---

# Text Encoder

```text
qwen3vl_32b_minimax_h3_nvfp4_awq.safetensors
```

Download:

```text
https://huggingface.co/Comfy-Org/MiniMax-H3/blob/main/text_encoders/qwen3vl_32b_minimax_h3_nvfp4_awq.safetensors
```

---

# VAE

## Video VAE

```text
minimax_h3_video_vae_int8_convrot.safetensors
```

Download:

```text
https://huggingface.co/Comfy-Org/MiniMax-H3/blob/main/vae/minimax_h3_video_vae_int8_convrot.safetensors
```

---

## Audio VAE

```text
minimax_h3_audio_vae_fp32.safetensors
```

Download:

```text
https://huggingface.co/Comfy-Org/MiniMax-H3/blob/main/vae/minimax_h3_audio_vae_fp32.safetensors
```

---

# PDD Acc Model

```text
MiniMax-H3-Ref2VA-Acc-8Step.safetensors
```

Download:

```text
https://huggingface.co/alibaba-pai/MiniMax-H3-Acc-LoRAs/blob/main/MiniMax-H3-Ref2VA-Acc-8Step.safetensors
```

---

# TaoMate LoRA

```text
taomate_h3_3step_comfy.safetensors
```

Download:

```text
https://huggingface.co/Robert1212star/TaoMate-H3-3Step-ComfyUI/blob/main/taomate_h3_3step_comfy.safetensors
```

Recommended model folder:

```text
ComfyUI/models/loras/
```

---

# FaceRefine Detector Models

## Face Detector

```text
face_yolov8m.pt
```

Download:

```text
https://huggingface.co/Bingsu/adetailer/blob/main/face_yolov8m.pt
```

---

## Person Detector

```text
person_yolov8m-seg.pt
```

Download:

```text
https://huggingface.co/Bingsu/adetailer/blob/main/person_yolov8m-seg.pt
```

---

# Model Storage Location

Standard ComfyUI folder structure:

```text
ComfyUI/
│
├── custom_nodes/
│   ├── rgthree-comfy/
│   ├── ComfyUI-KJNodes/
│   ├── Nvidia_RTX_Nodes_ComfyUI/
│   ├── ComfyUI-VideoHelperSuite/
│   ├── ComfyUI-H3-Motion-Context-MultiRef/
│   ├── ComfyUI-MiniMax-H3-PDD-Acc/
│   ├── ComfyUI-H3-FaceRefine/
│   └── ComfyUI-H3-FaceAutoBypass/
│
└── models/
    │
    ├── diffusion_models/
    │   ├── minimax_h3_fused_refdelta_r1024_turbo8_mystic07_int8_convrot.safetensors
    │   └── minimax_h3_ref2va_pruned_int8_convrot.safetensors
    │
    ├── loras/
    │   └── taomate_h3_3step_comfy.safetensors
    │
    ├── text_encoders/
    │   └── qwen3vl_32b_minimax_h3_nvfp4_awq.safetensors
    │
    ├── vae/
    │   ├── minimax_h3_video_vae_int8_convrot.safetensors
    │   └── minimax_h3_audio_vae_fp32.safetensors
    │
    ├── pdd_acc/
    │   └── MiniMax-H3-Ref2VA-Acc-8Step.safetensors
    │
    └── ultralytics/
        │
        ├── bbox/
        │   └── face_yolov8m.pt
        │
        └── segm/
            └── person_yolov8m-seg.pt
```

---

# Stability Matrix

For Stability Matrix, the shared model folders are typically stored under `Data/models/`, while package-specific folders such as `pdd_acc` and `ultralytics` may remain inside the selected ComfyUI package.

Example:

```text
Data/
│
├── models/
│   ├── diffusion_models/
│   │   ├── minimax_h3_fused_refdelta_r1024_turbo8_mystic07_int8_convrot.safetensors
│   │   └── minimax_h3_ref2va_pruned_int8_convrot.safetensors
│   │
│   ├── loras/
│   │   └── taomate_h3_3step_comfy.safetensors
│   │
│   ├── text_encoders/
│   │   └── qwen3vl_32b_minimax_h3_nvfp4_awq.safetensors
│   │
│   └── vae/
│       ├── minimax_h3_video_vae_int8_convrot.safetensors
│       └── minimax_h3_audio_vae_fp32.safetensors
│
└── Packages/
    └── <ComfyUI Package Name>/
        │
        ├── custom_nodes/
        │   ├── rgthree-comfy/
        │   ├── ComfyUI-KJNodes/
        │   ├── Nvidia_RTX_Nodes_ComfyUI/
        │   ├── ComfyUI-VideoHelperSuite/
        │   ├── ComfyUI-H3-Motion-Context-MultiRef/
        │   ├── ComfyUI-MiniMax-H3-PDD-Acc/
        │   ├── ComfyUI-H3-FaceRefine/
        │   └── ComfyUI-H3-FaceAutoBypass/
        │
        └── models/
            ├── pdd_acc/
            │   └── MiniMax-H3-Ref2VA-Acc-8Step.safetensors
            │
            └── ultralytics/
                ├── bbox/
                │   └── face_yolov8m.pt
                └── segm/
                    └── person_yolov8m-seg.pt
```

Folder behavior may differ depending on your Stability Matrix shared-model settings.

---

# How to Use the Workflow

## 1. Set Resolution

Use:

```text
Resolution Selector (Size)
```

Select:

- `aspect_ratio`
- `megapixels`
- `multiple`

`multiple = 32` is the normal setting used by this workflow.

The workflow includes a **Size Settings Reference** table. Example 16:9 presets:

| Mode | MP | 1st Pass | Final 2x |
|---|---:|---:|---:|
| FAST | 0.2 | 608 × 352 | 1216 × 704 |
| BALANCED | 0.3 | 736 × 416 | 1472 × 832 |
| QUALITY | 0.4 | 864 × 480 | 1728 × 960 |

RTX VSR performs the final 2x upscale **after the selected clips in the Part have been merged**.

### Reference-image guidance

For **character identity**, a multi-angle character style sheet is useful because it provides multiple views of the same subject.

For **backgrounds**, a single large one-shot image is recommended when possible. A multi-panel background style sheet divides the available pixels among several views, so each individual background panel contains less detail. Because the background may fill the entire video frame, a small panel can result in softer, blurrier, or less stable background detail.

Recommended practical rule:

```text
Character / person reference → style sheet is useful
Background reference          → large single-shot image recommended
Product reference             → style sheet or single-shot image depending on shot size
```

---

## 2. Load Audio

Load the **same song/audio file into both audio nodes**:

```text
Load Audio (Upload)
Song Duration Probe (Full Audio)
```

For `Song Duration Probe (Full Audio)` use:

```text
start_time = 0
duration = 0
```

This node is used only to read the exact full audio duration for Final Clip Auto Fit.

For **Part 1**:

```text
Load Audio (Upload)
start_time = 0
```

The normal `Load Audio (Upload)` duration is controlled automatically from the current Part duration.

---

## 3. Set Clip Durations

The workflow provides:

```text
Duration-1
Duration-2
Duration-3
Duration-4
Duration-5
Duration-6
Duration-7
```

Set the desired generation duration for each clip.

Example:

```text
Clip 1 = 3 sec
Clip 2 = 4 sec
Clip 3 = 4 sec
Clip 4 = 4 sec
Clip 5 = 4 sec
```

---

## 4. Select Number of Clips

Use:

```text
Use Clip Total
```

Available range:

```text
1 – 7 clips
```

Clip 1 is always active. For Clips 2–7, enable only the corresponding Clip Processors required for the current Part.

`Use Clip Total` and the number of enabled Clip Processors should match.

---

## 5. Configure Final Clip Auto Fit

Use:

```text
H3 Final Clip Auto Fit
```

### Ordinary Parts

For all Parts that do **not** contain the end of the song:

```text
enable_final_fit = false
```

The configured `Duration-1 ～ Duration-7` values are used without final-song fitting.

### Final Part

For the Part that contains the **end of the song**:

```text
enable_final_fit = true
```

Set:

```text
part_start_time
```

to exactly the same value as that Part's:

```text
Load Audio (Upload) start_time
```

Example:

```text
Load Audio start_time = 22.875

H3 Final Clip Auto Fit
part_start_time = 22.875
enable_final_fit = true
```

The node calculates:

```text
actual song duration
-
part_start_time
=
remaining song duration
```

If the selected clip durations are too short to reach the end of the song, **only the last active clip is extended**.

The generated clip itself becomes longer; the finished video is not stretched afterward.

---

## 6. Check Total Part Duration

Check:

```text
Total Part Duration Out
```

With Final Clip Auto Fit disabled, this is the total of the active clip duration settings.

Example:

```text
3 + 4 + 4 + 4 + 4
=
19.0 sec
```

With Final Clip Auto Fit enabled for the final Part, the value reflects the adjusted final clip duration when an extension is required.

Example:

```text
Original final clip = 6.00 sec
↓
Auto Fit adjusted final clip = 6.75 sec

Total Part Duration Out is also updated accordingly.
```

---

## 7. Select Generation Configuration

The workflow supports three main MiniMax H3 acceleration configurations.

| Configuration | UNET | TaoMate | PDD | Scheduler |
|---|---|---|---|---|
| ① PDD Acc 8-step | Ref2VA | OFF | ON | PDD side |
| ② Fused Turbo | Fused | OFF | OFF | 4-step |
| ③ TaoMate | Ref2VA | ON | OFF | 3-step |

### ① PDD Acc 8-step

```text
UNET:
minimax_h3_ref2va_pruned_int8_convrot.safetensors

TaoMate LoRA:
OFF

PDD Acc:
ON

PDD model:
MiniMax-H3-Ref2VA-Acc-8Step.safetensors
```

### ② Fused Turbo 4-step

```text
UNET:
minimax_h3_fused_refdelta_r1024_turbo8_mystic07_int8_convrot.safetensors

TaoMate LoRA:
OFF

PDD Acc:
OFF

Scheduler:
4-step
```

### ③ TaoMate 3-step

```text
UNET:
minimax_h3_ref2va_pruned_int8_convrot.safetensors

TaoMate LoRA:
ON
taomate_h3_3step_comfy.safetensors

PDD Acc:
OFF

Scheduler:
3-step
```

When PDD Acc is not used, keep the PDD path OFF. When TaoMate is not used, bypass the TaoMate LoRA loader.

### Recommended Attention / VRAM Optimization

Current recommended test configuration:

```text
Model Attention Backend
backend = comfy kitchen attention

Model Sparse Attention
method = sol-attn
tau = 1.30
start_percent = 0.20
end_percent = 1.00
min_tokens = 12288
extra_tokens = 256
sink_conditioning = exact_kv_and_rows

MiniMax H3 Chunk FeedForward
chunks = 2
seq_threshold = 4096

MiniMaxH3SigmaShift
shift_video = 12
shift_audio = 3
```

For this configuration, the older SageAttention / Low-VRAM patch nodes should remain bypassed.

---

## 8. FaceRefine Management Panel

The workflow provides one common FaceRefine management panel shared by the Clip Processors.

Recommended initial values:

```text
FR Steps = 5
FR Denoise = 0.35
FR Canvas = 768
FR Confidence = 0.35
FR Crop Factor = 2.50
FR Identity Threshold = 0.28
```

`FR Canvas = 768` is the recommended control value. The FaceRefine path can automatically use a smaller effective canvas such as 512 when the detected crop does not require the full 768 processing size.

Therefore, in normal use there is usually no need to manually change the control to 512.

---

## 9. Generate the Part

Start generation.

The workflow automatically performs:

```text
MiniMax H3 generation
↓
Motion Context continuity
↓
VAE Decode
↓
FaceRefine / Auto Bypass
↓
Motion Context overlap trim
↓
Clip concatenation
↓
RTX VSR 2x
↓
Video output
```

---

## 10. Check Actual Video Duration

After generation, check:

```text
Actual Video Duration
```

This is the **actual generated Part duration**, not merely the mathematical sum of the configured clip durations.

Example:

```text
Total Part Duration Out
13.000 sec

Actual Video Duration
12.958 sec
```

MiniMax H3 is frame-based and Motion Context overlap is trimmed, so the actual video duration can differ slightly from the configured total.

---

## 11. Set the Start Time for the Next Part

For the next Part, use the cumulative **Actual Video Duration** from all preceding Parts.

Example:

```text
Part 1 Actual Video Duration = 12.958 sec

Part 2 Load Audio start_time = 12.958
```

If:

```text
Part 2 Actual Video Duration = 12.250 sec
```

then:

```text
Part 3 start_time
= 12.958 + 12.250
= 25.208
```

Continue this process for subsequent Parts.

---

## 12. Final Part Auto Fit Report

When Final Clip Auto Fit is enabled, check:

```text
Final Clip Auto Fit Report
```

Main report values include:

```text
actual_song_duration
remaining_song_seconds
target_frames
predicted_before_frames
predicted_after_frames
added_frames
adjusted_final_duration
end_margin_frames
```

If:

```text
added_frames > 0
```

then the last active clip was extended so that the final Part can reach the song ending.

---

## 13. Merge the Finished Parts

After all Parts have been generated:

```text
Part 1
+
Part 2
+
Part 3
+
...
=
Final Music Video
```

Merge the Parts using your preferred editor or video concatenation tool.

---

# Part Generation Concept

```text
Music
│
├── Part 1
│   ├── Clip 1
│   ├── Clip 2
│   ├── Clip 3
│   └── ...
│
├── Part 2
│   ├── Clip 1
│   ├── Clip 2
│   └── ...
│
├── Part 3
│
└── ...
```

Each Part is generated and upscaled independently.

After all Parts have been created:

```text
Part 1
+
Part 2
+
Part 3
+
...
=
Final Music Video
```

Merge the Parts using your preferred video editor or video concatenation tool.

---

# Seamless Motion Context

Motion Context is used between clips to preserve motion continuity.

Concept:

```text
Clip 1
↓ Motion Context
Clip 2
↓ Motion Context
Clip 3
↓
...
```

The beginning of the next clip contains Motion Context frames inherited from the previous clip.

Those leading overlap frames are trimmed before the final clips are concatenated.

This allows the visible clips to connect while maintaining motion continuity.

---

# Final Clip Auto Fit

Final Clip Auto Fit is designed specifically for the **last Part of a song**.

It uses the exact song duration reported by `Song Duration Probe (Full Audio)` and the final Part's `part_start_time` to determine how much song time remains.

Processing concept:

```text
Full song duration
-
Final Part start time
↓
Remaining song duration
↓
Compare with selected active clip durations
↓
If necessary, extend only the last active clip
↓
Generate that clip at the adjusted duration
```

Important behavior:

- Ordinary Parts should use `enable_final_fit = false`.
- The final Part should use `enable_final_fit = true`.
- `part_start_time` must match the final Part's `Load Audio (Upload) start_time`.
- The node only extends when additional duration is required.
- It does not stretch completed video frames.
- It does not duplicate frames after generation.
- It works with the selected `Use Clip Total`, so the adjusted clip is the **last active clip**, not necessarily Clip 7.

---

# FaceRefine

FaceRefine is applied after the initial MiniMax H3 video generation.

Basic processing:

```text
MiniMax H3 video
↓
VAE Decode
↓
Face tracking / crop
↓
FaceRefine
↓
Face stitch
↓
Final clip
```

FaceRefine processes only the final visible frames after the Motion Context leading overlap is handled by the workflow.

This prevents unnecessary refinement of frames that will not appear in the final output.

## FaceRefine Management Panel

Recommended initial values:

```text
FR Steps = 5
FR Denoise = 0.35
FR Canvas = 768
FR Confidence = 0.35
FR Crop Factor = 2.50
FR Identity Threshold = 0.28
```

### FR Canvas

```text
FR Canvas = 768
```

`768` is the recommended default.  
When a smaller processing size is sufficient, the FaceRefine processing path can automatically use the smaller effective canvas as needed, so normally there is no need to manually change the control to `512`.

---

# FaceRefine Auto Bypass

The workflow also includes automatic FaceRefine detection.

```text
Video frames
↓
H3FacePresenceGate
↓
Face/person detected?
```

If detected:

```text
YES
↓
FaceRefine
```

If not detected:

```text
NO
↓
Original frames
```

This allows clips without a suitable visible face/person to skip unnecessary FaceRefine processing automatically.

---

# PDD Acc

The workflow supports:

```text
MiniMax-H3-Ref2VA-Acc-8Step.safetensors
```

PDD Acc can be enabled or disabled from the workflow controls.

The workflow automatically routes the sampler path according to the selected PDD configuration.

The MiniMax H3 diffusion model itself can be changed manually depending on your preferred generation setup.

---

# Performance Notes

Example local benchmark on the tested environment:

```text
Initial resolution:
0.3 MP

Clip durations:
3, 4, 4, 4, 4 sec

Total:
5 clips / 19 sec

FaceRefine:
enabled

RTX VSR:
2x after clip merge
```

Measured total workflow time:

| Configuration | Attention optimization | Time |
|---|---|---:|
| ② Fused Turbo 4-step | Model Sparse Attention + comfy kitchen attention | **812.13 sec** |
| ③ TaoMate 3-step | Model Sparse Attention + comfy kitchen attention | **821.10 sec** |
| ③ TaoMate 3-step | SageAttention path + comfy kitchen attention | **868.05 sec** |
| ③ TaoMate 3-step | Model Sparse Attention + pytorch attention | **930.31 sec** |
| ① PDD Acc 8-step | Model Sparse Attention + comfy kitchen attention | **1174.80 sec** |

These values are environment-specific and should be treated as reference measurements rather than universal performance figures.

In this test, `comfy kitchen attention` produced a particularly large speed improvement during FaceRefine, while `Model Sparse Attention` was faster overall than the SageAttention comparison path without an obvious visual-quality loss in the tested clips.

---

# RTX Video Super Resolution

RTX Video Super Resolution is applied **after all selected clips in the current Part have been concatenated**.

Processing:

```text
Clip 1
Clip 2
Clip 3
...
↓
Concatenate
↓
Complete Part
↓
RTX VSR 2x
↓
Save Video
```

RTX VSR is therefore not applied independently to each clip.

This helps keep the final Part processing simple and avoids introducing additional processing between seamless clip boundaries.

---

# Recommended Workflow Sequence

```text
1. Select resolution
2. Load the same audio into Load Audio (Upload) and Song Duration Probe
3. Set Duration-1 ～ Duration-7
4. Set Use Clip Total
5. For ordinary Parts: Final Clip Auto Fit = OFF
6. For the final Part: Final Clip Auto Fit = ON and set part_start_time
7. Enable the required Clip Processors
8. Select generation configuration (PDD / Fused / TaoMate)
9. Confirm Attention / VRAM optimization settings
10. Confirm FaceRefine Management Panel settings
11. Set Load Audio start_time
12. Generate
13. Check Actual Video Duration
14. For the final Part, check Final Clip Auto Fit Report
15. Use cumulative Actual Video Duration for the next Part
16. Repeat as required
17. Merge finished Parts
```

---

# Tested Environment

The workflow was developed and tested primarily with:

```text
GPU:
NVIDIA GeForce RTX 3060 12GB

System RAM:
64GB

OS:
Windows

Frontend:
ComfyUI / Stability Matrix
```

The current recommended test path uses Model Sparse Attention, `comfy kitchen attention`, MiniMax H3 Chunk FeedForward, MiniMaxH3SigmaShift, and dynamic VRAM management. The older SageAttention / Low-VRAM patch path is retained as an alternative comparison path.

Performance and VRAM requirements may vary depending on:

- Resolution
- Clip duration
- Diffusion model
- PDD / Fused / TaoMate configuration
- Attention backend and sparse-attention settings
- FaceRefine
- ComfyUI version
- PyTorch / CUDA version
- Installed custom nodes

---

# Important Notes

This repository contains the workflow and documentation only.

It does **not** include:

- MiniMax H3 model weights
- Text encoder weights
- VAE weights
- PDD Acc model weights
- TaoMate LoRA weights
- Face detector models
- Third-party custom nodes

Please download those files from their original repositories or model pages.

For Final Clip Auto Fit, load the **same audio file** into both `Load Audio (Upload)` and `Song Duration Probe (Full Audio)`.

---

# Credits

This workflow uses multiple community projects, including:

- MiniMax H3
- ComfyUI
- rgthree-comfy
- ComfyUI-KJNodes
- ComfyUI-VideoHelperSuite
- NVIDIA RTX Nodes for ComfyUI
- ComfyUI-H3-Motion-Context-MultiRef
- ComfyUI-MiniMax-H3-PDD-Acc
- ComfyUI-H3-FaceRefine
- ComfyUI-H3-FaceAutoBypass
- ComfyUI-H3-FinalClipAutoFit

Thank you to all developers and contributors of these projects.

---

# License

This repository is licensed under the **MIT License**.

See:

```text
LICENSE
```

Third-party custom nodes, models, checkpoints, LoRAs, detector models, and other external assets are subject to their respective licenses.

The MIT License in this repository applies only to the files distributed directly by this repository where applicable.

---

# Disclaimer

Model and custom-node behavior may change as ComfyUI and related projects are updated.

If a workflow node is missing after loading the JSON, confirm that all required custom nodes and model files are installed correctly.
