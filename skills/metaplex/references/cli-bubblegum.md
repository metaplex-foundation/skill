# Bubblegum CLI Reference

Commands for creating and managing compressed NFTs (cNFTs) via the `mplx` CLI.

> **Prerequisites**: CLI must be configured (RPC, keypair, funded wallet). If not yet verified this session, see `./cli-initial-setup.md`.
>
> **Batch limit**: CLI is practical for up to ~100 cNFTs. For larger mints (thousands+), use the Umi SDK (`./sdk-bubblegum.md`) or Candy Machine instead.

---

## When to Use Bubblegum

| Use Bubblegum | Use Core Instead |
|---------------|------------------|
| Minting thousands+ of NFTs | Small collections (< 1000) |
| Lowest possible cost per NFT | Need per-asset Core plugins (freeze, attributes) |
| Airdrops, loyalty programs, tickets | Marketplace-focused projects |
| Proof of attendance, credentials | Simple 1/1 NFTs |

---

## Key Concepts

### Merkle Trees

Bubblegum stores NFTs as leaves in a concurrent Merkle tree. Only the **Merkle root** (a single hash) lives on-chain — the actual NFT data is stored in transactions and indexed by RPC providers via the **DAS API**.

Each tree has two on-chain accounts:
- **Merkle Tree Account** — stores the tree structure, change log, and canopy
- **TreeConfigV2 Account** — PDA tracking tree creator, delegate, capacity, and mint count

### Tree Sizing

| cNFTs | Tree Depth | Canopy Depth | Buffer Size | Tree Cost | Cost per cNFT |
|-------|-----------|-------------|-------------|-----------|---------------|
| 16,384 | 14 | 8 | 64 | ~0.34 SOL | ~0.00002 SOL |
| 65,536 | 16 | 10 | 64 | ~0.71 SOL | ~0.00001 SOL |
| 262,144 | 18 | 12 | 64 | ~2.10 SOL | ~0.00001 SOL |
| 1,048,576 | 20 | 13 | 1024 | ~8.50 SOL | ~0.000008 SOL |
| 16,777,216 | 24 | 15 | 2048 | ~26.12 SOL | ~0.000002 SOL |
| 1,073,741,824 | 30 | 17 | 2048 | ~72.65 SOL | ~0.00000005 SOL |

- **Tree Depth** — determines max capacity (2^depth leaves)
- **Max Buffer Size** — concurrency limit (parallel mints in same block)
- **Canopy Depth** — cached upper tree nodes; higher = smaller proofs, better composability, but higher rent

### Proofs and DAS API

Operations on existing cNFTs (transfer, burn, update) require a **Merkle proof** to verify the leaf. Proofs are fetched from RPC providers via the DAS API — not from on-chain data.

---

## Commands

```bash
# Tree management
mplx bg tree create --wizard                                              # Interactive (recommended)
mplx bg tree create --maxDepth <N> --maxBufferSize <N> --canopyDepth <N>  # Manual
mplx bg tree list                                                         # List trees created via --wizard only

# cNFT operations
mplx bg nft create --wizard                                               # Interactive
mplx bg nft create <TREE> --name <NAME> --uri <URI>                       # With pre-uploaded metadata
mplx bg nft create <TREE> --name <NAME> --image <PATH> --description <DESC>  # With local files
mplx bg nft create <TREE> --name <NAME> --uri <URI> --collection <ADDR>   # Auto-inherit if collection has Royalties
mplx bg nft create <TREE> --name <NAME> --uri <URI> --collection <ADDR> --inherit-royalties
mplx bg nft create <TREE> --name <NAME> --uri <URI> --collection <ADDR> --royalties 7.5 --creator <ADDR>:60 --creator <ADDR2>:40
mplx bg nft fetch <ASSETID>
mplx bg nft transfer <ASSETID> <NEWOWNER>
mplx bg nft burn <ASSETID>
mplx bg nft update <ASSETID> --name <NAME>

# Collection for cNFTs (Core + BubblegumV2; --royalties adds the Royalties plugin)
mplx bg collection create --name <NAME> --uri <URI>
mplx bg collection create --name <NAME> --uri <URI> --royalties <PERCENT>
```

---

## CLI Workflow

```bash
# 1. Create a tree (wizard guides through depth/buffer/canopy selection)
mplx bg tree create --wizard

# 2. (Optional) Bubblegum-ready Core collection. Pass --royalties so mints can inherit.
mplx bg collection create --name "My Collection" --uri "https://arweave.net/xxx" --royalties 5

# 3. Mint into the tree. Omit --royalties/--creator (and JSON seller_fee_basis_points) to auto-inherit.
mplx bg nft create <TREE_NAME_OR_ADDRESS> --name "My cNFT" --image ./image.png --description "A compressed NFT" --collection <COL>

# 4. Transfer (requires DAS-compatible RPC, not available on localnet)
mplx bg nft transfer <ASSETID> <RECIPIENT_ADDRESS>
```

For batch minting, chain commands:
```bash
mplx bg nft create <TREE> --name "cNFT #1" --uri "<URI_1>" && \
mplx bg nft create <TREE> --name "cNFT #2" --uri "<URI_2>" && \
mplx bg nft create <TREE> --name "cNFT #3" --uri "<URI_3>"
```

---

## Inherited royalties (Bubblegum V2)

Requires the CLI inherit-royalties release (`feat/bgumInheritSfbp` / [cli#136](https://github.com/metaplex-foundation/cli/pull/136)). Older published CLI only has `--royalties` and mints an explicit leaf rate (default 0% + payer @ 100%).

Leaf `sellerFeeBasisPoints` `65535` (`0xffff`) plus empty leaf creators means **inherit from the Core collection `Royalties` plugin**. Effective rate and payees are collection-level; DAS display fields resolve them automatically.

**Create a collection that can inherit:**

```bash
mplx bg collection create --name "Col" --uri "https://arweave.net/xxx" --royalties 5
```

`--royalties` on `bg collection create` is a whole-number percent (adds the Royalties plugin). A plain `core collection create` is not enough — the collection needs `BubblegumV2` (and Royalties, to inherit).

**Mint rules** (`mplx bg nft create`):

| Intent | Flags |
|--------|--------|
| Auto-inherit | `--collection <COL>` and omit `--royalties`, `--creator`, and JSON `seller_fee_basis_points` |
| Force inherit | `--collection <COL> --inherit-royalties` |
| Explicit leaf rate | `--royalties <0-100>` (decimals ok, e.g. `7.5`). Opts out of inherit |
| Explicit splits | `--creator <ADDR>:<share>` (repeatable; shares sum to 100). Opts out of inherit even without `--royalties` (leaf rate is then `0%`) |

`--inherit-royalties` requires `--collection` with a Royalties plugin and cannot be combined with `--royalties` or `--creator`. Default explicit creators (no inherit) are payer @ 100%.

```bash
# Auto-inherit
mplx bg nft create <TREE> --name "cNFT" --uri "<URI>" --collection <COL>

# Force inherit
mplx bg nft create <TREE> --name "cNFT" --uri "<URI>" --collection <COL> --inherit-royalties

# Explicit 7.5% + 60/40 splits
mplx bg nft create <TREE> --name "cNFT" --uri "<URI>" --collection <COL> \
  --royalties 7.5 --creator <ADDR1>:60 --creator <ADDR2>:40
```

**Fetch** (`mplx bg nft fetch`): DAS `Royalty` / `Creators (display)` are the effective collection values when inherited. `Inherited: Yes (leaf sentinel 65535)` and `Creators (leaf / raw)` are diagnostic — only shown when DAS exposes `_raw` / `inherited`. Do not invent a sentinel if those fields are missing. Fetch needs DAS (not localnet).

**Update** (`mplx bg nft update`, same CLI release): rebuilds leaf-canonical metadata (`65535` + empty creators) from SDK `currentMetadata` or DAS `royalty.basis_points_raw` / `creators_raw`. Older CLI passes display `metadata` and inherited updates fail the hash check. Do not write the display royalty % into the leaf.

For SDK mint/read/write (`currentMetadata` vs display `metadata`), see `./sdk-bubblegum.md` "Mint with Inherited Royalties".

---

## Localnet Limitations

Only tree creation (`mplx bg tree create`) and minting (`mplx bg nft create`) work on localhost/localnet. Operations that require DAS API -- fetch, transfer, burn, update -- do NOT work on localnet because the test validator does not support DAS.

---

## Program IDs

```
Bubblegum V1: BGUMAp9SX3uS4efGcFjPjkAQZ4cUNZhtHaMq64nrGf9D
Bubblegum V2: BGUMAp9Gq7iTEuizy4pqaxsTyUCBK68MDfK752saRPUY
```

---

## Cost Comparison

| Standard | Cost per NFT | Accounts | Best For |
|----------|-------------|----------|----------|
| **Bubblegum** | ~$0.000005 | 0 (shared tree) | Massive scale |
| **Core** | ~0.0029 SOL | 1 | Small-medium collections |
| **Token Metadata** | ~0.022 SOL | 3-4 | Fungibles, legacy |

Bubblegum is ~98% cheaper than Token Metadata and ~90% cheaper than Core at scale. The trade-off is that cNFT operations require DAS API access for proof fetching.

## For more info

Consult these Metaplex docs when you need deeper detail than this reference provides:

- Bubblegum V2 overview: https://metaplex.com/docs/smart-contracts/bubblegum-v2
- Create trees: https://metaplex.com/docs/smart-contracts/bubblegum-v2/create-trees
- Mint cNFTs: https://metaplex.com/docs/smart-contracts/bubblegum-v2/mint-cnfts
- Transfer cNFTs: https://metaplex.com/docs/smart-contracts/bubblegum-v2/transfer-cnfts
- Update cNFTs: https://metaplex.com/docs/smart-contracts/bubblegum-v2/update-cnfts
- Burn cNFTs: https://metaplex.com/docs/smart-contracts/bubblegum-v2/burn-cnfts
- Fetch cNFTs: https://metaplex.com/docs/smart-contracts/bubblegum-v2/fetch-cnfts
- Collections: https://metaplex.com/docs/smart-contracts/bubblegum-v2/collections
- Concurrent merkle trees: https://metaplex.com/docs/smart-contracts/bubblegum-v2/concurrent-merkle-trees
- CLI Bubblegum: https://metaplex.com/docs/dev-tools/cli/bubblegum
- DAS API: https://metaplex.com/docs/dev-tools/das-api
