# DAS API SDK Reference

Query Solana digital assets via the Metaplex Digital Asset Standard (DAS) API.

> **Prerequisites**: Umi setup — see `./sdk-umi.md`.
>
> **Important**: DAS requires a DAS-compatible RPC (Helius, Triton, QuickNode, Shyft, Hello Moon). The default public Solana RPC does **not** support DAS methods.

## Which package?

| Need | Package | Call style |
|------|---------|------------|
| Core assets/collections/groups as `AssetResult` / `CollectionResult` / `GroupResult` | `@metaplex-foundation/mpl-core-das` | `das.getAssetsByCollection(umi, …)` |
| Multi-standard assets, cNFT proofs, editions, token accounts, raw DAS shapes | `@metaplex-foundation/digital-asset-standard-api` | `umi.rpc.getAsset(…)` |

Always register the base plugin. Add `mpl-core-das` when reading Core data as Core types. Requires DAS client `>=2.1.0` for groups / `getGrouping` / agent filters, and `mpl-core` `>=1.9.0` for `GroupV1`.

```bash
npm install @metaplex-foundation/digital-asset-standard-api
# Core-typed helpers (recommended for Core apps):
npm install @metaplex-foundation/mpl-core-das @metaplex-foundation/mpl-core
```

```typescript
import { createUmi } from '@metaplex-foundation/umi-bundle-defaults';
import { dasApi } from '@metaplex-foundation/digital-asset-standard-api';

const umi = createUmi('https://mainnet.helius-rpc.com/?api-key=YOUR_KEY').use(dasApi());

// Helius-only param names for getNftEditions / getTokenAccounts:
// umi.use(dasApi({ heliusCompatibility: true }));
```

---

## Core helpers (`mpl-core-das`)

Preferred for Core listing. Returns `AssetResult` / `CollectionResult` / `GroupResult` (Core types + DAS `content`, and optional `is_agent` / `agent_token` / `asset_signer`) and derives collection plugins by default.

```typescript
import { publicKey } from '@metaplex-foundation/umi';
import { das } from '@metaplex-foundation/mpl-core-das';

const byOwner = await das.getAssetsByOwner(umi, { owner: wallet });
const byCollection = await das.getAssetsByCollection(umi, {
  collection: collectionAddress,
});
const byGroup = await das.getAssetsByGroup(umi, { group: groupAddress });

const asset = await das.getAsset(umi, assetAddress);
const collection = await das.getCollection(umi, collectionAddress);
const group = await das.getGroup(umi, groupAddress);

// Agent discovery
const agents = await das.searchAssets(umi, {
  isAgent: true,
  skipDerivePlugins: true,
});
// agents[i].is_agent, .agent_token, .asset_signer when indexed
```

| Helper | Description |
|--------|-------------|
| `das.getAsset` / `getCollection` / `getGroup` | Single account by pubkey |
| `das.getAssetsByOwner` / `ByAuthority` / `ByCollection` | List Core assets |
| `das.getAssetsByGroup` | Members of an mpl-core `GroupV1` — assets, collections, or nested groups (`groupKey: 'group'`) |
| `das.getGrouping` | Group summary (`group_name`, `group_size`) without listing members |
| `das.searchAssets` / `searchCollections` / `searchGroups` | Filtered search |
| `das.getCollectionsByUpdateAuthority` / `getGroupsByUpdateAuthority` | By update authority |
| `das.dasAssetsToCoreAssets` | Raw `MplCoreAsset` items → `AssetResult[]` |
| `das.dasAssetToCoreCollection` | Raw `MplCoreCollection` → `CollectionResult` |
| `das.dasAssetToCoreGroup` | Raw `MplCoreGroup` → `GroupResult` |

Options: `skipDerivePlugins: true` to skip collection plugin inheritance. Only `displayOptions.showCollectionMetadata` is supported.

> `GroupResult` membership vectors may be empty from DAS — use `fetchGroupV1` from `mpl-core` for authoritative on-chain membership.

> Do **not** call `umi.rpc.getAssetsByCollection` — that method does not exist on base DAS. Use `das.getAssetsByCollection` (Core) or `umi.rpc.getAssetsByGroup({ groupKey: 'collection', … })` (base).

---

## Base DAS methods (`umi.rpc`)

| Method | Description |
|--------|-------------|
| `getAsset` | Single asset by ID |
| `getAssets` | Multiple assets by IDs |
| `getAssetProof` / `getAssetProofs` | Merkle proofs (cNFTs) |
| `getAssetsByOwner` | Assets owned by a wallet |
| `getAssetsByAuthority` | Assets by authority |
| `getAssetsByCreator` | Assets by creator (`onlyVerified`) |
| `getAssetsByGroup` | By grouping key/value (`collection` or `group`) |
| `getGrouping` | Grouping metadata (name + size) |
| `getAssetSignatures` | Transaction signatures for a compressed asset |
| `getNftEditions` | Print editions for a master edition mint |
| `getTokenAccounts` | Token accounts by owner and/or mint |
| `searchAssets` | Flexible multi-filter search |

### Common examples

```typescript
import { publicKey } from '@metaplex-foundation/umi';

const asset = await umi.rpc.getAsset(assetId);
// or with display options:
const asset2 = await umi.rpc.getAsset({
  assetId,
  displayOptions: { showCollectionMetadata: true },
});

const batch = await umi.rpc.getAssets([assetId1, assetId2]);

const byOwner = await umi.rpc.getAssetsByOwner({
  owner: wallet,
  limit: 100,
  page: 1,
});

// Collections (TM or Core) — NOT getAssetsByCollection
const byCollection = await umi.rpc.getAssetsByGroup({
  groupKey: 'collection',
  groupValue: collectionAddress,
});

// mpl-core GroupV1 members
const byCoreGroup = await umi.rpc.getAssetsByGroup({
  groupKey: 'group',
  groupValue: groupAddress,
});

const grouping = await umi.rpc.getGrouping({
  groupKey: 'group',
  groupValue: groupAddress,
});

const byCreator = await umi.rpc.getAssetsByCreator({
  creator: creatorAddress,
  onlyVerified: true,
});

const proof = await umi.rpc.getAssetProof(cnftAssetId);
```

### searchAssets

```typescript
const results = await umi.rpc.searchAssets({
  owner: wallet,
  burnt: false,
  compressed: true,
  interface: 'MplCoreAsset', // or MplCoreCollection, MplCoreGroup, V1_NFT, …
  limit: 50,
  page: 1,
});

// Agent filters (MPL Core only)
const agents = await umi.rpc.searchAssets({
  isAgent: true,
  agentToken: genesisMint,       // optional
  assetSigner: assetSignerPda,   // optional
  interface: 'MplCoreAsset',
});
```

Useful filters: `owner`, `creator`, `authority`, `grouping: ['collection'|'group', value]`, `delegate`, `frozen`, `compressed`, `burnt`, `jsonUri`, `name`, `tokenType`, `royaltyModel`, `isAgent`, `agentToken`, `assetSigner`, `negate`, `conditionType`.

Core response extras when present: `is_agent`, `agent_token`, `asset_signer`, plus `plugins` / `external_plugins` / `mpl_core_info`.

### Editions & token accounts

```typescript
const editions = await umi.rpc.getNftEditions({
  mintAddress: masterEditionMint,
  page: 1,
});

const tokenAccounts = await umi.rpc.getTokenAccounts({
  ownerAddress: wallet,
  // mintAddress: optionalMint,
  options: { showZeroBalance: false },
});
```

With `dasApi({ heliusCompatibility: true })`, these methods send Helius param names (`mint` / `owner`) instead of Metaplex names (`mintAddress` / `ownerAddress`).

### Pagination

Use **either** `page` **or** `before`/`after` — not both (the client throws `DasApiError`). `cursor` is also supported where the RPC provides it. `sortBy: { sortBy, sortDirection }`.

### Display options

`showUnverifiedCollections`, `showCollectionMetadata`, `showFungible`, `showInscription` (plus `showZeroBalance` on `getTokenAccounts`).

---

## GPA vs DAS (Core)

`fetchAssetsByOwner` / `fetchAssetsByCollection` in `mpl-core` use GPA and can fail on burned account remnants. Prefer:

```typescript
import { das } from '@metaplex-foundation/mpl-core-das';

const assets = await das.getAssetsByOwner(umi, { owner: wallet });
```

---

## For more info

- DAS API: https://metaplex.com/docs/dev-tools/das-api
- Getting started: https://metaplex.com/docs/dev-tools/das-api/getting-started
- Core groups: https://metaplex.com/docs/smart-contracts/core/groups
- OpenRPC playground: https://playground.open-rpc.org/?url=https://raw.githubusercontent.com/metaplex-foundation/digital-asset-standard-api/main/specification/metaplex-das-api.json
