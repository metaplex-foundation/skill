# Agent Registry SDK Reference (Umi)

Umi SDK operations for registering agent identities, reading agent data, and delegating execution.

> **Prerequisites**: Set up Umi first — see `./sdk-umi.md` for installation and basic setup.
> **Docs**: https://metaplex.com/docs/agents

> **Agents are MPL Core assets.** Before registering an agent, you need an MPL Core asset. See `./sdk-core.md` for creating assets and collections.

---

## Installation

```shell
npm install @metaplex-foundation/mpl-agent-registry
```

> **CJS/ESM interop**: `mpl-agent-registry` ships as CommonJS. In `.mjs` files or ESM projects, use: `import pkg from '@metaplex-foundation/mpl-agent-registry'; const { mintAndSubmitAgent } = pkg;`

## Setup

```typescript
import { createUmi } from '@metaplex-foundation/umi-bundle-defaults';
import { mplCore } from '@metaplex-foundation/mpl-core';
import { mplAgentIdentity, mplAgentTools } from '@metaplex-foundation/mpl-agent-registry';

const umi = createUmi('https://api.mainnet-beta.solana.com')
  .use(mplCore())
  .use(mplAgentIdentity())
  .use(mplAgentTools());
```

---

## Mint Agent API (Recommended)

The Metaplex Agent API creates an MPL Core asset and registers its identity in a single transaction. This is the recommended path for new agents.

### Mint and Submit in One Call

```typescript
import { createUmi } from '@metaplex-foundation/umi-bundle-defaults';
import { mintAndSubmitAgent } from '@metaplex-foundation/mpl-agent-registry';

const umi = createUmi('https://api.mainnet-beta.solana.com');

const result = await mintAndSubmitAgent(umi, {}, {
  wallet: umi.identity.publicKey,
  name: 'My AI Agent',
  uri: 'https://example.com/agent-metadata.json',
  agentMetadata: {
    type: 'agent',
    name: 'My AI Agent',
    description: 'An autonomous agent that executes DeFi strategies on Solana.',
    services: [
      { name: 'web', endpoint: 'https://myagent.ai' },
      { name: 'A2A', endpoint: 'https://myagent.ai/agent-card.json' },
    ],
    registrations: [],
    supportedTrust: ['reputation'],
  },
});

console.log('Agent minted! Asset:', result.assetAddress);
console.log('Signature:', result.signature);
```

| Parameter | Required | Description |
|-----------|----------|-------------|
| `wallet` | Yes | The agent owner's wallet public key (signs the transaction) |
| `name` | Yes | Display name for the Core asset |
| `uri` | Yes | Metadata URI for the Core asset |
| `agentMetadata` | Yes | `{ type, name, description, services: [{name, endpoint}], registrations: [{agentId, agentRegistry}], supportedTrust: string[] }` |
| `network` | No | Target network (defaults to `solana-mainnet`) |

Returns: `{ signature: Uint8Array, assetAddress: string }`

### Mint with Separate Signing

Use `mintAgent` for custom signing flows (hardware wallets, multi-sig):

```typescript
import { mintAgent, signAndSendAgentTransaction } from '@metaplex-foundation/mpl-agent-registry';

const mintResult = await mintAgent(umi, {}, {
  wallet: umi.identity.publicKey,
  name: 'My AI Agent',
  uri: 'https://example.com/agent-metadata.json',
  agentMetadata: {
    type: 'agent',
    name: 'My AI Agent',
    description: 'An autonomous agent.',
    services: [],
    registrations: [],
    supportedTrust: [],
  },
});

const signature = await signAndSendAgentTransaction(umi, mintResult);
```

Returns: `{ transaction: Transaction, blockhash: BlockhashWithExpiryBlockHeight, assetAddress: string }`

### API Configuration

Pass an `AgentApiConfig` object as the second argument:

| Option | Default | Description |
|--------|---------|-------------|
| `baseUrl` | `https://api.metaplex.com` | Base URL of the Metaplex API |
| `fetch` | `globalThis.fetch` | Custom fetch implementation |

### Supported Networks

`solana-mainnet` (default), `solana-devnet`, `localnet`, `eclipse-mainnet`, `sonic-mainnet`, `sonic-devnet`, `fogo-mainnet`, `fogo-testnet`

### API Error Handling

```typescript
import { isAgentApiError, isAgentApiNetworkError, isAgentValidationError } from '@metaplex-foundation/mpl-agent-registry';

try {
  const result = await mintAndSubmitAgent(umi, {}, input);
} catch (err) {
  if (isAgentValidationError(err)) {
    console.error('Validation error:', err.field, err.message);
  } else if (isAgentApiError(err)) {
    console.error('API error:', err.statusCode, err.responseBody);
  } else if (isAgentApiNetworkError(err)) {
    console.error('Network error:', err.cause.message);
  }
}
```

| Error Type | Description |
|------------|-------------|
| `AgentApiError` | HTTP response error — includes `statusCode` and `responseBody` |
| `AgentApiNetworkError` | Network connectivity issue — includes the underlying `cause` |
| `AgentValidationError` | Client-side validation failure — includes the `field` that failed |

---

## Register Agent Identity

Creates a PDA derived from the asset's public key and attaches an `AgentIdentity` plugin with lifecycle hooks for Transfer, Update, and Execute.

```typescript
import { registerIdentityV1 } from '@metaplex-foundation/mpl-agent-registry';

await registerIdentityV1(umi, {
  asset: assetPublicKey,
  collection: collectionPublicKey,          // optional but recommended
  agentRegistrationUri: 'https://arweave.net/agent-registration.json',
}).sendAndConfirm(umi);
```

| Parameter | Description |
|-----------|-------------|
| `asset` | The MPL Core asset to register |
| `collection` | The asset's collection (optional) |
| `agentRegistrationUri` | URI pointing to off-chain agent registration metadata (ERC-8004 format) |
| `payer` | Pays for rent and fees (defaults to `umi.payer`) |
| `authority` | Collection authority (defaults to `payer`) |

> Registration is a one-time operation per asset. Calling `registerIdentityV1` on an already-registered asset will fail.

## Agent Registration Document (ERC-8004)

The `agentRegistrationUri` points to a JSON document hosted on permanent storage (e.g. Arweave). Format follows [ERC-8004](https://eips.ethereum.org/EIPS/eip-8004):

```json
{
  "type": "https://eips.ethereum.org/EIPS/eip-8004#registration-v1",
  "name": "Plexpert",
  "description": "An informational agent providing help related to Metaplex protocols and tools.",
  "image": "https://arweave.net/agent-avatar-tx-hash",
  "services": [
    {
      "name": "web",
      "endpoint": "https://example.com/agent/<ASSET_PUBKEY>"
    },
    {
      "name": "A2A",
      "endpoint": "https://example.com/agent/<ASSET_PUBKEY>/agent-card.json",
      "version": "0.3.0"
    },
    {
      "name": "MCP",
      "endpoint": "https://example.com/agent/<ASSET_PUBKEY>/mcp",
      "version": "2025-06-18"
    }
  ],
  "active": true,
  "registrations": [
    {
      "agentId": "<MINT_ADDRESS>",
      "agentRegistry": "solana:101:metaplex"
    }
  ],
  "supportedTrust": ["reputation", "crypto-economic"]
}
```

### Fields

| Field | Required | Description |
|-------|----------|-------------|
| `type` | Yes | Schema identifier — use `https://eips.ethereum.org/EIPS/eip-8004#registration-v1` |
| `name` | Yes | Human-readable agent name |
| `description` | Yes | What the agent does, how it works, and how to interact with it |
| `image` | Yes | Avatar or logo URI |
| `services` | No | Array of service endpoints (each has `name`, `endpoint`, optional `version`/`skills`/`domains`) |
| `active` | No | Whether the agent is currently active (`true`/`false`) |
| `registrations` | No | Array of on-chain registrations (`agentId` = mint address, `agentRegistry` = `solana:101:metaplex`) |
| `supportedTrust` | No | Trust models: `reputation`, `crypto-economic`, `tee-attestation` |

---

## Set Agent Token (Genesis Link)

Associates a Genesis token with an existing agent identity. The Genesis account must use `Mint` funding mode. If the identity is still V1 (40 bytes), the program auto-upgrades it to V2 (104 bytes).

```typescript
import { setAgentTokenV1 } from '@metaplex-foundation/mpl-agent-registry';

await setAgentTokenV1(umi, {
  asset: assetPublicKey,
  genesisAccount: genesisAccountPublicKey,
}).sendAndConfirm(umi);
```

| Parameter | Description |
|-----------|-------------|
| `asset` | The registered agent's MPL Core asset |
| `genesisAccount` | The Genesis account for the agent's token launch |
| `payer` | Pays for additional rent if upgrading V1 → V2 (defaults to `umi.payer`) |
| `authority` | Must be the asset's Asset Signer PDA — no default, must be provided explicitly |
| `agentIdentity` | The agent identity PDA (auto-derived from `asset` if omitted) |

> The agent token can only be set once. Calling on an identity that already has a token fails with `AgentTokenAlreadySet`.

---

## Check Registration

```typescript
import { safeFetchAgentIdentityV2, findAgentIdentityV2Pda } from '@metaplex-foundation/mpl-agent-registry';

const pda = findAgentIdentityV2Pda(umi, { asset: assetPublicKey });
const identity = await safeFetchAgentIdentityV2(umi, pda);

console.log('Registered:', identity !== null);
```

> `safeFetchAgentIdentityV2` returns `null` for unregistered assets instead of throwing. V1 fetchers (`safeFetchAgentIdentityV1`, `findAgentIdentityV1Pda`) still work for legacy accounts.

## Fetch Identity from Seeds

```typescript
import { fetchAgentIdentityV1FromSeeds } from '@metaplex-foundation/mpl-agent-registry';

const identity = await fetchAgentIdentityV1FromSeeds(umi, {
  asset: assetPublicKey,
});
```

## Verify AgentIdentity Plugin on Asset

```typescript
import { fetchAsset } from '@metaplex-foundation/mpl-core';

const assetData = await fetchAsset(umi, assetPublicKey);

const agentIdentity = assetData.agentIdentities?.[0];
console.log(agentIdentity?.uri);                          // registration URI
console.log(agentIdentity?.lifecycleChecks?.transfer);     // truthy
console.log(agentIdentity?.lifecycleChecks?.update);       // truthy
console.log(agentIdentity?.lifecycleChecks?.execute);      // truthy
```

## Read Registration Document

```typescript
const assetData = await fetchAsset(umi, assetPublicKey);
const agentIdentity = assetData.agentIdentities?.[0];

if (agentIdentity?.uri) {
  const response = await fetch(agentIdentity.uri);
  const registration = await response.json();

  console.log(registration.name);
  console.log(registration.description);
  console.log(registration.active);

  for (const service of registration.services) {
    console.log(service.name, service.endpoint);
  }
}
```

## Fetch Agent Wallet (Asset Signer)

Every Core asset has a built-in PDA wallet — no private key exists, so it can't be stolen. Only the asset itself can sign for it through Core's Execute lifecycle hook.

```typescript
import { findAssetSignerPda } from '@metaplex-foundation/mpl-core';

const assetSignerPda = findAssetSignerPda(umi, { asset: assetPublicKey });

const balance = await umi.rpc.getBalance(assetSignerPda);
console.log('Agent wallet:', assetSignerPda);
console.log('Balance:', balance.basisPoints.toString(), 'lamports');
```

---

## Register Executive Profile

Before an executive can run any agent, it needs a profile. This is a one-time setup per wallet.

```typescript
import { registerExecutiveV1 } from '@metaplex-foundation/mpl-agent-registry';

await registerExecutiveV1(umi, {
  payer: umi.payer,
}).sendAndConfirm(umi);
```

| Parameter | Description |
|-----------|-------------|
| `payer` | Pays for rent and fees (also used as the authority) |
| `authority` | The wallet that owns this executive profile (defaults to `payer`) |

> Each wallet can only have one executive profile. PDA: `["executive_profile", <authority>]`.

## Delegate Execution

The agent asset owner delegates execution to an executive profile. Creates a delegation record on-chain.

```typescript
import { delegateExecutionV1, findAgentIdentityV1Pda, findExecutiveProfileV1Pda } from '@metaplex-foundation/mpl-agent-registry';

const agentIdentity = findAgentIdentityV1Pda(umi, { asset: agentAssetPublicKey });
const executiveProfile = findExecutiveProfileV1Pda(umi, { authority: executiveAuthorityPublicKey });

await delegateExecutionV1(umi, {
  agentAsset: agentAssetPublicKey,
  agentIdentity,
  executiveProfile,
}).sendAndConfirm(umi);
```

| Parameter | Description |
|-----------|-------------|
| `agentAsset` | The registered agent's MPL Core asset |
| `agentIdentity` | The agent identity PDA for the asset |
| `executiveProfile` | The executive profile PDA to delegate to |
| `payer` | Pays for rent and fees (defaults to `umi.payer`) |
| `authority` | Must be the asset owner (defaults to `payer`) |

> Only the asset owner can delegate execution. Delegation is per-asset.

## Verify Delegation

```typescript
import { findExecutiveProfileV1Pda, findExecutionDelegateRecordV1Pda } from '@metaplex-foundation/mpl-agent-registry';

const executiveProfile = findExecutiveProfileV1Pda(umi, {
  authority: executiveAuthorityPublicKey,
});

const delegateRecord = findExecutionDelegateRecordV1Pda(umi, {
  executiveProfile,
  agentAsset: agentAssetPublicKey,
});

const account = await umi.rpc.getAccount(delegateRecord);
console.log('Delegated:', account.exists);
```

## Revoke Execution

Removes an existing execution delegation. Either the asset owner or the executive authority can revoke. Rent from the closed delegation record is refunded to the destination.

```typescript
import { revokeExecutionV1 } from '@metaplex-foundation/mpl-agent-registry';

await revokeExecutionV1(umi, {
  executionDelegateRecord: delegateRecordPda,
  agentAsset: agentAssetPublicKey,
  destination: umi.payer.publicKey,
}).sendAndConfirm(umi);
```

| Parameter | Description |
|-----------|-------------|
| `executionDelegateRecord` | The delegation record PDA to close |
| `agentAsset` | The agent asset the delegation was for |
| `destination` | Receives the refunded rent from the closed account |
| `payer` | The payer (defaults to `umi.payer`) |
| `authority` | Must be the asset owner or the executive authority (defaults to `payer`) |

---

## Full Example: Register + Delegate

```typescript
import { generateSigner } from '@metaplex-foundation/umi';
import { create, createCollection } from '@metaplex-foundation/mpl-core';
import { mplAgentIdentity, mplAgentTools } from '@metaplex-foundation/mpl-agent-registry';
import {
  registerIdentityV1,
  registerExecutiveV1,
  delegateExecutionV1,
  findAgentIdentityV1Pda,
  findExecutiveProfileV1Pda,
} from '@metaplex-foundation/mpl-agent-registry';

// 1. Create a collection
const collection = generateSigner(umi);
await createCollection(umi, {
  collection,
  name: 'Agent Collection',
  uri: 'https://example.com/collection.json',
}).sendAndConfirm(umi);

// 2. Create an asset
const asset = generateSigner(umi);
await create(umi, {
  asset,
  name: 'My Agent',
  uri: 'https://example.com/agent.json',
  collection,
}).sendAndConfirm(umi);

// 3. Register identity
await registerIdentityV1(umi, {
  asset: asset.publicKey,
  collection: collection.publicKey,
  agentRegistrationUri: 'https://example.com/agent-registration.json',
}).sendAndConfirm(umi);

// 4. Register executive profile (one-time per wallet)
await registerExecutiveV1(umi, {
  payer: umi.payer,
}).sendAndConfirm(umi);

// 5. Delegate execution
const agentIdentity = findAgentIdentityV1Pda(umi, { asset: asset.publicKey });
const executiveProfile = findExecutiveProfileV1Pda(umi, { authority: umi.payer.publicKey });

await delegateExecutionV1(umi, {
  agentAsset: asset.publicKey,
  agentIdentity,
  executiveProfile,
}).sendAndConfirm(umi);
```

---

## Full Example: Register Agent + Launch Bonding Curve Token

End-to-end workflow using the Mint Agent API and Genesis Launch API to create an agent and launch a bonding curve token linked to it.

```typescript
import { createUmi } from '@metaplex-foundation/umi-bundle-defaults';
import { keypairIdentity } from '@metaplex-foundation/umi';
import { mintAndSubmitAgent } from '@metaplex-foundation/mpl-agent-registry';
import { createAndRegisterLaunch } from '@metaplex-foundation/genesis';

const umi = createUmi('https://api.mainnet-beta.solana.com');
const keypair = umi.eddsa.createKeypairFromSecretKey(mySecretKeyBytes);
umi.use(keypairIdentity(keypair));

// 1. Mint agent (creates Core asset + registers identity in one tx)
const agentResult = await mintAndSubmitAgent(umi, {}, {
  wallet: umi.identity.publicKey,
  name: 'My Trading Agent',
  uri: 'https://example.com/agent-metadata.json',
  agentMetadata: {
    type: 'agent',
    name: 'My Trading Agent',
    description: 'An autonomous trading agent on Solana.',
    services: [],
    registrations: [],
    supportedTrust: ['reputation'],
  },
});
console.log('Agent asset:', agentResult.assetAddress);

// 2. Wait for RPC propagation (API backend needs to index the new agent)
await new Promise(r => setTimeout(r, 30_000));

// 3. Launch bonding curve token linked to the agent
const launchResult = await createAndRegisterLaunch(umi, {}, {
  wallet: umi.identity.publicKey,
  agent: {
    mint: agentResult.assetAddress,  // Core asset address from step 1
    setToken: true,                   // permanently link token to agent (IRREVERSIBLE)
  },
  launchType: 'bondingCurve',
  token: {
    name: 'Agent Token',
    symbol: 'AGT',
    image: 'https://gateway.irys.xyz/your-image-id',
  },
  launch: {},  // protocol defaults for bonding curve params
});

console.log('Token mint:', launchResult.mintAddress);
console.log('Launch page:', launchResult.launch.link);
```

> When `agent` is provided:
> - Creator fee wallet auto-derived from agent's Core asset signer PDA
> - First buy buyer defaults to agent PDA (if `firstBuyAmount` is set in `launch`)
> - `setToken: true` is **irreversible** — permanently links the token to the agent
>
> See `./sdk-genesis.md` for full bonding curve options (creator fees, first buy, manual signing, swap integration).

> **RPC propagation**: If `createAndRegisterLaunch` fails with "Agent is not owned by the connected wallet", the API's backend hasn't indexed the new agent yet. The on-chain token creation often still succeeds — `createAndRegisterLaunch` calls `createLaunch` (on-chain) then `registerLaunch` (platform listing), and only the registration step fails. Call `registerLaunch` separately to complete it, or use the manual signing flow (`createLaunch` + `signAndSendLaunchTransactions` + `registerLaunch`) for full control over retry timing. When scripting both steps back-to-back, add a ~30 second delay or use a retry loop.

---

## PDA Reference

| Account | Seeds | Size |
|---------|-------|------|
| `AgentIdentityV2` | `["agent_identity", <asset_pubkey>]` | 104 bytes |
| `AgentIdentityV1` (legacy) | `["agent_identity", <asset_pubkey>]` | 40 bytes |
| `ExecutiveProfileV1` | `["executive_profile", <authority>]` | 40 bytes |
| `ExecutionDelegateRecordV1` | `["execution_delegate_record", <executive_profile>, <agent_asset>]` | 104 bytes |

## Program IDs

| Program | Address |
|---------|---------|
| Agent Identity | `1DREGFgysWYxLnRnKQnwrxnJQeSMk2HmGaC6whw2B2p` |
| Agent Tools | `TLREGni9ZEyGC3vnPZtqUh95xQ8oPqJSvNjvB7FGK8S` |

Same addresses on Mainnet and Devnet.

## Errors

### Agent Identity

| Code | Name | Description |
|------|------|-------------|
| 0 | `InvalidSystemProgram` | System program account is incorrect |
| 1 | `InvalidInstructionData` | Instruction data is malformed |
| 2 | `InvalidAccountData` | PDA derivation does not match the asset |
| 3 | `InvalidMplCoreProgram` | MPL Core program account is incorrect |
| 4 | `InvalidCoreAsset` | Asset is not a valid MPL Core asset |
| 5 | `InvalidAgentToken` | Agent token account is invalid |
| 6 | `OnlyAssetSignerCanSetAgentToken` | Authority must be the Asset Signer PDA |
| 7 | `AgentTokenAlreadySet` | Agent token has already been set on this identity |
| 8 | `InvalidAgentIdentity` | Agent identity account is invalid or not owned by this program |
| 9 | `AgentIdentityAlreadyRegistered` | Asset already has a registered identity |
| 10 | `InvalidGenesisAccount` | Genesis account is invalid (wrong owner, discriminator, or too small) |
| 11 | `GenesisNotMintFunded` | Genesis account does not use Mint funding mode |

### Agent Tools

| Code | Name | Description |
|------|------|-------------|
| 0 | `InvalidSystemProgram` | System program account is incorrect |
| 1 | `InvalidInstructionData` | Instruction data is malformed |
| 2 | `InvalidAccountData` | Invalid account data |
| 3 | `InvalidMplCoreProgram` | MPL Core program account is incorrect |
| 4 | `InvalidCoreAsset` | Asset is not a valid MPL Core asset |
| 5 | `ExecutiveProfileMustBeUninitialized` | Executive profile already exists |
| 6 | `InvalidExecutionDelegateRecordDerivation` | Delegation record PDA derivation mismatch |
| 7 | `ExecutionDelegateRecordMustBeUninitialized` | Delegation record already exists |
| 8 | `InvalidAgentIdentity` | Agent identity account is invalid |
| 9 | `AgentIdentityNotRegistered` | Asset does not have a registered identity |
| 10 | `AssetOwnerMustBeTheOneToDelegateExecution` | Only the asset owner can delegate execution |
| 11 | `InvalidExecutiveProfileDerivation` | Executive profile PDA derivation mismatch |
| 12 | `ExecutionDelegateRecordMustBeInitialized` | Delegation record does not exist or is not initialized |
| 13 | `UnauthorizedRevoke` | Signer is not the asset owner or the executive authority |
