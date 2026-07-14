# Intel NPU LLM Server

Run large language models on the Intel NPU built into your Core Ultra laptop, behind an
OpenAI-compatible API. No cloud, no GPU — just the AI Boost chip that's already in the box.

## What it does

- **Runs on the NPU** — Intel's Neural Processing Unit does the inference, so it sips power
  compared to CPU or GPU.
- **OpenAI-compatible API** — point any OpenAI client at it (Open WebUI, LangChain, N8N, curl).
- **Built-in chat UI** — a dark-mode web interface at `http://localhost:8000`. No Docker needed
  for the basic experience.
- **Multiple models** — load and switch between several models from the UI or the API.
- **Conversation history** — chats are saved server-side and restored when you reopen the UI.
- **Memory** — long chats get summarized into per-conversation memory the model can draw on later.
- **Tool calling** — function/tool calling for agents.
- **Stays local** — everything runs on your machine; nothing is sent anywhere unless you point it
  at a remote service yourself.

## Demos

| Quick overview & speed test | Feature deep dive |
| :---: | :---: |
| [![UI & performance demo](https://img.youtube.com/vi/00RTemT1Bbs/0.jpg)](https://www.youtube.com/watch?v=00RTemT1Bbs) | [![Building with Intel NPU & the OpenAI API](https://img.youtube.com/vi/6F6LbR2Xjcg/0.jpg)](https://www.youtube.com/watch?v=6F6LbR2Xjcg) |

## Requirements

- **CPU**: Intel Core Ultra with an NPU / AI Boost device. Meteor Lake, Arrow Lake, and Lunar
  Lake are the known-good generations; newer Core Ultra parts will run but may print a warning.
- **OS**: Windows 11 (build 22000 or newer).
- **NPU driver**: version 32.0.100.3104 or newer.
- **Python**: 3.11, managed through Miniconda.
- **Docker** (optional): only if you want the Open WebUI frontend.

## Conversation history, memory, and continual training

The server remembers your chats and can learn from them.

- **Cross-session history** — every turn is written to `intel-npu-llm/conversations/<id>.json` and
  also mirrored into the browser's `localStorage`. Refresh or reopen the page and the chat comes
  back; pick past chats from the **💬 History** sidebar.
- **Per-conversation memory** — when a chat grows past the context window, the dropped turns are
  summarized (by the strongest model you have loaded) into structured long-term memory scoped to
  that conversation, shown in the **🧠 Memory** sidebar. It's only injected where it belongs, and
  you can view or clear it.
- **Continual training** — the NPU is inference-only, so fine-tuning happens on CPU (your 32 GB of
  RAM): conversations become a LoRA (`train/finetune_lora.py`), get merged into the base weights
  (`train/merge.py`), recompiled to NPU `sym_int4`, and versioned under `adapters/`. Enable a
  trained variant per model via the `active_adapter` field in `models.json`. See `MULTI_MODEL.md`
  for the math.
- **API-key auth** — set `API_KEY` to lock down the OpenAI-compatible endpoints when you expose the
  server to the rest of your network. The local UI stays open.

```bat
:: 1) Collect chats from the UI, then turn them into a clean dataset
conda activate ipex-npu
python intel-npu-llm\train\prepare_dataset.py

:: 2) Fine-tune a LoRA on CPU (aim for a small/medium model)
python intel-npu-llm\train\finetune_lora.py --model qwen2.5-3b

:: 3) Merge + compile to NPU, then deploy
python intel-npu-llm\train\merge.py --model qwen2.5-3b --variant v<timestamp> --set-active
```

Training on raw chat logs will hurt the model, so `prepare_dataset.py` filters for quality. Look
over `train/data/train_sharegpt.jsonl` before you start a training run.

## Quick start

### 1. Install dependencies (first time only)

The easiest path is the bundled setup script, which handles Miniconda, the conda environment, and
all the Python packages:

```bat
.\setup.bat
```

It installs Miniconda if it's missing, creates the `ipex-npu` conda environment (Python 3.11),
installs `ipex-llm[npu]` plus the server dependencies, and can save a HuggingFace token. Each step
is skipped if it's already done, so re-running after a failure is safe.

Before installing anything, `setup.bat` also runs a hardware preflight: Windows build, CPU family,
NPU driver version, and any BIOS versions flagged in `setup/compatibility.json`.

Optional overrides:
```powershell
.\setup.bat -AllowUnsupportedHardware
.\setup.bat -SkipDriverCheck
.\setup.bat -EnvName my-ipex-npu
```

Support and validation tooling:
```powershell
powershell -ExecutionPolicy Bypass -File .\setup\collect_support_info.ps1
powershell -ExecutionPolicy Bypass -File .\setup\test_compatibility_matrix.ps1
```

If Miniconda was installed fresh, the script will ask you to close and reopen the terminal, then
run `.\setup.bat` again to finish.

<details>
<summary>Manual setup (advanced)</summary>

```powershell
# Install Miniconda (if not installed)
winget install Anaconda.Miniconda3
# Reopen terminal, then:

conda create -n ipex-npu python=3.11 -y
conda activate ipex-npu

pip install --pre --upgrade ipex-llm[npu]
pip install -r intel-npu-llm/requirements.txt
```

</details>

### 1b. HuggingFace auth (for gated models)

Some models (Llama 2, Llama 3, Llama 3.2) need a HuggingFace login. `setup.bat` will prompt for it,
or set it up by hand:

1. Make an account at [huggingface.co](https://huggingface.co).
2. Accept the model license on its page (e.g.
   [meta-llama/Llama-3.2-3B-Instruct](https://huggingface.co/meta-llama/Llama-3.2-3B-Instruct)).
3. Create a token at [huggingface.co/settings/tokens](https://huggingface.co/settings/tokens).
4. Run the token script:

```bat
powershell -ExecutionPolicy Bypass -File .\setup\04_hf_token.ps1
```

Or just write `intel-npu-llm/.env` yourself:
```
HF_TOKEN=hf_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

Without this, gated models won't download. Non-gated models (Qwen, DeepSeek, MiniCPM, GLM-Edge,
Baichuan2) work without any auth.

### 2. Start the server

```powershell
# From the project root - loads qwen1.5-1.8b by default
.\start_server.bat
```

`start_server.bat` runs the same preflight before starting and sets `IPEX_LLM_NPU_MTL=1`
automatically on known Meteor Lake systems. Newer/unknown Core Ultra generations start in a generic
profile unless the preflight blocks them.

Useful overrides:
```powershell
$env:NPU_CONDA_ENV = 'ipex-npu'
$env:NPU_ALLOW_UNSUPPORTED = '1'
$env:NPU_SKIP_DRIVER_CHECK = '1'
$env:NPU_SKIP_PREFLIGHT = '1'
.\start_server.bat --list
```

For a bug report, generate a redacted support bundle:
```powershell
.\start_server.bat --diagnose
```

Load specific models:
```powershell
.\start_server.bat --models "qwen1.5-1.8b,llama3.2-1b,qwen1.5-4b"
```

List what's available:
```powershell
.\start_server.bat --list
```

Use a different port if 8000 is taken:
```powershell
.\start_server.bat --port 8001
```

Both at once:
```powershell
.\start_server.bat --models "qwen1.5-4b" --port 8080
```

Or start it by hand:
```powershell
$env:IPEX_LLM_NPU_MTL = "1"  # Meteor Lake (Core Ultra Series 1) only
conda activate ipex-npu
cd intel-npu-llm
python npu_server.py
```

### 3. Open WebUI (optional)

```powershell
cd intel-npu-llm
docker compose up -d
```

### 4. Use it

#### Built-in chat UI (no Docker)

Open **http://localhost:8000**:

- Live NPU status with simple indicators (Connecting, Busy, Idle)
- Model picker that fills in as models load
- Conversation history with multi-turn context
- Markdown rendering (code blocks, lists, bold/italic)
- Keyboard shortcuts: `Enter` sends, `Shift+Enter` for a newline, `Ctrl+L` clears
- Live telemetry: NPU busy state, system RAM, model disk footprint (NPU + HF cache)
- A running token count for the session

#### API endpoints

- `/` — the built-in chat UI
- `/v1/chat/completions` — OpenAI Chat Completions (Open WebUI, LangChain, curl)
- `/v1/responses` — OpenAI Responses API (N8N)
- `/v1/models` — list loaded models
- `/v1/models/load` — queue a model to load
- `/v1/models/unload` — unload a model
- `/v1/models/delete` — delete local model caches (HF & NPU)
- `/v1/system/status` — telemetry (memory, CPU, NPU busy state)
- `/health` — health check
- `/v1/memory` — get the global memory store
- `/v1/memory/clear` — clear the global memory store
- `/v1/conversations` — conversation store (see below)

#### Conversation store API

Conversations are the server's source of truth for chat history and the input to the
continual-training pipeline. They're stored as JSON under `intel-npu-llm/conversations/`.

| Method | Path | What it does |
|--------|------|--------------|
| `POST` | `/v1/conversations` | Create a conversation. Optional JSON body: `{"id": "...", "model": "...", "title": "..."}`. If `id` is omitted, one is generated. Returns `{"id", "model", "title"}`. |
| `GET` | `/v1/conversations` | List all conversations (newest first), each with `id`, `model`, `created`, `updated`, `turns`, `title`. |
| `GET` | `/v1/conversations/{id}` | Fetch one conversation's full saved state. |
| `POST` | `/v1/conversations/{id}/messages` | Replace the canonical message list for a conversation. Body: `{"messages": [...], "model": "..."}`. Idempotent — useful for syncing the UI's view of a chat. |
| `DELETE` | `/v1/conversations/{id}` | Delete a conversation. Returns `{"id", "deleted": true/false}`. |

Conversation ids must match `[A-Za-z0-9_-]+`; anything else is rejected.

### 5. Point your own Open WebUI at it (optional)

If Open WebUI is already running somewhere (say, on a homelab box), point it at the NPU server:

1. In Open WebUI: **Settings → Connections → OpenAI API**.
2. Add a connection:
   - **API Base URL**: `http://<YOUR-WINDOWS-PC-IP>:8000/v1`
   - **API Key**: `sk-dummy` (the server doesn't actually check it)
3. Save, and your NPU models show up in the model dropdown.

Grab your Windows IP with `ipconfig`. If you're connecting from another machine, allow port 8000
through the Windows Firewall.

### 6. Connect N8N (optional)

1. In N8N, add an **OpenAI** node.
2. Credentials:
   - **API Key**: `sk-dummy`
   - **Base URL**: `http://<YOUR-WINDOWS-PC-IP>:8000/v1`
3. Pick a loaded model id (e.g. `qwen1.5-1.8b`).

N8N uses the `/v1/responses` endpoint, which is supported.

### 7. Tool / function calling (agents)

The server speaks OpenAI-style tool calling:

```json
{
    "model": "qwen2.5-7b",
    "messages": [{"role": "user", "content": "What's the weather in NYC?"}],
    "tools": [{
        "type": "function",
        "function": {
            "name": "get_weather",
            "description": "Get weather for a location",
            "parameters": {
                "type": "object",
                "properties": {"location": {"type": "string"}},
                "required": ["location"]
            }
        }
    }],
    "tool_choice": "auto"
}
```

`tool_choice` behavior:

| `tool_choice` | Behavior |
|---------------|----------|
| `"auto"` | Model decides whether to call a tool (default) |
| `"none"` | No tool calling; respond normally |
| `"required"` | Force at least one tool call |
| `{"type": "function", "function": {"name": "get_weather"}}` | Force a specific tool |

Other notes: parallel tool calls, streaming tool calls (emitted at the end of the stream), a retry
on malformed calls (up to 2 tries), and only defined tools are parsed. Tool calling works best on
3B+ models; smaller ones tend to struggle.

---

## Supported models

All of these are verified for Intel NPU via ipex-llm.

### Qwen (recommended)
| Model ID | Size | NPU speed | Notes |
|----------|------|-----------|-------|
| `qwen1.5-1.8b` | 1.8B | ~8 tok/s | Default — verified working |
| `qwen1.5-4b` | 4B | ~5 tok/s | Better quality |
| `qwen1.5-7b` | 7B | ~3 tok/s | Best Qwen1.5 |
| `qwen2-1.5b` | 1.5B | ~10 tok/s | NPU verified |
| `qwen2-7b` | 7B | ~3 tok/s | NPU verified |
| `qwen2.5-3b` | 3B | ~8 tok/s | Latest Qwen |
| `qwen2.5-7b` | 7B | ~3 tok/s | Best Qwen 2.5 |

### Llama
| Model ID | Size | NPU speed | Notes |
|----------|------|-----------|-------|
| `llama2-7b` | 7B | ~3 tok/s | Classic; needs HF login |
| `llama3-8b` | 8B | ~2 tok/s | Powerful; needs HF login |
| `llama3.2-1b` | 1B | ~15 tok/s | Fastest Llama; needs HF login |
| `llama3.2-3b` | 3B | ~10 tok/s | Fast & capable; needs HF login |

### DeepSeek R1 (reasoning)
| Model ID | Size | NPU speed | Notes |
|----------|------|-----------|-------|
| `deepseek-1.5b` | 1.5B | ~10 tok/s | Fast reasoning |
| `deepseek-7b` | 7B | ~3 tok/s | Best reasoning |

### GLM-Edge (bilingual)
| Model ID | Size | NPU speed | Notes |
|----------|------|-----------|-------|
| `glm-edge-1.5b` | 1.5B | ~10 tok/s | Chinese/English |
| `glm-edge-4b` | 4B | ~5 tok/s | Larger bilingual model |

### MiniCPM (ultra-compact)
| Model ID | Size | NPU speed | Notes |
|----------|------|-----------|-------|
| `minicpm-1b` | 1B | ~15 tok/s | Ultra-compact |
| `minicpm-2b` | 2B | ~10 tok/s | Small but capable |

### Baichuan2 (Chinese)
| Model ID | Size | NPU speed | Notes |
|----------|------|-----------|-------|
| `baichuan2-7b` | 7B | ~3 tok/s | Chinese-focused |

Load more than one:
```powershell
.\start_server.bat --models "qwen2.5-3b,llama3.2-1b,minicpm-2b"
```

First run downloads and compiles each model (1–3 min); after that they load from cache.

## NPU vs CPU/GPU

Typical power and efficiency:

| Metric | NPU | CPU | iGPU |
|--------|-----|-----|------|
| Power draw | ~5–10W | 15–45W | 20–35W |
| Battery life | Hours | ~1 hour | ~2 hours |
| Best for | Efficiency, background tasks | Fallback | Max performance, larger models |

Platform compute (INT8 TOPS):

| Architecture | NPU TOPS | GPU TOPS | CPU TOPS | Total |
|--------------|----------|----------|----------|-------|
| Core Ultra Series 1 (Meteor Lake), e.g. Core Ultra 9 185H | ~11 | ~18 | ~5 | ~34 |
| Core Ultra Series 2 (Arrow Lake) | ~13 | ~18 | ~5 | ~36 |
| Core Ultra Series 2 (Lunar Lake) | 48 | ~67 | ~5 | ~120 |

## Configuration

### Environment variables

| Variable | Value | Description |
|----------|-------|-------------|
| `IPEX_LLM_NPU_MTL` | `1` | Required for Meteor Lake (Core Ultra Series 1) |
| `NPU_CONDA_ENV` | env name | Conda env that `start_server.bat` activates |
| `NPU_ALLOW_UNSUPPORTED` | `1` | Continue past preflight failures with warnings |
| `NPU_SKIP_DRIVER_CHECK` | `1` | Skip only the NPU driver version check |
| `NPU_SKIP_PREFLIGHT` | `1` | Skip all startup hardware checks |
| `HF_HOME` | path | HuggingFace cache directory |
| `REQUESTS_CA_BUNDLE` | path | Custom CA bundle for HTTPS downloads |
| `SSL_CERT_FILE` | path | Alternate CA bundle pointer |
| `PORT` | `8001` | Server port |

### Compatibility data

`setup/compatibility.json` is the repo's compatibility policy:

- `blockedBiosVersions` — hard-blocks specific BIOS/firmware versions.
- `knownValidatedCombos` — reported-good CPU / BIOS / driver combinations.
- `knownProblemCombos` — reported-bad ones (errors or warnings).

Add only combinations that came from a real machine report or direct validation — keep speculation
out of this file.

### Processor-specific settings

| Processor series | Setting |
|------------------|---------|
| Core Ultra Series 1 (Meteor Lake) | `IPEX_LLM_NPU_MTL=1` |
| Core Ultra Series 2 (Arrow Lake) | none required |
| Core Ultra (Lunar Lake) | none required |

## Troubleshooting

### NPU not detected
1. Device Manager → Neural processors → Intel(R) AI Boost.
2. Update the NPU driver.
3. Make sure `IPEX_LLM_NPU_MTL=1` is set on Meteor Lake.
4. Run `.\setup\00_hardware_preflight.ps1` to see the exact failure.
5. If a BIOS release is known-bad for your machine, add it to `setup/compatibility.json` under
   `blockedBiosVersions`.
6. Run `.\start_server.bat --diagnose` and attach the JSON when you report an issue.

### Generation hangs
The first generation takes 1–3 minutes (NPU warmup); later ones are ~1 second.

### Port already in use
```powershell
Get-Process python* | Stop-Process -Force
```

### `.env` encoding error (`ValueError: embedded null character`)
Happens when `.env` was saved as UTF-16 (PowerShell's default `>` redirect). Recreate it as UTF-8:
```powershell
'HF_TOKEN=hf_your_token_here' | Out-File -FilePath .env -Encoding utf8
```
Or open it in Notepad → **File > Save As** → Encoding: UTF-8.

### HuggingFace TLS error (`SSLCertVerificationError`)
The setup installs `python-certifi-win32`, which lets Python use the Windows cert store for HF
downloads. If your environment predates that:
1. Re-run `.\setup.bat --skip-hf-token`.
2. Or `conda run -n ipex-npu pip install python-certifi-win32`.
3. If your org provides a custom CA bundle, set `REQUESTS_CA_BUNDLE` or `SSL_CERT_FILE` to that
   `.pem` before starting.

## Model storage

| Location | Contents | Path |
|----------|----------|------|
| HuggingFace cache | Original downloaded models | `%USERPROFILE%\.cache\huggingface\hub\` |
| NPU cache | Compiled NPU-optimized models | `intel-npu-llm\npu_model_cache\` |

The built-in UI shows total model disk usage live in the header.

Approximate space:

| Model size | HF cache | NPU cache | Total |
|------------|----------|-----------|-------|
| 1–2B | ~2–4 GB | ~1–2 GB | ~3–6 GB |
| 3–4B | ~6–8 GB | ~2–4 GB | ~8–12 GB |
| 7–8B | ~14–16 GB | ~4–8 GB | ~18–24 GB |

### Cache management

Check usage:
```powershell
"{0:N2} GB" -f ((Get-ChildItem -Recurse .\intel-npu-llm\npu_model_cache\ -ErrorAction SilentlyContinue | Measure-Object -Property Length -Sum).Sum / 1GB)
"{0:N2} GB" -f ((Get-ChildItem -Recurse "$env:USERPROFILE\.cache\huggingface\hub\" -ErrorAction SilentlyContinue | Measure-Object -Property Length -Sum).Sum / 1GB)
```

Clear the NPU cache (keeps downloads; recompiles on next run):
```powershell
Remove-Item -Recurse -Force .\intel-npu-llm\npu_model_cache\
```

Clear one model's NPU cache:
```powershell
Remove-Item -Recurse -Force ".\intel-npu-llm\npu_model_cache\Qwen_Qwen2.5-7B-Instruct\"
Get-ChildItem .\intel-npu-llm\npu_model_cache\
```

Clear the HuggingFace cache (forces a re-download — only if you need the space):
```powershell
Remove-Item -Recurse -Force "$env:USERPROFILE\.cache\huggingface\hub\"
```

Clear everything:
```powershell
Remove-Item -Recurse -Force .\intel-npu-llm\npu_model_cache\ -ErrorAction SilentlyContinue
Remove-Item -Recurse -Force "$env:USERPROFILE\.cache\huggingface\hub\" -ErrorAction SilentlyContinue
Write-Host "All model caches cleared. Models will re-download and recompile on next run."
```

Custom HuggingFace location (handy on a small C: drive):
```
HF_HOME=D:\models\huggingface
```
The NPU cache stays at `intel-npu-llm\npu_model_cache\`.

## Project structure

```
npu-windows/
├── start_server.bat             # One-click startup with auto CPU detection
├── QUICKSTART.md                # 5-minute getting-started guide
├── README.md                    # This file
└── intel-npu-llm/
    ├── npu_server.py            # The NPU-accelerated LLM server (FastAPI)
    ├── index.html               # Built-in dark-mode chat UI
    ├── models.json              # Model registry (add your own here)
    ├── docker-compose.yml       # Open WebUI frontend (optional)
    ├── requirements.txt         # Python dependencies
    ├── .env.example             # Environment variable template
    └── npu_model_cache/         # Compiled NPU models (created on first run)
```

## License

MIT
