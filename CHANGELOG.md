# Changelog

Vibe coded by a Web 3 enthusiast.

All changes to Front Desk (code name `tmf-holder-roles`), newest first. Commit hashes refer to the private source repository's history.

## 2026-10-11

### Who runs it
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
- `bea3d04`: Read `RPC_URL_<chainId>` directly by name; say when a value is set but isn't an `https://` URL.
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
- `7ae1df2`: Discord application ID and public key set; README invite link fixed.

### Initial release
- `b80465d`: Free token-gating bot on Cloudflare Workers:
  - Verify / My wallets panel and one-time personal links.
  - Sign-In With Ethereum, signature only.
  - ERC-721 and ERC-20 gates, ERC-6551 Token Bound Account holdings and delegate.xyz delegation.
  - Periodic re-check that removes roles after a sale.
  - Strict page security headers and no third-party scripts.
  - Tests and CI with pinned actions and `npm audit`.
