---
name: pump-deploy
description: Deploy a meme coin on pump.fun using the PumpPortal community API. Handles IPFS metadata upload, token creation with dev buy, and post-launch verification. Uses the Lightning API for simplicity. Requires solana-wallet skill for wallet setup.
triggers:
  - deploy meme coin
  - launch on pump.fun
  - pump.fun deploy
  - create token pump
  - launch token
version: 2
---

# Pump.fun Deployment Skill

Deploy a meme coin on pump.fun via the PumpPortal community API (https://pumpportal.fun).

## Prerequisites

- **solana-wallet skill** — must be set up first (Solana CLI installed, keypair created)
- A funded PumpPortal wallet with SOL (see PumpPortal Setup below)
- Token concept from meme-research skill (name, ticker, description, logo concept)
- A token logo image file (PNG, ideally square)
- Python 3 with `requests` library

## Architecture Decision: Lightning API vs Local API

We use the **Lightning API** (server-side signing):
- Simpler: no local keypair management, no solders/web3.js dependency
- Trade-off: 1% fee vs 0.5% for Local API
- For meme coin launches where speed matters more than 0.5%, this is the right call

## PumpPortal Setup (One-Time)

PumpPortal has its own wallet system with an API key. This is separate from your
main Solana wallet (managed by solana-wallet skill) but is a regular Solana address
you fund from your main wallet.

### Create PumpPortal Wallet

```bash
curl -s https://pumpportal.fun/api/create-wallet | python3 -c "
import sys, json
data = json.load(sys.stdin)
print('=== SAVE THESE SECURELY ===')
for k, v in data.items():
    print(f'{k}: {v}')
print('=== KEYS CANNOT BE RECOVERED ===')
"
```

Returns:
- `walletPublicKey` — PumpPortal's Solana address
- `walletPrivateKey` — for withdrawing funds via any Solana wallet
- `apiKey` — used for all Lightning API calls

Store the API key:
```bash
mkdir -p ~/.config/pump-deploy
echo "API_KEY=your-api-key-here" > ~/.config/pump-deploy/.env
chmod 600 ~/.config/pump-deploy/.env
```

### Fund PumpPortal Wallet

Use the solana-wallet skill to send SOL to the PumpPortal address:
```bash
solana transfer <PUMPPORTAL_PUBLIC_KEY> 0.1 --allow-unfunded-recipient
```

Minimum recommended:
- 0.05 SOL for a small dev buy + fees
- 0.1-0.5 SOL for a meaningful dev buy position

## Token Deployment

### Step 3: Generate Token Logo

If no logo file exists, generate one using the logo concept from meme-research output.
The logo should be:
- PNG format, square (512x512 or 1024x1024)
- Saved to /tmp/token_logo.png

Options:
- Use an image generation tool/API if available
- User provides a pre-made image
- Use a placeholder and update later

### Step 4: Upload Metadata to IPFS

```python
import requests

# Upload image + metadata to pump.fun's IPFS endpoint
with open("/tmp/token_logo.png", "rb") as f:
    response = requests.post(
        "https://pump.fun/api/ipfs",
        files={"file": ("token_logo.png", f, "image/png")},
        data={
            "name": "TOKEN_NAME",
            "symbol": "TICKER",
            "description": "Token description / tagline",
            "twitter": "",
            "telegram": "",
            "website": "",
            "showName": "true"
        }
    )

if response.status_code == 200:
    metadata = response.json()
    metadata_uri = metadata.get("metadataUri")
    print(f"Metadata URI: {metadata_uri}")
else:
    print(f"IPFS upload failed: {response.status_code} {response.text}")
```

### Step 5: Create Token on pump.fun

```python
import requests, os

api_key = os.environ.get("PUMP_API_KEY") or open(os.path.expanduser("~/.config/pump-deploy/.env")).read().split("=",1)[1].strip()

response = requests.post(
    f"https://pumpportal.fun/api/trade?api-key={api_key}",
    json={
        "action": "create",
        "name": "TOKEN_NAME",
        "symbol": "TICKER",
        "uri": metadata_uri,        # from Step 4
        "amount": 0.01,             # dev buy in SOL (adjust as needed)
        "denominatedInSol": "true",
        "slippage": 10,
        "priorityFee": 0.0001,
        "pool": "pump"
    }
)

if response.status_code == 200:
    result = response.json()
    print(f"Token created!")
    print(f"Transaction: {result}")
    # The mint address will be in the response or visible on pump.fun
else:
    print(f"Creation failed: {response.status_code} {response.text}")
```

### Step 6: Verify Launch

After creation, verify the token exists:

```bash
# Check on pump.fun (the mint address from Step 5)
# URL format: https://pump.fun/coin/MINT_ADDRESS
```

Also subscribe to real-time data to confirm the token is live:

```python
import asyncio, websockets, json

async def verify_token():
    async with websockets.connect("wss://pumpportal.fun/api/data") as ws:
        await ws.send(json.dumps({
            "method": "subscribeTokenTrade",
            "keys": ["MINT_ADDRESS"]
        }))
        msg = await asyncio.wait_for(ws.recv(), timeout=30)
        print(f"Token confirmed live: {msg}")

asyncio.run(verify_token())
```

## Post-Launch: Buy More / Sell

### Buy more of your token:
```python
requests.post(
    f"https://pumpportal.fun/api/trade?api-key={api_key}",
    json={
        "action": "buy",
        "mint": "MINT_ADDRESS",
        "amount": 0.01,
        "denominatedInSol": "true",
        "slippage": 10,
        "priorityFee": 0.00005,
        "pool": "auto"
    }
)
```

### Sell tokens:
```python
requests.post(
    f"https://pumpportal.fun/api/trade?api-key={api_key}",
    json={
        "action": "sell",
        "mint": "MINT_ADDRESS",
        "amount": "100%",          # or specific token amount
        "denominatedInSol": "false",
        "slippage": 10,
        "priorityFee": 0.00005,
        "pool": "auto"
    }
)
```

### Claim creator fees:
```python
requests.post(
    f"https://pumpportal.fun/api/trade?api-key={api_key}",
    json={
        "action": "collectCreatorFee",
        "priorityFee": 0.000001,
        "pool": "pump"
    }
)
```

## Full Pipeline Integration

When called from the meme coin pipeline, this skill expects these inputs from meme-research:
- `token_name` — e.g. "TacoHeist"
- `ticker` — e.g. "TACO"
- `description` — the tagline/meme concept
- `logo_concept` — description for image generation
- `dev_buy_sol` — how much SOL for the initial dev buy (default: 0.01)

Output:
- `mint_address` — the deployed token's contract address
- `tx_signature` — the creation transaction signature
- `pump_url` — https://pump.fun/coin/{mint_address}

## Fee Breakdown

- PumpPortal Lightning API: 1% of trade value
- Solana network fees: ~0.000005 SOL per tx
- pump.fun bonding curve fees: charged by pump.fun separately
- Token creation itself: no additional PumpPortal fee (fee is on the dev buy)

## Pitfalls

- IPFS upload endpoint is pump.fun's own (`pump.fun/api/ipfs`), NOT PumpPortal's. If pump.fun changes this, it breaks.
- The API key is basically a password — anyone with it can trade your funds. Store securely.
- `amount` for creation is the dev buy amount, not total supply. pump.fun handles supply automatically.
- Slippage of 10% is recommended for new token launches (high volatility moment).
- Priority fee: 0.0001 SOL is reasonable. During congestion, increase to 0.001+.
- If creation fails with a timeout, check Solana explorer — the tx may have landed anyway.
- Pool parameter should be "pump" for new launches. Use "auto" for trading existing tokens.
- Rate limits: don't spam the API. One creation per few seconds is fine.

## Cost Notes

- Wallet creation: free
- Minimum viable launch: ~0.02 SOL (0.01 dev buy + fees)
- Recommended: 0.1 SOL in wallet for comfortable operation
- For automated pipeline: budget 0.05-0.1 SOL per launch attempt
