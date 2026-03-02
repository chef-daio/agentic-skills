---
name: video-meme
description: Generate short meme videos (5-8 seconds) using Google Veo via the Gemini API. Supports Veo 2 (free tier, no audio) through Veo 3.1 (premium, native audio). Text-to-video and image-to-video. Chains with twitter-post for auto-posting video memes to @chefdaio.
triggers:
  - generate video meme
  - video meme
  - meme video
  - make a video
  - video clip
  - short video
  - veo video
  - animated meme
  - video with audio
  - veo 3
version: 4
---

# Video Meme Skill

Generate short meme video clips (5-8 seconds) using Google Veo via the Gemini API.
Supports multiple tiers: Veo 2 (free, silent) for daily content, Veo 3.1 (paid, with audio) for premium posts.
For static memes, use content-gen/meme-gen (templates) or content-gen/meme-creator (AI images).

## Prerequisites

- `curl`, `python3`, `jq` (pre-installed on macOS/Linux, install jq if missing)
- `ffmpeg` (for text overlays and format conversion)
- Gemini API key stored in `~/.hermes/.env` as `GEMINI_API_KEY`
- Get a key at: https://aistudio.google.com/apikey

## Model Info

| Model | ID | Max Duration | Resolution | Audio | Cost/sec | 8s clip |
|-------|-----|-------------|------------|-------|----------|---------|
| **Veo 2** | `veo-2.0-generate-001` | 8 seconds | 720p | No | ~$0.35 | ~$2.80 |
| **Veo 3.1 Fast** | `veo-3.1-fast-generate-preview` | 8 seconds | 720p-1080p | Yes | ~$0.15 | ~$1.20 |
| **Veo 3.1** | `veo-3.1-generate-preview` | 8 seconds | up to 4K | Yes | ~$0.40 | ~$3.20 |

All models support text-to-video (t2v) and image-to-video (i2v).
Veo 3+ models generate **native audio** (sound effects, ambient noise, music) synchronized with video.
Veo 3.1 additionally supports `resolution` param: `"720p"`, `"1080p"`, `"4k"`.

**Free tier:** Veo 2 has ~10 free generations/day at 720p (undocumented but confirmed March 2026).
Veo 3.1 models have NO free tier — all usage is billed.

**Recommendation:**
- Daily meme content → `veo-2.0-generate-001` (free, silent, 720p)
- Launches & bangers → `veo-3.1-fast-generate-preview` ($1.20/clip, audio, 1080p — best value)
- Premium showcase → `veo-3.1-generate-preview` ($3.20/clip, audio, up to 4K)

## API Details

- Base URL: `https://generativelanguage.googleapis.com/v1beta`
- Endpoint: `models/{model}:predictLongRunning`
- Auth: `x-goog-api-key` header
- Video generation is async — you submit a job, poll for completion, then download.

## Quick Start: Text-to-Video

### Step 1: Submit generation request

```bash
GEMINI_API_KEY=$(grep GEMINI_API_KEY ~/.hermes/.env | cut -d= -f2)
BASE_URL="https://generativelanguage.googleapis.com/v1beta"

# Pick your tier:
# MODEL="veo-2.0-generate-001"              # Free, no audio, 720p
MODEL="veo-3.1-fast-generate-preview"      # $1.20/clip, audio, 1080p (best value)
# MODEL="veo-3.1-generate-preview"          # $3.20/clip, audio, up to 4K

OPERATION=$(curl -s "${BASE_URL}/models/${MODEL}:predictLongRunning" \
  -H "x-goog-api-key: $GEMINI_API_KEY" \
  -H "Content-Type: application/json" \
  -X POST \
  -d '{
    "instances": [{
      "prompt": "A cartoon taco wearing sunglasses and a gold chain, riding a rocket through space, crypto coins flying everywhere, meme energy, funny and absurd"
    }],
    "parameters": {
      "aspectRatio": "16:9",
      "personGeneration": "allow_adult",
      "durationSeconds": 8,
      "includeAudio": true
    }
  }' | jq -r '.name')

echo "Operation: $OPERATION"
```

**Note:** `includeAudio` is only supported on Veo 3+ models. Omit it or set `false` for Veo 2.
Disabling audio on Veo 3.1 saves ~33% (e.g. Veo 3.1 Fast drops from $0.15 to ~$0.10/sec).

### Step 2: Poll until done

```bash
while true; do
  STATUS=$(curl -s -H "x-goog-api-key: $GEMINI_API_KEY" "${BASE_URL}/${OPERATION}")
  DONE=$(echo "$STATUS" | jq -r '.done // false')
  if [ "$DONE" = "true" ]; then
    echo "Video ready!"
    echo "$STATUS" > /tmp/veo_response.json
    break
  fi
  echo "Generating... (polling every 15s)"
  sleep 15
done
```

### Step 3: Download the video

```bash
VIDEO_URI=$(jq -r '.response.generateVideoResponse.generatedSamples[0].video.uri' /tmp/veo_response.json)
curl -L -o /tmp/meme_video.mp4 -H "x-goog-api-key: $GEMINI_API_KEY" "$VIDEO_URI"
echo "Saved: /tmp/meme_video.mp4"
```

## Python: Full Generate Function

```python
import requests, json, time, os

def generate_video_meme(prompt, output_path="/tmp/meme_video.mp4",
                        duration=8, aspect_ratio="16:9",
                        model="veo-2.0-generate-001",
                        audio=None,
                        poll_interval=15, max_wait=300):
    """Generate a short meme video using Google Veo.

    Args:
        prompt: Description of the video to generate
        output_path: Where to save the MP4
        duration: 5, 6, or 8 seconds (int)
        aspect_ratio: "16:9" (landscape) or "9:16" (portrait/vertical)
        model: Veo model ID (see Model Info table)
        audio: True/False to enable/disable audio. None = auto (True for Veo 3+, False for Veo 2)
        poll_interval: Seconds between status checks
        max_wait: Max seconds to wait before giving up

    Returns:
        Tuple of (video_path, None) on success or (None, error_message) on failure
    """
    api_key = os.environ.get("GEMINI_API_KEY")
    if not api_key:
        env_path = os.path.expanduser("~/.hermes/.env")
        if os.path.exists(env_path):
            with open(env_path) as f:
                for line in f:
                    if line.startswith("GEMINI_API_KEY="):
                        api_key = line.strip().split("=", 1)[1]
    if not api_key:
        return None, "No GEMINI_API_KEY found"

    base_url = "https://generativelanguage.googleapis.com/v1beta"
    headers = {"x-goog-api-key": api_key, "Content-Type": "application/json"}

    # Auto-detect audio support based on model
    is_veo3_plus = "3.0" in model or "3.1" in model
    include_audio = audio if audio is not None else is_veo3_plus

    # Step 1: Submit generation request
    params = {
        "aspectRatio": aspect_ratio,
        "personGeneration": "allow_adult",
        "durationSeconds": duration
    }
    if is_veo3_plus:
        params["includeAudio"] = include_audio

    payload = {
        "instances": [{"prompt": prompt}],
        "parameters": params
    }

    tier = "premium" if is_veo3_plus else "free"
    audio_str = "with audio" if include_audio else "silent"
    print(f"Using {model} ({tier}, {audio_str})")

    resp = requests.post(
        f"{base_url}/models/{model}:predictLongRunning",
        headers=headers, json=payload
    )
    if resp.status_code != 200:
        return None, f"API error {resp.status_code}: {resp.text[:300]}"

    operation_name = resp.json().get("name")
    if not operation_name:
        return None, f"No operation name in response: {resp.text[:300]}"

    print(f"Submitted. Operation: {operation_name}")

    # Step 2: Poll for completion
    elapsed = 0
    while elapsed < max_wait:
        time.sleep(poll_interval)
        elapsed += poll_interval

        status_resp = requests.get(
            f"{base_url}/{operation_name}",
            headers={"x-goog-api-key": api_key}
        )
        status = status_resp.json()

        if status.get("done"):
            # Check for errors
            if "error" in status:
                return None, f"Generation failed: {json.dumps(status['error'])[:300]}"

            # Step 3: Download video
            try:
                video_uri = status["response"]["generateVideoResponse"]["generatedSamples"][0]["video"]["uri"]
            except (KeyError, IndexError) as e:
                return None, f"Could not extract video URI: {e}. Response: {json.dumps(status)[:300]}"

            dl_resp = requests.get(video_uri, headers={"x-goog-api-key": api_key})
            if dl_resp.status_code != 200:
                return None, f"Download failed: {dl_resp.status_code}"

            os.makedirs(os.path.dirname(output_path) or ".", exist_ok=True)
            with open(output_path, "wb") as f:
                f.write(dl_resp.content)

            size_mb = len(dl_resp.content) / (1024 * 1024)
            print(f"Video saved: {output_path} ({size_mb:.1f} MB)")
            return output_path, None

        print(f"Generating... ({elapsed}s elapsed)")

    return None, f"Timed out after {max_wait}s"
```

## Image-to-Video (animate a still image)

Use an existing meme image as the starting frame and animate it:

```python
import base64

def image_to_video(image_path, prompt="", output_path="/tmp/meme_video.mp4",
                   duration=8, aspect_ratio="16:9",
                   model="veo-2.0-generate-001"):
    """Animate a still image into a video using Veo 2 i2v."""
    api_key = _load_api_key()  # same pattern as above
    base_url = "https://generativelanguage.googleapis.com/v1beta"
    headers = {"x-goog-api-key": api_key, "Content-Type": "application/json"}

    # Read and encode the image
    with open(image_path, "rb") as f:
        img_b64 = base64.b64encode(f.read()).decode()

    # Detect mime type
    ext = image_path.lower().rsplit(".", 1)[-1]
    mime = {"png": "image/png", "jpg": "image/jpeg", "jpeg": "image/jpeg"}.get(ext, "image/png")

    payload = {
        "instances": [{
            "prompt": prompt or "animate this image with subtle motion, meme energy",
            "image": {"bytesBase64Encoded": img_b64, "mimeType": mime}
        }],
        "parameters": {
            "aspectRatio": aspect_ratio,
            "personGeneration": "allow_adult",
            "durationSeconds": duration
        }
    }

    resp = requests.post(
        f"{base_url}/models/{model}:predictLongRunning",
        headers=headers, json=payload
    )
    # ... same polling + download pattern as text-to-video
```

## Adding Text Overlays with ffmpeg

Add meme-style text on top of the generated video:

```bash
# Top and bottom text overlay (Impact font, white with black outline)
ffmpeg -i /tmp/meme_video.mp4 \
  -vf "drawtext=text='WHEN SOMEONE SAYS MEME COINS ARE DEAD':fontsize=42:fontcolor=white:borderw=3:bordercolor=black:x=(w-text_w)/2:y=30, \
       drawtext=text='\$TACO\: HOLD MY SEASONING':fontsize=42:fontcolor=white:borderw=3:bordercolor=black:x=(w-text_w)/2:y=h-th-30" \
  -codec:a copy /tmp/meme_video_text.mp4
```

```python
import subprocess

def add_text_overlay(video_path, top_text, bottom_text, output_path=None):
    """Add meme-style top/bottom text to a video."""
    if not output_path:
        output_path = video_path.replace(".mp4", "_text.mp4")

    # Escape special chars for ffmpeg
    top_esc = top_text.replace("'", "'\\''").replace(":", "\\:")
    bot_esc = bottom_text.replace("'", "'\\''").replace(":", "\\:")

    cmd = [
        "ffmpeg", "-y", "-i", video_path,
        "-vf", (
            f"drawtext=text='{top_esc}':fontsize=42:fontcolor=white"
            f":borderw=3:bordercolor=black:x=(w-text_w)/2:y=30,"
            f"drawtext=text='{bot_esc}':fontsize=42:fontcolor=white"
            f":borderw=3:bordercolor=black:x=(w-text_w)/2:y=h-th-30"
        ),
        "-codec:a", "copy", output_path
    ]

    result = subprocess.run(cmd, capture_output=True, text=True)
    if result.returncode != 0:
        print(f"ffmpeg error: {result.stderr[:300]}")
        return None
    return output_path
```

## Video Prompt Engineering

Good video prompts for meme coins:

**Action / Hype:**
```
a cartoon taco blasting off on a rocket through space, crypto coins flying everywhere, epic and absurd, meme energy
a taco character doing a victory dance on top of a pile of gold coins, confetti falling, celebratory energy
```

**Story / Narrative:**
```
a taco in a ski mask sneaking through a Target store aisle, stuffing money into a bag, security cameras flashing, heist movie energy
a taco sitting at a trading desk, charts going vertical, taco starts sweating then celebrating
```

**Surreal / Absurd:**
```
a giant taco slowly rising from the ocean like godzilla, tiny boats fleeing, cinematic and absurd
a taco meditating and levitating, everything around it is on fire, serene chaos energy
```

**Reaction / Relatable:**
```
camera slowly zooming into a taco's face as it realizes the chart is pumping, dramatic zoom, meme energy
a taco scrolling through a phone, seeing a notification, eyes go wide, drops phone in excitement
```

## Full Workflow: Generate Video + Tweet

```python
import tweepy, os

def video_meme_and_tweet(prompt, tweet_text, output_path="/tmp/meme_video.mp4",
                         duration="8", aspect_ratio="16:9"):
    """Generate a video meme and tweet it from @chefdaio."""

    # Step 1: Generate video
    video_path, err = generate_video_meme(prompt, output_path, duration, aspect_ratio)
    if not video_path:
        print(f"Generation failed: {err}")
        return None

    # Step 2: Load Twitter credentials
    creds = {}
    creds_path = os.path.expanduser("~/.config/twitter-post/chefdaio.env")
    with open(creds_path) as f:
        for line in f:
            line = line.strip()
            if line and not line.startswith("#") and "=" in line:
                key, val = line.split("=", 1)
                creds[key.strip()] = val.strip()

    # Step 3: Upload video via v1.1 chunked upload
    auth = tweepy.OAuth1UserHandler(
        creds["TWITTER_API_KEY"], creds["TWITTER_API_SECRET"],
        creds["TWITTER_ACCESS_TOKEN"], creds["TWITTER_ACCESS_TOKEN_SECRET"],
    )
    api = tweepy.API(auth, wait_on_rate_limit=True)
    media = api.chunked_upload(output_path, media_category="tweet_video")

    # Step 4: Tweet via v2 API
    client = tweepy.Client(
        consumer_key=creds["TWITTER_API_KEY"],
        consumer_secret=creds["TWITTER_API_SECRET"],
        access_token=creds["TWITTER_ACCESS_TOKEN"],
        access_token_secret=creds["TWITTER_ACCESS_TOKEN_SECRET"],
    )
    response = client.create_tweet(text=tweet_text, media_ids=[media.media_id])
    tweet_url = f"https://x.com/i/status/{response.data['id']}"
    print(f"Video meme tweeted: {tweet_url}")
    return tweet_url
```

**Note:** Twitter video upload uses `chunked_upload` (not `media_upload`) because videos are larger.
Twitter accepts MP4 up to 512MB / 140 seconds. Veo 2 outputs 8s clips well under these limits.

## Parameters Reference

| Parameter | Values | Default | Notes |
|-----------|--------|---------|-------|
| `durationSeconds` | `5`, `6`, `7`, `8` | `8` | Must be a number, not string |
| `aspectRatio` | `"16:9"`, `"9:16"` | `"16:9"` | Landscape or portrait |
| `personGeneration` | `"allow_all"`, `"allow_adult"`, `"dont_allow"` | `"dont_allow"` | Set to allow_adult for meme faces |
| `includeAudio` | `true`, `false` | `true` (Veo 3+) | **Veo 3+ only.** Native audio generation. Disable to save ~33% |
| `enhancePrompt` | `true`, `false` | `true` | **Veo 2 only.** Auto-rewrites prompt with more cinematic detail |
| `negativePrompt` | free text | -- | Describe what to exclude: "blurry, text, watermark, static" |
| `seed` | `0`-`4294967295` | -- | For semi-reproducible output. Not perfectly deterministic |
| `sampleCount` | `1`-`4` | `1` | Generate multiple videos per request |
| `resolution` | `"720p"`, `"1080p"`, `"4k"` | `"720p"` | **Veo 3.1 only.** Higher res = more cost |

## Pipeline Integration

```
meme-research → pump-deploy → video-meme (Veo 2) → twitter-post
                    ^              OR
              solana-wallet   meme-gen (templates)
                                   OR
                              meme-creator (AI images)
```

**When to use which:**
- `meme-gen` (templates): Free, reliable, classic formats. Good for daily automated posts.
- `meme-creator` (AI images): Unique stills, creative, costs ~$0.01/image. Good for special posts.
- `video-meme` (Veo 2 video): Highest engagement, 5-8s clips, ~$2-3/video. Good for launches, milestones, viral attempts.

## Pitfalls

- Generation takes 1-3 minutes for all models. Always poll, don't assume instant results.
- Veo 3.1 may take slightly longer than Veo 2 due to audio generation.
- Prompts that are too vague produce boring results. Be specific about action, character, and mood.
- For Veo 3+ audio: describe sounds in your prompt for best results (e.g. "with dramatic music" or "crowd cheering").
- Person generation may be restricted in some regions. Use `allow_adult` if needed.
- The API may return safety blocks on certain prompts. Rephrase if blocked.
- Veo 2 is 720p only. Veo 3.1 supports 1080p and 4K via the `resolution` param.
- Twitter video upload requires chunked upload API, not the simple media_upload.
- Poll interval of 10-15s is fine. Going faster won't speed up generation and wastes quota.
- `includeAudio` param is ignored on Veo 2 — don't send it for Veo 2 requests.

## Cost Notes

- Veo 2: ~$0.35/sec (8s clip = ~$2.80) — **FREE TIER: ~10 videos/day at 720p**
- Veo 3.1 Fast: ~$0.15/sec (8s clip = ~$1.20) with audio, ~$0.10/sec silent
- Veo 3.1: ~$0.40/sec (8s clip = ~$3.20) with audio
- Free tier is Veo 2 only (undocumented but confirmed March 2026). Veo 3+ has no free tier.
- Strategy: Veo 2 free for daily grind, Veo 3.1 Fast for launches/bangers ($1.20 is cheap for video+audio).

## Download Note

Generated video files are stored for **2 days only** on Google's servers. Always download immediately.
The download URI requires the API key: `curl -L -o video.mp4 -H "x-goog-api-key: $KEY" "$VIDEO_URI"`
Or append `?key=$KEY` as query parameter.
