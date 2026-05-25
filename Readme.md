# Aegistra — AI-Driven Code Verification & Escrow

> A decentralised freelance platform where an agentic AI autonomously verifies code submissions and releases milestone payments via Ethereum smart contracts — no middleman required.

---

## What is Aegistra?

Aegistra removes trust from the freelance equation. A client funds a milestone in ETH. A freelancer submits their code. An **agentic AI pipeline** scores the submission automatically and instructs the smart contract to release payment, trigger a jury, or reject the work — all without human intervention.

---

## How It Works

```
Client funds milestone (ETH locked in EscrowContract)
        ↓
Freelancer submits work (emits WorkSubmitted on-chain)
        ↓
Node.js Oracle picks up event → calls AI Verification Service
        ↓
Agentic AI scores submission (tests + static analysis)
        ↓
Score ≥ 75%  →  APPROVED  →  Payment released to freelancer
Score 45–74% →  DISPUTED  →  3-juror staked vote triggered
Score < 45%  →  REJECTED  →  Freelancer may resubmit
```

---

## Architecture

```
┌─────────────────────────────────────────────────────┐
│                     Frontend                        │
│            Next.js  +  MetaMask  +  ethers.js       │
└────────────────────┬────────────────────────────────┘
                     │ RPC / REST
┌────────────────────▼────────────────────────────────┐
│                  Ethereum (Sepolia)                  │
│   EscrowContract  │  DisputeContract  │  JuryStaking │
│   EvidenceRegistry (IPFS CIDs stored on-chain)      │
└────────────────────┬────────────────────────────────┘
                     │ WebSocket events
┌────────────────────▼────────────────────────────────┐
│                  Node.js Oracle                      │
│   Listens for WorkSubmitted → calls AI service       │
│   Signs & posts verdict back on-chain                │
└────────────────────┬────────────────────────────────┘
                     │ HTTP
┌────────────────────▼────────────────────────────────┐
│           AI Verification Service (FastAPI)          │
│                                                     │
│  Mode A — Metric Pipeline (CodeVerifier)            │
│    pytest → pylint → flake8 → weighted score        │
│                                                     │
│  Mode B — Agentic LLM Pipeline (CodeVerificationAgent)│
│    tools run in parallel → 3 LLM sub-agents:        │
│    RequirementsAnalyzer → TestFailureInterpreter     │
│    → VerdictSynthesizer                             │
│                                                     │
│  Result → explainability bundle → pinned to IPFS    │
└─────────────────────────────────────────────────────┘
```

---

## AI Verification Service

The service exposes two verification modes, selectable via `LLM_PROVIDER` in `.env`:

### Mode 1 — Metric Pipeline (`CodeVerifier`)

A deterministic, LLM-free pipeline. Used when reproducibility and speed matter most.

```
Step 1 — Ingest submission
         Accepts a GitHub HTTPS URL (shallow clone via gitpython) or a local .zip file.
         Validates: non-empty, contains .py files, contains a tests/ directory.
         Computes a SHA-256 hash over all .py files (sorted) for tamper-evidence.

Step 2 — Sandboxed pytest run
         Runs one or more pytest commands via subprocess with --tb=short -v.
         Parses verbose output line-by-line to extract per-test PASSED/FAILED/ERROR/SKIPPED.
         Deduplicates tests that appear across multiple commands before counting.
         Enforces a 60-second wall-clock timeout; raises SuiteTimeoutError if exceeded.

Step 3 — Static analysis
         Pylint  — runs on all .py files, extracts "rated at X/10", clamps negatives to 0.
         Flake8  — counts violation lines; line-length tolerance set to 120 chars.

Step 4 — Weighted score calculation
         Final Score = (pass_rate × 0.60) + (pylint/10 × 0.25) + (flake8_score × 0.15)
         Flake8 score = 1.0 − min(violations / 100, 1.0)

Step 5 — Verdict
         ≥ 0.75  →  APPROVED   (oracle releases payment)
         0.45–0.74 → DISPUTED  (jury triggered)
         < 0.45  →  REJECTED   (freelancer may resubmit)

Step 6 — Explainability bundle
         Returns a structured JSON dict with individual test outcomes, pylint/flake8
         raw scores, weighted score breakdown, submission hash, and ISO timestamp.
         Pinned to IPFS via Pinata for on-chain auditability.
```

### Mode 2 — Agentic LLM Pipeline (`CodeVerificationAgent`)

An LLM-powered pipeline with three specialised sub-agents reasoning over the same tool outputs. Supports **Ollama** (default, `qwen3.5`), **OpenAI** (`gpt-5.4`), and **Anthropic** (`claude-4.6-sonnet`).

```
Step 1 — Ingest submission (same as Mode 1)

Step 2 — Run all tools in parallel (asyncio.gather)
         pytest_tool, pylint_tool, flake8_tool, code_extractor run concurrently.

Step 3 — Sub-agent A: RequirementsAnalyzer
         Reads the SRS acceptance criteria and the extracted source code.
         Returns a JSON array of { requirement, met: bool, evidence } for each criterion.
         Strict: a missing edge-case handler or wrong formula counts as NOT met.

Step 4 — Sub-agent B: TestFailureInterpreter
         Analyses pytest output to explain the root cause of each failure,
         not just the error message. Classifies severity as critical / moderate / minor.

Step 5 — Sub-agent C: VerdictSynthesizer
         Combines requirements coverage, test results, and static analysis into
         a final score and APPROVED / DISPUTED / REJECTED verdict with reasoning.

Step 6 — Validate verdict, apply decision gate, clean up work_dir
```

### Milestone-Scoped Verification

When a SRS document is provided, the `MilestoneResolver` parses the `## Milestone Deliverables` section and returns per-milestone `test_scope` (specific pytest paths) and `acceptance_criteria`. Verification is then scoped to only that milestone's tests and requirements rather than the full repo.

### Scoring Formula

| Component | Weight | Calculation |
|---|---|---|
| Test Pass Rate | **60%** | `passed / total` (after dedup) |
| Pylint Score | **25%** | `raw_score / 10.0` (negatives clamped to 0) |
| Flake8 Score | **15%** | `1.0 − min(violations / 100, 1.0)` |

```
Final Score = (test_pass_rate × 0.60)
            + (pylint_score   × 0.25)
            + (flake8_score   × 0.15)
```

All weights and thresholds are configurable per-job via the `thresholds` parameter without restarting the service.

---

## Tech Stack

| Layer | Technology |
|---|---|
| Smart Contracts | Solidity, Hardhat, OpenZeppelin |
| Blockchain | Ethereum Sepolia Testnet |
| Oracle | Node.js, ethers.js, WebSocket |
| AI Service | Python, FastAPI, Pylint, Flake8, pytest |
| LLM Providers | Ollama (default), OpenAI, Anthropic |
| Storage | IPFS via Pinata |
| Frontend | Next.js, Tailwind CSS, MetaMask |
| Database | SQLite (async), Redis (job queue) |

---

## Project Structure

```
Aegistra/
├── backend/
│   ├── contracts/          # Solidity smart contracts + Hardhat config
│   │   └── src/
│   │       ├── EscrowContract.sol
│   │       ├── DisputeContract.sol
│   │       ├── JuryStaking.sol
│   │       └── EvidenceRegistry.sol
│   ├── oracle/             # Node.js bridge between chain and AI service
│   ├── ai-verification/    # FastAPI scoring engine
│   │   └── app/
│   │       ├── services/
│   │       │   ├── code_verifier.py
│   │       │   ├── document_verifier.py
│   │       │   └── agents/         # Agentic tools (pytest, pylint, flake8)
│   │       └── api/endpoints/
│   └── governance/         # Governance API (FastAPI)
└── frontend/               # Next.js app (Client / Freelancer / Jury views)
```

---

## Getting Started

### Prerequisites

| Tool | Version |
|---|---|
| Node.js | ≥ 18 |
| Python | ≥ 3.11 |
| MetaMask | Browser extension |
| Sepolia ETH | ≥ 0.1 ETH ([faucet](https://sepoliafaucet.com)) |

You will also need free accounts at:
- [Alchemy](https://alchemy.com) — RPC & WebSocket provider
- [Pinata](https://pinata.cloud) — IPFS pinning
- [Etherscan](https://etherscan.io/myapikey) — contract verification

### Installation

```bash
# 1. Clone the repo
git clone https://github.com/sk00301/Aegistra.git
cd Aegistra

# 2. Set up environment variables
cp backend/.env.example backend/.env
# Fill in ALCHEMY_API_KEY, DEPLOYER_PRIVATE_KEY, PINATA keys, ETHERSCAN_API_KEY

# 3. Install contract dependencies
cd backend/contracts && npm install

# 4. Install oracle dependencies
cd ../oracle && npm install

# 5. Install AI verification dependencies
cd ../ai-verification
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt

# 6. Install frontend dependencies
cd ../../frontend && npm install
```

### Deploy Contracts

```bash
cd backend/contracts
npx hardhat compile
npm run deploy        # deploys to Sepolia + auto-syncs addresses to frontend
```

### Run All Services

Open four terminal tabs:

```bash
# Tab 1 — AI Verification Service
cd backend/ai-verification && source .venv/bin/activate
uvicorn main:app --reload --host 0.0.0.0 --port 8000

# Tab 2 — Oracle
cd backend/oracle && node oracle.js

# Tab 3 — Frontend
cd frontend && npm run dev

# Tab 4 — (Optional) Contract tests
cd backend/contracts && npx hardhat test
```

Open **http://localhost:3000** with MetaMask set to Sepolia.

---

## Smart Contract State Machine

```
CREATED → FUNDED → SUBMITTED → VERIFIED  → RELEASED
                             → REJECTED  → (resubmit)
                             → DISPUTED  → RESOLVED
          FUNDED  → REFUNDED  (deadline passed, no submission)
```

---

## Dispute Resolution

When a submission scores between 45–74%, the dispute flow activates:

1. Jury members stake ETH via `JuryStaking.sol` to register
2. A 3-juror panel reviews the evidence (stored on IPFS via `EvidenceRegistry.sol`)
3. Each juror casts a vote on-chain
4. After all votes, anyone can call **Tally Votes & Close** to resolve
5. The majority verdict determines payment outcome; jurors earn rewards

---

## Environment Variables

All backend services share a single `backend/.env` file:

```env
ALCHEMY_API_KEY=
ALCHEMY_RPC_URL=
ALCHEMY_WS_URL=
DEPLOYER_PRIVATE_KEY=
ORACLE_PRIVATE_KEY=
ORACLE_ADDRESS=
PINATA_API_KEY=
PINATA_SECRET_KEY=
ETHERSCAN_API_KEY=

# LLM (choose one: ollama / openai / anthropic)
LLM_PROVIDER=ollama
OLLAMA_BASE_URL=http://localhost:11434
OPENAI_API_KEY=          # optional
ANTHROPIC_API_KEY=       # optional
```

---

## Troubleshooting

| Symptom | Fix |
|---|---|
| "Contracts not initialised" in frontend | Switch MetaMask to Sepolia |
| Oracle: "AI service unreachable" | Start the FastAPI service first (Tab 1) |
| Oracle: "oracle address not authorised" | Redeploy with correct `ORACLE_ADDRESS` in `.env` |
| IPFS upload returns synthetic CID | Add `PINATA_API_KEY` + `PINATA_SECRET_KEY` to `.env` |
| `pip install` fails "externally managed" | Add `--break-system-packages` flag |
| MetaMask "nonce too high" | Settings → Advanced → Clear activity tab |

---
