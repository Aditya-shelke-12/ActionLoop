# ActionLoop — Product Requirements Document

**Version:** 0.2 (Draft for team review) · **Date:** October 31, 2026 · **Status:** Phase 0 deliverable · **Event:** MST Blockchain Buildathon — From Idea to MVP

**Tagline:** AI-assisted document processing with blockchain-backed approval and verification.

---

## 1. Document Purpose

This document defines what ActionLoop is, who it is for, what the MVP must do, and how we will know it is done. It is the source of truth for the architecture, smart contract specification, API specification and test plan that follow. Any change to scope, roles, workflow states or on-chain data should be reflected here first.

**Companion documents (to be produced next):** System Architecture, Smart Contract Specification, Database Schema, API Specification, AI Workflow, Security Plan, Testing Plan.

---

## 2. Product Vision and Summary

### 2.1 Vision

Organizations lose significant time on repetitive document handling, and they struggle to prove afterwards that an approval was genuine, authorized and unaltered. ActionLoop removes the repetitive work with AI while making the decision itself tamper-evident and independently verifiable through a smart contract.

### 2.2 Product statement

ActionLoop is a multi-stage workflow platform in which:

- **AI handles intelligence:** reading documents and extracting structured data.
- **Deterministic rules handle verification:** validating the extracted data against policy and records.
- **Humans handle authority:** reviewing uncertain cases and making the final decision.
- **Smart contracts handle authorization and state:** enforcing who may approve, in which state, and recording the outcome immutably.

Neither technology pretends to do the other's job. AI never authorizes a sensitive action, and the blockchain never claims that a document's content is factually true.

### 2.3 Design principle: blockchain is part of the workflow

Blockchain is not an optional audit log attached after the fact. The smart contract is the authoritative state machine for every case. A case cannot be approved unless the contract accepts the transition, and the approval is only shown as verified after the transaction is confirmed.

---

## 3. Problem Statement

In a typical college department, an equipment invoice follows a manual path:

1. Someone opens the PDF and re-types vendor, amount and tax details.
2. Someone checks the arithmetic and whether the invoice was already submitted.
3. The invoice is forwarded by email or paper to an approving authority.
4. The decision is recorded in a spreadsheet, register or email thread.

This creates four concrete problems:

| Problem | Impact |
| --- | --- |
| Manual data entry and checking | Slow processing, typing errors, inconsistent checks |
| Duplicate or inflated invoices slip through | Financial leakage |
| Approvals live in editable records | A database administrator or clerk can alter history; approvals are hard to prove |
| No single verifiable trail | Audits are slow and depend on trusting whoever holds the records |

---

## 4. Goals, Non-Goals and Success Metrics

### 4.1 Goals

- **G1.** Reduce the manual effort of processing an invoice from upload to approval-ready.
- **G2.** Ensure every approval or rejection is enforced by a smart contract, attributed to an authorized wallet, and independently verifiable.
- **G3.** Keep a human in control of every sensitive decision and every uncertain case.
- **G4.** Let approvers act without holding gas tokens, through a relayer, without giving the relayer approval authority.
- **G5.** Deliver one complete, publicly deployed vertical slice rather than many partial features.

### 4.2 Non-goals (MVP)

- Supporting every sector (hospitals, businesses, government) at once. The MVP covers **college invoice and reimbursement approval** only.
- Fully autonomous agents that execute actions without human approval.
- Actual payment execution or integration with banking or ERP systems.
- Proving that a document or an AI extraction is factually correct. A hash proves integrity, not truth.
- Storing invoices or personal data on a public blockchain.
- Mobile native applications.

### 4.3 Success metrics

| Metric | Target for MVP demo |
| --- | --- |
| End-to-end flow completes on deployed infrastructure | 100% of the 5 test invoice scenarios behave as specified |
| Extraction accuracy on clean sample invoices (key fields: vendor, number, date, subtotal, tax, total) | At least 90% of fields correct |
| Invalid state transitions rejected by contract | 100% of tested cases |
| Unauthorized approvals rejected by contract | 100% of tested cases |
| Approval without the approver paying gas | Works end to end via relayer |
| "Verify on Blockchain" result matches recomputed document hash | Works for every approved case |
| Time from upload to approval-ready (valid invoice) | Under 60 seconds, excluding human decision time |

These are demo-scale targets and should be re-measured and recorded in the Testing Plan.

---

## 5. Users and Roles

| Persona | Description | Key needs | Contract role |
| --- | --- | --- | --- |
| **Department Staff (Submitter)** | Uploads invoices on behalf of a department | Fast upload, clear status, minimal retyping | SUBMITTER |
| **Reviewer** | Resolves flagged or low-confidence extractions | See source evidence next to extracted values; correct or flag | REVIEWER |
| **Authorized Approver** | Faculty head, HOD, finance officer who decides | Complete case view; one-click signed decision; no gas handling | APPROVER |
| **Senior Approver** | Approves high-value invoices above a threshold | Same as approver, restricted to escalated cases | APPROVER (senior tier) |
| **Organization Administrator** | Configures users, roles, thresholds, vendors | Manage permissions safely | ADMIN |
| **Auditor** | Verifies history independently | Tamper-evident trail; recompute and compare hashes | None required (read-only) |

**Separation of duties:** a wallet that submitted a case may not approve that same case. This is enforced on-chain.

---

## 6. Primary Use Case: College Invoice and Reimbursement Approval

A college department purchases laboratory equipment and receives an invoice. ActionLoop:

1. Accepts the uploaded invoice (PDF or image).
2. Extracts vendor details, invoice number, date, line totals, tax and GSTIN.
3. Validates the data deterministically and checks for duplicates.
4. Plans the next action based on organizational policy.
5. Routes uncertain cases to a human reviewer, and valid cases to the right approver.
6. Records the case lifecycle and the approver's signed decision through a smart contract.
7. Lets anyone with access verify the evidence against the chain.

### 6.1 End-to-end workflow

```text
Staff uploads invoice
        |
        v
Document hash generated
        |
        v
createCase(caseId, documentHash) on-chain (SUBMITTED)
        |
        v
Extractor Agent (vision LLM, structured output)
        |
        v
recordExtraction(dataHash, version 1) on-chain (EXTRACTED)
        |
        v
Validator (deterministic rules + database checks)
        |
        v
recordValidation(validationHash, dataVersion) on-chain (VALIDATED)
        |
        v
Action Planner (policy-based routing)
     |                              |
  All checks pass             Needs review / flagged
     |                              |
     |                       Reviewer inspects the case
     |                         |                |
     |                   Data is correct   Data is wrong
     |                         |                |
     |              markReviewed()     recordCorrection(dataHash, version N+1)
     |                         |                |
     |                         |        back to EXTRACTED: re-validate
     |                         |        (previous versions are kept)
     +------------+------------+
                  v
           READY_FOR_APPROVAL
                  |
                  v
  Approver signs decision (EIP-712) --> Relayer submits transaction
                  |
                  v
  Contract verifies role, state, signature, nonce, deadline
                  |
           APPROVED or REJECTED (event emitted)
                  |
                  v
  Backend reconciles with chain; timeline shows verified result
```

---

## 7. Scope

### 7.1 In scope (MVP)

- Authentication and role-based access.
- PDF and image upload with validation.
- Extraction of invoice fields with field-level confidence and source evidence.
- Deterministic validation rules and duplicate detection.
- Policy-based action planning with LLM-generated explanations.
- Human review queue with correction capability.
- Approval queue and signed approve/reject actions.
- Workflow Registry smart contract with role-based access and a strict state machine.
- Relayer-sponsored (gasless) approval transactions.
- Case timeline with per-event transaction status.
- "Verify on Blockchain" feature.
- Public deployment (frontend, backend, contract on the designated network).

### 7.2 Out of scope (MVP)

Multi-organization tenancy, payment processing, ERP or accounting integrations, multi-language invoice support, bulk upload, advanced analytics, mobile apps, token economics.

---

## 8. Functional Requirements

Priority key: **P0** = must ship for the demo to count; **P1** = should ship; **P2** = nice to have.

### 8.1 Document intake

| ID | Requirement | Priority |
| --- | --- | --- |
| FR-01 | Users with the SUBMITTER role can upload PDF, JPG and PNG documents. | P0 |
| FR-01a | The system validates file type, file size and basic integrity before processing, and rejects unsafe files with a clear message. | P0 |
| FR-01b | The system computes a SHA-256 hash of the original file at upload and stores it with the document record. | P0 |
| FR-01c | Original files are stored off-chain in private storage; only hashes go on-chain. | P0 |

### 8.2 Extraction (Extractor Agent)

| ID | Requirement | Priority |
| --- | --- | --- |
| FR-02 | A vision-capable LLM extracts structured fields: document type, vendor name, vendor GSTIN, invoice number, invoice date, line items (if present), subtotal, tax, total, currency. | P0 |
| FR-02a | Each field carries its own confidence score and a source reference (page and region or quoted text) where available. | P1 |
| FR-02b | Model output must conform to a Pydantic schema; non-conforming output is retried a bounded number of times and then routed to human review. Once schema-valid, the canonicalized extracted data is hashed and recorded on-chain as data version 1. | P0 |
| FR-02c | Document content is treated as untrusted data. The extraction prompt and pipeline must resist instructions embedded in the document (prompt injection). | P0 |

### 8.3 Validation (Validator)

| ID | Requirement | Priority |
| --- | --- | --- |
| FR-03 | Validation is performed by deterministic Python rules, not by an LLM. | P0 |
| FR-03a | Required fields present (vendor, invoice number, date, total). | P0 |
| FR-03b | Arithmetic check: subtotal + tax equals total within a defined tolerance. | P0 |
| FR-03c | Duplicate detection on vendor + invoice number (and optionally document hash). | P0 |
| FR-03d | GSTIN format validation. | P1 |
| FR-03e | Vendor checked against the registered vendor list; unknown vendors flagged. | P1 |
| FR-03f | Per-field and overall confidence compared with configurable thresholds. | P0 |
| FR-03g | The validation output (pass/fail, rule results, errors) is canonicalized and hashed; the hash is recorded on-chain together with the data version it validated. | P0 |

### 8.4 Action planning (Action Planner)

| ID | Requirement | Priority |
| --- | --- | --- |
| FR-04 | The planner selects the next action from predefined organizational policy. | P0 |
| FR-04a | Policy table: all checks pass → request approval; missing information → human review; low confidence → manual verification; duplicate → flag for investigation; amount above threshold → senior approver; validation failure → return for correction. | P0 |
| FR-04b | An LLM may generate a human-readable explanation of the recommendation, but never makes or executes the decision. | P1 |

### 8.5 Human review and approval

| ID | Requirement | Priority |
| --- | --- | --- |
| FR-05 | Uncertain or flagged cases pause the workflow and enter a review queue. | P0 |
| FR-05a | Reviewers can see the original document beside extracted values, correct values, and either confirm the data or submit a correction. | P0 |
| FR-05b | Every correction is logged with user, time, old value and new value. A correction never overwrites earlier evidence: it creates a new data version with its own hash, recorded on-chain and attributed to the reviewer's wallet, and the case is re-validated. | P1 |
| FR-06 | Authorized approvers can approve or reject cases in READY\_FOR\_APPROVAL, with an optional reason. | P0 |
| FR-06a | Approvals above the senior threshold are visible only to senior approvers. | P1 |

### 8.6 Blockchain workflow

| ID | Requirement | Priority |
| --- | --- | --- |
| FR-07 | The Workflow Registry contract records case creation, extraction, validation, review, correction, approval and rejection as separate on-chain transactions with events. | P0 |
| FR-07a | The contract enforces roles (ADMIN, SUBMITTER, VALIDATOR, REVIEWER, APPROVER) using OpenZeppelin AccessControl. | P0 |
| FR-07b | The contract enforces the state machine and rejects invalid transitions (see section 9). | P0 |
| FR-07c | A submitter cannot approve or reject their own case. | P0 |
| FR-08 | Users can independently verify a case via a "Verify on Blockchain" action that retrieves the on-chain case and events, recomputes the document hash, compares evidence and displays a pass/fail result. | P0 |
| FR-08a | Events are shown as blockchain-verified only after the transaction is confirmed; pending and failed states are displayed honestly. | P0 |

### 8.7 Relayer

| ID | Requirement | Priority |
| --- | --- | --- |
| FR-09 | Approvers can approve or reject without paying gas; a relayer submits the transaction. | P0 |
| FR-09a | The approver signs a typed message (EIP-712) containing case ID, decision, nonce and deadline; the contract verifies the signature and the signer's role. | P0 |
| FR-09b | Replay protection through per-signer nonces and expiry deadlines. | P0 |
| FR-09c | The relayer can only submit transactions for the registry contract and never holds approval authority. | P0 |
| FR-09d | Relayer gas balance is monitored and failures are surfaced with retry handling that cannot cause duplicate submissions. | P1 |

### 8.8 Dashboard, audit and administration

| ID | Requirement | Priority |
| --- | --- | --- |
| FR-10 | Dashboard shows workflow counts by state, pending approvals and recent activity. | P1 |
| FR-11 | Each case has a timeline of events with timestamps, actors and transaction references. | P0 |
| FR-12 | Administrators can manage users, role mappings, wallet addresses, approval thresholds and the vendor list. | P1 |
| FR-13 | Audit logs record all significant application events (uploads, extractions, corrections, decisions, errors). | P1 |

---

## 9. Workflow State Machine

### 9.1 States

| State | Meaning |
| --- | --- |
| SUBMITTED | Case registered on-chain with case ID and document hash only |
| EXTRACTED | Extracted-data hash recorded for the current data version (version 1, or a later reviewer correction) |
| VALIDATED | Validation result hash recorded for the current data version |
| NEEDS\_REVIEW | Validation or confidence issues require a human reviewer |
| READY\_FOR\_APPROVAL | Validated, or reviewed and confirmed; awaiting an approver |
| APPROVED | Terminal. Approved by an authorized approver |
| REJECTED | Terminal. Rejected by an authorized approver |

### 9.2 Allowed transitions

| From | To | Who can trigger |
| --- | --- | --- |
| (none) | SUBMITTED | SUBMITTER (createCase, document hash only) |
| SUBMITTED | EXTRACTED | VALIDATOR (recordExtraction, data version 1) |
| EXTRACTED | VALIDATED | VALIDATOR (recordValidation for the latest data version) |
| VALIDATED | READY\_FOR\_APPROVAL | VALIDATOR (all checks passed) |
| VALIDATED | NEEDS\_REVIEW | VALIDATOR (flags raised) |
| NEEDS\_REVIEW | READY\_FOR\_APPROVAL | REVIEWER (markReviewed: current data confirmed correct) |
| NEEDS\_REVIEW | EXTRACTED | REVIEWER (recordCorrection: new data version, then re-validation) |
| READY\_FOR\_APPROVAL | APPROVED | APPROVER (not the submitter) |
| READY\_FOR\_APPROVAL | REJECTED | APPROVER (not the submitter) |

### 9.3 Invariants the contract must enforce

- An unauthorized wallet cannot approve or reject.
- A submitter cannot approve or reject their own case.
- A case cannot reach approval without passing through validation.
- A rejected case cannot later be approved, and an approved case cannot be approved or rejected again.
- A case ID cannot be registered twice.
- A signed decision cannot be replayed or used after its deadline.
- Evidence is append-only: a recorded data version or validation hash is never overwritten. A correction adds a new version and keeps every earlier one.
- A validation record must reference the latest data version, so a case cannot become READY\_FOR\_APPROVAL on validation of outdated data.
- Only a wallet with the REVIEWER role can confirm a review or record a correction, so every human review action is attributable to an individual.

---

## 10. AI Components

| Component | Technology | Responsibility | Boundaries |
| --- | --- | --- | --- |
| **Extractor Agent** | Vision-capable LLM, Pydantic schema | Turn a document into structured fields with confidence and evidence | Output is untrusted until validated |
| **Validator** | Deterministic Python | Establish whether data passes rules and checks | No LLM involvement in pass/fail |
| **Action Planner** | Policy rules + optional LLM explanation | Choose next step from predefined policy | Cannot authorize or execute a decision |
| **Orchestrator** | LangGraph | Sequence stages, persist state, pause for human input and resume | Interruption is required before any approval |

**Orchestration graph:** Extraction → Validation → Action Planning → (conditional) Human Review → Blockchain Recording → Approval → Final Audit Sync.

We deliberately use a controlled workflow with specialized components instead of unconstrained autonomous agents, because reliability and auditability matter more than the label.

---

## 11. Blockchain Requirements

### 11.1 Smart contract: Workflow Registry

| Function (conceptual) | Purpose |
| --- | --- |
| createCase | Register a case with case ID and document hash only |
| recordExtraction | Record the extracted-data hash as data version 1 (SUBMITTED to EXTRACTED) |
| recordValidation | Record the validation result hash for the latest data version; move to READY\_FOR\_APPROVAL or NEEDS\_REVIEW |
| markReviewed | REVIEWER confirms the current data version after review; move to READY\_FOR\_APPROVAL |
| recordCorrection | REVIEWER records the corrected-data hash as a new version and a reason hash; case returns to EXTRACTED for re-validation |
| approveCase / rejectCase | Record the decision with approver identity and reason hash |
| approveWithSignature / rejectWithSignature | Relayer-submitted variants verifying the approver's EIP-712 signature |
| getCase | Read case state, current data version and evidence references (earlier versions are available through stored version history and events) |

**Events:** CaseCreated, ExtractionRecorded, ValidationRecorded, CaseReviewed, CorrectionRecorded, CaseApproved, CaseRejected.

### 11.2 What lives where

| Data | Location | Reason |
| --- | --- | --- |
| Original PDF or image | Off-chain private storage | Privacy and storage cost |
| Extracted fields, vendor data | PostgreSQL | Application data |
| Document hash; extracted-data hash and validation-result hash for every data version | On-chain | Evidence integrity |
| Approval or rejection, approver wallet, workflow state | On-chain | Authoritative decision and attribution |
| Full audit details | PostgreSQL, with on-chain references | Searchability plus verifiability |

**Rules:** no invoices, personal information or sensitive business data on a public chain. Hash inputs must be canonicalized (stable field order and formatting) so the same data always yields the same hash. Consider salting hashes of low-entropy data.

### 11.3 Source of truth

PostgreSQL serves fast reads, but for blockchain-backed state the chain is authoritative. The backend reconciles database status with on-chain status, and a database edit must never silently flip an approved case to rejected. Any mismatch is displayed as a discrepancy.

### 11.4 Network dependency

The target network is the MST Blockchain infrastructure for the buildathon. Required before contract work begins: network name, chain ID, RPC endpoint, block explorer, faucet or test tokens, deployment instructions and supported relayer infrastructure. We will not build against an arbitrary EVM testnet.

### 11.5 Relayer approach

Preferred: EIP-712 typed signatures with nonces and deadlines, verified in the contract. Alternative: ERC-2771 meta-transactions if the network supports the required forwarder. Evaluate Gelato or Biconomy for network compatibility, or run an open-source relayer. Do not rely on legacy OpenZeppelin Defender Relayer tutorials, as the hosted service has been retired.

---

## 12. Data Model Overview

| Table | Purpose |
| --- | --- |
| users | Identity, linked wallet address |
| organizations | Organization profile and settings |
| roles | Role mapping per user and organization |
| vendors | Registered vendor records and GSTIN |
| documents | File metadata, storage path, file hash |
| extracted\_data | Fields, per-field confidence, source evidence, version number, previous-version reference, data hash, author (AI or reviewer) |
| validation\_results | Rule results, errors, result hash |
| workflows | Case ID, state, assigned reviewer or approver, chain reference |
| approvals | Human decisions, reasons, signatures |
| blockchain\_transactions | Tx hash, type, status (pending, confirmed, failed), block number |
| audit\_logs | Detailed application events |

The full schema, constraints and indexes will be defined in the Database Schema document.

---

## 13. Key API Surface (summary)

| Endpoint | Purpose |
| --- | --- |
| POST /api/documents/upload | Upload and register a document |
| GET /api/documents/{id} | Document metadata and extraction |
| POST /api/workflows/create | Start a workflow for a document |
| GET /api/workflows, GET /api/workflows/{id} | List and view cases |
| POST /api/workflows/{id}/review | Submit human review |
| POST /api/workflows/{id}/approve, /reject | Submit signed decision for relay |
| GET /api/workflows/{id}/audit | Case timeline |
| GET /api/blockchain/verify/{id} | On-chain verification result |

Request and response schemas must be agreed by all three workstreams before frontend integration.

---

## 14. User Journeys

### 14.1 Journey A: Clean invoice, approved

1. Staff uploads an invoice; the file hash is created and the case is registered on-chain.
2. The Extractor returns fields with high confidence; the Validator passes all checks.
3. The planner routes the case to the correct approver and explains why.
4. The approver inspects the document and extracted data, then signs approval.
5. The relayer submits the transaction; the contract verifies and records approval.
6. The timeline shows each event with confirmed transaction references; the approver wallet, document hash, transaction hash and block are visible.
7. Anyone clicks **Verify on Blockchain** and sees a pass result.

### 14.2 Journey B: Uncertain invoice

1. The extraction has a missing GSTIN and one low-confidence field.
2. The case enters NEEDS\_REVIEW; the reviewer sees the source region beside each value.
3. The reviewer corrects the field; the original extraction (version 1) is preserved and the correction is recorded on-chain as data version 2, attributed to the reviewer's wallet.
4. The Validator re-validates version 2 and records a new validation hash; the case then moves to READY\_FOR\_APPROVAL and continues as in Journey A.

### 14.3 Journey C: Duplicate or invalid invoice

1. The Validator finds a duplicate invoice number for the same vendor.
2. The planner flags it for investigation; it cannot reach approval until resolved or rejected.
3. If rejected, the rejection is recorded on-chain and cannot later be approved.

### 14.4 Journey D: Attempted misuse (demonstration of enforcement)

A submitter tries to approve their own case, or an approver replays an expired signature. The contract rejects the transaction and the UI shows a clear failure state.

---

## 15. Interface Requirements

| Screen | Purpose |
| --- | --- |
| Dashboard | Workflow overview and statistics |
| Document Upload | Upload invoices with progress feedback |
| Processing View | Live extraction and validation progress |
| Document Review | Original document next to extracted fields and confidence |
| Approval Queue | Pending requests filtered by role and threshold |
| Case Details | Full workflow timeline with transaction references |
| Blockchain Verification | Hashes, transaction, block, approver wallet, verification result |
| Settings | Users, roles, thresholds, vendors |

**UX rules:** show Pending, Confirmed and Failed states for every on-chain action; never label an event as blockchain-verified before confirmation; use plain language for errors; make every on-chain event link to the block explorer.

---

## 16. Non-Functional Requirements

| Category | Requirement |
| --- | --- |
| Security | Role-based access control, secure file handling, secrets in server-side configuration only |
| Reliability | Idempotent transaction submission, retry with duplicate protection, graceful degradation if the AI or chain is unavailable |
| Consistency | Database and chain reconciled; discrepancies surfaced, not hidden |
| Performance | Extraction and validation under about 30 seconds per typical invoice; UI responsive during processing |
| Observability | Structured logging, error tracking, transaction status tracking |
| Auditability | Every significant action attributable to a user and time |
| Usability | A first-time approver completes a review and approval without training |
| Maintainability | Modular code, typed interfaces, documented contracts and APIs |
| Portability | Environment-based configuration for network, RPC and relayer |

---

## 17. Security Requirements

- **Authentication and authorization** on every endpoint; server-side role checks in addition to on-chain roles.
- **File upload safety:** type and size limits, content-type verification, malware-conscious handling, no execution of uploaded content.
- **Prompt injection:** treat document text as data; constrain the extractor to a fixed schema; ignore instructions found in documents; never give the extraction model tools or authority.
- **Key management:** private keys (relayer, deployer) only in server-side secret stores, never in frontend code or public environment variables; relayer wallet funded minimally.
- **Signature safety:** EIP-712 domain separation including chain ID and contract address, per-signer nonces, expiry deadlines.
- **Privacy:** no sensitive or personal data on-chain; use hashes (salted where inputs are guessable); private storage with signed, short-lived URLs.
- **Abuse controls:** API rate limiting, relayer allow-listing of target contract and function, spending limits.
- **Smart contract hygiene:** use audited OpenZeppelin components, follow checks-effects-interactions, run static analysis where feasible, test negative paths.
- **Compliance posture:** acknowledge that invoices may contain personal and tax data; keep retention and deletion policy for off-chain data.

---

## 18. Testing and Acceptance

### 18.1 Test invoices (minimum)

1. Correct invoice.
2. Missing vendor information.
3. Incorrect total.
4. Duplicate invoice.
5. Poor-quality scanned invoice.

### 18.2 Smart contract tests

Authorized and unauthorized submission; successful and failed validation; authorized and unauthorized approval; double approval; approve after rejection; skipping validation; submitter self-approval; signature verification; replay and expired signature rejection; relayer restrictions.

### 18.3 Acceptance criteria (definition of done)

The MVP is complete only when all of the following are true on deployed infrastructure:

1. A user can upload an invoice.
2. AI extracts its information with schema-valid output.
3. The system validates the information with deterministic rules.
4. An uncertain invoice enters human review.
5. A valid invoice enters the approval workflow.
6. An authorized user can approve or reject it.
7. The smart contract enforces authorization and the state transition.
8. The transaction is successfully recorded on the target blockchain, with the approver not paying gas.
9. The frontend displays the transaction and the **Verify on Blockchain** action returns a correct result.
10. A public deployment link is available, with a repository and a 2 to 3 minute demo video.

---

## 19. Delivery Plan

| Phase | Objective | Deliverable |
| --- | --- | --- |
| 0 | Planning and architecture | PRD, System Architecture, Smart Contract Specification; network details confirmed |
| 1 | UI/UX design | Screen designs and timeline component |
| 2 | Smart contracts | Tested and deployed Workflow Registry with explorer link |
| 3 | AI backend | API returning extraction, validation and recommended action |
| 4 | Database and backend integration | Persistent workflows, reconciliation with chain |
| 5 | Relayer | Gasless signed approval verified end to end |
| 6 | Frontend integration | Complete user flow including Verify on Blockchain |
| 7 | Security and testing | Test report across invoice and contract scenarios |
| 8 | Deployment | Public deployment and demo video |

### 19.1 Priorities

**P0:** PRD and architecture, network confirmation, Workflow Registry contract with tests and deployment, blockchain integration, relayer-sponsored approval (the coordinators highlighted it as a bonus evaluation factor), final testing and demo. **P1:** upload, extraction, validation, planning, human approval. **P2:** dashboard polish, advanced audit visualization.

### 19.2 Team workstreams

| Workstream | Responsibility |
| --- | --- |
| Blockchain | Contracts, deployment, wallet integration, relayer |
| Backend and AI | FastAPI, LangGraph, extraction, validation, database |
| Frontend | Next.js interface, upload and approval flows, integration |

One member also owns architecture, API contracts, Git integration and final testing. Schema, API responses and contract interface are agreed jointly before parallel development. Branching: main, with feature/frontend, feature/backend-ai and feature/blockchain; merge only tested work.

---

## 20. Risks and Mitigations

| Risk | Likelihood | Impact | Mitigation |
| --- | --- | --- | --- |
| MST network or relayer details unavailable or different from assumption | Medium | High | Request details from coordinators immediately; keep network config environment-driven |
| Relayer incompatibility with the network | Medium | High | Prefer EIP-712 signature verification inside our contract, which needs no special forwarder; fall back to a self-hosted relayer |
| AI extraction errors on poor scans | High | Medium | Field confidence, deterministic validation, human review path, test on a poor scan early |
| Prompt injection through uploaded documents | Medium | High | Fixed schema, no tool access for the extractor, untrusted-data prompting |
| Chain and database drift | Medium | High | Reconciliation job; chain authoritative; visible discrepancy state |
| Transaction failures or duplicate submissions | Medium | Medium | Idempotency keys, status tracking, bounded retries |
| Scope creep across sectors and features | High | High | Single use case; vertical slice first; strict priority list |
| Integration chaos between three workstreams | Medium | High | Agreed contracts and schemas up front; independent module testing |
| Timeline pressure leading to fake blockchain functionality | Medium | High | Contract-first development; acceptance criteria require real state enforcement |
| LLM API cost or rate limits | Low | Medium | Cache results per document hash; small test set |

---

## 21. Assumptions, Dependencies and Open Questions

### 21.1 Assumptions

- The buildathon provides an EVM-compatible network with a faucet.
- Approvers can use a browser wallet, or an embedded wallet flow can be provided.
- A vision-capable LLM API is available within budget.

### 21.2 Dependencies

MST network details, LLM API access, Supabase project, Vercel and backend hosting, block explorer access.

### 21.3 Open questions

1. Exact MST network, chain ID, RPC, explorer and faucet details?
2. Which relayer infrastructure does the buildathon support or favor?
3. Is wallet-based login required, or can users log in conventionally and link a wallet?
4. What approval threshold should trigger senior approval in the demo?
5. Should vendor data be seeded, and from what source?
6. What is the hash salting and canonicalization format we standardize on?
7. What retention policy applies to uploaded documents in the prototype?

---

## 22. Future Scope

Additional document types (purchase orders, leave requests, certificates); additional sectors (hospitals, offices, businesses); multi-organization tenancy; ERP and payment integration; multi-signature or tiered approval chains; zero-knowledge proofs for selective disclosure; analytics and anomaly detection; multilingual document support; account abstraction wallets for frictionless onboarding.

---

## 23. Glossary

| Term | Definition |
| --- | --- |
| Relayer | A service that submits a user's signed transaction and pays its gas fee |
| EIP-712 | Standard for signing structured, human-readable data |
| ERC-2771 | Standard for meta-transactions via a trusted forwarder |
| Gas | Fee paid to execute a blockchain transaction |
| Document hash | Cryptographic fingerprint proving a file has not changed |
| LangGraph | Framework for stateful, interruptible AI workflows |
| Human-in-the-loop | A workflow pause that requires a person's input before continuing |
| Case | One document's lifecycle in the Workflow Registry |

---

## 24. Sign-off

| Role | Name | Status |
| --- | --- | --- |
| Product and architecture owner |  | Pending |
| Blockchain lead |  | Pending |
| Backend and AI lead |  | Pending |
| Frontend lead |  | Pending |
