# Agent Registry CLI Reference

Commands for registering agent identities and delegating execution via the `mplx` CLI.

> **Prerequisites**: CLI must be configured (RPC, keypair, funded wallet). If not yet verified this session, see `./cli-initial-setup.md`.
> **Docs**: https://metaplex.com/docs/agents

---

## Commands

### Register Agent Identity

Creates a Core asset and registers an on-chain agent identity. By default uses the Metaplex Agent API (single transaction, no Irys upload needed). Use `--use-ix` to send the `registerIdentityV1` instruction directly instead.

```bash
# API (default) — creates asset + registers identity in one transaction
mplx agents register --name "My Agent" --description "An autonomous agent" \
  --image "./avatar.png"

# API with services and trust models
mplx agents register --name "My Agent" --description "An autonomous agent" \
  --image "./avatar.png" \
  --services '[{"name":"MCP","endpoint":"https://example.com/mcp"}]' \
  --supported-trust '["reputation","crypto-economic"]'

# Direct IX — existing asset with local registration document
mplx agents register <ASSET> --use-ix --from-file "./agent-doc.json"

# Direct IX — existing asset with inline flags
mplx agents register <ASSET> --use-ix --name "My Agent" \
  --description "An autonomous agent" --image "./avatar.png"

# Direct IX — new asset with inline flags
mplx agents register --new --use-ix --name "My Agent" \
  --description "An autonomous agent" --image "./avatar.png"

# Direct IX — interactive wizard
mplx agents register --new --wizard
```

| Flag | Description |
|------|-------------|
| `--use-ix` | Send the `registerIdentityV1` instruction directly instead of using the API |
| `--new` | Create a new Core asset (only needed with `--use-ix`) |
| `--owner` | Owner public key for the new asset (defaults to signer, only with `--new`) |
| `--collection` | Collection address the asset belongs to |
| `--wizard` | Interactive wizard to build the registration document (implies `--use-ix`) |
| `--from-file <path>` | Path to a local agent registration JSON file to upload (implies `--use-ix`) |
| `--name` | Agent name |
| `--description` | Agent description |
| `--image` | Agent image file path (uploaded) or existing URI |
| `--active` | Set agent as active (default: true) |
| `--services` | Service endpoints as JSON array (e.g. `'[{"name":"MCP","endpoint":"https://..."}]'`) |
| `--supported-trust` | Trust models as JSON array (e.g. `'["reputation","tee-attestation"]'`) |
| `--save-document` | Save the generated document JSON to a local file |

> `--wizard`, `--from-file`, and `--name` are mutually exclusive — use one approach to provide the registration document.
> Passing an asset address or `--from-file`/`--wizard` automatically implies `--use-ix`.
> The API path detects the network from your configured RPC endpoint.

### Fetch Agent Identity

Reads the on-chain agent identity PDA and displays registration info, lifecycle hooks, and the agent's wallet.

```bash
mplx agents fetch <ASSET>
```

Returns: `registered`, `asset`, `owner`, `identityPda`, `wallet` (Asset Signer PDA), `registrationUri`, `lifecycleChecks`.

### Register Executive Profile

Creates a one-time on-chain executive profile for the current wallet. Required before any delegation.

```bash
mplx agents executive register
```

> Each wallet can only have one executive profile. Calling this again will fail.

### Delegate Execution

Links a registered agent to an executive profile, allowing the executive to sign transactions on behalf of the agent.

```bash
mplx agents executive delegate <ASSET> --executive <EXECUTIVE_WALLET>
```

| Flag | Description |
|------|-------------|
| `--executive` | The executive's wallet address (profile PDA derived automatically) |

> Only the asset owner can delegate. The agent must be registered and the executive must have a profile.

### Revoke Execution

Removes an existing execution delegation. Either the asset owner or the executive can revoke. Rent from the closed delegation record is refunded.

```bash
# Executive revoking their own delegation (--executive defaults to signer)
mplx agents executive revoke <ASSET>

# Asset owner revoking a specific executive's delegation
mplx agents executive revoke <ASSET> --executive <EXECUTIVE_WALLET>
```

| Flag | Description |
|------|-------------|
| `--executive` | The executive's wallet address (defaults to the current signer) |
| `--destination` | Wallet to receive refunded rent (defaults to the current signer) |

### Set Agent Token

Links a Genesis token to a registered agent identity. Must be run in asset-signer mode.

```bash
mplx agents set-agent-token <ASSET> <GENESIS_ACCOUNT>
```

> Requires asset-signer mode. Configure with `mplx config wallets add --name my-agent --type asset-signer --asset <ASSET>` then `mplx config wallets set my-agent`.

---

## Typical Workflows

### Register a New Agent (Quick)

```bash
# One command via the API — creates asset + registers identity
mplx agents register --name "My Agent" \
  --description "An autonomous trading agent on Solana" \
  --image "./avatar.png"
```

### Register + Delegate Execution

```bash
# 1. Register agent
mplx agents register --name "My Agent" \
  --description "An autonomous agent" --image "./avatar.png"
# Note the asset address from output

# 2. Register executive profile (one-time, on the executive's machine)
mplx agents executive register

# 3. Delegate execution (on the owner's machine)
mplx agents executive delegate <ASSET> --executive <EXECUTIVE_WALLET>
```

### Verify an Agent

```bash
mplx agents fetch <ASSET>
```

### Register Agent + Launch Token (Bonding Curve)

End-to-end workflow: register an agent and launch a bonding curve token linked to the agent. The `--agentMint` flag auto-derives the creator fee wallet from the agent's Core asset signer PDA.

```bash
# 1. Register agent
mplx agents register --name "My Agent" \
  --description "An autonomous trading agent" --image "./avatar.png"
# Note the asset address from output (e.g., 7BQj...)

# 2. Launch bonding curve token linked to the agent
mplx genesis launch create --launchType bonding-curve \
  --name "Agent Token" --symbol "AGT" \
  --image "https://gateway.irys.xyz/..." \
  --agentMint <ASSET> --agentSetToken

# 3. (Optional) Verify the agent has a token
mplx agents fetch <ASSET>
```

> `--agentSetToken` is **irreversible** — permanently links the token to the agent. Omit it to launch without linking, then link later with `mplx agents set-agent-token`.
> See `./cli-genesis.md` for full bonding curve launch options (creator fees, first buy, etc.).
> **RPC propagation**: If step 2 fails with "Agent is not owned by the connected wallet", the API's backend hasn't indexed the agent yet. The on-chain token creation may still have succeeded — check with `mplx agents fetch <ASSET>`. If the agent already shows a token set on retry, only the platform registration failed; complete it with `mplx genesis launch register`. When scripting, add a ~30 second delay between agent registration and the launch command.

### Register Agent + Launch Token (Launchpool)

```bash
# 1. Register agent
mplx agents register --name "My Agent" \
  --description "An agent with a community token" --image "./avatar.png"

# 2. Launch launchpool token linked to the agent
mplx genesis launch create \
  --name "Agent Token" --symbol "AGT" \
  --image "https://gateway.irys.xyz/..." \
  --tokenAllocation 500000000 \
  --depositStartTime "<FUTURE_ISO_DATE>" \
  --raiseGoal 200 --raydiumLiquidityBps 5000 \
  --fundsRecipient <WALLET> \
  --agentMint <ASSET> --agentSetToken
```

---

## Agent Registration Document

When using `--use-ix`, the `--name`/`--from-file`/`--wizard` flags produce an [ERC-8004](https://eips.ethereum.org/EIPS/eip-8004) JSON document uploaded to Arweave. The API path handles metadata storage on metaplex.com automatically. See `./sdk-agent.md` for the full field reference.

Service types: `web`, `A2A`, `MCP`, `OASF`, `DID`, `email`, or custom.
Trust models: `reputation`, `crypto-economic`, `tee-attestation`.
