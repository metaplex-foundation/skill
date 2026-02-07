# Metaplex Genesis SDK Reference

Genesis is a token launch protocol for Token Generation Events (TGEs) on Solana with fair distribution and liquidity graduation.

> **Concepts**: For lifecycle, fees, condition object format, and end behaviors, see `./concepts.md` Genesis section.

## Package

```bash
npm install @metaplex-foundation/genesis @metaplex-foundation/umi-bundle-defaults
```

## Before Starting — Gather from User

1. **Token details**: name, symbol, description, image/metadata URI
2. **Total supply**: how many tokens (remember: with 9 decimals, 1M tokens = `1_000_000_000_000_000n`)
3. **Allocation split**: percentage for launchpool vs team/treasury
4. **Timing**: deposit start, deposit duration, claim duration

---

## Launch Mechanisms

| Mechanism | Description |
|-----------|-------------|
| **Launch Pool** | Users deposit SOL during a window, receive tokens proportionally |
| **Presale** | Fixed price token sale, first-come-first-served |
| **Uniform Price Auction** | Bid-based allocation with uniform clearing price |

---

## Launch Lifecycle

```
1. Initialize Genesis Account → Creates token + coordination account
2. Add Buckets → Configure distribution (LaunchPool, Unlocked, etc.)
3. Finalize → Lock configuration, launch goes live
4. Active Period → Users deposit SOL
5. Transition → Execute end behaviors (send SOL to outflow buckets)
6. Claim Period → Users claim tokens proportionally
```

---

## Setup

```typescript
import { createUmi } from '@metaplex-foundation/umi-bundle-defaults';
import { genesis } from '@metaplex-foundation/genesis';

const umi = createUmi('https://api.devnet.solana.com')
  .use(genesis());
```

---

## Initialize Genesis Account

```typescript
import {
  findGenesisAccountV2Pda,
  initializeV2,
} from '@metaplex-foundation/genesis';
import { generateSigner, publicKey } from '@metaplex-foundation/umi';

const baseMint = generateSigner(umi);
const WSOL_MINT = publicKey('So11111111111111111111111111111111111111112');

const [genesisAccount] = findGenesisAccountV2Pda(umi, {
  baseMint: baseMint.publicKey,
  genesisIndex: 0,
});

await initializeV2(umi, {
  baseMint,
  quoteMint: WSOL_MINT,
  fundingMode: 0,
  totalSupplyBaseToken: 1_000_000_000_000_000n,  // 1M tokens (9 decimals)
  name: 'My Token',
  symbol: 'MTK',
  uri: 'https://example.com/metadata.json',
}).sendAndConfirm(umi);
```

**Token Supply with Decimals:**
```typescript
const ONE_TOKEN = 1_000_000_000n;              // 1 token (9 decimals)
const ONE_MILLION = 1_000_000_000_000_000n;    // 1,000,000 tokens
const ONE_BILLION = 1_000_000_000_000_000_000n; // 1,000,000,000 tokens
```

---

## Add Launch Pool Bucket

```typescript
import {
  addLaunchPoolBucketV2,
  findLaunchPoolBucketV2Pda,
  findUnlockedBucketV2Pda,
  NOT_TRIGGERED_TIMESTAMP,
} from '@metaplex-foundation/genesis';

const [launchPoolBucket] = findLaunchPoolBucketV2Pda(umi, {
  genesisAccount,
  bucketIndex: 0,
});

const [unlockedBucket] = findUnlockedBucketV2Pda(umi, {
  genesisAccount,
  bucketIndex: 0,
});

const now = BigInt(Math.floor(Date.now() / 1000));
const depositEnd = now + 86400n * 3n;  // 3 days
const claimStart = depositEnd + 1n;
const claimEnd = claimStart + 86400n * 7n;  // 7 days

await addLaunchPoolBucketV2(umi, {
  genesisAccount,
  baseMint: baseMint.publicKey,
  baseTokenAllocation: 600_000_000_000_000n,  // 60% of supply
  depositStartCondition: {
    __kind: 'TimeAbsolute',
    padding: Array(47).fill(0),
    time: now,
    triggeredTimestamp: NOT_TRIGGERED_TIMESTAMP,
  },
  depositEndCondition: {
    __kind: 'TimeAbsolute',
    padding: Array(47).fill(0),
    time: depositEnd,
    triggeredTimestamp: NOT_TRIGGERED_TIMESTAMP,
  },
  claimStartCondition: {
    __kind: 'TimeAbsolute',
    padding: Array(47).fill(0),
    time: claimStart,
    triggeredTimestamp: NOT_TRIGGERED_TIMESTAMP,
  },
  claimEndCondition: {
    __kind: 'TimeAbsolute',
    padding: Array(47).fill(0),
    time: claimEnd,
    triggeredTimestamp: NOT_TRIGGERED_TIMESTAMP,
  },
  minimumDepositAmount: null,
  endBehaviors: [
    {
      __kind: 'SendQuoteTokenPercentage',
      padding: Array(4).fill(0),
      destinationBucket: publicKey(unlockedBucket),
      percentageBps: 10000,  // 100%
      processed: false,
    },
  ],
}).sendAndConfirm(umi);
```

---

## Add Unlocked Bucket (Team/Treasury)

```typescript
import { addUnlockedBucketV2 } from '@metaplex-foundation/genesis';

await addUnlockedBucketV2(umi, {
  genesisAccount,
  baseMint: baseMint.publicKey,
  baseTokenAllocation: 200_000_000_000_000n,  // 20% of supply
  recipient: umi.identity.publicKey,
  claimStartCondition: {
    __kind: 'TimeAbsolute',
    padding: Array(47).fill(0),
    time: claimStart,
    triggeredTimestamp: NOT_TRIGGERED_TIMESTAMP,
  },
  claimEndCondition: {
    __kind: 'TimeAbsolute',
    padding: Array(47).fill(0),
    time: claimEnd,
    triggeredTimestamp: NOT_TRIGGERED_TIMESTAMP,
  },
}).sendAndConfirm(umi);
```

---

## Finalize Launch

```typescript
import { finalizeV2 } from '@metaplex-foundation/genesis';

await finalizeV2(umi, {
  baseMint: baseMint.publicKey,
  genesisAccount,
}).sendAndConfirm(umi);
```

⚠️ **Finalization is irreversible.** No more buckets can be added after this.

---

## User Operations

### Deposit SOL

```typescript
import { depositLaunchPoolV2 } from '@metaplex-foundation/genesis';

await depositLaunchPoolV2(umi, {
  genesisAccount,
  bucket: launchPoolBucket,
  baseMint: baseMint.publicKey,
  amountQuoteToken: 10_000_000_000n,  // 10 SOL
}).sendAndConfirm(umi);
```

### Withdraw SOL

```typescript
import { withdrawLaunchPoolV2 } from '@metaplex-foundation/genesis';

await withdrawLaunchPoolV2(umi, {
  genesisAccount,
  bucket: launchPoolBucket,
  baseMint: baseMint.publicKey,
  amountQuoteToken: 3_000_000_000n,  // 3 SOL
}).sendAndConfirm(umi);
```

### Claim Tokens

```typescript
import { claimLaunchPoolV2 } from '@metaplex-foundation/genesis';

await claimLaunchPoolV2(umi, {
  genesisAccount,
  bucket: launchPoolBucket,
  baseMint: baseMint.publicKey,
  recipient: umi.identity.publicKey,
}).sendAndConfirm(umi);
```

---

## Execute Transition

After deposit period ends, execute transition to process end behaviors:

```typescript
import { transitionV2, WRAPPED_SOL_MINT } from '@metaplex-foundation/genesis';
import { findAssociatedTokenPda } from '@metaplex-foundation/mpl-toolbox';

const unlockedBucketQuoteTokenAccount = findAssociatedTokenPda(umi, {
  owner: unlockedBucket,
  mint: WRAPPED_SOL_MINT,
});

await transitionV2(umi, {
  genesisAccount,
  primaryBucket: launchPoolBucket,
  baseMint: baseMint.publicKey,
})
  .addRemainingAccounts([
    { pubkey: unlockedBucket, isSigner: false, isWritable: true },
    { pubkey: publicKey(unlockedBucketQuoteTokenAccount), isSigner: false, isWritable: true },
  ])
  .sendAndConfirm(umi);
```

---

## Revoke Authorities (Post-Launch)

```typescript
import { revokeMintAuthorityV2, revokeFreezeAuthorityV2 } from '@metaplex-foundation/genesis';

// No more tokens can ever be minted
await revokeMintAuthorityV2(umi, {
  baseMint: baseMint.publicKey,
}).sendAndConfirm(umi);

// Tokens can never be frozen
await revokeFreezeAuthorityV2(umi, {
  baseMint: baseMint.publicKey,
}).sendAndConfirm(umi);
```

⚠️ **Authority revocation is irreversible.**

---

## Fetching State

```typescript
import {
  fetchLaunchPoolBucketV2,
  fetchLaunchPoolDepositV2,
  findLaunchPoolDepositV2Pda,
} from '@metaplex-foundation/genesis';

// Bucket state
const bucket = await fetchLaunchPoolBucketV2(umi, launchPoolBucket);
console.log('Total deposits:', bucket.quoteTokenDepositTotal);
console.log('Token allocation:', bucket.bucket.baseTokenAllocation);

// User deposit state
const [depositPda] = findLaunchPoolDepositV2Pda(umi, {
  bucket: launchPoolBucket,
  recipient: umi.identity.publicKey,
});
const deposit = await fetchLaunchPoolDepositV2(umi, depositPda);
console.log('User deposit:', deposit.amountQuoteToken);
```

---

## Fees

Genesis charges protocol-level fees on deposits and withdrawals. For current rates, see: https://developers.metaplex.com/protocol-fees

---

## Program ID

```
Genesis: GENSkbJAfXcp9nvQm9eBPMg4MUefawD4oBNK7P8aLvEC
```

## Documentation

Full documentation: https://developers.metaplex.com/genesis
