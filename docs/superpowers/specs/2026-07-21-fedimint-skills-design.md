# Fedimint end-user skills — design & validation spec

**Date:** 2026-07-21
**Repo:** `fedimint-skills` (agent-facing skills for operating Fedimint as an end user)
**Source under study:** `github.com/fedimint/fedimint` @ **v0.11.1** (pinned; matches published docker images and release binaries)

## Goal

Ship three agent-facing skills (same `SKILL.md` format as the existing `gateway-liquidity`
skill in fedimint) that let an agent help a real end user:

1. `fedimint-bitcoin-wallet` — use `fedimint-cli` as a Lightning, on-chain, and ecash wallet.
2. `fedimint-federation-setup` — set up a federation with `fedimintd` and operate it (incl. meta fields).
3. `fedimint-gateway-operation` — run and operate a Lightning gateway (`gatewayd`/`gateway-cli`).

Each skill must be **validated** by fresh subagents that receive *only* that one skill plus a
concrete task, working in clean Ubuntu, Debian, and NixOS containers (simulating end-user devices).
One commit per validated skill. Validate skills one after another.

## Decisions (confirmed with user)

- **Interface:** CLI-first (fedimint-cli / gateway-cli / fedimintd env+setup-CLI+meta module). The
  web UI (`:8175` guardian, `:8176` gateway) is documented as a secondary human path only.
- **Primary install path, per domain (also what validation installs):**
  - wallet → **release binary** (`curl` the static `fedimint-cli-v0.11.1` from GitHub releases).
  - federation & gateway → **docker-compose** (repo `docker/fedimintd`, `docker/gatewayd`).
- **Validation fidelity:** subagents run the **real steps** themselves (own `fedimintd`/`gatewayd`,
  own DKG ceremony, own `connect-fed`), using devimint only for the surrounding substrate
  (regtest bitcoind + lightning, and — for the wallet skill — a funded federation).
- **Version pin:** everything (substrate build, release binaries, docker images) = **v0.11.1**, so
  wire/API versions match across the pieces.

## Validation harness

### Substrate = devimint, driven headlessly via `--exec`

`just devimint-env` wraps `devimint dev-fed --exec bash -c devimint_env`. The interactive shell is
just the default `--exec` payload; `--exec <cmd…>` runs any script against a fully-set-up env and
tears down after (`devimint/src/cli.rs`). Keep-alive form for letting a subagent work against it:

```bash
devimint <sub> --exec bash -c 'touch "$FM_TEST_DIR/ready"; sleep infinity'
```

- `dev-fed` → funded 4-guardian federation + LDK & LND gateways + CLN/LND lightning + funded
  `fedimint-cli` client. Used for the **wallet** skill (real ecash/LN/on-chain flows, no real funds).
- `external-daemons` → just bitcoind + lightning nodes. Used for **federation-setup** and
  **gateway** skills, so the subagent stands up its *own* `fedimintd`/`gatewayd`.

The substrate is built once via nix from the v0.11.1 flake (`#devimint`, `#fedimint-pkgs`,
`#gateway-pkgs`; external daemons from the dev shell). The nix workspace compile is the main
one-time cost (~tens of minutes; fedimint crates are not cache-hits under the current nixpkgs).

### Environment topology

This host: NixOS, `nix` 2.34 (flakes), docker 29.5 daemon, passwordless sudo, 24 cores / 125 GB.
Other user workloads run on this host — **everything created here is namespaced `fmskill-*` and only
`fmskill-*` resources are ever cleaned up; never a global docker prune.** Host port 18443 is in use,
so the substrate runs in its own container network, not host ports.

- Substrate runs in/behind a dedicated docker network `fmskill-net`.
- Validation containers (`ubuntu:24.04`, `debian:12`, `nixos/nix`) join `fmskill-net`, install per
  the skill's documented path, and reach the substrate (invite code / regtest RPC / gateway addr)
  over the network.

### Protocol per skill (writing-skills RED→GREEN→REFACTOR)

1. **RED (optional baseline):** note where a fresh agent flails without the skill.
2. **GREEN:** spawn a subagent given *only* that `SKILL.md` + a concrete task + a handle to the
   running substrate + "you are on a fresh `<distro>`". It attempts the task in Ubuntu, then Debian,
   then NixOS.
3. **REFACTOR:** read each transcript for friction the *docs* should have removed; fold fixes back
   into the skill; re-run until green on all three distros.
4. **One commit** for that skill. Then move to the next.

### Example validation tasks

- **wallet:** join the federation (invite code) → receive 5 000 msat via `ln-invoice` +
  `await-invoice` → `spend` 1 000 msat OOB and `reissue` it → `deposit-address`, mine to it,
  `await-deposit` → `withdraw` to a regtest address. (ecash + LN + on-chain in one run.)
- **federation:** stand up a solo `fedimintd` on the regtest backend, complete the setup ceremony,
  emit an invite code, set `federation_name` = `ValidationMint`, verify a fresh client sees it
  (`fedimint-cli dev meta-fields`).
- **gateway:** run an LDK `gatewayd`, `connect-fed` to the running federation, peg-in ecash, confirm
  non-zero `balance`, set routing fees.

## Skill contents (CLI-first)

- **fedimint-bitcoin-wallet:** install (release binary; docker/nix noted) · `join` · `info` ·
  ecash `spend`/`validate`/`reissue`/`decode` · Lightning `ln-invoice`/`await-invoice`/`ln-pay` +
  gateway selection (`list-gateways`/`switch-gateway`) · on-chain `deposit-address`/`await-deposit`/
  `withdraw` · `backup`/`restore`/`print-secret` · reading meta (`dev meta-fields`) · common errors.
- **fedimint-federation-setup:** run `fedimintd` (docker-compose; binary noted) · `FM_PASSWORD_UI`
  vs `FM_PASSWORD_API` · setup ceremony (`admin setup <endpoint> set-local-params/add-peer/start-dkg`
  or UI) · solo vs multi-guardian (3m+1) · get invite code · **meta fields** via the meta module
  (submit + threshold; solo → single submit reaches consensus) · `admin status/audit/
  guardian-config-backup/shutdown` · troubleshooting.
- **fedimint-gateway-operation:** run `gatewayd` (docker-compose LDK; LND noted) · password hash ·
  `gateway-cli connect-fed`/`info`/`balance`/`leave-fed` · funding via peg-in · routing fees ·
  `payment-log`/`payment-summary` · backup (mnemonic + invite codes) · LNv1 vs LNv2 · registration
  with guardians · troubleshooting.

Structure: inline `SKILL.md`; a `references/*.md` file only where a section would exceed ~100 lines
(e.g. full command tables). Web-UI steps kept to a short secondary subsection.

## macOS testing strategy (documented, deferred — not executed)

No macOS host in this environment. Recorded in `references/testing-on-macos.md`:

- Native `nix` (Determinate installer supports macOS) running devimint on `aarch64-darwin` is the
  faithful substrate; Docker Desktop's Linux VM is *not* representative of native macOS.
- Validate the wallet skill's release-binary path with the `*-aarch64-apple-darwin` release assets;
  validate docker-compose paths via Docker Desktop.
- Watch for macOS-specific friction: Gatekeeper quarantine on downloaded binaries
  (`xattr -d com.apple.quarantine`), BSD vs GNU CLI flag differences, `host.docker.internal`.

## Known fidelity caveats (documented)

- Skills teach docker-compose for operators (mainnet/signet); headless validation runs the
  equivalent binaries/config against regtest. The config knowledge is validated; a literal
  `docker compose up` on mainnet/signet is not.
- `nixos/nix` stands in for a NixOS device (nix package manager on a minimal base), not a full
  NixOS system.

## Execution status

- [x] Research fedimint (CLIs, deploy paths, meta module, devimint `--exec`).
- [x] Environment confirmed: docker + sudo + nix available.
- [x] Pin v0.11.1; background nix build of devimint/fedimint-pkgs/gateway-pkgs started.
- [ ] Spike: headless `dev-fed`/`external-daemons` reachable from a container.
- [ ] Skill 1 `fedimint-bitcoin-wallet` → validate (ubuntu/debian/nixos) → commit.
- [ ] Skill 2 `fedimint-federation-setup` → validate → commit.
- [ ] Skill 3 `fedimint-gateway-operation` → validate → commit.
- [ ] macOS strategy doc (deferred execution).
