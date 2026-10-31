# ActionLoop — Security Plan

| Field | Value |
|---|---|
| Document version | 0.1 — draft for team review |
| Date | October 31, 2026 |
| Product | ActionLoop — AI-assisted document processing with blockchain-backed approval and verification |
| Source documents | Product Requirements Document; System Architecture v0.2; Smart Contract Specification v0.1; Database Schema v0.1; API Specification v0.1; AI Workflow Specification v0.1; MST Blockchain Buildathon 2.0 Guide |
| Scope | MVP security controls for the college invoice and reimbursement approval workflow |
| Status | Planning deliverable; controls must be implemented and tested before deployment |

> This document is an engineering security plan, not an independent security audit, legal opinion, or guarantee that the system is secure. The team must implement the controls, run the tests, record exceptions, and review all findings before deploying to MST Mainnet.

---

## 1. Purpose and Security Objectives

ActionLoop accepts documents, extracts financial information with an LLM, validates the information, routes cases for human review, and records approval or rejection through a smart contract on MST Blockchain. It therefore handles untrusted files, potentially sensitive invoice data, application identities, wallet signatures, privileged service credentials, and irreversible public-chain records.

The security plan defines how the MVP protects those assets and prevents unauthorized workflow transitions.

### 1.1 Security objectives

1. **Decision integrity:** Only an authorized human approver can approve or reject a case. The smart contract is authoritative for the on-chain decision.
2. **Evidence integrity:** Document, extracted-data, validation, and decision evidence are versioned and cryptographically bound. A decision must apply to the current evidence version.
3. **Separation of duties:** A submitter cannot approve their own case. A reviewer cannot approve a case they reviewed, according to the proposed MVP policy.
4. **Confidentiality:** Original documents and sensitive invoice fields remain off-chain in private storage and are served only to authorized users.
5. **AI containment:** Document content and model output are untrusted. AI components cannot call tools, access wallets, write to the blockchain, or authorize decisions.
6. **Blockchain correctness:** Only intended contract methods are called; roles, workflow state, signatures, nonces, deadlines, and evidence versions are checked on-chain.
7. **Operational resilience:** Transaction failures, RPC outages, worker restarts, indexer lag, and database/chain disagreement do not produce false approval states.
8. **Credential protection:** Private keys, API keys, service credentials, signing salts, and session tokens are never committed to source control or exposed to the browser.
9. **Auditability:** Important actions are attributable to a user or service identity and can be investigated without exposing secrets or unnecessary personal data.

### 1.2 Security principles

- Deny by default; grant the minimum required permission.
- Verify identity and authorization on every protected request.
- Treat the browser, uploaded documents, LLM output, RPC responses, and all user-provided values as untrusted inputs.
- Keep confidential data off public networks and public blockchains.
- Prefer deterministic checks for authorization, validation, routing, and state transitions.
- Never treat transaction submission as transaction confirmation.
- Preserve evidence versions; do not silently overwrite prior evidence.
- Make failures visible and recoverable; do not silently downgrade security checks.
- Do not claim gasless support, wallet compatibility, or network finality until tested on MST Testnet.

---

## 2. Scope and Security Boundaries

### 2.1 In scope

- Next.js browser application and BridgeKey wallet interaction.
- FastAPI authentication, authorization, document intake, review, approval preparation, and read APIs.
- LangGraph worker, Extractor Agent, deterministic Validator, Action Planner, and workflow checkpoints.
- Supabase Auth, PostgreSQL, and private file storage.
- Workflow Registry smart contract deployed on MST Blockchain.
- Backend validator service identity and private key.
- TypeScript relayer service and dedicated relayer key, if gasless transactions pass compatibility testing.
- Chain indexer, event ingestion, reconciliation, and independent verification.
- Local, MST Testnet, and MST Mainnet deployment environments.
- GitHub repository, dependency management, CI/CD, operational logs, and demo data.

### 2.2 Out of scope for the MVP

- Banking, payment execution, or transfer of funds.
- ERP/accounting integrations.
- Production handling of real invoices without organizational authorization and an approved retention policy.
- Multi-organization tenant isolation beyond the single-organization MVP model.
- Formal smart-contract verification or third-party penetration testing unless separately commissioned.
- A claim that a blockchain hash proves an invoice is genuine or that extracted information is true.

### 2.3 Trust boundaries

| Boundary | Untrusted input or component | Required controls |
|---|---|---|
| Browser → API | User input, browser state, JWT presented by client | TLS, JWT verification, server-side RBAC, schema validation, rate limits |
| Browser → BridgeKey | User-selected account and signature response | Human-readable signing summary, chain/domain checks, signature verification server-side and on-chain |
| Upload → parser/renderer | PDF, JPG, PNG bytes and embedded content | Size limits, MIME sniffing, parser hardening, isolated rendering, no execution |
| Document → LLM | Text and images, including adversarial instructions | Treat as data, fixed schema, no tools, bounded retries, Pydantic validation |
| LLM → validator | Model-generated fields and confidence indicators | Untrusted output; deterministic validation; source evidence; confidence policy |
| API/worker → database | Application queries and mutations | Parameterized queries, least-privilege DB identity, RLS where compatible, audit logging |
| Backend → relayer | Signed intent and request metadata | Authenticated private service channel, fixed schema, idempotency, destination/method allow-list |
| Relayer → contract | Transaction calldata and signer intent | Contract authorization, EIP-712 domain, signer role, nonce, deadline, state and evidence checks |
| Chain → indexer/database | RPC results, logs, reorgs, duplicate events | Verify chain ID and contract address, cursor/block-hash tracking, idempotent event ingestion, reconciliation |
| Runtime → secrets | Private keys and API credentials | Server-side secret store, scoped access, rotation, no logs or client exposure |

---

## 3. Assets and Data Classification

### 3.1 Assets requiring protection

| Asset | Examples | Security requirement |
|---|---|---|
| Original documents | Invoice PDFs and images | Confidentiality, integrity, access control, retention/deletion |
| Extracted invoice data | Vendor, GSTIN, invoice number, dates, amounts | Confidentiality, integrity, version history |
| Validation evidence | Rule outcomes, duplicate checks, thresholds, snapshot IDs | Integrity, reproducibility, attribution |
| Human decisions | Approval/rejection, reason, reviewer corrections | Authorization, attribution, non-repudiation evidence |
| Wallet identity | Linked user address, role and signature | Correct binding, protection against account takeover |
| Contract state | Case state, data version, hashes, decider | On-chain integrity and correct access control |
| Private keys | Validator, relayer, deployer/admin keys | Confidentiality; strict isolation; rotation/recovery |
| Application credentials | Supabase, LLM, RPC, relayer service token | Confidentiality and least privilege |
| Hash salts | Per-case data and validation salts | Confidentiality and controlled disclosure for verification |
| Audit and operational logs | Actor, timestamp, event, case ID, error code | Integrity, minimal disclosure, retention |
| Build artifacts | ABI, contract bytecode, dependency lock files | Provenance and integrity |

### 3.2 Data classification

| Classification | Data examples | Allowed location |
|---|---|---|
| Public | Product name, source code intended for public GitHub, public contract address, public transaction hash | Public website, repository, blockchain |
| Internal | Architecture, test reports, non-secret operational metrics, synthetic demo metadata | Private team workspace/repository as appropriate |
| Confidential | Invoice files, vendor data, GSTIN, amounts, user profile, review notes | Private database/storage; authorized API responses only |
| Restricted secret | Private keys, seed phrases, API keys, Supabase service key, relayer token, signing salts | Secret manager or protected server-side environment only; never browser or repository |

### 3.3 On-chain data minimization

The contract may store only the minimum identifiers and evidence required for workflow enforcement and verification, such as:

- Case identifier (opaque `bytes32` value).
- Document and structured-evidence hashes.
- Current data version and workflow state.
- Role/decision attribution through wallet addresses.
- Approval tier or policy version if required by the contract design.
- Timestamps, nonces, and event metadata.

Never put invoice contents, vendor names, GSTINs, invoice numbers, amounts, free-text reasons, emails, authentication data, or personal information on-chain. Public chain data is permanent and visible. Hashes can also leak information when the possible input space is small; use the specified per-case salt for structured evidence and keep the salt off-chain with restricted access. The plain document hash remains useful for file-integrity verification but may reveal that two parties hold the same file.

---

## 4. Identity, Authentication and Authorization

### 4.1 Identity model

Each human user has:

1. An application identity authenticated through Supabase Auth.
2. A wallet identity linked to the application identity through a signed, expiring challenge.
3. One or more application role assignments.
4. Matching on-chain role grants for actions enforced by the contract.

A wallet address must not be linked to multiple active users. Wallet changes require an administrator-controlled recovery process, revocation of the old address's roles, and a new signed challenge. Account recovery must not silently transfer authority to a new wallet.

### 4.2 Authentication controls

- Use Supabase Auth with a supported login method selected by the team.
- Verify JWT signature, issuer, audience, expiry, and expected claims on every protected API request.
- Do not trust user IDs, organization IDs, role names, wallet addresses, or approval tier values supplied by the browser without server-side lookup.
- Use secure, HTTP-only, same-site cookies where the chosen frontend/auth deployment supports them; otherwise follow Supabase's documented secure session-storage pattern and mitigate XSS.
- Set short, appropriate session lifetimes and support revocation/sign-out.
- Require fresh authentication or re-authentication for sensitive account and wallet-link changes where available.
- Wallet-link challenges must include a cryptographically random nonce, domain, chain/application context, issued time and expiry. Consume each nonce once.
- Verify the wallet signature on the server; never accept a wallet address merely because the browser reports it.

### 4.3 Authorization layers

| Layer | Enforcement | Responsibility |
|---|---|---|
| Session | FastAPI authentication middleware | Confirm a valid application session |
| Application RBAC | FastAPI service layer | Confirm user role, organization, case visibility, assignment, and tier |
| Smart contract | Workflow Registry | Authoritatively enforce contract role, state, submitter separation, signer, nonce, deadline and evidence binding |

Application RBAC improves user experience and data confidentiality. It does not replace contract checks. The contract remains authoritative for blockchain-backed transitions.

### 4.4 MVP permission policy

| Action | SUBMITTER | REVIEWER | APPROVER | SENIOR_APPROVER | ADMIN | AUDITOR |
|---|---:|---:|---:|---:|---:|---:|
| Upload and register own case | Yes | No | No | No | No by default | No |
| View own/assigned cases | Yes | Assigned | Assigned | Assigned tier | Organization scope | Authorized read-only scope |
| Review/correct assigned case | No | Yes | No | No | No by default | No |
| Approve/reject eligible case | No | No | Yes, subject to tier and separation rules | Yes, subject to tier | No | No |
| Manage roles and settings | No | No | No | No | Yes | No |
| Verify evidence | Yes, visible cases | Yes, visible cases | Yes, visible cases | Yes, visible cases | Yes | Yes, read-only scope |

**Separation of duties:** a submitter cannot decide their own case. The proposed MVP also prevents a reviewer from approving a case they reviewed. Enforce both rules in the API and smart contract where the contract has the necessary evidence/role information. If the contract cannot enforce a rule without an unsafe trust assumption, document that limitation and do not imply it is enforced on-chain.

### 4.5 Administrator controls

- Bootstrap the initial ADMIN role through a documented deployment ceremony and a named wallet.
- Do not use one shared administrator wallet among team members.
- Require a second team member to review production role-grant and revoke operations before execution.
- Keep ADMIN authority limited to role management, approved policy/threshold configuration, and emergency pause if that function is implemented.
- ADMIN must not have an override function that directly approves or rejects a case.
- Record every role grant/revocation with actor, subject, reason, timestamp, transaction hash, and resulting chain status.

---

## 5. Application and API Security

### 5.1 API controls

- Enforce HTTPS for deployed environments.
- Verify authentication and authorization on every protected endpoint, including read endpoints and SSE streams.
- Validate all request bodies, query parameters, path identifiers, file metadata, and typed intent payloads with Pydantic.
- Use parameterized SQL or a safe ORM; do not concatenate user-controlled values into SQL.
- Apply CORS only to approved frontend origins. Do not use wildcard origins with credentials.
- Set request body, upload, and processing time limits.
- Rate-limit login/challenge, upload, wallet-link, signature-prepare, signature-submit, and admin endpoints by user and IP.
- Use stable error codes and correlation IDs. Do not return stack traces, SQL details, secret values, signed payloads, or raw provider errors to clients.
- Authorize document access before generating short-lived private storage URLs. Do not expose permanent public file URLs.
- Treat SSE event streams as protected resources; emit only safe metadata and re-check access on connection/reconnection.
- Make mutating operations idempotent using server-generated or validated idempotency keys.
- Ensure approval endpoints cannot accept arbitrary target addresses, arbitrary calldata, or a caller-supplied role claim.

### 5.2 Browser security

- Never place private keys, seed phrases, relayer tokens, backend API secrets, Supabase service-role credentials, or signing salts in client-side code or `NEXT_PUBLIC_*` variables.
- Use output encoding and avoid rendering raw document text as HTML.
- Add a Content Security Policy appropriate for the deployed Next.js application; minimize unsafe inline script exceptions.
- Set secure HTTP response headers, including frame restrictions, MIME sniffing protection, and an appropriate referrer policy.
- Keep dependencies minimal and pinned; review any package that handles wallet connection, file parsing, authentication, or transaction construction.
- Display a readable approval summary before wallet signing: organization, case ID, vendor and amount from the authorized case view, decision, evidence version, deadline, and network/contract context. The values displayed must match the typed message being signed.
- The frontend must not show an on-chain event as confirmed until the configured confirmation policy is met.

### 5.3 File upload and document handling

Accepted formats for the MVP: PDF, JPG and PNG, subject to configured size and page limits.

Controls:

1. Enforce file-size and page-count limits before expensive processing.
2. Check both declared content type and file signature/content sniffing; do not rely only on filename extensions.
3. Generate storage names server-side; do not use user filenames as storage paths.
4. Store documents in a private bucket with access scoped by case and role.
5. Scan or safely inspect files where tooling is available; reject malformed, encrypted when unsupported, or suspicious files with a safe message.
6. Render PDFs/images in an isolated process/container with resource limits; never execute uploaded content or embedded scripts.
7. Protect against decompression bombs, parser vulnerabilities, oversized images, path traversal, and temporary-file leakage.
8. Compute SHA-256 on the original bytes server-side before transformation; store the original file hash and immutable file reference.
9. Apply authorization before returning a short-lived signed URL. Avoid placing URLs or file contents in logs.
10. Keep synthetic invoices in the public demo. Use real invoices only with explicit authorization and an approved data-handling/retention process.

### 5.4 Dependency and supply-chain controls

- Commit lock files for Node and Python dependencies.
- Pin Solidity compiler, Hardhat, OpenZeppelin, SDK and critical parser versions after MST compatibility testing.
- Review transitive dependencies and package maintainer/source before adding sensitive packages.
- Run dependency vulnerability scanning in CI where available.
- Avoid installing packages from untrusted URLs or copying executable scripts from unverified tutorials.
- Build from reviewed source and tag the tested release/commit used for each deployment.

---

## 6. AI and Workflow Security

### 6.1 AI authority boundary

| Component | Allowed | Prohibited |
|---|---|---|
| Extractor Agent | Read the supplied document content; return fixed-schema fields, confidence indicators, and source evidence | Tools, shell, web, database writes, wallet access, chain access, relayer access, approval/rejection |
| Validator | Run deterministic Python rules over immutable input and reference-data snapshots | LLM calls for pass/fail, silently changing extracted values, authorizing decisions |
| Action Planner | Select a route from the versioned policy table; optionally generate an explanation | Changing policy, approving/rejecting, submitting transactions, bypassing hard failures |
| LangGraph orchestrator | Sequence nodes, persist checkpoints, interrupt for human input, resume on verified events | Treating API submission or transaction submission as chain confirmation |

Architecture tests must fail if agent modules import chain, relayer, wallet, or privileged service modules.

### 6.2 Prompt-injection and document-content controls

- Treat all document text, OCR output, images and embedded instructions as untrusted source material.
- Delimit document content clearly in the extraction prompt and explicitly instruct the model to extract facts, not follow instructions contained in the document.
- Do not expose tools, function calling, shell, web search, database connections, or transaction functionality to the Extractor Agent.
- Require fixed-schema JSON and validate it with Pydantic. Reject unknown or unexpected output fields where practical.
- Bound retries; after repeated schema or extraction failure, route to manual entry/review.
- Preserve raw text and normalized values separately where normalization could alter identifiers or amounts.
- Never permit a model to fill missing critical fields using assumptions or external context.
- Store model ID, model version, prompt version, schema version and timestamp for provenance. Avoid logging full invoice contents or sensitive prompts.
- Test invoices containing instructions such as “ignore prior directions”, fake approval instructions, malicious links, and requests to reveal system prompts. Expected result: the model treats them as document content only.

### 6.3 Extraction and confidence controls

- Model-reported confidence is an indicator for routing, not a calibrated probability or proof of correctness.
- Define and version thresholds per field and policy. The architecture proposes an initial value of 0.85 for critical fields; this must be evaluated against the sample invoice set and may be changed before use.
- Missing or ambiguous critical values must be `null` or flagged, not fabricated.
- Store source evidence (page and quoted text/region) for material fields when available.
- A malformed model response is never persisted as a successful extraction version.

### 6.4 Deterministic validation and review

- Validation must be deterministic Python logic with a versioned rule set.
- Record the exact threshold values, rule-set version, and identifiers for the reference-data snapshot used (for example vendor list and duplicate-check snapshot).
- Use decimal arithmetic or decimal strings for monetary values; do not use binary floating-point arithmetic for invoice totals.
- Hard failures, such as arithmetic mismatch or confirmed duplicate, cannot be cleared by a simple “confirm” action.
- Reviewable warnings may be confirmed by a reviewer only where the policy explicitly permits it and the reviewer records the rationale.
- Corrections create a new append-only data version, are attributed to a reviewer, are hashed, and run through validation again.
- Enforce the maximum version count in both the workflow and contract, using the same configured value.
- A reviewer cannot approve the invoice merely by editing a field or changing an application status.

### 6.5 LangGraph workflow safety

- Persist checkpoints in the configured PostgreSQL checkpointer with restricted access.
- Make nodes idempotent and use stable operation keys for all external side effects.
- An approval wait node resumes only after the indexer observes the matching confirmed `CaseApproved` or `CaseRejected` event at the configured confirmation depth.
- A submitted or pending transaction is not a completed action.
- Bound extraction retries, correction loops, and job retries.
- A worker crash must resume from a checkpoint without repeating a confirmed chain write; check chain state and idempotency records before resubmission.
- Do not store private keys, API tokens, raw wallet signatures, or unnecessary document text in graph state/checkpoints.

---

## 7. Smart Contract, Wallet and Blockchain Security

### 7.1 Contract security invariants

The Workflow Registry must enforce, at minimum:

- Only wallets with the required role can perform each state transition.
- Only a SUBMITTER can register a case; a case ID cannot be registered twice.
- A submitter cannot approve or reject their own case.
- A reviewer cannot approve a case they reviewed if this policy is included in the frozen contract design.
- A case must pass the required extraction/validation/review path before `READY_FOR_APPROVAL`.
- Hard validation failure cannot be turned into approval-ready state by a reviewer confirmation alone.
- A decision can only be made from the permitted state and by the correct approval tier.
- APPROVED and REJECTED are terminal states.
- A signed decision binds the case ID, decision, data version, evidence hash, reason hash where applicable, nonce, and deadline.
- Signatures are domain-separated by chain ID and verifying contract address.
- Nonces are per signer and consumed once; expired or replayed signatures fail.
- The relayer address has no approval or review role and cannot originate an approval by itself.
- Only approved hashes/identifiers/enums are accepted; no free text or sensitive document fields are stored.
- Emergency pause, if implemented, must not provide an admin override to fabricate or reverse a decision.

### 7.2 Contract implementation controls

- Use tested OpenZeppelin components for access control, EIP-712 and signature verification; do not implement custom cryptography.
- Pin the exact Solidity compiler and OpenZeppelin versions after the MST Testnet POC.
- Use custom errors for expected reverts; test both successful and unauthorized/invalid paths.
- Keep the MVP non-upgradeable unless the hackathon or network requires otherwise. A logic defect is handled by a reviewed redeployment and explicit app configuration migration.
- Avoid external calls from state-changing workflow functions where possible.
- Review role-admin relationships carefully. No general-purpose “owner can approve” bypass may exist.
- Run static analysis (e.g. Slither where supported) and manually review every finding; record accepted exceptions.
- Do not deploy Mainnet until Testnet acceptance and negative-path tests pass.

### 7.3 BridgeKey and wallet security

- Use BridgeKey as the preferred MST wallet integration, subject to browser and signing compatibility tests.
- Users must confirm the correct MST network and account before signing.
- The wallet must display the actual EIP-712 typed data or transaction being signed. The app must show a human-readable matching summary.
- Verify returned signatures independently; never trust the wallet UI response alone.
- Do not request seed phrases or private keys from users. The application must never collect them.
- A wallet disconnect, account change, chain switch, or stale case version invalidates the pending signing flow; fetch current state and rebuild the intent.
- Do not claim EIP-712 gasless signing until the BridgeKey signature format is accepted by the contract on MST Testnet.
- Provide a direct wallet transaction fallback if the relayer path fails, provided that path is supported and tested. Clearly show that the user may pay gas.

### 7.4 EIP-712 signature controls

The proposed signed decision binds:

- Case identifier.
- Decision enum (approve/reject).
- Current data version.
- Current evidence hash.
- Reason hash (not raw reason text).
- Per-signer nonce.
- Expiry deadline.

The EIP-712 domain must include the intended application name/version, chain ID, and verifying contract address. The contract checks signer role, submitter/reviewer separation, case state, evidence version/hash, nonce, and deadline. A corrected case makes an earlier signature stale and invalid.

Compatibility with MST and BridgeKey is a blocking proof-of-concept gate. If EIP-712 is unsupported or incompatible, do not silently substitute a weaker signature flow; document the limitation and use the tested direct wallet transaction path while the team obtains technical guidance.

### 7.5 Relayer security

The MVP relayer sponsors only the signed decision methods (`approveWithSignature` and `rejectWithSignature`) unless the team explicitly expands scope after the core flow works.

Mandatory controls:

1. Separate relayer service and dedicated low-balance key.
2. Relayer key has no contract role.
3. Allow-list the MST chain ID, Workflow Registry address, and exact approved method selectors.
4. Reject arbitrary destination addresses and arbitrary calldata from callers.
5. Validate payload schema, signature, deadline, nonce and current evidence before submission.
6. Simulate the call where the MST RPC supports it.
7. Enforce per-signer limits, global request limits, gas limit/price ceilings, and daily spending ceiling.
8. Keep a minimum-balance threshold and alert before the service becomes unable to submit.
9. Submit the signed intent unchanged; never replace the signer or decision fields.
10. Use a durable sender nonce queue and idempotency key to prevent duplicate transaction submission.
11. Track `SUBMITTED`, `PENDING`, `CONFIRMED`, `FAILED`, and `REPLACED`; confirmation comes from the indexer, not the relayer response.
12. Authenticate backend-to-relayer calls over a private service channel with a rotated service token.
13. Log action, case ID, transaction hash, status and safe error code; never log private keys or full sensitive payloads.

If EIP-712 or BridgeKey compatibility fails, the relayer feature is not considered complete. The direct-wallet path remains the safe fallback.

### 7.6 MST network and deployment controls

Use the network values supplied in the MST Buildathon guide, but keep them configurable:

| Setting | MST Testnet | MST Mainnet |
|---|---|---|
| Chain ID | `91562037` | `4646` |
| RPC URL | `https://testnetrpc.mstblockchain.com` | `https://mariorpc.mstblockchain.com/` |
| Native currency | `tMSTC` | `MSTC` |
| Explorer | `https://testnet.mstscan.com` | `https://mstscan.com` |

Controls:
- Verify `eth_chainId` (or the corresponding SDK network identity) at startup and before deployment/transaction submission.
- Verify the configured contract address and chain ID together; never reuse a Testnet address as a Mainnet address by assumption.
- Keep separate RPC credentials, deployer/admin, validator and relayer keys per environment.
- Use synthetic data on public Testnet and Mainnet demos.
- Keep deployer keys out of running application containers; remove them after deployment if no longer required.
- Record contract address, compiler version, source commit, deployment transaction hash, role setup transactions, and explorer links.
- Determine the confirmation/finality policy through MST documentation/coordinator guidance and observation; do not guess it.
- Validate contract bytecode/source on the explorer if the explorer supports it; otherwise retain reproducible build artifacts and deployment evidence.
- Mainnet deployment requires a separate two-person review of address, bytecode/build, role setup, RPC, chain ID, pause controls, and initial test transaction.

---

## 8. Database, Storage, Hashing and Audit Security

### 8.1 Database access

- Browser code must not connect directly to PostgreSQL or receive database credentials.
- All business-data access goes through FastAPI authorization and repositories.
- Enable Supabase RLS on organization-owned tables as defense in depth.
- Important caveat: Supabase `service_role` credentials bypass RLS. If used, keep them server-side and enforce authorization in the API; do not claim RLS protects service-role queries. Prefer a restricted `NOBYPASSRLS` role or user-scoped database access where practical.
- Use least-privilege database roles and restrict table access to the operations required by each process.
- Use parameterized queries, reviewed migrations, and explicit constraints/foreign keys.
- Separate API, worker, indexer and relayer credentials where the deployment architecture allows it.
- Restrict updates/deletes on append-only evidence and audit tables. Corrections create new rows, not edits to old evidence.

### 8.2 Storage security

- Private storage bucket only; public bucket access disabled.
- Short-lived signed URLs, generated only after server-side authorization.
- Random server-generated object paths; prevent path traversal and user-controlled path selection.
- Limit access to necessary API/worker identities.
- Define retention and deletion policy for documents and derived content; ensure backup retention is understood.
- Do not place private storage URLs in public GitHub issues, public logs, or demo slides.

### 8.3 Hashing and evidence verification

- Use SHA-256 for original document bytes.
- Use a shared, versioned canonicalization method for structured data and validation evidence (the architecture proposes RFC 8785 canonical JSON encoded as UTF-8).
- Use decimal strings and normalized date/identifier formats as specified by schemas before canonicalization.
- Store canonicalization version, schema version, rule-set version, and relevant reference snapshot identifiers with evidence.
- Include a random per-case salt in structured-data and validation-result hash inputs; store the salt in restricted off-chain storage.
- Keep file hash unsalted for file-integrity comparison, acknowledging that a file hash can identify matching copies.
- Never disclose salt through public APIs or logs. Authorized verification should disclose it only within an access-controlled verification operation, and should avoid returning it to clients unnecessarily.
- Verification must compare recomputed hashes against the correct data version and contract evidence. It must also check event sequence, decision attribution, chain ID, contract address and confirmation policy.
- A matching hash proves that the checked bytes match the committed evidence; it does not prove document authenticity, vendor identity, or AI correctness.

### 8.4 Audit log protection

- Audit records are append-only for application actors; revoke normal update/delete permissions.
- Record actor (user or service identity), event type, case ID, timestamp, correlation ID, and safe metadata.
- Do not log invoice contents, signed payloads, private keys, salts, tokens, or full model prompts.
- Consider hash chaining application log rows as a tamper-detection aid, but do not describe it as a replacement for on-chain decision evidence.
- Restrict audit-log reads by role; auditor access is read-only and scoped.
- Define retention period and backup policy before using non-synthetic data.

---

## 9. Service Secrets and Key Management

### 9.1 Secret inventory

| Secret/key | Used by | Rules |
|---|---|---|
| Supabase service credential / database password | API and approved backend processes | Server-side only; least privilege; never browser or GitHub |
| LLM API key | Worker/extractor adapter | Worker only; quota limits; rotate if exposed |
| Validator private key | Worker chain adapter | Dedicated key; minimal `VALIDATOR` role; managed secret store; never relayer key |
| Relayer private key | Relayer service | Dedicated key; no contract role; low balance; spend limits; isolated from API/worker |
| Deployer key | Deployment operator | Used only for deployment; offline/hardware-backed custody preferred; not mounted in runtime services |
| ADMIN key | Designated administrator | Distinct from deployer and relayer; tightly controlled; documented recovery and rotation |
| Backend-to-relayer token | API/outbox dispatcher | Private service channel, scoped, rotated, never exposed to browser |
| RPC credentials, if any | Chain clients | Server-side configuration; least privilege; rotate if exposed |
| Structured evidence salts | Verification service | Restricted database access; protected backups; not logged or placed on-chain |
| Supabase public client key | Browser | Public by design but must be paired with correct RLS policies and API controls |

### 9.2 Secret handling rules

- Store secrets in the hosting provider's secret manager or protected environment configuration.
- Commit `.env.example` with placeholder names only; ignore `.env`, `.env.*` except approved examples, keystore files, seed phrase files, and local secret exports.
- Scan commits and pull requests for secrets. If a secret is committed, treat it as compromised even if the commit is later deleted.
- Never print secrets in terminal logs, exception messages, CI output, screenshots, or AI coding assistant prompts.
- Do not paste private keys, seed phrases, wallet recovery phrases, access tokens or the coordinator-provided MCP credential into ChatGPT, Antigravity, GitHub, or issue trackers.
- Use different keys for local, Testnet and Mainnet.
- Limit relayer wallet funding and validator role scope; monitor all service-wallet transactions.
- Rotate a key after suspected exposure, team-member departure, unexpected transaction, or compromise; revoke old on-chain roles before granting new keys when possible.

### 9.3 Exposed buildathon credential

The supplied MST guide contains an MCP OAuth client credential in plaintext. Treat it as exposed. Do not copy it into this repository, frontend settings, documentation, or AI prompts. If the team needs this MCP integration, ask the coordinators to confirm whether the credential is intended for shared use or to issue a rotated/team-specific credential. This plan intentionally does not reproduce the credential.

---

## 10. Logging, Monitoring and Incident Response

### 10.1 Security-relevant events to log

- Authentication success/failure and wallet-link attempts.
- Role grant/revocation and administrator configuration changes.
- Upload accepted/rejected, file hash creation, and storage access decisions.
- Extraction and validation completion/failure using safe identifiers and versions.
- Reviewer correction/confirmation and associated evidence version.
- Signed-intent preparation and submission status (never log raw private keys or complete signature payloads).
- Relayer allow-list rejection, nonce/deadline/signature failure, spending threshold, low balance, and transaction replacement.
- Contract deployment and role setup.
- Chain event ingestion, reorg detection, indexer cursor movement, and reconciliation discrepancies.
- Rate-limit triggers, authorization failures, and unexpected service-wallet transactions.

### 10.2 Operational alerts

Configure alerts or visible operator warnings for:

- Repeated authentication failures or wallet-link abuse.
- Upload volume spikes and parser errors.
- LLM error rate, unexpected cost, and retry exhaustion.
- Worker queue depth and oldest job age.
- Relayer low balance, unusual spend, failed transaction rate, and pending transaction age.
- Validator wallet transactions outside expected methods.
- Indexer lag, cursor failure, reorg, or repeated RPC errors.
- Any `DISCREPANCY` state or database claim that differs from confirmed chain state.
- Unexpected role grants, role revocations, or admin-wallet activity.

### 10.3 Incident response procedure

**Step 1 — Detect and record**

Capture time, affected service, environment, case IDs, transaction hashes, correlation IDs, and safe error details. Do not include secrets or document contents in incident notes.

**Step 2 — Contain**

- Disable affected API capability or stop the relevant worker/relayer queue.
- If contract behavior may be unsafe and the deployed contract supports pause, use the documented emergency pause process with the designated ADMIN key.
- Do not pause or revoke roles casually; follow the team-approved authority and record the transaction.
- If a key may be compromised, stop using it and prepare revocation/rotation.

**Step 3 — Preserve evidence**

Retain relevant application logs, chain transaction hashes, block references, deployment commit, and configuration version. Restrict access to incident evidence.

**Step 4 — Assess impact**

Determine whether documents were exposed, a wallet/role was compromised, a signed intent was misused, an on-chain transition occurred, or the database projection diverged from chain state.

**Step 5 — Recover**

Rotate/revoke affected credentials, correct configuration, restore services, reconcile the chain, and redeploy only after review. For an immutable contract logic defect, follow the documented redeployment and address migration process; never pretend the old contract was modified.

**Step 6 — Close and learn**

Document root cause, affected cases, transactions, remediation, regression tests, and remaining risk. Obtain review from at least one team member not responsible for the change.

### 10.4 MVP incident scenarios

| Scenario | Immediate response |
|---|---|
| Relayer key exposed | Stop relayer; verify it has no role; rotate key and service token; inspect spend and transactions |
| Validator key exposed | Stop worker chain writes; revoke validator role; rotate key; inspect extraction/validation events; identify affected cases for re-review |
| ADMIN key exposed | Pause contract if safe/available; revoke compromised role through a secure recovery authority if possible; assess all role changes; redeploy only under reviewed plan |
| Supabase service credential exposed | Rotate credential; review access logs; verify authorization and RLS assumptions; inspect for data export |
| LLM key exposed | Revoke/rotate; inspect usage and billing; remove from logs/prompts/repository |
| Invoice data publicly exposed | Restrict access immediately; determine affected documents/users; preserve evidence; notify relevant organization contacts under applicable requirements |
| Unexpected approval transaction | Verify event and signer; block affected workflow actions; inspect wallet and contract roles; preserve chain evidence and escalate |
| Chain/database mismatch | Mark `DISCREPANCY`, block further approval actions, reconcile from confirmed chain events and investigate cause |

---

## 11. Secure Development and CI/CD

### 11.1 Repository protections

- Make the GitHub repository public only after removing secrets, private invoices, local databases, deployment keystores, and sensitive configuration.
- Protect `main` with pull requests and passing CI checks.
- Require at least one review for smart-contract, authentication, authorization, secret-management, and deployment changes.
- Use separate work branches for frontend, backend/AI, and blockchain/relayer workstreams.
- Do not merge generated code without understanding it and running the associated tests.
- Never commit real invoice samples; use synthetic fixtures.

### 11.2 Required CI checks

| Area | Required check |
|---|---|
| Frontend | Lint, type check, build, dependency scan, browser security checks where available |
| Backend | Formatting/lint, type checks, unit tests, API authorization tests, dependency scan |
| AI | Pydantic schema tests, prompt-injection fixtures, validator golden tests, architecture import-boundary test |
| Database | Migration checks, constraints, RLS/access tests, append-only permission tests |
| Contracts | Compile, unit tests, role tests, state machine tests, signature/replay/stale-evidence tests, static analysis |
| Relayer | Allow-list, signature, nonce, deadline, rate-limit, spend-limit, idempotency and key-isolation tests |
| Integration | End-to-end workflow tests, indexer/reconciliation tests, transaction status tests |
| Secrets | Secret scanning; fail build on verified secret detection |

### 11.3 AI coding assistant rules

AI coding assistants are permitted, but generated code must be reviewed and understood by the team.

- Never provide AI tools with private keys, seed phrases, API secrets, production invoices, or user data.
- Use synthetic fixtures and redacted logs.
- Ask AI to work on one bounded feature at a time and inspect the existing code before editing.
- Require tests for authorization and negative paths, not only happy paths.
- Do not accept generated blockchain code that uses mock transaction hashes or fabricated explorer links.
- Verify SDK methods against official documentation or package types; do not invent methods from model output.
- Review every generated contract, cryptographic operation, database migration, and auth change manually.

---

## 12. Security Testing Plan

Security tests are required at unit, integration, and deployed Testnet levels. Tests should include both allowed and denied actions.

### 12.1 Authentication and API tests

- Missing, invalid, expired, wrong-issuer, and wrong-audience JWTs are rejected.
- A user cannot access another user's or unauthorized case by changing a UUID.
- Each endpoint enforces the expected role and case assignment.
- Wallet-link challenge cannot be reused, is expired after deadline, and cannot bind an address already linked to another active user.
- Role/organization values submitted from the browser are ignored unless confirmed from trusted records.
- CORS, request limits, rate limits and error responses match deployment configuration.
- SSE does not leak event data to unauthorized clients.

### 12.2 File and AI tests

- Unsupported extension, mismatched MIME, oversized file, malformed PDF, oversized image, and suspicious parser input are rejected safely.
- File path traversal and filename collision attempts cannot overwrite another document.
- Document instructions that attempt prompt injection do not cause tool use or schema deviation.
- Missing values remain null; the extractor does not invent amounts, vendor identifiers, or dates.
- Malformed model output fails Pydantic validation and follows bounded retry/manual-entry behavior.
- Low-confidence required fields route to review.
- Hard arithmetic errors and confirmed duplicates cannot be cleared with a confirmation-only action.
- Corrections create new versions and trigger validation again.
- Validator output is deterministic for the same immutable input, rule version, thresholds and reference snapshot.

### 12.3 Database and storage tests

- Unauthorized users cannot read another user's documents or cases.
- Public storage access is disabled.
- Signed URLs expire and cannot be created without authorization.
- Browser roles cannot access database credentials or privileged tables.
- Service-role/RLS assumptions are tested explicitly.
- Append-only records reject update/delete through application roles.
- Secrets, document text and signatures are absent from logs.
- Retention/deletion process behaves as documented for off-chain records.

### 12.4 Smart contract tests

- Unauthorized role attempts revert.
- Duplicate case ID reverts.
- Invalid state transitions revert.
- Validation or review cannot be skipped.
- Hard failure cannot transition to approval-ready state without the permitted correction and revalidation path.
- Submitter self-approval reverts.
- Reviewer/approver separation rule is enforced if included in contract.
- Wrong approval tier reverts.
- Double decision, decision after terminal state, and decision from wrong state revert.
- Invalid signature, wrong domain, wrong chain, wrong verifying contract, wrong nonce, expired deadline, changed data version, and changed evidence hash revert.
- Signature replay fails after nonce consumption.
- Relayer address has no decision role and cannot originate a decision without a valid approver signature.
- Pause behavior blocks only the intended operations and does not introduce an admin decision override.

### 12.5 Relayer tests

- Unknown chain, contract address, selector, malformed calldata and arbitrary destination are rejected.
- Invalid/expired signature and stale nonce are rejected before submission.
- Simulated revert does not get broadcast where simulation is supported.
- Per-signer rate limits, global rate limits, gas ceilings, spending ceiling and low-balance stop work.
- Duplicate idempotency key does not create a second logical decision.
- Relayer sends the signed intent unchanged.
- Relayer key is isolated from validator, admin and deployer keys.
- Transaction state remains pending until indexer confirmation reaches configured depth.

### 12.6 Indexer, reconciliation and verification tests

- Duplicate logs are idempotently ignored using transaction hash and log index.
- Indexer resumes from saved cursor after restart.
- Reorg/block-hash mismatch rewinds and reprocesses affected events.
- RPC failure does not advance the cursor incorrectly.
- Chain/database mismatch sets `DISCREPANCY` and blocks new decisions.
- Verification recomputes the correct file/data/validation version hashes.
- Verification fails if file, salt, data version, validation hash, event order, signer, chain ID, contract address or confirmation status is wrong.
- The UI never reports verified on transaction submission alone.

### 12.7 Deployed Testnet security acceptance

Before Mainnet:

1. Deploy the contract to MST Testnet using the approved compiler/target.
2. Record contract address and deployment transaction hash.
3. Grant roles to test wallets and verify role state.
4. Register at least one synthetic invoice case.
5. Record extraction and validation evidence.
6. Exercise review/correction and revalidation.
7. Submit authorized approval and rejection decisions.
8. Demonstrate unauthorized and replay attempts fail.
9. If relayer is enabled, execute a valid EIP-712 relayed decision and prove relayer has no role.
10. Verify all successful transactions on MSTScan and through the application.
11. Demonstrate hash verification and at least one deliberate mismatch that returns failure.
12. Capture the test report and transaction references for the submission.

---

## 13. Deployment and Environment Security

### 13.1 Environment separation

| Environment | Purpose | Data | Keys |
|---|---|---|---|
| Local | Developer testing | Synthetic fixtures | Local-only test keys; never reused |
| MST Testnet | Integration, demonstration and acceptance | Synthetic invoices only | Dedicated Testnet deployer/admin/validator/relayer keys |
| MST Mainnet | Required public deployment after Testnet acceptance | Synthetic public-demo data only unless separately approved | Separate Mainnet keys; stricter custody and two-person review |

No secret or database data is shared between environments. Testnet configuration must never be copied into Mainnet by replacing only a chain ID.

### 13.2 Deployment gates

**Gate A — Local readiness**

- CI passes.
- Contract tests and backend/frontend tests pass.
- Environment examples contain placeholders only.
- No secrets or real invoices are committed.

**Gate B — MST Testnet deployment**

- Network identity and RPC are verified.
- Compiler/EVM compatibility, BridgeKey signing and SDK behavior are tested.
- Contract source/build artifacts are reproducible.
- Initial ADMIN, VALIDATOR, SUBMITTER, REVIEWER, APPROVER and SENIOR_APPROVER roles are reviewed and granted to intended wallets.
- Relayer, if enabled, has no contract role.
- End-to-end synthetic case and negative-path tests pass.

**Gate C — MST Mainnet deployment**

- Testnet acceptance report is approved by all three team members.
- Mainnet address, chain ID, RPC, explorer and deployment key are independently checked.
- Contract source and bytecode/build configuration are reviewed.
- Role setup and pause/recovery process are reviewed by two people.
- Mainnet relayer funding and spending limits are explicitly approved.
- No real private invoices or personal data are used in the public demo.
- A small, low-risk state-changing transaction is tested and independently verified.

**Gate D — Public application launch**

- HTTPS and domain configuration are active.
- Secrets are configured in hosting secret stores.
- CORS, storage policies, auth, role checks, rate limits and monitoring are enabled.
- Smoke test uses synthetic data.
- Repository README lists the correct contract address, network, setup and real transaction references.

### 13.3 MST deployment reference

The buildathon guide specifies:

- MST Testnet: chain ID `91562037`, RPC `https://testnetrpc.mstblockchain.com`, currency `tMSTC`, explorer `https://testnet.mstscan.com`.
- MST Mainnet: chain ID `4646`, RPC `https://mariorpc.mstblockchain.com/`, currency `MSTC`, explorer `https://mstscan.com`.

Network values must be checked against current official MST documentation before deployment. Chain compatibility, finality depth, SDK behavior, BridgeKey EIP-712 support, and relayer support must be established by the Testnet POC, not assumed from this table.

---

## 14. Privacy, Retention and Demo Data

- Use synthetic invoices for public demos, recordings, GitHub fixtures, screenshots, and judging sessions.
- Do not use a person's tax details, bank account, phone number, personal address, signature, or other personal data in public demo materials.
- Before sending a real invoice to a vision LLM provider, review its data-retention, training-use, region, and deletion terms. If those terms are not acceptable or unknown, use synthetic data or a local alternative.
- Set explicit retention periods for original files, extracted fields, review drafts, logs and backups before production use.
- Delete off-chain data according to the approved retention policy and record the deletion event. On-chain hashes and events cannot be assumed erasable; do not put data on-chain that may later need deletion.
- Restrict access to exported reports and verification outputs because they may reveal vendor and financial details.
- Do not present a public blockchain hash as anonymization by itself; linkability and low-entropy guessing risks remain.

---

## 15. Security Ownership and Team Responsibilities

The workstream owners below are proposed to align with the current three-person plan. The team should confirm names and ownership at sign-off.

| Workstream | Primary security responsibilities | Required peer review |
|---|---|---|
| Blockchain (Tejas) | Contract roles/state machine, EIP-712, relayer, deployment scripts, key isolation, transaction tests | Backend/AI lead reviews authorization semantics; one other member reviews Mainnet role setup |
| Backend and AI (Aditya) | JWT/RBAC, file handling, extraction containment, deterministic validation, LangGraph checkpoints, DB access, chain adapter, reconciliation | Frontend lead reviews API permissions and data exposure; blockchain lead reviews chain adapter and evidence mapping |
| Frontend (Siddhika) | BridgeKey UX, readable signing summaries, route protection, XSS/CSP, safe document display, transaction status and verification UI | Backend lead reviews API calls and authorization; blockchain lead reviews typed-data fields and network handling |
| Whole team | Threat review, interface changes, Testnet acceptance, Mainnet go/no-go, incident response rehearsal | All three approve deployment gates |

No developer should be the sole reviewer of their own smart-contract authorization, secret-handling, or Mainnet deployment changes.

---

## 16. Open Decisions and Security Blockers

The following items remain open and must be explicitly resolved. This plan does not assume undocumented MST features.

| ID | Open item | Required decision/evidence | Blocks |
|---|---|---|---|
| SEC-01 | MST EVM/compiler compatibility | Successful minimal contract compile/deploy/call on Testnet; confirmed EVM target | Contract deployment |
| SEC-02 | BridgeKey EIP-712 signing compatibility | Browser signing POC and contract recovery test | Gasless approval |
| SEC-03 | MST SDK versus standard JSON-RPC adapter | Test required deployment, read/write, receipt, and event operations using official SDK/docs | Chain adapter choice |
| SEC-04 | Confirmation depth/finality and reorg behavior | Official guidance plus observed Testnet behavior | Verified status and indexer policy |
| SEC-05 | Relayer compatibility and funding model | End-to-end Testnet relayed decision; approved spend limits | Relayer launch |
| SEC-06 | Role bootstrap and ADMIN custody | Named owner wallet(s), recovery method, review procedure | Testnet/Mainnet role setup |
| SEC-07 | Approval threshold and policy version | Team-approved synthetic threshold and versioning approach | Tier enforcement |
| SEC-08 | Reviewer/approver separation | Confirm whether contract and application both enforce it | ABI freeze |
| SEC-09 | File limits and retention | Maximum bytes/pages, retention, backup deletion behavior | Real data use |
| SEC-10 | LLM provider privacy terms | Model/provider selection and documented data-retention assessment | Real invoice processing |
| SEC-11 | Pause and recovery authority | Confirm contract pause semantics, role, and emergency process | Mainnet deployment |
| SEC-12 | Hosted secret manager | Select provider and test access/rotation | Public deployment |
| SEC-13 | Mainnet key custody | Separate funded wallet(s), two-person check, recovery plan | Mainnet deployment |

### 16.1 Hard blockers

Do not:

- Freeze the contract ABI before SEC-01 through SEC-04 are resolved and the workflow/role policy is approved.
- Claim EIP-712 gasless approval until SEC-02 and SEC-05 pass on Testnet.
- Deploy Mainnet before all Testnet acceptance tests, negative-path contract tests, and role checks pass.
- Process real invoices until SEC-09 and SEC-10 are resolved.
- publish a repository or demo that contains real private keys, credentials, private invoices, or the plaintext MCP credential included in the supplied guide.

---

## 17. Definition of Done — Security

Security readiness for the MVP requires all items below:

### Application and identity
- [ ] Protected endpoints verify JWT and enforce role/case authorization.
- [ ] Wallet linking uses a one-time, expiring signed challenge.
- [ ] Submitter cannot decide own case; reviewer separation rule is enforced as designed.
- [ ] File upload restrictions and private storage access are tested.
- [ ] API rate limits, CORS, security headers and safe error responses are configured.

### AI and data
- [ ] Extractor has no tools or chain/relayer access.
- [ ] Prompt-injection and malformed-output fixtures pass.
- [ ] Validator is deterministic and versioned.
- [ ] Hard failures cannot be cleared by confirmation-only review.
- [ ] Corrections are append-only and force revalidation.
- [ ] Logs and SSE do not expose sensitive data, keys, signatures or salts.

### Blockchain and relayer
- [ ] Role and state-transition negative tests pass.
- [ ] EIP-712 domain, nonce, deadline and evidence-binding tests pass if the signature path is enabled.
- [ ] Relayer has no approval role and only allows approved target/methods.
- [ ] Transaction is not shown as confirmed before indexed confirmation depth.
- [ ] Indexer handles duplicate events, restart, RPC errors and reorg recovery.
- [ ] Reconciliation marks mismatch as `DISCREPANCY` and blocks affected decisions.
- [ ] Testnet deployment and transactions are independently verifiable.

### Operations and release
- [ ] No secrets or real invoices are present in the public repository.
- [ ] Secret scanning and dependency checks pass or have reviewed exceptions.
- [ ] Incident response and key rotation steps are documented.
- [ ] Mainnet deployment checklist has two-person review.
- [ ] Public demo uses synthetic data.
- [ ] README documents network, contract address, setup, security assumptions, and real transaction links.

---

## 18. References

### Team-provided project documents

- `ActionLoop_PRD.md` — Product Requirements Document.
- `ActionLoop_System_Architecture_v0.2.md` — System Architecture.
- `ActionLoop_Smart_Contract_Specification_v0.1.md` — Smart Contract Specification.
- `ActionLoop_Database_Schema_v0.1.md` — Database Schema.
- `ActionLoop_API_Specification_v0.1.md` — API Specification.
- `ActionLoop_AI_Workflow_Specification_v0.1.md` — AI Workflow Specification.
- `MST BUILDATHON 2.0 GUIDE final.pdf` — network details, deployment requirements and developer resources.

### MST resources listed in the buildathon guide

- MST developer documentation: https://docs.mstblockchain.com
- MST Testnet explorer: https://testnet.mstscan.com
- MST Mainnet explorer: https://mstscan.com
- MST Testnet faucet: https://faucet.mstblockchain.com
- BridgeKey: https://bridgekey.io
- MST TypeScript SDK: https://www.npmjs.com/package/@mstblockchain/mst-sdk
- MST Python SDK: https://pypi.org/project/mst-sdk-python/
- MST VibeKit: https://www.npmjs.com/package/@mstblockchain/mst-vibe-kit

---

## 19. Version History and Sign-off

| Version | Date | Changes | Status |
|---|---|---|---|
| 0.1 | October 31, 2026 | Initial security plan based on ActionLoop PRD and companion technical specifications, including MST Buildathon network/deployment requirements | Draft for team review |

| Role | Name | Review status | Date |
|---|---|---|---|
| Product / architecture owner | Aditya Shelke | Pending | |
| Blockchain lead | Tejas Bankar | Pending | |
| Frontend lead | Siddhika Hanumante | Pending | |

**Approval rule:** This document is a draft until all three team members review it and record the unresolved decisions that apply to their workstreams. Security controls marked as mandatory must not be silently omitted; any exception must be documented with an owner, reason, compensating control, and acceptance.
