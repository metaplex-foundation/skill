# Metaplex Concepts Reference

## Token Standards

### Token Metadata Standards

| Standard | Use Case | Accounts |
|----------|----------|----------|
| `Fungible` | SPL tokens, memecoins, utility tokens | Mint + Metadata |
| `FungibleAsset` | Semi-fungible, non-divisible | Mint + Metadata |
| `NonFungible` | Standard NFT (1/1) | Mint + Metadata + MasterEdition |
| `NonFungibleEdition` | Print editions | Mint + Metadata + Edition |
| `ProgrammableNonFungible` | NFT with enforced royalties (pNFT) | + TokenRecord |
| `ProgrammableNonFungibleEdition` | Print edition of a pNFT | + TokenRecord + Edition |

### Core vs Token Metadata

| Aspect | Core | Token Metadata |
|--------|------|----------------|
| Accounts per NFT | 1 | 3-4 |
| Mint cost | ~0.0029 SOL | ~0.022 SOL |
| Compute units | ~17,000 CU | ~205,000 CU |
| Royalty enforcement | Built-in | Requires pNFT |
| Plugins | Yes | No |
| Best for | New NFT projects | Fungibles, legacy NFTs |

---

## Account Structures

### Token Metadata Accounts

```
Metadata Account
├── key: Key (1 byte)
├── updateAuthority: PublicKey
├── mint: PublicKey
├── name: string (max 32)
├── symbol: string (max 10)
├── uri: string (max 200)
├── sellerFeeBasisPoints: u16
├── creators: Option<Vec<Creator>>
├── primarySaleHappened: bool
├── isMutable: bool
├── editionNonce: Option<u8>
├── tokenStandard: Option<TokenStandard>
├── collection: Option<Collection>
├── uses: Option<Uses>
├── collectionDetails: Option<CollectionDetails>
└── programmableConfig: Option<ProgrammableConfig>

MasterEdition Account
├── key: Key
├── supply: u64
└── maxSupply: Option<u64>

TokenRecord Account (pNFTs only)
├── key: Key
├── bump: u8
├── state: TokenState
├── delegate: Option<PublicKey>
├── delegateRole: Option<TokenDelegateRole>
└── lockedTransfer: Option<PublicKey>
```

### Core Asset Structure

```
Asset Account (Single Account)
├── key: Key
├── owner: PublicKey
├── updateAuthority: UpdateAuthority
├── name: string
├── uri: string
├── seq: Option<u64>
└── plugins: Vec<PluginHeader + Plugin>

Collection Account
├── key: Key
├── updateAuthority: PublicKey
├── name: string
├── uri: string
├── numMinted: u32
├── currentSize: u32
└── plugins: Vec<PluginHeader + Plugin>
```

---

## PDA Seeds

### Token Metadata PDAs

| Account | Seeds |
|---------|-------|
| Metadata | `['metadata', program_id, mint]` |
| Master Edition | `['metadata', program_id, mint, 'edition']` |
| Edition | `['metadata', program_id, mint, 'edition']` |
| Token Record | `['metadata', program_id, mint, 'token_record', token]` |
| Collection Authority | `['metadata', program_id, mint, 'collection_authority', authority]` |
| Use Authority | `['metadata', program_id, mint, 'user', use_authority]` |

---

## Metadata JSON Standard

```json
{
  "name": "Asset Name",
  "description": "Description of the asset",
  "image": "https://...",
  "external_url": "https://...",
  "attributes": [
    {
      "trait_type": "Background",
      "value": "Blue"
    },
    {
      "trait_type": "Rarity",
      "value": "Legendary"
    }
  ],
  "properties": {
    "files": [
      {
        "uri": "https://...",
        "type": "image/png"
      }
    ],
    "category": "image"
  }
}
```

**Categories:** `image`, `video`, `audio`, `vr`, `html`

---

## Authorities

### Token Metadata Authorities

| Authority | Can |
|-----------|-----|
| Update Authority | Update metadata, verify creators, verify collection |
| Mint Authority | Mint new tokens (fungibles) |
| Freeze Authority | Freeze token accounts |

### Core Authorities

| Authority | Can |
|-----------|-----|
| Owner | Transfer, burn, add owner-managed plugins |
| Update Authority | Update metadata, add authority-managed plugins |

---

## Collection Patterns

### Core Collections

- Assets created with `collection` param are **auto-verified**
- No separate verification step needed

### Token Metadata Collections

- Collection is an NFT itself
- Items reference collection but start **unverified**
- Must call `verifyCollectionV1` as collection authority

---

## Core Plugin Types

### Owner-Managed
- `TransferDelegate` - Allow another to transfer
- `FreezeDelegate` - Allow another to freeze
- `BurnDelegate` - Allow another to burn

### Authority-Managed
- `Royalties` - Enforce royalties on transfers
- `UpdateDelegate` - Allow another to update
- `Attributes` - On-chain attributes

### Permanent (Immutable after adding)
- `PermanentTransferDelegate`
- `PermanentFreezeDelegate`
- `PermanentBurnDelegate`

---

## Common Errors

| Error | Cause | Solution |
|-------|-------|----------|
| `InvalidTokenStandard` | Wrong standard for operation | Check asset's actual token standard |
| `InvalidAuthority` | Signer doesn't match authority | Verify update/mint authority |
| `CollectionNotVerified` | Collection not verified | Call `verifyCollectionV1` |
| `TokenRecordNotFound` | Missing TokenRecord for pNFT | Include token record PDA |
| `PluginNotFound` | Plugin doesn't exist | Add plugin first or check name |
| `InsufficientFunds` | Not enough SOL | Fund wallet |
| `Invalid data enum variant` | Wrong JSON format | Check array format, type field, ruleSet object |

---

## Cost Comparison

| Operation | Token Metadata | Core | Savings |
|-----------|---------------|------|---------|
| Mint NFT | ~0.022 SOL | ~0.0029 SOL | 87% |
| Compute Units | ~205,000 CU | ~17,000 CU | 92% |
| Accounts | 3-4 | 1 | 75% |

---

## Genesis Concepts

### Lifecycle

```
Initialize → Add Buckets → Finalize (irreversible) → Deposit Period → Transition → Claim Period
```

- **Initialize**: Creates a Genesis account + the token mint. All configuration happens here (name, symbol, supply, quote mint).
- **Add Buckets**: Configure how tokens are distributed (LaunchPool for public, Unlocked for team/treasury).
- **Finalize**: Locks the configuration. No more buckets can be added. **Irreversible.**
- **Deposit Period**: Users deposit SOL (quote token) into the LaunchPool bucket.
- **Transition**: After deposit period ends, executes end behaviors (e.g., routes deposited SOL to outflow buckets).
- **Claim Period**: Users claim tokens proportional to their deposit.

### Fees

Genesis charges protocol-level fees on deposits and withdrawals. For current fee rates, see: https://developers.metaplex.com/protocol-fees

### Condition Objects

Buckets use condition objects for timing (deposit start/end, claim start/end). The format:

```typescript
{
  __kind: 'TimeAbsolute',           // Trigger type — absolute Unix timestamp
  padding: Array(47).fill(0),       // Reserved bytes for future use (required by on-chain layout)
  time: BigInt(unixTimestamp),       // When to trigger
  triggeredTimestamp: NOT_TRIGGERED_TIMESTAMP,  // Sentinel value — program sets this when triggered
}
```

The `padding` field is required by the on-chain account layout for forward compatibility. Always use `Array(47).fill(0)`. The `NOT_TRIGGERED_TIMESTAMP` constant is exported from the Genesis package.

### End Behaviors

After a LaunchPool's deposit period ends, `endBehaviors` define what happens to the deposited SOL:

```typescript
{
  __kind: 'SendQuoteTokenPercentage',  // Send % of collected SOL to another bucket
  padding: Array(4).fill(0),           // Reserved bytes (required)
  destinationBucket: publicKey(bucket), // Target bucket address (e.g., Unlocked bucket for team)
  percentageBps: 10000,                // 10000 = 100% of collected SOL
  processed: false,                    // Set by program after execution — always pass false
}
```

Available end behavior types:
- `SendQuoteTokenPercentage` — Route a percentage of collected SOL to another bucket

### Token Supply Decimals

Genesis tokens default to 9 decimals. Supply is specified in base units:

| Human Amount | Base Units (9 decimals) |
|---|---|
| 1 token | `1_000_000_000n` |
| 1,000,000 tokens | `1_000_000_000_000_000n` |
| 1,000,000,000 tokens | `1_000_000_000_000_000_000n` |
