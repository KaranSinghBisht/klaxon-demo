# klaxon-demo

A CI pipeline that deploys a treasury contract with a **deployer private key**. The same secret is
protected two ways, so you can watch the difference.

- **`ordinary.yml`** — the key is a plain GitHub Actions secret. A compromised dependency
  (`worm/`, a postinstall) reads the job environment on `npm install` and ships every variable to a
  collector. The key is in that dump.
- **`deploy.yml`** — the key is split by [KLAXON](https://github.com/KaranSinghBisht/klaxon). Share A
  is encrypted under a Ledger Key Ring and committed here (`.klaxon/DEPLOYER_PRIVATE_KEY.enc`); share
  B is released by a witness service **only** against a runner-made x402 micropayment on Hedera whose
  memo equals a signed commitment, bound to this workflow's GitHub OIDC identity. The plaintext lives
  in exactly one step's environment. A worm in any other job sees nothing.
- **`worm-attack.yml`** — the attacker has the member key, the pay key, the encrypted share, and a
  copy of the exact release step. They run it from a job they control. The witness takes the payment,
  then **refuses** — the OIDC token proves the job is not the one the policy authorizes — **revokes
  the project**, and the owner's phone buzzes.

  Be precise about what the payment is. It is a commitment: a real, publicly ordered, timestamped act
  that names the attempt before any secret can move, which neither the witness nor the runner can
  forge, suppress or back-date. It is **not** a cost borne by the attacker — here they are spending
  the pay credential they just stole from the runner, so the victim funds it. The property worth
  having is that the attempt cannot happen quietly, not that it is expensive.

## What's real

Every release is a real Hedera testnet payment settled through the Blocky402 facilitator, memo'd with
the commitment, and recorded on a Hedera Consensus Service topic. The registry and the treasury are
on Ethereum Sepolia. Nothing here is mocked.

| Thing | Where |
|---|---|
| Witness (live x402 service) | `https://44-198-37-65.sslip.io` — see `/.well-known/klaxon.json` |
| Baseline treasury, key is stolen | [`0xa3Ddb847…`](https://sepolia.etherscan.io/address/0xa3Ddb8470b184042c17E9858d1b406D908Fa2812) — owner `0x0CD00642…` |
| Protected treasury, key never leaves KLAXON | [`0xe44A598a…`](https://sepolia.etherscan.io/address/0xe44A598aA52E2ceF879916961c4a6167f8aD4F96) — owner `0x20140254…` |
| Registry (Sepolia) | `0xd93f10104d4069B26c8ee883c3eAb3AAaaD56885` |
| HCS audit topic | `0.0.10503843` |
| Policy (anchored on-chain) | `klaxon.policy.json`, sha256 committed by the owner's Ledger |

## Reproduce

Run the workflows from the Actions tab (`workflow_dispatch`). `ordinary` and `deploy` both deploy a
treasury to Sepolia; `worm-attack` is refused and revokes the project (recover with the physical
Ledger). Secrets required: `KLAXON_MEMBER`, `KLAXON_PAY_KEY`, `DEPLOYER_PRIVATE_KEY`,
`SEPOLIA_RPC_URL`, and `COLLECTOR_URL` for the collector sink.
