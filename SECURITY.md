# Security Report: Front Desk

Run by vv23. Vibe coded by a Web 3 enthusiast.

**Where to find this report:** the same text is published in three places:
- on the bot's own site, at `https://tmf-holder-roles.vv23-ct.workers.dev/security` (linked from the verify page and the Discord panel)
- in the public documents repository, at `https://github.com/vv23-CT/tmf-unofficial-front-desk` (`SECURITY.md`)
- in the private source repository, `vv23-CT/tmf-holder-roles`

The web page is generated from this file (`scripts/build-pages.mjs`), and a test fails if the two differ (`test/branding.test.ts`).

This document describes how Front Desk (code name `tmf-holder-roles`) works and what protects its users. It states only what the code does, plus tests and observations that were actually performed.

**Source availability.** The source code is private. Each claim names the file and test that implement it, so a reviewer given access by the operator can check every claim against the code. Readers without access can still check the points in "Check it yourself" below.

- **Report date:** 2026-10-13 (first version 2026-10-11)
- **Code covered:** the private source repository at the commit that contains this version of the report. It includes the first security sweep (`406f970`, `2a668d3`, `cad38e3`), the delegation changes of 2026-10-12 and the second security sweep of 2026-10-13 (see `CHANGELOG.md`).
- **Runtime:** a Cloudflare Worker at `https://tmf-holder-roles.vv23-ct.workers.dev`.
- **Who runs it:** vv23 (GitHub `vv23-CT`), the sole operator and developer as of 2026-10-11. The bot, its site and this report are **not affiliated with, endorsed by, or speaking for The Mutual Fun (TMF)**. The verify page and the Discord panel say so. The signed message names the desk as "an unofficial TMF community desk".
- **How it was made:** vibe coded. vv23 built it by directing an AI coding assistant (Claude Code) instead of writing every line by hand, and is the only person working on it. The code was then reviewed and tested as this report describes (section 8). Nothing here relies on trusting how it was written: every claim names its code and test, and the "Check it yourself" steps below need no code access.

---

## Check it yourself (no code access needed)

| What to check | How |
|---|---|
| The bot's only permission is Manage Roles | In Discord: Server Settings → Integrations → the bot, or Server Settings → Roles → the bot's role |
| You are on the real site | The address bar reads exactly `tmf-holder-roles.vv23-ct.workers.dev`, over HTTPS |
| You are asked for a signature only | Your wallet shows one message. Its first line names that domain, the second is your address, and the statement after them says "This is only a signature: no transaction, no approval, no gas." The lines after that are the address of the site, a version, the network, a one-time number and the times it was issued and expires. Any other request (approve, send, permit, switch network) does not come from this desk. |
| The page loads nothing from other sites | Browser developer tools → Network: every request goes to `tmf-holder-roles.vv23-ct.workers.dev` |
| The security headers | Browser developer tools → Network → the page → Response headers: the `Content-Security-Policy` matches section 3 |
| Links only come from the bot's private reply | The bot never sends links by direct message. A link in a DM is not from this desk. |

---

## 1. What the bot does

The bot gives Discord roles to members who hold specific blockchain assets.

1. A member clicks **Check in** on the bot's panel in the Discord server (or uses `/verify`).
2. They open a private, one-time link.
3. They connect a wallet and **sign one message**.
4. The bot reads their holdings from the blockchain (read-only) and adds or removes the configured Discord roles.

The current configuration (`config.json`) has one role, **Pass Holder**, for holders of at least 1 token of the ERC-721 contract `0xED37605FF0e513e46d50B26244DEb2024189751a` on Robinhood Chain (chain ID 4663).

---

## 2. What the bot can and cannot do

### Wallets

| Action | Possible? | Where to check |
|---|---|---|
| Ask the wallet for its address (`eth_requestAccounts`) | Yes | `src/web.ts` |
| Ask the wallet which network it's on (`eth_chainId`) | Yes | `src/web.ts` |
| Ask the wallet to sign a message (`personal_sign`) | Yes, once per verification | `src/web.ts` |
| Send a transaction, approve tokens, sign a permit or typed data, switch networks | **No.** The page contains no code that calls `eth_sendTransaction`, `eth_signTypedData*`, `wallet_*` or any other wallet method. | `src/web.ts` (these 3 calls are the page's only wallet requests) |
| Read private keys or seed phrases | **No.** Nothing in the code asks for them. | `src/web.ts` |

### Blockchain

The bot only reads, using `eth_call` and `eth_getCode` (`src/chain.ts`). It holds no keys and sends no transactions.

### Discord

| Permission | Granted? |
|---|---|
| Manage Roles | Yes. It's the only permission in the bot's authorization link (`permissions=268435456`, see `SETUP.md`). |
| Anything else (Administrator, reading messages, kick, ban, managing channels…) | No |

The bot changes **only** the role IDs listed in `config.json`. It refuses any other role, enforced in `Discord.roleChange` in `src/discord.ts` and tested in `test/flow.test.ts`. Discord itself also prevents the bot from changing roles above its own role.

---

## 3. How a verification works, step by step

1. **Discord → Worker.** Each Discord request is checked with Discord's Ed25519 signature, using the app's public key (`verifyInteraction`, `src/discord.ts`).
   - Requests whose timestamp is more than **300 seconds** from the current time are rejected (`src/index.ts`).
   - Only requests from the configured server ID are acted on (`src/interactions.ts`).
2. **Check in button → one-time link.** The bot creates a random 32-byte secret and gives the member a link containing their Discord ID and that secret, in a reply only they can see.
   - Only a SHA-256 hash of the secret is stored, and it expires after **600 seconds** (`Store.createLink`, `src/store.ts`).
   - A new link can't be made within **15 seconds** of the previous one, and one member gets at most **10 links per UTC day** (`linkAllowance`, `src/store.ts`; tested in `test/limits.test.ts`).
   - Creating a new link replaces the old one.
3. **Page loads.** It is served with these security headers (`SECURITY_HEADERS`, `src/index.ts`):
   - Content-Security-Policy: `default-src 'none'; script-src 'self'; style-src 'self'; img-src 'self' data:; font-src 'self'; connect-src 'self'; base-uri 'none'; form-action 'none'; frame-ancestors 'none'`
   - `X-Frame-Options: DENY`, `Referrer-Policy: no-referrer`, HSTS and `Cache-Control: no-store`.

   The page loads no third-party scripts, fonts or images, and all text it shows is inserted as plain text (`textContent`), never as HTML (`src/web.ts`). A test checks the page and its styles for external addresses (`test/branding.test.ts`).

   **Static files** (the fonts, the panel image and this report's web page) are in `public/` and served by Cloudflare's static assets feature. `public/_headers` gives them the same security headers, except `Cache-Control: public, max-age=3600`, and a test fails if those headers drift from the Worker's. The fonts are Libre Caslon Text, Source Serif 4 and IBM Plex Mono, self-hosted under the SIL Open Font License (licence files in `public/fonts/`).

   **Visible text.** Most of what the page, the panel and the replies show comes from `branding` in `config.json`. `parseBranding` in `src/config.ts` rejects it unless every field is one line within a length limit, contains no `<`, `>`, dashes, `!` or the word "invite" (the `thumbnail` path is exempt from the word rules), and the page and panel footers both contain "not affiliated". Some text is not from `branding`: `appName` (the page title and the signed statement), the gate labels in replies, and fixed messages in the code (errors, button replies).
4. **Start (`/api/start`).** The server checks the link: hash match, not expired, at most **5** attempts per link (each start with a new wallet or network, and each failed signature, is one). Reloading the page with the same wallet and network reuses the stored message, with no new attempt and no storage write. It then **builds the message itself** as a Sign-In With Ethereum (EIP-4361) message (`buildMessage`, `src/siwe.ts`) containing:
   - the domain `tmf-holder-roles.vv23-ct.workers.dev`
   - the wallet address
   - the statement *"Link this wallet to Discord user <id> for Front Desk, an unofficial TMF community desk. This is only a signature: no transaction, no approval, no gas."*
   - the site address, the network the wallet reports (`eth_chainId`) and a random nonce
   - an issue time and an expiry **10 minutes** later

   The message is stored server-side.
5. **The member signs** in their wallet.
6. **Finish (`/api/finish`).** The server checks the signature against **the exact stored message** (`verifySignature`, `src/siwe.ts`):
   - The message must be valid EIP-4361, for the official domain, not expired.
   - The signature must come from the address in the message. Normal wallets are checked by recovering the signer locally. Smart-contract wallets are checked on-chain (EIP-1271/ERC-6492) through viem, **on Robinhood Chain**, where the holdings are, never on the network the browser reports: the same address on another chain can be a different contract with a different owner. A plain 65-byte signature from an address with no contract code is never sent to the RPC.
   - A failed signature counts as one of the link's 5 attempts; after 5, the link can't be used again. Only one finish per member runs at a time in each Cloudflare isolate, so parallel requests can't run several signature checks at once there. Across isolates the count is best effort (section 10).
   - The 3-wallet limit is checked before the link is used up.
   - On success, the link is ended (single use): its record is replaced by a "used" marker, and the member's record keeps the used link's hash, so the link is refused even if a concurrent write restores its old record. The wallet is then linked.
7. **Holdings and roles.** The bot reads balances and sets roles (`src/holdings.ts`, `src/service.ts`).

The API only accepts requests whose `Origin` header is the official site (`src/index.ts`). Request bodies are capped at **16 KB** (`readBody`, `src/index.ts`).

---

## 4. Rules that prevent abuse

| Rule | Implementation | Tested in |
|---|---|---|
| A link works once, only for the member who requested it, for 10 minutes | `src/store.ts`, `src/index.ts` | `test/flow.test.ts` ("links are single-use", "expired links are rejected", "a forged or tampered token is rejected") |
| A signature from a different wallet, over a different message, or for a different domain is rejected | `src/siwe.ts` | `test/flow.test.ts` |
| **One wallet counts for one Discord account.** Verifying a wallet that's already linked moves it, and the previous account's roles are re-checked. The wallet index (`wallet:<address>`) decides: if two sign-ins race and leave a wallet on two records, the next re-check drops it from the account the index doesn't name. | `apiFinish`, `src/index.ts`; `recheck`, `src/service.ts` | `test/flow.test.ts` ("one wallet counts for one Discord account…", "a wallet left on two records by a race…") |
| A delegated vault (delegate.xyz) counts for only one Discord account. The claim is released when that member no longer has the delegation, and moves when another member has it and the claimant doesn't. A wallet or token-bound account linked directly to someone else can't be borrowed through delegation. | `src/service.ts` | `test/flow.test.ts` ("a delegated vault counts for only one Discord account", "a vault claim moves…", "a revoked delegation removes the role…", "a token-bound account linked directly to another member…") |
| Only relevant delegations count or claim a vault: the whole wallet, a whole gate or TBA contract, or one token / an amount of a gate contract. All must have no rights restriction and be addressed to the member's own wallet. **Anyone can delegate to anyone, so junk can't crowd out a real vault:** a whole-wallet or whole-contract vault counts (and is claimed) only if it holds a token of a contract its delegation covers, and a single-token or amount delegation only while the vault holds it. Up to 200 vaults and 200 token delegations are read per check, and up to 10 holding vaults count. If any of those limits is passed, the result is partial (roles are only added, section 10) instead of silently dropping a vault. | `check`, `holdingVaults`, `src/holdings.ts` | `test/holdings.test.ts`, `test/flow.test.ts`, `test/sweep2.test.ts` ("junk delegations can't crowd out a real vault") |
| Delegations made on **Ethereum or ApeChain** count for Robinhood Chain holdings only if they are "entire wallet" delegations with no rights restriction, and the vault is a **plain wallet on both chains**: no contract code on Robinhood Chain and none on the chain where the delegation was made (an EIP-7702 account, which is still controlled by its key, counts as plain). A plain wallet is controlled by the same key on every chain; a contract wallet (for example a Safe) at the same address on another chain may belong to someone else, so it is refused. Contract and single-token delegations from other chains are ignored. The same vault delegated on several chains counts once. | `check`, `src/holdings.ts`; `delegationAlsoFrom`, `config.json` | `test/crosschain.test.ts`, `test/sweep2.test.ts` |
| At most 3 wallets per member | `src/index.ts` | `test/flow.test.ts` |
| Roles are **never removed because a blockchain read failed**. If delegation data can't be read, roles can only be added, for **24 hours at most** from the first failure in a row. After that, roles sync normally and unreadable vaults count as zero, so a member can't keep a role after selling by making their own delegation lookup fail. | `src/service.ts`, `src/holdings.ts` | `test/flow.test.ts` ("an RPC outage never removes roles", "a failed delegation lookup…", "add-only on unreadable vaults lasts 24 hours at most…") |
| Only Discord's "Unknown Member" answer means a member left. Any other "not found" (for example the bot removed from the server) changes nothing. | `memberRoles`, `src/discord.ts` | `test/flow.test.ts` ("a 404 for anything but Unknown Member…") |
| Only roles in `config.json` are ever changed | `src/discord.ts` | `test/flow.test.ts` |
| Re-check: at most once per 30 seconds per member; Remove a wallet: once per 15 seconds, and it re-checks only when a wallet was actually removed. Best effort, see section 10. | `src/cooldown.ts`, `src/interactions.ts` | `test/interactions.test.ts`, `test/limits.test.ts`, `test/sweep2.test.ts` |
| Bot replies never ping anyone (`allowed_mentions: { parse: [] }`) | `src/interactions.ts` | `test/interactions.test.ts` |
| `/verify-setup`, which posts the panel, needs Manage Server or Administrator | `src/interactions.ts` | `test/interactions.test.ts` |

---

## 5. What is stored

Storage is Cloudflare KV. Every key and what it holds:

| Key | Content | Lifetime |
|---|---|---|
| `link:<discordUserId>` | SHA-256 hash of the link secret, creation/expiry time, attempt count, the pending message to sign, and the day's link count. After use, a "used" marker with the creation time and day count only. | The link works for 600 s; the record is kept until the end of the UTC day |
| `user:<discordUserId>` | Linked wallet addresses and when each was added; the vaults this member holds the claim for; when vault reads first failed in a row (if they are failing); the hash of the last link used | Until every wallet is removed, or the member leaves the server |
| `wallet:<address>` | The Discord user ID the wallet belongs to | Until unlinked, or the member leaves |
| `vault:<address>` | The Discord user ID a delegated vault counts for | Until released (delegation gone, member unlinks or leaves) or moved |
| `meta:cursor`, `meta:leftover` | Position of the periodic re-check; members the last run didn't reach | Rolling |
| `cache:code:<chain>:<address>`, `cache:nomulticall:<chain>` | Whether an address has contract code; whether Multicall3 works on a chain | 24 hours ("no code": 1 hour) |

No private keys, seed phrases, signatures, Discord login tokens or IP addresses are stored. Wallet addresses and Discord IDs are pseudonymous identifiers.

---

## 6. Secrets

| Secret | Where it lives | Purpose |
|---|---|---|
| `DISCORD_BOT_TOKEN` | Cloudflare encrypted secret | Adds and removes roles. Sent only to `https://discord.com/api/v10/` (tested in `test/flow.test.ts`). |
| `RPC_URL_1`, `RPC_URL_33139` (optional, not set today) | Cloudflare encrypted secret | Private RPC endpoints for Ethereum and ApeChain delegation lookups, handled exactly like `RPC_URL_4663`. Without them the public endpoints in section 7 are used. |
| `RPC_URL_4663` | Cloudflare encrypted secret | Private RPC endpoint, which contains an API key. Only an encrypted address is used: `https://`, or a `wss://` address used over `https://` on the same host. `http://` is refused. If the name was pasted into the value (`RPC_URL_4663=…`), it is removed. When the value can't be used, the error names only its shape and length, never the value (`rpcOverridesFromEnv`, `src/chain.ts`; tested in `test/chain.test.ts`). |

No secret is in the source repository or its git history. On 2026-10-13 all 40 commits of the source repository and all 14 of the public documents repository were scanned again for the same patterns, with no matches. On 2026-10-12 every commit up to and including `cad38e3` (28 commits, 257 file versions, including the 79 files inside a deleted art-kit archive) was scanned for Discord bot tokens, webhook URLs, RPC URLs with keys, GitHub and cloud tokens, private keys and API-key patterns. The only matches were the placeholder `SECRETKEY` in a test, an all-zero salt and the public Discord application key. An independent review on 2026-10-11 scanned the 25 commits that existed then and found the same.

Error messages shown to members:
- for Discord failures, are fixed texts that never include the token or the raw error (`explainFailure`, `src/discord.ts`)
- for blockchain failures, can include the RPC server's own error text, with every URL removed and cut to 160 characters (`describeError`, `src/chain.ts`). A provider that put an API key in its error text outside a URL would expose it there; none has been observed.
- can name a missing or wrong secret by its name (for example "no RPC_URL_4663 secret found"), never its value

Both behaviours are tested in `test/holdings.test.ts`, `test/chain.test.ts` and `test/flow.test.ts`.

Public values in `wrangler.toml` and `config.json`: the Discord application ID and public key, the site address, the storage ID, the server ID, the role ID and the contract address.

---

## 7. Blockchain reads

- **Endpoints:** for each chain, a private RPC secret (`RPC_URL_<chain ID>`) is tried first if set, then the chain's public RPC from `config.json` (`src/chain.ts`):
  - **Robinhood Chain (4663):** `RPC_URL_4663` (set), then `https://rpc.mainnet.chain.robinhood.com`.
  - **Ethereum (1)** and **ApeChain (33139)** are read only for delegate.xyz delegations: `RPC_URL_1` / `RPC_URL_33139` if set, then `https://ethereum.reth.rs/rpc` and `https://rpc.apechain.com/http` (the defaults viem ships for those chains).
- **Limits:** a 20-second timeout per request, a 2 MB response limit, and 2 retries (3 for the public RPC, which often answers "Too Many Requests"). Retries wait 1 to 1.5 seconds and double each time. Work that runs after a Discord reply uses shorter limits to fit Cloudflare's time limit there: 8 seconds and 1 retry for the private RPC, 6 seconds and 2 retries for the public RPC. The 10-minute re-check stops starting new members after 30 outbound requests (the free plan allows 50) and starts with the members it missed next time.
- **Multicall3:** used where it works. Observed by the operator on 2026-10-11: on Robinhood Chain, bundled calls failed while direct calls worked, so direct calls are used and that fact is cached for 24 hours. Multicall3's own deployment list has no entry for chain 4663.
- **Delegations from other chains:** Robinhood Chain also honours delegations made on Ethereum and ApeChain (`"delegationAlsoFrom": [1, 33139]`), under the rule in section 4. delegate.xyz's official README lists the registry at the same address on Ethereum and ApeChain. If one of those chains can't be read, the result is partial: roles are only added, for 24 hours at most.
- **delegate.xyz is on** for Robinhood Chain (`"delegation": true` in `config.json`). The registry is deployed there at `0x00000000000000447e69651d841bD8D104Bed493`:
  - delegate.xyz's official README lists it for Robinhood.
  - On 2026-10-12 the operator checked the address on the Robinhood Chain Blockscout explorer (screenshots). Blockscout shows a contract named **DelegateRegistry**, tagged "delegate.xyz v2", with 2,077 transactions.
  - Blockscout reports its source as **verified (partial match)** through the Blockscout Bytecode Database: `src/DelegateRegistry.sol`, compiler v0.8.21, optimization on with 9,999,999 runs, verified on Jul 29, 2026. In Blockscout's terms, a partial match means the deployed bytecode matches the published source except for the compiler's metadata.

  If the registry ever stops answering, every check becomes a partial result: roles are only added, for 24 hours at most.
- **ERC-6551 (Token Bound Accounts)** support exists in the code but isn't enabled: `tba` is empty in `config.json`.

---

## 8. Testing performed

- **Automated tests:** 122, using Node's built-in test runner, with no network access (`npm test`). All pass. They run on every push in GitHub Actions (`.github/workflows/test.yml`).
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

  This mutation testing is described from the development record; it can't be re-checked from the code itself.
- **Security sweep, 2026-10-11 and 12:** three independent reviews (the whole codebase, the delegation logic, and wallet support), each finding checked against the code. The fixes are in commits `406f970`, `2a668d3` and `cad38e3`. Every behaviour fix has a test that fails on the previous code and passes on the fixed code; this was checked by running the new tests against both. The reviewer's two race scripts (one wallet on two accounts, a used link restored) were re-run against the fixed code and no longer succeed.
- **Security sweep, 2026-10-13:** three independent reviews (request handling and sign-in, holdings and delegation, and setup, dependencies and documents). The fixes are in the commit that contains this version of the report:
  - smart-contract wallet signatures are checked on Robinhood Chain only, not on the network the browser reports
  - junk delegations can no longer push a real vault out of the limits, and a limit that is passed gives a partial result
  - delegations from Ethereum or ApeChain need a plain wallet on both chains
  - a token-bound account claimed as another member's vault doesn't count twice
  - parallel finishes, page reloads and "Remove a wallet" with a wallet that isn't linked no longer cost RPC calls, storage writes or re-checks

  Each fix has a test in `test/sweep2.test.ts`; 13 of its 14 tests fail on the previous code (the 14th covers EIP-7702 accounts, which the previous code also accepted). The reviewers' attack scripts were re-run against the fixed code and no longer succeed.
- **Dependency audit:** `npm audit` reported **0 vulnerabilities** on 2026-10-13.
- **Live test**, by the operator on 2026-10-12. Items 1 and 2 ran on the code from the first security sweep (`a7bd601`), item 3 on `fdef7c5`:
  1. Checking in with the Pass wallet showed **ADMITTED** and kept the role.
  2. Pressing Re-check twice within 30 seconds: the second press was refused.
  3. Delegation through delegate.xyz was tested live on 2026-10-12, on `fdef7c5`, with a temporary gate for the ERC-721 `0x23313954729344C899B6c67951347cb57422cB64` on Robinhood Chain. The operator's vault was not linked, and its "entire wallet" delegations existed only on Ethereum and ApeChain. Re-check roles showed **Admitted** with 3 on record, matching the 3 tokens the operator reports in the vault. The temporary gate was then removed.
- **Live test**, by the server owner on 2026-10-11:
  1. A wallet holding 2 TMF Passes was verified. Blockscout's "Read contract" independently showed `balanceOf = 2`.
  2. The bot showed "✅ Pass Holder (2)" (the reply format at the time; today it reads "**Admitted** to the Breakroom: Pass Holder (2 on record)") and granted the role.

---

## 9. Dependencies

| Package | Version | Used for |
|---|---|---|
| `viem` | 2.56.9 | The only runtime dependency: signature checks, ABI encoding, blockchain reads. Its own dependencies are `@noble/curves`, `@noble/hashes`, `@scure/bip32`, `@scure/bip39`, `abitype`, `isows`, `ox` and `ws`. |
| `wrangler` | 4.149.0 | Build/deploy tool only, not included in the deployed code |
| `typescript` | 6.0.3 | Type checking only |
| `@cloudflare/workers-types` | 5.20261006.1 | Type definitions only |

Versions are pinned exactly (`.npmrc` `save-exact=true`, plus `package-lock.json`). GitHub Actions are pinned to full commit SHAs; on 2026-10-13 those SHAs were checked with `git ls-remote` and are the tagged releases named in the comments (v6.1.0, v6.5.0).

---

## 10. Known limitations

- **Cloudflare KV is eventually consistent**, so writes made at one location can take up to about a minute to be seen at another. The checks above (single-use links, one wallet per account, one claim per vault) use KV, so in that window two requests can both pass. The effects are temporary: the next re-check settles wallets and vault claims, and a used link is refused once the member's record carries its hash. The 5-attempt limit is also counted in KV, so parallel requests reaching different isolates can each pass it; they can't run on-chain checks for a plain wallet, though (section 4). Fully atomic checks would need Durable Objects, which this free setup doesn't use.
- **Whoever holds a link can use it.** A link is bound to one Discord member, but anyone who has it (for example a wallet company's link service, see below) could link a wallet *they* control to that member while it is valid. They can't take anything, and the member can remove that wallet under My papers.
- **Mobile "Open in wallet" buttons pass the page URL**, including the one-time link, through the wallet company's own link service (MetaMask, Coinbase, Trust).
- **Holdings come from the RPC providers.** The bot trusts the configured RPC endpoints (the private provider, Robinhood's public RPC, and for delegations the Ethereum and ApeChain RPCs) to report blockchain data correctly. A wrong answer from the Ethereum or ApeChain RPC could add or hide a delegation from those chains.
- **The plain-wallet check reflects both chains today.** A wallet that later gains contract code stops counting for delegations from other chains; that is the safe direction. A "plain wallet" answer is cached for up to 1 hour, so a contract deployed at that address within the hour still counts until then.
- **Cooldowns are best effort.** Cloudflare's edge cache has no effect on `*.workers.dev` addresses (Cloudflare's Cache API documentation), so on this address the cooldowns rely on the memory of each Cloudflare isolate, which is not shared between isolates or data centres. A Cloudflare Rate Limiting binding is supported if configured (`RATE_LIMITER`); it is not configured today. The daily link budget is stored with the link record and does apply across isolates.
- **Delegation failures and the 24-hour limit.** Anyone can create delegations to any wallet. Junk can't push a real vault out (section 4), but enough of it (more than 200 delegations, or a list too large to read) makes the lookup partial or fail. If that lasts more than 24 hours, roles that came only from vaults are removed until the lookup works again. A member affected by this can link the vault wallet directly instead.
- **The free plan's daily storage limit** (1,000 KV writes) is shared by everyone. The daily link budget bounds what one member can use (up to about 70 writes a day for one account that keeps failing), but many accounts together could still use it up, which would stop new check-ins until midnight UTC.
- **Holdings are re-checked periodically:** 5 members every 10 minutes (`recheckBatchSize`, `wrangler.toml` cron). A sale removes the role at the next re-check, not instantly.
- **The site is on a free `workers.dev` address.** Members should only use links from the bot's private reply; the bot never sends links by DM.
- **Anyone with push access to the source repository's `main` branch can change the deployed bot**, because Cloudflare deploys automatically from it. The operator is the only person with push access. The one-time `register-commands` workflow runs in a GitHub environment (`discord-admin`), and its bot token is deleted after use (SETUP.md).

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
