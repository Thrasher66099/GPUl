<!-- SPDX-License-Identifier: CC-BY-SA-4.0 -->

# GPUl

**Technical Design Document**

Version 0.1 — Draft | March 2026

*A decentralized compute marketplace where idle GPUs earn credits redeemable for on-demand access to any hardware in the network.*

> **GPUl** (pronounced "G-Pool") — a peer-to-peer network where GPU owners pool idle hardware, earn credits, and trade up to any compute tier in the network.

---

## 1. Overview

### 1.1 Problem

High-end GPU hardware is expensive, frequently idle, and inaccessible to most independent researchers and small teams. Cloud GPU rental is priced for continuous use, making occasional burst compute disproportionately costly.

### 1.2 Solution

A peer-to-peer network that allows GPU owners to contribute idle hardware in exchange for fungible compute credits. Credits can be redeemed for time on any GPU in the network, enabling access to hardware tiers that would otherwise be out of reach.

### 1.3 Core Value Propositions

- **For small-card owners (e.g. RTX 4090):** Bank idle time and trade up to high-end compute (A100, H100) when needed.
- **For large-card owners (e.g. A100):** Earn stored value during idle periods; redeem later for lighter workloads without maintaining separate hardware.
- **For the ecosystem:** Reduce global GPU waste, lower the barrier to AI research, and create a liquid market for compute.

### 1.4 Design Principles

- **Credits are the unit of account.** No fiat currency required to participate.
- **Credits circulate, never burn.** When a requester pays for compute, those credits transfer through the network to the contributor who performed the work. Total supply is conserved.
- **Decentralized by design.** There is no central operator. All participants — buyers, sellers, and validators — collectively form the network, like a blockchain.
- **Auction-driven pricing.** Supply and demand set exchange rates through an open auction mechanism. More available GPUs drive prices down; more demand drives prices up.
- **Security by default.** All job outputs are encrypted end-to-end. Contributors never see results.
- **Honest by incentive.** Staking and slashing make cheating economically irrational.

---

## 2. System Architecture

### 2.1 High-Level Components

The system is composed of five major subsystems that communicate through well-defined interfaces.

| Component | Responsibility | Runs On |
|-----------|---------------|---------|
| Node Agent | Manages local GPU, executes jobs, encrypts output, reports health | Contributor's machine |
| Orchestrator | Matches jobs to nodes, manages queues, enforces SLAs | Distributed across network nodes |
| Credit Ledger | Tracks balances, processes transfers between participants, records transactions | Blockchain or signed ledger |
| Storage Layer | Stores encrypted job outputs, serves retrieval requests | Distributed (IPFS / S3-compatible) |
| Verification Service | Validates that compute was performed correctly | Distributed across network nodes |

### 2.2 Component Interactions

The network is decentralized: all participants collectively provide the infrastructure, similar to a blockchain. The Orchestrator logic is distributed across network nodes rather than running on a central server. When a requester submits a job, the network matches it to available GPUs based on the job's requirements, assigns it to a qualified node, and monitors execution. The Node Agent on the contributor's machine pulls the job payload, executes it, encrypts the output with the requester's public key, uploads the encrypted result to the Storage Layer, and reports completion. The Verification Service spot-checks a configurable percentage of completed jobs. Upon verified completion, the Credit Ledger atomically transfers credits from the requester to the contributor who performed the work.

### 2.3 Protocol Maturity Roadmap

- **Phase 1 — Coordinating Bootstrap Nodes:** A small set of bootstrap nodes assists with matchmaking and scheduling while the network grows. The Credit Ledger is cryptographically signed and append-only for auditability. Even in this phase, the protocol is permissionless — anyone can join as a buyer or seller.
- **Phase 2 — Federated Orchestration:** Multiple orchestrator nodes with consensus. Credit Ledger moves to a lightweight chain or L2.
- **Phase 3 — Fully On-Chain:** Smart-contract-based job matching and settlement. Orchestrator logic runs on-chain or via a decentralized protocol. Storage pointers and credit transfers are fully on-chain.

---

## 3. Node Agent

The Node Agent is the software that runs on every contributing machine. It is the most critical piece of infrastructure from the contributor's perspective.

### 3.1 Responsibilities

- Register the node with the network (GPU model, VRAM, compute capability, bandwidth)
- Send periodic heartbeats to prove liveness and availability
- Pull job payloads from the network when assigned
- Execute jobs in a sandboxed environment (container or VM)
- Encrypt job outputs using the requester's public key (see Section 6)
- Upload encrypted outputs to the Storage Layer
- Report execution metadata (wall time, GPU utilization, checksums) for verification

### 3.2 Sandboxing

Jobs must execute in isolation to protect both the contributor's host system and the requester's data. The Node Agent launches each job inside a container (Docker with GPU passthrough via NVIDIA Container Toolkit) or a lightweight VM. The sandbox has no network access by default; only the encrypted output upload path is permitted. The host filesystem is inaccessible from within the sandbox.

### 3.3 Health and Availability

Nodes send heartbeats every 30 seconds. If three consecutive heartbeats are missed, the node is marked offline and any in-progress jobs are rescheduled. Nodes can declare scheduled downtime windows to avoid being assigned jobs they cannot complete.

### 3.4 Graceful Shutdown

If a contributor needs their GPU back (e.g. to play a game), the Node Agent signals the network with a graceful shutdown request. The current job is checkpointed if the workload supports it, and the job is migrated to another node. Frequent premature shutdowns affect the node's reputation score but do not trigger slashing unless a job is abandoned without the graceful shutdown protocol.

---

## 4. Job Lifecycle

### 4.1 End-to-End Flow

1. Requester submits a job specification: container image, input data reference, GPU requirements (model, VRAM, minimum compute capability), maximum wall time, and credit bid.
2. The network validates the request, confirms the requester has sufficient credit balance, and places a hold on the bid amount.
3. The network queries the Node Registry for available nodes matching the requirements and selects the best match based on price, latency, and reputation score.
4. Selected Node Agent pulls the job payload and launches it in a sandbox.
5. Node Agent streams execution metrics (GPU utilization, memory usage, progress indicators if supported by the workload).
6. Upon completion, the Node Agent encrypts the output using the requester's public key and a symmetric session key (see Section 6), uploads the encrypted blob to the Storage Layer, and reports the storage pointer and execution metadata.
7. Verification Service performs spot-check validation (see Section 7).
8. Credit Ledger atomically transfers credits from the requester to the contributor based on actual wall time and the agreed rate.
9. Requester retrieves the encrypted output from the Storage Layer and decrypts it locally with their private key.

### 4.2 Job Specification Schema

Jobs are defined as a declarative specification. The minimal required fields are shown below.

```json
{
  "job_id": "uuid-v4",
  "container_image": "registry.gpul.net/user/training-job:v1.2",
  "input_refs": ["storage://abc123", "storage://def456"],
  "gpu_requirements": {
    "min_vram_gb": 24,
    "min_compute_capability": 8.0,
    "preferred_models": ["A100", "H100"],
    "max_wall_time_minutes": 120
  },
  "encryption": {
    "requester_public_key": "base64-encoded-key",
    "output_encryption": "AES-256-GCM"
  },
  "credit_bid": {
    "max_credits_per_hour": 150,
    "max_total_credits": 300
  }
}
```

### 4.3 Job States

| State | Description | Transitions To |
|-------|-------------|---------------|
| SUBMITTED | Job received, credit hold placed | QUEUED or REJECTED |
| QUEUED | Waiting for a matching node | ASSIGNED or CANCELLED |
| ASSIGNED | Node selected, payload being transferred | RUNNING or QUEUED (on failure) |
| RUNNING | Actively executing on contributor hardware | COMPLETED, FAILED, or MIGRATING |
| MIGRATING | Checkpointing and moving to a new node | ASSIGNED |
| COMPLETED | Output encrypted and uploaded, awaiting verification | VERIFIED or DISPUTED |
| VERIFIED | Spot-check passed, credits settled | Terminal |
| DISPUTED | Verification failed, under review | VERIFIED or SLASHED |
| SLASHED | Contributor penalized, job re-queued | QUEUED |
| FAILED | Unrecoverable error during execution | Terminal (credits refunded) |
| CANCELLED | Requester cancelled before execution began | Terminal (hold released) |

---

## 5. Token Economics

### 5.1 Credit System

The network uses a single fungible credit token as its unit of account. Credits circulate between participants: when a requester pays for compute, those credits are transferred through the network to the GPU owner who performed the work. Credits are never burned — they flow from buyer to seller in a continuous cycle, just as tokens move between wallets on a blockchain. There is no fixed exchange rate to fiat currency; credits derive value purely from their redeemability for compute within the network.

### 5.2 Auction-Based Pricing

Each GPU model class has an independent floating price determined by real-time supply and demand within the network, using an auction mechanism.

- **Auction model:** The network maintains a decentralized order book per GPU class. Contributors set a minimum ask (credits per hour they're willing to accept). Requesters set a maximum bid. The clearing price emerges where supply meets demand — as more GPUs become available, prices fall; as more jobs compete for hardware, prices rise.
- **Self-correcting equilibrium:** When supply is abundant, competition among contributors drives prices down, attracting more requesters. When demand surges, rising prices incentivize more GPU owners to contribute capacity, restoring balance.
- **Rate caps:** To prevent price manipulation in a thin market, the protocol enforces a maximum rate of change per time window (e.g. no more than 20% price swing per hour).

### 5.3 Exchange Rates Between GPU Classes

A 4090 hour is not equivalent to an A100 hour. The market naturally establishes relative pricing. If A100 supply is scarce and demand is high, its credit-per-hour rate rises, meaning a 4090 contributor must bank more hours to afford an A100 hour. Conversely, when A100 supply is abundant, the exchange becomes more favorable. This self-corrects: high A100 prices incentivize more A100 owners to contribute, increasing supply and pushing prices back down.

### 5.4 Staking

Every contributing node must stake a deposit before accepting jobs. The stake serves as a bond against misbehavior.

| Parameter | Value | Rationale |
|-----------|-------|-----------|
| Minimum stake | Proportional to GPU tier | Higher-value hardware handles higher-value jobs |
| Slash: abandoned job | 10–25% of stake | Proportional to wasted requester time |
| Slash: failed verification | 25–50% of stake | Strong deterrent against returning bad results |
| Slash: repeated early shutdowns | 5% cumulative | Encourages reliability without over-penalizing occasional interruptions |
| Stake unlock period | 48 hours after last job | Prevents stake-and-run attacks |

### 5.5 Credit Minting and Circulation

Credits are minted when compute is contributed and verified, creating the initial supply that enters circulation. Once minted, credits are never destroyed — they circulate perpetually between participants. When a requester pays for a job, those credits transfer to the contributor who performed the work. That contributor can then spend those same credits to request compute from others, and so on. The total credit supply grows as new compute is contributed to the network, reflecting the cumulative useful work the network has produced. No credits are pre-mined or allocated to founders. If the network needs initial liquidity, a bootstrap mechanism (Section 10) provides it.

---

## 6. Encryption and Security Model

### 6.1 Threat Model

We assume contributors are semi-honest: they will follow the protocol if it is economically rational, but will cheat if the expected value of cheating exceeds the expected penalty. We do not assume contributors are trustworthy with plaintext data. The security model aims to minimize the window during which data is exposed and make data exfiltration detectable and punishable.

### 6.2 Key Management

Every user (requester or contributor) generates a key pair upon registration.

- **Public key:** Published to the network. Used by contributors to encrypt job outputs destined for a specific requester.
- **Private key:** Never leaves the requester's local machine. Used to decrypt job outputs.
- **Key algorithm:** X25519 for key exchange, Ed25519 for signatures. Both are fast, well-audited, and widely supported.

### 6.3 Envelope Encryption for Job Outputs

Raw asymmetric encryption is impractical for large outputs (model checkpoints, datasets). The network uses envelope encryption, which is the industry standard approach.

1. The Node Agent generates a random 256-bit symmetric key (the session key) for each job.
2. The job output is encrypted with AES-256-GCM using the session key. GCM provides both confidentiality and integrity (authenticated encryption).
3. The session key is encrypted with the requester's X25519 public key.
4. The encrypted output blob and the encrypted session key are uploaded together to the Storage Layer.
5. The requester downloads the bundle, decrypts the session key with their X25519 private key, and decrypts the output with the recovered session key.

### 6.4 Data Exposure During Execution

This is the hardest security problem in decentralized compute. During execution, the job's input data and model weights exist in plaintext in GPU memory. The current design accepts this exposure window and mitigates it through several layers.

- **Sandboxing:** No network access from within the execution container. The contributor cannot exfiltrate data during the job.
- **Reputation and staking:** Contributors with high stakes and long track records are preferentially matched to sensitive jobs.
- **TEE integration (future):** When AMD SEV-SNP or Intel TDX support matures for GPU workloads, the Node Agent can execute inside a trusted enclave, providing hardware-attested confidentiality.
- **Job splitting (future):** For highly sensitive workloads, the network can split a job across multiple nodes such that no single node sees the complete input. This is workload-dependent and adds latency.

### 6.5 Integrity of Job Inputs

Job inputs are hashed before submission. The Node Agent verifies the hash before execution, ensuring inputs were not tampered with in transit or at rest. The hash is recorded on the Credit Ledger alongside the job record.

---

## 7. Proof of Compute

### 7.1 The Verification Problem

A malicious contributor could return garbage data, claim they executed the job, and collect credits. The network must make this strategy unprofitable.

### 7.2 Strategy: Optimistic Execution with Probabilistic Spot-Checking

The network uses an optimistic model: jobs are assumed to be executed honestly, and a random subset is re-executed by a Verification Node to check. The parameters are tunable.

| Parameter | Default | Notes |
|-----------|---------|-------|
| Spot-check rate | 5–10% of jobs | Higher for new or low-reputation nodes |
| Verification tolerance | Bitwise match or statistical threshold | Floating-point non-determinism requires tolerance for GPU compute |
| Verification node selection | Random, different from executor | Prevents collusion |
| Dispute window | 1 hour after completion | Requester can also challenge results |

### 7.3 Verification Flow

1. The network randomly selects completed jobs for re-execution.
2. A different node (the Verifier) re-runs the job with identical inputs.
3. The Verifier's output hash is compared against the original executor's output hash.
4. If hashes match within tolerance, the original job is marked VERIFIED.
5. If hashes diverge beyond tolerance, the job enters DISPUTED state and a third node arbitrates.
6. If the dispute resolves against the original executor, their stake is slashed and the job is re-queued.

### 7.4 GPU Floating-Point Non-Determinism

Identical GPU computations can produce slightly different results across different hardware due to floating-point ordering, driver versions, and hardware-level non-determinism. The verification system must account for this. For AI workloads, verification uses statistical comparison (e.g. cosine similarity of output tensors above 0.9999) rather than bitwise matching. The tolerance threshold is specified per job type and is a configurable parameter.

### 7.5 Requester-Initiated Challenges

If a requester suspects bad output (e.g. a training run that produced a model with anomalous loss), they can initiate a challenge within the dispute window. The network triggers a verification re-execution. If the challenge is upheld, the contributor is slashed and the requester is refunded plus a small bonus from the slashed stake. If the challenge is frivolous (original output was correct), the requester pays a small challenge fee to prevent spam.

---

## 8. Storage Architecture

### 8.1 Design Principles

- All stored data is encrypted. The Storage Layer never sees plaintext.
- The Storage Layer is a dumb blob store. It does not interpret, index, or process contents.
- On-chain (or on-ledger) records contain only pointers (content hashes / URIs), never data.
- Requesters are responsible for retrieving and decrypting their own outputs.

### 8.2 Storage Flow

1. Node Agent encrypts output (see Section 6.3), producing an encrypted blob.
2. Node Agent computes a SHA-256 hash of the encrypted blob (the content address).
3. Node Agent uploads the blob to the Storage Layer. The storage URI is returned.
4. Node Agent reports the storage URI and content hash to the network.
5. The network records the storage URI and content hash on the Credit Ledger as part of the job completion record.
6. The requester uses the storage URI to retrieve the blob and the content hash to verify integrity before decrypting.

### 8.3 Storage Backends

The protocol is storage-agnostic. The initial implementation supports two backends.

- **IPFS:** Content-addressed, decentralized. Natural fit for the network's ethos. Pinning services (Pinata, web3.storage) provide persistence guarantees.
- **S3-compatible object storage:** Simpler operations, better performance for large blobs. Can be self-hosted (MinIO) or cloud-hosted. Appropriate for Phase 1.

### 8.4 Retention and Garbage Collection

Encrypted outputs are retained for a configurable period (default: 30 days). After the retention window, storage nodes may garbage-collect the blob. The on-ledger pointer remains permanently as a receipt. Requesters who need longer retention can pin their data on IPFS or mirror it to their own storage.

---

## 9. API Surface

GPUl exposes a REST API for programmatic access and a CLI for interactive use. Both authenticate via Ed25519 signed requests.

### 9.1 Core Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/nodes/register` | POST | Register a new node with GPU specs and stake deposit |
| `/nodes/{id}/heartbeat` | POST | Report node health and availability |
| `/nodes/{id}/status` | GET | Get node status, reputation, and earnings |
| `/jobs/submit` | POST | Submit a new job specification |
| `/jobs/{id}/status` | GET | Query job state and execution metrics |
| `/jobs/{id}/result` | GET | Retrieve encrypted output storage pointer |
| `/jobs/{id}/challenge` | POST | Initiate a verification challenge |
| `/credits/balance` | GET | Query current credit balance and holds |
| `/credits/history` | GET | Transaction history with filtering |
| `/market/rates` | GET | Current credit-per-hour rates by GPU class |
| `/market/supply` | GET | Available GPU inventory by class |

### 9.2 Authentication

All API requests are signed with the user's Ed25519 private key. The signature covers the request method, path, timestamp, and body hash. The server verifies against the user's registered public key. Timestamps must be within a 5-minute window to prevent replay attacks.

### 9.3 CLI

The CLI wraps the REST API and manages local key storage. Example commands:

```bash
gpul node register --gpu-model RTX4090 --stake 500
gpul node start
gpul job submit --spec job.json
gpul job status <job-id>
gpul job download <job-id> --output ./results/
gpul credits balance
gpul market rates
```

---

## 10. Bootstrapping and Cold Start

### 10.1 The Chicken-and-Egg Problem

The network has no value if there is no compute to redeem credits against. Contributors will not join if there are no requesters, and requesters will not join if there is no available hardware. This requires a deliberate bootstrapping strategy.

### 10.2 Bootstrap Mechanisms

- **Seed hardware:** The founding team contributes initial GPU capacity to guarantee minimum availability. This hardware operates at cost and ensures early requesters can actually use the network.
- **Fiat on-ramp (optional):** Allow users to purchase credits directly during the bootstrap phase. This gives requesters immediate access even before contributing hardware. The on-ramp can be phased out once organic supply is sufficient.
- **Contributor incentive bonus:** Early contributors earn a temporary bonus multiplier on credits earned (e.g. 1.5x for the first 90 days). This accelerates supply-side growth.
- **Partnership seeding:** Partner with AI research labs, universities, or hackathon organizers who can bring both supply and demand simultaneously.

---

## 11. MVP Scope

The minimum viable product focuses on proving the core loop: contribute GPU time, earn credits, spend credits on someone else's GPU. Everything else is deferred.

### 11.1 In Scope for MVP

- Node Agent with Docker-based sandboxing and GPU passthrough
- Orchestrator with job matching and scheduling (bootstrap nodes in Phase 1)
- Envelope encryption for job outputs (X25519 + AES-256-GCM)
- S3-compatible storage backend for encrypted outputs
- Signed append-only credit ledger (not yet on-chain)
- Auction-based pricing with a decentralized order book per GPU class
- Staking with basic slashing for abandoned jobs
- Spot-check verification at 10% rate
- REST API and CLI client
- Support for inference and fine-tuning job types

### 11.2 Deferred to Post-MVP

- Blockchain-based credit ledger and on-chain settlement
- Fully on-chain orchestration and settlement
- TEE-based confidential compute
- Job splitting across multiple nodes
- Distributed training support (multi-node synchronous training)
- Fiat on-ramp / off-ramp
- Mobile monitoring app
- Web dashboard for analytics

---

## 12. Proposed Repository Structure

```
gpul/
├── README.md
├── docs/
│   ├── design-doc.md              # This document
│   ├── architecture.md            # Diagrams and component details
│   ├── token-economics.md         # Credit system deep-dive
│   ├── security-model.md          # Encryption, keys, threat model
│   ├── verification.md            # Proof of compute details
│   └── api-reference.md           # Full API documentation
├── agent/                         # Node Agent (Rust or Go)
│   ├── src/
│   ├── Dockerfile
│   └── README.md
├── orchestrator/                  # Job scheduler and matchmaker
│   ├── src/
│   ├── Dockerfile
│   └── README.md
├── ledger/                        # Credit ledger service
│   ├── src/
│   └── README.md
├── crypto/                        # Shared encryption library
│   ├── src/
│   └── README.md
├── cli/                           # Command-line client
│   ├── src/
│   └── README.md
├── proto/                         # Protocol buffer / API definitions
│   └── gpul.proto
├── deploy/                        # Docker Compose, Kubernetes configs
└── tests/                         # Integration and end-to-end tests
```

---

## 13. Open Questions

The following questions are unresolved and should be addressed before or during MVP development.

- **Language choice for Node Agent:** Rust offers memory safety and performance. Go offers faster development and simpler deployment. The agent is performance-sensitive (encryption, GPU management) which favors Rust, but community contributions may be easier with Go.

- **Credit fungibility and transferability:** Should credits be transferable between users? Transferability creates a secondary market (good for liquidity) but also enables speculation and potential regulatory issues as a token/security.

- **Multi-GPU jobs:** The MVP assumes single-GPU jobs. Supporting multi-GPU (e.g. 8xA100 for distributed training) requires coordinated scheduling across multiple contributors with low-latency interconnects. This is substantially harder and may require dedicated data center partnerships.

- **Data sovereignty:** Some users may have legal restrictions on where their data can be processed (GDPR, HIPAA). The network may need to support geographic constraints on node selection.

- **Abuse prevention:** The network could be used for crypto mining, CSAM generation, or other abusive compute. The Node Agent should reject known-malicious container images, and the network should implement reporting and takedown mechanisms.

- **Verification cost:** Spot-checking consumes real GPU time. Who pays for verification compute? Options include a small network fee on all jobs (spreads cost), the contributor pays from their earnings (incentive misalignment), or verification nodes earn credits at a reduced rate (creates a verification market).

- **Insurance and SLAs:** What guarantees does the network offer if a job fails due to hardware error? A small insurance pool funded by transaction fees could cover retries.
