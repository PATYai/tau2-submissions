# Granite 4.1-8B Testing Methodology

## Overview

This document describes the methodology used to evaluate `ibm-granite/granite-4.1-8b` on the τ² benchmark.

Granite 4.1-8B was tested as the agent model. The user simulator was `gpt-5.1`.

## Model Under Test

| Field | Value |
|---|---|
| Agent model | `ibm-granite/granite-4.1-8b` |
| Served model name | `granite` |
| Serving backend | `vLLM` |
| API mode | OpenAI-compatible chat completions |
| Hardware | NVIDIA RTX 3090 24GB |
| Runtime environment | Ubuntu distrobox |
| Precision | `bfloat16` |
| Max model context | `16384` |
| GPU memory utilization | `0.85` |
| Tool-call parser | `granite4` |

## vLLM Launch Command

```bash
vllm serve ibm-granite/granite-4.1-8b \
  --served-model-name granite \
  --host 0.0.0.0 \
  --port 8000 \
  --dtype bfloat16 \
  --gpu-memory-utilization 0.85 \
  --max-model-len 16384 \
  --enable-auto-tool-choice \
  --tool-call-parser granite4
````

## Tool-Calling Validation

Before running τ², the vLLM endpoint was tested to confirm that tool calls were returned as structured OpenAI-compatible `tool_calls`.

```bash
curl http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer local" \
  -d '{
    "model": "granite",
    "messages": [
      {
        "role": "user",
        "content": "Look up reservation EHGLP3."
      }
    ],
    "tools": [
      {
        "type": "function",
        "function": {
          "name": "get_reservation_details",
          "description": "Get details for an airline reservation.",
          "parameters": {
            "type": "object",
            "properties": {
              "reservation_id": {
                "type": "string",
                "description": "The reservation ID."
              }
            },
            "required": ["reservation_id"]
          }
        }
      }
    ],
    "tool_choice": "auto",
    "temperature": 0,
    "max_tokens": 256
  }'
```

Expected validation result:

```json
{
  "choices": [
    {
      "message": {
        "content": null,
        "tool_calls": [
          {
            "type": "function",
            "function": {
              "name": "get_reservation_details",
              "arguments": "{\"reservation_id\": \"EHGLP3\"}"
            }
          }
        ]
      },
      "finish_reason": "tool_calls"
    }
  ]
}
```

## Benchmark Configuration

| Field                | Value                      |
| -------------------- | -------------------------- |
| Benchmark            | `tau2-bench`               |
| Agent                | `llm_agent`                |
| Agent model string   | `openai/granite`           |
| Agent endpoint       | `http://localhost:8000/v1` |
| Agent API key        | `local`                    |
| User simulator       | `user_simulator`           |
| User simulator model | `gpt-5.1`                  |
| Trials per domain    | `4`                        |

## Agent Configuration

The local Granite endpoint was passed through `--agent-llm-args`.

```bash
--agent-llm openai/granite \
--agent-llm-args '{"api_base":"http://localhost:8000/v1","api_key":"local"}' \
--user-llm gpt-5.1
```

## Domains

The submission run targeted:

```text
airline
retail
telecom
banking_knowledge
```

For `banking_knowledge`, the retrieval configuration was:

```text
bm25
```

## Run Commands

### Airline

```bash
uv run tau2 run \
  --domain airline \
  --agent-llm openai/granite \
  --agent-llm-args '{"api_base":"http://localhost:8000/v1","api_key":"local"}' \
  --user-llm gpt-5.1 \
  --num-trials 4 \
  --save-to granite41_16k_airline
```

### Retail

```bash
uv run tau2 run \
  --domain retail \
  --agent-llm openai/granite \
  --agent-llm-args '{"api_base":"http://localhost:8000/v1","api_key":"local"}' \
  --user-llm gpt-5.1 \
  --num-trials 4 \
  --save-to granite41_16k_retail
```

### Telecom

```bash
uv run tau2 run \
  --domain telecom \
  --agent-llm openai/granite \
  --agent-llm-args '{"api_base":"http://localhost:8000/v1","api_key":"local"}' \
  --user-llm gpt-5.1 \
  --num-trials 4 \
  --save-to granite41_16k_telecom
```

### Banking Knowledge

```bash
uv run tau2 run \
  --domain banking_knowledge \
  --retrieval-config bm25 \
  --agent-llm openai/granite \
  --agent-llm-args '{"api_base":"http://localhost:8000/v1","api_key":"local"}' \
  --user-llm gpt-5.1 \
  --num-trials 4 \
  --save-to granite41_16k_banking_knowledge_bm25
```

## Submission Preparation

Use simulation outputs under `data/simulations`, not terminal log files.

```bash
uv run tau2 submit prepare \
  data/simulations/granite41_16k_airline \
  data/simulations/granite41_16k_retail \
  data/simulations/granite41_16k_telecom \
  data/simulations/granite41_16k_banking_knowledge_bm25 \
  --output ~/ai-work/tau2-runs/granite-4.1-8b-16k/granite-4.1-8b-16k
```

Validate:

```bash
uv run tau2 submit validate \
  ~/ai-work/tau2-runs/granite-4.1-8b-16k/granite-4.1-8b-16k \
  --mode public
```

## Reproducibility Artifacts

```bash
git rev-parse HEAD > granite_submit_git_sha.txt # 7483cc60e4957fb2cf834c05a41059a715ee287c
git status --short > granite_submit_git_status.txt # ?? granite_submit_git_sha.txt
vllm --version > granite_submit_vllm_version.txt # 0.20.1
nvidia-smi > granite_submit_nvidia_smi.txt
Mon May 18 19:36:58 2026
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 580.142                Driver Version: 580.142        CUDA Version: 13.0     |
+-----------------------------------------+------------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |           Memory-Usage | GPU-Util  Compute M. |
|                                         |                        |               MIG M. |
|=========================================+========================+======================|
|   0  NVIDIA GeForce RTX 3090        Off |   00000000:0B:00.0 Off |                  N/A |
|100%   67C    P2            348W /  350W |   20843MiB /  24576MiB |    100%      Default |
|                                         |                        |                  N/A |
+-----------------------------------------+------------------------+----------------------+
```

## Notes

* Smoke tests using `--num-tasks` were excluded from submission results.
* Submission runs used full domain task sets.
* No τ² prompts, task data, tools, domains, evaluator logic, or agent orchestration were modified.
* Granite 4.1-8B was served locally through vLLM.
* `granite4` was required for structured OpenAI-compatible tool calls.
* The user simulator was `gpt-5.1`.
