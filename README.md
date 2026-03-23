<!-- SPDX-License-Identifier: CC-BY-SA-4.0 -->

# GPUl

*A decentralized compute marketplace where idle GPUs earn credits redeemable for on-demand access to any hardware in the network.*

> **GPUl** (pronounced "G-Pool") — a peer-to-peer network where GPU owners pool idle hardware, earn credits, and trade up to any compute tier in the network.

---

## What This Is

GPUl is a protocol for a decentralized GPU compute marketplace. GPU owners contribute idle hardware to the network and earn fungible credits. Those credits can be spent on any other GPU in the network — including hardware tiers the contributor doesn't own.

A researcher with an RTX 4090 can bank idle time and trade up to an A100 or H100 when they need it. A data center with idle A100s earns stored value during off-peak hours. Supply and demand set exchange rates through an open auction mechanism.

## What This Is Not

- **Not a centralized cloud provider.** There is no central operator. All participants — buyers, sellers, and validators — collectively form the network.
- **Not a generic "AI + blockchain" project.** GPUl solves a specific problem (GPU access and utilization) with a concrete mechanism (credit-based exchange with auction pricing). The protocol uses blockchain primitives where they add value (credit ledger, staking) and avoids them where they don't.
- **Not a token launch.** Credits are the internal unit of account. No credits are pre-mined or allocated to founders.

## How It Works

1. **Contribute** — Run the Node Agent on your machine. Your idle GPU is registered with the network and available for jobs.
2. **Earn** — When the network assigns a job to your GPU, you execute it in a sandboxed container and earn credits based on wall time and the auction clearing price.
3. **Spend** — Use your credits to submit your own jobs to any GPU in the network. The network matches your job to available hardware based on your requirements and bid.

Credits **circulate** — they are never burned. When a requester pays for compute, those credits transfer to the contributor who performed the work. Total supply is conserved. Auction-driven pricing means rates float with supply and demand per GPU class.

## Design Principles

- **Credits are the unit of account.** No fiat currency required to participate.
- **Credits circulate, never burn.** When a requester pays for compute, those credits transfer through the network to the contributor who performed the work. Total supply is conserved.
- **Decentralized by design.** There is no central operator. All participants collectively form the network.
- **Auction-driven pricing.** Supply and demand set exchange rates through an open auction mechanism.
- **Security by default.** All job outputs are encrypted end-to-end. Contributors never see results.
- **Honest by incentive.** Staking and slashing make cheating economically irrational.

## Current Status

GPUl is in the **design phase**. The technical design document is complete and covers the full protocol: node agent, job lifecycle, credit economics, encryption model, proof of compute, storage, and API surface. No code has been written yet.

## Documentation

| Document | Description |
|----------|-------------|
| [Technical Design Document](Docs/gpu_exchange_design_doc.md) | Full protocol specification (v0.1 draft) |

## Proposed Architecture

The system is composed of five major subsystems:

| Component | Responsibility | Runs On |
|-----------|---------------|---------|
| **Node Agent** | Manages local GPU, executes jobs, encrypts output, reports health | Contributor's machine |
| **Orchestrator** | Matches jobs to nodes, manages queues, enforces SLAs | Distributed across network nodes |
| **Credit Ledger** | Tracks balances, processes transfers, records transactions | Blockchain or signed ledger |
| **Storage Layer** | Stores encrypted job outputs, serves retrieval requests | Distributed (IPFS / S3-compatible) |
| **Verification Service** | Validates that compute was performed correctly | Distributed across network nodes |

## License

GPUl uses a split-license model:

- **Code** — [AGPL-3.0-or-later](LICENSE-AGPL)
- **Documentation** — [CC BY-SA 4.0](LICENSE-CC-BY-SA)

See [LICENSES.md](LICENSES.md) for full details.
