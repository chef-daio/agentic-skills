---
name: meme-image-suite
description: General-purpose static meme generation for any agent. Supports classic template memes and original AI meme images. Image-only workflow, no videos.
triggers:
  - create meme image
  - generate meme
  - template meme
  - ai meme image
  - make a meme
  - static meme
version: 1
---

# Meme Image Suite (Static Only)

Use this skill when an agent needs to generate meme images only.

Supported modes:
1) template mode (classic meme templates)
2) ai mode (original generated meme art)

This skill intentionally excludes video generation.

## Mode selection

- Use template mode for classic formats and reliable caption text.
- Use ai mode for custom visuals and unique scenes.
- If ai mode is unavailable in the host runtime, fallback to template mode.

## Input contract

Required:
- `mode`: `template` or `ai`
- `output_path`: local path to save PNG

Template mode:
- `template_id` (example: `drake`, `distracted`, `fry`)
- `captions` (array of caption lines)

AI mode:
- `prompt` (natural language prompt)
- optional style hints (cartoon, surreal, photorealistic)

## Output contract

Return:
- `ok` (bool)
- `path` (string)
- `mode` (string)
- `source` (template id or model name)
- `bytes` (int)
- `error` (string)

## Template mode implementation

Use Memegen API:
- list templates: `https://api.memegen.link/templates`
- generate image: `https://api.memegen.link/images` (POST)

Python reference:

```python
import os
import requests


def generate_template_meme(template_id, captions, output_path="/tmp/meme.png", width=800):
    resp = requests.post(
        "https://api.memegen.link/images",
        json={"template_id": template_id, "text": captions, "extension": "png"},
        timeout=30,
    )
    if resp.status_code != 200:
        return {"ok": False, "error": f"template create failed: {resp.status_code}"}

    meme_url = resp.json().get("url")
    if not meme_url:
        return {"ok": False, "error": "template API returned no url"}

    sep = "&" if "?" in meme_url else "?"
    final_url = f"{meme_url}{sep}width={width}"

    img = requests.get(final_url, timeout=30)
    if img.status_code != 200:
        return {"ok": False, "error": f"template download failed: {img.status_code}"}

    os.makedirs(os.path.dirname(output_path) or ".", exist_ok=True)
    with open(output_path, "wb") as f:
        f.write(img.content)

    return {
        "ok": True,
        "path": output_path,
        "mode": "template",
        "source": f"template:{template_id}",
        "bytes": len(img.content),
        "error": "",
    }
```

## AI mode implementation

Use a preconfigured image-generation client provided by the host runtime.
Do not hardcode secrets inside the skill.

Python reference:

```python
import os


def generate_ai_meme(prompt, output_path, image_client, model="default-image-model"):
    result = image_client.generate_image(prompt=prompt, model=model)
    image_bytes = result["image_bytes"]

    os.makedirs(os.path.dirname(output_path) or ".", exist_ok=True)
    with open(output_path, "wb") as f:
        f.write(image_bytes)

    return {
        "ok": True,
        "path": output_path,
        "mode": "ai",
        "source": f"model:{model}",
        "bytes": len(image_bytes),
        "error": "",
    }
```

## Unified wrapper

```python
def generate_meme(mode, output_path="/tmp/meme.png", template_id=None, captions=None, prompt=None, image_client=None, model="default-image-model"):
    if mode == "template":
        if not template_id or not captions:
            return {"ok": False, "error": "template mode requires template_id and captions"}
        return generate_template_meme(template_id, captions, output_path)

    if mode == "ai":
        if not prompt:
            return {"ok": False, "error": "ai mode requires prompt"}
        if image_client is None:
            return {"ok": False, "error": "ai client unavailable in runtime"}
        return generate_ai_meme(prompt, output_path, image_client=image_client, model=model)

    return {"ok": False, "error": f"unsupported mode: {mode}"}
```

## Safeguards

- Validate file exists and size > 0 before returning success.
- If ai mode fails, retry once then fallback to template mode when possible.
- Keep captions concise for readability.

## Non-goals

- no video generation
- no audio generation
- no editing pipeline
