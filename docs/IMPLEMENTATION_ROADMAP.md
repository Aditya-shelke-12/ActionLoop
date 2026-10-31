# ActionLoop — Implementation Readiness & Project Setup

| Field | Value |
|---|---|
| Document version | 0.1 |
| Status | Draft for team execution |
| Project | ActionLoop — Team Granted |
| Event | MST Blockchain Buildathon 2.0 |
| Purpose | Transition from planning documents to a verified development environment and first working blockchain proof |

> **Engineering decision:** The planning documents are a baseline, not proof that the proposed stack works on MST. The next activity is implementation readiness and a small MST Testnet proof of concept (POC). Do not begin the full Workflow Registry implementation or freeze its ABI until the POC and listed decisions are recorded.

---

## 1. Objective and Exit Criteria

This phase prepares the team to implement ActionLoop in parallel without guessing network capabilities or allowing frontend, backend and blockchain interfaces to diverge.

### 1.1 Exit criteria

This phase is complete when:

- [ ] The team has reviewed the PRD, System Architecture, Smart Contract Specification, Database Schema, API Specification, AI Workflow, Security Plan and Testing Plan.
- [ ] Open decisions that affect contract/API interfaces have explicit decisions and owners.
- [ ] A public or private Git repository is created and the initial project structure is committed.
- [ ] Each developer can run the agreed local development tools.
- [ ] MST Testnet RPC connectivity is verified using the published chain ID.
- [ ] A minimal Solidity contract is deployed to MST Testnet.
- [ ] A state-changing Testnet transaction is confirmed and independently visible in MSTScan.
- [ ] BridgeKey connection and signing compatibility are tested and recorded.
- [ ] EIP-712 signature creation and on-chain recovery are tested separately before the relayer design is finalized.
- [ ] The team has written a short POC report containing real addresses, transaction hashes, tested versions and known limitations.

A local Hardhat transaction, screenshot of a wallet, or mocked transaction is not evidence of MST Testnet deployment.

## 2. Source Documents and Change Control

Use the current project documents as the engineering baseline:

| Document | Current baseline |
|---|---|
| Product Requirements Document | v0.1 draft |
| System Architecture | v0.2 draft |
| Smart Contract Specification | v0.1 draft |
| Database Schema | v0.1 draft |
| API Specification | v0.1 draft |
| AI Workflow Specification | v0.1 draft |
| Security Plan | v0.1 draft |
| Testing Plan | v0.1 draft |

Before implementation, team members must review the documents together. Do not assume that a document is approved merely because it exists.

### 2.1 Decision log

Create `docs/decisions/DECISIONS.md`. Each decision must include:

- ID and title.
- Context and options considered.
- Selected option and reason.
- Owner and date.
- Documents/interfaces affected.
- Whether implementation or migration is required.

At minimum, record decisions for:

1. Compiler and EVM target after MST compatibility testing.
2. Whether the official MST TypeScript/Python SDKs are adopted or a standard JSON-RPC library is used behind an adapter.
3. BridgeKey provider behavior and EIP-712 support.
4. Relayer MVP scope (initially approve/reject only, unless the team proves another scope is feasible).
5. Standard and senior approval threshold and role policy.
6. Confirmation depth/finality behavior.
7. Whether a reviewer may approve a case they reviewed (recommended: no).
8. Handling of hard-block/duplicate cases (they must not be cleared by a generic review confirmation).
9. Manual-entry path after extraction failure.
10. Hosting approach for the long-running worker, indexer and relayer.

Do not fill unknown network or SDK behavior with assumptions. Record the test result or mark it unresolved.

## 3. Confirmed MST Network Configuration

Use these values from the MST Buildathon guide. Keep them in environment-specific configuration, not scattered through source code.

### MST Testnet

| Setting | Value |
|---|---|
| Network name | MST Testnet |
| Chain ID | `91562037` |
| RPC URL | `https://testnetrpc.mstblockchain.com` |
| Native symbol | `tMSTC` |
| Explorer base URL | `https://testnet.mstscan.com` |
| Faucet | `https://faucet.mstblockchain.com` |

### MST Mainnet

| Setting | Value |
|---|---|
| Network name | MST Mainnet |
| Chain ID | `4646` |
| RPC URL | `https://mariorpc.mstblockchain.com/` |
| Native symbol | `MSTC` |
| Explorer base URL | `https://mstscan.com` |

**Release rule:** Perform development and acceptance testing on Testnet first. Mainnet deployment is a separate controlled release and must not be used as the compatibility POC.

## 4. Repository Bootstrap

Use one repository for the MVP so the team shares the same contracts, API types, fixtures and documentation.

### 4.1 Initial repository layout

```text
ActionLoop/
├── README.md
├── .gitignore
├── .editorconfig
├── docs/
│   ├── PRD.md
│   ├── SYSTEM_ARCHITECTURE.md
│   ├── SMART_CONTRACT_SPEC.md
│   ├── DATABASE_SCHEMA.md
│   ├── API_SPECIFICATION.md
│   ├── AI_WORKFLOW.md
│   ├── SECURITY_PLAN.md
│   ├── TESTING_PLAN.md
│   └── decisions/
│       └── DECISIONS.md
├── mst-poc/
│   ├── contracts/
│   ├── scripts/
│   ├── test/
│   ├── hardhat.config.ts
│   ├── package.json
│   └── README.md
├── blockchain/
├── relayer/
├── backend/
├── frontend/
├── packages/
│   ├── contract-abi/
│   ├── api-schema/
│   └── test-fixtures/
├── infra/
│   └── docker/
└── .github/
    └── workflows/
```

Create the full structure gradually. Do not create empty modules simply to make the repository look complete. The `mst-poc/` folder is the first implementation target.

### 4.2 Branch strategy

- `main`: reviewed, runnable baseline only.
- `feature/blockchain`: MST POC, contracts, ABI exports and relayer.
- `feature/backend-ai`: FastAPI, database, worker and AI workflow.
- `feature/frontend`: Next.js interface and wallet experience.
- `feature/integration`: optional branch for end-to-end integration.

Rules:

- Use pull requests; do not push unfinished work directly to `main`.
- Every PR states what changed, which tests ran and whether an interface changed.
- A workstream must not silently modify another workstream's owned interface.
- Pin dependency versions and commit lockfiles.
- Never commit `.env`, private keys, service-role keys, API tokens, signatures or real invoices.

## 5. Team Ownership

Proposed workstream ownership, to be confirmed by all three team members:

| Member | Workstream | Initial responsibility |
|---|---|---|
| Tejas Bankar | Blockchain and relayer | MST Testnet POC, BridgeKey compatibility, contract deployment, ABI and later relayer |
| Aditya Shelke | Backend and AI; integration coordinator | FastAPI foundation, data contracts, extraction/validation pipeline, interface review and end-to-end integration |
| Siddhika Hanumante | Frontend and UX | Next.js foundation, dashboard shell, upload/review/approval/verification screens and wallet UI integration |

The initial work can proceed in parallel only after shared interfaces are agreed. During the POC, backend and frontend developers can prepare mock-based shells, but they must not invent ABI or API shapes that conflict with the specification.

## 6. Development Environment Checklist

Each developer should install and record versions for:

- Git.
- Node.js and npm (select one supported Node LTS version and pin it in `.nvmrc`).
- Python (select one version compatible with project dependencies and pin it in `.python-version` or the team's chosen version manager).
- A Solidity development toolchain through the selected Hardhat version.
- Docker Desktop or an agreed local PostgreSQL setup for later phases.
- VS Code or another editor with Solidity, Python and TypeScript support.
- BridgeKey browser extension for wallet testing.

Do not mix package managers within a workstream. Commit `package-lock.json` for npm and the chosen Python lock/dependency file for the backend.

### 6.1 Environment file policy

Commit `.env.example` files containing names and safe placeholders only. Each developer maintains an ignored local `.env`.

Example root-independent blockchain POC configuration:

```dotenv
# mst-poc/.env.example
MST_TESTNET_RPC_URL=https://testnetrpc.mstblockchain.com
MST_TESTNET_CHAIN_ID=91562037
MST_TESTNET_EXPLORER_URL=https://testnet.mstscan.com
DEPLOYER_PRIVATE_KEY=
```

The deployer key must be a dedicated test wallet with only the test funds required. Never paste it into an AI chat, commit it, put it in frontend variables, or include it in logs. If a key has been exposed, stop using it and rotate to a new wallet.

For later services, create separate server-side secrets for:

- Supabase URL and server credential.
- LLM API key.
- Validator service key.
- Relayer private key and internal service token.
- Database connection details.

The relayer and validator must use different keys. The relayer receives no contract role.

## 7. MST Testnet POC — First Implementation Task

### 7.1 Purpose

Prove the network and developer-tool compatibility before implementing the full Workflow Registry. The POC should be deliberately small: one contract, one state-changing function, one event, one successful Testnet transaction and a reproducible verification procedure.

### 7.2 POC contract behavior

Create a minimal contract such as `MSTConnectionPOC` with:

- A deployer/owner address recorded at construction.
- A numeric value stored on-chain.
- A restricted function that changes the value.
- An event emitted when the value changes.
- A read function or public state variable.

This is only a connectivity test. It is not the ActionLoop Workflow Registry and must not be presented as the final product contract.

### 7.3 POC execution sequence

1. Initialize the isolated `mst-poc/` project.
2. Install and pin Hardhat, the Solidity compiler configuration and required libraries.
3. Configure the Testnet RPC and chain ID from Section 3.
4. Verify that the RPC responds and reports the expected chain ID.
5. Create a dedicated BridgeKey/deployer test account as appropriate for the chosen deployment flow.
6. Fund the test account using the official MST faucet.
7. Compile the minimal contract locally.
8. Run local deployment and unit tests first.
9. Deploy the contract to MST Testnet.
10. Call the state-changing function using a real Testnet transaction.
11. Wait for a successful receipt and record the transaction hash and block number.
12. Read the contract's changed state from MST Testnet.
13. Open the transaction in MSTScan and independently confirm the result.
14. Test BridgeKey connection, account detection and network switching in a minimal browser page or test harness.
15. Test typed-data signing and signature recovery in an isolated test; record whether BridgeKey and MST support the exact EIP-712 path required by ActionLoop.
16. Test event retrieval using the proposed SDK/adapter.
17. Write the POC findings in `mst-poc/README.md`.

Do not assume a specific SDK method name. Inspect the official documentation and package exports before coding. The official MST TypeScript and Python SDKs should be evaluated, but the ActionLoop architecture should retain a chain-adapter interface so the rest of the application is not coupled to one SDK.

### 7.4 POC acceptance checklist

- [ ] RPC connectivity succeeds.
- [ ] RPC reports Testnet chain ID `91562037`.
- [ ] Contract compiles using the selected compiler settings.
- [ ] Contract deploys to MST Testnet.
- [ ] Deployment transaction has a successful receipt.
- [ ] State-changing transaction has a successful receipt.
- [ ] State read returns the expected changed value.
- [ ] Event is retrievable from the target network.
- [ ] Transaction hash is independently visible in MSTScan.
- [ ] BridgeKey connects to the application/test harness.
- [ ] EIP-712 signing/recovery result is recorded (pass or documented limitation).
- [ ] SDK capability findings and tested versions are documented.
- [ ] No secret or private key is present in the repository.

### 7.5 POC report template

```text
MST Testnet POC Report
Date:
Developer:
Node version:
Hardhat version:
Solidity compiler version:
MST SDK package and version (if evaluated):
Python SDK package and version (if evaluated):
BridgeKey version:

RPC connectivity: PASS / FAIL
Reported chain ID:
Contract compile: PASS / FAIL
Contract address:
Deployment transaction hash:
State-changing transaction hash:
MSTScan verification link:
Read-after-write result:
Event retrieval: PASS / FAIL
BridgeKey connection: PASS / FAIL
EIP-712 signing and recovery: PASS / FAIL / NOT SUPPORTED
Known limitations:
Decision/recommendation:
```

## 8. After the POC: Implementation Sequence

Do not start all modules at once. Follow these gates.

| Milestone | Build | Exit proof |
|---|---|---|
| M0 | MST Testnet POC | Real contract deployment and transaction verified on MSTScan; wallet and signature findings documented |
| M1 | Workflow Registry contract | Roles, state machine, evidence versioning and negative-path tests pass locally |
| M2 | Testnet Workflow Registry | Contract deployed; roles configured; real case lifecycle transactions verified |
| M3 | Backend foundation | FastAPI, database migrations, storage adapter, auth and health checks run |
| M4 | AI pipeline | Sample invoice extraction, Pydantic validation, deterministic rules and planner route work using fixtures and live model |
| M5 | Chain/backend integration | Backend records evidence, indexer reads events and reconciliation is correct |
| M6 | Human review | Warning confirmation and correction/revalidation work; hard-block cases cannot bypass rules |
| M7 | Relayer | Gasless approve/reject works with user signature; relayer has no contract role |
| M8 | Frontend vertical slice | Upload → processing → review → approval → on-chain verification works on Testnet |
| M9 | Test and release | Required scenarios pass; public repository and application are deployed; Mainnet release completed under controlled procedure |

### 8.1 Vertical-slice priority

If time is constrained, complete one end-to-end scenario before adding polish:

1. Submit one synthetic invoice.
2. Register its hash on MST Testnet.
3. Extract fields and validate them.
4. Route a clean invoice to approval.
5. Have an authorized approver sign the decision.
6. Record approval on-chain (relayed if the relayer gate has passed; otherwise direct BridgeKey transaction with gas paid by the signer).
7. Verify the transaction, state and document hash.

Then add one uncertain invoice scenario and one duplicate/hard-block scenario. Do not claim gasless functionality unless the relayer actually submitted the transaction and the contract verified the original user's signature.

## 9. Interface Freeze Rules

The team must freeze interfaces in this order:

1. Contract states, roles, functions, events, custom errors and typed-data fields, after the MST POC.
2. Shared extraction and validation Pydantic schemas and canonicalization rules.
3. Database migrations and queue/outbox models.
4. FastAPI OpenAPI schemas, response envelopes, status values and error codes.
5. Exported ABI and generated frontend API types.

Any change after a freeze must include:

- The reason for the change.
- Compatibility impact.
- Migration or redeployment impact.
- Tests updated.
- Approval from affected workstream owners.

## 10. Engineering Quality Gates

Every feature PR must include:

- A narrow scope and linked requirement/acceptance criterion.
- Input validation and clear error handling.
- Unit tests for business logic.
- Integration tests for external boundaries where practical.
- No hard-coded credentials or network values.
- No mock presented as a real chain transaction.
- No unverified state displayed as confirmed.
- A short note describing how to run and test the feature.

The following are release blockers:

- Unauthorized wallet can approve or reject.
- Submitter can approve their own case.
- Hard validation failure can be cleared through a generic review action.
- Stale or replayed signature can be accepted.
- Relayer has an approval role.
- Sensitive document data is written on-chain.
- UI shows an approval as verified before the configured confirmation condition is met.
- Mainnet contract is deployed before Testnet acceptance and security checks pass.

## 11. First-Day Task Board

| Task | Owner | Output | Blocker/dependency |
|---|---|---|---|
| Create repository and base folders | Aditya / integration coordinator | Git repo, README, branch protection and docs copied | Team agreement |
| Run MST RPC connectivity test | Tejas | Recorded RPC and chain ID check | Network access |
| Build minimal Solidity POC | Tejas | Contract, local tests and Testnet deployment | Toolchain and test funds |
| Test BridgeKey connect/network/sign | Tejas + Siddhika | Wallet compatibility report | Extension and Testnet config |
| Create frontend shell with mock API | Siddhika | Routes and shared layout; no fake blockchain claims | Agree initial route structure |
| Create FastAPI shell and fixtures | Aditya | Health endpoint, Pydantic fixture models and basic tests | Agree Python version and API conventions |
| Create decision log and track blockers | Aditya | `docs/decisions/DECISIONS.md` | Team review |

## 12. Buildathon Release Evidence Checklist

Before submission, ensure the repository and demo contain real evidence:

- [ ] Public GitHub repository is accessible.
- [ ] README explains MST Testnet and Mainnet integration.
- [ ] Testnet contract address is recorded.
- [ ] At least one real Testnet transaction hash is recorded and verifiable.
- [ ] Mainnet contract address is recorded after controlled deployment.
- [ ] Application is deployed and its demonstrated features work.
- [ ] Demo uses synthetic or consented sample invoices.
- [ ] Contract address and transaction links point to the correct MST explorer.
- [ ] Demo clearly distinguishes pending, confirmed and failed transactions.
- [ ] 2–3 minute demo video shows core functionality and real on-chain proof.
- [ ] No secrets, private keys, personal invoice information or private data are present in GitHub.

## 13. Immediate Next Action

**Start with M0 — MST Testnet POC.** The full product build should proceed only after the team has demonstrated a real Testnet deployment, state-changing transaction, read-back and explorer verification. Keep the POC isolated from the production Workflow Registry code so it can be discarded or retained as a reproducible network smoke test without contaminating the product architecture.
