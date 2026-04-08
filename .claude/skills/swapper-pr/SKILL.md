---
name: swapper-pr
description: Create a PR for a new swapper integration on ShapeShift Web. Handles pre-PR checks (lint, type-check, build), multi-token screenshot capture via agent-browser, image hosting, and PR creation following ShapeShift PR rules. Use when a swapper is ready to PR.
allowed-tools: Bash, Read, Write, Edit, Glob, Grep, WebFetch
---

# Swapper PR Skill

Creates a complete, rule-compliant PR for a new swapper integration, including automated multi-token screenshots via agent-browser.

## Rules (from AGENTS.md + swapper-integration contract)

- Base branch: **`develop`** (never `main`)
- Run `pnpm run lint --fix` + `pnpm run type-check` before committing
- Build the swapper package: `cd packages/swapper && pnpm run build`
- Use `.github/PULL_REQUEST_TEMPLATE.md` for PR body
- **Never commit** `.env.development` local flag changes
- Screenshots must be **in English** and show **only the feature**, not unrelated UI
- Test **multiple token pairs** — not just one — to prove the swapper handles diverse routes
- **Verify affiliate fees are supported by the protocol before implementing** (pre-condition check)

---

## Pre-condition: Affiliate Fee Verification

**Before writing any code**, confirm the protocol supports affiliate/referral fees. This is a ShapeShift revenue requirement — integrations without it may be deprioritized.

Check the protocol's docs or API for:

```bash
# 1. Search the API docs for affiliate / referral / fee fields
# Common patterns:
#   - affiliateBps / affiliateFeeBps (basis points)
#   - referralCode / partnerCode
#   - integratorFee / partnerFee / protocolFee

# 2. Make a test API call and verify the fee is applied
# Example (adapt to the protocol's API):
curl -s "https://api.<protocol>.com/quote?..." | python3 -c "
import sys, json
q = json.loads(sys.stdin.read())
# Check response for affiliate fee fields
print(json.dumps(q, indent=2))
"

# 3. Document in the PR whether affiliate fees are:
#   - Supported natively (preferred)
#   - Not supported (explain why the integration is still worthwhile)
#   - Partially supported (explain limitations)
```

If the protocol does NOT support affiliate fees, flag this clearly in the PR description.

---

## Workflow

### Step 1 — Pre-flight checks

```bash
# 1. Verify .env.development is NOT staged
git diff --name-only  # should NOT include .env.development

# 2. Lint
pnpm run lint --fix

# 3. Type-check
pnpm run type-check

# 4. Build swapper package
cd packages/swapper && pnpm run build && cd ../..

# 5. Commit lint fixes (never .env.development)
git add <changed files except .env.development>
git commit -m "chore: apply lint fixes to <SwapperName> swapper integration"
```

### Step 2 — Set language to English in the app

Before any screenshots, ensure the app is in English:

```bash
AB="agent-browser --session qabot --profile ~/.agent-browser/profiles/qabot"

$AB open "$APP_URL/trade"
$AB eval --stdin <<'JS'
Array.from(document.querySelectorAll("button")).find(b => b.getAttribute("aria-label") === "Settings" || b.textContent.trim() === "Settings")?.click()
JS
sleep 1
$AB eval --stdin <<'JS'
Array.from(document.querySelectorAll("button")).find(b => b.textContent.includes("Langue") || b.textContent.includes("Language"))?.click()
JS
sleep 1
$AB eval --stdin <<'JS'
Array.from(document.querySelectorAll("button, li")).find(b => b.textContent.trim() === "English")?.click()
JS
sleep 1
$AB press "Escape"
```

### Step 3 — Import wallet (if not already done for this origin)

Use **keystore** import (headless-friendly), not seed phrase.

```bash
$AB open "$APP_URL/trade"
$AB eval --stdin <<'JS'
Array.from(document.querySelectorAll("button")).find(b => b.textContent.includes("Connect Wallet"))?.click()
JS
sleep 2
$AB eval --stdin <<'JS'
Array.from(document.querySelectorAll("button, a")).find(b => b.textContent.includes("Import existing"))?.click()
JS
sleep 1
```

**Via keystore (preferred):**
```bash
$AB eval --stdin <<'JS'
Array.from(document.querySelectorAll("button")).find(b => b.textContent.includes("Keystore"))?.click()
JS
$AB upload "input[type=file]" "$(ls ~/Desktop/thorswap-keystore*.txt 2>/dev/null | head -1)"
$AB fill "input[placeholder*=Password]" "$NATIVE_WALLET_PASSWORD"
$AB eval --stdin <<'JS'
Array.from(document.querySelectorAll("button")).find(b => b.textContent.includes("Import Keystore"))?.click()
JS
```

**Via seed phrase (fallback — IMPORTANT: use `keyboard type`, never `fill` for React inputs):**
```bash
$AB eval --stdin <<'JS'
Array.from(document.querySelectorAll("button")).find(b => b.textContent.includes("Secret Recovery"))?.click()
JS
sleep 1
$AB eval --stdin <<'JS'
document.querySelector("textarea")?.focus()
JS
$AB keyboard type "spawn want merge pulp rapid jungle humor flame tomato absent into basket"
sleep 1
printf 'Array.from(document.querySelectorAll("button")).find(b=>b.textContent.trim()==="Next")?.click()' > /tmp/click-next.js
$AB eval "$(cat /tmp/click-next.js)"
sleep 2
$AB fill "input[placeholder*='Enter Password']" "$NATIVE_WALLET_PASSWORD"
$AB fill "input[placeholder*='Confirm Password']" "$NATIVE_WALLET_PASSWORD"
$AB eval "$(cat /tmp/click-next.js)"
sleep 3
printf 'Array.from(document.querySelectorAll("button")).find(b=>b.textContent.trim()==="Skip")?.click()' > /tmp/click-skip.js
$AB eval "$(cat /tmp/click-skip.js)"
```

### Step 4 — Test multiple token pairs

**Test at least 3 different pairs** to demonstrate the swapper handles diverse routes. Choose pairs that cover:
- Two different source chains (if cross-chain)
- Native token vs stablecoin
- Different destination chains

Example pairs for a cross-chain bridge swapper:
| # | Sell | Buy | Notes |
|---|------|-----|-------|
| 1 | USDC (Ethereum) | USDC (Arbitrum) | Stablecoin cross-chain |
| 2 | ETH (Ethereum) | ETH (Optimism) | Native token cross-chain |
| 3 | USDC (Arbitrum) | USDC (Base) | Non-Ethereum source |

For each pair:
```bash
$AB open "$APP_URL/trade"
$AB wait --text "Pay With"

# Select sell asset
$AB eval --stdin <<'JS'
document.querySelector("[data-testid='trade-sell-asset-picker']")?.click()
JS
sleep 1
$AB fill "input[placeholder*='Search']" "USDC"
sleep 1
$AB eval --stdin <<'JS'
document.querySelector("[data-testid='asset-row-USDC-eip155:1/erc20:0xa0b86991c6218b36c1d19d4a2e9eb0ce3606eb48']")?.click()
JS
sleep 1

# Select buy asset with chain filter
$AB eval --stdin <<'JS'
document.querySelector("[data-testid='trade-buy-asset-picker']")?.click()
JS
sleep 1
$AB eval --stdin <<'JS'
document.querySelector("[data-testid='chain-filter-button']")?.click()
JS
sleep 0.5
$AB eval --stdin <<'JS'
Array.from(document.querySelectorAll("[data-testid*='chain-menu-item']")).find(b => b.textContent.includes("Arbitrum"))?.click()
JS
sleep 0.5
$AB fill "input[placeholder*='Search']" "USDC"
sleep 1
$AB eval --stdin <<'JS'
document.querySelector("[data-testid='asset-row-USDC-eip155:42161/erc20:0xaf88d065e77c8cc2239327c5edb3a432268e5831']")?.click()
JS

# Enter amount
$AB fill "input[type='number'], input[inputmode='decimal']" "1"
$AB wait --text "Protocol" --timeout 15000
sleep 2

# Verify the swapper name appears as Protocol
$AB snapshot -i | grep -i "protocol\|<swappername>"
$AB screenshot /tmp/swapper-quote-pair-1.png
```

### Step 5 — Execute one full swap for the result screenshot

Pick the pair with the smallest amount to minimize cost. Execute it fully:

```bash
printf 'Array.from(document.querySelectorAll("button")).find(b=>b.textContent.trim()==="Preview Trade")?.click()' > /tmp/click-preview.js
printf 'Array.from(document.querySelectorAll("button")).find(b=>b.textContent.trim()==="Confirm and Trade")?.click()' > /tmp/click-confirm.js
printf 'Array.from(document.querySelectorAll("button")).find(b=>b.textContent.trim()==="Approve")?.click()' > /tmp/click-approve.js
printf 'Array.from(document.querySelectorAll("button")).find(b=>b.textContent.includes("Sign")&&b.textContent.includes("Swap"))?.click()' > /tmp/click-sign.js

$AB eval "$(cat /tmp/click-preview.js)"
$AB wait --text "Confirm and Trade"
$AB screenshot /tmp/swapper-confirm.png

$AB eval "$(cat /tmp/click-confirm.js)"
sleep 2

# Handle ERC20 approval if needed
$AB eval "$(cat /tmp/click-approve.js)"
$AB wait --text "Sign & Swap" --timeout 120000

$AB eval "$(cat /tmp/click-sign.js)"
$AB wait --text "is complete" --timeout 180000
$AB screenshot /tmp/swapper-complete.png

# Dismiss feedback dialog if shown
printf 'Array.from(document.querySelectorAll("button")).find(b=>b.textContent.includes("Maybe Later"))?.click()' > /tmp/click-later.js
$AB eval "$(cat /tmp/click-later.js)"
```

### Step 6 — Get on-chain tx hash

```bash
WALLET_ADDRESS="0x..."  # test wallet address

curl -s "https://dev-api.ethereum.shapeshift.com/api/v1/account/$WALLET_ADDRESS/txs?pageSize=3" \
  | python3 -c "
import sys, json
txs = json.loads(sys.stdin.read()).get('txs', [])
for tx in txs[:3]:
    print(tx.get('txid'), tx.get('to','')[:42])
"
# Most recent tx to the swapper contract = the swap tx
TX_HASH="0x..."
```

### Step 7 — Upload screenshots

```bash
upload_img() {
  curl -s -F "reqtype=fileupload" -F "time=72h" -F "fileToUpload=@$1" \
    https://litterbox.catbox.moe/resources/internals/api.php
}

# Upload all quote screenshots (one per pair tested)
URL_QUOTE_1=$(upload_img /tmp/swapper-quote-pair-1.png)
URL_QUOTE_2=$(upload_img /tmp/swapper-quote-pair-2.png)
URL_QUOTE_3=$(upload_img /tmp/swapper-quote-pair-3.png)
URL_CONFIRM=$(upload_img /tmp/swapper-confirm.png)
URL_COMPLETE=$(upload_img /tmp/swapper-complete.png)
```

> **Note:** litterbox.catbox.moe links expire after 72h. Acceptable for PR review.

### Step 8 — Push and create PR

```bash
git remote set-url origin git@github.com:swdiscordia/web.git
git push origin feat/<swapper-name>-swapper

~/bin/gh pr create \
  --repo shapeshift/web \
  --base develop \
  --head swdiscordia:feat/<swapper-name>-swapper \
  --title "feat: add <SwapperName> swapper" \
  --body "$(cat <<EOF
## Description

<!-- Describe the protocol, what assets/chains it supports, key implementation notes -->

**Affiliate fees:** <!-- Supported / Not supported — explain -->

## Issue (if applicable)

closes #

## Risk

Low — new swapper integration, entirely behind \`VITE_FEATURE_<SWAPPER>_SWAP\` feature flag.
No changes to existing swap flows.

> What protocols, transaction types, wallets or contract interactions might be affected by this PR?

<SwapperName> contracts only. No changes to existing swappers.

## Testing

### Engineering

1. Set \`VITE_FEATURE_<SWAPPER>_SWAP=true\` in \`.env.development\`
2. Connect a native wallet with <asset> on <chain>
3. Navigate to Trade, test the following pairs:
   - <SellAsset> (<SellChain>) → <BuyAsset> (<BuyChain>)
   - <SellAsset2> (<SellChain2>) → <BuyAsset2> (<BuyChain2>)
4. Verify <SwapperName> appears as Protocol with valid quotes
5. Confirm and Trade → verify swap completes successfully

### Operations

- [x] :checkered_flag: My feature is behind a flag and doesn't require operations testing (yet)

## Screenshots (if applicable)

**Quote — USDC Ethereum → USDC Arbitrum**
![]($URL_QUOTE_1)

**Quote — ETH Ethereum → ETH Optimism**
![]($URL_QUOTE_2)

**Quote — USDC Arbitrum → USDC Base**
![]($URL_QUOTE_3)

**Confirm Details**
![]($URL_CONFIRM)

**Swap complete**
![]($URL_COMPLETE)

**On-chain verification**
- [Ethereum tx (Etherscan)](https://etherscan.io/tx/$TX_HASH)
- [Cross-chain message (LayerZero Scan)](https://layerzeroscan.com/tx/$TX_HASH)
EOF
)"
```

---

## Common Pitfalls

| Problem | Fix |
|---|---|
| `fill` doesn't update React state | Use `keyboard type` for seed phrase / password inputs |
| `click --ref` times out | Use JS eval via `--stdin` instead |
| Screenshots in French | Set language to English first (Settings > Language) |
| `.env.development` accidentally committed | Only `git add` specific files, never `.` or `-A` |
| Only tested one pair | Test minimum 3 pairs covering different chains and asset types |
| Affiliate fees not checked | Verify protocol supports them BEFORE coding — document result in PR |
| Playwright/CDP needed | Not needed — use `agent-browser keyboard type` for React inputs |
