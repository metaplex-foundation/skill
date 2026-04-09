# Genesis CLI Reference

Commands for creating and managing token launches (TGEs) via the `mplx` CLI.

> **Prerequisites**: CLI must be configured (RPC, keypair, funded wallet). If not yet verified this session, see `./cli-initial-setup.md`.
> **Concepts**: For lifecycle, fees, condition objects, and end behaviors, see `./concepts.md` Genesis section.
> **Docs**: https://metaplex.com/docs/smart-contracts/genesis

---

## Lifecycle

**Low-level** (manual on-chain setup):

```text
create → bucket add-* → finalize → deposit → transition → graduation → claim → revoke
```

**Launch API** (recommended — handles everything in one command):

```text
Launchpool:     launch create  →  deposit window (48h)  →  Raydium graduation  →  claim
Bonding Curve:  launch create  →  swap (buy/sell)  →  sell-out  →  auto-graduation to Raydium CPMM
```

---

## Commands

### Launch API (Recommended)

Two launch types: **launchpool** (default, fully configurable, 48h deposit) and **bonding-curve** (instant bonding curve).

```bash
# Launchpool (all-in-one, default)
mplx genesis launch create --name <NAME> --symbol <SYMBOL> \
  --image <IRYS_URL> --tokenAllocation <AMOUNT> --depositStartTime <ISO_DATE_OR_UNIX_TS> \
  --raiseGoal <WHOLE_UNITS> --raydiumLiquidityBps <BPS> --fundsRecipient <ADDR>

# Bonding curve launch (instant, no deposit window)
mplx genesis launch create --launchType bonding-curve --name <NAME> --symbol <SYMBOL> \
  --image <IRYS_URL>

# Bonding curve with creator fee and first buy
mplx genesis launch create --launchType bonding-curve --name <NAME> --symbol <SYMBOL> \
  --image <IRYS_URL> --creatorFeeWallet <ADDR> --firstBuyAmount <SOL_AMOUNT>

# Bonding curve with agent (auto-derives creator fee wallet from agent PDA)
mplx genesis launch create --launchType bonding-curve --name <NAME> --symbol <SYMBOL> \
  --image <IRYS_URL> --agentMint <AGENT_ASSET> --agentSetToken

# Register an existing genesis account on the platform
mplx genesis launch register <GENESIS_ACCOUNT> --launchConfig <PATH_TO_JSON>
```

### Low-Level Commands

```bash
# Create Genesis account
mplx genesis create --name <NAME> --symbol <SYMBOL> --totalSupply <AMOUNT>
mplx genesis create --name <NAME> --symbol <SYMBOL> --totalSupply <AMOUNT> --uri <URI> --decimals <N>

# Fetch
mplx genesis fetch <GENESIS>

# Add buckets (before finalize)
mplx genesis bucket add-launch-pool <GENESIS> --allocation <AMOUNT> \
  --depositStart <UNIX_TS> --depositEnd <UNIX_TS> --claimStart <UNIX_TS> --claimEnd <UNIX_TS>
mplx genesis bucket add-presale <GENESIS> --allocation <AMOUNT> --quoteCap <AMOUNT> \
  --depositStart <UNIX_TS> --depositEnd <UNIX_TS> --claimStart <UNIX_TS> --bucketIndex <N>
mplx genesis bucket add-unlocked <GENESIS> --recipient <ADDR> --claimStart <UNIX_TS>
mplx genesis bucket fetch <GENESIS> --bucketIndex <N> --type <TYPE>

# Finalize (irreversible)
mplx genesis finalize <GENESIS>

# Launch pool operations
mplx genesis deposit <GENESIS> --amount <AMOUNT>
mplx genesis withdraw <GENESIS> --amount <AMOUNT>
mplx genesis transition <GENESIS> --bucketIndex <N>
mplx genesis claim <GENESIS>

# Presale operations
mplx genesis presale deposit <GENESIS> --amount <AMOUNT>
mplx genesis presale claim <GENESIS>

# Unlocked bucket
mplx genesis claim-unlocked <GENESIS>

# Revoke authorities (irreversible)
mplx genesis revoke <GENESIS> --revokeMint
mplx genesis revoke <GENESIS> --revokeFreeze
mplx genesis revoke <GENESIS> --revokeMint --revokeFreeze
```

---

## Command Details

### `mplx genesis launch create`

All-in-one command: creates the token, sets up the genesis account with a launch pool, optionally adds locked (vesting) allocations, signs and sends transactions, then registers the launch on the Metaplex platform. Returns a public launch page link.

**Launch types:**
- **`launchpool`** (default): Total supply 1B, 48-hour deposit period, fully configurable allocations. Requires `--tokenAllocation`, `--raiseGoal`, `--raydiumLiquidityBps`, `--fundsRecipient`.
- **`bonding-curve`**: Instant bonding curve (constant product AMM). No deposit window — trading starts immediately. Optional creator fees, first buy, and agent mode. Only needs `--name`, `--symbol`, `--image`.

| Flag | Short | Required | Default | Description |
|------|-------|----------|---------|-------------|
| `--launchType` | - | No | `launchpool` | `launchpool` or `bonding-curve` |
| `--name` | `-n` | Yes | - | Token name (1-32 characters) |
| `--symbol` | `-s` | Yes | - | Token symbol (1-10 characters) |
| `--image` | - | Yes | - | Token image URL (must be `https://gateway.irys.xyz/...`) |
| `--depositStartTime` | - | Launchpool only | - | ISO date string or unix timestamp. 48h deposit window. Not used for bonding-curve. |
| `--tokenAllocation` | - | Launchpool only | - | Launch pool token allocation (portion of 1B total supply) |
| `--raiseGoal` | - | Launchpool only | - | Minimum quote tokens to raise, in **whole units** (e.g., `200` = 200 SOL) |
| `--raydiumLiquidityBps` | - | Launchpool only | - | Basis points for Raydium LP (2000-10000, i.e., 20%-100%) |
| `--fundsRecipient` | - | Launchpool only | - | Wallet receiving the unlocked portion of raised funds |
| `--creatorFeeWallet` | - | No (bonding-curve only) | Launching wallet | Wallet to receive creator fees from swaps |
| `--firstBuyAmount` | - | No (bonding-curve only) | - | SOL amount for fee-free initial purchase at launch |
| `--agentMint` | - | No | - | Agent NFT (Core asset) address. Wraps launch transactions in Core execute instructions. Auto-derives creator fee wallet from agent PDA. |
| `--agentSetToken` | - | No | `false` | Set the launched token as the agent's primary token. **Irreversible.** Requires `--agentMint`. |
| `--description` | - | No | - | Token description (max 250 characters) |
| `--website` | - | No | - | Project website URL |
| `--twitter` | - | No | - | Project Twitter URL |
| `--telegram` | - | No | - | Project Telegram URL |
| `--creatorWallet` | - | No | - | Override the launch owner wallet for registration (public key) |
| `--twitterVerificationToken` | - | No | - | Twitter verification token for verified badge on the launch page |
| `--lockedAllocations` | - | No (launchpool only) | - | Path to JSON file with locked allocation configs (Streamflow vesting) |
| `--quoteMint` | - | No | `SOL` | `SOL`, `USDC`, or a mint address |
| `--network` | - | No | auto-detected | `solana-mainnet` or `solana-devnet` |
| `--apiUrl` | - | No | `https://api.metaplex.com` | Genesis API base URL |

**Output**: Genesis account address, mint address, launch ID, launch link, token ID, transaction signatures.

**Locked Allocations JSON** (`--lockedAllocations` file format):
```json
[
  {
    "name": "Team",
    "recipient": "<ADDRESS>",
    "tokenAmount": 100000000,
    "vestingStartTime": "2026-04-05T00:00:00Z",
    "vestingDuration": { "value": 1, "unit": "YEAR" },
    "unlockSchedule": "MONTH",
    "cliff": {
      "duration": { "value": 3, "unit": "MONTH" },
      "unlockAmount": 10000000
    }
  }
]
```

TimeUnit values: `SECOND`, `MINUTE`, `HOUR`, `DAY`, `WEEK`, `TWO_WEEKS`, `MONTH`, `QUARTER`, `YEAR`.

### `mplx genesis launch register <GENESIS_ACCOUNT>`

Registers an existing genesis account (created via low-level commands or SDK) with the Metaplex platform to get a public launch page.

| Flag | Short | Required | Default | Description |
|------|-------|----------|---------|-------------|
| `--launchConfig` | - | Yes | - | Path to JSON file with launch configuration |
| `--creatorWallet` | - | No | - | Override the launch owner wallet for registration (public key) |
| `--twitterVerificationToken` | - | No | - | Twitter verification token for verified badge on the launch page |
| `--network` | - | No | auto-detected | `solana-mainnet` or `solana-devnet` |
| `--apiUrl` | - | No | `https://api.metaplex.com` | Genesis API base URL |

**Launch Config JSON** — launchpool example (`--launchConfig` file format):
```json
{
  "wallet": "<ADDRESS>",
  "token": {
    "name": "My Token",
    "symbol": "MTK",
    "image": "https://gateway.irys.xyz/...",
    "description": "Optional description",
    "externalLinks": {
      "website": "https://...",
      "twitter": "https://...",
      "telegram": "https://..."
    }
  },
  "launchType": "launchpool",
  "launch": {
    "launchpool": {
      "tokenAllocation": 500000000,
      "depositStartTime": "<FUTURE_ISO_DATE>",
      "raiseGoal": 200,
      "raydiumLiquidityBps": 5000,
      "fundsRecipient": "<ADDRESS>"
    },
    "lockedAllocations": []
  },
  "network": "solana-mainnet",
  "quoteMint": "SOL"
}
```

**Launch Config JSON** — bonding curve example:

```json
{
  "wallet": "<ADDRESS>",
  "token": {
    "name": "My Token",
    "symbol": "MTK",
    "image": "https://gateway.irys.xyz/..."
  },
  "launchType": "bondingCurve",
  "launch": {
    "creatorFeeWallet": "<FEE_WALLET_ADDRESS>",
    "firstBuyAmount": 0.1
  },
  "network": "solana-mainnet",
  "quoteMint": "SOL"
}
```

**Launch Config JSON** — bonding curve with agent:

```json
{
  "wallet": "<ADDRESS>",
  "token": {
    "name": "Agent Token",
    "symbol": "AGT",
    "image": "https://gateway.irys.xyz/..."
  },
  "launchType": "bondingCurve",
  "agent": {
    "mint": "<AGENT_CORE_ASSET_ADDRESS>",
    "setToken": true
  },
  "launch": {},
  "network": "solana-mainnet",
  "quoteMint": "SOL"
}
```

**Output**: Launch ID, launch link, token ID, mint address.

---

### `mplx genesis create`

| Flag | Short | Required | Default | Description |
|------|-------|----------|---------|-------------|
| `--name` | `-n` | Yes | - | Token name |
| `--symbol` | `-s` | Yes | - | Token symbol (e.g., MTK) |
| `--totalSupply` | - | Yes | - | Total supply in base units |
| `--uri` | `-u` | No | `""` | Metadata JSON URI |
| `--decimals` | `-d` | No | `9` | Token decimals |
| `--quoteMint` | - | No | Wrapped SOL | Quote token mint address |
| `--fundingMode` | - | No | `new-mint` | `new-mint` or `transfer` (use existing mint) |
| `--baseMint` | - | No | - | Existing mint address (required when `fundingMode=transfer`) |
| `--genesisIndex` | - | No | `0` | Index for multiple launches on same mint |

All amounts are in **base units**. With 9 decimals: 1M tokens = `1000000000000000`.

### `mplx genesis bucket add-launch-pool <GENESIS>`

Pro-rata allocation: users deposit SOL, receive tokens proportionally. Internally sends up to 3 transactions: (1) base bucket creation, (2) optional extensions, (3) end behaviors.

| Flag | Short | Required | Default | Description |
|------|-------|----------|---------|-------------|
| `--allocation` | `-a` | Yes | - | Token allocation for this bucket (base units) |
| `--depositStart` | - | Yes | - | Unix timestamp (seconds) |
| `--depositEnd` | - | Yes | - | Unix timestamp (seconds) |
| `--claimStart` | - | Yes | - | Unix timestamp (seconds) |
| `--claimEnd` | - | Yes | - | Unix timestamp (seconds) |
| `--bucketIndex` | `-b` | No | `0` | Bucket index |
| `--endBehavior` | - | No* | - | `<bucketAddress>:<percentageBps>` (repeatable, 10000 = 100%). *Required for `finalize` to succeed |
| `--minimumDeposit` | - | No | - | Min deposit per transaction (base units) |
| `--depositLimit` | - | No | - | Max deposit per user (base units) |
| `--minimumQuoteTokenThreshold` | - | No | - | Min total deposits for bucket to succeed |
| `--depositPenalty` | - | No | - | JSON: `{"slopeBps":0,"interceptBps":200,"maxBps":200,"startTime":0,"endTime":0}` |
| `--withdrawPenalty` | - | No | - | Same JSON format as depositPenalty |
| `--bonusSchedule` | - | No | - | Same JSON format as depositPenalty |
| `--claimSchedule` | - | No | - | JSON: `{"startTime":0,"endTime":0,"period":0,"cliffTime":0,"cliffAmountBps":0}` |
| `--allowlist` | - | No | - | JSON: `{"merkleTreeHeight":10,"merkleRoot":"<hex>","endTime":0,"quoteCap":0}` |

### `mplx genesis bucket add-presale <GENESIS>`

Fixed-price allocation: price = quoteCap / allocation.

| Flag | Short | Required | Default | Description |
|------|-------|----------|---------|-------------|
| `--allocation` | `-a` | Yes | - | Token allocation (base units) |
| `--quoteCap` | - | Yes | - | Total quote tokens accepted (determines price) |
| `--depositStart` | - | Yes | - | Unix timestamp |
| `--depositEnd` | - | Yes | - | Unix timestamp |
| `--claimStart` | - | Yes | - | Unix timestamp |
| `--bucketIndex` | `-b` | Yes | - | Bucket index |
| `--claimEnd` | - | No | Year 2100 | Unix timestamp |
| `--minimumDeposit` | - | No | - | Min deposit per transaction |
| `--depositLimit` | - | No | - | Max deposit per user |

### `mplx genesis bucket add-unlocked <GENESIS>`

Treasury/team allocation. No deposits — tokens go directly to recipient.

| Flag | Short | Required | Default | Description |
|------|-------|----------|---------|-------------|
| `--recipient` | - | Yes | - | Address that can claim tokens |
| `--claimStart` | - | Yes | - | Unix timestamp |
| `--allocation` | `-a` | No | `0` | Token allocation (base units) |
| `--bucketIndex` | `-b` | No | `0` | Bucket index |
| `--claimEnd` | - | No | Year 2100 | Unix timestamp |

### `mplx genesis swap <GENESIS>`

Swap on a Genesis bonding curve. Buy tokens with quote tokens (e.g. SOL) or sell tokens back. The bonding curve uses a constant-product formula for pricing. Auto-wraps SOL to WSOL when buying with native SOL.

Use `--info` to display curve status and price quotes without executing a swap. Combine `--info` with `--buyAmount` or `--sellAmount` to get a quote without swapping.

| Flag | Short | Required | Default | Description |
|------|-------|----------|---------|-------------|
| `--buyAmount` | - | One of buy/sell required (unless `--info`) | - | Amount of quote tokens to spend (e.g. lamports for SOL) |
| `--sellAmount` | - | One of buy/sell required (unless `--info`) | - | Amount of base tokens to sell |
| `--slippage` | - | No | `200` (2%) | Slippage tolerance in basis points |
| `--bucketIndex` | `-b` | No | `0` | Index of the bonding curve bucket |
| `--info` | - | No | `false` | Display curve status and price quotes without executing a swap |

**Examples:**
```bash
# View curve info (price, reserves, fill %, swappable)
mplx genesis swap <GENESIS> --info

# Get a buy quote (no swap executed)
mplx genesis swap <GENESIS> --info --buyAmount 100000000

# Get a sell quote (no swap executed)
mplx genesis swap <GENESIS> --info --sellAmount 1000000000

# Buy tokens (0.05 SOL, auto-wraps SOL to WSOL)
mplx genesis swap <GENESIS> --buyAmount 50000000

# Buy with 1% slippage
mplx genesis swap <GENESIS> --buyAmount 50000000 --slippage 100

# Sell tokens
mplx genesis swap <GENESIS> --sellAmount 500000000000
```

**Output (swap)**: Genesis account, bucket, direction, amount in, expected out, min out (with slippage), fee, creator fee, transaction signature.

**Output (info)**: Tokens per quote unit, quote per token, reserves, fill %, first buy pending, swappable, sold out. When combined with `--buyAmount` or `--sellAmount`, also shows the quote details (amount out, fees, min out).

### `mplx genesis bucket fetch <GENESIS>`

Auto-detects the bucket type by trying all known types at the given index. Use `--type` to specify explicitly.

| Flag | Short | Required | Default | Description |
|------|-------|----------|---------|-------------|
| `--bucketIndex` | `-b` | No | `0` | Bucket index |
| `--type` | `-t` | No | auto-detect | `launch-pool`, `presale`, `unlocked`, or `bonding-curve` |

### Other Commands

All take `<GENESIS>` as positional argument:

| Command | Key Flags | Description |
|---------|-----------|-------------|
| `swap` | `--buyAmount` or `--sellAmount`, `--slippage`, `--info` | Buy/sell on bonding curve, or view curve info/quotes |
| `deposit` | `--amount` (required), `--bucketIndex` | Deposit into launch pool |
| `withdraw` | `--amount` (required), `--bucketIndex` | Withdraw from launch pool |
| `transition` | `--bucketIndex` (required) | Execute end behaviors after deposit period |
| `claim` | `--bucketIndex`, `--recipient` | Claim from launch pool |
| `claim-unlocked` | `--bucketIndex`, `--recipient` | Claim from unlocked bucket |
| `presale deposit` | `--amount` (required), `--bucketIndex` | Deposit into presale |
| `presale claim` | `--bucketIndex`, `--recipient` | Claim from presale |
| `revoke` | `--revokeMint`, `--revokeFreeze` | Revoke authorities (at least one required) |

---

## Workflows

### Launch via API (Recommended)

```bash
# Bonding curve launch — instant trading, no deposit window
mplx genesis launch create --launchType bonding-curve \
  --name "My Token" \
  --symbol "MTK" \
  --image "https://gateway.irys.xyz/abc123"

# Bonding curve with creator fee wallet and first buy
mplx genesis launch create --launchType bonding-curve \
  --name "My Token" \
  --symbol "MTK" \
  --image "https://gateway.irys.xyz/abc123" \
  --creatorFeeWallet <FEE_WALLET> \
  --firstBuyAmount 0.1

# Bonding curve with agent (auto fee wallet + set token)
mplx genesis launch create --launchType bonding-curve \
  --name "Agent Token" \
  --symbol "AGT" \
  --image "https://gateway.irys.xyz/abc123" \
  --agentMint <AGENT_ASSET> --agentSetToken

# Launchpool — configurable allocations, 48-hour deposit window
mplx genesis launch create \
  --name "My Token" \
  --symbol "MTK" \
  --image "https://gateway.irys.xyz/abc123" \
  --tokenAllocation 500000000 \
  --depositStartTime "<FUTURE_ISO_DATE>" \
  --raiseGoal 200 \
  --raydiumLiquidityBps 5000 \
  --fundsRecipient <WALLET_ADDRESS>

# Launchpool with agent mode
mplx genesis launch create \
  --name "My Token" \
  --symbol "MTK" \
  --image "https://gateway.irys.xyz/abc123" \
  --tokenAllocation 500000000 \
  --depositStartTime "<FUTURE_ISO_DATE>" \
  --raiseGoal 200 \
  --raydiumLiquidityBps 5000 \
  --fundsRecipient <WALLET_ADDRESS> \
  --agentMint <AGENT_ASSET> --agentSetToken

# Launchpool with optional metadata
mplx genesis launch create \
  --name "My Token" \
  --symbol "MTK" \
  --image "https://gateway.irys.xyz/abc123" \
  --tokenAllocation 500000000 \
  --depositStartTime "<FUTURE_ISO_DATE>" \
  --raiseGoal 200 \
  --raydiumLiquidityBps 5000 \
  --fundsRecipient <WALLET_ADDRESS> \
  --description "A revolutionary token" \
  --website "https://mytoken.com" \
  --twitter "https://twitter.com/mytoken" \
  --telegram "https://t.me/mytoken"

# Launchpool with team vesting (locked allocations)
mplx genesis launch create \
  --name "My Token" \
  --symbol "MTK" \
  --image "https://gateway.irys.xyz/abc123" \
  --tokenAllocation 500000000 \
  --depositStartTime "<FUTURE_ISO_DATE>" \
  --raiseGoal 200 \
  --raydiumLiquidityBps 5000 \
  --fundsRecipient <WALLET_ADDRESS> \
  --lockedAllocations allocations.json

# Register an existing genesis account
mplx genesis launch register <GENESIS_ACCOUNT> --launchConfig launch.json
```

### Bonding Curve Swap

After a bonding curve launch is live, use `genesis swap` to trade or inspect the curve.

```bash
# Check curve status before trading
mplx genesis swap <GENESIS> --info

# Get a buy quote for 0.1 SOL without swapping
mplx genesis swap <GENESIS> --info --buyAmount 100000000

# Get a sell quote for 1000 tokens (9 decimals) without swapping
mplx genesis swap <GENESIS> --info --sellAmount 1000000000000

# Buy tokens with 0.05 SOL (auto-wraps SOL to WSOL)
mplx genesis swap <GENESIS> --buyAmount 50000000

# Buy with tighter slippage (1%)
mplx genesis swap <GENESIS> --buyAmount 50000000 --slippage 100

# Sell tokens back for SOL
mplx genesis swap <GENESIS> --sellAmount 500000000000
```

> **`--info` alone** shows price, reserves, fill %, and whether the curve is swappable.
> **`--info` + `--buyAmount`** adds a buy quote: tokens out, fees, min out at current slippage.
> **`--info` + `--sellAmount`** adds a sell quote: SOL out, fees, min out at current slippage.
> Without `--info`, `--buyAmount` or `--sellAmount` executes the swap on-chain.
> Amounts are in **base units**: 1 SOL = 1000000000 lamports, token amounts depend on decimals (default 9).

### Launch Pool (Fair Launch — Low-Level)

```bash
# 1. Create Genesis account
mplx genesis create --name "My Token" --symbol "MTK" --totalSupply 1000000000000000

# 2. Add unlocked bucket as end-behavior destination (allocation for remaining supply)
mplx genesis bucket add-unlocked <GENESIS> \
  --recipient <TEAM_WALLET> --claimStart <CLAIM_START_TS> --allocation 200000000000000

# 3. Add launch pool bucket with endBehavior (required for finalize)
#    Note: DEPOSIT_END_TS < CLAIM_START_TS < CLAIM_END_TS (strict ordering required)
mplx genesis bucket add-launch-pool <GENESIS> \
  --allocation 800000000000000 --bucketIndex 1 \
  --depositStart <DEPOSIT_START_TS> --depositEnd <DEPOSIT_END_TS> \
  --claimStart <CLAIM_START_TS> --claimEnd <CLAIM_END_TS> \
  --endBehavior "<UNLOCKED_BUCKET_ADDR>:10000"

# 4. Finalize (irreversible — requires 100% supply allocated)
mplx genesis finalize <GENESIS>

# 5. Users deposit
mplx genesis deposit <GENESIS> --amount 10000000000 --bucketIndex 1  # 10 SOL

# 6. Users claim tokens (after claim period starts)
mplx genesis claim <GENESIS> --bucketIndex 1

# 7. (Optional) Revoke authorities
mplx genesis revoke <GENESIS> --revokeMint --revokeFreeze
```

### Presale

```bash
mplx genesis create --name "My Token" --symbol "MTK" --totalSupply 1000000000000000

# Presale bucket (partial supply)
mplx genesis bucket add-presale <GENESIS> \
  --allocation 400000000000000 --quoteCap 1000000000000 \
  --depositStart <TS> --depositEnd <TS> --claimStart <TS> --bucketIndex 0

# Must allocate remaining supply (finalize requires 100%)
mplx genesis bucket add-unlocked <GENESIS> \
  --recipient <TEAM_WALLET> --allocation 600000000000000 --claimStart <TS> --bucketIndex 1

mplx genesis finalize <GENESIS>
mplx genesis presale deposit <GENESIS> --amount 5000000000
mplx genesis presale claim <GENESIS>
```

### Treasury Only

```bash
mplx genesis create --name "My Token" --symbol "MTK" --totalSupply 1000000000000000
mplx genesis bucket add-unlocked <GENESIS> \
  --recipient <WALLET> --allocation 1000000000000000 --claimStart <TS>
mplx genesis finalize <GENESIS>
mplx genesis claim-unlocked <GENESIS>
```

---

## Important Notes

- Default deposit/withdraw fees: 200 bps (2%). See: https://metaplex.com/docs/protocol-fees
- Launch API `--raiseGoal` is in **whole units** (e.g., `200` = 200 SOL), NOT base units.
- Low-level commands use **base units** (1 SOL = 1000000000) and **Unix seconds** (not milliseconds).
- Cannot add buckets after `finalize`. `finalize` and `revoke` are **irreversible**.
- **`finalize` requires 100% supply allocation** — add unlocked buckets for any remainder.
- **`claimStart` must be strictly after `depositEnd`** — setting them equal causes an error.
- **`--endBehavior` is required on launch pool buckets** for `finalize` to succeed.
- Low-level commands have no `--wizard` mode — all flags must be provided explicitly.
- **Bonding curve** launches have no deposit window — trading starts immediately after creation. Graduation to Raydium CPMM fires automatically when all tokens are sold.
- **`--agentSetToken` is irreversible** — permanently links the launched token to the agent. Cannot be undone.
- **`--agentMint`** auto-derives the creator fee wallet from the agent's Core asset signer PDA (`['mpl-core-execute', <agent_mint>]`). The first buy buyer also defaults to the agent PDA.
- **`--firstBuyAmount`** is fee-free (no protocol or creator fee) and is executed as part of the launch transaction. Only applies to bonding curve launches.
- **`--agentMint` RPC propagation**: The Genesis API verifies agent ownership on-chain. Its backend may lag behind the CLI's RPC after a fresh agent registration. If the error says "Agent is not owned", the on-chain launch may still have succeeded — check with `agents fetch`. If the agent already has a token set, only the platform registration failed; complete it with `genesis launch register`. When scripting, add a ~30 second delay between `agents register` and `genesis launch create`.
