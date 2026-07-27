---
name: fedimint-federation-setup
description: >-
  Use when setting up and running a Fedimint federation with `fedimintd` — to
  offer your community a private, scalable way to hold and transact in Bitcoin
  under shared (community) custody, rather than trusting a single custodian.
  Covers installing/launching guardian(s), running the setup/DKG ceremony,
  obtaining the invite code your members use to join, setting meta fields
  (federation name, welcome message, etc.), and day-to-day guardian operations
  (status, audit, config backup, coordinated shutdown/upgrade). Triggers on:
  "fedimintd", "set up a federation", "run a federation", "community custody",
  "guardian", "DKG", "setup ceremony", "set-local-params", "start-dkg", "invite
  code", "meta fields", "federation_name", "welcome_message", "meta module",
  "guardian dashboard", "FM_PASSWORD", "solo federation".
---

# Setting up & operating a Fedimint federation

A Fedimint federation lets a community pool Bitcoin under **shared custody**: instead of one custodian
holding everyone's funds, a group of trusted **guardians** jointly control the wallet, and members
transact privately and instantly in ecash (see the wallet side in `fedimint-bitcoin-wallet`). It's a
way to give your community a private, scalable Bitcoin bank without any single point of trust or
failure.

Each guardian runs `fedimintd`. Fedimint is Byzantine-Fault-Tolerant: a federation of `3m + 1`
guardians tolerates `m` malicious or offline ones (4 guardians → 1, 7 → 2), so no minority of
guardians can steal funds or halt the federation. A **solo** (1-guardian) federation is fine for
testing but has no fault tolerance — a real community federation uses several independent guardians.

Bringing a federation to life is a **setup ceremony**: each guardian launches `fedimintd`, sets local
params, guardians exchange setup codes, then jointly run **Distributed Key Generation (DKG)**. After
DKG the federation is live and produces an **invite code** that wallets use to join.

This skill is CLI-first (`fedimintd` + `fedimint-cli admin setup`). The same ceremony can be done
through the guardian web UI (`FM_BIND_UI`, default `:8175`); UI steps are noted where relevant.

## 1. Deploy `fedimintd`

**Docker Compose (recommended for real deployments):**
```bash
mkdir fedimintd && cd fedimintd
curl -O https://raw.githubusercontent.com/fedimint/fedimint/releases/v0.11/docker/deploy-fedimintd/docker-compose.yaml
# (or the master copy: docker/fedimintd/docker-compose.yaml)
docker compose up -d
```
The bundled compose runs mainnet with an Esplora backend and exposes the UI on `127.0.0.1:8175`.
Edit the environment (see below) before first launch.

**Binaries without Docker** (single box, or to drive setup from the CLI). You need **both**
`fedimintd` and `fedimint-cli`:
```bash
apt-get update && apt-get install -y curl ca-certificates   # fresh servers often lack curl
base=https://github.com/fedimint/fedimint/releases/download/v0.11.1
# Debian/Ubuntu:
curl -L -O $base/fedimintd_0.11.1_amd64.deb
curl -L -O $base/fedimint-cli_0.11.1_amd64.deb
# the two packages share some bundled libs, so install together with --force-overwrite:
apt-get install -y ./fedimintd_0.11.1_amd64.deb && dpkg -i --force-overwrite ./fedimint-cli_0.11.1_amd64.deb
# Fedora/RHEL: the same page has fedimintd-0.11.1-1.x86_64.rpm and fedimint-cli-0.11.1-1.x86_64.rpm
# NixOS / any Nix (install Nix first if absent):
#   nix profile install --accept-flake-config github:fedimint/fedimint/v0.11.1#fedimintd
#   nix profile install --accept-flake-config github:fedimint/fedimint/v0.11.1#fedimint-cli
```

## 2. Configure (environment variables)

| Variable | Purpose |
|----------|---------|
| `FM_DATA_DIR` | Guardian data/keys directory (persist this!). |
| `FM_BITCOIN_NETWORK` | `bitcoin` (mainnet), `signet`, or `regtest`. |
| `FM_BITCOIND_URL` + `FM_BITCOIND_USERNAME` + `FM_BITCOIND_PASSWORD` | Use your own Bitcoin Core node. |
| `FM_ESPLORA_URL` | Alternative Bitcoin backend (e.g. `https://mempool.space/api`). Set one backend or both (bitcoind primary, esplora fallback). |
| `FM_ENABLE_IROH=true` | Use Iroh P2P (no public domain/TLS needed — easiest for self-hosting). |
| `FM_BIND_P2P` / `FM_BIND_API` / `FM_BIND_UI` | Listen addresses (defaults `:8173`/`:8174`/`:8175`). |
| `FM_PASSWORD_UI` | Gates the admin **web UI** login. |
| `FM_PASSWORD_API` | Gates admin **RPCs** on the public API — **required for the CLI ceremony below.** Unset → admin RPCs return 401. |

`FM_PASSWORD_UI` and `FM_PASSWORD_API` may be the same value; set **both** if you will drive setup
from the CLI. Example env for a solo regtest test:
```bash
export FM_DATA_DIR=/data/fedimintd FM_BITCOIN_NETWORK=regtest FM_ENABLE_IROH=true
export FM_BITCOIND_URL=http://127.0.0.1:8332 FM_BITCOIND_USERNAME=bitcoin FM_BITCOIND_PASSWORD=bitcoin
# ^ match the RPC port to your node/network (mainnet 8332, testnet 18332, regtest 18443, or whatever you configured)
export FM_BIND_P2P=0.0.0.0:8173 FM_BIND_API=0.0.0.0:8174 FM_BIND_UI=0.0.0.0:8175
export FM_BIND_METRICS=127.0.0.1:8176   # defaults to :8176 if unset — set it to avoid a port clash
export FM_PASSWORD_UI=somepass FM_PASSWORD_API=somepass
fedimintd
```
On boot the logs show `Setup UI running at http://…:8175` and the API on `ws://…:8174`. The API URL
(`ws://<host>:<FM_BIND_API port>`) is the **endpoint** for the ceremony commands below.

## 3. Run the setup ceremony (CLI)

All ceremony commands take the guardian password as a **global** flag: `fedimint-cli --password
"$FM_PASSWORD_API" admin setup <ENDPOINT> <subcommand>`. Track progress any time with `… admin setup
<ENDPOINT> status` (`AwaitingLocalParams` → `SharingConnectionCodes` → `ConsensusIsRunning`).

### Solo federation (1 guardian)
```bash
EP=ws://127.0.0.1:8174
fedimint-cli --password "$FM_PASSWORD_API" admin setup $EP set-local-params "MyGuardian" \
    --federation-name "My Federation" --federation-size 1
# -> prints YOUR setup code (fedimint9…); for solo you don't share it
fedimint-cli --password "$FM_PASSWORD_API" admin setup $EP start-dkg
# -> null (success). status will move to "ConsensusIsRunning"
```

### Multi-guardian federation
Every guardian runs their own `fedimintd`. Then, on each guardian:
```bash
# 1. Each guardian sets local params with the SAME federation-name and federation-size (e.g. 4):
fedimint-cli --password "$PW" admin setup $EP set-local-params "GuardianN" \
    --federation-name "My Federation" --federation-size 4
#    -> each prints its own setup code
# 2. Guardians exchange setup codes out-of-band, then EACH guardian adds EVERY OTHER guardian's code:
fedimint-cli --password "$PW" admin setup $EP add-peer "<other-guardian-setup-code>"   # repeat per peer
# 3. Once everyone has added everyone, each guardian starts DKG:
fedimint-cli --password "$PW" admin setup $EP start-dkg
```
DKG runs jointly and can take a while; the UI/API is unavailable until it finishes.

### Get the invite code
After `ConsensusIsRunning`, the invite code is written to **`$FM_DATA_DIR/invite-code`**:
```bash
cat "$FM_DATA_DIR/invite-code"   # fed11… — hand this to wallet users / gateways
```
Verify: `fedimint-cli --data-dir /tmp/testclient join "$(cat $FM_DATA_DIR/invite-code)" && \
fedimint-cli --data-dir /tmp/testclient info`.

## 4. Meta fields

Meta fields are federation-wide, consensus-relevant key/values clients read (name, welcome message,
etc.). Two ways they get set:

- **`--federation-name` at `set-local-params`** seeds the `federation_name` in the **client config**
  (so joined clients see it via `dev meta-fields`). Note this does **not** populate the meta *module's*
  key-0 map, so a fresh `module meta get` reads `null` until you `submit` — the name isn't lost.
- **The meta module** changes/adds fields on a **live** federation without a restart.

Meta-module commands run through a **client** (a `--data-dir` that has `join`ed the federation), with
guardian auth added via `--our-id` + `--password`. So first join a client (any data dir):
`fedimint-cli --data-dir <client> join "$(cat $FM_DATA_DIR/invite-code)"`.

The meta module stores the **entire meta map as one JSON object under key `0`**. Submitting
**replaces** the whole map — so always include the fields you want to keep:

```bash
# read current meta
fedimint-cli --data-dir <client> module meta get           # -> {"revision":N,"value":{…}}
# submit a new full map as a guardian (needs guardian auth: --our-id + --password)
fedimint-cli --data-dir <client> --our-id 0 --password "$FM_PASSWORD_API" \
    module meta submit --key 0 \
    '{"federation_name":"My Federation","welcome_message":"gm and welcome"}'
# verify
fedimint-cli --data-dir <client> dev meta-fields
```

- **Solo** federation: one `submit` reaches consensus immediately.
- **Multi-guardian**: a **threshold** of guardians must submit the **identical** JSON value before it
  becomes consensus. Coordinate the exact string; check progress with
  `module meta get-submissions --key 0`.
- Well-known keys: `federation_name`, `welcome_message`, `federation_expiry_timestamp`,
  `federation_successor`, `vetted_gateways`, `meta_override_url`, `recurringd_api`, `lnaddress_api`.
  Third-party keys should be namespaced `app:field`.

## 5. Operate the federation

All admin commands take guardian auth (`--our-id <peer_id> --password "$FM_PASSWORD_API"`); store it
once with `fedimint-cli admin auth --peer-id 0 --password "$FM_PASSWORD_API"` to skip the flags.

| Task | Command |
|------|---------|
| Health / consensus status | `fedimint-cli admin status` |
| Balance sheet across modules | `fedimint-cli admin audit` |
| Back up guardian config | `fedimint-cli admin guardian-config-backup` |
| Coordinated upgrade | `fedimint-cli admin shutdown <session_idx>` (stop after a session, upgrade binary, restart) |
| Client-backup stats | `fedimint-cli admin backup-statistics` |

The **guardian dashboard** (web UI at `FM_BIND_UI`, default `:8175`, login with `FM_PASSWORD_UI`)
shows the same status and offers a meta editor and setup wizard for operators who prefer a GUI. For a
remote server, tunnel it: `ssh -NL 8175:127.0.0.1:8175 <server>`.

## Common mistakes & troubleshooting

- **`CLI needs password set` / 401 on ceremony commands:** pass `--password "$FM_PASSWORD_API"` as a
  global flag *before* `admin`, and ensure the guardian was started with `FM_PASSWORD_API` set.
- **Ceremony endpoint:** it's the **API** URL (`ws://host:<FM_BIND_API port>`, e.g. `:8174`), not the
  UI port (`:8175`).
- **Meta submit wiped a field:** `submit --key 0` replaces the whole map — include every field you
  want to keep. Read the current map with `module meta get` first.
- **Multi-guardian meta won't apply:** every submitting guardian must send the byte-identical JSON;
  a threshold is required. Check `module meta get-submissions --key 0`.
- **DKG hangs:** every guardian must have added every *other* guardian's setup code, with matching
  `--federation-name` and `--federation-size`, before anyone runs `start-dkg`.
- **Persist `FM_DATA_DIR`** (and the guardian password): losing it loses the guardian's keys. Take an
  `admin guardian-config-backup`.
- **Noisy stderr:** `fedimint-cli` logs INFO lines (`mainline::dht`, `fm::db`, …) to stderr, which
  interleaves with JSON on stdout. For clean, scriptable output set `RUST_LOG=warn` and/or read
  stdout only (`2>/dev/null`).
- Discover subcommand flags with `fedimint-cli admin --help` and `fedimint-cli admin setup --help`.
