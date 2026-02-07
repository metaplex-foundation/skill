# Publishing Guide

## Registries

### 1. Anthropic Official Plugin Directory

Built into every Claude Code install. Submit via form (do NOT open PRs to the repo):

    https://clau.de/plugin-directory-submission

Anthropic reviews for quality/security. Approved plugins may receive "Anthropic Verified" badge.

### 2. Anthropic Skills Repository

PR to add the skill to Anthropic's official skills repo:

```bash
gh repo fork anthropics/skills --clone
cd skills
cp -r /path/to/repo/skills/metaplex skills/metaplex
git checkout -b add-metaplex-skill
git add skills/metaplex
git commit -m "Add Metaplex development skill"
gh pr create --title "Add Metaplex skill" --body "Metaplex development skill for Solana NFTs, tokens, compressed NFTs, candy machines, and token launches."
```

### 3. skills.sh (Vercel)

Auto-indexed when users install. Push to GitHub, then users run:

```bash
npx skills add metaplex-foundation/metaplex-skill
```

Appears on https://skills.sh leaderboard automatically via install telemetry.

### 4. ClawHub (OpenClaw)

```bash
clawhub login
clawhub publish ./skills/metaplex \
  --slug metaplex \
  --name "Metaplex" \
  --version 0.1.0 \
  --changelog "Initial release — Core, Token Metadata, Bubblegum, Candy Machine, Genesis" \
  --tags "solana,nft,metaplex,web3,blockchain"
```

### 5. Self-Hosted Claude Code Plugin Marketplace

For team/org distribution, add a `marketplace.json` to `.claude-plugin/`:

```json
{
  "name": "metaplex-tools",
  "owner": {
    "name": "Metaplex Foundation"
  },
  "plugins": [
    {
      "name": "metaplex",
      "source": ".",
      "description": "Metaplex development on Solana.",
      "version": "0.1.0",
      "author": { "name": "Metaplex Foundation" }
    }
  ]
}
```

Users add with:
```
/plugin marketplace add metaplex-foundation/metaplex-skill
/plugin install metaplex@metaplex-tools
```

---

## Auto-Indexed Directories

These crawl GitHub for SKILL.md files automatically (no action needed):

| Directory | URL | Requirement |
|-----------|-----|-------------|
| SkillsMP | https://skillsmp.com | Public repo + 2+ GitHub stars |
| SkillHub | https://www.skillhub.club | Public repo with SKILL.md |
| MCPServers.org | https://mcpservers.org/claude-skills | Auto-indexed |

## Manual Submission Directories

| Directory | URL | How to Submit |
|-----------|-----|---------------|
| Skills Directory | https://www.skillsdirectory.com | Submit form at /submit |
| Awesome Claude Code | https://github.com/hesreallyhim/awesome-claude-code | PR (~6.6k stars) |
| Awesome Claude Skills | https://github.com/travisvn/awesome-claude-skills | PR (~5.5k stars) |
| Awesome Agent Skills | https://github.com/VoltAgent/awesome-agent-skills | PR |

---

## Version Bumping

Update version in two places:
1. `skills/metaplex/SKILL.md` — frontmatter `version`
2. `.claude-plugin/plugin.json` — `version` field

## Validation

```bash
npx skills-ref validate ./skills/metaplex
```
