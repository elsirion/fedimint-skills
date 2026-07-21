---
name: fedimint-cli-wallet
description: >-
  Use when a user wants to use `fedimint-cli` as a wallet to hold and move money
  through a Fedimint federation — joining a federation from an invite code,
  checking balance, sending/receiving Chaumian ecash (out-of-band notes),
  sending/receiving Lightning payments via a gateway, and depositing/withdrawing
  on-chain Bitcoin. Triggers on: "fedimint-cli", "fedimint wallet", "join
  federation", "invite code", "ecash", "e-cash", "reissue", "spend notes",
  "ln-invoice", "ln-pay", "lightning invoice", "deposit address", "peg-in",
  "peg-out", "withdraw on-chain", "fedimint balance", "fedimint backup".
---

# fedimint-cli as a wallet

`fedimint-cli` is a command-line ecash wallet for a **single client** that talks to a Fedimint
federation. It holds Chaumian ecash notes and can move value three ways: **ecash** (out-of-band
notes handed to another person), **Lightning** (via a gateway), and **on-chain Bitcoin** (peg-in /
peg-out with the federation's wallet).

**Core mental model:** the client's balance is ecash. Everything else is a conversion — Lightning and
on-chain are just ways to get value into or out of that ecash balance, brokered by the federation
(on-chain) or a gateway (Lightning).

## Setup

### 1. Install `fedimint-cli`

Pick the method that matches the platform. **On Debian/Ubuntu use the `.deb`** — it is by far the
cleanest path.

**Debian / Ubuntu (`.deb`):**
```bash
apt-get update && apt-get install -y curl                 # if curl is missing (fresh containers)
curl -L -O https://github.com/fedimint/fedimint/releases/download/v0.11.1/fedimint-cli_0.11.1_amd64.deb
apt-get install -y ./fedimint-cli_0.11.1_amd64.deb        # installs to /usr/bin/fedimint-cli
fedimint-cli version-hash                                  # sanity check -> prints a git hash
```
> The `.deb` is ~170 MB — let the download **finish** before installing. Installing a truncated file
> fails with a misleading `E: Invalid archive member header`. `curl -L -O` (no progress bar in scripts)
> gives no hint; check the file size or use `curl -fL` so a failed download errors instead of half-writing.
(Fedora/RHEL: the same page has `fedimint-cli-0.11.1-1.x86_64.rpm` — `dnf install ./…rpm`.)

**NixOS / any system with Nix:**
```bash
# --accept-flake-config lets Nix use Fedimint's binary cache (fedimint.cachix.org) instead of
# compiling from source — without it the install can take tens of minutes.
nix profile install --accept-flake-config github:fedimint/fedimint/v0.11.1#fedimint-cli
# or run without installing: nix run --accept-flake-config github:fedimint/fedimint/v0.11.1#fedimint-cli -- info
```
Requires Nix with flakes enabled (`experimental-features = nix-command flakes`). On NixOS do **not**
use the raw release binary below — an unpatched dynamically-linked binary won't run; use Nix.

**Other Linux (no package manager):** the release page also has a bare
`fedimint-cli-v0.11.1` asset, but it is a **self-extracting archive**, not a plain ELF — it needs
`hexdump`, `tar`, and `xz` present to unpack on first run (`apt-get install -y bsdmainutils xz-utils`
on Debian/Ubuntu). Prefer the `.deb`/Nix/docker methods.

**Docker:** the CLI ships in the `fedimintd` image:
```bash
docker run --rm -v fm-cli-data:/data fedimint/fedimintd:v0.11.1 fedimint-cli --data-dir /data info
```

### 2. Pick a data directory

The client stores its keys and database in a data directory. Set it once so every command uses the
same wallet — **a different data dir is a different, empty wallet.**

```bash
export FM_CLIENT_DIR="$HOME/.fedimint-cli"       # picked up automatically (same as --data-dir)
mkdir -p "$FM_CLIENT_DIR"
```

Every command below can also take `--data-dir <dir>` explicitly. In scripts or any non-interactive
setting where each command runs in a fresh shell (so the `export` above won't persist), pass
`--data-dir` on every invocation instead — otherwise you silently get a fresh, empty wallet.

### 3. Join a federation

You need an **invite code** (`fed11…`) from the federation you want to use.

```bash
fedimint-cli join <INVITE_CODE>
fedimint-cli info
```

`info` prints the federation id, network, meta fields, and your holdings:

```json
{
  "denominations_msat": { "1": 2, "2": 3, "512": 3, "1024": 3 },
  "federation_id": "c166a596345c126c2cde3fc57e399fec48b1ad741b49dcd48d313cb6cd4458b8",
  "meta": { "federation_name": "Devimint Federation" },
  "network": "regtest",
  "total_amount_msat": 495166,
  "total_num_notes": 41
}
```

> **Units — read this, it is the #1 footgun:** amounts default to **millisatoshis (msat)**. This
> includes `spend`, `ln-invoice --amount`, **and `withdraw --amount`**. A bare number is msat:
> `withdraw --amount 20000` means 20 000 **msat = 20 sats**, which fails as "under the dust limit".
> To withdraw 20 000 **sats**, write `--amount 20000000` (msat) or use an explicit unit suffix:
> `--amount '20000 sat'` or `--amount 0.0002btc`. `total_amount_msat: 495166` = 495 166 msat ≈ 495 sats.
> (`deposit-address` takes no amount — you send BTC to the address from elsewhere.)

> **Fees:** the credited amount is often slightly less than the nominal amount. Reissuing/receiving
> ecash can incur a small federation mint fee (e.g. `reissue` of 500 000 msat may credit ~495 000),
> and receiving Lightning pays the gateway's fee. This is expected, not an error — check `info` for
> the actual balance.

## Ecash (out-of-band notes)

Ecash is transferred by handing someone a base64 note string. The sender `spend`s (removing the value
from their wallet), the receiver `reissue`s (claiming it). Reissue also prevents double-spends.

```bash
# Sender: carve out 100 000 msat of notes (selects the smallest note set for the amount)
fedimint-cli spend 100000
# -> {"notes": "BgAAAA…"}   (hand this string to the receiver)

# Receiver: claim the notes into their own wallet.
# NOTE: `spend` returns JSON `{"notes": "BgAAAA…"}` — pass just the inner base64
# string to reissue, not the whole JSON object. Extract it with, e.g.:
#   NOTES=$(fedimint-cli spend 100000 | jq -r .notes)   # then: fedimint-cli reissue "$NOTES"
#   (no jq? grep -oP '"notes":\s*"\K[^"]+' works too)
fedimint-cli reissue BgAAAA…
# -> the reissued amount, e.g. 100000

# Check validity WITHOUT claiming (signatures only; does not detect double-spends)
fedimint-cli validate BgAAAA…
# Inspect notes as JSON without touching them
fedimint-cli dev decode notes BgAAAA…
```

- `spend` fails if it can't represent the exact amount with available note denominations (notes come
  in powers of two, so round decimal amounts like 500 000 often can't be made exactly). Add
  `--allow-overpay` to send slightly more instead of failing.
- Spent-but-unclaimed notes are auto-reclaimed by the sender after `--timeout` seconds (default 1 week),
  so a receiver who never redeems doesn't cost the sender the money permanently.
- Add `--include-invite` to `spend` so the receiver can join the federation from the notes alone.

The forward-looking form is `fedimint-cli module mint {spend,reissue,split,combine}`; the top-level
`spend`/`reissue` still work and print a deprecation note — either is fine.

## Lightning (via a gateway)

Lightning payments are brokered by a **gateway**. List the gateways the federation knows and,
optionally, pick one:

```bash
fedimint-cli list-gateways
fedimint-cli switch-gateway <GATEWAY_ID>          # optional; otherwise one is chosen for you
```

### Receive over Lightning

```bash
fedimint-cli ln-invoice --amount 100000           # 100 000 msat
# -> {"invoice": "lnbcrt…", "operation_id": "5b37…"}
# Give the "invoice" to the payer, then wait for it to be paid:
fedimint-cli await-invoice 5b37…                  # pass the operation_id
# On success this prints your updated balance JSON (same shape as `info`) and returns.
fedimint-cli info                                 # balance increased
```

### Send over Lightning

```bash
fedimint-cli ln-pay "lnbcrt1u1p…"                 # a BOLT11 invoice (or lnurl)
```

Notes:
- Sending needs enough ecash balance; receiving needs the gateway to have inbound Lightning liquidity.
- If a payment fails, try `switch-gateway` to a different gateway, or check `list-gateways` for one
  that is active.
- The newer LNv2 protocol is under `fedimint-cli module lnv2 {receive,send,await-receive,await-send}`.

## On-chain Bitcoin (peg-in / peg-out)

### Deposit (peg-in): on-chain BTC -> ecash

```bash
fedimint-cli deposit-address
# -> {"address": "bcrt1q…", "idx": 0, "operation_id": "…"}
# Send BTC to that address from any wallet, then wait for confirmations:
fedimint-cli await-deposit <OPERATION_ID>
fedimint-cli info
```

Deposits need the federation's `finality_delay` in confirmations (often ~10 blocks) before the ecash
is credited. `await-deposit` blocks until then, and on success prints a bare `null` — that is **not**
an error; confirm the credit with `info`.

### Withdraw (peg-out): ecash -> on-chain BTC

```bash
fedimint-cli withdraw --amount 50000000 --address bcrt1q…    # 50 000 000 msat = 50 000 sats
fedimint-cli withdraw --amount '50000 sat' --address bcrt1q…  # same, explicit unit
fedimint-cli withdraw --amount all --address bcrt1q…         # sweep everything
```

The response contains a `txid` and `fees_sat`. On regtest the peg-out is broadcast once the federation
signs it; mine a block to confirm.

## Backup & restore

The client is deterministic from a BIP-39 mnemonic; ecash notes can be recovered by restoring from
the federation's encrypted backup.

```bash
fedimint-cli print-secret                          # reveals the client's secret — handle carefully
fedimint-cli backup                                # upload encrypted note snapshot to the federation
fedimint-cli restore --mnemonic "word1 word2 …" --invite-code fed11…   # into a fresh data dir
```

Restore is a scan and can take a while; run `fedimint-cli info` afterwards to confirm balances.

## Quick reference

| Goal | Command |
|------|---------|
| Join federation | `fedimint-cli join <invite>` |
| Balance / info / meta | `fedimint-cli info` |
| Read federation meta fields | `fedimint-cli dev meta-fields` |
| Send ecash | `fedimint-cli spend <msat>` → give `notes` |
| Receive ecash | `fedimint-cli reissue <notes>` |
| Receive Lightning | `fedimint-cli ln-invoice --amount <msat>` → `await-invoice <op_id>` |
| Send Lightning | `fedimint-cli ln-pay <bolt11>` |
| List/switch gateway | `fedimint-cli list-gateways` / `switch-gateway <id>` |
| Deposit on-chain | `fedimint-cli deposit-address` → `await-deposit <op_id>` |
| Withdraw on-chain | `fedimint-cli withdraw --amount <msat, or '<n> sat'> --address <addr>` |
| Backup / restore | `fedimint-cli backup` / `restore --mnemonic … --invite-code …` |

## Common mistakes & troubleshooting

- **Empty balance after `join`:** joining does not fund you. You need someone to `spend` you ecash,
  a Lightning payment, or an on-chain deposit.
- **Wrong wallet:** if `info` shows an unexpected balance, you are almost certainly pointing at a
  different `--data-dir`/`FM_CLIENT_DIR`. Keep it consistent.
- **msat vs sats:** every `--amount` (ecash, LN, **and on-chain `withdraw`**) defaults to **msat**. A
  small bare number on `withdraw` fails as "under the dust limit" — use `<n>000` msat or `'<n> sat'`.
- **`spend` fails with an amount error:** denominations can't represent it exactly — use
  `--allow-overpay`.
- **Lightning payment fails / no gateway:** `list-gateways`; if the active one is down, `switch-gateway`
  to another.
- **Deprecation warnings** on `spend`/`reissue`/`ln-invoice`/etc. are expected — the commands still work.
- **Deposit not credited:** on-chain deposits wait for the federation's `finality_delay` confirmations;
  keep `await-deposit` running (on regtest you must mine blocks).
- **A killed or contested `reissue` can wedge the wallet:** interrupting `reissue`, or reissuing notes
  that were already spent/double-spent, can hang and leave a stuck pending state that makes later
  commands panic (e.g. `Cannot claim input, additional funding needed`). Recover by starting over in a
  fresh `--data-dir` and re-joining (your on-federation `backup` can be `restore`d there). Only reissue
  notes you control and haven't already claimed.
- Discover any subcommand's flags with `fedimint-cli help` or `fedimint-cli <cmd> --help`.
