# Front Desk

Vibe coded by a Web 3 enthusiast.

**Unofficial.** Front Desk is run by vv23 (GitHub `vv23-CT`), its sole developer. It is not affiliated with, endorsed by, or speaking for The Mutual Fun (TMF).

Front Desk is a Discord bot with a small website. It gives holders of a **TMF Pass** the **Pass Holder** role, which admits them to the Breakroom. You prove you hold a Pass by signing one text message with your wallet. Signing costs nothing and moves nothing.

- **Website:** https://tmf-holder-roles.vv23-ct.workers.dev
- **Security report:** [SECURITY.md](SECURITY.md), also at https://tmf-holder-roles.vv23-ct.workers.dev/security
- **Changes:** [CHANGELOG.md](CHANGELOG.md)
- **Public documents on GitHub:** https://github.com/vv23-CT/tmf-unofficial-front-desk
- **Source code:** private. The security report says what that means and what you can check yourself.

## How to check in safely

1. In the server's #verify channel, press **Check in** on the Front Desk panel.
2. Use only the private link in the reply that appears. It works once, only for you, for 10 minutes. **Front Desk never sends links by direct message.**
3. Before connecting a wallet, check that the address bar reads exactly `tmf-holder-roles.vv23-ct.workers.dev`.
4. Your wallet should ask for **one text signature**. The message names the same domain and your wallet address, and says: "This is only a signature: no transaction, no approval, no gas."
5. If anything asks you to **approve**, **send**, **pay** or **confirm a transaction**, close the page. Front Desk never asks for that.

The page then shows **Admitted** with your Pass count, or **Declined** if the wallet holds no Pass. **Pending** means the blockchain did not answer, and nothing was changed.

## What Front Desk can and cannot do

| | |
|---|---|
| Ask your wallet for its address and its network | Yes |
| Ask your wallet to sign one text message | Yes, once per check-in |
| Send a transaction, approve tokens, sign a permit, switch networks | **No.** The page has no code that asks for these. |
| Ask for a private key or seed phrase | **No.** Never share these with anyone. |
| Change your tokens or NFTs | **No.** It only reads balances from the blockchain. |
| Discord permissions | **Manage Roles** only, and only for the Pass Holder role |

## What is stored

- Your Discord user ID.
- The public addresses of the wallets you link (up to 3), and when each was added.
- While a link is open: a hash of its secret and the message to sign. Both are deleted when the link is used or after 10 minutes.

No private keys, seed phrases, signatures or IP addresses are stored. Removing your last wallet deletes your record. If you leave the server, it is deleted at the next automatic re-check. The security report lists every stored item.

## Using it

- **Check in** (or the `/verify` command) gives you a private one-time link.
- **My papers** shows your linked wallets, with **Re-check roles** and **Remove a wallet**.
- **Phones:** on the page, tap *Open in MetaMask*, *Open in Coinbase Wallet* or *Open in Trust Wallet*, then sign inside the wallet app. Hardware wallets work through MetaMask or Rabby.
- **After a sale:** holdings are re-checked automatically, and the role is removed at the next re-check.
- **One wallet counts for one Discord account.** Linking a wallet that is already on another account moves it to yours.

## Reporting a problem

Please report security problems privately, not in a public channel or issue. Use **Report a vulnerability** at https://github.com/vv23-CT/tmf-unofficial-front-desk/security, or message the server owner in Discord. For a suspected fake Front Desk site or a scam link, warn the server staff straight away.
