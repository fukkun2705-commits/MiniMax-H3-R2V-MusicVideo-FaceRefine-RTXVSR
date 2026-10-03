# MiniMax_H3_R2V_MV_Studio_FaceRefine_LatentUpscale

ComfyUI workflow for **MiniMax H3 Reference-to-Video music video generation**.

This workflow is designed for creating longer music videos by generating up to **7 clips per Part**, with Motion Context continuity, crossfade smoothing between every clip boundary, Latent Upscale + Pass2 refinement, FaceRefine, automatic FaceRefine bypass, Final Clip Auto Fit for the last Part, audio synchronization, SageAttention / Low-VRAM optimization, and optional RTX Video Super Resolution.

![Workflow](workflow.png)

---

## Features

- MiniMax H3 Reference-to-Video
- Up to 7 clips per Part
- Selectable number of clips
- Motion Context continuity between clips
- **Crossfade smoothing across all clip boundaries (Clip1→2 through Clip6→7)**
- Audio synchronization
- **Latent Upscale + Pass2 refinement**
- FaceRefine for face detail enhancement
- Automatic FaceRefine bypass when no face/person is detected
- PDD Acc 8-step LoRA support
- Fused Turbo 4-step support
- TaoMate 3-step LoRA support
- SageAttention / MiniMax H3 Memory Efficient SageAttention support
- MiniMax H3 Low VRAM Attention support
- Model Attention Backend support
- Part-based long video generation
- **Final Clip Auto Fit** for automatically extending only the last active clip when the final Part is shorter than the remaining song duration
- **Song Duration Probe** for reading the full audio duration used by Final Clip Auto Fit
- Optional RTX Video Super Resolution after the Part is assembled
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
(final Part only)
   ↓
Clip 1 Pass1
   ↓
Motion Context
   ↓
Clip 2 Pass1
   ↓
...
   ↓
Clip 7 Pass1
   ↓
Video / Audio latent separation
   ↓
Video Latent Upscale
   ↓
Audio latent passthrough
   ↓
AV latent recombination
   ↓
Pass2 Refine
   ↓
VAE Decode
   ↓
FaceRefine / Auto Bypass
   ↓
Motion Context overlap trim
   ↓
Crossfade at every active clip boundary
   ↓
Complete Part
   ↓
Optional RTX Video Super Resolution
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
MiniMax_H3_R2V_MV_Studio_FaceRefine_LatentUpscale.json
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
- CrossFadeImages
- MiniMax H3 Chunk FeedForward
- Patch SageAttention
- MiniMax H3 Memory Efficient SageAttention Patch
- MiniMax H3 Low VRAM Attention

`CrossFadeImages` is used to smooth the visible transition between every active clip boundary after Motion Context overlap handling.

### Current Attention / VRAM path

The current workflow uses the SageAttention-based path:

```text
Patch SageAttention
sage_attention = auto

↓

MiniMax H3 Memory Efficient SageAttention Patch

↓

MiniMax H3 Low VRAM Attention
head_chunks = 1

↓

Model Attention Backend
attention = pytorch attention

↓

MiniMaxH3SigmaShift
shift_video = 12
shift_audio = 3
```

**Model Sparse Attention is not used in the current distributed workflow.**

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

## Comfyui_Minimax_h3_latent_Upscaler

Repository:

```text
https://github.com/LBH-123-AI/Comfyui_Minimax_h3_latent_Upscaler
```

Installation:

```bash
cd ComfyUI/custom_nodes
git clone https://github.com/LBH-123-AI/Comfyui_Minimax_h3_latent_Upscaler.git
```

Used node:

- `MinimaxH3LatentUpscaler3D`

This workflow separates the MiniMax H3 AV latent, sends only the **video latent** to the learned 3D latent upscaler, preserves the audio latent, then recombines video and audio before Pass2.

Do not substitute the similarly named `Tr1dae/ComfyUI-MiniMaxH3_LatentUpscaler`; it is not the latent-upscaler implementation used by this workflow.

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

Repository:

```text
https://github.com/fukkun2705-commits/ComfyUI-H3-FinalClipAutoFit
```

Installation:

```bash
cd ComfyUI/custom_nodes
git clone https://github.com/fukkun2705-commits/ComfyUI-H3-FinalClipAutoFit.git
```

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

# PDD Acc LoRA

Recommended pair used by the workflow:

```text
Diffusion:
minimax_h3_ref2va_pruned_int8_convrot.safetensors

LoRA:
MiniMax-H3-Ref2VA-Acc-8Step_pruned_comfy.safetensors
```

Download:

```text
https://huggingface.co/Kijai/MiniMax-H3-experimental/resolve/main/loras/MiniMax-H3-Ref2VA-Acc-8Step_pruned_comfy.safetensors
```

The PDD Acc path now uses the normal ComfyUI LoRA loader. A dedicated PDD custom node is not required.

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

# Latent Upscaler Model

```text
minimax_h3_latent_upscaler_3d_conv_v1_fp16.safetensors
```

Download:

```text
https://huggingface.co/LBH-123-AI/Minimax_h3_latent_Upscaler/resolve/main/minimax_h3_latent_upscaler_3d_conv_v1/minimax_h3_latent_upscaler_3d_conv_v1_fp16.safetensors
```

Model folder:

```text
ComfyUI/models/latent_upscale_models/
```

For Stability Matrix, this model may need to be placed in the selected ComfyUI package's local `models/latent_upscale_models/` folder if that folder is not mapped to shared models.

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
│   ├── ComfyUI-H3-FaceRefine/
│   ├── ComfyUI-H3-FaceAutoBypass/
│   ├── Comfyui_Minimax_h3_latent_Upscaler/
│   └── ComfyUI-H3-FinalClipAutoFit/
│
└── models/
    │
    ├── diffusion_models/
    │   ├── minimax_h3_fused_refdelta_r1024_turbo8_mystic07_int8_convrot.safetensors
    │   └── minimax_h3_ref2va_pruned_int8_convrot.safetensors
    │
    ├── loras/
    │   ├── MiniMax-H3-Ref2VA-Acc-8Step_pruned_comfy.safetensors
    │   └── taomate_h3_3step_comfy.safetensors
    │
    ├── text_encoders/
    │   └── qwen3vl_32b_minimax_h3_nvfp4_awq.safetensors
    │
    ├── vae/
    │   ├── minimax_h3_video_vae_int8_convrot.safetensors
    │   └── minimax_h3_audio_vae_fp32.safetensors
    │
    ├── latent_upscale_models/
    │   └── minimax_h3_latent_upscaler_3d_conv_v1_fp16.safetensors
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
│   │   ├── MiniMax-H3-Ref2VA-Acc-8Step_pruned_comfy.safetensors
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
                │   ├── ComfyUI-H3-FaceRefine/
        │   ├── ComfyUI-H3-FaceAutoBypass/
│   ├── Comfyui_Minimax_h3_latent_Upscaler/
│   └── ComfyUI-H3-FinalClipAutoFit/
        │
        └── models/
            ├── pdd_acc/
            │   └── MiniMax-H3-Ref2VA-Acc-8Step.safetensors
            │
            ├── latent_upscale_models/
    │   └── minimax_h3_latent_upscaler_3d_conv_v1_fp16.safetensors
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

| Mode | MP | 1st Pass | Latent Scale | After Latent Upscale |
|---|---:|---:|---:|---:|
| FAST | 0.2 | 608 × 352 | 1.6x | 960 × 576 |
| BALANCED | 0.3 | 736 × 416 | 1.6x | 1184 × 672 |
| QUALITY | 0.4 | 864 × 480 | 1.6x | 1376 × 768 |

### Recommended 768p output settings (16:9)

| Mode | MP | 1st Pass | Latent Scale | After Latent | RTX VSR | Final Target |
|---|---:|---:|---:|---:|---:|---:|
| FAST | 0.2 | 608 × 352 | 1.6x | 960 × 576 | about 1.33x | about 1280 × 768 |
| BALANCED | 0.3 | 736 × 416 | 1.6x | 1184 × 672 | about 1.14x | about 1353 × 768 |
| QUALITY | 0.4 | 864 × 480 | 1.6x | 1376 × 768 | **BYPASS** | 1376 × 768 |

For **QUALITY / 0.4 MP**, Latent Upscale 1.6x already reaches a 768-pixel output height, so RTX VSR is normally bypassed when the target is 768p.

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

> ⚠️ **FINAL PART ONLY**  
> Ordinary Part: `enable_final_fit = false`  
> Final Part only: `enable_final_fit = true`

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

PDD LoRA:
MiniMax-H3-Ref2VA-Acc-8Step_pruned_comfy.safetensors

LoRA strength:
1.0
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

When PDD Acc is not used, bypass the PDD Acc LoRA loader. When TaoMate is not used, bypass the TaoMate LoRA loader.

### Recommended Attention / VRAM Optimization

Current recommended configuration:

```text
Patch SageAttention
sage_attention = auto

MiniMax H3 Memory Efficient SageAttention Patch

MiniMax H3 Low VRAM Attention
head_chunks = 1

Model Attention Backend
attention = pytorch attention

MiniMaxH3SigmaShift
shift_video = 12
shift_audio = 3
```

`Model Sparse Attention` is intentionally not used in the current workflow because temporal background instability was observed in testing, especially with the PDD Acc path.

---

## 8. Latent Upscale / Pass2

The workflow provides one common Latent Upscale / Pass2 management panel shared by the Clip Processors.

Recommended initial values:

```text
Use Latent Upscale / Pass2 = yes
Upscale Scale = 1.6
Pass2 Steps = 3
Pass2 Denoise = 0.3
Pass2 Sampler = euler
```

Processing:

```text
MiniMax H3 Pass1 AV latent
↓
Separate video / audio latent
↓
Video latent → 3D Latent Upscale
Audio latent → passthrough
↓
Recombine AV latent
↓
Pass2 Refine
↓
VAE Decode
```

This avoids a VAE Decode → image upscale → VAE Encode round trip before Pass2.

---

## 9. FaceRefine Management Panel

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

## 10. Generate the Part

Start generation.

The workflow automatically performs:

```text
MiniMax H3 Pass1
↓
Motion Context continuity
↓
Latent Upscale + Pass2
↓
VAE Decode
↓
FaceRefine / Auto Bypass
↓
Motion Context overlap trim
↓
Crossfade between active clip boundaries
↓
Complete Part
↓
Optional RTX VSR
↓
Video output
```

---

## 11. Check Actual Video Duration

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

## 12. Set the Start Time for the Next Part

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

## 13. Final Part Auto Fit Report

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

## 14. Merge the Finished Parts

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

Those leading overlap frames are handled by `MiniMaxH3MotionContextTrim`.

The current workflow also uses the retained `crossfade_images` / `crossfade_frames` outputs to apply a short crossfade at **every active clip boundary**:

```text
Clip1 → Clip2
Clip2 → Clip3
Clip3 → Clip4
Clip4 → Clip5
Clip5 → Clip6
Clip6 → Clip7
```

This reduces visible discontinuities that can otherwise appear after Latent Upscale / Pass2 and FaceRefine modify each clip independently.

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

FaceRefine is applied after Pass2 / VAE Decode.

Basic processing:

```text
MiniMax H3 Pass1
↓
Latent Upscale + Pass2
↓
VAE Decode
↓
Face tracking / crop
↓
FaceRefine
↓
Face stitch
↓
Motion Context trim / crossfade
↓
Final visible clip sequence
```

Motion Context overlap handling and final crossfade smoothing are performed after the clip's visual refinement path.

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

The current workflow uses the pruned PDD Acc LoRA pair:

```text
Diffusion:
minimax_h3_ref2va_pruned_int8_convrot.safetensors

LoRA:
MiniMax-H3-Ref2VA-Acc-8Step_pruned_comfy.safetensors
```

The LoRA is loaded through the standard ComfyUI `LoraLoaderModelOnly` path.

A dedicated `ComfyUI-MiniMax-H3-PDD-Acc` custom node is not required by the current workflow.

For the PDD Acc path, the current workflow uses SageAttention rather than Model Sparse Attention.

---

# Performance Notes

Generation time depends strongly on:

- First-pass resolution
- Clip duration
- Number of active clips
- MiniMax H3 acceleration configuration
- Latent Upscale / Pass2
- FaceRefine
- RTX VSR usage
- VRAM offloading behavior

For quick tests, use `FAST / 0.2 MP`.

For normal production, `BALANCED / 0.3 MP` is a practical starting point.

For 768p quality-focused output, `QUALITY / 0.4 MP + Latent Scale 1.6x` reaches approximately `1376 × 768`, allowing RTX VSR to be bypassed.
---

# RTX Video Super Resolution

RTX Video Super Resolution is optional and is applied **after the active clips have been assembled into the current Part**.

Current workflow control:

```text
RTX Upscale
```

The RTX VSR node uses a configurable multiplier. The current workflow node is set to:

```text
scale = 1.35
quality = ULTRA
```

It is **not fixed to 2x**.

Recommended 768p behavior:

```text
FAST / 0.2 MP
→ Latent 1.6x
→ RTX VSR as needed

BALANCED / 0.3 MP
→ Latent 1.6x
→ small RTX VSR upscale as needed

QUALITY / 0.4 MP
→ Latent 1.6x
→ 1376 × 768
→ RTX VSR BYPASS for 768p target
```

RTX VSR is not applied independently to each clip, so it does not introduce extra processing between clip boundaries.
---

# Recommended Workflow Sequence

```text
1. Select resolution
2. Load the same audio into Load Audio (Upload) and Song Duration Probe
3. Set Duration-1 ～ Duration-7
4. Set Use Clip Total
5. For ordinary Parts: Final Clip Auto Fit = OFF
6. For the final Part only: Final Clip Auto Fit = ON and set part_start_time
7. Enable the required Clip Processors
8. Select generation configuration
9. Confirm SageAttention / Low-VRAM settings
10. Confirm Latent Upscale / Pass2 settings
11. Confirm FaceRefine settings
12. Set Load Audio start_time
13. Set RTX Upscale or BYPASS according to target resolution
14. Generate
15. Check Actual Video Duration
16. For the final Part, check Final Clip Auto Fit Report
17. Use cumulative Actual Video Duration for the next Part
18. Repeat as required
19. Merge finished Parts
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

The current workflow uses SageAttention, MiniMax H3 Memory Efficient SageAttention, MiniMax H3 Low VRAM Attention, MiniMaxH3SigmaShift, and dynamic VRAM management. Model Sparse Attention is not used in the current distributed workflow.

Performance and VRAM requirements may vary depending on:

- Resolution
- Clip duration
- Diffusion model
- PDD / Fused / TaoMate configuration
- SageAttention / attention-backend settings
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
- PDD Acc LoRA weights
- TaoMate LoRA weights
- Latent Upscaler model weights
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
- ComfyUI-H3-FaceRefine
- ComfyUI-H3-FaceAutoBypass
- Comfyui_Minimax_h3_latent_Upscaler
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
