# AirInput — Push-to-Talk Voice Input for macOS

## What This App Does

AirInput is an **invisible background process** (no window, no dock icon, no GUI) that turns your voice into keyboard input for **any** macOS application.

**How it works from the user's perspective:**

1. You're typing in any app — VS Code, Slack, LINE, Safari, Notes, Terminal, anything
2. You **hold down Right Option (⌥)** and start speaking
3. You **release the key** — within ~1-2 seconds, the transcribed text appears at your cursor as if you typed it
4. Your clipboard is untouched — AirInput saves and restores it behind the scenes

**Hands-free mode:** Double-tap Right ⌥ quickly (< 300ms) to start continuous recording. Tap again to stop. Good for long dictation.

**Everything runs locally on your Mac.** No internet required. No cloud. No data leaves your machine. Audio is processed by whisper.cpp with Metal GPU acceleration on Apple Silicon. The app uses ~1.5GB RAM during inference.

**Primary language:** Taiwanese Mandarin + English code-switching (混合中英文). Powered by [Breeze ASR 25](https://huggingface.co/MediaTek-Research/Breeze-ASR-25), a Whisper-large-v2 fine-tune by MediaTek optimized for Traditional Chinese.

Inspired by [ByeType](https://byetype.com/blog/byetype-macos-desktop).

---

## Architecture

```
User holds Right ⌥ → Mic starts recording (16kHz mono)
User releases Right ⌥ → Recording stops
                       → WAV saved to /tmp
                       → whisper.cpp transcribes (Metal GPU, ~1-2s)
                       → (optional) llama.cpp polishes text
                       → Clipboard saved
                       → Transcription copied to clipboard
                       → ⌘V pasted into active app
                       → Original clipboard restored
                       → Text appears at cursor ✓

┌─────────────┐     ┌──────────┐     ┌────────────┐     ┌──────────┐     ┌──────────┐
│  Key Event  │────▶│ PyAudio  │────▶│ whisper.cpp│────▶│llama.cpp │────▶│ Clipboard│
│  (pynput)   │     │ Record   │     │ Transcribe │     │ Polish   │     │ Paste ⌘V │
│             │     │ 16kHz    │     │ Breeze ASR │     │(optional)│     │          │
└─────────────┘     └──────────┘     └────────────┘     └──────────┘     └──────────┘
```

### Why clipboard paste instead of simulated keystrokes?

`pyautogui.typewrite()` only supports ASCII — it **cannot type Chinese characters (中文), Japanese, Korean, or any CJK text**. Clipboard paste (⌘V) works with any Unicode text in any macOS app. AirInput saves your existing clipboard content before pasting and restores it immediately after, so you never lose what you copied.

### Why whisper.cpp instead of Python transformers?

- **4-8x faster** on Apple Silicon with Metal GPU acceleration
- **~1.0GB RAM** (quantized Q5_0) vs ~3GB+ for Python transformers
- No Python GIL bottleneck — runs as a subprocess with crash isolation
- Same Breeze ASR 25 model, just converted to efficient GGML format

### Why llama.cpp text polish is optional?

After whisper.cpp transcribes your speech, an optional llama.cpp step can:
- Fix recognition errors (錯字修正)
- Add punctuation (標點符號)
- Remove filler words (嗯、啊、那個)

This adds ~1-2s latency. Breeze ASR 25 is already good for Traditional Chinese, so many users won't need it. Configurable via `config.yaml`.

### Why Breeze ASR 25?

- Whisper-large-v2 fine-tune optimized for **Taiwanese Mandarin + English**
- Best-in-class accuracy for Traditional Chinese (5-star per ByeType comparison)
- **Code-switching support**: Handles mixed 中英文 naturally within the same sentence
- Standard Whisper architecture → verified compatible with whisper.cpp GGML conversion

---

## Core Stack

| Component | Technology | Role |
|-----------|-----------|------|
| ASR Engine | [Breeze ASR 25](https://huggingface.co/MediaTek-Research/Breeze-ASR-25) via [whisper.cpp](https://github.com/ggerganov/whisper.cpp) | Speech → text (Taiwanese Mandarin + English) |
| Text Polish | [llama.cpp](https://github.com/ggerganov/llama.cpp) (optional) | Punctuation, filler removal, error correction |
| Audio Capture | PyAudio (PortAudio) | 16kHz mono microphone recording |
| Hotkey Detection | pynput | Global key listener for Right Option (⌥) |
| Text Injection | Clipboard paste (⌘V) via pyperclip + pyautogui | Types into any active app, supports CJK |

---

## Usage

| Action | What Happens |
|--------|-------------|
| **Hold Right ⌥** | Start recording from microphone |
| **Release Right ⌥** | Stop recording → transcribe → paste text at cursor |
| **Double-tap Right ⌥** (< 300ms) | Toggle hands-free mode (continuous recording until next tap) |
| **Short press** (< 0.5s) | Ignored — prevents accidental triggers |

The app runs silently in the background. No window. No dock icon. Start it from Terminal and forget about it:

```bash
cd /Users/wanghsuanchung/Projects/AirInput && uv run breeze_type.py
```

---

## System Requirements

- macOS 14+ (Sonoma or later)
- Apple Silicon (M1/M2/M3/M4) — required for Metal GPU acceleration
- Python 3.11+
- ~2GB disk for models
- ~1.5GB RAM during inference

## macOS Permissions Required

| Permission | Path | Why | What happens without it |
|-----------|------|-----|------------------------|
| **Accessibility** | System Settings → Privacy & Security → Accessibility → add Terminal | pynput needs this to monitor global key events | **Silent failure** — no error, key listener just doesn't work |
| **Microphone** | System Settings → Privacy & Security → Microphone → enable Terminal | PyAudio needs this to capture audio | Error or silent audio (all zeros) |

**Important:** After granting Accessibility permission, you must **restart your Terminal app** for it to take effect.

---

## Quick Start

### 1. Install system dependencies

```bash
xcode-select --install          # Xcode Command Line Tools (for Metal headers)
brew install cmake ffmpeg portaudio
```

### 2. Clone and setup

```bash
git clone git@github.com:GoatWang/AirInput.git
cd AirInput
bash setup.sh                    # Builds whisper.cpp, downloads + converts model
```

### 3. Grant macOS permissions

- System Settings → Privacy & Security → **Accessibility** → add Terminal.app → restart Terminal
- System Settings → Privacy & Security → **Microphone** → enable Terminal.app

### 4. Run

```bash
uv run breeze_type.py
```

Hold Right ⌥, speak, release. Text appears at your cursor.

---

## Configuration (config.yaml)

```yaml
# Trigger key
trigger_key: "alt_r"              # Right Option (⌥). Configurable if your keyboard differs.
language: "zh"                    # zh = Mandarin, en = English, auto = auto-detect
double_tap_threshold_ms: 300      # Max ms between taps to trigger hands-free mode

# whisper.cpp
whisper_binary: "./whisper.cpp/build/bin/whisper-cli"
model_path: "./models/breeze-asr-25-q5_0.bin"
whisper_threads: 4                # CPU threads for inference

# llama.cpp text polish (optional — disabled by default)
polish_enabled: false
llama_binary: "./llama.cpp/build/bin/llama-cli"
llama_model: "./models/qwen2.5-3b-instruct-q4_k_m.gguf"
polish_prompt: "修正以下語音辨識文字的錯字，補上標點符號，移除語助詞（嗯、啊、那個），保持原意不改寫："

# Audio recording
sample_rate: 16000                # 16kHz required by Whisper
channels: 1                      # Mono
chunk_size: 1024
temp_wav: "/tmp/airinput_recording.wav"

# Behavior
min_recording_seconds: 0.5       # Ignore recordings shorter than this (prevents accidental taps)
max_recording_seconds: 300       # Auto-stop after 5 minutes
```

---

## Project Structure

```
AirInput/
├── CLAUDE.md                        # Dev conventions (fail-fast, no silent failure)
├── README.md                        # This file
├── .gitignore
├── pyproject.toml
├── config.yaml                      # Runtime configuration
├── breeze_type.py                   # Main entry point — run this
├── setup.sh                         # One-command setup (brew, whisper.cpp, model)
├── .claude/
│   ├── CLAUDE.md
│   ├── commands/                    # Claude Code slash commands
│   │   ├── review_prompt.md
│   │   ├── debug_prompt.md
│   │   └── todo.md
│   └── agents/
│       └── core-review.md           # Silent failure detection agent
├── src/
│   ├── __init__.py
│   ├── audio_recorder.py            # PyAudio 16kHz mono capture with start/stop
│   ├── key_listener.py              # pynput hotkey detection + double-tap logic
│   ├── transcriber.py               # whisper.cpp subprocess integration
│   ├── text_polisher.py             # llama.cpp text cleanup (optional)
│   ├── text_inputter.py             # Clipboard save → paste → restore
│   └── config.py                    # YAML config loader
├── models/                          # gitignored — created by setup.sh
│   ├── breeze-asr-25/               # Original HuggingFace download (2.9GB)
│   ├── ggml-model.bin               # Intermediate f32 (2.9GB, deletable)
│   └── breeze-asr-25-q5_0.bin       # Production model (1.0GB Q5_0)
├── whisper.cpp/                     # gitignored — built from source by setup.sh
├── openai-whisper/                  # gitignored — needed for model conversion only
├── tool_scripts/
│   └── convert_model/
├── prompts/                         # gitignored — working files
│   └── _TODO.md
└── tests/
    ├── test_audio_recorder.py
    ├── test_key_listener.py
    └── test_transcriber.py
```

---

## Current Build State (2026-03-08)

This section tells the next developer (or Claude agent) exactly where things stand.

### Phase 1: Foundation — MOSTLY DONE

| Task | Status | Details |
|------|--------|---------|
| Xcode CLI Tools | ✅ Done | Already installed |
| Homebrew deps | ✅ Done | cmake, ffmpeg, portaudio all installed |
| whisper.cpp build | ✅ Done | Built with **Metal acceleration** at `whisper.cpp/build/bin/whisper-cli` |
| Breeze ASR 25 download | ✅ Done | 2.9GB safetensors at `models/breeze-asr-25/model.safetensors` |
| Safetensors → GGML conversion | ✅ Done | Used `whisper.cpp/models/convert-h5-to-ggml.py` — works perfectly |
| Q5_0 quantization | ✅ Done | 2.9GB → **1.0GB** at `models/breeze-asr-25-q5_0.bin` |
| CLAUDE.md | ✅ Done | Fail-fast conventions adapted from PodcastContentProducer |
| GitHub repo | ✅ Done | https://github.com/GoatWang/AirInput |
| Key listener test | ⏳ Blocked | Accessibility permission granted, but **Terminal not restarted yet** — test not run |
| whisper.cpp inference test | ❌ Not done | Need to record a test WAV and run whisper-cli |

### Phase 2: Core Modules — NOT STARTED

| Module | File | Description |
|--------|------|-------------|
| Audio Recorder | `src/audio_recorder.py` | PyAudio capture: start/stop recording, save 16kHz mono WAV |
| Key Listener | `src/key_listener.py` | pynput: detect Right ⌥ press/release, double-tap detection with 300ms threshold |
| Transcriber | `src/transcriber.py` | Run whisper.cpp as subprocess, parse stdout for transcription text |
| Text Inputter | `src/text_inputter.py` | Save clipboard → copy text → ⌘V paste → restore clipboard |
| Config | `src/config.py` | Load config.yaml, validate required fields exist |

### Phase 3: Integration — NOT STARTED

| Task | Description |
|------|-------------|
| `breeze_type.py` | Main event loop: key_listener triggers audio_recorder, on release calls transcriber, then text_inputter |
| End-to-end test | Hold key → speak → release → text appears in a text editor |
| Error handling | Startup checks: model file exists, mic permission granted, accessibility permission granted |

### Phase 4: Polish — NOT STARTED

| Task | Description |
|------|-------------|
| `src/text_polisher.py` | llama.cpp integration for punctuation/filler removal |
| `setup.sh` | Automated one-command setup script |
| Edge cases | Very short recordings (< 0.5s), empty transcriptions, rapid key presses |

### Phase 5: Hardening — NOT STARTED

| Task | Description |
|------|-------------|
| LaunchAgent plist | Auto-start AirInput on macOS login |
| Graceful shutdown | Handle SIGTERM/SIGINT, release mic, clean up temp files |
| Memory profiling | Target < 2GB idle with model loaded |

---

## How to Continue (For Next Agent)

**READ THIS SECTION FIRST before writing any code.**

### Step 1: Test the two remaining Phase 1 items

#### 1a. Key listener test

Accessibility permission was granted on the previous machine but Terminal was not restarted. On the new machine, ensure Accessibility is granted (System Settings → Privacy & Security → Accessibility → Terminal), then:

```bash
cd /tmp && uv run --no-project --with pynput python /Users/wanghsuanchung/Projects/AirInput/prompts/20260308_1_test_key_listener.py
```

- Press Right Option (⌥) and Left Option (⌥)
- **Expected:** Right shows `Key.alt_r`, Left shows `Key.alt` or `Key.alt_l`
- **If both show `Key.alt`:** Right ⌥ can't be distinguished. Options:
  1. Use a different configurable trigger key
  2. Switch to PyObjC `CGEventTap` for lower-level key code access
  3. Use `NSEvent.addGlobalMonitorForEvents` via PyObjC

#### 1b. whisper.cpp inference test

Record audio and test transcription:

```bash
# Record 3 seconds
cd /tmp && uv run --no-project --with sounddevice --with numpy --with scipy python -c "
import sounddevice as sd
import numpy as np
from scipy.io import wavfile
print('Recording 3 seconds... speak now!')
audio = sd.rec(int(3 * 16000), samplerate=16000, channels=1, dtype='int16')
sd.wait()
wavfile.write('/tmp/airinput_test.wav', 16000, audio)
print('Saved to /tmp/airinput_test.wav')
"

# Transcribe with Breeze ASR 25
/Users/wanghsuanchung/Projects/AirInput/whisper.cpp/build/bin/whisper-cli \
  -m /Users/wanghsuanchung/Projects/AirInput/models/breeze-asr-25-q5_0.bin \
  -f /tmp/airinput_test.wav \
  -l zh --no-timestamps
```

**Note:** If whisper.cpp/models don't exist on the new machine, you need to re-run setup. See "Model Conversion Commands" below.

### Step 2: Build Phase 2 modules

Once both tests pass, implement the `src/` modules. Each module should be independently testable. Follow fail-fast conventions in CLAUDE.md.

### Step 3: Wire up in breeze_type.py

The main loop logic:

```python
# Pseudocode for breeze_type.py
config = load_config("config.yaml")
recorder = AudioRecorder(config)
transcriber = Transcriber(config)
inputter = TextInputter()
polisher = TextPolisher(config) if config["polish_enabled"] else None

def on_key_press(key):
    if key == trigger_key:
        recorder.start()

def on_key_release(key):
    if key == trigger_key:
        wav_path = recorder.stop()
        if wav_path:  # recording was long enough
            text = transcriber.transcribe(wav_path)
            if polisher:
                text = polisher.polish(text)
            if text.strip():
                inputter.type_text(text)

# Start global key listener (blocks)
listener = KeyListener(on_press=on_key_press, on_release=on_key_release, config=config)
listener.run()
```

---

## Model Conversion Commands (Reference)

These were already run and verified on the original machine. Re-run only if setting up a new machine.

```bash
# 1. Clone whisper.cpp and build with Metal
git clone https://github.com/ggerganov/whisper.cpp.git
cd whisper.cpp && cmake -B build -DGGML_METAL=ON && cmake --build build -j $(sysctl -n hw.ncpu)
cd ..

# 2. Clone OpenAI whisper repo (tokenizer assets needed for conversion)
git clone https://github.com/openai/whisper.git openai-whisper

# 3. Download Breeze ASR 25 from HuggingFace
cd /tmp && uv run --no-project --with huggingface_hub python -c "
from huggingface_hub import snapshot_download
snapshot_download('MediaTek-Research/Breeze-ASR-25',
                  local_dir='/Users/wanghsuanchung/Projects/AirInput/models/breeze-asr-25',
                  ignore_patterns=['*.pkl', 'optimizer*', 'training_args*'])
"

# 4. Convert safetensors → GGML format
cd /tmp && uv run --no-project --with transformers --with torch --with safetensors --with numpy \
  python /Users/wanghsuanchung/Projects/AirInput/whisper.cpp/models/convert-h5-to-ggml.py \
  /Users/wanghsuanchung/Projects/AirInput/models/breeze-asr-25 \
  /Users/wanghsuanchung/Projects/AirInput/openai-whisper \
  /Users/wanghsuanchung/Projects/AirInput/models

# 5. Quantize GGML → Q5_0 (2.9GB → 1.0GB)
/Users/wanghsuanchung/Projects/AirInput/whisper.cpp/build/bin/whisper-quantize \
  /Users/wanghsuanchung/Projects/AirInput/models/ggml-model.bin \
  /Users/wanghsuanchung/Projects/AirInput/models/breeze-asr-25-q5_0.bin \
  q5_0

# Result: models/breeze-asr-25-q5_0.bin (1.0GB) — this is the production model
# Optional: delete intermediate files to save space
# rm models/ggml-model.bin          # 2.9GB f32 intermediate
# rm -rf models/breeze-asr-25/      # 2.9GB original safetensors
# rm -rf openai-whisper/            # Only needed during conversion
```

**Key gotcha:** `huggingface-cli` doesn't work via `uv run` (entry point not exposed). Use the Python API (`from huggingface_hub import snapshot_download`) instead. Also always use `--no-project` flag with `uv run` when running from `/tmp` to avoid pyproject.toml conflicts.

---

## Dependencies

Python (managed by uv via pyproject.toml):
```
pyaudio>=0.2.14       # Audio capture (requires: brew install portaudio)
pynput>=1.7.6         # Global key listener
pyperclip>=1.8.2      # Clipboard read/write
numpy>=1.24.0         # Audio buffer handling
pyyaml>=6.0           # Config file parsing
```

System (via Homebrew):
```bash
brew install cmake ffmpeg portaudio
```

Build from source (by setup.sh):
- [whisper.cpp](https://github.com/ggerganov/whisper.cpp) — with Metal GPU acceleration
- [llama.cpp](https://github.com/ggerganov/llama.cpp) — optional, for text polish

---

## Troubleshooting

### "This process is not trusted! Input event monitoring will not be possible"
Accessibility permission not granted or Terminal not restarted after granting.
→ System Settings → Privacy & Security → Accessibility → add Terminal.app
→ **Quit and reopen Terminal** (required for permission to take effect)

### Key listener runs but doesn't detect any keys
Same as above — Accessibility permission issue. Also check: if running inside VS Code integrated terminal, **VS Code itself** needs Accessibility permission, not Terminal.app.

### whisper.cpp returns garbage or crashes
→ Verify model: `ls -lh models/breeze-asr-25-q5_0.bin` should be ~1.0GB
→ Test with known good WAV: must be 16kHz, mono, 16-bit PCM
→ Check Metal: rebuild with `cmake -B build -DGGML_METAL=ON`

### PyAudio "No Default Input Device Available"
→ System Settings → Privacy & Security → Microphone → enable Terminal
→ Verify: `system_profiler SPAudioDataType`

### High latency (> 3 seconds)
→ Verify Metal is active (build log should show "Metal framework found")
→ Try fewer threads: `whisper_threads: 2`
→ Disable polish: `polish_enabled: false`
→ Check if model is Q5_0 (~1.0GB), not f32 (~2.9GB)

### Clipboard content lost after paste
→ Bug in text_inputter.py — clipboard save/restore must happen atomically
→ Check that restore runs even if ⌘V paste fails
