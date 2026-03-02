---
name: solana-wallet
description: Manage Solana wallets from the CLI. Create keypairs, check balances, send/receive SOL, switch networks (devnet/mainnet), and fund external wallets (e.g. PumpPortal). Uses the official Solana CLI tools.
triggers:
  - solana wallet
  - create solana wallet
  - check sol balance
  - send sol
  - fund wallet
  - solana keypair
  - airdrop sol
version: 1
---

# Solana Wallet Management Skill

Manage Solana wallets using the official Solana CLI. Covers keypair creation, balance checks, transfers, network switching, and key management.

## Installation

```bash
# One-command install (macOS/Linux/WSL)
curl --proto '=https' --tlsv1.2 -sSfL https://solana-install.solana.workers.dev | bash

# Verify
solana --version
solana-keygen --version
```

After install, restart your shell or run the PATH export command shown in the installer output.

## Step 1: Create a Wallet (Keypair)

```bash
# Generate default keypair at ~/.config/solana/id.json
solana-keygen new

# Generate to a specific file (for multiple wallets)
solana-keygen new --outfile ~/.config/solana/meme-deployer.json

# Skip passphrase prompt (for automation)
solana-keygen new --no-passphrase --outfile ~/.config/solana/meme-deployer.json

# View your public key (address)
solana address
solana-keygen pubkey ~/.config/solana/meme-deployer.json
```

The keypair file is a JSON array of 64 bytes (32 private + 32 public). Guard it like a password.

### Vanity Address (optional)

```bash
# Generate address starting with specific chars
solana-keygen grind --starts-with taco:1
```

### Set Default Keypair

```bash
solana config set --keypair ~/.config/solana/meme-deployer.json
```

## Step 2: Choose Network

```bash
# Devnet (for testing — free airdrop SOL available)
solana config set --url devnet

# Mainnet (real money)
solana config set --url mainnet-beta

# Check current config
solana config get
```

Always test on devnet first before touching mainnet.

## Step 3: Fund the Wallet

### On Devnet (free test SOL)

```bash
solana airdrop 2
# or to a specific address
solana airdrop 2 <ADDRESS>
```

Limited to 5 SOL per request. If rate-limited, wait a minute and retry.

### On Mainnet

Buy SOL on an exchange (Coinbase, Binance, etc.) and withdraw to your wallet address. Or receive from another wallet:

```bash
# From another local keypair
solana transfer <YOUR_ADDRESS> 0.5 --from ~/.config/solana/other-wallet.json --allow-unfunded-recipient
```

## Step 4: Check Balance

```bash
# Default wallet
solana balance

# Specific address
solana balance <ADDRESS>

# In lamports (1 SOL = 1,000,000,000 lamports)
solana balance --lamports

# On a specific network without changing config
solana balance <ADDRESS> --url devnet
```

### Lightweight Balance Check (no CLI needed)

```bash
curl -s https://api.mainnet-beta.solana.com -X POST \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"getBalance","params":["ADDRESS"]}' \
  | python3 -c "import sys,json; d=json.load(sys.stdin); print(f'{d[\"result\"][\"value\"]/1e9:.4f} SOL')"
```

Replace `mainnet-beta` with `devnet` for testnet balance.

## Step 5: Send SOL

```bash
# Basic transfer
solana transfer <RECIPIENT> <AMOUNT_SOL>

# To a new (unfunded) address
solana transfer <RECIPIENT> <AMOUNT_SOL> --allow-unfunded-recipient

# From a specific keypair
solana transfer <RECIPIENT> <AMOUNT_SOL> --from <KEYPAIR_PATH> --fee-payer <KEYPAIR_PATH>

# Verify transaction
solana confirm -v <TX_SIGNATURE>
```

### Fund a PumpPortal Wallet

PumpPortal gives you a Solana address when you create a wallet via their API. Fund it like any other address:

```bash
solana transfer <PUMPPORTAL_ADDRESS> 0.1 --allow-unfunded-recipient
```

## Key Management

### Backup

```bash
# Your keypair IS the backup — copy it securely
cp ~/.config/solana/id.json ~/secure-backup/solana-keypair.json
```

### Recover from Seed Phrase

```bash
solana-keygen recover --outfile ~/.config/solana/recovered.json
# You'll be prompted to enter the seed phrase
```

### Import into Browser Wallet

The keypair file can be imported into Phantom, Solflare, etc. The first 32 bytes of the JSON array are the private key.

### Multiple Wallets

```bash
# Create named wallets
solana-keygen new --outfile ~/.config/solana/trading.json
solana-keygen new --outfile ~/.config/solana/deployer.json

# Switch between them
solana config set --keypair ~/.config/solana/trading.json

# Or specify per-command
solana balance --keypair ~/.config/solana/deployer.json
```

## Config Storage

The skill stores wallet config at:
```
~/.config/solana/          — keypair files (Solana default)
```

The Solana CLI config lives at `~/.config/solana/cli/config.yml`.

## Transaction History

```bash
solana transaction-history <ADDRESS>
```

Or via RPC:
```bash
curl -s https://api.mainnet-beta.solana.com -X POST \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"getSignaturesForAddress","params":["ADDRESS",{"limit":10}]}' \
  | python3 -m json.tool
```

## Common Pitfalls

- Airdrop only works on devnet/testnet. On mainnet you need real SOL.
- Keep at least 0.01 SOL for transaction fees (rent-exempt minimum).
- Keypair files are plaintext — chmod 600 them and don't commit to git.
- If `solana` command not found after install, run: `export PATH="$HOME/.local/share/solana/install/active_release/bin:$PATH"`
- Network matters: a balance check on devnet won't show mainnet funds. Always check `solana config get`.

## Quick Reference

```
solana-keygen new                    Create wallet
solana address                       Show your address
solana config set --url devnet       Switch to devnet
solana config set --url mainnet-beta Switch to mainnet
solana config get                    Show current config
solana airdrop 2                     Get test SOL (devnet only)
solana balance                       Check balance
solana transfer <ADDR> <AMT>         Send SOL
solana confirm -v <TX>               Verify transaction
```
