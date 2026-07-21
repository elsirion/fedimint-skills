# macOS testing strategy (deferred)

The three skills were validated on Linux (Ubuntu 24.04, Debian 12, `nixos/nix`) against a real v0.11.1
`devimint`/regtest substrate. macOS validation is **deferred** (no macOS host was available in the
build environment). This document is the plan for running it when a macOS machine (ideally
Apple-Silicon, `aarch64-darwin`) is available.

## Why macOS needs its own pass

- The **install paths differ**: no `apt`/`.deb`, no `.rpm`. macOS users get binaries from the
  `*-aarch64-apple-darwin` release tarballs, from **Nix** (Determinate Systems installer supports
  macOS), or from **Docker Desktop** images.
- **Gatekeeper quarantine**: binaries downloaded via a browser/curl get a `com.apple.quarantine`
  xattr and are blocked on first run (`"cannot be opened because the developer cannot be verified"`).
  Expect to need `xattr -d com.apple.quarantine <binary>` or `spctl` approval. The skills should gain
  a macOS note if this bites.
- **BSD vs GNU userland**: `sed`, `grep -P`, `base64`, `date`, `stat` differ from Linux. The skills
  lean on `grep`/`sed`/`jq` for JSON extraction — verify those one-liners on BSD tools (the `jq` path
  is portable; the `grep -oP` fallback in the wallet skill is GNU-only and will need a BSD variant).
- **Docker Desktop ≠ native**: Docker Desktop runs a Linux VM, so testing the `.deb`/docker paths
  inside it exercises Linux, not macOS. Native coverage means running the darwin binaries / Nix on the
  host directly.

## Recommended matrix

| Layer | How to test on macOS |
|-------|----------------------|
| Substrate (federation + gateway + LN + bitcoind) | `nix develop github:fedimint/fedimint/v0.11.1` then `devimint dev-fed --exec …` **natively** on `aarch64-darwin` (the same harness used on Linux — `devimint`/`fedimint-pkgs`/`gateway-pkgs` build or fetch for darwin). This is the faithful path. |
| `fedimint-cli-wallet` install | (a) `*-aarch64-apple-darwin` release binary + de-quarantine; (b) `nix profile install --accept-flake-config …#fedimint-cli`; (c) `docker run … fedimint/fedimintd` via Docker Desktop. |
| `fedimint-federation-setup` / `fedimint-gateway-operation` | Run `fedimintd`/`gatewayd` from the darwin release binaries or Nix against a native regtest `bitcoind` (Homebrew `bitcoin` or Nix). docker-compose path via Docker Desktop. |
| Validation method | Same as Linux: a fresh sub-agent gets only the `SKILL.md` + a task + the running substrate, and drives it via the terminal. No container is needed — macOS *is* the "device". |

## Execution notes

- Prefer a clean user account (or a fresh VM via `tart`/UTM) to simulate a fresh device, since macOS
  can't be as cheaply disposable as a Linux container.
- Reuse the exact validation tasks from the Linux runs (wallet: join→fund→spend→LN→deposit→withdraw;
  federation: solo ceremony→meta; gateway: deploy→init→connect-fed→pegin→fees).
- Capture the transcript and fold macOS-specific friction (quarantine, BSD flags, port defaults) back
  into the skills — most likely a short "macOS" note in each install section.
