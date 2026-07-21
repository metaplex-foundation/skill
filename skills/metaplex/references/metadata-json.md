# Off-Chain Metadata JSON Standards

## NFT Metadata JSON (Core & Token Metadata NFTs)

```json
{
  "name": "Asset Name",
  "description": "Description of the asset",
  "image": "https://...",
  "external_url": "https://yourproject.com",
  "animation_url": "https://...",
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

**Required:** `name`, `image`, `properties`
**Optional:** `description`, `external_url`, `animation_url`, `attributes`

> **Note**: While `external_url` and `attributes` are optional, including them is strongly recommended — wallets and marketplaces rely on them for display and indexing.

**Categories:** `image`, `video`, `audio`, `vr`, `html`

**`properties.files` ordering convention:**
- **Index 0** — always the image file, matching the top-level `image` field
- **Index 1** — the `animation_url` file, if present — can be any rich media type: video, audio, HTML, 3D model, etc.
- **Index 2+** — any additional files

## Fungible Token Metadata JSON

```json
{
  "name": "TOKEN_NAME",
  "symbol": "TOKEN_SYMBOL",
  "description": "TOKEN_DESC",
  "image": "TOKEN_IMAGE_URL"
}
```

## For more info

Consult these Metaplex docs when you need deeper detail than this reference provides:

- Core JSON schema: https://metaplex.com/docs/smart-contracts/core/json-schema
- Preparing Candy Machine assets: https://metaplex.com/docs/smart-contracts/core-candy-machine/preparing-assets
- Create an NFT: https://metaplex.com/docs/nfts/create-nft
- Create a token (fungible metadata): https://metaplex.com/docs/tokens/create-a-token
