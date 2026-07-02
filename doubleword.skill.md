---
name: doubleword
description: Orchestrate cost-aware LLM inference across realtime, async, and batch tiers, and run local data-processing pipelines through the Doubleword CLI.
required_env:
  - DOUBLEWORD_API_KEY
compatibility:
  - openclaw
  - hermes
---

# Doubleword Inference Agent Skill

Use this skill when a user asks the agent to run LLM generation, extraction,
classification, OCR, embedding, evaluation, or dataset transformation work on
Doubleword (`https://api.doubleword.ai/v1`).

Prefer the native `dw` CLI for file and batch workflows. Use the OpenAI Python
SDK only for code paths that already need programmatic request construction or
streaming control.

## Safety and Setup Checks

1. Confirm the API key is available before any remote call:

   ```bash
   test -n "$DOUBLEWORD_API_KEY"
   ```

2. Verify the local CLI identity before uploading data:

   ```bash
   dw whoami
   ```

   If this fails, do not upload files. Re-check the shell environment and the
   Doubleword CLI installation first.

3. Never print API keys, authorization headers, or full signed URLs in user
   messages or logs.

4. Treat uploaded datasets as potentially sensitive. Do not upload hidden files,
   unrelated repository files, secrets, credentials, `.env` files, or raw user
   data that is outside the requested task.

## Mode Selection

Choose the slowest tier that still satisfies the user's stated urgency.

| Mode | CLI pattern | Use when | Agent behavior |
| --- | --- | --- | --- |
| Realtime | `dw realtime <model>` | Single prompt, interactive lookup, small ad hoc request, or explicit immediate result requirement. | Run synchronously, return the result, and include the model used. |
| Async | `dw stream <file> --completion-window 1h` or `dw batches create --file <file_id> --completion-window 1h` | Background workflow needed in the current session, medium datasets, or roughly 100+ requests where waiting is acceptable. | Submit the job, capture the batch ID, report it, and check with discrete status commands only when useful or requested. |
| Batch | `dw stream <file> --completion-window 24h` or `dw batches create --file <file_id> --completion-window 24h` | Large datasets, evals, overnight jobs, lowest-cost processing, or "process this large file" with no deadline. | Submit the job, capture the batch ID, explain that 24h batch is the lowest-cost tier, and stop unless the user asks for follow-up checks. |

Default to Batch for large datasets when the user gives no deadline. If the user
cares about completion during the current working session, use Async. If the
user asks for immediate interaction, use Realtime.

### Non-Blocking Scheduling Protocol

Hermes agents must not idle in sleep loops. For Async or Batch jobs:

1. Submit the job.
2. Capture and report the batch ID.
3. Exit the active wait loop.
4. Check status later with a discrete command, only when the workflow resumes or
   the user asks.

Use:

```bash
dw batches get <batch_id>
```

Do not run `while ... sleep ...` polling loops.

## Request Preparation

1. Generate JSONL payloads in the format required by the selected Doubleword
   endpoint or CLI command.
2. Keep batch files under Doubleword limits: 200 MB and 50,000 requests. Split
   larger workloads into numbered shard files.
3. Include stable request identifiers in each row when possible so results can
   be joined back to inputs.
4. Validate every generated JSONL file before upload:

   ```bash
   dw files validate path/to/dataset.jsonl
   dw files stats path/to/dataset.jsonl
   ```

5. If validation fails, fix the local payload and validate again. Do not upload
   invalid files.

## Submission and Retrieval Commands

Explicit batch creation:

```bash
dw files upload batch.jsonl
dw batches create --file <file_id> --completion-window 1h
dw batches create --file <file_id> --completion-window 24h
```

Status check:

```bash
dw batches get <batch_id>
```

Download results:

```bash
dw batches results <batch_id> -o results.jsonl
```

After downloading results, validate the output file shape before using it in a
pipeline:

```bash
dw files stats results.jsonl
```

## Resuming Interrupted Downloads

If a large result download fails after partial progress, inspect the partial
file and resume from the last fully processed line. When the CLI cannot resume
directly, use the files content endpoint with an offset:

```bash
curl -G "https://api.doubleword.ai/v1/files/<output_file_id>/content" \
  -H "Authorization: Bearer $DOUBLEWORD_API_KEY" \
  --data-urlencode "offset=<last_processed_line>"
```

Do not expose this curl command with a live token expanded in logs or user
messages.

## Model Selection

Respect an explicit user model choice unless it is incompatible with the task.
If the user does not specify a model:

- choose the cheapest model that is likely to satisfy the task;
- use larger or higher-quality models for ambiguous reasoning, long-form
  generation, difficult extraction, or high-stakes evals;
- use specialized OCR or embedding models for those task types;
- mention the selected model and tier in the response;
- if current pricing is critical, verify pricing with Doubleword documentation
  or CLI output before submitting expensive work.

### Text Generation Models

Prices are per 1M tokens, input / output. Realtime is available for most models
at standard rates; use Doubleword's current pricing source for exact realtime
costs.

| Model | Async, 1h | Batch, 24h |
| --- | --- | --- |
| DeepSeek-V4-Pro | $1.31 / $2.75 | $1.05 / $2.20 |
| DeepSeek-V4-Flash | $0.10 / $0.20 | $0.07 / $0.14 |
| Kimi-K2.6 | $0.70 / $3.00 | $0.45 / $2.00 |
| GLM-5.2-FP8 | $1.05 / $3.30 | $0.70 / $2.20 |
| GLM-5.1-FP8 | $1.05 / $3.30 | $0.70 / $2.20 |
| Qwen3.5-397B-A17B | $0.30 / $1.80 | $0.15 / $1.20 |
| Qwen3.6-35B-A3B-FP8 | $0.07 / $0.30 | $0.05 / $0.20 |
| Qwen3.5-35B-A3B-FP8 | $0.07 / $0.30 | $0.05 / $0.20 |
| Qwen3.5-9B | $0.04 / $0.35 | $0.03 / $0.29 |
| Qwen3.5-4B | $0.05 / $0.08 | $0.04 / $0.06 |
| Gemma-4-31B | $0.11 / $0.30 | $0.07 / $0.20 |
| Nemotron-3-Ultra-550B-A55B | $0.37 / $1.87 | $0.25 / $1.25 |
| Nemotron-3-Super-120B-A12B | $0.23 / $0.56 | $0.15 / $0.38 |
| GPT-OSS-20B | $0.03 / $0.20 | $0.02 / $0.15 |
| Qwen3-VL-235B-A22B | $0.15 / $0.55 | $0.10 / $0.40 |
| Qwen3-VL-30B-A3B | $0.07 / $0.30 | $0.05 / $0.20 |
| Qwen3-14B-FP8 | $0.03 / $0.30 | $0.02 / $0.20 |

### Specialized Models

| Task | Model | Async | Batch |
| --- | --- | --- | --- |
| OCR | DeepSeek-OCR-2 | $0.08 / $0.08 | $0.05 / $0.05 |
| OCR | olmOCR-2-7B | $0.15 / $0.15 | $0.10 / $0.10 |
| OCR | LightOnOCR-2-1B | $0.08 / $0.08 | $0.05 / $0.05 |
| Embeddings | Qwen3-Embedding-8B | $0.03 input | $0.02 input |

## OpenAI SDK Fallback

When using the OpenAI Python SDK, configure it with Doubleword's base URL and
the existing API key:

```python
from openai import OpenAI

client = OpenAI(
    api_key=os.environ["DOUBLEWORD_API_KEY"],
    base_url="https://api.doubleword.ai/v1",
)
```

Use SDK calls only when they are clearer than CLI commands for the specific
pipeline. For bulk work, still prefer validated JSONL plus `dw` batch commands.

## User Communication

For every submitted Async or Batch job, tell the user:

- the selected mode and model;
- the batch ID;
- the source file or shard count;
- the expected completion window;
- the command to check status;
- where results will be written when retrieved.

For Realtime jobs, return the result and note the selected model. For Batch
jobs, explicitly state that the 24h window was selected for cost savings when
that was the reason.
