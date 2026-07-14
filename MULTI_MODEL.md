# Running Multiple Local Models

This project already supports loading several models at once with `start_server.bat --models`.
This guide covers **budgets** for your hardware and how the new features (auth, context
profiles, per-model trained adapters) interact when you run more than one model.

## Your hardware

- **System RAM:** 32 GB
- **Intel NPU:** ~17 GB addressable, shared between weights + KV cache
- **No discrete GPU** → all training runs on CPU; all inference runs on the NPU

## Memory budget (NPU, sym_int4 weights + KV cache)

Compiled `sym_int4` weights are roughly 0.5 GB per 1B params. KV cache scales with
`max_context_len` × turns. Rough footprints:

| Model | Weights (int4) | +medium context (2048) | Verdict |
|-------|---------------|------------------------|---------|
| 1–2B (qwen1.5-1.8b, llama3.2-1b, minicpm) | ~1 GB | ~1.5 GB | Trivial; run several |
| 3B (qwen2.5-3b, llama3.2-3b, glm-edge-4b) | ~1.8 GB | ~2.5 GB | Comfortable |
| 4B (qwen1.5-4b) | ~2.2 GB | ~3 GB | Fine |
| 7B (qwen2.5-7b, deepseek-7b, llama3-8b) | ~4 GB | ~5 GB | One 7B + a couple small ones fits |

**Rule of thumb:** keep total NPU resident weights ≤ ~12 GB to leave headroom for KV
cache and the OS. On 17 GB NPU that's typically "one 7B + two 3B" or "four small models".

System RAM matters for: (a) CPU training (a 3B LoRA needs ~12 GB fp32, 7B ~24 GB — stay on
3B for training), and (b) HuggingFace download + compile staging.

## Good multi-model combos to start

```bat
:: Quality + speed, both comfortably in budget
.\start_server.bat --models "qwen2.5-3b,llama3.2-1b"

:: Coding + chat
.\start_server.bat --models "qwen2.5-coder,qwen2.5-3b,minicpm-2b"

:: Reasoning + fast
.\start_server.bat --models "deepseek-1.5b,qwen2.5-3b"
```

Pick the model from the UI dropdown per task. The first loaded model is selected by default.

## Context profiles

`models.json` assigns each model a `context_profile` (`small` 1024/512, `medium` 2048/1024,
`large` 4096/2048). The default `qwen1.5-1.8b` keeps explicit `small` limits so its existing
cache isn't invalidated. **These limits are baked into the NPU compile** — changing a
profile deletes/refreshes that model's `npu_model_cache/` entry on next load (the server
detects the mismatch automatically). Larger profiles use more NPU memory per model, so
raise them gradually and watch the disk/RAM chips.

## Concurrency

The NPU is one shared device. The server serializes generations with a **global semaphore**
plus a per-model lock, so even with several models loaded, only one generation runs at a
time and requests queue fairly. You will not see two models generating simultaneously.

## Exposing to your architecture (auth)

When you point Open WebUI / N8N / your own scripts at this server across the network, set an
`API_KEY` so the OpenAI-compatible inference endpoints are protected:

```powershell
$env:API_KEY = 'some-long-secret'
.\start_server.bat --models "qwen2.5-3b,llama3.2-1b"
```

With `API_KEY` set, `/v1/chat/completions`, `/v1/responses`, `/v1/models`, and `/v1/fs/*`
require `Authorization: Bearer <key>` (or `x-api-key`). The built-in UI, `/health`, and
`/v1/system/status` stay public so local use keeps working. Unset the env var to return to
the open `sk-dummy` behavior.

## Per-model trained adapters

Each model can carry its own continually-trained variant. In `models.json`:

```json
"qwen2.5-3b": { "hf_id": "Qwen/Qwen2.5-3B-Instruct", "context_profile": "medium",
                "active_adapter": "v1699..." }
```

`active_adapter` points at `adapters/<model>/<variant>/`. Remove the key to revert to the
base model. See the training section in `README.md` for the full `train/` pipeline.
