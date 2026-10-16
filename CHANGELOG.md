# Changelog

Run by vv23. Vibe coded by a Web 3 enthusiast.

All changes to Front Desk (code name `tmf-holder-roles`), newest first. Commit hashes refer to the private source repository's history.

## 2026-10-16

### Front Desk retired
- The server's wallet check moved to the Gated bot (#gated-verify). Front Desk was removed from the server on 2026-10-15.
- The Worker, its secrets and its stored data (linked wallets) are deleted, so the old address no longer works.
- The Discord application and its bot token are deleted, and the private RPC keys are revoked.
- The GitHub build connection and the command-registration setup are removed. The source and these documents are kept as a record.

## 2026-10-14

### Secret names out of public text
- The public documents and the messages members can see no longer name the bot's secrets; they describe them instead ("the bot token", "the private RPC"). No secret value was ever published: the full-history scans found none. Older versions in the public repository's history still contain the names, which unlock nothing on their own.

## 2026-10-13

### Private RPCs for Ethereum and ApeChain
- Private RPC URLs for Ethereum and ApeChain are set as Cloudflare secrets, so delegate.xyz lookups on Ethereum and ApeChain use a private RPC first and the free public RPCs only as a fallback. No code change. After setting them, Re-check roles showed **Admitted** with no errors.

### Security sweep 2
Three independent reviews, each finding checked against the code. 15 new tests (122 in total); 13 of the 14 that cover behaviour fail on the previous code.
- **Smart-contract wallet sign-in is checked on Robinhood Chain only.** Adding Ethereum and ApeChain for delegations had also let the browser choose either of them for the on-chain signature check. A contract wallet at the same address on another chain can have a different owner, so that could have linked someone else's Robinhood wallet.
- **Junk delegations can't crowd out a real vault.** Anyone can delegate to anyone. A vault now counts, and is claimed, only if it holds something relevant. Up to 200 vaults and 200 token delegations are read; passing a limit gives a partial result (roles only added) instead of dropping a vault.
- **Delegations from Ethereum and ApeChain need a plain wallet on both chains**, not only on Robinhood. EIP-7702 accounts count as plain.
- **A token-bound account another member claims as a vault doesn't count twice.** (TBA support is off today.)
- **Cheaper to abuse less:** parallel finishes for one member run one at a time per isolate; a wrong plain signature from a wallet without code never reaches the RPC; reloading the page reuses the message without a storage write; "Remove a wallet" with a wallet that isn't linked no longer runs a re-check.
- The 3-wallet limit is checked before the link is used up.
- The `register-commands` workflow runs in a GitHub environment (`discord-admin`).
- Documents corrected: what an unused link keeps, what counts as an attempt, the optional RPC secrets, the live-test commits, the secret scan (now all commits), the Actions SHAs (checked), and three uses of a word the house style avoids.

### Wording
- What your wallet signs is now called a "message" everywhere. The word "text" was dropped.
- The check-in channel is now **#digital-check-in** (it was #verify). The Discord app is now named **Unofficial TMF Front Desk**.

## 2026-10-12

### Delegation tested live; temporary gate removed
- **Live test (operator):** with the vault not linked and its "entire wallet" delegations only on Ethereum and ApeChain, Re-check roles showed **Admitted** to Delegation Test with 3 on record, matching the 3 tokens the operator reports in the vault.
- Removed the temporary **Delegation Test** gate. Pass Holder is again the only role the bot manages.

### Delegations made on Ethereum and ApeChain now count
- **The problem:** a live delegation test found nothing. The operator's delegate.xyz delegations were on Ethereum and ApeChain, and the bot only read Robinhood Chain's registry.
- **The change:** for Robinhood Chain holdings, the bot now also reads delegate.xyz on Ethereum and ApeChain (`delegationAlsoFrom`).
  - It accepts only "entire wallet" delegations with no rights restriction.
  - The vault must have no contract code on Robinhood Chain (a plain wallet), since a contract wallet at the same address on another chain may have a different owner.
  - All existing rules apply: one vault per account, never removing roles on read errors, and the 24-hour add-only limit.
- Optional private RPC secrets for Ethereum and ApeChain. 9 new tests (107 in total).

### Temporary Delegation Test gate
- Added a temporary gate, **Delegation Test**, for the ERC-721 `0x23313954729344C899B6c67951347cb57422cB64` on Robinhood Chain. It's there to test delegate.xyz delegation live with an NFT the operator holds in a vault, and will be removed after the test.

### Private RPC secret: clearer errors, two paste mistakes fixed
- **The problem:** a live re-check showed "Pending". the private Robinhood Chain RPC secret was set but wasn't an `https://` URL, and the public RPC fallback was rate limited ("Too Many Requests"). No roles changed.
- **The fix:** the secret check now corrects a `wss://` address and the secret's name pasted in front of the value. It still refuses `http://`. For anything else it says what shape the value has (no `https://`, `http://`, spaces) and its length, never the value. SETUP.md gives the steps to re-save the secret.
- **The public RPC now retries more patiently when it is busy:** 3 retries starting at 1.5 s, or 2 retries after a Discord reply.

### Security sweep fixes
Three independent reviews (code, delegation, wallet support) were checked against the code and every finding fixed. Each behaviour fix has a test that fails on the old code.
- `406f970` **Safety:**
  - add-only on unreadable vaults lasts 24 h at most
  - the wallet index decides who a wallet counts for (closes a race that put one wallet on two accounts)
  - vault claims are released or moved when the delegation is gone, and only relevant delegations count or claim
  - only Discord "Unknown Member" means a member left
  - failed signatures use up link attempts
  - a used link can't be restored by a race
  - per-chain `delegation` switch, and a missing registry is a partial result
  - TBAs linked to another member don't count, and unconfirmed scan failures are errors
  - the doc link filter refuses `//` links
- `2a668d3` **Abuse limits:**
  - 10 links per member per UTC day
  - cooldowns that work on `workers.dev` (isolate memory), with an optional Rate Limiting binding
  - Remove a wallet cooldown and no-op writes skipped
  - the cron cursor is written only when it changes
  - the cron stops at 30 subrequests and resumes with the members it missed
  - shorter RPC timeouts after a Discord reply
- `cad38e3` **Delegation upgrades:**
  - single-token (ERC721) and amount (ERC20) delegations count
  - linking a vault wallet directly re-checks its previous delegate
  - members are told when vaults couldn't be read
  - tests decode delegations with the real ABI
- **Follow-up:** "no code" answers are cached for 1 hour (not 24), so a missing Multicall3 doesn't cost an extra lookup on every read.
- **Documents:** SECURITY.md corrected in 12 places, and updated for all of the above. New full-history secret scan: nothing found.
- 30 new tests (98 in total).
- **Live test (operator):** the check-in showed ADMITTED, and a second Re-check within 30 s was refused. Delegation has not been tested live yet.
- **delegate.xyz confirmed on Robinhood Chain:** the operator checked Blockscout. It shows DelegateRegistry (delegate.xyz v2), source verified (partial match) and 2,077 transactions. Delegation stays on, and the report and README were updated.

## 2026-10-11

### Who runs it
- **One attribution everywhere:** "Run by vv23. Vibe coded by a Web 3 enthusiast." now appears word for word on the verify page, the Discord panel, `/about`, `/security` and every document. The README gains a short "Vibe coded" section explaining what that means.
- Every document, the verify page and the Discord panel now say the desk is **run by vv23**, its sole developer, instead of "members of the community". Repost the panel with `/verify-setup` to update it in Discord.

### Public documents
- **README rewritten for members:** how to check in safely, what the desk can and cannot do, what is stored. The operator guide moved to `SETUP.md`, which stays private.
- **Security report polished:** a "Check it yourself" table, a note that the source is private, three published locations, and a fresh secret scan of the full history.
- **New page `/about`**, generated from `README.md` like `/security` is from `SECURITY.md`. The verify page footer links to it.
- **Public documents repository:** `vv23-CT/tmf-unofficial-front-desk` holds `README.md`, `SECURITY.md`, `CHANGELOG.md` and a short `LICENSE.md` only.
- 4 new tests (68 in total): document links, the credit line, the storage ID kept out of public copies, and the "Run by vv23" wording.

### The desk clerk
- New bot character: an original pixel-art robot, a CRT terminal with amber eyes and an antenna, on oxblood, drawn by `scripts/draw-desk-clerk.py`. Used as the bot icon and the Discord panel image, replacing the Midas shareholder cutout.

### Look, wording and the public security page
- **New design, "Ledger"**, from the TMF community art kit: fund-colour rule, Caslon headline, mono figures, an ADMITTED stamp, light mode and the dark "After Hours" theme.
- **Clearly unofficial.** The page, the Discord panel and the signed message say the desk is unofficial and not affiliated with The Mutual Fun. Config validation refuses footers without that line.
- **New wording.** Panel "The front desk" with **Check in** and **My papers**; results read **Admitted** to the Breakroom, **Declined** or **Pending**. The signed message now names "Front Desk, an unofficial TMF community desk".
- **All visible text is editable** in `branding` in `config.json`, and checked against the house style (no dashes, `!`, `<`, `>` or "invite").
- **Static files** via Cloudflare static assets (`public/`): self-hosted fonts, the panel image and a bot icon, with the same security headers (`public/_headers`). The CSP adds `font-src 'self'`; still nothing loads from other sites.
- **Security report on the bot's own site** at `/security`, generated from `SECURITY.md` by `scripts/build-security-page.mjs` (now `scripts/build-pages.mjs`). Linked from the page footer, the Discord panel and the README.
- 8 new tests (64 at that point), covering the disclaimer, external resources, header drift, the report page and the style rules.
- Removed committed `.DS_Store` files.

### Security review, report and hardening
- **Re-check cooldown:** at most once per 30 seconds per member, kept in Cloudflare's edge cache so it uses no KV writes. This protects the RPC quota and KV's daily write limit from button spam.
- **Request bodies** are read as a stream and cut off at 16 KB, even when no `Content-Length` is sent.
- Added `SECURITY.md` (the public security report) and this changelog.

### Live-test fixes (Robinhood Chain)
- `bea3d04`: Read each chain's private RPC secret directly by name; say when a value is set but isn't an `https://` URL.
- `e00cc2e`: Try the private RPC first, then the public RPC, and report each endpoint's failure separately. Every error detail shows the deployed build ID.
- `90fefdb`: Support a private RPC as a Cloudflare secret. Cache "contract exists" and "Multicall3 unusable" in KV for 24 hours. Gentler retries.
- `1ac6429`: Trim spaces and line breaks around the bot token secret.
- `ce66aaa`: Fall back to direct calls when Multicall3 fails. On Robinhood Chain, bundled `balanceOf` calls failed while direct calls worked.
- `98fba06`: Show the contract's own error when a call fails.
- `64f7ccd`: Explain Discord-side failures (bad token, bot role too low, rate limit) instead of "Something went wrong".
- `2a11a76`: No JSON-RPC batching, retries with backoff, labels on every contract call. A failed delegation lookup no longer blocks verification, and in that case roles are only ever added, never removed.
- `3a734fc`: Explain why a blockchain read failed. Check each gate contract has code first.

### First security review
- `48dd38c`:
  - **One wallet counts for one Discord account**, and re-linking moves it.
  - **Each delegated vault counts for one account.**
  - Reject Discord requests older than 5 minutes.
  - 15-second cooldown between new verify links.
  - Bot replies never ping anyone.
  - Wallet and vault claims are released when a member unlinks or leaves.

### Configuration and deployment
- `95b1404`: Public address set to `https://tmf-holder-roles.vv23-ct.workers.dev`.
- `c265a3f`: Storage (KV namespace) ID set.
- `af76d4a`: Pass Holder gate, for ERC-721 `0xED37605FF0e513e46d50B26244DEb2024189751a` on Robinhood Chain (4663).
- `7ae1df2`: Discord application ID and public key set; README authorization link fixed.

### Initial release
- `b80465d`: Free token-gating bot on Cloudflare Workers:
  - Verify / My wallets panel and one-time personal links.
  - Sign-In With Ethereum, signature only.
  - ERC-721 and ERC-20 gates, ERC-6551 Token Bound Account holdings and delegate.xyz delegation.
  - Periodic re-check that removes roles after a sale.
  - Strict page security headers and no third-party scripts.
  - Tests and CI with pinned actions and `npm audit`.
