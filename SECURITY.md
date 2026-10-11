# Security Report: Front Desk

Vibe coded by a Web 3 enthusiast.

**Where to find this report:** the same text is published in three places:
- on the bot's own site, at `https://tmf-holder-roles.vv23-ct.workers.dev/security` (linked from the verify page and the Discord panel)
- in the public documents repository, at `https://github.com/vv23-CT/tmf-unofficial-front-desk` (`SECURITY.md`)
- in the private source repository, `vv23-CT/tmf-holder-roles`

The web page is generated from this file (`scripts/build-pages.mjs`), and a test fails if the two differ (`test/branding.test.ts`).

This document describes how Front Desk (code name `tmf-holder-roles`) works and what protects its users. It states only what the code does, plus tests and observations that were actually performed.

**Source availability.** The source code is private. Each claim names the file and test that implement it, so a reviewer given access by the operator can check every claim against the code. Readers without access can still check the points in "Check it yourself" below.

- **Report date:** 2026-10-11
- **Code covered:** the private source repository at the commit that contains this version of the report. On 2026-10-11 that is the documentation commit that follows `e578ed1` (see `CHANGELOG.md`).
- **Runtime:** a Cloudflare Worker at `https://tmf-holder-roles.vv23-ct.workers.dev`.
- **Who runs it:** members of the community. The bot, its site and this report are **not affiliated with, endorsed by, or speaking for The Mutual Fun (TMF)**. The verify page, the Discord panel and the signed message all say so.
- **How it was made:** vibe coded by a Web 3 enthusiast, meaning it was written with an AI coding assistant and reviewed and tested as this report describes.

---

## Check it yourself (no code access needed)

| What to check | How |
|---|---|
| The bot's only permission is Manage Roles | In Discord: Server Settings → Integrations → the bot, or Server Settings → Roles → the bot's role |
| You are on the real site | The address bar reads exactly `tmf-holder-roles.vv23-ct.workers.dev`, over HTTPS |
| You are asked for a signature only | Your wallet shows one text message naming that domain and your address, ending "This is only a signature: no transaction, no approval, no gas." Any other request (approve, send, permit, switch network) does not come from this desk. |
| The page loads nothing from other sites | Browser developer tools → Network: every request goes to `tmf-holder-roles.vv23-ct.workers.dev` |
| The security headers | Browser developer tools → Network → the page → Response headers: the `Content-Security-Policy` matches section 3 |
| Links only come from the bot's private reply | The bot never sends links by direct message. A link in a DM is not from this desk. |

---

## 1. What the bot does

The bot gives Discord roles to members who hold specific blockchain assets.

1. A member clicks **Check in** on the bot's panel in the Discord server (or uses `/verify`).
2. They open a private, one-time link.
3. They connect a wallet and **sign one text message**.
4. The bot reads their holdings from the blockchain (read-only) and adds or removes the configured Discord roles.

The current configuration (`config.json`) has one role, **Pass Holder**, for holders of at least 1 token of the ERC-721 contract `0xED37605FF0e513e46d50B26244DEb2024189751a` on Robinhood Chain (chain ID 4663).

---

## 2. What the bot can and cannot do

### Wallets

| Action | Possible? | Where to check |
|---|---|---|
| Ask the wallet for its address (`eth_requestAccounts`) | Yes | `src/web.ts` |
| Ask the wallet which network it's on (`eth_chainId`) | Yes | `src/web.ts` |
| Ask the wallet to sign a text message (`personal_sign`) | Yes, once per verification | `src/web.ts` |
| Send a transaction, approve tokens, sign a permit or typed data, switch networks | **No.** The page contains no code that calls `eth_sendTransaction`, `eth_signTypedData*`, `wallet_*` or any other wallet method. | `src/web.ts` (these 3 calls are the page's only wallet requests) |
| Read private keys or seed phrases | **No.** Nothing in the code asks for them. | `src/web.ts` |

### Blockchain

The bot only reads, using `eth_call` and `eth_getCode` (`src/chain.ts`). It holds no keys and sends no transactions.

### Discord

| Permission | Granted? |
|---|---|
| Manage Roles | Yes. It's the only permission in the invite link (`permissions=268435456`, see `README.md`). |
| Anything else (Administrator, reading messages, kick, ban, managing channels…) | No |

The bot changes **only** the role IDs listed in `config.json`. It refuses any other role, enforced in `Discord.roleChange` in `src/discord.ts` and tested in `test/flow.test.ts`. Discord itself also prevents the bot from changing roles above its own role.

---

## 3. How a verification works, step by step

1. **Discord → Worker.** Each Discord request is checked with Discord's Ed25519 signature, using the app's public key (`verifyInteraction`, `src/discord.ts`).
   - Requests whose timestamp is more than **300 seconds** from the current time are rejected (`src/index.ts`).
   - Only requests from the configured server ID are acted on (`src/interactions.ts`).
2. **Check in button → one-time link.** The bot creates a random 32-byte secret and gives the member a link containing their Discord ID and that secret, in a reply only they can see.
   - Only a SHA-256 hash of the secret is stored, and it expires after **600 seconds** (`Store.createLink`, `src/store.ts`).
   - A new link can't be made within **15 seconds** of the previous one.
   - Creating a new link replaces the old one.
3. **Page loads.** It is served with these security headers (`SECURITY_HEADERS`, `src/index.ts`):
   - Content-Security-Policy: `default-src 'none'; script-src 'self'; style-src 'self'; img-src 'self' data:; font-src 'self'; connect-src 'self'; base-uri 'none'; form-action 'none'; frame-ancestors 'none'`
   - `X-Frame-Options: DENY`, `Referrer-Policy: no-referrer`, HSTS and `Cache-Control: no-store`.

   The page loads no third-party scripts, fonts or images, and all text it shows is inserted as plain text (`textContent`), never as HTML (`src/web.ts`). A test checks the page and its styles for external addresses (`test/branding.test.ts`).

   **Static files** (the fonts, the panel image and this report's web page) are in `public/` and served by Cloudflare's static assets feature. `public/_headers` gives them the same security headers, except `Cache-Control: public, max-age=3600`, and a test fails if those headers drift from the Worker's. The fonts are Libre Caslon Text, Source Serif 4 and IBM Plex Mono, self-hosted under the SIL Open Font License (licence files in `public/fonts/`).

   **Visible text** (the page, the panel and the replies) comes from `branding` in `config.json`. `parseBranding` in `src/config.ts` rejects it unless every field is one line within a length limit, contains no `<`, `>`, dashes, `!` or the word "invite", and the page and panel footers both contain "not affiliated".
4. **Start (`/api/start`).** The server checks the link: hash match, not expired, at most **5** attempts per link. It then **builds the message itself** as a Sign-In With Ethereum (EIP-4361) message (`buildMessage`, `src/siwe.ts`) containing:
   - the domain `tmf-holder-roles.vv23-ct.workers.dev`
   - the wallet address
   - the statement *"Link this wallet to Discord user <id> for Front Desk, an unofficial TMF community desk. This is only a signature: no transaction, no approval, no gas."*
   - a random nonce
   - an issue time and an expiry **10 minutes** later

   The message is stored server-side.
5. **The member signs** in their wallet.
6. **Finish (`/api/finish`).** The server checks the signature against **the exact stored message** (`verifySignature`, `src/siwe.ts`):
   - The message must be valid EIP-4361, for the official domain, not expired.
   - The signature must come from the address in the message. Normal wallets are checked by recovering the signer locally; smart-contract wallets are checked on-chain (EIP-1271/ERC-6492) through viem.
   - On success, the link is deleted (single use) and the wallet is linked.
7. **Holdings and roles.** The bot reads balances and sets roles (`src/holdings.ts`, `src/service.ts`).

The API only accepts requests whose `Origin` header is the official site (`src/index.ts`). Request bodies are capped at **16 KB** (`readBody`, `src/index.ts`).

---

## 4. Rules that prevent abuse

| Rule | Implementation | Tested in |
|---|---|---|
| A link works once, only for the member who requested it, for 10 minutes | `src/store.ts`, `src/index.ts` | `test/flow.test.ts` ("links are single-use", "expired links are rejected", "a forged or tampered token is rejected") |
| A signature from a different wallet, over a different message, or for a different domain is rejected | `src/siwe.ts` | `test/flow.test.ts` |
| **One wallet counts for one Discord account.** Verifying a wallet that's already linked moves it, and the previous account's roles are re-checked. | `apiFinish`, `src/index.ts` | `test/flow.test.ts` ("one wallet counts for one Discord account…") |
| A delegated vault (delegate.xyz) counts for only one Discord account. A wallet linked directly to someone else can't be borrowed through delegation. | `src/service.ts` | `test/flow.test.ts` ("a delegated vault counts for only one Discord account") |
| At most 3 wallets per member | `src/index.ts` | `test/flow.test.ts` |
| Roles are **never removed because a blockchain read failed**. If delegation data can't be read, roles can only be added. | `src/service.ts`, `src/holdings.ts` | `test/flow.test.ts` ("an RPC outage never removes roles", "a failed delegation lookup…") |
| Only roles in `config.json` are ever changed | `src/discord.ts` | `test/flow.test.ts` |
| Re-check: at most once per 30 seconds per member (edge cache, best effort per data centre) | `src/cooldown.ts`, `src/interactions.ts` | `test/interactions.test.ts` |
| Bot replies never ping anyone (`allowed_mentions: { parse: [] }`) | `src/interactions.ts` | `test/interactions.test.ts` |
| `/verify-setup`, which posts the panel, needs Manage Server or Administrator | `src/interactions.ts` | `test/interactions.test.ts` |

---

## 5. What is stored

Storage is Cloudflare KV. Every key and what it holds:

| Key | Content | Lifetime |
|---|---|---|
| `link:<discordUserId>` | SHA-256 hash of the link secret, creation/expiry time, attempt count, the pending message to sign | Expires after 600 s, or deleted when used |
| `user:<discordUserId>` | Linked wallet addresses and when each was added | Until unlinked, or the member leaves the server |
| `wallet:<address>` | The Discord user ID the wallet belongs to | Until unlinked, or the member leaves |
| `vault:<address>` | The Discord user ID a delegated vault counts for | Until reassigned |
| `meta:cursor` | Position of the periodic re-check | Rolling |
| `cache:code:<chain>:<address>`, `cache:nomulticall:<chain>` | Whether an address has contract code; whether Multicall3 works on a chain | 24 hours |

No private keys, seed phrases, signatures, Discord login tokens or IP addresses are stored. Wallet addresses and Discord IDs are pseudonymous identifiers.

---

## 6. Secrets

| Secret | Where it lives | Purpose |
|---|---|---|
| `DISCORD_BOT_TOKEN` | Cloudflare encrypted secret | Adds and removes roles. Sent only to `https://discord.com/api/v10/` (tested in `test/flow.test.ts`). |
| `RPC_URL_4663` | Cloudflare encrypted secret | Private RPC endpoint, which contains an API key |

Neither secret is in the source repository or its git history. On 2026-10-11 every commit up to and including `e578ed1` (21 commits, 194 file versions, including the 79 files inside a deleted art-kit archive) was scanned for Discord bot tokens, webhook URLs, RPC URLs with keys, GitHub and cloud tokens, private keys and API-key patterns. The only matches were the placeholder `SECRETKEY` in a test, an all-zero salt and the public Discord application key.

Error messages shown to members:
- never include error text that could contain a secret (`explainFailure`, `src/discord.ts`)
- have URLs removed and are cut to 160 characters (`describeError`, `src/chain.ts`)

Both behaviours are tested in `test/holdings.test.ts`, `test/chain.test.ts` and `test/flow.test.ts`.

Public values in `wrangler.toml` and `config.json`: the Discord application ID and public key, the site address, the storage ID, the server ID, the role ID and the contract address.

---

## 7. Blockchain reads

- **Endpoints:** the private RPC (`RPC_URL_4663`) is tried first, then Robinhood's public RPC `https://rpc.mainnet.chain.robinhood.com` (`src/chain.ts`).
- **Limits:** a 20-second timeout per request, a 2 MB response limit, 2 retries.
- **Multicall3:** used where it works. On Robinhood Chain it failed, so direct calls are used instead, and that fact is cached for 24 hours.
- **ERC-6551 (Token Bound Accounts)** and **delegate.xyz** support exist in the code but aren't enabled for the current configuration: `tba` is empty in `config.json`. Delegation is only used on chains where contract code exists at the delegate.xyz registry address. Whether that registry is deployed on Robinhood Chain hasn't been independently confirmed. It could not be checked while preparing this report, because that environment's network policy blocks the Robinhood RPC.

---

## 8. Testing performed

- **Automated tests:** 67, using Node's built-in test runner, with no network access (`npm test`). All pass. They run on every push in GitHub Actions (`.github/workflows/test.yml`).
- **Mutation testing**, done manually during development: deliberate bugs were planted in a copy of the code, one at a time, and the test suite was run each time. 19 such bugs were tried, and every one caused a test failure. They included:
  - skipping the link-secret check, the link expiry or single use
  - accepting any signer
  - skipping the domain check
  - removing roles on an RPC outage
  - touching roles that aren't configured
  - acting on other servers
  - letting anyone post the panel
  - accepting restricted delegations
  - skipping the origin check or the Discord signature check
  - letting a wallet stay on an old account
  - removing the replay window, the link cooldown or the no-ping setting
  - sharing a vault across accounts
  - not releasing wallets when a member leaves
  - the Multicall3 fallback
- **Dependency audit:** `npm audit` reported **0 vulnerabilities** on 2026-10-11.
- **Live test**, by the server owner on 2026-10-11:
  1. A wallet holding 2 TMF Passes was verified. Blockscout's "Read contract" independently showed `balanceOf = 2`.
  2. The bot showed "✅ Pass Holder (2)" and granted the role.

---

## 9. Dependencies

| Package | Version | Used for |
|---|---|---|
| `viem` | 2.56.9 | The only runtime dependency: signature checks, ABI encoding, blockchain reads. It depends on `@noble/curves`, `@noble/hashes`, `@scure/base`, `@scure/bip32`, `@scure/bip39` and `abitype`. |
| `wrangler` | 4.149.0 | Build/deploy tool only, not included in the deployed code |
| `typescript` | 6.0.3 | Type checking only |
| `@cloudflare/workers-types` | 5.20261006.1 | Type definitions only |

Versions are pinned exactly (`.npmrc` `save-exact=true`, plus `package-lock.json`). GitHub Actions are pinned to full commit SHAs.

---

## 10. Known limitations

- **Single-use links rely on Cloudflare KV**, which is eventually consistent, so a deleted key can briefly still be readable at another location. A link is bound to one Discord member and needs that member's wallet signature, so a reuse inside that window can only re-link the same member's own wallet.
- **Mobile "Open in wallet" buttons pass the page URL**, including the one-time link, through the wallet company's own link service (MetaMask, Coinbase, Trust).
- **Holdings come from the RPC providers.** The bot trusts the configured RPC endpoints (the private provider and Robinhood's public RPC) to report blockchain data correctly.
- **The Re-check cooldown is per Cloudflare data centre**, not global.
- **Holdings are re-checked periodically:** 5 members every 10 minutes (`recheckBatchSize`, `wrangler.toml` cron). A sale removes the role at the next re-check, not instantly.
- **The site is on a free `workers.dev` address.** Members should only use links from the bot's private reply; the bot never sends links by DM.
- **Anyone with push access to the source repository's `main` branch can change the deployed bot**, because Cloudflare deploys automatically from it.

---

## 11. Operator checklist

- 2FA on the GitHub, Cloudflare, Discord and RPC-provider accounts.
- Bot role placed directly above the holder roles and below all staff roles.
- `DISCORD_BOT_TOKEN` and `RPC_URL_4663` stored as **Secrets** in Cloudflare, under Production.
- Discord app: Public Bot off; only Guild Install enabled.
- **If something looks wrong:**
  1. Remove the bot from the server, or take away its Manage Roles permission.
  2. Reset the bot token in the Discord Developer Portal and update the Cloudflare secret.
  3. Rotate the RPC key at the provider.

## 12. Reporting a vulnerability

Please report security issues privately, not in a public issue or channel:
- **GitHub:** **Report a vulnerability** at `https://github.com/vv23-CT/tmf-unofficial-front-desk/security` (private vulnerability reporting).
- **Discord:** message the server owner directly.

For a fake Front Desk site or a scam link, warn the server staff straight away.
