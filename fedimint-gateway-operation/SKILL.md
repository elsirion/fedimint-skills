---
name: fedimint-gateway-operation
description: >-
  Use when running or operating a Fedimint Lightning **gateway** (`gatewayd` /
  `gateway-cli`) — deploying the gateway, initializing its seed, connecting it to
  federations, funding it with ecash (peg-in), setting routing fees, checking
  balances, managing its Lightning node, monitoring payments, and backing it up.
  Triggers on: "gateway", "gatewayd", "gateway-cli", "lightning gateway",
  "connect-fed", "set fees", "routing fees", "peg-in", "gateway balance",
  "create-password-hash", "LDK gateway", "LND gateway", "LNv1", "LNv2",
  "register gateway", "gateway mnemonic".
---

# Operating a Fedimint Lightning gateway

A **Lightning gateway** bridges Fedimint federations and the Lightning Network: it swaps ecash for
Lightning sends/receives, and can serve multiple federations. It is not a guardian — it's an
untrusted economic actor that holds ecash and Lightning liquidity and earns routing fees.

Two Lightning backends: **LDK** (embedded node, turnkey — `gatewayd ldk`) or **LND** (connect an
existing LND — `gatewayd lnd`). This skill is CLI-first (`gatewayd` + `gateway-cli`); the same
operations are available in the gateway web UI (`FM_GATEWAY_LISTEN_ADDR`, default `:8176`).

## 1. Deploy `gatewayd`

**Docker Compose (recommended):**
```bash
mkdir fedimint-gateway && cd fedimint-gateway
curl -O https://raw.githubusercontent.com/fedimint/fedimint/master/docker/gatewayd/docker-compose.yaml
docker compose up -d          # after setting the password hash (below)
```
**Binaries (single box / CLI-driven):** you need **both** `gatewayd` and `gateway-cli`:
```bash
apt-get update && apt-get install -y curl ca-certificates
base=https://github.com/fedimint/fedimint/releases/download/v0.11.1
curl -L -O $base/gatewayd_0.11.1_amd64.deb
curl -L -O $base/gateway-cli_0.11.1_amd64.deb
# the two packages each bundle the same libs under /nix/store, so the second install collides;
# --force-overwrite is expected and safe (identical hashed paths):
apt-get install -y ./gatewayd_0.11.1_amd64.deb && dpkg -i --force-overwrite ./gateway-cli_0.11.1_amd64.deb
```
```bash
# NixOS / any Nix — install each target on its own line (brace expansion isn't portable to sh/dash):
nix profile install --accept-flake-config github:fedimint/fedimint/v0.11.1#gatewayd
nix profile install --accept-flake-config github:fedimint/fedimint/v0.11.1#gateway-cli
```

## 2. Password hash (mind the `$`)

The gateway authenticates admin requests with a bcrypt hash of your password:
```bash
gateway-cli create-password-hash 'YOUR_PASSWORD'    # prints a JSON-quoted "$2b$12$…" hash
```
The hash **contains `$`** and the output is wrapped in quotes — both bite you:
- **Shell env:** strip the surrounding double-quotes and **single-quote** the value, or the shell
  expands `$2b`/`$12` and corrupts it: `export FM_GATEWAY_BCRYPT_PASSWORD_HASH='$2b$12$…'`.
- **docker-compose.yaml:** escape every `$` as `$$`:
  `gateway-cli create-password-hash 'YOUR_PASSWORD' | sed 's/\$/$$/g'` → paste into
  `FM_GATEWAY_BCRYPT_PASSWORD_HASH`.

## 3. Configure & launch (LDK example)

| Variable | Purpose |
|----------|---------|
| `FM_GATEWAY_DATA_DIR` | Gateway data/seed dir (persist!). |
| `FM_GATEWAY_LISTEN_ADDR` | API/UI webserver bind (default `0.0.0.0:8176`). |
| `FM_GATEWAY_NETWORK` | `bitcoin` / `signet` / `regtest`. |
| `FM_GATEWAY_BCRYPT_PASSWORD_HASH` | Admin password hash (see above). |
| `FM_GATEWAY_IROH_LISTEN_ADDR` | Iroh P2P bind (default `0.0.0.0:8177`). |
| `FM_GATEWAY_METRICS_LISTEN_ADDR` | Metrics bind. **Defaults to `LISTEN_ADDR` port + 1** — i.e. `8177`, the same as the default iroh port. If you set iroh to `LISTEN+1` they overlap (iroh is UDP/QUIC, metrics is TCP, so it may not hard-fail, but set this explicitly to be safe). |
| `FM_BITCOIND_URL` + `FM_BITCOIND_USERNAME` + `FM_BITCOIND_PASSWORD` **or** `FM_ESPLORA_URL` | Bitcoin backend for the LDK node. |
| `FM_PORT_LDK` | LDK Lightning node port (default `10010`). |
| `FM_LDK_ALIAS` | LDK node alias. |

```bash
export FM_GATEWAY_DATA_DIR=/data/gateway FM_GATEWAY_NETWORK=regtest
export FM_GATEWAY_LISTEN_ADDR=0.0.0.0:8176 FM_GATEWAY_IROH_LISTEN_ADDR=0.0.0.0:8177 FM_PORT_LDK=10010
export FM_GATEWAY_BCRYPT_PASSWORD_HASH='<single-quoted hash>' FM_LDK_ALIAS=my-gateway
export FM_BITCOIND_URL=http://127.0.0.1:8332 FM_BITCOIND_USERNAME=bitcoin FM_BITCOIND_PASSWORD=bitcoin
mkdir -p "$FM_GATEWAY_DATA_DIR"   # gatewayd does NOT create it for you
gatewayd ldk            # LND backend: `gatewayd lnd` + FM_LND_RPC_ADDR/FM_LND_TLS_CERT/FM_LND_MACAROON
```
All `gateway-cli` commands take the webserver address and password. **`--rpcpassword` is the
*plaintext* password** — the same secret you hashed for `FM_GATEWAY_BCRYPT_PASSWORD_HASH`, not the
hash itself:
```bash
GW="gateway-cli --address http://127.0.0.1:8176 --rpcpassword YOUR_PASSWORD"
```

## 4. Initialize the seed (first run — required)

On first launch the gateway **waits for a mnemonic** and won't run (its Lightning loops stay idle;
`info` returns an `internal`/`Invalid request` error) until you set one:
```bash
$GW cfg set-mnemonic                       # create a NEW wallet
# restore instead:  $GW cfg set-mnemonic --words "word1 word2 … word12"
```
After this the log shows `Gateway successfully synced with the chain` → `Gateway is running`, and
`$GW info` returns node info (`gateway_state: Running`, `synced_to_chain: true`, LDK port/alias, and
`registrations` URLs — an `iroh` URL always, plus an `http` URL only if the HTTP endpoint is enabled).
Back up the 12 words: `$GW seed`.

## 5. Connect to a federation

```bash
$GW connect-fed "<federation-invite-code>"    # -> the federation's config + your ecash wallet (balance 0)
$GW get-balances                              # on-chain, lightning, and per-federation ecash balances
$GW invite-codes                              # list invite codes of connected federations (back these up!)
$GW leave-fed --federation-id <id>            # disconnect
```
On **LNv2**, guardians must **register** your gateway before clients use it: give a guardian the URL
from `info`'s `registrations` — for self-hosted gateways this is the **`iroh`** URL (the `http`
endpoint is only present if you run behind a public domain + TLS certificate, so most deployments have
iroh only). They register it in their guardian UI. **LND** gateways auto-register for the older
**LNv1** protocol without guardian approval.

## 6. Fund the gateway with ecash (peg-in)

The gateway needs ecash in each federation to serve **incoming** Lightning payments (it gives users
ecash and keeps the Lightning payment):
```bash
$GW ecash pegin --federation-id <id>          # -> {"address":"bcrt1…"} — send BTC to it
# send BTC to that address from any wallet and wait for confirmations (regtest: mine ~11 blocks)
# Detection often lags after confirmations — trigger it explicitly. NOTE: --address is REQUIRED and
# must be the SAME address `pegin` returned:
$GW ecash pegin-recheck --federation-id <id> --address <pegin-address>   # returns {} silently
# Claiming is async: poll get-balances for a few seconds after the recheck.
$GW get-balances                              # ecash_balance_msats rises
$GW ecash pegout --federation-id <id> --amount <msat> --address <btc-addr>   # withdraw ecash -> on-chain
```
The LDK node also has its own on-chain wallet: `$GW onchain address` / `$GW onchain send`.

## 7. Routing fees

Fees are per-federation. Amounts are in **millisatoshis**; PPM is parts-per-million.
```bash
$GW cfg set-fees --federation-id <id> --ln-base 1000 --ln-ppm 500 --tx-base 100 --tx-ppm 200
$GW cfg display                               # verify configured fees per federation
```
- `ln-base`/`ln-ppm`: fixed + variable fee on outgoing **Lightning** payments.
- `tx-base`/`tx-ppm`: fixed + variable fee on inter-federation **swaps**.

## 8. Monitor & back up

```bash
$GW get-balances          # 3 liquidity layers: onchain (channel funding), lightning (routing), ecash (per federation)
$GW payment-summary       # last-24h success/fail counts, fees, latency
$GW payment-log --federation-id <id>   # per-payment events for debugging
$GW ecash backup          # snapshot ecash; also back up `seed` (12 words) + `invite-codes` output
$GW lightning --help      # channel open/close, on-chain balance, invoices (LN node management)
$GW stop                  # safely stop the gateway
```
Recover a lost gateway: redeploy, `cfg set-mnemonic --words "<12 words>"`, then `connect-fed` each
federation (from your backed-up invite codes).

## Common mistakes & troubleshooting

- **`Invalid hash` on startup:** the bcrypt hash was corrupted by the shell — **single-quote** it
  (it contains `$`), and strip the JSON quotes from `create-password-hash` output. In compose, `$`→`$$`.
- **`info` returns `Invalid request` / gateway "waiting for mnemonic":** you haven't initialized the
  seed — run `gateway-cli cfg set-mnemonic`.
- **Command groups:** it's `cfg` (not `config`) and `get-balances` (not `balance`); fee/seed commands
  live under `cfg` (`cfg set-fees`, `cfg set-mnemonic`, `cfg display`).
- **Clients can't see the gateway (LNv2):** it must be **registered** by the federation's guardians;
  send them your `info` `registrations` URL.
- **Incoming Lightning fails:** the gateway has no ecash in that federation — `ecash pegin` to fund it.
  Outgoing fails: no outbound Lightning liquidity / channels.
- **Port clashes:** `FM_GATEWAY_LISTEN_ADDR` (8176), `FM_GATEWAY_IROH_LISTEN_ADDR` (8177), and
  `FM_PORT_LDK` (10010) must all be free and distinct.
- Discover flags with `gateway-cli --help`, `gateway-cli cfg --help`, `gateway-cli ecash --help`.
