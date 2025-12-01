# 🎥 VideoContext Engine v3.19 (Qwen3-VL + RAM Modes)

> 🚧 **Status: Public Beta**  
> The core engine is stable enough for real use on macOS, but the project is still evolving.  
> Expect possible breaking changes (prompts, models, API parameters). Feedback and PRs are very welcome.

Local-first microservice for **understanding videos**: scene segmentation + Whisper ASR + Qwen3-VL vision-language model + global summary.

## 💬 Community & Questions

If you have questions, ideas or feedback, please use the
[Discussions](https://github.com/dolphin-creator/VideoContext-Engine/discussions) tab.

Bug reports and feature requests are welcome in the
[Issues](https://github.com/dolphin-creator/VideoContext-Engine/issues) section.

> ⚠️ **Compatibility note**
> This project was originally architected and optimized for **macOS (Apple Silicon)** using the MLX framework.
> While support for **Windows and Linux** has been implemented (via `llama.cpp`), it has **not yet been extensively tested** on these platforms. You may encounter platform-specific bugs or installation hurdles. Feedback is welcome!

> ⚠️ **Windows / Linux (llama.cpp)**
> Default context size is **4096 tokens** (safe for small/medium videos).
> For long videos (>15–20 min), the **global summary** may be truncated.
> If you have **16GB+ RAM**, you can increase it in `VideoContextEngine_v3.19.py`
> inside the `LlamaCppEngine` class:
> ```python
> n_ctx = 16384  # or 32768 for very long videos
> ```
> (macOS / MLX users are not affected)

This version `v3.19` is a more industrial, robust evolution of the early 3.x line. It is designed to run **fully locally** (no external LLM calls) and provide structured context that can later be consumed by another LLM or RAG pipeline.

---

## ✨ Features Overview

- 🔍 **Automatic Scene Detection (CPU)**
  - HSV histogram-based visual change detection
  - Configurable minimum and maximum scene durations
  - Forced cut when a scene exceeds `max_scene_duration`

- 🎙️ **Audio Transcription (Whisper)**
  - Uses `openai-whisper` locally (`tiny`, `base`, `small`, `medium`, `large`, etc.)
  - Produces time-aligned segments with text
  - Segments are grouped back per scene
  - **Audio features per scene** (optional):
    - `speech_duration`
    - `speaking_rate_wpm`
    - `speech_ratio` (speech vs total time)
    - `silence_ratio`

- 👁️ **Visual Analysis (Qwen3-VL)**
  - macOS: **MLX backend** (`mlx-vlm`) with `mlx-community/Qwen3-VL-2B-Instruct-4bit`
  - Windows / Linux: **llama.cpp backend** with Qwen3-VL GGUF
  - 1–5 **keyframes per scene** (configurable, default: 1)
  - For each scene, the VLM returns:
    ```jsonc
    {
      "description": "Short, factual visual description of the scene.",
      "tags": {
        "people_count": 1,
        "place_type": "studio | tv_set | classroom | office | home | outdoor | nature | stage | other",
        "main_action": "short description of the main action",
        "emotional_tone": "calm | neutral | tense | conflictual | joyful | sad | enthusiastic | serious | other",
        "movement_level": "low | medium | high"
      }
    }
    ```
  - If the JSON is malformed / truncated, the engine tries to repair it and, if it fails, **cleans and extracts text** from the raw output so you still get a usable `visual_description`.

- 🧠 **Global Video Summary**
  - Combines all scene-level notes (audio + visual) into a single global summary
  - Uses the same VLM, with a separate summary prompt
  - The output language follows the **language used in the user prompt**

- 📤 **Flexible Output Formats**
  - `response_format="json"` (default) → structured JSON
  - `response_format="text"` → plain text report (`.txt`) downloadable in HTTP
  - Optional `generate_txt=true` → also writes a `.txt` report to disk on the server

- 🧱 **RAG-Friendly Structure**
  - For each scene you get:
    - `audio_transcript`
    - `audio_features` (optional)
    - `visual_description`
    - `visual_tags`
  - Global summary in `meta.global_summary`
  - Easy to flatten to Markdown before feeding another LLM / RAG

- 🧮 **RAM Modes**
  - `ram-` (default): load/unload Whisper and VLM on every request → **RAM friendly**
  - `ram+`: pre-loads and keeps models in RAM → **much faster** if you have enough memory

- 🧪 **Robustness**
  - `safe_json_parse` attempts to recover valid JSON even if the VLM “spills” text around it
  - Fallback cleaning functions extract the useful `description` / `summary` when JSON is irrecoverable
  - Detailed timing profile per request (Whisper load/infer, VLM load/infer, total)

---

## 🧰 Requirements

- Python **3.10+**
- FFmpeg installed on the system (for yt-dlp and video processing)
- GPU recommended (for Whisper and VLM), but CPU-only also works (slower)

### Install FFmpeg

**macOS (Homebrew)**

```bash
brew install ffmpeg
```

**Ubuntu / Debian**

```bash
sudo apt update
sudo apt install ffmpeg
```

**Windows (winget)**

```bash
winget install ffmpeg
```

---

## 🧪 Virtual Environment

```bash
python -m venv venv
```

**macOS / Linux**

```bash
source venv/bin/activate
```

**Windows (Cmd)**

```bash
venv\Scripts\activate
```

---

## 📦 Dependencies Installation

> The following commands are designed for copy/paste.

### Common (all platforms)

```bash
python -m pip install --upgrade pip
pip install fastapi "uvicorn[standard]" opencv-python yt-dlp pillow numpy openai-whisper huggingface_hub
# Whisper relies on PyTorch. If it's not installed yet:
# pip install torch
```

### 🍎 macOS (Apple Silicon / Intel) – MLX backend (Qwen3-VL 2B 4bit)

```bash
pip install mlx mlx-vlm torchvision
```

The model `mlx-community/Qwen3-VL-2B-Instruct-4bit` will be downloaded automatically via MLX on first use.

### 🐧 / 🪟 Linux & Windows – llama.cpp backend (Qwen3-VL GGUF)

```bash
pip install llama-cpp-python
```

The script can auto-download the default GGUF weights from Hugging Face:

- `qwen3-vl-2b-instruct-q4_k_m.gguf`
- `mmproj-model-f16.gguf`

---

## 🚀 Running the Server

Place `VideoContextEngine_v3.19.py` in your project directory.

### Default RAM Mode (`ram-` – memory friendly)

```bash
python VideoContextEngine_v3.19.py
```

- Whisper and VLM are **loaded and unloaded** for each request.
- Good for low-memory systems.

### High-Performance RAM Mode (`ram+`)

#### macOS / Linux

```bash
VIDEOCONTEXT_RAM_MODE=ram+ python VideoContextEngine_v3.19.py
```

#### Windows (PowerShell)

```powershell
$env:VIDEOCONTEXT_RAM_MODE="ram+"
python VideoContextEngine_v3.19.py
```

In `ram+` mode:

- Whisper (default `small`) and the VLM (default Qwen3-VL-2B 4bit) are **preloaded** and kept in RAM.
- Subsequent requests skip model loading → much faster end-to-end time.
- Recommended if you have **≥ 16 GB RAM** (with your 128 GB, you are more than fine).

By default, the server listens on:

```text
http://localhost:7555
```

---

## 🖥️ Visual Interface (Swagger / OpenAPI)

Open:

```text
http://localhost:7555/docs
```

You’ll see a Swagger UI.

1. Click on `POST /api/v1/analyze`
2. Click on **“Try it out”**
3. Fill either:
   - `video_file` (upload) **OR**
   - `video_url` (YouTube / direct link)
4. Key parameters to adjust:

   - `visual_user_prompt`  
     → extra instructions for scene visual descriptions  
     → example:  
       > "Describe the scene in **at most 80 words**, in English, focusing on gestures, posture and context."

   - `summary_user_prompt`  
     → extra instructions for the global summary  
     → example:  
       > "Summarize the video in **4 sentences max**, in English, as if explaining it to someone who hasn't seen it."

   - `response_format`  
     - `"json"` → full structured JSON (default)
     - `"text"` → downloadable plain-text report (`.txt`)

   - `keyframes_per_scene`  
     → from 1 to 5, **default = 1** (best speed/quality tradeoff)

   - `skip_audio` / `skip_visual`  
     → disable Whisper or VLM if needed

   - `enable_audio_features`  
     → compute or skip per-scene audio metrics (enabled by default)

   - `generate_summary`  
     → enable/disable the global summary (enabled by default)

5. Click **“Execute”** to run the analysis.

---

## ⚙️ API – Main Endpoint

### `POST /api/v1/analyze`

**Content-Type**: `multipart/form-data`

At least **one** of `video_file` or `video_url` must be provided.

### Main Parameters (Form fields)

#### 🎬 Video Input

| Name         | Type   | Description                                |
|--------------|--------|--------------------------------------------|
| `video_file` | File   | Local video file (optional)                |
| `video_url`  | string | YouTube or direct URL (optional)          |

#### 🧠 User Prompts (Vision & Summary)

| Name                 | Type   | Default (FR Text)                                                                                                                                                     |
|----------------------|--------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `visual_user_prompt` | string | "Décris factuellement ce qui se passe dans la scène à partir des images, en te concentrant sur les gestes, la posture, l'ambiance et le contexte."                   |
| `summary_user_prompt`| string | "Résume la vidéo de façon claire et concise en t'appuyant sur l'ensemble des scènes, comme si tu expliquais la vidéo à quelqu'un qui ne l'a pas vue."                 |

> 💡 **Important behavior:**
>
> - The **output language** (descriptions, tags, summary) will follow the **language used in these prompts**.  
>   - If you write in French → the model answers in French.  
>   - If you write in English → the model answers in English, etc.
> - To avoid the model being cut mid-sentence because of `max_tokens`:
>   - You should explicitly ask for a length that is **less than half** of the corresponding `vlm_max_tokens` value.
>   - Example:
>     - `vlm_max_tokens_scene = 220` → ask for **≤ 80–100 words** in `visual_user_prompt`.
>     - `vlm_max_tokens_summary = 260` → ask for **3–5 sentences** in `summary_user_prompt`.

#### 🤖 Models & VLM configuration

| Name                     | Type   | Default                                          | Description                                         |
|--------------------------|--------|--------------------------------------------------|-----------------------------------------------------|
| `vlm_model`              | string | OS-dependent (MLX or GGUF default)              | VLM identifier (Hugging Face ID or GGUF path)      |
| `whisper_model`          | string | `"small"`                                       | Whisper model size                                  |
| `vlm_resolution`         | int    | `768` (range 128–2048)                          | Image resize resolution for the VLM                 |
| `vlm_max_tokens_scene`   | int    | `220` (range 16–1024)                           | Max tokens for **per-scene** VLM outputs            |
| `vlm_max_tokens_summary` | int    | `260` (range 16–2048)                           | Max tokens for the **global summary**               |
| `keyframes_per_scene`    | int    | `1` (range 1–5)                                 | Number of keyframes per scene                       |

#### 🎬 Scene Detection parameters

| Name                  | Type   | Default | Description                                   |
|-----------------------|--------|---------|-----------------------------------------------|
| `scene_threshold`     | float  | `0.35`  | HSV histogram difference threshold            |
| `min_scene_duration`  | float  | `2.0`   | Minimum scene length in seconds               |
| `max_scene_duration`  | float  | `60.0`  | Maximum scene length before forced cut       |

#### 🎚 Toggles & options

| Name                    | Type | Default | Description                                                                |
|-------------------------|------|---------|----------------------------------------------------------------------------|
| `skip_audio`            | bool | `false` | If `true`, skip Whisper (no transcript, no audio features)                |
| `skip_visual`           | bool | `false` | If `true`, skip VLM (no visual descriptions)                              |
| `enable_audio_features` | bool | `true`  | If `true`, compute per-scene audio features                               |
| `generate_summary`      | bool | `true`  | If `true`, generate global summary                                        |
| `generate_txt`          | bool | `false` | If `true`, also write a `.txt` report to disk on the server               |
| `response_format`       | enum | `"json"`| `"json"` → JSON response; `"text"` → plain-text `.txt` HTTP response       |

---

## 🧾 Response Structure

### 1. `response_format = "json"` (default)

Simplified example:

```jsonc
{
  "meta": {
    "source": "https://www.youtube.com/shorts/xxxxx",
    "duration": 58.4,
    "process_time": 67.5,
    "global_summary": "The video explains the history of a computing machine...",
    "scene_count": 13,
    "models": {
      "vlm": "mlx-community/Qwen3-VL-2B-Instruct-4bit",
      "whisper": "small"
    },
    "skipped": {
      "audio": false,
      "visual": false
    },
    "params": {
      "keyframes_per_scene": 1,
      "enable_audio_features": true,
      "generate_summary": true,
      "vlm_max_tokens_scene": 220,
      "vlm_max_tokens_summary": 260,
      "ram_mode": "ram+"
    },
    "timings": {
      "total_process_time": 67.5,
      "total_request_time": 75.2,
      "whisper": {
        "model": "small",
        "load_time": 1.6,
        "inference_time": 9.1
      },
      "vlm": {
        "model": "mlx-community/Qwen3-VL-2B-Instruct-4bit",
        "load_time": 3.4,
        "inference_time": 42.1
      },
      "ram_mode": "ram+"
    }
  },
  "segments": [
    {
      "scene_id": 1,
      "start": 0.0,
      "end": 5.0,
      "audio_transcript": "A machine that would perform calculations for man...",
      "audio_features": {
        "speech_duration": 4.2,
        "speaking_rate_wpm": 145.3,
        "speech_ratio": 0.84,
        "silence_ratio": 0.16
      },
      "visual_description": "A presenter stands in a modern studio with a board behind him.",
      "visual_tags": {
        "people_count": 1,
        "place_type": "studio",
        "main_action": "standing presentation facing the camera",
        "emotional_tone": "serious",
        "movement_level": "low"
      },
      "emotion": {}
    }
  ],
  "txt_filename": null
}
```

> If `generate_txt=true`, `txt_filename` will contain the name of the generated report file (e.g. `video_context.txt`) in the current working directory.

### 2. `response_format = "text"`

- HTTP `Content-Type: text/plain`
- HTTP `Content-Disposition: attachment; filename="xxx_context.txt"`

Example content:

```text
### VIDEO CONTEXT (VideoContext v3.19)
Source : https://www.youtube.com/shorts/xxxxx
Duration : 58.40s | Processing : 67.49s
Config : Res=768px | Threshold=0.35 | Min=2.0s | Max=60.0s | Keyframes/scene=1 | RAM_MODE=ram+

--- GLOBAL SUMMARY ---
The video explores the history of the calculating machine ...

⏱️ [0.00 - 5.00] SCENE 1
   🎙️ TEXT : "A machine that would perform calculations for man ..."
   🔊 AudioFeatures : speech=0.84, silence=0.16, wpm=145.3
   👀 VISUAL : A presenter stands in a modern studio with a board behind him.
   🧩 Tags : people=1, place=studio, action=standing presentation facing the camera, tone=serious, movement=low

...
```

---

## 🧪 Example API Calls

### 1. cURL – JSON output

```bash
curl -X POST "http://localhost:7555/api/v1/analyze"   -F "video_url=https://www.youtube.com/shorts/owEhqKcFcu8"   -F "whisper_model=small"   -F "skip_audio=false"   -F "skip_visual=false"   -F "visual_user_prompt=Describe the scene in at most 80 words, in English, focusing on gestures and overall mood."   -F "summary_user_prompt=Summarize the video in 4 short sentences, in English."   -F "vlm_max_tokens_scene=220"   -F "vlm_max_tokens_summary=260"   -F "response_format=json"
```

### 2. cURL – Text report (`.txt`)

```bash
curl -X POST "http://localhost:7555/api/v1/analyze"   -F "video_url=https://www.youtube.com/shorts/owEhqKcFcu8"   -F "response_format=text"   -F "generate_summary=true"   -F "skip_audio=false"   -F "skip_visual=false"   -o video_context.txt
```

### 3. Python – Local file

```python
import requests

API_URL = "http://localhost:7555/api/v1/analyze"
VIDEO_PATH = "my_video.mp4"

with open(VIDEO_PATH, "rb") as f:
    files = {"video_file": f}
    data = {
        "visual_user_prompt": (
            "Describe what happens in the scene in at most 80 words, "
            "focusing on gestures, posture and overall context."
        ),
        "summary_user_prompt": (
            "Summarize the whole video in 3 to 5 sentences, "
            "as if explaining it to someone who has not seen it."
        ),
        "whisper_model": "small",
        "vlm_max_tokens_scene": 220,
        "vlm_max_tokens_summary": 260,
        "response_format": "json",
    }

    resp = requests.post(API_URL, files=files, data=data)

resp.raise_for_status()
data = resp.json()
print(data["meta"]["global_summary"])
```

---

## 🔐 Limits & Safety

- Maximum video duration: **4 hours** (configurable via `MAX_TOTAL_VIDEO_DURATION`)
- Accepted extensions: `.mp4`, `.mov`, `.avi`, `.mkv`, `.webm`, `.m4v`
- Temporary files are removed in a `finally` block
- No external cloud LLM calls – everything (Whisper, VLM) runs **locally**

---

## 🔌 OpenWebUI Tool (ContextVideo)

You can use VideoContext Engine directly from **OpenWebUI** via a custom tool, so that your local chat model can “see” and summarize videos from video links (YouTube, etc.).

- Example tool file: `examples/openwebui/contextvideo_tool.py`

**Quick setup:**

1. Run the engine locally:

```bash
python VideoContextEngine_v3.19.py
```

2. In OpenWebUI:  
   - `Workspace → Tools → New Tool` → paste `contextvideo_tool.py` → save.  
   - `Workspace → Models → (your model) → Tools` → enable **ContextVideo (Local VideoContext Engine)**.

The tool will detect the latest video URL in the chat, call `POST /api/v1/analyze` with `response_format=text`, then inject the full report back into the conversation and ask the LLM to summarize or answer your question.

Note that the tool is calibrated to a maximum of 900 seconds, but this can be modified within the tool.
---

### 🗣️ Language & prompts

The tool exposes two valves you should customize:

- `scene_prompt` – instructions for **per-scene visual analysis**  
- `summary_prompt` – instructions for the **global summary**

By default they are in **French** with a word limit.  
**Edit them in the OpenWebUI tool UI** and rewrite them in **your language** (EN/ES/IT/…), for example:

```text
scene_prompt:
"Describe what happens in the scene based on the images, focusing on gestures, posture, mood and context. Maximum 80 words."

summary_prompt:
"Summarize the whole video clearly and concisely using all scenes as context. Maximum 120 words."
```

The language of these prompts will drive the language and style of the engine output.

---

### ⚙️ Tips

- I **strongly recommend “warming up” your chat model** in a new chat before using ContextVideo:
  - send a first short message like `hello` or `bonjour`,
  - then send your video link and request (summary, analysis, etc.).  
  This avoids some edge-case issues on the very first request.

- For long videos, you can toggle the `engine_generate_summary` valve:
  - `true` → engine computes its own global summary (better for long videos),
  - `false` → let your chat model do the final summarization from the per-scene context.

- If you see: `Error: Could not connect to VideoContextEngine...`  
  check that the engine is running and, if needed, adjust `videocontext_base_url`
  in the tool valves (e.g. `http://localhost:7555` or your LAN IP).

## ✅ Summary

`VideoContext Engine v3.19` is a solid base to:

- cut videos into semantically meaningful scenes,
- extract both audio and visual context per scene,
- compute simple audio metrics (speech/silence, WPM),
- generate a global summary of the video,
- and expose everything through a clean HTTP API with either JSON or TXT outputs.

---

Use it:

- as a **local microservice** for your own tools,
- as a pre-processor for **RAG pipelines**,
- or as the “eyes and ears” of a higher-level assistant / agent.
