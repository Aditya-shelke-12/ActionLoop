# ActionLoop — Testing Plan

| Field | Value |
|---|---|
| Document version | 0.1 — Draft for team review |
| Date | October 31, 2026 |
| Product | ActionLoop — AI-assisted document processing with blockchain-backed approval and verification |
| Event | MST Blockchain Buildathon 2.0 |
| Source documents | PRD; System Architecture v0.2; Smart Contract Specification v0.1; Database Schema v0.1; API Specification v0.1; AI Workflow Specification v0.1; Security Plan v0.1; MST Blockchain Buildathon 2.0 Guide |
| Status | Planning deliverable — not an execution report |

> **Purpose.** This plan defines what must be tested, at which layer, with which test data, by whom, and what evidence is required before ActionLoop can be considered ready for the buildathon submission. It is a test specification; passing results must be recorded separately in test reports and CI logs. No test is considered passed merely because a feature appears to work in the UI.

> **Important distinction.** Local Hardhat tests prove contract logic in a local EVM environment. They do not prove MST compatibility. MST Testnet deployment and real transactions are required to prove network integration. Mainnet deployment is a separate release gate and must not be used as a substitute for Testnet testing.

---

## 1. Testing Objectives

The testing strategy must establish that:

1. A submitter can upload a safe invoice and register a case through a real MST Blockchain transaction.
2. The Extractor returns schema-valid structured information without inventing missing values.
3. The Validator produces deterministic results from a fixed data version, rule-set version, thresholds and reference-data snapshot.
4. The Action Planner selects routes from predefined policy and cannot be overridden by an LLM explanation.
5. Reviewers can resolve permitted warnings or correct data, but cannot bypass hard validation failures.
6. Corrections create append-only data versions and force revalidation.
7. Only an authorized human approver can approve or reject a case through a valid wallet signature and permitted contract transition.
8. A relayer can submit an unchanged signed intent and pay gas without holding approval authority.
9. The indexer and reconciler accurately project confirmed chain events into PostgreSQL and surface discrepancies.
10. The Verify on Blockchain feature independently compares the on-chain commitments with recomputed evidence.
11. The deployed product demonstrates genuine MST Testnet activity and has a deployed MST Mainnet contract before submission, as required by the buildathon guide.
12. Failures, retries, worker restarts and transaction delays do not create duplicate or falsely confirmed actions.

## 2. Scope

### 2.1 In scope

- MST Testnet compatibility POC.
- `WorkflowRegistry` contract roles, state machine, evidence versions, signatures, events and pause behavior.
- BridgeKey connection, network detection/switching and typed-data signing, subject to compatibility verified in the POC.
- Frontend upload, review, approval, case timeline and verification flows.
- FastAPI authentication, authorization, validation, endpoints, idempotency and error mapping.
- PostgreSQL constraints, append-only records, jobs, outbox, indexer events and reconciliation.
- Extractor, deterministic Validator, policy-based Action Planner and LangGraph orchestration.
- Relayer pre-flight checks, signed-intent submission, transaction tracking and gas sponsorship.
- End-to-end integration and deployment smoke tests.
- Mainnet deployment safety checks and a minimal read-only / low-risk smoke test after deployment.

### 2.2 Out of scope for MVP testing

- Real reimbursement or banking/payment execution.
- ERP or accounting integration.
- Multi-organization tenancy beyond the single-organization MVP.
- Mobile native applications.
- Bulk document processing and advanced analytics.
- Production-scale load certification or formal third-party smart contract audit.
- Legal or financial correctness of invoice documents beyond the explicitly implemented rules.

## 3. Quality and Release Principles

1. **No false confirmation:** a submitted transaction is `PENDING`, not `CONFIRMED`; the configured confirmation/finality condition must be met before displaying a blockchain-backed state as verified.
2. **Chain-authoritative decisions:** PostgreSQL is a read model for performance. Confirmed contract state/events are authoritative for approval and rejection.
3. **Negative paths matter:** every permission, workflow transition and signature must have both allowed and denied tests.
4. **Evidence is version-bound:** a decision signature must bind the exact current data version and evidence hash. Any correction invalidates stale approval intents.
5. **AI is untrusted:** model output must pass Pydantic validation and deterministic business rules. AI components cannot access chain or relayer adapters.
6. **Reproducible validation:** the same immutable input, rule version, thresholds and reference snapshot must produce the same canonical output and hash.
7. **Idempotent side effects:** retries must not create duplicate cases, versions, decisions, events or logical chain operations.
8. **Privacy by default:** use synthetic invoices. Never put source documents, personal data, GSTIN, vendor names, amounts or free text on a public chain.
9. **Network claims require network evidence:** MST compatibility is established only by actual MST Testnet transactions and reads, not by local mocks or a local Hardhat node.
10. **A demo path is not enough:** test the clean path and the exception, denial and recovery paths.

## 4. Test Levels and Strategy

| Level | Purpose | Typical tools / method | Required evidence |
|---|---|---|---|
| L0 — Static and architecture checks | Catch unsafe dependencies, secrets, lint/type errors and schema drift | Linters, type checkers, secret scanning, dependency checks, architecture tests | CI run and reports |
| L1 — Unit tests | Validate small pure functions and components | pytest, Vitest/Jest, Hardhat tests | Test results and coverage summary |
| L2 — Component tests | Test one module with adapters mocked or local dependencies | FastAPI TestClient, PostgreSQL test DB, mocked LLM/relayer | Component test report |
| L3 — Contract integration tests | Test Solidity contract on local EVM with real ABI and wallets | Hardhat local network | Contract test report |
| L4 — System integration tests | Test API, worker, DB, storage, LLM adapter, relayer and indexer together | Docker/local stack, controlled fixtures | Integration report and traces |
| L5 — MST Testnet acceptance | Prove actual MST network, wallet, contract, transaction, event and relayer compatibility | MST Testnet, BridgeKey, MSTScan, RPC | Contract address, real transaction hashes, explorer links, signed test evidence |
| L6 — Mainnet release checks | Prove required deployment and safe configuration | MST Mainnet, deployment runbook | Mainnet contract address, deployment transaction and smoke-test report |
| L7 — User/demo acceptance | Confirm a judge can complete and inspect the full workflow | Deployed application and synthetic test cases | Recorded demo, public app and README |

### 4.1 Test pyramid

- Keep the majority of tests at unit and component level; these are fast and repeatable.
- Use local EVM tests for exhaustive smart-contract negative paths before using Testnet.
- Use a small but complete suite on MST Testnet because network transactions and confirmations are slower and use shared test funds.
- Mainnet tests should be minimal and controlled. Do not use real sensitive documents or any payment-related action.

## 5. Environments and Configuration

### 5.1 Environments

| Environment | Purpose | Blockchain | Data |
|---|---|---|---|
| Local | Developer iteration and automated tests | Local Hardhat node | Synthetic fixtures, local PostgreSQL, storage emulator, mock LLM optional |
| Staging | Cross-workstream integration and demo rehearsal | MST Testnet | Dedicated Supabase staging project and synthetic invoices |
| Mainnet release | Required deployment and final configuration validation | MST Mainnet | Synthetic/non-sensitive demo data only; no financial execution |

### 5.2 MST network configuration

Values below are taken from the MST Buildathon 2.0 guide and must be configurable per environment, not hard-coded in business logic.

| Setting | MST Testnet | MST Mainnet |
|---|---|---|
| Chain ID | `91562037` | `4646` |
| RPC URL | `https://testnetrpc.mstblockchain.com` | `https://mariorpc.mstblockchain.com/` |
| Native symbol | `tMSTC` | `MSTC` |
| Explorer | `https://testnet.mstscan.com` | `https://mstscan.com` |
| Faucet | `https://faucet.mstblockchain.com` | Not applicable |

**POC gates still required:** Solidity compiler/EVM target compatibility, OpenZeppelin version compatibility, SDK capability, BridgeKey provider behavior, EIP-712 signing/recovery, event retrieval, confirmation depth and relayer behavior. The network guide provides network endpoints; it does not by itself prove every library or signature feature is compatible.

### 5.3 Test accounts and keys

Create separate test identities for:

- `ADMIN`
- `SUBMITTER_A`
- `SUBMITTER_B`
- `REVIEWER_A`
- `REVIEWER_B`
- `APPROVER_STANDARD`
- `APPROVER_SENIOR`
- `VALIDATOR_SERVICE`
- `RELAYER`
- `UNAUTHORIZED_WALLET`

The relayer must have no contract role. Do not reuse deployer, validator and relayer private keys. Use test-only keys on local and Testnet. Mainnet deployer/admin keys must follow the Security Plan and must not be committed or used by routine application services.

## 6. Test Data and Fixtures

All test documents must be synthetic or otherwise explicitly consented for testing. Store them in `packages/test-fixtures/` with expected structured extraction, validation disposition, expected route and expected contract outcome.

### 6.1 Required invoice fixtures

| ID | Fixture | Expected extraction | Expected validation | Expected route / outcome |
|---|---|---|---|---|
| FX-01 | Correct invoice | Required fields extracted; confidence meets configured thresholds | `PASS`; arithmetic, duplicate and vendor rules pass | Standard or senior approval according to configured policy; may become approval-ready |
| FX-02 | Missing vendor field | Vendor is null or low confidence; no invented value | `CORRECTION_REQUIRED` | Human review; correction or manual entry creates a new version and requires revalidation |
| FX-03 | Incorrect total | Printed total is faithfully extracted | `CORRECTION_REQUIRED` due to arithmetic mismatch | Cannot be cleared by confirmation; corrected version must be revalidated |
| FX-04 | Duplicate invoice | Fields extracted correctly | `HARD_BLOCK` for confirmed duplicate | Investigation; remains blocked; permitted approver may reject from `NEEDS_REVIEW` under contract policy |
| FX-05 | Poor-quality scan | Unreadable or ambiguous fields remain null/low confidence | `REVIEWABLE_WARNING` or `CORRECTION_REQUIRED` based on rules; schema failure after bounded retries follows manual-entry path | Human review; no approval until warning is legitimately confirmed or new data is validated |

### 6.2 Additional test documents

Add cases for: unsupported file extension, MIME/extension mismatch, oversized file, corrupt PDF, unusually large image dimensions, duplicate filename, prompt-injection text, ambiguous date, GSTIN with invalid format, unknown vendor and amount crossing the senior approval threshold.

### 6.3 Golden data

For each fixture, retain:

- Original file and SHA-256 hash.
- Expected extraction JSON, schema version and field confidence expectations.
- Expected deterministic validation result and rule-set version.
- Expected planner route and approval tier.
- Expected data/validation canonicalization and hash test vectors.
- Expected on-chain events and terminal/non-terminal state.

Golden outputs must be reviewed when the model, prompt, schema, policy or rule version changes. Do not silently update expected outputs just to make a failing test pass.

## 7. Smart Contract Test Plan

Run contract tests locally for every pull request that changes Solidity, ABI, roles, typed-data definitions or deployment scripts. The ABI is not frozen until the MST Testnet POC has passed.

### 7.1 Role and access-control tests — `SC-ROLE-*`

- `SC-ROLE-01`: deployer receives the intended initial administrative roles.
- `SC-ROLE-02`: ADMIN can grant each operational role.
- `SC-ROLE-03`: ADMIN can revoke each operational role.
- `SC-ROLE-04`: non-admin cannot grant or revoke roles.
- `SC-ROLE-05`: revoked wallet cannot perform new role-gated actions.
- `SC-ROLE-06`: validator cannot approve or reject without an explicit approver role; the MVP setup does not grant it one.
- `SC-ROLE-07`: relayer has no role and cannot call role-gated workflow functions directly.
- `SC-ROLE-08`: senior approver role is required for a `SENIOR` case; standard-only approver is rejected.

### 7.2 Case registration and extraction tests — `SC-CASE-*`

- `SC-CASE-01`: authorized submitter registers a case with valid non-zero case ID and document hash.
- `SC-CASE-02`: unauthorized wallet cannot register a case.
- `SC-CASE-03`: duplicate case ID reverts.
- `SC-CASE-04`: zero case ID or zero document hash reverts.
- `SC-CASE-05`: validator records extraction only from the permitted state.
- `SC-CASE-06`: extraction from an incorrect state reverts.
- `SC-CASE-07`: extraction failure follows the specified review/manual-entry route and cannot be mistaken for successful extraction.
- `SC-CASE-08`: manual entry after extraction failure creates version 1 only when the case has no prior data version.

### 7.3 Validation, warning and correction tests — `SC-VAL-*`

- `SC-VAL-01`: validator records validation only for the current data version.
- `SC-VAL-02`: the same data version cannot be validated twice unless the contract specification explicitly permits a new validation revision.
- `SC-VAL-03`: `PASS` follows the approved transition to `READY_FOR_APPROVAL`.
- `SC-VAL-04`: `REVIEWABLE_WARNING` routes to `NEEDS_REVIEW`.
- `SC-VAL-05`: `CORRECTION_REQUIRED` routes to `NEEDS_REVIEW`.
- `SC-VAL-06`: `HARD_BLOCK` routes to `NEEDS_REVIEW` and cannot be cleared by generic confirmation.
- `SC-VAL-07`: reviewer may confirm only `REVIEWABLE_WARNING`.
- `SC-VAL-08`: reviewer cannot confirm `CORRECTION_REQUIRED` or `HARD_BLOCK`.
- `SC-VAL-09`: correction appends a new data version; previous version and hash remain unchanged.
- `SC-VAL-10`: corrected data must be revalidated before approval-ready state.
- `SC-VAL-11`: correction version limit is enforced.
- `SC-VAL-12`: reviewer attribution is recorded and the same reviewer cannot approve/reject that case under the separation-of-duties policy.

### 7.4 Approval and terminal-state tests — `SC-APP-*`

- `SC-APP-01`: standard approver can decide a standard-tier case.
- `SC-APP-02`: senior approver can decide a senior-tier case.
- `SC-APP-03`: standard approver cannot decide a senior-tier case.
- `SC-APP-04`: submitter self-approval and self-rejection revert.
- `SC-APP-05`: reviewer/corrector approval and rejection revert when separation of duties is enabled.
- `SC-APP-06`: approval from any state other than `READY_FOR_APPROVAL` reverts.
- `SC-APP-07`: permitted rejection from `NEEDS_REVIEW` follows the explicitly specified tier policy.
- `SC-APP-08`: double approval, approve-after-reject and reject-after-approve revert.
- `SC-APP-09`: decision stores the correct signer, timestamp, tier, current data version and reason hash.
- `SC-APP-10`: terminal cases cannot be reopened or modified.

### 7.5 EIP-712 and relayer-signature tests — `SC-SIG-*`

- `SC-SIG-01`: valid typed-data signature succeeds.
- `SC-SIG-02`: signature from a wallet without the required role reverts.
- `SC-SIG-03`: signature with wrong chain ID/domain reverts.
- `SC-SIG-04`: signature for another verifying contract address reverts.
- `SC-SIG-05`: expired signature reverts.
- `SC-SIG-06`: replayed nonce reverts after first successful use.
- `SC-SIG-07`: stale data version reverts after correction.
- `SC-SIG-08`: stale evidence hash reverts after new evidence is recorded.
- `SC-SIG-09`: approve function cannot execute a reject intent and vice versa.
- `SC-SIG-10`: relayer pays gas but the recovered signer is stored/emitted as approver.
- `SC-SIG-11`: relayer cannot approve without a valid approver signature.
- `SC-SIG-12`: signature recovered on MST Testnet matches the BridgeKey account that signed the typed data.

### 7.6 Pause and invariant tests — `SC-PAUSE-*`

- `SC-PAUSE-01`: ADMIN can pause and unpause.
- `SC-PAUSE-02`: non-admin cannot pause or unpause.
- `SC-PAUSE-03`: intended workflow mutations revert while paused.
- `SC-PAUSE-04`: read-only methods remain available while paused.
- `SC-INV-01`: every reachable state transition matches the approved state machine.
- `SC-INV-02`: no transition reaches `APPROVED` without an authorized human signature.
- `SC-INV-03`: no transition from a terminal state is possible.
- `SC-INV-04`: a case cannot be duplicated or have an old version substituted for the current version.

### 7.7 Contract test methods

- Unit tests for each function and custom error.
- State-transition table tests covering every allowed and forbidden edge.
- Property/fuzz tests for case IDs, hashes, versions, nonces, deadlines and role combinations where feasible.
- Event assertions for exact case ID, data version, actor, hash and decision fields.
- Static analysis and dependency/version review in CI where feasible.

## 8. Backend and API Test Plan

### 8.1 Authentication and authorization — `API-AUTH-*`

- Missing, malformed, expired, wrong-issuer and wrong-audience JWTs are rejected.
- A user cannot access another user's or unauthorized case by changing a UUID.
- Every endpoint checks session, organization, role and case assignment as appropriate.
- Browser-supplied role or organization values are not trusted.
- Wallet-link challenge is single-use, expires, and cannot bind an address already linked to another active user.
- Reviewer, approver, admin and auditor endpoint permissions match the approved role matrix.
- SSE progress streams do not disclose events to unauthorized users.

### 8.2 Document intake — `API-DOC-*`

- Valid PDF, JPG and PNG uploads are accepted within configured limits.
- Unsupported extension, MIME mismatch, unsafe/corrupt files and oversized uploads are rejected with stable error codes.
- API computes SHA-256 from the uploaded bytes, not from the filename or client-provided hash.
- The stored object is private; signed URL access is short-lived and authorized.
- Filename/path traversal and collision attempts cannot overwrite another user's file.
- Upload idempotency prevents duplicate logical workflows on retries.
- Uploading a file and creating the workflow record are failure-safe; orphaned storage/database records are detectable and recoverable.

### 8.3 Workflow and API contract — `API-WF-*`

- Each endpoint matches the OpenAPI request/response schema.
- Invalid workflow state returns a stable conflict/error code.
- Mutating endpoints require idempotency keys where specified.
- Repeated requests return the original logical result rather than duplicating a transaction or data version.
- Optimistic concurrency rejects a review/decision prepared against stale data.
- Asynchronous endpoints return `202 Accepted` with resource/progress references.
- API never reports chain confirmation from transaction submission alone.
- Internal relayer endpoints require service authentication and reject calls from untrusted clients.
- Error responses expose useful codes and correlation IDs without leaking secrets, signatures or document content.

### 8.4 Review, correction and decision API — `API-REV-*`

- Review preparation returns typed data bound to case ID, current version, hashes, nonce and deadline.
- Confirmation endpoint accepts only `REVIEWABLE_WARNING` cases.
- `CORRECTION_REQUIRED` and `HARD_BLOCK` cannot be cleared by confirmation.
- Correction endpoint appends a new version and preserves the previous version.
- Correction always schedules deterministic revalidation.
- Manual-entry endpoint is available only after confirmed extraction failure and only for the initial missing data version.
- Approval prepare endpoint reads the authoritative contract state/evidence before preparing typed data.
- Approval/rejection submission rejects stale, expired, malformed or wrong-wallet signatures.

### 8.5 Verification API — `API-VER-*`

- Verification response reports each check separately: document hash, data hash/version, validation hash, event sequence, signer/role attribution and confirmation status.
- Any mismatch returns a failed result and identifies the failed check.
- API does not return pass solely because the database row says approved.
- Missing storage object, unavailable RPC or incomplete event history returns an explicit inconclusive/unavailable status, not a false pass.

## 9. Database and Storage Test Plan

### 9.1 Integrity and constraints — `DB-INT-*`

- Required foreign keys and unique constraints prevent orphaned or duplicate records.
- Case ID, transaction hash/log index and idempotency key uniqueness rules behave as specified.
- Extraction and validation records are append-only for application roles.
- Audit log update/delete operations are denied to normal application roles.
- Workflow versions increase monotonically and link to the correct prior version.
- An approval record references the exact data version and evidence hash that was signed.

### 9.2 Queue, outbox and jobs — `DB-OUT-*`

- Outbox row is created atomically with the application operation that requires a chain write.
- Worker/relayer sender claims jobs safely when multiple workers run.
- Expired locks are reclaimed after worker crash.
- Repeated job execution with the same idempotency key does not duplicate external side effects.
- Transaction status follows permitted lifecycle transitions.
- Failed deterministic reverts are terminal and are not retried endlessly.
- Pending transactions are periodically rechecked and never marked confirmed without a receipt/event at the configured depth.

### 9.3 Indexer and reconciliation data — `DB-IDX-*`

- Duplicate log ingestion is idempotent using transaction hash and log index.
- Cursor only advances after successfully persisting/applying the corresponding event batch.
- Block hash mismatch triggers reorg handling and event replay.
- A chain/database mismatch sets `DISCREPANCY`, blocks new decision actions and creates an operational alert.
- Confirmed on-chain terminal state supersedes a stale database projection.
- Periodic full re-scan detects missed or corrupted projections.

### 9.4 RLS and storage privacy — `DB-SEC-*`

- Public bucket access is disabled.
- Unauthorized users cannot list or fetch another user's documents.
- Expired signed URLs fail.
- Client-side credentials cannot read privileged tables or access the service key.
- Application database role cannot update/delete append-only evidence tables.
- Logs contain identifiers and error codes only; they do not contain uploaded file bytes, full extracted values, secrets or full signatures.

## 10. AI, Validator and LangGraph Test Plan

### 10.1 Extractor tests — `AI-EXT-*`

- Valid invoice yields schema-conforming output.
- Missing fields are represented as null/unknown, not fabricated.
- Monetary values use the agreed decimal representation and currency.
- Ambiguous dates or figures are flagged rather than guessed.
- Field-level confidence is present where required by the schema.
- Source page/quote/region references are preserved where available.
- Malformed JSON or schema-invalid output is retried only the configured number of times.
- Exhausted retries follow the manual-entry path; the workflow does not stall.
- Prompt-injection instructions embedded in a document do not cause tool calls, alter system behavior or escape the fixed schema.
- Model, prompt, schema version and timestamp provenance are recorded.

### 10.2 Deterministic Validator tests — `AI-VAL-*`

- Missing required field returns `CORRECTION_REQUIRED`.
- Arithmetic mismatch returns `CORRECTION_REQUIRED`.
- Confirmed vendor + invoice-number duplicate returns `HARD_BLOCK`.
- Possible/ambiguous duplicate returns the configured warning disposition.
- Low confidence routes according to configured threshold.
- Invalid GSTIN format and unknown vendor produce the configured rule results.
- Amount crossing configured threshold assigns the senior tier but does not itself mean the invoice data is invalid.
- Same immutable input, rule-set version, thresholds and reference snapshot produce identical output and canonical hash.
- Validator module has no LLM import or LLM call in its dependency graph.

### 10.3 Action Planner tests — `AI-PLAN-*`

- Each validation disposition maps to exactly one permitted route.
- `PASS` routes to approval preparation.
- `REVIEWABLE_WARNING` routes to human review.
- `CORRECTION_REQUIRED` routes to human correction/revalidation.
- `HARD_BLOCK` remains blocked and can only follow the permitted rejection/investigation path.
- Amount tier is taken from deterministic policy output.
- LLM explanation failure does not change the route.
- LLM explanation cannot alter disposition, approval tier, assigned role or action.
- A mismatch between planner recommendation and contract-accepted disposition halts progression and surfaces an error.

### 10.4 Human review and correction tests — `AI-REV-*`

- Reviewer can confirm a `REVIEWABLE_WARNING` only after required evidence review.
- Reviewer cannot confirm `CORRECTION_REQUIRED` or `HARD_BLOCK`.
- Correction stores old value, new value, reviewer, timestamp and reason off-chain.
- Correction creates a new version and leaves prior data/hash untouched.
- New version is revalidated with the current configured rule-set.
- Manual entry can occur only after extraction failure and when no prior data version exists.
- Correction-loop maximum is enforced; exceeding it moves the case to a controlled terminal/manual operational state as specified.
- Reviewer cannot approve/reject the same case under the separation-of-duties policy.

### 10.5 LangGraph durability tests — `WF-GRAPH-*`

- Graph follows the documented topology and conditional routes.
- Checkpoints persist in PostgreSQL and allow resume after worker restart.
- A worker crash after each side-effect boundary does not duplicate contract transactions.
- Graph does not advance because a transaction hash was returned; it waits for the matching confirmed event.
- `await_approval` resumes only after the indexer observes a confirmed `CaseApproved` or `CaseRejected` event.
- Human review interrupt resumes only from a valid reviewer input.
- Retry count and correction loop are bounded.
- Job cancellation, timeout and model-provider unavailability result in visible recoverable states.
- Agent modules cannot import chain or relayer modules; an architecture test enforces this boundary.

## 11. Relayer and Signature Test Plan

### 11.1 Pre-flight and allow-list — `REL-PRE-*`

- Unknown chain ID, contract address, function selector or destination is rejected.
- Malformed calldata and missing fields are rejected.
- Invalid, expired, wrong-domain and wrong-signer signatures are rejected.
- Nonce mismatch and stale evidence are rejected before broadcast where possible.
- Transaction simulation catches deterministic reverts where supported.
- Gas limit, fee, per-signer rate and global spending ceilings are enforced.
- Low relayer balance stops accepting new submissions and raises an alert.
- API-to-relayer requests require service authentication.

### 11.2 Submission and lifecycle — `REL-TX-*`

- Valid signed intent is relayed without modifying the signed fields.
- Relayer account pays transaction gas.
- Contract-recognized signer remains the user/approver, not the relayer.
- Relayer has no `APPROVER`, `SENIOR_APPROVER` or other business role.
- Duplicate logical submissions with the same idempotency key do not create duplicate decisions.
- Transaction lifecycle tracks submitted, pending, confirmed, failed and replaced states.
- Stuck transactions are replaced only with the same sender nonce and bounded fee escalation.
- Deterministic contract reverts are not retried automatically.

### 11.3 Direct fallback

If gasless signing is unavailable on MST Testnet, a direct user-paid transaction may be used temporarily to test the contract's direct path. It must be clearly labeled as a fallback and must not be presented as successful relayer functionality. The final demo may claim gasless support only after a real relayed transaction succeeds and the approver is shown as signer/decider.

## 12. Event Indexer and Independent Verification Tests

### 12.1 Indexer tests — `IDX-*`

- Every expected contract event is decoded using the ABI version deployed at that address.
- Duplicate logs are ignored idempotently.
- Indexer stores transaction hash, log index, block number, block hash, event name, case ID and decoded fields.
- Cursor survives restart and advances only after persistence succeeds.
- RPC timeout does not advance the cursor incorrectly.
- Reorg detection rewinds to the last canonical checkpoint and replays events.
- Missing events can be detected through periodic direct contract reads/full re-scan.
- Role changes are reconciled against on-chain role state.

### 12.2 Independent verification tests — `VER-*`

For each case, verification must check:

1. On-chain case exists and has the expected chain ID/contract address.
2. Recomputed original-file SHA-256 equals the on-chain document hash.
3. Recomputed canonical extracted-data hash equals the commitment for the decided version.
4. Recomputed validation-result hash equals the validation commitment bound to that data version.
5. Event sequence is legal under the deployed state machine.
6. Approval/rejection signer matches the wallet attributed by the contract event/state.
7. Signer had the required role/tier at decision time, according to the supported role/event evidence model.
8. Transaction receipt and confirmation depth satisfy the configured finality policy.
9. No stale version was approved after a later correction.

Expected outcomes:

- All checks pass: `VERIFIED`.
- A cryptographic or state-machine check fails: `FAILED` with the failed check named.
- RPC, storage or event history is unavailable: `INCONCLUSIVE` / `UNAVAILABLE`, never a false pass.
- Transaction not yet deep enough: `PENDING_CONFIRMATION`.

Test at least one deliberate mismatch by changing a local copy of a test invoice or the corresponding test evidence, and verify that the application returns `FAILED`.

## 13. End-to-End Acceptance Scenarios

These scenarios must run on the integrated staging environment. At least the clean path and the key exception paths must also be demonstrated on MST Testnet.

### E2E-01 — Clean invoice, standard approval

1. Submitter signs in and links BridgeKey wallet.
2. Submitter uploads `FX-01`.
3. Backend validates the file, stores it privately and calculates the SHA-256 hash.
4. Submitter signs case-registration intent.
5. Relayer (if registration relaying is implemented) or approved transaction path registers the case on MST Testnet.
6. Extractor produces structured fields; Validator returns `PASS`; Planner assigns standard tier.
7. Extraction and validation evidence are recorded on-chain and confirmed.
8. Standard approver opens the evidence-bound decision screen and signs approval.
9. Relayer submits the approval; contract verifies signature, role, state, nonce, deadline and evidence.
10. Indexer observes confirmation; UI reports `APPROVED` only after configured confirmation depth.
11. Verifier clicks Verify on Blockchain and receives `VERIFIED`.

**Expected:** full audit timeline, genuine transaction references, correct approver attribution, matching hashes and no gas paid by the approver when relayer path is claimed.

### E2E-02 — Reviewable warning confirmed

1. Submitter uploads a poor-quality but sufficiently extractable invoice that produces a `REVIEWABLE_WARNING`.
2. Workflow enters `NEEDS_REVIEW`.
3. Reviewer inspects source evidence and confirms the warning.
4. Review action is signed/recorded through the specified contract path.
5. Case reaches `READY_FOR_APPROVAL` only after confirmed chain evidence.
6. Authorized approver signs a decision.

**Expected:** warning confirmation is permitted; no hard failure is bypassed.

### E2E-03 — Correction and revalidation

1. Submitter uploads `FX-02` or `FX-03`.
2. Validator returns `CORRECTION_REQUIRED` and case enters `NEEDS_REVIEW`.
3. Reviewer submits a correction with a reason.
4. New data version is appended; old version/hash remains queryable.
5. Validator runs again against the new version.
6. Only after a passing validation can the case become approval-ready.
7. Any previously prepared approval signature is rejected as stale.

**Expected:** append-only history, new evidence hash, fresh validation and rejection of stale approval intent.

### E2E-04 — Duplicate invoice hard block

1. Submitter uploads `FX-04` or a duplicate of a previously registered test invoice.
2. Validator returns `HARD_BLOCK`.
3. Confirmation/correction-only action cannot clear the block.
4. Authorized approver uses the allowed rejection route from `NEEDS_REVIEW` if supported by the approved contract policy.

**Expected:** no route to `READY_FOR_APPROVAL` from generic confirmation; rejection is recorded as a terminal on-chain state.

### E2E-05 — Extraction failure and manual entry

1. Submitter uploads a deliberately unreadable/corrupt-but-safe test scan that reaches bounded extraction failure rather than unsafe upload rejection.
2. Workflow records extraction failure and enters the specified review/manual-entry path.
3. Reviewer submits manual version 1 with required provenance.
4. Validator runs on version 1.

**Expected:** no fabricated extraction output; manual-entry path is bounded to the permitted initial version; approval is unavailable until validation passes.

### E2E-06 — Unauthorized or stale approval attempt

1. Submitter attempts to approve own case, or reviewer attempts to approve a case they corrected.
2. Alternatively, prepare an approval intent, correct the case, then submit the old signature.
3. Contract rejects the action.

**Expected:** no terminal state change; UI displays the correct authorization or stale-evidence error; no false approval event is indexed.

### E2E-07 — Relayer unavailable / chain unavailable

1. Prepare a valid signed intent.
2. Simulate relayer outage, low balance, RPC timeout or transaction pending beyond threshold.
3. Restore the dependency and allow bounded recovery.

**Expected:** case remains pending or reports a controlled failure; no duplicate logical decision; UI never reports confirmed prematurely.

### E2E-08 — Database tampering / reconciliation discrepancy

1. In a controlled test database, deliberately alter a projected workflow state without a corresponding chain event.
2. Run reconciliation.

**Expected:** discrepancy is identified, approval actions are blocked, confirmed chain state is reapplied to the projection and an incident/audit record is created.

## 14. Reliability, Performance and Usability Tests

### 14.1 Reliability and recovery — `RELIA-*`

- Kill/restart API, worker, indexer and relayer at controlled points.
- Verify queued work is reclaimed and LangGraph resumes from its checkpoint.
- Drop an RPC response after transaction broadcast and verify the outbox reconciles by idempotency key/transaction lookup.
- Simulate duplicate webhooks/log polling and confirm no duplicate events or actions.
- Simulate chain reorganization according to the test network's supported behavior; confirm affected transactions return to pending/non-canonical status until reprocessed.
- Verify service degradation is explicit when LLM, Supabase or RPC is unavailable.

### 14.2 Performance targets

Measure and report, rather than assume, the following demo targets:

| Metric | Target / measurement rule |
|---|---|
| Upload validation and hash calculation | Record p50/p95 for configured maximum demo file size |
| Extraction + deterministic validation | Typical invoice target: about 30 seconds or less, excluding network confirmation and human review |
| Upload to approval-ready | PRD demo target: under 60 seconds excluding human decision time; separately report chain confirmation delay because it depends on MST network finality |
| API response time | Measure p50/p95 for list/detail/prepare endpoints under demo load |
| Queue delay | Measure enqueue-to-worker-start and oldest pending job |
| Indexer lag | Measure head block minus last processed confirmed block and elapsed time |
| Relayer latency | Measure intent accepted to transaction broadcast and to confirmation |
| Verification latency | Measure time to perform on-chain reads and recompute stored evidence hashes |

Do not weaken confirmation requirements to meet the 60-second target. Report processing time and chain confirmation time as separate measures.

### 14.3 Usability checks

- First-time submitter can upload a sample invoice and understand its processing status without developer assistance.
- Reviewer can see the original document beside extracted fields and source evidence.
- Approver sees a human-readable summary of case ID, vendor, amount, decision, data version and evidence commitment before wallet signing.
- User understands Pending, Confirmed, Failed, Discrepancy and Inconclusive states.
- Explorer links open the correct MST transaction.
- Keyboard navigation, labels and basic accessible contrast are checked on upload, review and approval screens.

## 15. MST Testnet POC Test Gate

This is the first external-infrastructure gate and must pass before the Workflow Registry ABI and compiler settings are frozen.

### 15.1 POC checks — `MST-POC-*`

- `MST-POC-01`: connect to MST Testnet using chain ID `91562037` and the supplied RPC.
- `MST-POC-02`: fund a dedicated test wallet using the official faucet.
- `MST-POC-03`: compile a minimal Solidity contract with the selected compiler/EVM target.
- `MST-POC-04`: deploy the contract to MST Testnet.
- `MST-POC-05`: call a state-changing method and capture the real transaction hash/receipt.
- `MST-POC-06`: read back the updated state from MST Testnet RPC.
- `MST-POC-07`: find and verify the transaction on MSTScan.
- `MST-POC-08`: connect BridgeKey, detect account and network, and test the wallet interaction path.
- `MST-POC-09`: test EIP-712 signing and on-chain signature recovery with a minimal isolated contract.
- `MST-POC-10`: retrieve and decode an emitted event.
- `MST-POC-11`: determine a supported confirmation/finality policy and test status transitions.
- `MST-POC-12`: evaluate official MST TypeScript/Python SDK methods for required reads, writes, receipts and logs; record the selected integration adapter and tested versions.

### 15.2 POC pass criteria

The POC passes only when a real MST Testnet contract deployment and state-changing transaction succeed, the read returns the updated state, the receipt is obtained, the transaction is visible on MSTScan, event retrieval works, and the BridgeKey/EIP-712 tests produce a documented outcome. A limitation must be recorded as a limitation; it must not be presented as a successful capability.

### 15.3 POC evidence bundle

Save in `docs/test-evidence/mst-poc/`:

- POC source code and reproducible README.
- Tested compiler, EVM target and dependency versions.
- Contract address and deployment transaction hash.
- State-changing transaction hash and MSTScan link.
- RPC read/receipt output with secrets removed.
- BridgeKey connection and signing result.
- EIP-712 signing/recovery result.
- Event decoding result.
- SDK capability matrix.
- Decision log for ABI/compiler/chain adapter freeze.

No Mainnet deployment is part of this POC.

## 16. CI/CD and Automation Requirements

### 16.1 Pull request checks

Required checks for relevant changes:

- Frontend lint, TypeScript type check and unit tests.
- Backend lint, type checks and pytest unit tests.
- Contract compile, unit tests, state-transition tests and static analysis where available.
- Database migration validation and schema/constraint tests.
- API OpenAPI generation and diff check.
- ABI export and diff check.
- Architecture test: AI agent package cannot import chain or relayer packages.
- Secret scanning and dependency vulnerability scan.
- Formatting and lockfile consistency.

### 16.2 Merge gates

- No merge to `main` with failing required checks.
- Cross-workstream interface changes require review from affected owners.
- Contract ABI, OpenAPI schema, Pydantic extraction schema and database migrations must be updated together when a shared boundary changes.
- Tests must not rely on live LLM calls for deterministic CI; use golden fixtures/mocked provider responses for most automated tests.
- Testnet integration tests run on a controlled schedule or release branch to avoid spending faucet funds on every commit.

### 16.3 Deployment smoke tests

After staging deployment:

1. `/health` returns expected dependency status.
2. Login and wallet-link flows work.
3. Upload a synthetic invoice.
4. Confirm the case appears with correct pending/confirmed state.
5. Complete extraction and deterministic validation.
6. Exercise review or approval path.
7. Confirm transaction receipt and indexed event.
8. Run Verify on Blockchain.
9. Check logs for secrets or document contents.

## 17. Test Ownership

| Workstream | Primary owner | Test ownership |
|---|---|---|
| Blockchain and relayer | Tejas / Blockchain lead | Solidity roles/state/signatures/events, local EVM tests, MST POC, relayer behavior, deployment evidence |
| Backend and AI | Aditya / Backend-AI lead | FastAPI/API tests, database/outbox/indexer integration, extraction, validation, planner, LangGraph durability and verification API |
| Frontend | Siddhika / Frontend lead | UI/component tests, BridgeKey connection/signing UX, API integration, pending/confirmed/failed states, verification screen and accessibility basics |
| Shared integration | All three; architecture owner coordinates | End-to-end scenarios, contract/API/schema review, staging test report, Mainnet release checklist and demo rehearsal |

The names above reflect the proposed workstream allocation and should be confirmed by the team. Each owner must provide evidence for their workstream; the integration owner must not mark a feature complete solely from a teammate's verbal status.

## 18. Defect Severity and Triage

| Severity | Definition | Examples | Release response |
|---|---|---|---|
| S0 — Critical | Funds/key exposure, unauthorized approval, forged terminal state, privacy breach, false verification | Relayer can approve without user signature; secret committed; hard block bypassed | Stop deployment; block release until fixed and retested |
| S1 — High | Core workflow or mandatory blockchain requirement fails | Contract cannot deploy; valid approval fails; invalid transition accepted; no Testnet proof | Block MVP submission until fixed or clearly scope-reduced with coordinator approval where applicable |
| S2 — Medium | Non-critical feature fails but core vertical slice remains usable | Dashboard count wrong; recoverable progress display bug; optional vendor admin issue | Fix before final demo where feasible; document workaround |
| S3 — Low | Cosmetic or minor usability issue | Spacing, wording, minor responsive issue | May defer with team agreement |

Every defect must include: ID, severity, environment, build/commit, reproduction steps, expected result, actual result, logs/transaction reference with secrets removed, owner, fix commit and retest evidence.

## 19. Exit Criteria and Definition of Done

### 19.1 Planning/testing readiness

- [ ] Team approves this Testing Plan and assigns owners.
- [ ] Contract/API/database/Pydantic interfaces are consistent and versioned.
- [ ] MST Testnet POC passes and evidence bundle is saved.
- [ ] Test fixtures and expected outputs are committed.
- [ ] Required CI workflows are active.

### 19.2 Functional MVP acceptance

- [ ] All five required invoice fixtures produce their specified routes and outcomes.
- [ ] Clean invoice completes upload → extraction → validation → planning → approval → chain confirmation → independent verification.
- [ ] Warning confirmation, correction/revalidation, hard-block and manual-entry paths work.
- [ ] Unauthorized, self-approval, stale-signature, replay and invalid-state attempts are rejected.
- [ ] Relayer gas sponsorship is proven by a real signed transaction if relayer functionality is claimed.
- [ ] Indexer and reconciler pass restart, duplicate-event and discrepancy tests.
- [ ] No test or demo flow displays an unconfirmed transaction as verified.

### 19.3 Buildathon delivery acceptance

- [ ] Workflow Registry is deployed on MST Testnet and at least one real Testnet transaction demonstrates core functionality.
- [ ] Testnet transaction hashes and contract address are verifiable on MSTScan.
- [ ] Workflow Registry is deployed on MST Mainnet before submission and its contract address is recorded.
- [ ] Mainnet configuration uses Mainnet RPC/chain ID and is not accidentally connected to Testnet.
- [ ] Public GitHub repository includes smart contracts, frontend, backend, relayer, deployment scripts and README.
- [ ] README documents setup, environment variables (names only, no values), contract addresses, network details, MST integration and demo steps.
- [ ] Public working application/demo link is available where applicable.
- [ ] 2–3 minute demo video shows the real application, actual on-chain interaction and verifiable transaction(s).
- [ ] No private keys, tokens, OAuth secrets, real invoices or sensitive personal/business data are committed or exposed.

## 20. Test Report Template

Create one report per release candidate at `docs/test-evidence/<release>/TEST_REPORT.md` using the following fields:

| Field | Value |
|---|---|
| Release / Git commit |  |
| Environment | Local / MST Testnet / MST Mainnet |
| Chain ID and contract address |  |
| Test date and tester |  |
| Test suite/version |  |
| Total / passed / failed / blocked |  |
| Known defects |  |
| Contract deployment transaction |  |
| Core workflow transaction hashes |  |
| Relayed decision transaction hash (if applicable) |  |
| MSTScan links |  |
| Verification result and evidence |  |
| Performance measurements |  |
| Release recommendation | GO / NO-GO |

A blocked test is not a pass. A skipped test must include a reason and impact statement. Screenshots may support a report, but transaction hashes and contract reads remain the primary proof for blockchain behavior.

## 21. Open Decisions and Test Blockers

The following items must be resolved or explicitly risk-accepted before the relevant test can be considered complete:

| ID | Open item | Required evidence / decision | Blocks |
|---|---|---|---|
| T-01 | Solidity compiler and EVM target | Successful MST Testnet compile/deploy/call POC | ABI freeze and contract deployment |
| T-02 | OpenZeppelin compatibility | Selected version compiles and passes tests on local and MST Testnet | Contract release |
| T-03 | BridgeKey typed-data signing | Signed payload recovered to the correct account on MST Testnet | Gasless decision path |
| T-04 | Official MST SDK capability | Versioned read/write/receipt/log capability matrix | Chain adapter selection |
| T-05 | Confirmation depth/finality | Network-supported policy measured/documented | Verified status behavior |
| T-06 | Senior approval threshold | Team-approved demo policy and policy version | Senior-tier test cases |
| T-07 | Rejection from `NEEDS_REVIEW` | Confirm contract transition and approval tier policy | Duplicate/hard-block closure tests |
| T-08 | Reviewer/approver separation | Confirm contract and API enforce no same-case approval after review/correction | Authorization tests |
| T-09 | Mainnet role/key custody | Named accountable deployer/admin and safe handover process | Mainnet deployment |
| T-10 | Hosting and always-on services | API, worker, indexer and relayer operational in staging | Public end-to-end test |
| T-11 | Test invoice retention | Synthetic data policy, upload size and cleanup procedure | Staging/demo data use |

## 22. Traceability to Source Requirements

| Source requirement / specification area | Tests in this plan |
|---|---|
| PRD document upload, hashing and private storage (FR-01 family) | `API-DOC-*`, `DB-SEC-*`, `VER-*`, `E2E-01` |
| PRD extraction and schema validation (FR-02 family) | `AI-EXT-*`, `E2E-01`, `E2E-05` |
| PRD deterministic validation and evidence hash (FR-03 family) | `AI-VAL-*`, `SC-VAL-*`, `VER-*` |
| PRD policy-based action planning (FR-04 family) | `AI-PLAN-*` |
| PRD human review/correction and approval (FR-05/FR-06 families) | `AI-REV-*`, `SC-APP-*`, `E2E-02` through `E2E-06` |
| PRD on-chain workflow and independent verification (FR-07/FR-08 families) | `SC-*`, `IDX-*`, `VER-*`, `MST-POC-*` |
| PRD relayer (FR-09 family) | `SC-SIG-*`, `REL-*`, `E2E-01`, `E2E-07` |
| Architecture worker/indexer/outbox/reconciliation | `DB-OUT-*`, `DB-IDX-*`, `WF-GRAPH-*`, `RELIA-*`, `E2E-08` |
| Security Plan | Security-focused checks across `API-AUTH-*`, `API-DOC-*`, `AI-EXT-*`, `SC-*`, `REL-*`, `DB-SEC-*`, `VER-*`; detailed threat controls remain in the Security Plan |
| MST Buildathon mandatory delivery requirements | Sections 5, 15, 16, 19.3 and 20 |

## 23. References

### Team project documents

- `ActionLoop_PRD.md`
- `ActionLoop_System_Architecture_v0.2.md`
- `ActionLoop_Smart_Contract_Specification_v0.1.md`
- `ActionLoop_Database_Schema_v0.1.md`
- `ActionLoop_API_Specification_v0.1.md`
- `ActionLoop_AI_Workflow_Specification_v0.1.md`
- `ActionLoop_Security_Plan_v0.1.md`
- `MST BUILDATHON 2.0 GUIDE final.pdf`

### MST resources listed in the buildathon guide

- MST documentation: https://docs.mstblockchain.com
- MST Testnet RPC: https://testnetrpc.mstblockchain.com
- MST Mainnet RPC: https://mariorpc.mstblockchain.com/
- MST Testnet explorer: https://testnet.mstscan.com
- MST explorer: https://mstscan.com
- MST faucet: https://faucet.mstblockchain.com
- BridgeKey: https://bridgekey.io
- MST TypeScript SDK: https://www.npmjs.com/package/@mstblockchain/mst-sdk
- MST Python SDK: https://pypi.org/project/mst-sdk-python/

---

## 24. Version History and Sign-off

| Version | Date | Changes | Status |
|---|---|---|---|
| 0.1 | October 31, 2026 | Initial testing strategy covering POC, unit, integration, Testnet, Mainnet and demo acceptance | Draft for team review |

| Role | Name | Review status | Date |
|---|---|---|---|
| Product / integration owner | Aditya Shelke | Pending |  |
| Blockchain lead | Tejas Bankar | Pending |  |
| Frontend lead | Siddhika Hanumante | Pending |  |

