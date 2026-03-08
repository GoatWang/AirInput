# AirInput — Push-to-Talk Voice Input for macOS

Headless (no GUI) push-to-talk voice input. Hold a key → record mic → release → transcribe locally → type into any active app. All processing on-device, no cloud.

Inspired by [ByeType](https://byetype.com/blog/byetype-macos-desktop).

## Core Stack

| Component | Technology | Role |
|-----------|-----------|------|
| ASR | [Breeze ASR 25](https://huggingface.co/MediaTek-Research/Breeze-ASR-25) via whisper.cpp | Speech-to-text (Taiwanese Mandarin + English code-switching) |
| Text Polish | llama.cpp (optional) | Punctuation, filler removal, error correction |
| Audio Capture | PyAudio | 16kHz mono recording |
| Key Detection | pynput | Right Option (⌥) key listener |
| Text Input | Clipboard paste (⌘V) | Supports Unicode/CJK into any app |

## Architecture

```
Hold Right ⌥ → Start Recording
Release      → Stop Recording
Double-tap   → Toggle hands-free mode

┌─────────────┐     ┌──────────┐     ┌────────────┐     ┌──────────┐     ┌──────────┐
│  Key Event  │────▶│ PyAudio  │────▶│ whisper.cpp│────▶│llama.cpp │────▶│ Clipboard│
│  (pynput)   │     │ Record   │     │ Transcribe │     │ Polish   │     │ Paste ⌘V │
│             │     │ 16kHz    │     │ Breeze ASR │     │(optional)│     │          │
└─────────────┘     └──────────┘     └────────────┘     └──────────┘     └──────────┘
```

### Pipeline Detail

1. **Key Press (Right ⌥)** → Start recording
2. **Key Release** → Stop recording, save WAV (16kHz, mono, 16-bit)
3. **whisper.cpp** → `whisper-cli -m models/breeze-asr-25-q5_0.bin -f /tmp/airinput.wav -l zh --no-timestamps`
4. **llama.cpp** (optional) → Clean up transcription
5. **Clipboard paste** → Save current clipboard → copy result → ⌘V → restore clipboard

---

## Current Project State (2026-03-08)

### What's DONE (Phase 1 — Foundation)

| Item | Status | Details |
|------|--------|---------|
| Xcode CLI Tools | ✅ | Already installed |
| Homebrew | ✅ | cmake installed, ffmpeg + portaudio already present |
| whisper.cpp clone + build | ✅ | Built with **Metal acceleration** at `whisper.cpp/build/bin/whisper-cli` |
| Breeze ASR 25 download | ✅ | 2.9GB safetensors at `models/breeze-asr-25/model.safetensors` |
| GGUF conversion | ✅ | `whisper.cpp/models/convert-h5-to-ggml.py` works with Breeze ASR 25 |
| Q5_0 quantization | ✅ | **2.9GB → 1.0GB** at `models/breeze-asr-25-q5_0.bin` |
| CLAUDE.md | ✅ | Fail-fast conventions from PodcastContentProducer |
| Project proposal | ✅ | `prompts/20260308_0_airinput_project_proposal.md` |
| `/init_project` command | ✅ | `~/Projects/.claude/commands/init_project.md` — reusable project bootstrapper |

### What's IN PROGRESS

| Item | Status | Details |
|------|--------|---------|
| Key listener test (pynput) | ⏳ | Script at `prompts/20260308_1_test_key_listener.py`. Accessibility permission was granted but **not yet tested after granting**. Need to restart Terminal and re-run. |

### What's NOT DONE

| Item | Phase | Notes |
|------|-------|-------|
| Verify Right ⌥ key detection | Phase 1 | Run key listener test after Terminal restart. If `Key.alt_r` doesn't work, fallback to configurable key or CGEventTap via PyObjC |
| Run whisper.cpp inference test | Phase 1 | Test: `whisper.cpp/build/bin/whisper-cli -m models/breeze-asr-25-q5_0.bin -f <test.wav> -l zh` |
| `src/audio_recorder.py` | Phase 2 | PyAudio 16kHz mono capture with start/stop |
| `src/key_listener.py` | Phase 2 | pynput Right ⌥ detection + double-tap logic |
| `src/transcriber.py` | Phase 2 | whisper.cpp subprocess call |
| `src/text_inputter.py` | Phase 2 | Clipboard save/paste/restore |
| `src/config.py` | Phase 3 | YAML config loader |
| `breeze_type.py` | Phase 3 | Main event loop wiring all components |
| `src/text_polisher.py` | Phase 4 | llama.cpp integration (optional) |
| `setup.sh` | Phase 4 | Automated setup script |
| LaunchAgent plist | Phase 5 | Auto-start on login |

---

## How to Continue (For Next Agent)

### Immediate Next Steps

1. **Test key listener** — Restart Terminal (Accessibility permission was just granted), then:
   ```bash
   cd /tmp && uv run --no-project --with pynput python /Users/wanghsuanchung/Projects/AirInput/prompts/20260308_1_test_key_listener.py
   ```
   - If Right ⌥ shows as `Key.alt_r` → proceed with pynput
   - If both Option keys show as `Key.alt` → need configurable trigger key or PyObjC CGEventTap fallback

2. **Test whisper.cpp inference** — Record a short WAV and test:
   ```bash
   # Record 3 seconds of audio (need a test WAV file)
   cd /tmp && uv run --no-project --with sounddevice --with numpy --with scipy python -c "
   import sounddevice as sd
   import numpy as np
   from scipy.io import wavfile
   print('Recording 3 seconds...')
   audio = sd.rec(int(3 * 16000), samplerate=16000, channels=1, dtype='int16')
   sd.wait()
   wavfile.write('/tmp/airinput_test.wav', 16000, audio)
   print('Saved to /tmp/airinput_test.wav')
   "

   # Transcribe
   /Users/wanghsuanchung/Projects/AirInput/whisper.cpp/build/bin/whisper-cli \
     -m /Users/wanghsuanchung/Projects/AirInput/models/breeze-asr-25-q5_0.bin \
     -f /tmp/airinput_test.wav \
     -l zh --no-timestamps
   ```

3. **Build Phase 2 modules** — Once both tests pass, implement `src/` modules per the project structure below.

### Key Files to Read

| File | Purpose |
|------|---------|
| `CLAUDE.md` | Dev conventions (fail-fast, dict access, error handling) |
| `prompts/20260308_0_airinput_project_proposal.md` | Full architecture, design decisions, risk matrix, implementation plan |
| `prompts/20260308_1_test_key_listener.py` | Key listener prototype (not yet successfully run) |

### Conversion Commands (Already Done — For Reference)

The GGUF conversion pipeline that was validated:

```bash
# 1. Clone whisper.cpp and build with Metal
git clone https://github.com/ggerganov/whisper.cpp.git
cd whisper.cpp && cmake -B build -DGGML_METAL=ON && cmake --build build -j $(sysctl -n hw.ncpu)

# 2. Clone OpenAI whisper (for tokenizer assets)
git clone https://github.com/openai/whisper.git openai-whisper

# 3. Download model
cd /tmp && uv run --no-project --with huggingface_hub python -c "
from huggingface_hub import snapshot_download
snapshot_download('MediaTek-Research/Breeze-ASR-25', local_dir='models/breeze-asr-25',
                  ignore_patterns=['*.pkl', 'optimizer*', 'training_args*'])
"

# 4. Convert safetensors → GGML (requires: transformers, torch, safetensors, numpy)
cd /tmp && uv run --no-project --with transformers --with torch --with safetensors --with numpy \
  python whisper.cpp/models/convert-h5-to-ggml.py models/breeze-asr-25 openai-whisper models

# 5. Quantize to Q5_0
whisper.cpp/build/bin/whisper-quantize models/ggml-model.bin models/breeze-asr-25-q5_0.bin q5_0
```

Output: `models/ggml-model.bin` (2.9GB f32) → `models/breeze-asr-25-q5_0.bin` (1.0GB Q5_0)

---

## Project Structure

```
AirInput/
├── CLAUDE.md                        # Dev instructions (fail-fast, no silent failure)
├── README.md                        # This file
├── .gitignore
├── pyproject.toml                   # uv project config
├── breeze_type.py                   # Main application entry point
├── .claude/
│   ├── CLAUDE.md
│   ├── commands/
│   │   ├── review_prompt.md
│   │   └── todo.md
│   ├── agents/
│   │   └── core-review.md           # Silent failure detection agent
│   └── skills/
├── src/
│   ├── __init__.py
│   ├── audio_recorder.py            # PyAudio capture (16kHz mono)
│   ├── key_listener.py              # pynput key detection + double-tap
│   ├── transcriber.py               # whisper.cpp CLI integration
│   ├── text_polisher.py             # llama.cpp integration (optional)
│   ├── text_inputter.py             # Clipboard save/paste/restore
│   └── config.py                    # Configuration management
├── models/
│   ├── breeze-asr-25/               # Original HuggingFace download (2.9GB safetensors)
│   ├── ggml-model.bin               # Intermediate f32 GGML (2.9GB) — can delete after quantize
│   └── breeze-asr-25-q5_0.bin       # Production model (1.0GB Q5_0) ✅
├── whisper.cpp/                     # Built from source with Metal ✅
│   └── build/bin/whisper-cli        # Main inference binary
├── openai-whisper/                  # OpenAI whisper repo (for tokenizer assets during conversion)
├── tool_scripts/
│   └── convert_model/               # Model conversion utilities
├── prompts/                         # Working files (gitignored)
│   ├── _TODO.md
│   ├── 20260308_0_airinput_project_proposal.md
│   └── 20260308_1_test_key_listener.py
└── tests/
    ├── test_audio_recorder.py
    ├── test_key_listener.py
    └── test_transcriber.py
```

---

## Configuration (config.yaml)

```yaml
# AirInput Configuration
trigger_key: "alt_r"          # right option key (configurable)
language: "zh"                # zh, en, auto
double_tap_threshold_ms: 300  # ms between taps for toggle mode

# whisper.cpp
whisper_binary: "./whisper.cpp/build/bin/whisper-cli"
model_path: "./models/breeze-asr-25-q5_0.bin"
whisper_threads: 4

# llama.cpp (optional text polish)
polish_enabled: false
llama_binary: "./llama.cpp/build/bin/llama-cli"
llama_model: "./models/qwen2.5-3b-instruct-q4_k_m.gguf"
polish_prompt: "修正以下語音辨識文字的錯字，補上標點符號，移除語助詞（嗯、啊、那個），保持原意不改寫："

# Audio
sample_rate: 16000
channels: 1
chunk_size: 1024
temp_wav: "/tmp/airinput_recording.wav"

# Behavior
min_recording_seconds: 0.5   # ignore recordings shorter than this
max_recording_seconds: 300   # auto-stop after 5 minutes
```

---

## Design Decisions

### Why whisper.cpp over Python transformers?
- **4-8x faster** on Apple Silicon with Metal acceleration
- **~1.0GB RAM** (Q5_0) vs ~3GB+ for Python transformers
- No Python GIL bottleneck during inference
- Runs as subprocess — crash isolation from main app

### Why clipboard paste instead of pyautogui.typewrite()?
- `typewrite()` only supports ASCII — **cannot type CJK characters**
- Clipboard paste (⌘V) works with any Unicode text
- Supports all macOS apps uniformly
- Clipboard save/restore prevents data loss

### Why llama.cpp polish is optional?
- **Quality vs speed trade-off**: Adds 1-2s latency
- **Privacy**: Some users may not want LLM post-processing
- **Breeze ASR 25 is already good**: Fine-tuned for Traditional Chinese
- **Configurable**: Users can enable/disable via config

### Why Breeze ASR 25?
- Whisper-large-v2 fine-tune optimized for **Taiwanese Mandarin + English**
- Best-in-class for Traditional Chinese (5-star accuracy per ByeType comparison)
- **Code-switching support**: Handles mixed Chinese-English naturally
- Standard Whisper architecture → compatible with whisper.cpp conversion (verified)

---

## System Requirements

- macOS 14+ (Sonoma or later)
- Apple Silicon (M1/M2/M3/M4) — Metal acceleration
- Python 3.11+
- ~2GB disk for models
- ~1.5GB RAM during inference

## Permissions Required

| Permission | Where to Grant | Why |
|-----------|---------------|-----|
| Accessibility | System Settings → Privacy & Security → Accessibility | pynput key monitoring |
| Microphone | System Settings → Privacy & Security → Microphone | PyAudio recording |

---

## Dependencies

Python (via uv):
```
pyaudio>=0.2.14
pynput>=1.7.6
pyperclip>=1.8.2
numpy>=1.24.0
pyyaml>=6.0
```

System (via brew):
```bash
brew install cmake ffmpeg portaudio
```

Build from source:
- whisper.cpp (with Metal) — already built ✅
- llama.cpp (optional, for text polish)

---

## Troubleshooting

### "This process is not trusted! Input event monitoring will not be possible"
→ Grant Accessibility permission: System Settings → Privacy & Security → Accessibility → add your terminal app. **Restart the terminal after granting.**

### whisper.cpp returns garbage or crashes
→ Verify the model file: `ls -lh models/breeze-asr-25-q5_0.bin` should be ~1.0GB
→ Test with a known good WAV file (16kHz, mono, 16-bit PCM)

### PyAudio "No Default Input Device Available"
→ Grant Microphone permission: System Settings → Privacy & Security → Microphone
→ Check: `system_profiler SPAudioDataType` to list audio devices

### High latency (>3 seconds)
→ Ensure Metal is enabled (check whisper.cpp build log for "Metal framework found")
→ Reduce threads if memory-constrained: `whisper_threads: 2`
→ Disable polish: `polish_enabled: false`
