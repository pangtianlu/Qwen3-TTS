# AGENTS.md

## Cursor Cloud specific instructions

This repo is the **`qwen-tts`** Python package (Qwen3-TTS): a text-to-speech library plus a
Gradio web UI demo (`qwen-tts-demo`). The public entry points are `Qwen3TTSModel` and
`Qwen3TTSTokenizer` (see `README.md` for full usage and model list). There is no test suite
and no lint config in this repo (the only GitHub workflows are translation/inactive helpers),
so "build/run" here means importing the package and synthesizing audio.

### Environment layout
- Dependencies are installed into a virtualenv at `/workspace/.venv` (system Python is
  PEP 668 "externally managed", so a venv is required). The startup/update script recreates
  and refreshes it via `pip install -e .`.
- **Always activate the venv first:** `source /workspace/.venv/bin/activate` before running
  `python`, `pip`, or `qwen-tts-demo`.
- System packages `sox` (+ `libsox-fmt-all`) and `python3.12-venv` are required and are
  baked into the environment snapshot. `sox` (the CLI binary) is only actually used by the
  25Hz tokenizer path; without it, imports print a noisy "SoX could not be found!" banner but
  the 12Hz models still work.

### No GPU — run everything on CPU
This VM has **no GPU** (`torch.cuda.is_available()` is `False`). The README/CLI defaults
(`--device cuda:0`, `--dtype bfloat16`, FlashAttention-2) will fail here. Always override:
- Pass `--device cpu --dtype float32 --no-flash-attn` to `qwen-tts-demo`.
- In Python, use `Qwen3TTSModel.from_pretrained(..., device_map="cpu", dtype=torch.float32, attn_implementation=None)`.
- `flash-attn` is intentionally **not** installed; the code prints a warning and falls back to
  a manual PyTorch attention path automatically. This is expected, not an error.

### Models are downloaded on demand from Hugging Face
`from_pretrained` pulls weights from Hugging Face into `~/.cache/huggingface`. Prefer the
smallest checkpoint on CPU: **`Qwen/Qwen3-TTS-12Hz-0.6B-CustomVoice`** (~2.5 GB, and it
bundles its own speech tokenizer, so no separate tokenizer repo is needed). The 1.7B models
work too but are slower on CPU. Model downloads are **not** part of the update script; the
0.6B CustomVoice weights are pre-cached in the snapshot. Other checkpoints
(`*-VoiceDesign`, `*-Base`, `Qwen3-TTS-Tokenizer-12Hz`, 1.7B variants) download on first use.

### Run the web UI demo
```bash
source /workspace/.venv/bin/activate
qwen-tts-demo Qwen/Qwen3-TTS-12Hz-0.6B-CustomVoice \
  --device cpu --dtype float32 --no-flash-attn --ip 0.0.0.0 --port 8000
```
Then open `http://localhost:8000`. On CPU, a short sentence synthesizes in ~10-30s. The demo
UI adapts to the model type: CustomVoice (speaker + optional instruct), VoiceDesign (style
description), or Base (voice clone from reference audio). Note the 0.6B CustomVoice model
ignores the instruct field by design.

### Quick sanity check (headless synthesis)
```bash
source /workspace/.venv/bin/activate
python -c "import torch, soundfile as sf; from qwen_tts import Qwen3TTSModel; \
m=Qwen3TTSModel.from_pretrained('Qwen/Qwen3-TTS-12Hz-0.6B-CustomVoice', device_map='cpu', dtype=torch.float32, attn_implementation=None); \
w,sr=m.generate_custom_voice(text='Hello world.', language='English', speaker='Ryan', max_new_tokens=512); \
sf.write('out.wav', w[0], sr); print('wrote out.wav', len(w[0])/sr, 's')"
```
