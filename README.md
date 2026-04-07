# Metaplex Skill

An [Agent Skill](https://agentskills.io) for building on [Metaplex](https://metaplex.com/) — the standard infrastructure for NFTs and tokens on Solana.

## What It Does

This skill gives your AI coding agent full knowledge of Metaplex programs, CLI commands, and SDK patterns. Instead of guessing at APIs or hallucinating flags, the agent reads the correct reference on demand and executes accurately.

## Programs Covered

| Program | What It Does | CLI | Umi SDK | Kit SDK |
|---------|-------------|-----|---------|---------|
| **Agent Registry** | On-chain agent identity, wallets, execution delegation | Yes | Yes | - |
| **Genesis** | Token launches (launchpool + bonding curve) + Raydium graduation | Yes | Yes | - |
| **Core** | Next-gen NFTs — single account, plugins, royalty enforcement | Yes | Yes | - |
| **Token Metadata** | Fungible tokens, NFTs, pNFTs, editions | Yes | Yes | Yes |
| **Bubblegum** | Compressed NFTs via Merkle trees — massive scale at minimal cost | Yes | Yes | - |
| **Candy Machine** | NFT drops with guards (allowlists, payments, limits) | Yes | Yes | - |

## Operations Supported

**CLI (`mplx`)** — direct execution, no code needed:
- Register agent identities, delegate/revoke execution, link Genesis tokens
- Launch tokens via bonding curve or launchpool (with optional agent mode)
- Create/update/burn NFTs, collections, fungible tokens
- Upload assets to Irys storage
- Configure and deploy Candy Machines with guards
- Create Merkle trees and mint compressed NFTs
- Transfer NFTs and tokens
- Check balances, airdrop SOL

**Umi SDK** — full programmatic access:
- Agent Registry: mint agents, register identity, delegate execution, set agent token
- Genesis Launch API: launchpool and bonding curve launches, agent integration, creator fees, first buy
- Bonding curve swap integration: fetch state, lifecycle helpers, quotes, slippage, execute swaps
- All CLI operations plus: fetch by owner/collection/creator, DAS API queries, delegates, lock/unlock, print editions, verify/unverify creators and collections, freeze/thaw, soulbound NFTs, plugin management

**Kit SDK** (@solana/kit) — minimal dependencies:
- Token Metadata operations: create/transfer NFTs, pNFTs, fungibles, PDAs

## How It Works

The skill uses progressive disclosure — a lightweight router (SKILL.md, ~100 lines) directs the agent to the specific reference file it needs. Only relevant content is loaded:

```
SKILL.md                         Router — loaded when skill activates
  references/cli.md              Shared CLI setup (loaded for any CLI task)
  references/cli-core.md         Core CLI commands
  references/sdk-token-metadata.md  TM SDK patterns
  references/concepts.md         Account structures, PDAs
  ... (12 reference files total)
```

## Install

**Claude Code** (via skills.sh):
```bash
npx skills add metaplex-foundation/skill
```

**Claude Code** (manual):
```bash
# Project-scoped
cp -r skills/metaplex /path/to/project/.claude/skills/metaplex

# Global
cp -r skills/metaplex ~/.claude/skills/metaplex
```

**ClawHub**:
```bash
clawhub install metaplex
```

## Compatibility

Built on the [Agent Skills specification](https://agentskills.io/specification). Works with:
- Claude Code
- Cursor
- GitHub Copilot
- Codex
- Windsurf
- Any agent that supports the Agent Skills format

## Resources

- Metaplex Docs: https://metaplex.com/docs
- Agent Skills Spec: https://agentskills.io
- Protocol Fees: https://metaplex.com/docs/protocol-fees

## License

Apache-2.0
