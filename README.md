# Agentic Skills

Reusable skills for AI coding agents. Framework-agnostic — works with Hermes Agent, Claude Code, and other agentic systems that support SKILL.md format.

## Skills

| Skill | Category | Description |
|-------|----------|-------------|
| [solana-wallet](crypto/solana-wallet/SKILL.md) | Crypto | Manage Solana wallets from the CLI — keypairs, balances, transfers, network switching |
| [meme-image-suite](content-gen/meme-image-suite/SKILL.md) | Content Gen | General-purpose static meme generation for any agent (templates + AI images, no video) |

## Structure

```
category/
  skill-name/
    SKILL.md    # Skill definition with frontmatter + instructions
```

## Usage

Each skill is a self-contained SKILL.md file with YAML frontmatter (name, description, triggers, version) followed by instructions an AI agent can follow to complete tasks.

## License

MIT
