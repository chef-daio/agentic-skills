---
name: meme-research
description: Scrape Reddit and news sources to find trending stories with meme coin potential. Scores headlines on virality, ticker potential, cultural moment, visual potential, longevity, and controversy. Outputs ranked candidates with token concepts.
triggers:
  - meme research
  - find meme coin ideas
  - scan reddit for memes
  - memetic research
  - meme coin candidates
version: 1
---

# Meme Research Skill

Find trending stories with meme coin potential by scraping Reddit and scoring headlines.

## Sources

Primary:
- r/worldnews (geopolitics, absurd international news)
- r/nottheonion (inherently absurd real news)

Future additions:
- r/wallstreetbets (financial memes)
- Twitter/X trending topics
- Google Trends
- pump.fun trending tokens (to avoid duplicates)

## Step 1: Scrape Reddit

Use curl to hit Reddit's public JSON API (no auth needed, free):

```bash
# World News - hot 25
curl -s -H 'User-Agent: MemeResearchBot/1.0' \
  'https://www.reddit.com/r/worldnews/hot.json?limit=25' > /tmp/worldnews.json

# Not The Onion - hot 25
curl -s -H 'User-Agent: MemeResearchBot/1.0' \
  'https://www.reddit.com/r/nottheonion/hot.json?limit=25' > /tmp/nottheonion.json
```

## Step 2: Parse Headlines

Use python with `strict=False` to handle Reddit's JSON control characters:

```python
import json

for name, path in [("worldnews", "/tmp/worldnews.json"), ("nottheonion", "/tmp/nottheonion.json")]:
    with open(path) as f:
        data = json.loads(f.read(), strict=False)
    for post in data["data"]["children"]:
        p = post["data"]
        if p.get("stickied"):
            continue
        print(f"[{p['score']:,} pts | {p['num_comments']} comments] {p['title']}")
```

## Step 3: Score for Memeability

Send the parsed headlines to the LLM for analysis. Use a delegate_task subagent with this prompt framework:

### Scoring Criteria (each out of 10):
1. **VIRALITY** - Is it absurd, ironic, or funny enough to spread?
2. **TICKER POTENTIAL** - Can you make a catchy 3-5 letter ticker?
3. **CULTURAL MOMENT** - Does it capture a zeitgeist people will rally around?
4. **VISUAL POTENTIAL** - Can you imagine a funny logo/meme?
5. **LONGEVITY** - Will people still care in 24-48 hours?
6. **CONTROVERSY** - Edgy enough to be interesting, not so toxic it gets banned

### Filters:
- AVOID anything directly about deaths/casualties/war crimes (bad taste)
- FOCUS on absurd, ironic, funny stories
- CHECK if similar tokens already exist on pump.fun

### Output Format:
For each top 5 candidate:
- Headline source
- Suggested token NAME and TICKER
- The meme concept/angle (the joke, the vibe)
- Composite score out of 10
- Suggested tagline/description
- Logo concept description

## Step 4: Duplicate Check (Optional)

Search pump.fun to see if similar tokens already exist:
```
https://pump.fun/?search=TICKER_NAME
```

## Pitfalls

- Reddit JSON API rate limits: ~60 requests/minute without auth. Don't hammer it.
- Reddit JSON has control characters - always use `json.loads(data, strict=False)`
- The failed approach: using subagents to scrape Reddit directly (they try to write files). Do the scraping in execute_code, send parsed text to subagent for analysis.
- Timing matters: best meme coins capture a MOMENT. Run this frequently during high-news periods.
- The analysis subagent only needs headlines + scores, not full articles.

## Cost Notes

- Reddit scraping: $0.00 (free public API)
- LLM analysis: depends on model. For hourly cronjobs, use a cheap model (gemini-flash, deepseek). For deep analysis, use the main model.
