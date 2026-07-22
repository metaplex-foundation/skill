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

### Compute Budget

```typescript
import { setComputeUnitLimit, setComputeUnitPrice } from '@metaplex-foundation/mpl-toolbox';

await createV1(umi, { ...args })
  .prepend(setComputeUnitLimit(umi, { units: 200_000 }))
  .prepend(setComputeUnitPrice(umi, { microLamports: 5000 }))
  .sendAndConfirm(umi);
```

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
- Storage: https://metaplex.com/docs/dev-tools/umi/storage
- RPC: https://metaplex.com/docs/dev-tools/umi/rpc
- DAS API: https://metaplex.com/docs/dev-tools/das-api
- DAS getting started: https://metaplex.com/docs/dev-tools/das-api/getting-started
