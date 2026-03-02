---
name: meme-creator
description: Generate original AI meme images using Gemini Nano Banana 2. Creates unique, creative meme art — not templates. Surreal, absurdist, and custom scenes for meme coins, reactions, and shitposting. Chains with twitter-post for auto-posting.
triggers:
  - create original meme
  - ai meme
  - generate meme art
  - creative meme
  - custom meme image
  - meme creator
  - ai generated meme
version: 1
---

# AI Meme Creator Skill

Generate original meme images using Gemini Nano Banana 2 (gemini-3.1-flash-image-preview).
This is NOT template-based — it creates unique AI-generated scenes, characters, and absurd visuals.

For classic template memes (Drake, Fry, etc.), use content-gen/meme-gen instead.

## Prerequisites

- `curl` and `python3` (pre-installed on macOS/Linux)
- Gemini API key stored in `~/.hermes/.env` as `GEMINI_API_KEY`
- Get a key at: https://aistudio.google.com/apikey

## Available Models

| Model | ID | Speed | Quality | Cost |
|-------|-----|-------|---------|------|
| **Nano Banana 2** | `gemini-3.1-flash-image-preview` | Fast | Great, 1408x768 | Free tier available |
| Nano Banana | `gemini-2.5-flash-image` | Fast | Good | Free tier available |
| Nano Banana Pro | `gemini-3-pro-image-preview` | Slower | Best | Free tier limited |
| Imagen 4 | `imagen-4.0-generate-001` | Fast | Photorealistic | Free tier available |

Default: **Nano Banana 2** — best balance of speed, quality, and creativity.

## Quick Start: Generate an Image

### Bash (curl)

```bash
# Load API key
GEMINI_API_KEY=$(grep GEMINI_API_KEY ~/.hermes/.env | cut -d= -f2)

# Generate image
curl -s "https://generativelanguage.googleapis.com/v1beta/models/gemini-3.1-flash-image-preview:generateContent?key=$GEMINI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "contents": [{
      "parts": [{
        "text": "Generate an image: a cartoon taco wearing a ski mask, running away from a Target store with bags of money. meme style, funny, exaggerated expressions."
      }]
    }],
    "generationConfig": {
      "responseModalities": ["TEXT", "IMAGE"]
    }
  }' | python3 -c "
import json, sys, base64
resp = json.load(sys.stdin)
for part in resp['candidates'][0]['content']['parts']:
    if 'inlineData' in part:
        img = base64.b64decode(part['inlineData']['data'])
        with open('/tmp/meme.png', 'wb') as f:
            f.write(img)
        print(f'Saved /tmp/meme.png ({len(img)} bytes)')
    elif 'text' in part:
        print(f'Description: {part[\"text\"][:200]}')
"
```

### Python (full function)

```python
import requests, json, base64, os

def generate_ai_meme(prompt, output_path="/tmp/meme.png", model="gemini-3.1-flash-image-preview"):
    """Generate an original AI meme image.

    Args:
        prompt: Creative description of the meme to generate
        output_path: Where to save the PNG
        model: Gemini model ID (default: Nano Banana 2)

    Returns:
        Tuple of (image_path, description) or (None, error_message)
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

    url = f"https://generativelanguage.googleapis.com/v1beta/models/{model}:generateContent?key={api_key}"
    payload = {
        "contents": [{"parts": [{"text": prompt}]}],
        "generationConfig": {"responseModalities": ["TEXT", "IMAGE"]}
    }

    resp = requests.post(url, json=payload)
    if resp.status_code != 200:
        return None, f"API error {resp.status_code}: {resp.text[:200]}"

    data = resp.json()
    if "candidates" not in data or not data["candidates"]:
        return None, f"No candidates in response: {json.dumps(data)[:200]}"

    description = ""
    image_saved = False
    for part in data["candidates"][0]["content"]["parts"]:
        if "inlineData" in part:
            img_bytes = base64.b64decode(part["inlineData"]["data"])
            os.makedirs(os.path.dirname(output_path) or ".", exist_ok=True)
            with open(output_path, "wb") as f:
                f.write(img_bytes)
            image_saved = True
            print(f"Meme saved: {output_path} ({len(img_bytes)} bytes)")
        elif "text" in part:
            description = part["text"]

    if not image_saved:
        return None, "No image in response"

    return output_path, description
```

## Prompt Engineering for Memes

The key to good AI memes is the prompt. Here are patterns that work:

### Style Directives (append to any prompt)

```
meme style, funny, exaggerated cartoon, bold text optional
internet meme aesthetic, shitpost energy, absurd humor
surreal meme, deep fried aesthetic, chaotic energy
clean cartoon style, reaction image, expressive face
```

### Meme Coin Prompts — Prompt Bank

Use these as starting points. Replace $TACO/taco with the relevant coin and its theme.

**Origin Story Memes:**
```
a cartoon taco wearing a ski mask running out of a Target store with bags of money, security guards chasing, meme style, funny exaggerated expressions
a taco sitting at a computer looking at a crypto chart going up, sweating with excitement, cartoon meme style
```

**Hype / Moon Memes:**
```
a taco riding a rocket to the moon, wearing sunglasses, crypto coins trailing behind, cartoon meme style, epic and funny
a massive taco floating in space above earth like a deity, golden glow, absurd meme energy
```

**Community / Holder Memes:**
```
two tacos high-fiving with diamond hands, crypto meme style, celebratory energy
a taco meditating peacefully while everything around it is on fire, "this is fine" energy, cartoon meme
```

**Reaction Memes:**
```
a shocked taco looking at a phone screen, eyes bulging out cartoon style, reaction meme format
a smug taco wearing a tiny crown, looking down at other coins, meme energy, funny
```

**Absurdist / Surreal:**
```
a taco transcending reality, floating through a vaporwave landscape, deep fried meme aesthetic, surreal
cursed image of a taco with human legs walking into a bank, liminal space energy, weird meme
```

**vs. Other Coins:**
```
a tiny taco standing confidently next to a giant scared bitcoin, david vs goliath energy, cartoon meme
a taco casually walking past a graveyard of failed crypto logos, unbothered energy, funny meme
```

## Generating Memes for Any Coin

When generating memes for a deployed token, follow this pattern:

1. Read the token info from `~/.config/meme-pipeline/deployed-tokens.json`
2. Use the token's `theme` and `backstory` to craft a relevant prompt
3. Add meme style directives
4. Generate with Nano Banana 2
5. Save and optionally tweet

```python
import json, os

def meme_for_coin(ticker, style="cartoon meme"):
    """Generate an AI meme for a deployed coin."""
    tokens_path = os.path.expanduser("~/.config/meme-pipeline/deployed-tokens.json")
    with open(tokens_path) as f:
        tokens = json.load(f)

    token = next((t for t in tokens if t["ticker"] == ticker), None)
    if not token:
        print(f"Token {ticker} not found in deployed tokens")
        return None

    # Build a creative prompt from the token's theme
    prompt = (
        f"Generate a funny meme image about {token['name']} (${token['ticker']}). "
        f"Theme: {token.get('theme', 'crypto meme coin')}. "
        f"Backstory: {token.get('backstory', '')}. "
        f"Style: {style}, exaggerated expressions, internet humor, shitpost energy."
    )

    return generate_ai_meme(prompt, output_path=f"/tmp/meme_{ticker.lower()}.png")
```

## Full Workflow: Generate + Tweet

```python
import tweepy, os

def ai_meme_and_tweet(prompt, tweet_text, output_path="/tmp/meme.png"):
    """Generate an AI meme and tweet it from @chefdaio."""

    # Step 1: Generate the meme
    img_path, desc = generate_ai_meme(prompt, output_path)
    if not img_path:
        print(f"Generation failed: {desc}")
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

    # Step 3: Upload image via v1.1 API
    auth = tweepy.OAuth1UserHandler(
        creds["X_API_KEY"], creds["X_API_KEY_SECRET"],
        creds["X_ACCESS_TOKEN"], creds["X_ACCESS_TOKEN_SECRET"],
    )
    api = tweepy.API(auth)
    media = api.media_upload(output_path)

    # Step 4: Tweet via v2 API
    client = tweepy.Client(
        consumer_key=creds["X_API_KEY"],
        consumer_secret=creds["X_API_KEY_SECRET"],
        access_token=creds["X_ACCESS_TOKEN"],
        access_token_secret=creds["X_ACCESS_TOKEN_SECRET"],
    )
    response = client.create_tweet(text=tweet_text, media_ids=[media.media_id])
    tweet_url = f"https://x.com/i/status/{response.data['id']}"
    print(f"AI meme tweeted: {tweet_url}")
    return tweet_url
```

## Image-Only Mode (no text response)

If you only want the image and no text description back:

```python
payload = {
    "contents": [{"parts": [{"text": prompt}]}],
    "generationConfig": {"responseModalities": ["IMAGE"]}  # IMAGE only
}
```

## Using Imagen 4 Instead (photorealistic)

For photorealistic memes (less cartoon, more "cursed stock photo" energy):

```bash
GEMINI_API_KEY=$(grep GEMINI_API_KEY ~/.hermes/.env | cut -d= -f2)

curl -s "https://generativelanguage.googleapis.com/v1beta/models/imagen-4.0-generate-001:predict?key=$GEMINI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "instances": [{"prompt": "a photorealistic taco sitting in a corporate boardroom meeting, looking serious"}],
    "parameters": {"sampleCount": 1}
  }' | python3 -c "
import json, sys, base64
resp = json.load(sys.stdin)
img = base64.b64decode(resp['predictions'][0]['bytesBase64Encoded'])
with open('/tmp/meme.png', 'wb') as f:
    f.write(img)
print(f'Saved /tmp/meme.png ({len(img)} bytes)')
"
```

## Adding Text Overlays

Nano Banana 2 can include text in generated images, but for reliable text placement,
generate the image first then add text with ImageMagick:

```bash
# Add top/bottom text to an AI-generated meme
convert /tmp/meme.png \
  -gravity North -pointsize 48 -stroke black -strokewidth 2 -fill white \
  -annotate +0+20 "WHEN SOMEONE SAYS MEME COINS ARE DEAD" \
  -gravity South -pointsize 48 -stroke black -strokewidth 2 -fill white \
  -annotate +0+20 '$TACO: HOLD MY SEASONING' \
  /tmp/meme_with_text.png
```

Install ImageMagick if needed: `brew install imagemagick` (macOS) or `apt install imagemagick` (Linux).

## Pipeline Integration

This skill is an alternative to content-gen/meme-gen for stage 3 of the pipeline.

```
meme-research → pump-deploy → meme-creator (AI) → twitter-post
                    ^              OR
              solana-wallet   meme-gen (templates)
```

**When to use which:**
- `meme-gen` (templates): Free, reliable, classic formats. Good for daily automated posts.
- `meme-creator` (AI): Unique visuals, more creative, costs API quota. Good for launches, special posts, engagement.

Inputs:
- `ticker` — e.g. "TACO"
- `theme` — the meme concept/backstory
- `style` — cartoon, surreal, photorealistic, deep-fried

Output:
- `meme_path` — local path to generated PNG (typically 1408x768 or 1024x1024)
- `tweet_url` — URL of posted tweet (if auto-posted)

## Pitfalls

- Gemini may refuse prompts it considers unsafe. Rephrase if you get a safety block.
- Text rendering in AI images is hit-or-miss. Use ImageMagick overlay for reliable text.
- Free tier has daily quota limits. If exhausted, fall back to meme-gen templates.
- Images are ~1-2MB PNGs. Twitter accepts up to 5MB so this is fine.
- Nano Banana 2 outputs 1408x768 by default. No size control in the API.
- The model may add artistic interpretation. Be specific in prompts if you want exact scenes.

## Cost Notes

- Gemini free tier: generous daily quota (varies by model)
- Paid tier: ~$0.01-0.02 per image generation
- Recommendation: use meme-gen (free templates) for daily cron, meme-creator for special content
