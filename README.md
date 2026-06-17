# Atomic Planning

Save tokens using local models on constrained hardware.

A planner-executor pipeline for [OpenCode](https://opencode.ai). DeepSeek plans with **atomic specs (EARS format)** → Qwen 7B executes locally → 3-gate validation before apply. Every token Qwen generates is one token your primary model doesn't bill.

## How It Works

```
You ask for a change
    │
    ▼
DeepSeek reads files + creates atomic plan (EARS specs)
    │
    ▼
Asks you: "Delegate to Qwen local? [✅ Yes | ❌ No]"
    │
    ▼ (if Yes)
purge RAM → Qwen 7B generates code from specs → 3-gate validation (integrity → conformance → syntax) → apply → report savings
```

Each task is **one file, max 200 lines or ~200 words of specs**. Dependencies declared explicitly (`DEPENDS_ON`). Shared files (routes, configs, types) get a dedicated integration task.

## Practical Example

> **User says:** "Add a red delete button to the user profile page"

**DeepSeek reads the file** → produces this atomic plan:

```
## FILE: src/app/components/ProfileForm.tsx

### TASK: Add a red delete button
FILES: [modify: src/app/components/ProfileForm.tsx]

#### EXISTING CODE
```tsx
<Button variant="primary">Save</Button>
```

#### SPECS (EARS format)
- WHEN user clicks THE delete button SHALL call `onDelete()` prop
- THE button SHALL display the text "Delete Account"
- THE button SHALL have style `backgroundColor: "#dc2626"` and `color: "#fff"`
- IF `onDelete` is not provided THE button SHALL NOT render
- SHALL NOT add imports or modify any other component

#### CONSTRAINTS
- Keep existing Button import
- Only modify the return block, nothing else
```

**DeepSeek asks:**
> "¿Delegar a Qwen local? ✅ Sí — tarea pequeña (~30 tokens de output), specs claras, 1 archivo, sin juicio de diseño"

**User says Sí** → `sudo purge` → Qwen generates the code → 3-gate validation passes → applied.

**Report:**
> "Done! ~120 tokens saved locally (~$0.0001)"

## What It Saves

| Metric | DeepSeek V4 Pro | Qwen 7B local |
|---|---|---|
| Cost per 1M output tokens | $0.87 | ~$0.00 (electricity) |
| Typical output per task | ~300 tokens | ~300 tokens (same job) |
| Savings per 100 tasks | ~$26 | — |
| Cumulative session savings | — | Tracked automatically |

## Recommended Stack

### Minimum viable (16 GB RAM)

| Component | macOS | Linux |
|---|---|---|
| **Orchestrator** | OpenCode 0.x+ | OpenCode 0.x+ |
| **Planner model** | DeepSeek V4 Pro (or any OpenCode provider) | Same |
| **Executor model** | Qwen2.5-Coder-7B-Instruct-Q8_0 (~8.1 GB) | Same |
| **Local runner** | Ollama 0.30+ (Metal GPU) | Ollama 0.30+ (ROCm/CUDA/CPU) |
| **RAM** | ≥16 GB unified | ≥16 GB (system + VRAM if GPU) |
| **OS overhead** | ~4-6 GB | ~1-2 GB (more free RAM for Qwen) |

### Better (32 GB or NVIDIA GPU)

| Component | Benefit |
|---|---|
| Qwen3-Coder 14B Q4_K_M (~9 GB) | Better code generation |
| NVIDIA RTX 3060+ (12+ GB VRAM) | ~40-60 tok/s vs ~22 tok/s |
| 32 GB system RAM | Room for larger models + OS |

## Expected Performance

| Hardware | Backend | Qwen 7B Q8_0 | Qwen3-Coder 14B Q4_K_M |
|---|---|---|---|
| Mac Mini M4 (16 GB) | Metal | ~22 tok/s | ~12 tok/s (might swap) |
| Ryzen 7 8845HS + 16 GB | ROCm (iGPU) | ~18-25 tok/s | — |
| Intel + 16 GB + CPU only | CPU | ~8-12 tok/s | — |
| Any + RTX 3060 12 GB | CUDA | ~40-60 tok/s | ~20-30 tok/s |
| Any + RTX 4090 | CUDA | ~80+ tok/s | ~40-60 tok/s |

## Installation

### 1. Install Ollama

**macOS:**
```bash
brew install ollama
# or download from https://ollama.com
```

**Linux:**
```bash
curl -fsSL https://ollama.com/install.sh | sh
```

### 2. Download and configure the executor model

```bash
# Pull Qwen2.5-Coder-7B Q8_0
ollama pull qwen2.5-coder:7b-q8_0

# Create custom model with tuned parameters
ollama create qwen-executor:7b-q8 -f - << 'EOF'
FROM qwen2.5-coder:7b-q8_0
PARAMETER temperature 0.1
PARAMETER num_ctx 16384
PARAMETER stop "```"
EOF
```

### 3. Configure passwordless RAM purge

**macOS (`/etc/sudoers.d/purge`):**
```bash
echo 'yourusername ALL=(ALL) NOPASSWD: /usr/sbin/purge' | sudo tee /etc/sudoers.d/purge
sudo chmod 440 /etc/sudoers.d/purge
```

**Linux (`/etc/sudoers.d/purge`):**
```bash
echo 'yourusername ALL=(ALL) NOPASSWD: /usr/bin/tee' | sudo tee /etc/sudoers.d/purge
sudo chmod 440 /etc/sudoers.d/purge
```

The skill uses `sync && echo 3 | sudo tee /proc/sys/vm/drop_caches` on Linux.

### 4. Set up Ollama environment (recommended)

Add to `~/.zshrc` or `~/.bashrc`:
```bash
export OLLAMA_KEEP_ALIVE=20s      # keep model loaded for 20s after last use
export OLLAMA_FLASH_ATTENTION=1   # faster inference
export OLLAMA_KV_CACHE_TYPE=q8_0  # memory-efficient cache
```

### 5. Copy files to OpenCode config

```bash
cp AGENTS.md ~/.config/opencode/
mkdir -p ~/.config/opencode/agents ~/.config/opencode/skills/qwen-executor
cp agents/executor.md ~/.config/opencode/agents/
cp skills/qwen-executor/SKILL.md ~/.config/opencode/skills/qwen-executor/
```

### 6. Configure opencode.jsonc

Add to `~/.config/opencode/opencode.jsonc`:

```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  "instructions": ["~/.config/opencode/AGENTS.md"],
  "provider": {
    "ollama": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "Ollama (local)",
      "options": {
        "baseURL": "http://localhost:11434/v1"
      },
      "models": {
        "qwen-executor:7b-q8": {
          "name": "Qwen Executor 7B Q8_0"
        }
      }
    }
  },
  "agent": {
    "executor": {
      "mode": "subagent",
      "model": "ollama/qwen-executor:7b-q8",
      "num_ctx": 16384,
      "temperature": 0.1,
      "permission": {
        "edit": "deny",
        "bash": "deny",
        "task": "deny",
        "webfetch": "deny",
        "glob": "deny",
        "grep": "deny",
        "todowrite": "deny",
        "question": "deny"
      }
    }
  }
}
```

Restart OpenCode for changes to take effect.

## Validation Gates

Every executor output is checked before applying:

1. **File integrity** — file exists, non-empty, valid format
2. **Specs conformance** — spot-check 2-3 critical EARS requirements
3. **Syntax check** — code parses cleanly

If a check fails: max 2 retries with specific feedback, then re-plan from scratch.

## Files

| File | Purpose |
|---|---|
| `README.md` | This file |
| `AGENTS.md` | Permanent workflow rules (load via `instructions` in opencode.jsonc) |
| `skills/qwen-executor/SKILL.md` | Delegation protocol, plan format, purge, validation |
| `agents/executor.md` | Subagent definition (no tools, pure code generation) |

## How Is This Different From...?

| Project | Key difference |
|---|---|
| **oh-my-openagent** | 11 agents, 52 hooks, requires Opus/GPT-5.5. Ours is minimal + local-first. |
| **Aider** | Two-model split but natural-language specs. Ours uses EARS structured specs + 3-gate validation. |
| **GitHub Spec-Kit** | Spec-driven toolkit, no executor component. Ours binds specs to a local executor. |
| **SPOQ** | Wave-based dispatch, requires Claude Code. Ours targets OpenCode + 16 GB RAM. |
