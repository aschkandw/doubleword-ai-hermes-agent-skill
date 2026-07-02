# Doubleword CLI Recipes

Use this reference when executing Doubleword commands, validating payloads,
submitting jobs, retrieving results, resuming interrupted downloads, or using
the OpenAI-compatible SDK.

## Readiness Checks

Confirm the API key is present without printing it:

```bash
test -n "$DOUBLEWORD_API_KEY"
```

Verify the authenticated CLI identity:

```bash
dw whoami
```

If `dw whoami` fails, do not upload data. Check whether the Doubleword CLI is
installed and whether `DOUBLEWORD_API_KEY` is available in the shell context.

## JSONL Validation

Validate every generated batch payload before upload:

```bash
dw files validate path/to/dataset.jsonl
dw files stats path/to/dataset.jsonl
```

Fix validation errors locally and rerun validation before submitting.

## Mode Commands

Realtime, for immediate single-request use:

```bash
dw realtime <model>
```

Async, for same-session background jobs:

```bash
dw stream path/to/dataset.jsonl --completion-window 1h
```

Batch, for lowest-cost large jobs:

```bash
dw stream path/to/dataset.jsonl --completion-window 24h
```

## Explicit Batch Creation

Use explicit file upload and batch creation when you need to capture file IDs,
control submission, or shard large workloads.

Upload:

```bash
dw files upload batch.jsonl
```

Create an Async job:

```bash
dw batches create --file <file_id> --completion-window 1h
```

Create a 24h Batch job:

```bash
dw batches create --file <file_id> --completion-window 24h
```

## Status and Results

Check status with a discrete command:

```bash
dw batches get <batch_id>
```

Download results:

```bash
dw batches results <batch_id> -o results.jsonl
```

Check the downloaded output shape:

```bash
dw files stats results.jsonl
```

## Hermes Non-Blocking Rule

For Async and Batch jobs:

1. Submit the job.
2. Capture the batch ID.
3. Report the batch ID and status command to the user.
4. Stop active waiting.
5. Run `dw batches get <batch_id>` later only when the workflow resumes or the
   user asks for status.

Do not use loops such as:

```bash
while true; do
  dw batches get <batch_id>
  sleep 60
done
```

## Resuming Interrupted Downloads

If a large result download breaks mid-stream, inspect the partial file and
resume from the last fully processed line. When the CLI cannot resume directly,
use the files content endpoint with an offset:

```bash
curl -G "https://api.doubleword.ai/v1/files/<output_file_id>/content" \
  -H "Authorization: Bearer $DOUBLEWORD_API_KEY" \
  --data-urlencode "offset=<last_processed_line>"
```

Do not run commands in a way that expands and logs the live token.

## OpenAI SDK Fallback

Use the SDK only when it is clearer than CLI commands for the pipeline. For bulk
work, prefer validated JSONL plus `dw` batch commands.

```python
import os
from openai import OpenAI

client = OpenAI(
    api_key=os.environ["DOUBLEWORD_API_KEY"],
    base_url="https://api.doubleword.ai/v1",
)
```

Keep SDK credentials in environment variables. Do not hard-code or print them.
