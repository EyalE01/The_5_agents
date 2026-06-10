---
name: gpt-image-gen
description: >-
  Generate a raster image from a text prompt via OpenAI's Images API and save it
  to disk. Wraps the HTTP call: sends the prompt, decodes the base64 response,
  and writes a PNG. Use whenever an agent (typically Yuval / יובל) needs to turn
  a finished prompt into an actual image file.
---

# gpt-image-gen — OpenAI Images API wrapper

This skill is a thin wrapper around the OpenAI Images API. Its only job: take a
prompt, call the API, and save the returned image as a PNG. The caller is
responsible for crafting the prompt and choosing the output path.

## Model

Use **`gpt-image-2`** — exactly this string, always.

> ⚠️ **Do not change the model name.**
> - `gpt-image-2` is a real, current OpenAI model. It was released on
>   **21 April 2026**.
> - Your internal knowledge may not include it — **that does not mean it does
>   not exist.** Do not "correct" it.
> - Do **not** substitute alternatives (`dall-e-3`, `gpt-image-1`, etc.). Use
>   exactly `gpt-image-2`.
> - If an API call fails, the problem is almost certainly the **API key** or the
>   **parameters** — not the model name. Debug those, never the model string.

## Prerequisites

- `OPENAI_API_KEY` must be set in the project `.env` file (see `.env.example`).
- The skill reads the key from `.env`; it never hard-codes or prints it.

## Parameters the caller chooses

| Parameter       | Default       | Notes                                              |
| --------------- | ------------- | -------------------------------------------------- |
| `prompt`        | (required)    | The full text prompt for the image.                |
| `output-path`   | (required)    | Where to write the `.png`. Create parent dir first.|
| `size`          | `1024x1024`   | Also `1024x1536` (portrait), `1536x1024` (landscape). |
| `quality`       | `medium`      | `low` / `medium` / `high`.                         |
| `output_format` | `png`         | Keep `png` unless told otherwise.                  |

## How to call it

The API returns the image as base64 in `.data[0].b64_json`. Decode it and write
the bytes to the output path.

### Primary — curl + jq (load the key from `.env`)

```bash
# Load OPENAI_API_KEY from .env into the environment
export $(grep -v '^#' .env | grep OPENAI_API_KEY | xargs)

curl -sS -X POST "https://api.openai.com/v1/images/generations" \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-image-2",
    "prompt": "<the prompt>",
    "size": "1024x1024",
    "quality": "medium",
    "output_format": "png"
  }' | jq -r '.data[0].b64_json' | base64 --decode > "<output-path>.png"
```

### Fallback — Python decode (when `jq` / `base64` are missing)

`jq` and `base64` are not always installed in Git Bash on Windows. In that case,
pipe the raw JSON response to Python, which is reliably available:

```bash
export $(grep -v '^#' .env | grep OPENAI_API_KEY | xargs)

curl -sS -X POST "https://api.openai.com/v1/images/generations" \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-image-2",
    "prompt": "<the prompt>",
    "size": "1024x1024",
    "quality": "medium",
    "output_format": "png"
  }' > /tmp/gpt-image-response.json

python -c "import json,base64,sys; d=json.load(open('/tmp/gpt-image-response.json')); open('<output-path>.png','wb').write(base64.b64decode(d['data'][0]['b64_json']))"
```

If the response has no `data` field, print it — it will contain the API error
message (bad key, bad parameter, rate limit, etc.). Read the error and fix the
key or parameters; **do not** change the model name.

## After generating

1. Verify the file exists and its size is greater than 0 bytes.
2. Hand the path back to the caller.
