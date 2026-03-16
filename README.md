# TrashClaw

A local tool-use coding agent that runs on a 2013 Mac Pro ("trashcan"). No cloud, no API keys — just llama.cpp and a single Python file with zero dependencies.

## What it does

TrashClaw is a terminal agent that reads files, edits code, runs commands, and searches codebases using a local LLM. It follows the tool-use pattern: the model decides what tools to call, sees the results, and iterates until the task is done.

```
trashclaw src> add error handling to the parse function

  [read] src/parser.py
  [edit] src/parser.py
  [run] python3 -m pytest tests/ [y/N] y

  Added try/except to parse(). All 12 tests pass.
```

It works with any OpenAI-compatible endpoint (llama.cpp server, Ollama, vLLM, etc). The interesting part is the hardware we're running it on.

## The hardware

We're running this on a 2013 Mac Pro — the cylinder everyone called a trash can.

- Xeon E5-1650 v2 (6c/12t, 3.5GHz, Ivy Bridge)
- 2x AMD FirePro D500 (3GB VRAM each)
- 16GB DDR3, macOS Monterey 12.7.6
- ~$150 on eBay

With Qwen2.5-3B (Q4_K_M, 2GB) it does 15.6 tokens/sec generation on CPU. That's 2+ seconds for a typical agent response — usable for real work.

### Metal on discrete AMD GPUs

The FirePro D500s support Metal (GPUFamily macOS 2), but llama.cpp's Metal backend crashed on load because it assumes unified memory. The `set_tensor` and `get_tensor` functions use `newBufferWithBytesNoCopy` with `StorageModeShared`, which returns nil on discrete GPUs.

The [fix](https://github.com/ggml-org/llama.cpp/pull/20615) is three hunks: use `newBufferWithBytes` + `StorageModeManaged` when `has_unified_memory` is false, plus a `memcpy` on the get path. Prompt processing is 16% faster with Metal on the 3B model; generation is slower due to PCIe round-trips, so CPU wins there.

| Model (Qwen 3B Q4) | pp128 | tg32 |
|---------------------|-------|------|
| CPU-only (12 threads) | 20.3 t/s | **15.5 t/s** |
| Metal (ngl=99) | **23.4 t/s** | 2.9 t/s |

This should work on any Mac with a discrete AMD GPU (2013-2019 Mac Pro, iMac, MacBook Pro with Radeon).

## Tools

| Tool | Description |
|------|-------------|
| `read_file` | Read files with optional line range |
| `write_file` | Create new files |
| `edit_file` | Find-and-replace (must match uniquely) |
| `run_command` | Shell execution with approval prompt |
| `search_files` | Regex search across files |
| `find_files` | Glob pattern matching |
| `list_dir` | Directory listing |
| `think` | Internal reasoning (no side effects) |

The agent loop runs up to 15 tool rounds per request. The LLM decides which tools to call based on the system prompt and conversation context.

### Model compatibility

Tool calls are parsed three ways, so it works with most models:

1. Native function calling (Qwen 2.5, Llama 3.1+, Mistral)
2. `<tool_call>` XML tags
3. JSON in code blocks or bare JSON

Qwen2.5-3B-Instruct is the sweet spot for this hardware — small enough to fit, capable enough for tool use.

## Setup

```bash
# Build llama.cpp
git clone --depth 1 https://github.com/ggml-org/llama.cpp.git
cd llama.cpp && mkdir build && cd build
cmake .. -DCMAKE_BUILD_TYPE=Release && make -j6

# Get a model
mkdir -p ~/models
curl -L -o ~/models/qwen2.5-3b-instruct-q4.gguf \
  "https://huggingface.co/Qwen/Qwen2.5-3B-Instruct-GGUF/resolve/main/qwen2.5-3b-instruct-q4_k_m.gguf"

# Start the server
./bin/llama-server --host 0.0.0.0 --port 8080 \
  -m ~/models/qwen2.5-3b-instruct-q4.gguf -t 12 -c 4096

# Run the agent (separate terminal)
python3 trashclaw.py
```

No pip install. No venv. Just Python 3.7+ stdlib.

## Configuration

| Variable | Default | What |
|----------|---------|------|
| `TRASHCLAW_URL` | `http://localhost:8080` | Server endpoint |
| `TRASHCLAW_MAX_ROUNDS` | `15` | Tool rounds per request |
| `TRASHCLAW_AUTO_SHELL` | `0` | Skip command approval if `1` |

```bash
# Point at a remote server
TRASHCLAW_URL=http://192.168.0.50:8080 python3 trashclaw.py

# Work in a specific directory
python3 trashclaw.py --cwd ~/myproject
```

## Limitations

- **3B models are limited.** Complex refactoring or multi-file changes often need manual correction. A 7B+ model would help but needs more RAM or GPU offload.
- **No streaming.** Responses appear all at once after generation completes.
- **Single-GPU only.** llama.cpp uses `MTLCreateSystemDefaultDevice()` — only one of the two D500s is used.
- **Discrete GPU overhead.** Every Metal tensor operation copies across PCIe. For small models, CPU is faster for generation.

## What this isn't

This is not a replacement for Claude Code or Cursor. Those use frontier models with 200B+ parameters and sophisticated context management. TrashClaw is a 3B model on a 12-year-old computer.

What it *is*: proof that local tool-use agents work on hardware most people threw away. The interaction pattern — model calls tools, sees results, iterates — doesn't need GPT-4. It needs a model that can follow a system prompt and output structured tool calls. Qwen 3B does this well enough.

## License

MIT
