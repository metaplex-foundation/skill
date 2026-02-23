# Metaplex Genesis SDK Reference

Genesis is a token launch protocol for Token Generation Events (TGEs) on Solana with fair distribution and liquidity graduation.

> **Concepts**: For lifecycle, fees, condition object format, and end behaviors, see `./concepts.md` Genesis section.

## Package

```bash
npm install @metaplex-foundation/genesis @metaplex-foundation/umi-bundle-defaults
```

## Before Starting — Gather from User

**For Launch API** (recommended):
1. **Token details**: name (1-32 chars), symbol (1-10 chars), image (Irys URL), description (optional, max 250 chars)
2. **Launch pool**: token allocation (portion of 1B), deposit start time, raise goal, Raydium liquidity %, funds recipient
3. **Optional**: locked allocations (team vesting), external links (website, twitter, telegram), quote mint (SOL/USDC)

**For low-level SDK**:
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

**Launch API** (recommended):
```
createAndRegisterLaunch()  →  deposit window (48h)  →  Raydium graduation  →  claim
```

**Low-level SDK**:
```
1. Initialize Genesis Account → Creates token + coordination account
2. Add Buckets → Configure distribution (LaunchPool, Unlocked, etc.)
3. Finalize → Lock configuration, launch goes live
4. Active Period → Users deposit SOL
5. Transition → Execute end behaviors (send SOL to outflow buckets)
6. Claim Period → Users claim tokens proportionally
```

---

## Launch API (Recommended)

The Launch API handles everything in a single call: token creation, genesis account setup, launch pool configuration, Raydium LP, transaction signing, and platform registration.

### `createAndRegisterLaunch` — All-in-One

```typescript
import { createUmi } from '@metaplex-foundation/umi-bundle-defaults';
import { keypairIdentity } from '@metaplex-foundation/umi';
import {
  createAndRegisterLaunch,
  CreateLaunchInput,
} from '@metaplex-foundation/genesis';

const umi = createUmi('https://api.mainnet-beta.solana.com')
  .use(keypairIdentity(myKeypair));

const input: CreateLaunchInput = {
  wallet: umi.identity.publicKey,
  token: {
    name: 'My Token',
    symbol: 'MTK',
    image: 'https://gateway.irys.xyz/...',
    // optional:
    description: 'A revolutionary token',
    externalLinks: {
      website: 'https://mytoken.com',
      twitter: 'https://twitter.com/mytoken',
      telegram: 'https://t.me/mytoken',
    },
  },
  launchType: 'project',
  launch: {
    launchpool: {
      tokenAllocation: 500_000_000,   // out of 1B total supply
      depositStartTime: new Date(Date.now() + 48 * 60 * 60 * 1000),
      raiseGoal: 200,                 // 200 SOL (whole units)
      raydiumLiquidityBps: 5000,      // 50% to Raydium LP
      fundsRecipient: umi.identity.publicKey,
    },
  },
  // optional:
  quoteMint: 'SOL',                   // 'SOL' | 'USDC' | mint address
  network: 'solana-mainnet',          // auto-detected if omitted
};

const result = await createAndRegisterLaunch(umi, { baseUrl: 'https://api.metaplex.com' }, input);

console.log('Genesis account:', result.genesisAccount);
console.log('Mint:', result.mintAddress);
console.log('Launch page:', result.launch.link);
console.log('Signatures:', result.signatures);
```

### With Locked Allocations (Team Vesting via Streamflow)

```typescript
const input: CreateLaunchInput = {
  wallet: umi.identity.publicKey,
  token: {
    name: 'My Token',
    symbol: 'MTK',
    image: 'https://gateway.irys.xyz/...',
  },
  launchType: 'project',
  launch: {
    launchpool: {
      tokenAllocation: 500_000_000,
      depositStartTime: new Date('2026-03-01T00:00:00Z'),
      raiseGoal: 200,
      raydiumLiquidityBps: 5000,
      fundsRecipient: umi.identity.publicKey,
    },
    lockedAllocations: [
      {
        name: 'Team',
        recipient: 'TeamWallet111...',
        tokenAmount: 100_000_000,
        vestingStartTime: new Date('2026-04-05T00:00:00Z'),
        vestingDuration: { value: 1, unit: 'YEAR' },
        unlockSchedule: 'MONTH',
        cliff: {
          duration: { value: 3, unit: 'MONTH' },
          unlockAmount: 10_000_000,
        },
      },
    ],
  },
};
```

TimeUnit values: `'SECOND'`, `'MINUTE'`, `'HOUR'`, `'DAY'`, `'WEEK'`, `'TWO_WEEKS'`, `'MONTH'`, `'QUARTER'`, `'YEAR'`.

### `createLaunch` + `registerLaunch` — Full Control

Use when you need custom transaction handling (multisig, custom sending logic).

```typescript
import {
  createLaunch,
  registerLaunch,
  GenesisApiConfig,
} from '@metaplex-foundation/genesis';

const config: GenesisApiConfig = { baseUrl: 'https://api.metaplex.com' };

// Step 1: Get unsigned transactions
const createResult = await createLaunch(umi, config, input);
// createResult.transactions — unsigned Umi transactions
// createResult.blockhash — for confirmation strategy
// createResult.mintAddress, createResult.genesisAccount

// Step 2: Sign and send transactions yourself
for (const tx of createResult.transactions) {
  const signedTx = await umi.identity.signTransaction(tx);
  const signature = await umi.rpc.sendTransaction(signedTx, { commitment: 'confirmed' });
  await umi.rpc.confirmTransaction(signature, {
    commitment: 'confirmed',
    strategy: { type: 'blockhash', ...createResult.blockhash },
  });
}

// Step 3: Register on the platform (idempotent — safe to retry)
const registerResult = await registerLaunch(umi, config, {
  genesisAccount: createResult.genesisAccount,
  createLaunchInput: input,
});
console.log('Launch page:', registerResult.launch.link);
```

### Custom Transaction Sender

```typescript
import {
  createAndRegisterLaunch,
  SignAndSendOptions,
} from '@metaplex-foundation/genesis';

const options: SignAndSendOptions = {
  txSender: async (transactions) => {
    const signatures: Uint8Array[] = [];
    for (const tx of transactions) {
      const signed = await myMultisigSign(tx);
      const sig = await myCustomSend(signed);
      signatures.push(sig);
    }
    return signatures;
  },
};

const result = await createAndRegisterLaunch(umi, config, input, options);
```

### Error Handling

```typescript
import {
  createAndRegisterLaunch,
  isGenesisValidationError,
  isGenesisApiError,
  isGenesisApiNetworkError,
} from '@metaplex-foundation/genesis';

try {
  const result = await createAndRegisterLaunch(umi, config, input);
} catch (err) {
  if (isGenesisValidationError(err)) {
    console.error(`Invalid "${err.field}":`, err.message);
  } else if (isGenesisApiError(err)) {
    console.error('API error:', err.statusCode, err.responseBody);
  } else if (isGenesisApiNetworkError(err)) {
    console.error('Network error:', err.cause.message);
  }
}
```

### Launch API Key Points

- **Total supply** is always 1 billion tokens; `tokenAllocation` is how many go to the launch pool
- **Deposit window** is always 48 hours from `depositStartTime`
- **raiseGoal** and amounts are in **whole units** (e.g., `200` = 200 SOL), NOT base units
- **Image** must be hosted on Irys (`https://gateway.irys.xyz/...`)
- Remaining tokens (1B minus launchpool minus locked) go to the creator automatically
- **registerLaunch** is idempotent — safe to call again if it fails
- Fund routing is automatic: `raydiumLiquidityBps` goes to Raydium LP, rest goes to `fundsRecipient`

---

## Low-Level SDK

The following sections cover direct on-chain instructions for full control over genesis accounts and buckets.

### Setup

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
