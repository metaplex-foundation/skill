# Metaplex Genesis SDK Reference

Genesis is a token launch protocol for Token Generation Events (TGEs) on Solana with fair distribution and liquidity graduation.

> **Concepts**: For lifecycle, fees, condition object format, and end behaviors, see `./concepts.md` Genesis section.
> **Low-level SDK (manual buckets, presale, vesting, graduation)**: See `./sdk-genesis-low-level.md`.

## Package

```bash
npm install @metaplex-foundation/genesis @metaplex-foundation/umi-bundle-defaults
```

## Before Starting — Gather from User

**For Launch API** (recommended):
1. **Launch type**: `'launchpool'` (48h deposit, configurable) or `'bondingCurve'` (instant bonding curve).
2. **Token details**: name (1-32 chars), symbol (1-10 chars), image (Irys URL), description (optional, max 250 chars)
3. **For launchpool launches**: token allocation (portion of 1B), deposit start time, raise goal, Raydium liquidity %, funds recipient
4. **For bonding curve launches**: only token details needed. Optional: creator fee wallet, first buy amount, agent config.
5. **Optional**: locked allocations (team vesting, launchpool only), external links (website, twitter, telegram), quote mint (SOL/USDC), agent config

**For low-level SDK**: See `./sdk-genesis-low-level.md`.

---

## Launch Mechanisms

| Mechanism | Description |
|-----------|-------------|
| **Bonding Curve** | Constant product AMM — instant trading, price rises as SOL flows in, auto-graduates to Raydium CPMM on sell-out |
| **Launch Pool** | Users deposit SOL during a window, receive tokens proportionally |
| **Presale** | Fixed price token sale, first-come-first-served |
| **Uniform Price Auction** | Bid-based allocation with uniform clearing price |

---

## Launch Lifecycle

**Launch API** (recommended):

```text
Launchpool:     createAndRegisterLaunch()  →  deposit window (48h)  →  Raydium graduation  →  claim
Bonding Curve:  createAndRegisterLaunch()  →  instant trading (buy/sell)  →  sell-out  →  auto-graduation to Raydium CPMM
```

**Low-level SDK** (see `./sdk-genesis-low-level.md`):

```text
1. Initialize Genesis Account → Creates token + coordination account
2. Add Buckets → Configure distribution (LaunchPool, Unlocked, etc.)
3. Finalize → Lock configuration, launch goes live
4. Active Period → Users deposit SOL
5. Transition → Execute end behaviors (send SOL to outflow buckets)
6. Graduation → LP tokens graduated to Raydium
7. Claim Period → Users claim tokens proportionally
```

---

## Launch API (Recommended)

The Launch API handles everything in a single call: token creation, genesis account setup, configuration, Raydium LP, transaction signing, and platform registration.

### Launchpool Launch

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
  launchType: 'launchpool',
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
  launchType: 'launchpool',
  launch: {
    launchpool: {
      tokenAllocation: 500_000_000,
      depositStartTime: new Date(Date.now() + 48 * 60 * 60 * 1000),
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

### Bonding Curve Launch

```typescript
import { createAndRegisterLaunch } from '@metaplex-foundation/genesis';

const result = await createAndRegisterLaunch(umi, {}, {
  wallet: umi.identity.publicKey,
  launchType: 'bondingCurve',
  token: {
    name: 'My Token',
    symbol: 'MTK',
    image: 'https://gateway.irys.xyz/your-image-id',
  },
  launch: {},
});

console.log('Mint:', result.mintAddress);
console.log('View at:', result.launch.link);
```

All protocol parameters (supply splits, virtual reserves, fund flows) are set to defaults when `launch: {}` is empty.

> For devnet: add `network: 'solana-devnet'` to the input.

### Bonding Curve with Creator Fees

```typescript
const result = await createAndRegisterLaunch(umi, {}, {
  wallet: umi.identity.publicKey,
  launchType: 'bondingCurve',
  token: {
    name: 'My Token',
    symbol: 'MTK',
    image: 'https://gateway.irys.xyz/your-image-id',
  },
  launch: {
    creatorFeeWallet: 'FeeRecipientWalletAddress...',
  },
});
```

### Bonding Curve with First Buy

Fee-free initial purchase reserved for the launching wallet at curve creation:

```typescript
const result = await createAndRegisterLaunch(umi, {}, {
  wallet: umi.identity.publicKey,
  launchType: 'bondingCurve',
  token: {
    name: 'My Token',
    symbol: 'MTK',
    image: 'https://gateway.irys.xyz/your-image-id',
  },
  launch: {
    firstBuyAmount: 0.1,  // 0.1 SOL, fee-free
  },
});
```

### Agent Launch (Auto Creator Fee Wallet)

When launching on behalf of an agent, the SDK auto-derives the creator fee wallet from the agent's Core asset signer PDA (`['mpl-core-execute', <agent_mint>]`). First buy buyer also defaults to the agent PDA.

```typescript
const result = await createAndRegisterLaunch(umi, {}, {
  wallet: umi.identity.publicKey,
  agent: {
    mint: agentAssetAddress,  // Core asset (NFT) address
    setToken: true,           // permanently link token to agent (IRREVERSIBLE)
  },
  launchType: 'bondingCurve',
  token: {
    name: 'Agent Token',
    symbol: 'AGT',
    image: 'https://gateway.irys.xyz/your-image-id',
  },
  launch: {},
});
```

> `setToken: true` permanently associates the token with the agent — **irreversible**. To override the auto-derived fee wallet, set `launch.creatorFeeWallet` explicitly.
>
> **Agent launches with `setToken: true`**: Prefer the manual flow below (`createLaunch` + `signAndSendLaunchTransactions` + `registerLaunch`). With `createAndRegisterLaunch`, the on-chain `createLaunch` step may succeed (permanently setting the token) while `registerLaunch` fails due to API indexing lag. Retrying then fails with "Agent already has a different agent token set". The manual flow lets you retry `registerLaunch` independently.

### `createLaunch` + `registerLaunch` — Full Control

Use when you need custom transaction handling (multisig, Jito bundles, priority fees, retry logic), or for agent launches with `setToken: true` (see note above).

```typescript
import {
  createLaunch,
  registerLaunch,
  signAndSendLaunchTransactions,
} from '@metaplex-foundation/genesis';

const createResult = await createLaunch(umi, {}, {
  wallet: umi.identity.publicKey,
  launchType: 'bondingCurve',
  token: { name: 'My Token', symbol: 'MTK', image: 'https://gateway.irys.xyz/...' },
  launch: { creatorFeeWallet: 'FeeWallet...' },
});

const signatures = await signAndSendLaunchTransactions(umi, createResult);

const registered = await registerLaunch(umi, {}, {
  genesisAccount: createResult.genesisAccount,
  createLaunchInput: { /* same input as above */ },
});
console.log('Launch:', registered.launch.link);
```

> Call `registerLaunch` only after create transactions are confirmed on-chain.

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

### Launch Types

```typescript
interface CreateBondingCurveLaunchInput {
  wallet: PublicKey | string;
  launchType: 'bondingCurve';
  token: TokenMetadata;
  network?: 'solana-mainnet' | 'solana-devnet';
  quoteMint?: 'SOL';
  agent?: {
    mint: PublicKey | string;   // Core asset (NFT) address
    setToken: boolean;          // permanently link token to agent
  };
  launch: BondingCurveLaunchInput;
}

interface BondingCurveLaunchInput {
  creatorFeeWallet?: PublicKey | string;
  firstBuyAmount?: number;   // SOL (e.g. 0.1 = 0.1 SOL)
  firstBuyWallet?: Signer;
}
```

### Launch API Key Points

- **Two launch types**: `'launchpool'` (default, 48h deposit, configurable) and `'bondingCurve'` (instant)
- **Total supply** is always 1 billion tokens; `tokenAllocation` is how many go to the launch pool (launchpool only)
- **Deposit window**: 48 hours from `depositStartTime` (launchpool only)
- **Bonding curve** has no deposit window — trading starts immediately, graduates to Raydium CPMM on sell-out
- **raiseGoal** and amounts are in **whole units** (e.g., `200` = 200 SOL), NOT base units
- **Image** must be hosted on Irys (`https://gateway.irys.xyz/...`)
- Remaining tokens (1B minus launchpool minus locked) go to the creator automatically (launchpool only)
- **registerLaunch** is idempotent — safe to call again if it fails
- Fund routing is automatic: `raydiumLiquidityBps` goes to Raydium LP, rest goes to `fundsRecipient` (launchpool only)

---

## Bonding Curve Swap Integration

After a bonding curve is created, use these SDK functions to read state, get quotes, and execute swaps.

### Setup (Swap Operations)

```typescript
import { createUmi } from '@metaplex-foundation/umi-bundle-defaults';
import { genesis } from '@metaplex-foundation/genesis';
import { mplToolbox } from '@metaplex-foundation/mpl-toolbox';
import { keypairIdentity } from '@metaplex-foundation/umi';

const umi = createUmi('https://api.mainnet-beta.solana.com').use(mplToolbox()).use(genesis());

const keypair = umi.eddsa.createKeypairFromSecretKey(mySecretKeyBytes);
umi.use(keypairIdentity(keypair));
```

> **`mplToolbox()`** registers SPL Token and Associated Token programs — required for `findAssociatedTokenPda`, `createTokenIfMissing`, `transferSol`, `syncNative`, and `closeToken`.

### Fetch Bonding Curve State

```typescript
import {
  findBondingCurveBucketV2Pda,
  findGenesisAccountV2Pda,
  fetchBondingCurveBucketV2,
} from '@metaplex-foundation/genesis';
import { publicKey } from '@metaplex-foundation/umi';

// From genesis account
const [bucketPda] = findBondingCurveBucketV2Pda(umi, {
  genesisAccount: publicKey('GENESIS_ACCOUNT'),
  bucketIndex: 0,
});
const bucket = await fetchBondingCurveBucketV2(umi, bucketPda);

// From token mint (derive genesis account first)
const [genesisAccount] = findGenesisAccountV2Pda(umi, {
  baseMint: publicKey('TOKEN_MINT'),
  genesisIndex: 0,
});
const [bucketFromMint] = findBondingCurveBucketV2Pda(umi, { genesisAccount, bucketIndex: 0 });
```

The returned bucket object has nested fields. The lifecycle helpers (`isSwappable`, `getSwapResult`, `getFillPercentage`, etc.) accept the full bucket object directly — you rarely need to access fields manually. If you do: `bucket.bucket.baseTokenBalance` (tokens remaining), `bucket.bucket.quoteTokenBalance` (SOL deposited into curve), `bucket.quoteTokenDepositTotal` (total real SOL deposited), `bucket.constantProductParams.virtualSol`, `bucket.constantProductParams.virtualTokens`, `bucket.depositFee` (protocol fee bps), `bucket.withdrawFee` (creator fee bps).

### Lifecycle Helpers

```typescript
import {
  isSwappable,
  isFirstBuyPending,
  isSoldOut,
  getFillPercentage,
  isGraduated,
} from '@metaplex-foundation/genesis';

isSwappable(bucket);           // true when curve accepts public swaps
isFirstBuyPending(bucket);     // true when first-buy not yet executed
isSoldOut(bucket);             // true when baseTokenBalance === 0
getFillPercentage(bucket);     // 0–100 percentage sold
await isGraduated(umi, bucket); // true when Raydium CPMM pool exists (async RPC call)
```

> Always check `isSwappable(bucket)` before quoting or sending a swap.

### Get Swap Quote

```typescript
import {
  getSwapResult, getCurrentPrice, getCurrentPriceQuotePerBase, SwapDirection,
} from '@metaplex-foundation/genesis';

// Buy quote (SOL → tokens)
const buyQuote = getSwapResult(bucket, 1_000_000_000n, SwapDirection.Buy);  // 1 SOL
// Returns: { amountIn, fee, creatorFee, amountOut }

// Sell quote (tokens → SOL)
const sellQuote = getSwapResult(bucket, 500_000_000_000n, SwapDirection.Sell);  // 500 tokens

// First buy (fee-free) — pass true as 4th arg
const firstBuyQuote = getSwapResult(bucket, 2_000_000_000n, SwapDirection.Buy, true);

// Current price (returns 0 on a fresh curve with no deposits)
const tokensPerSol = getCurrentPrice(bucket);          // tokens per SOL
const lamportsPerToken = getCurrentPriceQuotePerBase(bucket);  // lamports per token
```

> **Important**: Use `SwapDirection.Buy` / `SwapDirection.Sell` (enum), not strings. Passing `'buy'`/`'sell'` strings will **silently misinterpret the direction** (e.g., `'buy'` is treated as Sell) without throwing an error — this produces wrong results and can lose funds.

### Slippage Protection

```typescript
import { applySlippage, SwapDirection } from '@metaplex-foundation/genesis';

const quote = getSwapResult(bucket, 1_000_000_000n, SwapDirection.Buy);
const minAmountOut = applySlippage(quote.amountOut, 100);  // 1% slippage (100 bps)
// Use 50 bps (0.5%) for stable, 200 bps (2%) for volatile launches
```

> Never send a swap without `minAmountOut` — the price moves with every trade.

### Execute Swap

```typescript
import { swapBondingCurveV2, getSwapResult, applySlippage, isSwappable, SwapDirection } from '@metaplex-foundation/genesis';
import { findAssociatedTokenPda } from '@metaplex-foundation/mpl-toolbox';
import { publicKey } from '@metaplex-foundation/umi';

const quoteMint = publicKey('So11111111111111111111111111111111111111112'); // wSOL

const quote = getSwapResult(bucket, 1_000_000_000n, SwapDirection.Buy);
const minAmountOut = applySlippage(quote.amountOut, 100);

const [userBaseTokenAccount] = findAssociatedTokenPda(umi, {
  mint: baseMint, owner: umi.identity.publicKey,
});
const [userQuoteTokenAccount] = findAssociatedTokenPda(umi, {
  mint: quoteMint, owner: umi.identity.publicKey,
});

// NOTE: Must manually wrap SOL to wSOL ATA before buy, unwrap after sell
await swapBondingCurveV2(umi, {
  genesisAccount,
  bucket: bucketPda,
  baseMint,
  quoteMint,
  baseTokenAccount: userBaseTokenAccount,
  quoteTokenAccount: userQuoteTokenAccount,
  amount: quote.amountIn,
  minAmountOutScaled: minAmountOut,
  swapDirection: SwapDirection.Buy,
}).sendAndConfirm(umi);
```

> **wSOL handling is manual.** The swap instruction operates on token accounts only — it does not wrap or unwrap native SOL. You must handle this before and after the swap.

### wSOL Wrap / Unwrap

Requires `.use(mplToolbox())` on the Umi instance.

```typescript
import {
  mplToolbox, createTokenIfMissing, findAssociatedTokenPda,
  transferSol, syncNative, closeToken,
} from '@metaplex-foundation/mpl-toolbox';
import { publicKey, transactionBuilder, sol } from '@metaplex-foundation/umi';

// umi must have .use(mplToolbox()) applied

const WSOL = publicKey('So11111111111111111111111111111111111111112');
const SPL_TOKEN = publicKey('TokenkegQfeZyiNwAJbNbGKPFXCWuBvf9Ss623VQ5DA');

const [userQuoteTokenAccount] = findAssociatedTokenPda(umi, {
  mint: WSOL,
  owner: umi.identity.publicKey,
});

// --- Before a BUY: wrap SOL into wSOL ATA ---
await transactionBuilder()
  .add(createTokenIfMissing(umi, { mint: WSOL, token: userQuoteTokenAccount, tokenProgram: SPL_TOKEN }))
  .add(transferSol(umi, { destination: userQuoteTokenAccount, amount: sol(0.1) }))
  .add(syncNative(umi, { account: userQuoteTokenAccount }))
  .sendAndConfirm(umi);

// ... execute swapBondingCurveV2 here ...

// --- After a BUY (or after a SELL to reclaim SOL): unwrap wSOL ---
await closeToken(umi, {
  account: userQuoteTokenAccount,
  destination: umi.identity.publicKey,
  owner: umi.identity,  // must be a Signer (owner of the token account)
}).sendAndConfirm(umi);
```

> Use `createTokenIfMissing` (NOT `createAssociatedToken`) — it is truly idempotent (no-op if the ATA already exists). `createAssociatedToken` fails if the ATA exists.
> For buys, transfer at least `quote.amountIn` lamports. `sol(0.1)` = 0.1 SOL = 100_000_000 lamports.
> `closeToken` returns all remaining wSOL balance as native SOL to `destination`. The `owner` must be the token account owner as a `Signer`.
> The base token ATA does not need to be created manually — `swapBondingCurveV2` creates it internally via CPI.

### Swap Event Decoding

```typescript
import { getBondingCurveSwapEventSerializer } from '@metaplex-foundation/genesis';

// In confirmed transaction inner instructions, find Genesis program (GNS1S5J5AspKXgpjz6SvKL66kPaKWAhaGRhCqPRxii2B)
// with discriminator byte 255, slice it off, then deserialize:
const [event] = getBondingCurveSwapEventSerializer().deserialize(data.slice(1));
// event: { direction, amountIn, amountOut, fee, baseTokenBalanceAfter, quoteTokenDepositTotalAfter }
```

### End-to-End: Launch Bonding Curve + Buy

Complete working example: create a bonding curve token and immediately buy on it.

```typescript
import { createUmi } from '@metaplex-foundation/umi-bundle-defaults';
import { keypairIdentity, publicKey, transactionBuilder, sol } from '@metaplex-foundation/umi';
import {
  genesis,
  createAndRegisterLaunch,
  findGenesisAccountV2Pda,
  findBondingCurveBucketV2Pda,
  fetchBondingCurveBucketV2,
  getSwapResult,
  applySlippage,
  swapBondingCurveV2,
  isSwappable,
  SwapDirection,
} from '@metaplex-foundation/genesis';
import {
  mplToolbox, createTokenIfMissing, findAssociatedTokenPda,
  transferSol, syncNative, closeToken,
} from '@metaplex-foundation/mpl-toolbox';
import { readFileSync } from 'fs';

// --- Setup ---
const keypairFile = JSON.parse(readFileSync('/path/to/keypair.json', 'utf-8'));
const umi = createUmi('https://api.mainnet-beta.solana.com')
  .use(mplToolbox())
  .use(genesis());
const kp = umi.eddsa.createKeypairFromSecretKey(Uint8Array.from(keypairFile));
umi.use(keypairIdentity(kp));

const WSOL = publicKey('So11111111111111111111111111111111111111112');
const SPL_TOKEN = publicKey('TokenkegQfeZyiNwAJbNbGKPFXCWuBvf9Ss623VQ5DA');

// --- Step 1: Launch bonding curve ---
const result = await createAndRegisterLaunch(umi, {}, {
  wallet: umi.identity.publicKey,
  launchType: 'bondingCurve',
  token: {
    name: 'My Token',
    symbol: 'MTK',
    image: 'https://gateway.irys.xyz/your-image-id',
  },
  launch: {},
});
const baseMint = publicKey(result.mintAddress);
console.log('Mint:', result.mintAddress);

// --- Step 2: Derive PDAs and fetch bucket ---
const [genesisAccount] = findGenesisAccountV2Pda(umi, { baseMint, genesisIndex: 0 });
const [bucketPda] = findBondingCurveBucketV2Pda(umi, { genesisAccount, bucketIndex: 0 });
const bucket = await fetchBondingCurveBucketV2(umi, bucketPda);

if (!isSwappable(bucket)) throw new Error('Curve is not swappable');

// --- Step 3: Get buy quote (0.01 SOL) ---
const buyAmount = 10_000_000n; // 0.01 SOL in lamports
const quote = getSwapResult(bucket, buyAmount, SwapDirection.Buy);
const minOut = applySlippage(quote.amountOut, 200); // 2% slippage

// --- Step 4: Wrap SOL → wSOL ---
const [userQuoteAta] = findAssociatedTokenPda(umi, { mint: WSOL, owner: umi.identity.publicKey });
const [userBaseAta] = findAssociatedTokenPda(umi, { mint: baseMint, owner: umi.identity.publicKey });

// Create wSOL ATA if missing (idempotent), fund it — swap creates the base token ATA via CPI
await transactionBuilder()
  .add(createTokenIfMissing(umi, { mint: WSOL, token: userQuoteAta, tokenProgram: SPL_TOKEN }))
  .add(transferSol(umi, { destination: userQuoteAta, amount: sol(0.01) }))
  .add(syncNative(umi, { account: userQuoteAta }))
  .sendAndConfirm(umi);

// --- Step 5: Execute swap ---
await swapBondingCurveV2(umi, {
  genesisAccount,
  bucket: bucketPda,
  baseMint,
  quoteMint: WSOL,
  baseTokenAccount: userBaseAta,
  quoteTokenAccount: userQuoteAta,
  amount: quote.amountIn,
  minAmountOutScaled: minOut,
  swapDirection: SwapDirection.Buy,
}).sendAndConfirm(umi);

console.log('Buy executed! Tokens received:', quote.amountOut.toString());

// --- Step 6: Unwrap leftover wSOL → SOL ---
await closeToken(umi, {
  account: userQuoteAta,
  destination: umi.identity.publicKey,
  owner: umi.identity,
}).sendAndConfirm(umi);
```

> **Field names matter**: use `amount` (not `amountIn`) and `minAmountOutScaled` (not `minAmountOut`).
> **SwapDirection is an enum**: use `SwapDirection.Buy` / `.Sell`, never strings `'buy'` / `'sell'`.
> For **sells**, reverse the flow: no wSOL wrapping needed upfront, but close the wSOL ATA after to reclaim SOL.

### Bonding Curve Key Points

- **No deposit window** — trading starts immediately after creation
- **Constant product AMM** — `x × y = k` with virtual reserves for bounded pricing
- **Auto-graduation** — fires when all tokens sold, migrates SOL to Raydium CPMM pool
- **Two fee types** — protocol fee + optional creator fee, both on SOL side, don't compound. First buy waives all fees.
- **wSOL is manual** — `swapBondingCurveV2` does not wrap/unwrap native SOL
- **Re-fetch bucket before every swap** in production — price changes with every trade
- Between `isSoldOut` and `isGraduated`, the curve is closed but Raydium pool not yet funded

---

## Program ID

```
Genesis: GNS1S5J5AspKXgpjz6SvKL66kPaKWAhaGRhCqPRxii2B
```

## Documentation

Full documentation: https://metaplex.com/docs/smart-contracts/genesis
