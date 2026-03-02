---
name: meme-gen
description: Generate memes from popular templates using the Memegen.link API. Pick trending formats, write captions for meme coins or any topic, download as PNG. Free, no API key needed. Chains with twitter-post for auto-posting.
triggers:
  - generate meme
  - create meme
  - make a meme
  - meme about
  - meme for coin
  - meme content
version: 1
---

# Meme Generation Skill

Generate memes from 200+ classic templates using the free Memegen.link API. No API key required.

## Prerequisites

- `curl` (pre-installed on macOS/Linux)
- `python3` (for JSON parsing)
- No API keys, no dependencies to install

## How It Works

Memegen.link hosts 200+ meme templates (Drake, Distracted Boyfriend, Fry, etc.) and generates
images on-the-fly via URL. You pick a template, supply captions, and download the result.

## Quick Reference: Popular Templates

| ID | Name | Lines | Best For |
|----|------|-------|----------|
| `drake` | Drakeposting | 2 | X vs Y comparisons |
| `distracted` | Distracted Boyfriend | 3 | temptation/switching |
| `fry` | Futurama Fry | 2 | "not sure if..." |
| `buzz` | Buzz & Woody | 2 | "X everywhere" |
| `astronaut` | Always Has Been | 2 | reveals |
| `mordor` | One Does Not Simply | 2 | difficulty |
| `change` | Change My Mind | 1 | hot takes |
| `rollsafe` | Roll Safe | 1 | "smart" takes |
| `both` | Why Not Both | 1 | having it all |
| `kermit` | Evil Kermit | 2 | inner voice |
| `doge` | Doge | 2 | such wow |
| `picard` | Picard Facepalm | 2 | frustration |
| `success` | Success Kid | 2 | wins |
| `disaster` | Disaster Girl | 2 | chaos |
| `fine` | This Is Fine | 2 | denial |
| `sad-biden` | Sad Biden | 2 | disappointment |
| `panik` | Panik Kalm Panik | 3 | escalating panic |
| `waiting` | Waiting Skeleton | 2 | waiting forever |
| `money` | Shut Up and Take My Money | 2 | impulse buys |
| `blob` | Blob Pointing | 2 | pointing at things |

## Listing All Templates

```bash
curl -s https://api.memegen.link/templates | python3 -c "
import json, sys
templates = json.load(sys.stdin)
for t in templates:
    print(f\"{t['id']:20s} {t['name']:35s} ({t['lines']} lines)\")
print(f'\nTotal: {len(templates)} templates')
"
```

Filter by keyword:
```bash
curl -s "https://api.memegen.link/templates?filter=dog" | python3 -c "
import json, sys
for t in json.load(sys.stdin):
    print(f\"{t['id']:20s} {t['name']}\")
"
```

## Generating a Meme

### Method 1: Direct URL (simplest)

URL format: `https://api.memegen.link/images/{template}/{line1}/{line2}.png`

```bash
# Drake meme
curl -o /tmp/meme.png "https://api.memegen.link/images/drake/buying_random_coins/buying_~hTACO.png"

# Fry "not sure if" meme
curl -o /tmp/meme.png "https://api.memegen.link/images/fry/not_sure_if_early/or_just_delusional.png"

# One Does Not Simply
curl -o /tmp/meme.png "https://api.memegen.link/images/mordor/one_does_not_simply/ignore_~hTACO.png"

# Single-line template (Change My Mind)
curl -o /tmp/meme.png "https://api.memegen.link/images/change/~hTACO_is_the_next_100x.png"

# Skip a line with underscore
curl -o /tmp/meme.png "https://api.memegen.link/images/doge/_/such_taco_very_moon.png"
```

### Method 2: POST API (programmatic)

```python
import requests, json

# Generate meme via POST
response = requests.post(
    "https://api.memegen.link/images",
    json={
        "template_id": "drake",
        "text": ["selling at a loss", "buying more $TACO"],
        "extension": "png"
    }
)
meme_url = response.json()["url"]

# Download the image
img = requests.get(meme_url)
with open("/tmp/meme.png", "wb") as f:
    f.write(img.content)
print(f"Saved to /tmp/meme.png")
```

### Method 3: Full pipeline function

```python
import requests, os, re

def generate_meme(template_id, lines, output_path="/tmp/meme.png", width=800):
    """Generate a meme and save it locally.

    Args:
        template_id: Template slug (e.g. 'drake', 'fry', 'distracted')
        lines: List of caption strings
        output_path: Where to save the PNG
        width: Image width in pixels (default 800)

    Returns:
        Path to saved image, or None on failure
    """
    response = requests.post(
        "https://api.memegen.link/images",
        json={
            "template_id": template_id,
            "text": lines,
            "extension": "png"
        }
    )
    if response.status_code != 200:
        print(f"API error: {response.status_code} {response.text}")
        return None

    meme_url = response.json()["url"]
    if width:
        sep = "&" if "?" in meme_url else "?"
        meme_url += f"{sep}width={width}"

    img = requests.get(meme_url)
    if img.status_code != 200:
        print(f"Download error: {img.status_code}")
        return None

    os.makedirs(os.path.dirname(output_path) or ".", exist_ok=True)
    with open(output_path, "wb") as f:
        f.write(img.content)

    print(f"Meme saved: {output_path} ({len(img.content)} bytes)")
    return output_path
```

## Special Character Encoding (URL method only)

When building URLs directly, these characters need special encoding:

| Character | Encode as | Example |
|-----------|-----------|---------|
| space | `_` | `hello_world` |
| literal `_` | `__` | `__init__` |
| `?` | `~q` | `really~q` |
| `&` | `~a` | `you~a_me` |
| `#` | `~h` | `~hTACO` |
| `/` | `~s` | `yes~sno` |
| `%` | `~p` | `100~p` |
| newline | `~n` | `line1~nline2` |
| `"` | `''` | `he_said_''hi''` |

The POST API handles encoding automatically — no escaping needed.

## Meme Coin Content Strategy

When generating memes for a coin like $TACO, rotate through these formats:

**Comparison memes** (drake, distracted):
- "Buying ETH at the top" vs "Buying $TACO at launch"
- Template: drake, distracted

**Hype memes** (astronaut, buzz):
- "Wait, it's all $TACO?" / "Always has been"
- Template: astronaut

**FOMO memes** (fry, panik):
- "Not sure if early" / "or everyone else is late"
- Template: fry

**Community memes** (doge, kermit, fine):
- "Portfolio down 80%" / "This is fine, I have $TACO"
- Template: fine

**Hot take memes** (change, rollsafe):
- "$TACO is the only honest meme coin"
- Template: change

## Full Workflow: Generate + Tweet

```python
import requests, tweepy, os

def meme_and_tweet(template_id, lines, tweet_text, output_path="/tmp/meme.png"):
    """Generate a meme and tweet it from @chefdaio."""

    # Step 1: Generate meme
    resp = requests.post(
        "https://api.memegen.link/images",
        json={"template_id": template_id, "text": lines, "extension": "png"}
    )
    meme_url = resp.json()["url"] + "?width=800"
    img = requests.get(meme_url)
    with open(output_path, "wb") as f:
        f.write(img.content)

    # Step 2: Load Twitter credentials
    creds = {}
    with open(os.path.expanduser("~/.config/twitter-post/.env")) as f:
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
    print(f"Meme tweeted: {tweet_url}")
    return tweet_url
```

## Pipeline Integration

This skill is stage 3 of the meme coin pipeline.

Inputs (from meme-research or manual):
- `ticker` — e.g. "TACO"
- `token_name` — e.g. "TacoHeist"
- `pump_url` — e.g. "https://pump.fun/coin/MINT_ADDRESS"
- `theme` — the meme concept/angle to play on

Output:
- `meme_path` — local path to generated PNG
- `tweet_url` — URL of posted tweet (if auto-posted)

Pipeline flow:
```
meme-research → pump-deploy → content-gen/meme-gen → twitter-post
                    ^
              solana-wallet
```

## Query Parameters

| Param | Description | Example |
|-------|-------------|---------|
| `width` | Image width | `?width=800` |
| `height` | Image height | `?height=600` |
| `font` | Font choice | `?font=impact` |
| `style` | Template variant | `?style=default` |

Available fonts: `impact`, `titilliumweb` (thick), `notosans`, `kalam` (comic), `titilliumweb-thin`.

## Pitfalls

- Template IDs are slugs, not display names. Use the list endpoint to find the right ID.
- The URL method requires special encoding for #, ?, &, etc. The POST API is safer.
- Some templates have 3 lines (distracted, panik), most have 2, some have 1. Check `lines` field.
- Images are generated on-the-fly — first request for a unique combo may be slow (~1-2s).
- No rate limit documented, but don't spam. A few dozen per hour is fine.
- Output is standard meme quality (not AI art). Good for Twitter/community engagement, not for logos.

## Cost Notes

- Memegen.link API: completely free, no auth required
- No rate limits for reasonable usage
- Combined with twitter-post free tier: $0 total for meme content pipeline
