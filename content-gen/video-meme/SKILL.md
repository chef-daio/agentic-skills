---
name: video-meme
description: Generate short meme videos (5-8 seconds) using Google Veo 2 via the Gemini API. Text-to-video and image-to-video. Chains with twitter-post for auto-posting video memes to @chefdaio.
triggers:
  - generate video meme
  - video meme
  - meme video
  - make a video
  - video clip
  - short video
  - veo video
  - animated meme
version: 2
---

# Video Meme Skill

Generate short meme video clips (5-8 seconds) using Google Veo 2 via the Gemini API.
For static memes, use content-gen/meme-gen (templates) or content-gen/meme-creator (AI images).

## Prerequisites

- `curl`, `python3`, `jq` (pre-installed on macOS/Linux, install jq if missing)
- `ffmpeg` (for text overlays and format conversion)
- Gemini API key stored in `~/.hermes/.env` as `GEMINI_API_KEY`
- Get a key at: https://aistudio.google.com/apikey

## Model Info

| Model | ID | Max Duration | Resolution | Cost | Notes |
|-------|-----|-------------|------------|------|-------|
| **Veo 2** | `veo-2.0-generate-001` | 8 seconds | 720p | ~$0.35/sec | Stable, good default |
| **Veo 3** | `veo-3.0-generate-001` | 8 seconds | 720p-1080p | ~$0.50/sec | Better quality |
| **Veo 3 Fast** | `veo-3.0-fast-generate-001` | 8 seconds | 720p | ~$0.25/sec | Faster, cheaper |
| **Veo 3.1** | `veo-3.1-generate-preview` | 8 seconds | up to 4K | ~$0.50/sec | Best quality, preview |
| **Veo 3.1 Fast** | `veo-3.1-fast-generate-preview` | 8 seconds | 720p-1080p | ~$0.25/sec | Fast + high quality |

All models support text-to-video (t2v) and image-to-video (i2v).
Veo 3.1 additionally supports `resolution` param: `"720p"`, `"1080p"`, `"4k"`.

**Recommendation:** Use `veo-2.0-generate-001` for daily meme content (cheap, reliable).
Use `veo-3.1-generate-preview` for high-impact posts like launches.

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
MODEL="veo-2.0-generate-001"  # or veo-3.0-generate-001, veo-3.1-generate-preview, etc.

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
      "durationSeconds": 8
    }
  }' | jq -r '.name')

echo "Operation: $OPERATION"
```

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
                        poll_interval=15, max_wait=300):
    """Generate a short meme video using Google Veo.

    Args:
        prompt: Description of the video to generate
        output_path: Where to save the MP4
        duration: 5, 6, or 8 seconds (int)
        aspect_ratio: "16:9" (landscape) or "9:16" (portrait/vertical)
        model: Veo model ID (see Model Info table)
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

    # Step 1: Submit generation request
    payload = {
        "instances": [{"prompt": prompt}],
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
| `durationSeconds` | `5`, `6`, `8` | `8` | Must be a number, not string |
| `aspectRatio` | `"16:9"`, `"9:16"` | `"16:9"` | Landscape or portrait |
| `personGeneration` | `"allow_all"`, `"allow_adult"`, `"dont_allow"` | `"dont_allow"` | Set to allow_adult for meme faces |

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

- Veo 2 generation takes 1-3 minutes. Always poll, don't assume instant results.
- Cost is ~$0.35/second of video = ~$2.80 for an 8-second clip. Use sparingly for high-impact posts.
- Prompts that are too vague produce boring results. Be specific about action, character, and mood.
- Person generation may be restricted in some regions. Use `allow_adult` if needed.
- The API may return safety blocks on certain prompts. Rephrase if blocked.
- Videos are 720p. No resolution control on Veo 2 (Veo 3.1 supports 1080p).
- Twitter video upload requires chunked upload API, not the simple media_upload.
- Poll interval of 10-15s is fine. Going faster won't speed up generation and wastes quota.

## Cost Notes

- Veo 2: ~$0.35/sec (8s clip = ~$2.80)
- Veo 3 Fast / 3.1 Fast: ~$0.25/sec (8s clip = ~$2.00)
- Veo 3 / 3.1: ~$0.50/sec (8s clip = ~$4.00)
- No free tier for video generation (unlike image models)
- Recommendation: use for launches, milestones, and high-engagement moments. Use free template memes for daily automated posts.
