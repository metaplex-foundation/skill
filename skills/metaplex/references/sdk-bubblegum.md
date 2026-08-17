# Bubblegum SDK Reference (Umi)

Umi SDK operations for creating and managing compressed NFTs (cNFTs).

> **Prerequisites**: Set up Umi first — see `./sdk-umi.md` for installation and basic setup.

---

## Installation

```bash
npm install @metaplex-foundation/mpl-bubblegum @metaplex-foundation/mpl-core @metaplex-foundation/umi-bundle-defaults @metaplex-foundation/digital-asset-standard-api
```

> `mpl-core` is needed because Bubblegum V2 uses Core collections.

## Setup

```typescript
import { createUmi } from '@metaplex-foundation/umi-bundle-defaults';
import { mplBubblegum } from '@metaplex-foundation/mpl-bubblegum';
import { mplCore } from '@metaplex-foundation/mpl-core';

const umi = createUmi('https://api.devnet.solana.com')
  .use(mplBubblegum())
  .use(mplCore());  // Needed for Core collection operations

// Add identity — see sdk-umi.md "Node.js / Scripts" section for keypair loading
```

---

## Create Tree

```typescript
import { createTreeV2 } from '@metaplex-foundation/mpl-bubblegum';
import { generateSigner } from '@metaplex-foundation/umi';

const merkleTree = generateSigner(umi);

await createTreeV2(umi, {
  merkleTree,
  maxDepth: 14,
  maxBufferSize: 64,
  canopyDepth: 8,
}).sendAndConfirm(umi);
```

## Mint cNFT

> **Royalties**: Leaf `sellerFeeBasisPoints` is **informational for payment** — Bubblegum does not escrow royalty SOL on transfer. **Enforcement** is via the Core collection `Royalties` plugin rule set (`ProgramAllowList` / `ProgramDenyList`) so sales go through compliant marketplaces. See `./sdk-core.md` "Add Plugin to Collection".

```typescript
import { mintV2 } from '@metaplex-foundation/mpl-bubblegum';
import { none } from '@metaplex-foundation/umi';

const { signature } = await mintV2(umi, {
  leafOwner: umi.identity.publicKey,
  merkleTree: merkleTree.publicKey,
  metadata: {
    name: 'My cNFT',
    uri: 'https://arweave.net/xxx',
    sellerFeeBasisPoints: 550,
    collection: none(),
    creators: [],
  },
}).sendAndConfirm(umi);
```

## Mint cNFT into Collection

```typescript
import { mintV2 } from '@metaplex-foundation/mpl-bubblegum';
import { some } from '@metaplex-foundation/umi';

await mintV2(umi, {
  leafOwner: umi.identity.publicKey,
  merkleTree: merkleTree.publicKey,
  collectionMint: collectionAddress,
  metadata: {
    name: 'My cNFT',
    uri: 'https://arweave.net/xxx',
    sellerFeeBasisPoints: 550,
    collection: some({ key: collectionAddress, verified: false }),
    creators: [],
  },
}).sendAndConfirm(umi);
```

Note: The `collectionMint` param triggers on-chain collection verification. The `collection.verified` field in metadata is set to `false` — the program sets it to `true` during minting.

## Mint with Inherited Royalties (Bubblegum V2)

When minting into a Core collection that has a `Royalties` plugin, omit `sellerFeeBasisPoints` (or pass `65535` / `SELLER_FEE_BASIS_POINTS_INHERIT`) and keep `creators: []`. The leaf stores the inherit sentinel; effective rate and payees live on the collection plugin.

```typescript
import { mintV2 } from '@metaplex-foundation/mpl-bubblegum';
import { some } from '@metaplex-foundation/umi';

// Collection must have BubblegumV2 + Royalties plugins (see ./sdk-core.md)
await mintV2(umi, {
  collectionAuthority: umi.identity,
  leafOwner: umi.identity.publicKey,
  merkleTree: merkleTree.publicKey,
  coreCollection: collectionAddress,
  metadata: {
    name: 'Inherited SFBP cNFT',
    uri: 'https://arweave.net/xxx',
    // sellerFeeBasisPoints omitted → leaf stores 65535
    collection: some(collectionAddress),
    creators: [], // required empty when inheriting
  },
}).sendAndConfirm(umi);
```

### Reading inherited royalties from DAS

DAS puts **collection-resolved** values on main fields and **leaf** values on `_raw`:

| Use | Fields |
|-----|--------|
| Display / payout UI | `royalty.basis_points`, `royalty.percent`, `creators` |
| Hashing / proofs / write ix | `royalty.basis_points_raw`, `creators_raw` |
| Detect inherit | `royalty.inherited === true` or `basis_points_raw === 65535` (also `assetWithProof.inherited`) |

```typescript
import {
  isInheritedSfbpRoyalty,
  getRawSellerFeeBasisPoints,
  getResolvedSellerFeeBasisPoints,
} from '@metaplex-foundation/digital-asset-standard-api';

const asset = await umi.rpc.getAsset(assetId);
if (isInheritedSfbpRoyalty(asset.royalty)) {
  getResolvedSellerFeeBasisPoints(asset.royalty); // e.g. 750
  getRawSellerFeeBasisPoints(asset.royalty); // 65535
  asset.creators; // collection payees
  asset.creators_raw ?? []; // leaf creators for hashing
}
```

`getAssetWithProof` returns:

| Field | Role |
|-------|------|
| `metadata` | DAS display (`MetadataArgs`) — resolved rate/payees when inherited |
| `currentMetadata` | Leaf-canonical `MetadataArgsV2Args` for write instructions (sentinel `65535` when inherited) |
| `sellerFeeBasisPointsRaw` / `creatorsRaw` | Optional DAS `_raw` siblings |
| `inherited` | Sugar for inherit detection |

Spread `...assetWithProof` into writes so `currentMetadata` is used. Instructions that take a leaf `metadata` arg (`setCollectionV2`, `verifyCreatorV2`, …) must pass `assetWithProof.currentMetadata` — do **not** pass display `metadata`. Use `asCurrentMetadata` / `asCurrentMetadataV2` only when building leaf metadata outside that helper.

> **Outdated DAS / marketplaces**: Inherited royalties need a DAS indexer that resolves collection rates onto the main fields. On an outdated endpoint, `getAsset` returns the leaf as-is (`basis_points` ≈ `65535`, `creators: []`, no `_raw` / `inherited`). Marketplaces that only trust those DAS fields for payouts may pay creators **nothing**. Prefer an upgraded DAS or read the Core collection Royalties plugin directly. Transfer allow/deny lists (`ProgramAllowList` / `ProgramDenyList`) are separate from payment — Bubblegum does not escrow royalties on transfer.

Docs: https://metaplex.com/docs/smart-contracts/bubblegum-v2/reading-inherited-royalties

## Get Asset ID from Mint Transaction

Use the `signature` returned by `sendAndConfirm` to extract the asset ID:

```typescript
import { parseLeafFromMintV2Transaction } from '@metaplex-foundation/mpl-bubblegum';

// signature comes from: const { signature } = await mintV2(umi, {...}).sendAndConfirm(umi);
const leaf = await parseLeafFromMintV2Transaction(umi, signature);
const assetId = leaf.id;
```

## Update cNFT

> Requires `dasApi()` plugin — all mutation operations need the Merkle proof from DAS API.

```typescript
import { getAssetWithProof, updateMetadataV2, UpdateArgsArgs } from '@metaplex-foundation/mpl-bubblegum';
import { some } from '@metaplex-foundation/umi';

const assetWithProof = await getAssetWithProof(umi, assetId, { truncateCanopy: true });

const updateArgs: UpdateArgsArgs = {
  name: some('Updated Name'),
  uri: some('https://arweave.net/new-uri'),
};

// Spread includes currentMetadata (leaf-canonical). Do not pass metadata here.
await updateMetadataV2(umi, {
  ...assetWithProof,
  authority: treeAuthority,
  updateArgs,
}).sendAndConfirm(umi);
```

## Burn cNFT

```typescript
import { getAssetWithProof, burnV2 } from '@metaplex-foundation/mpl-bubblegum';

const assetWithProof = await getAssetWithProof(umi, assetId, { truncateCanopy: true });

await burnV2(umi, {
  ...assetWithProof,
  authority: leafOwner,   // Owner or burn delegate
}).sendAndConfirm(umi);
```

## Transfer cNFT

> Requires `dasApi()` plugin — `getAssetWithProof` fetches the Merkle proof via DAS API. See "Fetch cNFTs" section below for setup.

```typescript
import { getAssetWithProof, transferV2 } from '@metaplex-foundation/mpl-bubblegum';

// Fetch proof from DAS API
const assetWithProof = await getAssetWithProof(umi, assetId, {
  truncateCanopy: true,
});

await transferV2(umi, {
  ...assetWithProof,
  authority: leafOwner,
  newLeafOwner: recipient.publicKey,
}).sendAndConfirm(umi);
```

## Freeze / Thaw

### Delegate and Freeze (Single Transaction)

```typescript
import { delegateAndFreezeV2, getAssetWithProof } from '@metaplex-foundation/mpl-bubblegum';

const assetWithProof = await getAssetWithProof(umi, assetId, { truncateCanopy: true });

await delegateAndFreezeV2(umi, {
  ...assetWithProof,
  authority: leafOwner,
  delegate: delegateAddress,
}).sendAndConfirm(umi);
```

### Freeze / Thaw Separately

```typescript
import { freezeV2, thawV2 } from '@metaplex-foundation/mpl-bubblegum';

const assetWithProof = await getAssetWithProof(umi, assetId, { truncateCanopy: true });

// Freeze (by delegate)
await freezeV2(umi, {
  ...assetWithProof,
  authority: delegateSigner,
}).sendAndConfirm(umi);

// Thaw (by delegate)
const frozenProof = await getAssetWithProof(umi, assetId, { truncateCanopy: true });
await thawV2(umi, {
  ...frozenProof,
  authority: delegateSigner,
}).sendAndConfirm(umi);
```

### Thaw and Revoke Delegate

```typescript
import { thawAndRevokeV2 } from '@metaplex-foundation/mpl-bubblegum';

const assetWithProof = await getAssetWithProof(umi, assetId, { truncateCanopy: true });

await thawAndRevokeV2(umi, {
  ...assetWithProof,
  authority: delegateSigner,
}).sendAndConfirm(umi);
```

## Delegate

### Approve/Revoke Leaf Delegate

```typescript
import { delegateV2, getAssetWithProof } from '@metaplex-foundation/mpl-bubblegum';

const assetWithProof = await getAssetWithProof(umi, assetId, { truncateCanopy: true });

// Approve delegate
await delegateV2(umi, {
  ...assetWithProof,
  authority: leafOwner,
  previousLeafDelegate: leafOwner.publicKey,
  newLeafDelegate: delegateAddress,
}).sendAndConfirm(umi);
```

### Set Tree Delegate

```typescript
import { setTreeDelegate } from '@metaplex-foundation/mpl-bubblegum';

// Approve tree delegate (can mint on your behalf)
await setTreeDelegate(umi, {
  merkleTree: treeAddress,
  treeCreatorOrTreeDelegate: treeCreator,
  newTreeDelegate: delegateAddress,
}).sendAndConfirm(umi);
```

## Make cNFT Non-Transferable

```typescript
import { setNonTransferableV2 } from '@metaplex-foundation/mpl-bubblegum';

// Requires collection authority. Makes ALL cNFTs in this collection non-transferable.
await setNonTransferableV2(umi, {
  collectionMint: collectionAddress,
  collectionAuthority: collectionAuthoritySigner,
}).sendAndConfirm(umi);
```

## Verify / Unverify Creator

```typescript
import { verifyCreatorV2, unverifyCreatorV2, getAssetWithProof } from '@metaplex-foundation/mpl-bubblegum';

const assetWithProof = await getAssetWithProof(umi, assetId, { truncateCanopy: true });

// Verify (signer must be the creator)
await verifyCreatorV2(umi, {
  ...assetWithProof,
  metadata: assetWithProof.currentMetadata,
  authority: creatorSigner,
  creator: creatorSigner.publicKey,
}).sendAndConfirm(umi);

// Unverify
const updatedProof = await getAssetWithProof(umi, assetId, { truncateCanopy: true });
await unverifyCreatorV2(umi, {
  ...updatedProof,
  metadata: updatedProof.currentMetadata,
  authority: creatorSigner,
  creator: creatorSigner.publicKey,
}).sendAndConfirm(umi);
```

## Fetch cNFTs (DAS API)

> Requires a DAS-compatible RPC (e.g., Helius, Triton, QuickNode) and the `dasApi()` plugin. See `./sdk-umi.md` DAS section.

```typescript
import { createUmi } from '@metaplex-foundation/umi-bundle-defaults';
import { mplBubblegum } from '@metaplex-foundation/mpl-bubblegum';
import { mplCore } from '@metaplex-foundation/mpl-core';
import { dasApi } from '@metaplex-foundation/digital-asset-standard-api';

// Add DAS plugin (use DAS-compatible RPC)
const umi = createUmi('https://mainnet.helius-rpc.com/?api-key=YOUR_KEY')
  .use(mplBubblegum())
  .use(mplCore())
  .use(dasApi());

// Single asset
const asset = await umi.rpc.getAsset(assetId);

// By owner
const assets = await umi.rpc.getAssetsByOwner({ owner: walletAddress });

// By collection
const collectionAssets = await umi.rpc.getAssetsByCollection({
  collection: collectionAddress,
});
```

---

## Bubblegum V2 Features

### Core Collections Integration

V2 cNFTs use **MPL Core Collections** (not Token Metadata collections). Create collections with `createCollection` from `@metaplex-foundation/mpl-core` (see `./sdk-core.md`). This enables:
- Royalty enforcement via Core plugins (e.g., `ProgramDenyList`)
- Collection-level operations and delegates

**Royalties for cNFTs**: Leaf `sellerFeeBasisPoints` is informational for payment. **Enforcement** is the Core collection `Royalties` plugin rule set (`ProgramAllowList` / `ProgramDenyList`) — Bubblegum does not pay creators on transfer. For inherited SFBP (`65535` on the leaf), DAS resolves collection rate/payees onto main fields and exposes leaf values on `_raw`; `getAssetWithProof` puts leaf values on `currentMetadata` (see "Mint with Inherited Royalties" above).

### Soulbound NFTs

Non-transferable cNFTs permanently bound to an owner. Two approaches:
1. **`PermanentFreezeDelegate`** plugin on Core collection (see `./sdk-core.md` Soulbound section)
2. **`setNonTransferableV2`** — marks entire collection as non-transferable (see above)

Use cases: credentials, proof of attendance, identity tokens.

### Freeze/Thaw

- **Asset-level freeze** by owner delegate
- **Collection-level freeze** by permanent freeze delegate
- Useful for vesting, event gating, or conditional transfers

### Permanent Delegates

- `PermanentTransferDelegate` — transfer without owner signature
- `PermanentBurnDelegate` — burn without owner signature
- `PermanentFreezeDelegate` — freeze/thaw for soulbound and collection control

These are set on the Core collection and apply to all cNFTs in it.

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
- Reading inherited royalties: https://metaplex.com/docs/smart-contracts/bubblegum-v2/reading-inherited-royalties
- Burn cNFTs: https://metaplex.com/docs/smart-contracts/bubblegum-v2/burn-cnfts
- Fetch cNFTs: https://metaplex.com/docs/smart-contracts/bubblegum-v2/fetch-cnfts
- Freeze cNFTs: https://metaplex.com/docs/smart-contracts/bubblegum-v2/freeze-cnfts
- Collections: https://metaplex.com/docs/smart-contracts/bubblegum-v2/collections
- Delegate cNFTs: https://metaplex.com/docs/smart-contracts/bubblegum-v2/delegate-cnfts
- Verify creators: https://metaplex.com/docs/smart-contracts/bubblegum-v2/verify-creators
- JavaScript SDK: https://metaplex.com/docs/smart-contracts/bubblegum-v2/sdk/javascript
- DAS API: https://metaplex.com/docs/dev-tools/das-api
