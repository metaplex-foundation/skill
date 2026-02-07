# Genesis CLI Reference

Commands for creating and managing token launches (TGEs) via the `mplx` CLI.

> **Prerequisites**: Run Initial Setup from `./cli.md` first (RPC, keypair, SOL balance).
> **Concepts**: For lifecycle, fees, condition objects, and end behaviors, see `./concepts.md` Genesis section.
> **Docs**: https://developers.metaplex.com/genesis

---

## Lifecycle

```
create → bucket add-* → finalize → deposit → transition → claim → revoke
```

---

## Commands

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

Pro-rata allocation: users deposit SOL, receive tokens proportionally.

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

### `mplx genesis bucket fetch <GENESIS>`

| Flag | Short | Required | Default | Description |
|------|-------|----------|---------|-------------|
| `--bucketIndex` | `-b` | No | `0` | Bucket index |
| `--type` | `-t` | No | `launch-pool` | `launch-pool`, `presale`, or `unlocked` |

### Other Commands

All take `<GENESIS>` as positional argument:

| Command | Key Flags | Description |
|---------|-----------|-------------|
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

### Launch Pool (Fair Launch)

```bash
# 1. Create Genesis account
mplx genesis create --name "My Token" --symbol "MTK" --totalSupply 1000000000000000

# 2. Add unlocked bucket as end-behavior destination (allocation for remaining supply)
mplx genesis bucket add-unlocked <GENESIS> \
  --recipient <TEAM_WALLET> --claimStart <TS> --allocation 200000000000000

# 3. Add launch pool bucket with endBehavior (required for finalize)
#    Note: claimStart must be strictly AFTER depositEnd
mplx genesis bucket add-launch-pool <GENESIS> \
  --allocation 800000000000000 --bucketIndex 1 \
  --depositStart <TS> --depositEnd <TS> --claimStart <TS+1> --claimEnd <TS> \
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

- All timestamps are **Unix seconds** (not milliseconds).
- All amounts are in **base units** (with 9 decimals: 1 SOL = 1000000000).
- Cannot add buckets after `finalize`.
- `finalize` and `revoke` are **irreversible**.
- **`finalize` requires 100% supply allocation** — all tokens must be assigned to buckets. Add unlocked buckets for any remainder.
- **`claimStart` must be strictly after `depositEnd`** — setting them equal causes an error.
- **`--endBehavior` is required on launch pool buckets** for `finalize` to succeed.
- Default deposit/withdraw fees: 200 bps (2%). See: https://developers.metaplex.com/protocol-fees
- No `--wizard` mode — all flags must be provided explicitly.
