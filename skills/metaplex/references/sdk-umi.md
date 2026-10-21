# Metaplex Umi SDK Reference

Umi is Metaplex's modular JavaScript framework for Solana program clients.

## Packages

| Package | Purpose |
|---------|---------|
| `@metaplex-foundation/umi-bundle-defaults` | Base Umi setup |
| `@metaplex-foundation/mpl-token-metadata` | Token Metadata (NFTs, fungibles) |
| `@metaplex-foundation/mpl-core` | Core NFT standard |
| `@metaplex-foundation/mpl-bubblegum` | Compressed NFTs (Bubblegum) |
| `@metaplex-foundation/umi-uploader-irys` | Upload to Arweave via Irys |
| `@metaplex-foundation/umi-signer-wallet-adapters` | Wallet adapter integration |
| `@metaplex-foundation/mpl-toolbox` | SPL token helpers (transfers, compute budget) |
| `@metaplex-foundation/digital-asset-standard-api` | DAS API (asset queries) |
| `@metaplex-foundation/mpl-core-das` | Core-typed DAS helpers (`das.*`) |

## Installation

```bash
npm install @metaplex-foundation/umi-bundle-defaults \
  @metaplex-foundation/mpl-core \
  @metaplex-foundation/mpl-token-metadata \
  @metaplex-foundation/mpl-toolbox \
  @metaplex-foundation/digital-asset-standard-api \
  @metaplex-foundation/mpl-core-das
# Optional — add if needed:
#   @metaplex-foundation/umi-uploader-irys (for uploading files)
#   @metaplex-foundation/umi-signer-wallet-adapters (for browser wallet integration)
```

The default bundle also needs `@solana/web3.js`. Umi 1.6.0+ requires `@solana/web3.js@^1.99.0`, because that's the first release that understands V1 transactions.

---

## Basic Setup

### Browser / Wallet Adapter

```typescript
import { createUmi } from '@metaplex-foundation/umi-bundle-defaults';
import { mplCore } from '@metaplex-foundation/mpl-core';
import { mplTokenMetadata } from '@metaplex-foundation/mpl-token-metadata';
import { walletAdapterIdentity } from '@metaplex-foundation/umi-signer-wallet-adapters';
import { irysUploader } from '@metaplex-foundation/umi-uploader-irys';

const umi = createUmi('https://api.devnet.solana.com')
  .use(mplCore())
  .use(mplTokenMetadata())
  .use(walletAdapterIdentity(wallet))
  .use(irysUploader());
```

### Node.js / Scripts (Keypair from File)

```typescript
import { createUmi } from '@metaplex-foundation/umi-bundle-defaults';
import { keypairIdentity } from '@metaplex-foundation/umi';
import { mplCore } from '@metaplex-foundation/mpl-core';
import { mplTokenMetadata } from '@metaplex-foundation/mpl-token-metadata';
import fs from 'fs';

// Create Umi first, then load keypair using its eddsa helper
const umi = createUmi('https://api.devnet.solana.com')
  .use(mplCore())
  .use(mplTokenMetadata());

const secretKey = JSON.parse(fs.readFileSync('/path/to/keypair.json', 'utf-8'));
const keypair = umi.eddsa.createKeypairFromSecretKey(new Uint8Array(secretKey));
umi.use(keypairIdentity(keypair));
```

---

## Program-Specific SDK Guides

| Program | Detail File |
|---------|-------------|
| Core NFTs | `./sdk-core.md` |
| Token Metadata | `./sdk-token-metadata.md` |
| Bubblegum (compressed NFTs) | `./sdk-bubblegum.md` |
| DAS API (asset queries) | `./sdk-das.md` |
| Genesis (token launches) | `./sdk-genesis.md` |
| Token Metadata with Kit | `./sdk-token-metadata-kit.md` |

---

## Transaction Patterns

### Chaining Instructions

```typescript
await createV1(umi, { ...args })
  .add(anotherInstruction(umi, { ...args }))
  .sendAndConfirm(umi);
```

### Compute Budget (V0 — default)

```typescript
import { setComputeUnitLimit, setComputeUnitPrice } from '@metaplex-foundation/mpl-toolbox';

await createV1(umi, { ...args })
  .prepend(setComputeUnitLimit(umi, { units: 200_000 }))
  .prepend(setComputeUnitPrice(umi, { microLamports: 5000 }))
  .sendAndConfirm(umi);
```

### V1 Transactions (SIMD-0385, Umi >= 1.6.0)

V1 transactions can be up to **4,096 bytes** (V0 tops out at 1,232). The compute budget lives in the message as a `TransactionV1Config` rather than in ComputeBudget instructions, and **address lookup tables are not supported**. Umi still **defaults to V0**, so V1 is opt-in.

```typescript
import { lamports } from '@metaplex-foundation/umi';

await createV1(umi, { ...args })
  .useV1()
  .setTransactionConfig({
    computeUnitLimit: 600_000,
    priorityFee: lamports(600), // TOTAL fee, not a per-CU price
  })
  .sendAndConfirm(umi);
```

**App-wide opt-in:** `createUmi(endpoint, { defaultTransactionVersion: 1 })`. Without the bundle, use `umi.use(web3JsTransactionFactory({ defaultTransactionVersion: 1 }))`. Builders returned by Metaplex program libraries pick up the default too. A builder that calls `.useV0()` keeps V0.

**Rules (Umi throws before sending if you break them):**
- Do not add `setComputeUnitLimit` / `setComputeUnitPrice` (or any ComputeBudget instruction) to a V1 builder. Set them with `setTransactionConfig()` instead.
- A V1 builder can't call `setAddressLookupTables`. A transaction that needs an ALT has to stay on V0 (`.useV0()`).
- Calling `setTransactionConfig()` on a legacy or V0 builder throws.
- `setTransactionConfig()` replaces the whole config, so pass every field in one call.

**Converting a V0 price to V1:** `priorityFee` = `ceil(computeUnitLimit × microLamports / 1_000_000)` lamports. For example, 600,000 CU × 1,000 µL = `lamports(600)`.

**`TransactionV1Config` fields:**

| Field | Range | Builder default if omitted |
|-------|-------|----------------------------|
| `computeUnitLimit` | integer 0–1,400,000 | `min(200_000 × ixCount, 1_400_000)` |
| `priorityFee` | `SolAmount` (total) | none |
| `loadedAccountsDataSizeLimit` | ≤ 64 MiB | 64 MiB |
| `heapSize` | 32,768–262,144, a multiple of 1,024 | none |

**Direct `umi.transactions.create()`:** `version` is now required on every input, including `version: 0` for V0. For V1, set `version: 1` and pass explicit nonzero `transactionConfig.computeUnitLimit` and `loadedAccountsDataSizeLimit`. The builder defaults above are not applied here, and the runtime treats an unset limit as zero.

**Wallets:** the connected wallet has to support transaction version `1`. Umi can serialize V1 for wallet adapters, but it can't make a wallet sign it. Don't set V1 as the app default until every wallet path you support has been tested.

### Safe Fetch (returns null if not found)

```typescript
import { safeFetchMetadata } from '@metaplex-foundation/mpl-token-metadata';

const metadata = await safeFetchMetadata(umi, metadataPda);
if (metadata) {
  // exists
}
```

Use `safeFetch*` variants when the account may not exist (e.g., checking if a mint has metadata). Regular `fetch*` throws if the account is missing.

---

## Uploading

```typescript
import { irysUploader } from '@metaplex-foundation/umi-uploader-irys';

const umi = createUmi(rpcEndpoint).use(irysUploader());

// Upload file
const [imageUri] = await umi.uploader.upload([imageFile]);

// Upload JSON — follow the full NFT metadata schema (see metadata-json.md)
const metadataUri = await umi.uploader.uploadJson({
  name: 'My NFT',
  description: 'Description',           // optional
  image: imageUri,
  external_url: 'https://yourproject.com', // optional but recommended
  animation_url: animationUri,           // optional, omit if not applicable
  attributes: [{ trait_type: 'Background', value: 'Blue' }], // optional but recommended
  properties: {
    files: [{ uri: imageUri, type: 'image/png' }],
    category: 'image',
  },
});
```

---

## DAS API (Asset Queries)

> Full method reference: `./sdk-das.md`.
>
> **Important**: DAS requires a DAS-compatible RPC (e.g., Helius, Triton, QuickNode). Public Solana RPC does **not** support DAS.

```typescript
import { dasApi } from '@metaplex-foundation/digital-asset-standard-api';
import { das } from '@metaplex-foundation/mpl-core-das';

const umi = createUmi('https://mainnet.helius-rpc.com/?api-key=YOUR_KEY').use(dasApi());

// Core-typed (preferred for Core apps)
const coreAssets = await das.getAssetsByOwner(umi, { owner: walletAddress });
const byCollection = await das.getAssetsByCollection(umi, {
  collection: collectionAddress,
});

// Base DAS (multi-standard / raw shapes)
const asset = await umi.rpc.getAsset(assetId);
const byGroup = await umi.rpc.getAssetsByGroup({
  groupKey: 'collection',
  groupValue: collectionAddress,
});
const results = await umi.rpc.searchAssets({
  owner: walletAddress,
  burnt: false,
});
```

### Inherited Bubblegum V2 royalties

When `asset.royalty.inherited` is true (or `royalty.basis_points_raw === 65535`):

- Display / payout: `royalty.basis_points`, `royalty.percent`, `creators`
- Leaf hashing: `royalty.basis_points_raw`, `creators_raw`
- Helpers: `isInheritedSfbpRoyalty`, `getRawSellerFeeBasisPoints`, `getResolvedSellerFeeBasisPoints` from this package
- Bubblegum writes (`mpl-bubblegum` >= 6.0.0): spread `getAssetWithProof(...)` and pass **leaf** data — `currentMetadata: assetWithProof.currentMetadata` on `updateMetadataV2`; `metadata: assetWithProof.currentMetadata` on `setCollectionV2` / `verifyCreatorV2` / `unverifyCreatorV2`. Do not pass display `metadata` into those args when inherited. Core collection updates also need `authority: collectionAuthority` and `coreCollection`. On 5.x, rebuild leaf metadata from `rpcAsset.royalty.basis_points_raw` / `creators_raw`.

See `./sdk-bubblegum.md` "Mint with Inherited Royalties" and https://metaplex.com/docs/smart-contracts/bubblegum-v2/reading-inherited-royalties
> There is no `umi.rpc.getAssetsByCollection`. Use `das.getAssetsByCollection` (Core) or `umi.rpc.getAssetsByGroup({ groupKey: 'collection', … })` (base DAS).

---

## Error Handling

```typescript
try {
  await instruction.sendAndConfirm(umi);
} catch (error) {
  if (error.name === 'TokenMetadataError') {
    console.error('Token Metadata Error:', error.message);
  }
  throw error;
}
```

## For more info

Consult these Metaplex docs when you need deeper detail than this reference provides:

- Umi overview: https://metaplex.com/docs/dev-tools/umi
- Getting started: https://metaplex.com/docs/dev-tools/umi/getting-started
- Plugins: https://metaplex.com/docs/dev-tools/umi/plugins
- Metaplex Umi plugins: https://metaplex.com/docs/dev-tools/umi/metaplex-umi-plugins
- Public keys and signers: https://metaplex.com/docs/dev-tools/umi/public-keys-and-signers
- Transactions: https://metaplex.com/docs/dev-tools/umi/transactions
- Migrating V0 → V1 transactions: https://metaplex.com/docs/dev-tools/umi/guides/migrate-to-transaction-v1
- Optimal CU / priority fees (V1): https://metaplex.com/docs/dev-tools/umi/guides/optimal-transactions-with-compute-units-and-priority-fees
- Storage: https://metaplex.com/docs/dev-tools/umi/storage
- RPC: https://metaplex.com/docs/dev-tools/umi/rpc
- DAS API: https://metaplex.com/docs/dev-tools/das-api
- DAS getting started: https://metaplex.com/docs/dev-tools/das-api/getting-started
