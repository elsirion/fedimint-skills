# Validation notes

Each skill was validated by fresh sub-agents that received **only** that one `SKILL.md` plus a
concrete task, working on clean **Ubuntu 24.04**, **Debian 12**, and **`nixos/nix`** containers
against a real v0.11.1 `devimint`/regtest substrate. Every run produced a transcript; the friction it
surfaced was fed back into the skill (the RED→GREEN→REFACTOR loop of skill-writing). All nine runs
(3 skills × 3 distros) ended **PASS**. This doc records the transcript-derived inefficiencies and the
documentation fixes that removed them — the answer to "what could docs have prevented?".

## fedimint-cli-wallet

| Friction observed in a transcript | Fix folded into the skill |
|---|---|
| Raw release asset `fedimint-cli-v0.11.1` is a self-extracting bundle needing `hexdump`; failed on a fresh box | Lead with the `.deb` (Ubuntu/Debian) and `nix` (NixOS); document the raw asset's unpack deps |
| `withdraw --amount 20000` failed as "under the dust limit" | Prominent units warning: **every `--amount` is msat by default**, incl. on-chain withdraw; use `20000000` or `'20000 sat'` |
| `.deb` is ~170 MB; installing a half-downloaded file gives `Invalid archive member header` | Note the size and "let the download finish" / use `curl -fL` |
| Counterparty hands notes as `{"notes":"…"}`; needed to extract; no `jq`/`python3` on a fresh box | Show `jq -r .notes` **and** a `grep` fallback |
| `await-deposit` prints bare `null`, `await-invoice` prints balance JSON — looked like failures | Document each `await-*` return value; confirm via `info` |
| Reissued/received amounts credited slightly less than nominal | Add a "Fees" note (mint/gateway fee on inbound) |
| `export FM_CLIENT_DIR` doesn't persist across one-shot shells → silent empty wallet | Note: pass `--data-dir` per-command in non-interactive use |
| `spend 500000` couldn't be represented exactly | Note notes are powers of two; use `--allow-overpay` |
| A killed/contested `reissue` wedged the client (panics on later commands) | Troubleshooting entry: recover via a fresh `--data-dir` + `restore` |
| NixOS: raw binary won't run | Explicit "use Nix on NixOS, `--accept-flake-config` for the cache" |

## fedimint-federation-setup

| Friction | Fix |
|---|---|
| Only the docker-compose URL was given; no `.deb`/`.rpm` URLs, and `fedimint-cli` install wasn't documented | Add explicit release URLs for **both** `fedimintd` and `fedimint-cli` |
| The two `.deb`s share bundled libs → `dpkg` overwrite error | Document the `dpkg -i --force-overwrite` install |
| `curl` absent on fresh Debian | Prepend `apt-get install -y curl ca-certificates` |
| Meta commands run through a joined client, stated only implicitly | Add explicit "join a client first" prerequisite |
| `module meta get` returns `null` after `--federation-name` was set — looked lost | Note `--federation-name` seeds client config, not the meta module map |
| Noisy `mainline::dht`/`fm::db` INFO on stderr interleaves JSON | Tip: `RUST_LOG=warn` / read stdout only |
| Example paired `:8332` with `regtest` | Aside: match the RPC port to the network |

## fedimint-gateway-operation

| Friction | Fix |
|---|---|
| bcrypt hash contains `$`; sourced unquoted, the shell corrupted it → `Invalid hash` | Emphasize **single-quoting** the hash (shell) / `$`→`$$` (compose), and stripping the JSON quotes |
| Gateway silently "waiting for mnemonic"; `info` returns `Invalid request` until initialized | Dedicated §"Initialize the seed" — `cfg set-mnemonic` is required first-run |
| `config`/`balance` don't exist — groups are `cfg`/`get-balances` | Use the real names throughout + a troubleshooting entry |
| **`ecash pegin-recheck` requires `--address`** (my example omitted it) — hard blocker on the funding path | Fixed the example; note `--address` is required and detection lags |
| Metrics server binds the iroh port | Document the metrics listener + `FM_GATEWAY_METRICS_LISTEN_ADDR` |
| `--rpcpassword` takes plaintext, easy to confuse with the hash | State they're the same secret in two forms |
| `FM_GATEWAY_DATA_DIR` not auto-created | `mkdir -p` in the launch example |
| `info` `registrations` had only `iroh` (no `http`) | Clarify: `http` needs a public domain + TLS; self-hosted → iroh URL for LNv2 registration |
| Peg-in claim is async; `pegin-recheck` returns `{}` silently, balance lags a few seconds | Note recheck is async — poll `get-balances` after it |
| Nix install was a trailing comment using `#{a,b}` brace expansion (breaks in `sh`/`dash`) | Promote to a fenced block with each target on its own line |
| Metrics defaults to `LISTEN_ADDR` port + 1 (= default iroh port) | State the default rule; recommend setting `FM_GATEWAY_METRICS_LISTEN_ADDR` |

## macOS run (fedimint-cli-wallet, Apple M2 / macOS 26.2)

A wallet pass was later run on a native Apple-Silicon Mac (the audience most likely to run
`fedimint-cli`). Confirmed:

- **Install works with no Nix/Homebrew/admin:** the `fedimint-pkgs-*-aarch64-apple-darwin.tar.gz`
  tarball downloads, extracts, and runs; a `curl` download carries no Gatekeeper quarantine, so
  `version-hash` runs as-is (correct hash). → added a **macOS install section** to the skill.
- **Real federation ops run natively:** `join` (both over iroh across the internet and over a TCP
  reverse tunnel), `info`, and `deposit-address` all worked (network stack, RocksDB migrations, crypto,
  JSON output exercised). The binary is the same Rust code validated on Linux, so the remaining ops
  follow.
- **BSD/macOS deltas fixed in the skill:** `grep -P` and `timeout` don't exist on stock macOS — the
  skill's `grep -oP` notes-extraction fallback was replaced with a GNU/BSD-portable `sed`, and a
  macOS-userland caveat was added.
- **Not a skill issue, but noted:** a federation's **iroh** address isn't always reachable from a
  remote client (NAT), so a Mac wallet joining a remote federation may see iroh connection retries; a
  TCP endpoint (or a well-connected iroh federation) is deterministic.

## Harness efficiency notes (for future validation runs)

- The `nixos/nix` image is very minimal — no `bash`/coreutils/`jq` on the default PATH, and the nix
  profile symlinks can be empty until the first `nix profile install`. Sub-agents burned time
  rediscovering `/nix/store/*/bin`. A future harness should either use a fuller Nix image or pre-seed
  a shell; this is an *environment* cost, not a skill defect (a real NixOS box has these).
- Version-pinning everything to v0.11.1 (substrate build, release binaries, docker images) avoided all
  wire-compat issues — keep doing this.
- Pre-seeding a local Nix binary cache (`nix copy --to file://…`) made the NixOS installs fetch
  prebuilt instead of compiling for ~30 min — essential for a tractable NixOS matrix.
