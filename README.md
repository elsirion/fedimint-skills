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
for the design and the validation methodology. Each skill is validated by fresh subagents that receive
*only* that skill plus a concrete task, working in clean Ubuntu, Debian, and NixOS containers against a
real regtest [devimint](https://github.com/fedimint/fedimint/tree/master/devimint) substrate. A macOS
testing strategy is documented in [`references/testing-on-macos.md`](references/testing-on-macos.md)
(deferred).
