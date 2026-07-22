# fedimint-skills

Agent-facing [skills](https://agentskills.io) for operating [Fedimint](https://github.com/fedimint/fedimint)
as an end user. Each skill is a self-contained `SKILL.md` that teaches an AI agent how to accomplish a
concrete, real-world Fedimint task from the command line.

Pinned to Fedimint **v0.11.1**.

## Skills

| Skill | What it covers |
|-------|----------------|
| [`fedimint-cli-wallet`](fedimint-cli-wallet/SKILL.md) | Use `fedimint-cli` as a Lightning, on-chain, and ecash wallet: join a federation, check balance, send/receive ecash, Lightning payments via a gateway, and on-chain deposits/withdrawals. |
| [`fedimint-federation-setup`](fedimint-federation-setup/SKILL.md) | Run `fedimintd`, complete the setup/DKG ceremony, obtain an invite code, and operate the federation — including setting meta fields. |
| [`fedimint-gateway-operation`](fedimint-gateway-operation/SKILL.md) | Run and operate a Lightning gateway (`gatewayd`/`gateway-cli`): connect to federations, fund with ecash, manage routing fees, and monitor payments. |

All skills are **CLI-first** (the web UIs at `:8175`/`:8176` are noted as a secondary human path).

## Design & validation

See [`docs/superpowers/specs/2026-07-21-fedimint-skills-design.md`](docs/superpowers/specs/2026-07-21-fedimint-skills-design.md)
for the design and the validation methodology. Each skill was validated by fresh subagents that received
*only* that skill plus a concrete task, working in clean Ubuntu, Debian, and NixOS containers against a
real regtest [devimint](https://github.com/fedimint/fedimint/tree/master/devimint) substrate — all nine
runs (3 skills × 3 distros) passed.

- [`docs/validation-notes.md`](docs/validation-notes.md) — what each validation transcript surfaced and
  the documentation fixes it drove.
- [`references/testing-on-macos.md`](references/testing-on-macos.md) — macOS testing strategy. The
  wallet skill has since been validated on a native Apple-Silicon Mac (macOS 26.2); federation/gateway
  operator skills remain deferred (they target servers, not Macs).
