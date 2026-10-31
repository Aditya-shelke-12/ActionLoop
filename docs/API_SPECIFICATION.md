# ActionLoop — API Specification

| Field | Value |
|---|---|
| Document version | 0.1 — Draft for team review |
| Date | October 31, 2026 |
| Product | ActionLoop — AI-assisted document processing with blockchain-backed approval and verification |
| Buildathon | MST Blockchain Buildathon 2.0 |
| API style | REST/JSON over HTTPS; Server-Sent Events (SSE) for workflow progress |
| Base path | `/api/v1` |
| Backend | Python + FastAPI + Pydantic |
| Authentication | Supabase Auth JWT; wallet identity linked separately |
| Database | PostgreSQL through Supabase |
| Blockchain | MST Blockchain; environment-selected Testnet/Mainnet deployment |
| Status | Proposed API contract; implementation-ready after team/interface review and MST POC gates |

> **Purpose.** This document defines the HTTP interface between the ActionLoop frontend and backend, the internal backend-to-relayer interface, request/response schemas, authorization, error semantics, asynchronous processing, idempotency and endpoint acceptance criteria. It is the API contract for frontend, backend/AI and blockchain workstreams.
>
> **Source of truth.** This specification is derived from ActionLoop PRD v0.1, System Architecture v0.2, Database Schema v0.1, Smart Contract Specification v0.1 and the supplied MST Blockchain Buildathon 2.0 guide. Where an upstream document leaves an interface decision open, this document marks it as open rather than inventing a network or wallet capability.

---

## 1. API Design Principles

1. **API is the application boundary.** The browser calls FastAPI for business data and workflow actions. It does not connect directly to PostgreSQL or private storage.
2. **Blockchain remains authoritative for contract state.** The API exposes database projections, but it must distinguish application state, chain state, sync status and transaction status. It never reports an approval as blockchain-verified until a matching canonical contract event reaches the configured confirmation policy.
3. **The API does not impersonate users.** The user's linked wallet signs on-chain actions. The API may prepare transaction arguments or EIP-712 typed data, persist a signed intent, and submit it to the relayer; it must not manufacture a user approval signature.
4. **AI does not authorize.** Extraction and planner endpoints return structured information and recommendations. They cannot approve/reject or call the relayer.
5. **Deterministic validation is server-side.** The client cannot submit a validation pass/fail result or change the selected route.
6. **Append-only evidence.** New extraction/correction/validation/review/decision records create new evidence. API endpoints do not overwrite prior versions.
7. **Idempotent mutations.** Every retryable state-changing request uses an `Idempotency-Key`; duplicate requests return the original operation rather than create duplicate cases or transactions.
8. **Async for long work.** Upload processing, AI work, chain confirmation and relayer dispatch are not held open in an HTTP request. The API returns a resource/status reference and exposes polling and/or SSE.
9. **Least privilege.** Every endpoint checks the Supabase session, active user, organization scope, application role and case assignment where applicable. The contract independently enforces on-chain permissions.
10. **Privacy by default.** Do not return private storage paths, raw signatures, hash salts, API keys, private keys, provider prompts or unrestricted internal errors.

---

## 2. Environments and Base URL

The API uses environment-specific deployment configuration. The frontend must not hard-code RPC or contract values.

| Environment | Frontend/API | MST network | Chain ID | Explorer |
|---|---|---|---:|---|
| Local | Local development URLs | Local Hardhat node | Local-configured | Local |
| Staging | Staging deployment | MST Testnet | `91562037` | `https://testnet.mstscan.com` |
| Production demo | Public deployment | MST Mainnet | `4646` | `https://mstscan.com` |

MST RPC configuration supplied in the buildathon guide:

- Testnet: `https://testnetrpc.mstblockchain.com`
- Mainnet: `https://mariorpc.mstblockchain.com/`

The API exposes the active non-secret chain configuration through `GET /api/v1/config`. Do not return private RPC credentials, relayer keys or server secrets. Contract address is absent until that environment's contract has actually been deployed.

---

## 3. Common HTTP Conventions

### 3.1 Headers

| Header | Required | Purpose |
|---|---|---|
| `Authorization: Bearer <supabase-jwt>` | All protected routes | Authenticated application identity |
| `Content-Type: application/json` | JSON requests | JSON payload |
| `Idempotency-Key: <opaque-client-key>` | Mutating routes | Stable identity for one user action/retry |
| `X-Request-ID: <uuid>` | Optional | Client-provided correlation ID; server generates one if absent |
| `Last-Event-ID` | SSE reconnect | Resume progress stream after last delivered event |

For multipart upload, the browser sets the multipart content type and boundary automatically; it must not manually set a bare `multipart/form-data` header.

### 3.2 Identifiers and formats

| Value | API format | Notes |
|---|---|---|
| Internal user/document/workflow ID | UUID string | PostgreSQL identity |
| On-chain `case_id` | `0x` + 64 lowercase hex characters | `bytes32`; not interchangeable with workflow UUID |
| EVM address | `0x` + 40 hex characters | Normalize to lowercase for storage/comparison; checksum display is optional |
| Transaction/block hash | `0x` + 64 hex characters | `bytes32` |
| Money | Decimal string, e.g. `"11800.00"` | Never serialize currency as binary float |
| Timestamp | RFC 3339 UTC, e.g. `2026-10-31T04:00:00Z` | Store UTC; render in user's locale |
| SHA-256 hash | `0x` + 64 hex characters | 32-byte digest |
| Confidence | Number in `[0,1]` | Decimal JSON number; bounded by schema |

### 3.3 Success response convention

Responses return the documented DTO directly; do not wrap every response in a generic `data` object. Paginated list endpoints use:

```json
{
  "items": [],
  "next_cursor": null,
  "has_more": false
}
```

Asynchronous acceptance responses include a stable resource ID and status:

```json
{
  "operation_id": "1fca0b31-f20b-47e1-b54e-c2dd8d10f26f",
  "resource_id": "9a8a6c11-9404-4cd4-9d7f-9eeb66da0c20",
  "status": "ACCEPTED",
  "status_url": "/api/v1/workflows/9a8a6c11-9404-4cd4-9d7f-9eeb66da0c20"
}
```

### 3.4 Error envelope

All API errors use the same structure:

```json
{
  "error": {
    "code": "WORKFLOW_INVALID_STATE",
    "message": "This case is not ready for approval.",
    "correlation_id": "6bf52ed4-f4b4-41c9-8a37-d38a5ed852f3",
    "details": {
      "current_chain_state": "NEEDS_REVIEW",
      "required_chain_state": "READY_FOR_APPROVAL"
    }
  }
}
```

Rules:
- `code` is stable and machine-readable. Frontend behavior must key off `code`, not message text.
- `message` is safe for display and must not expose stack traces, secrets or sensitive document content.
- `correlation_id` is included in logs and support diagnostics.
- `details` is optional, allow-listed and must not contain internal credentials, raw signatures or unredacted invoice content.

### 3.5 HTTP status mapping

| HTTP | Use |
|---:|---|
| `200` | Successful read or completed synchronous action |
| `201` | Resource created synchronously, e.g. uploaded document/workflow draft |
| `202` | Accepted asynchronous action, transaction pending or workflow processing |
| `204` | Successful action with no response body |
| `400` | Malformed request or invalid parameter |
| `401` | Missing, invalid or expired Supabase JWT |
| `403` | Authenticated but not authorized for role/organization/case |
| `404` | Resource absent or intentionally concealed due to access policy |
| `409` | State conflict, duplicate/idempotency conflict, stale version or sync discrepancy |
| `413` | File/request exceeds configured size limit |
| `415` | Unsupported media type |
| `422` | Schema/business validation failure |
| `429` | Rate limit exceeded |
| `500` | Unexpected internal error |
| `502` | Required upstream dependency failed (LLM/RPC/relayer) |
| `503` | Service/dependency temporarily unavailable |
| `504` | Upstream operation timed out; caller may query operation status |

---

## 4. Authentication and Authorization

### 4.1 Application authentication

- User signs in through Supabase Auth.
- Frontend sends the Supabase access JWT as a Bearer token.
- FastAPI verifies token signature, issuer, audience, expiry and subject.
- API resolves the matching `users` row and requires `status = ACTIVE`.
- All application records are scoped to the user's `organization_id`.

The API does not accept a user ID or organization ID from the request body as proof of identity. Those values come from the verified session and server-side records.

### 4.2 Wallet linking

Application identity and wallet identity are separate. A user must link one wallet through a single-use challenge before on-chain actions are enabled.

Wallet challenge response and signature verification method must be validated with BridgeKey in the MST POC. The API must verify domain, nonce, expiry, requested wallet and recovered address. Do not assume a particular BridgeKey signing method until tested.

### 4.3 Authorization layers

| Layer | Enforcement | Purpose |
|---|---|---|
| Session | FastAPI JWT middleware | Is this an authenticated active user? |
| Application RBAC | FastAPI service layer | May this user view/request this operation for this case? |
| On-chain authorization | WorkflowRegistry contract | May the linked wallet execute this state transition now? |

The contract is authoritative for chain actions. Application RBAC provides early feedback and data confidentiality but cannot override contract checks.

### 4.4 Role permissions

| Role | Main API permissions |
|---|---|
| `SUBMITTER` | Upload; register own case; view own cases and timeline |
| `REVIEWER` | View assigned review cases; prepare/submit review, correction or manual entry |
| `APPROVER` | View standard approval queue; prepare/submit standard-tier approve/reject |
| `SENIOR_APPROVER` | View senior approval queue; prepare/submit senior-tier approve/reject |
| `ADMIN` | Manage users, role assignments, policy versions and vendors; view organization cases |
| `AUDITOR` | Read-only case/timeline/verification access granted by organization policy |
| Service identity | Worker/indexer/validator functions only; no interactive login |

Additional separation-of-duty checks:
- Submitter cannot approve/reject their own case.
- Reviewer who reviewed/corrected a case cannot approve/reject that same case under the current MVP policy.
- Relayer wallet has no contract role.
- Validator service wallet must not be granted an approval role.

---

## 5. State Models Exposed by the API

Do not merge these fields into one generic `status`.

### 5.1 Application state

`UPLOADED`, `REGISTERING`, `EXTRACTING`, `VALIDATING`, `PLANNING`, `NEEDS_REVIEW`, `READY_FOR_APPROVAL`, `DECISION_PENDING`, `APPROVED`, `REJECTED`, `FAILED`.

`FAILED` is an operational processing failure; it is not an on-chain rejection.

### 5.2 Chain state

`NOT_REGISTERED`, `SUBMITTED`, `EXTRACTED`, `VALIDATED`, `NEEDS_REVIEW`, `READY_FOR_APPROVAL`, `APPROVED`, `REJECTED`.

This is a projection of confirmed WorkflowRegistry state/events.

### 5.3 Sync status

`IN_SYNC`, `PENDING_CHAIN`, `DISCREPANCY`.

If `DISCREPANCY`, new review/approval actions are blocked until reconciliation.

### 5.4 Transaction status

`PREPARED`, `SUBMITTED`, `PENDING`, `CONFIRMED`, `FAILED`, `REPLACED`.

### 5.5 Validation disposition

`NONE`, `PASS`, `REVIEWABLE_WARNING`, `CORRECTION_REQUIRED`, `HARD_BLOCK`.

- `PASS`: ready for approval after the contract records the permitted transition.
- `REVIEWABLE_WARNING`: reviewer may confirm only after inspecting source evidence.
- `CORRECTION_REQUIRED`: correction creates a new version and requires deterministic revalidation.
- `HARD_BLOCK`: cannot be cleared by generic review confirmation or correction in the MVP; eligible approver may reject from `NEEDS_REVIEW`.

---

## 6. Shared Response DTOs

These are API data-transfer objects. They are not raw database rows and must not expose private storage keys, encrypted salts, signatures or credentials.

### 6.1 `WorkflowSummary`

```json
{
  "workflow_id": "9a8a6c11-9404-4cd4-9d7f-9eeb66da0c20",
  "case_id": "0xaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa",
  "document": {
    "document_id": "a1e4cb64-3bd2-4f03-9e43-4558e0bf6c86",
    "filename": "lab-equipment-invoice.pdf"
  },
  "submitter": {
    "display_name": "Department Staff",
    "wallet_address": "0x1111111111111111111111111111111111111111"
  },
  "application_state": "READY_FOR_APPROVAL",
  "chain_state": "READY_FOR_APPROVAL",
  "sync_status": "IN_SYNC",
  "validation_disposition": "PASS",
  "approval_tier": "STANDARD",
  "created_at": "2026-10-31T04:00:00Z",
  "updated_at": "2026-10-31T04:01:00Z"
}
```

### 6.2 `ExtractedDataVersion`

```json
{
  "data_version": 1,
  "author_kind": "EXTRACTOR",
  "author": {
    "display_name": "ActionLoop Extractor",
    "wallet_address": null
  },
  "fields": {
    "document_type": "invoice",
    "vendor_name": "ABC Technologies",
    "vendor_gstin": "27ABCDE1234F1Z5",
    "invoice_number": "INV-1042",
    "invoice_date": "2026-10-28",
    "subtotal": "10000.00",
    "tax": "1800.00",
    "total_amount": "11800.00",
    "currency": "INR"
  },
  "field_confidence": {
    "vendor_name": 0.98,
    "invoice_number": 0.97,
    "invoice_date": 0.94,
    "total_amount": 0.96
  },
  "source_evidence": {
    "invoice_number": [{"page": 1, "quote": "Invoice No: INV-1042"}]
  },
  "data_hash": "0xbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb",
  "created_at": "2026-10-31T04:00:30Z"
}
```

`author_kind` is `EXTRACTOR`, `REVIEWER_CORRECTION` or `MANUAL_ENTRY`. Money values are decimal strings. `data_hash` is a digest, not a substitute for returning the authorized fields.

### 6.3 `ValidationResult`

```json
{
  "data_version": 1,
  "disposition": "PASS",
  "passed": true,
  "approval_tier": "STANDARD",
  "rule_set_version": 1,
  "policy_version": 1,
  "rules": [
    {"code": "REQUIRED_FIELDS", "status": "PASS", "message": "Required fields are present."},
    {"code": "TOTAL_MATCH", "status": "PASS", "message": "Subtotal plus tax matches the total."},
    {"code": "DUPLICATE_CHECK", "status": "PASS", "message": "No matching vendor/invoice number was found."}
  ],
  "error_codes": [],
  "validation_hash": "0xcccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccc",
  "created_at": "2026-10-31T04:00:45Z"
}
```

### 6.4 `TransactionStatus`

```json
{
  "action": "APPROVE_CASE",
  "status": "PENDING",
  "tx_hash": "0xdddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd",
  "block_number": null,
  "confirmations": 0,
  "required_confirmations": 1,
  "explorer_url": null,
  "safe_error_code": null,
  "safe_error_message": null
}
```

The explorer URL pattern shown is illustrative. Confirm the actual MSTScan URL format during the POC; if unknown, return a null `explorer_url` and always return the raw transaction hash.

### 6.5 `AuditTimelineEvent`

```json
{
  "event_id": "b4b02c69-6a62-4ff2-bb0e-1a4b0f43b4a2",
  "timestamp": "2026-10-31T04:01:00Z",
  "event_name": "CaseReadyForApproval",
  "actor_label": "ActionLoop Validator",
  "actor_wallet": "0x2222222222222222222222222222222222222222",
  "data_version": 1,
  "source": "BLOCKCHAIN",
  "verification_status": "CONFIRMED",
  "transaction": {
    "tx_hash": "0xdddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd",
    "block_number": 12345,
    "explorer_url": null
  }
}
```

`source` may be `APPLICATION`, `BLOCKCHAIN` or `SYSTEM`. A local application event must not be mislabeled as an on-chain transaction.

### 6.6 `ApprovalIntent`

```json
{
  "intent_id": "2f5f1f20-9021-4f83-a0bf-97b44c8cd020",
  "decision": "APPROVE",
  "workflow_id": "9a8a6c11-9404-4cd4-9d7f-9eeb66da0c20",
  "case_id": "0xaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa",
  "data_version": 1,
  "evidence_hash": "0xeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee",
  "signer_wallet": "0x3333333333333333333333333333333333333333",
  "submission_mode": "EIP712_RELAYED",
  "typed_data": {
    "domain": {
      "name": "ActionLoop Workflow Registry",
      "version": "1",
      "chainId": 91562037,
      "verifyingContract": "0x4444444444444444444444444444444444444444"
    },
    "primaryType": "Decision",
    "types": {
      "Decision": [
        {"name": "caseId", "type": "bytes32"},
        {"name": "decision", "type": "uint8"},
        {"name": "dataVersion", "type": "uint32"},
        {"name": "evidenceHash", "type": "bytes32"},
        {"name": "reasonHash", "type": "bytes32"},
        {"name": "nonce", "type": "uint256"},
        {"name": "deadline", "type": "uint256"}
      ]
    },
    "message": {
      "caseId": "0xaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa",
      "decision": 1,
      "dataVersion": 1,
      "evidenceHash": "0xeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee",
      "reasonHash": "0x0000000000000000000000000000000000000000000000000000000000000000",
      "nonce": "0",
      "deadline": "1793420000"
    }
  },
  "summary": {
    "vendor_name": "ABC Technologies",
    "invoice_number": "INV-1042",
    "amount": "11800.00",
    "currency": "INR",
    "decision": "Approve",
    "data_version": 1
  },
  "expires_at": "2026-10-31T04:15:00Z"
}
```

The `types` object must be generated from the approved EIP-712 schema in the Smart Contract Specification; it must not be guessed or independently maintained by frontend/backend. If EIP-712 signing does not pass BridgeKey/MST compatibility tests, return `submission_mode = DIRECT_WALLET` and structured contract-call details instead of returning unusable typed data.

### 6.7 `VerificationReport`

```json
{
  "workflow_id": "9a8a6c11-9404-4cd4-9d7f-9eeb66da0c20",
  "case_id": "0xaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa",
  "overall_status": "PASS",
  "checks": [
    {"name": "DOCUMENT_HASH", "status": "PASS", "expected": "0x...", "actual": "0x..."},
    {"name": "DATA_HASH", "status": "PASS", "expected": "0x...", "actual": "0x..."},
    {"name": "VALIDATION_HASH", "status": "PASS", "expected": "0x...", "actual": "0x..."},
    {"name": "VERSION_CONSISTENCY", "status": "PASS"},
    {"name": "STATE_TRANSITION_SEQUENCE", "status": "PASS"},
    {"name": "APPROVER_AUTHORIZATION", "status": "PASS"},
    {"name": "TRANSACTION_CONFIRMATION", "status": "PASS"}
  ],
  "chain": {
    "network_name": "MST Testnet",
    "chain_id": 91562037,
    "contract_address": "0x4444444444444444444444444444444444444444",
    "final_chain_state": "APPROVED",
    "decision_tx_hash": "0xdddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd",
    "block_number": 12345,
    "confirmations": 1
  },
  "verified_at": "2026-10-31T04:05:00Z"
}
```

Each check status is `PASS`, `FAIL`, `PENDING` or `UNAVAILABLE`. Overall status is `PASS` only when all required checks pass. Do not say that a current `hasRole` read proves historical authorization if role grant/revocation history at the decision block is unavailable; return `UNAVAILABLE` for that check.

---

## 7. Endpoint Catalogue

All paths below are relative to `/api/v1`.

### 7.1 Configuration and health

| Method + path | Auth | Purpose |
|---|---|---|
| `GET /config` | Public | Return active network ID, symbol, explorer base, contract address if deployed, supported wallet/submission modes |
| `GET /health/live` | Public | Process liveness only |
| `GET /health/ready` | Internal/admin or deployment probe | Readiness for required dependencies; do not expose secret values or detailed internals |

`GET /config` example:

```json
{
  "environment": "TESTNET",
  "network": {"name": "MST Testnet", "chain_id": 91562037, "currency_symbol": "tMSTC", "explorer_base_url": "https://testnet.mstscan.com"},
  "contract": {"workflow_registry_address": null, "abi_version": null, "deployment_status": "NOT_DEPLOYED"},
  "wallet": {"preferred": "BRIDGEKEY", "eip712_supported": false, "direct_transaction_supported": true},
  "submission_modes": ["DIRECT_WALLET"]
}
```

Boolean support flags must reflect tested deployment configuration, not assumptions. Do not mark EIP-712 true until a BridgeKey signature is recovered successfully by the deployed test contract.

### 7.2 Wallet linking

#### `POST /auth/wallet/challenge`

**Auth:** required. **Role:** any active user. **Purpose:** issue a one-time wallet-link challenge.

Request:

```json
{"wallet_address": "0x3333333333333333333333333333333333333333"}
```

Response `201`:

```json
{
  "challenge_id": "2f5f1f20-9021-4f83-a0bf-97b44c8cd020",
  "wallet_address": "0x3333333333333333333333333333333333333333",
  "message": "ActionLoop wallet link\\nDomain: app.example\\nAddress: 0x3333333333333333333333333333333333333333\\nNonce: ...\\nIssued At: ...\\nExpiration Time: ...",
  "expires_at": "2026-10-31T04:15:00Z",
  "signing_method": "BRIDGEKEY_POC_PENDING"
}
```

The actual message format and signing method must be selected after BridgeKey testing. Challenge lifetime: five minutes recommended. A challenge is single-use.

Errors: `WALLET_ALREADY_LINKED`, `WALLET_CHALLENGE_RATE_LIMITED`, `WALLET_FORMAT_INVALID`.

#### `POST /auth/wallet/link`

**Auth:** required. **Purpose:** verify the challenge signature and attach wallet to current user.

Request:

```json
{
  "challenge_id": "2f5f1f20-9021-4f83-a0bf-97b44c8cd020",
  "wallet_address": "0x3333333333333333333333333333333333333333",
  "signature": "0x..."
}
```

Response `200`:

```json
{
  "wallet_address": "0x3333333333333333333333333333333333333333",
  "wallet_linked": true,
  "linked_at": "2026-10-31T04:10:00Z",
  "on_chain_roles": []
}
```

Wallet linking does not automatically grant contract roles. Role grants require the admin process and confirmed chain state.

Errors: `WALLET_CHALLENGE_NOT_FOUND`, `WALLET_CHALLENGE_EXPIRED`, `WALLET_CHALLENGE_ALREADY_USED`, `WALLET_SIGNATURE_INVALID`, `WALLET_ALREADY_LINKED_TO_ANOTHER_USER`.

### 7.3 Document intake

#### `POST /documents/upload`

**Auth:** required. **Role:** `SUBMITTER`. **Content type:** `multipart/form-data`.

Fields:
- `file`: required PDF/JPG/PNG file.
- `document_type`: optional; if supplied, must be an allowed value (MVP default `INVOICE`).

Server behavior:
1. Enforce configured file-size limit and allow-listed extension/MIME/signature checks.
2. Reject encrypted, malformed or unsafe files according to the upload policy.
3. Compute SHA-256 over exact original file bytes.
4. Store original in private storage.
5. Create `documents` and draft `workflows` records with application state `UPLOADED`, chain state `NOT_REGISTERED` and sync status `PENDING_CHAIN`.
6. Generate deterministic `case_id` using the documented shared derivation function.
7. Return structured transaction intent information for the submitter to register the case through BridgeKey.

Response `201`:

```json
{
  "document_id": "a1e4cb64-3bd2-4f03-9e43-4558e0bf6c86",
  "workflow_id": "9a8a6c11-9404-4cd4-9d7f-9eeb66da0c20",
  "case_id": "0xaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa",
  "document": {"filename": "lab-equipment-invoice.pdf", "mime_type": "application/pdf", "size_bytes": 184320, "sha256": "0x..."},
  "application_state": "UPLOADED",
  "chain_state": "NOT_REGISTERED",
  "registration": {
    "mode": "DIRECT_WALLET",
    "chain_id": 91562037,
    "contract_address": "0x4444444444444444444444444444444444444444",
    "function": "createCase",
    "args": {"caseId": "0xaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa", "documentHash": "0x..."}
  },
  "next_action": "SIGN_CASE_REGISTRATION"
}
```

If the contract is not deployed/configured in the selected environment, the upload may be stored in local development mode only; staging/production must return a clear `CHAIN_NOT_CONFIGURED` error rather than pretending registration occurred. The direct-wallet registration path is the MVP default; relayed case creation is not part of the first relayer scope.

Errors: `FILE_TYPE_NOT_ALLOWED`, `FILE_TOO_LARGE`, `FILE_INVALID`, `FILE_SCAN_REJECTED`, `STORAGE_UNAVAILABLE`, `CHAIN_NOT_CONFIGURED`, `IDEMPOTENCY_KEY_REUSED_WITH_DIFFERENT_REQUEST`.

#### `POST /workflows/{workflow_id}/register`

**Auth:** required. **Role:** owning `SUBMITTER`. **Purpose:** submit evidence that the submitter broadcast the `createCase` transaction from BridgeKey.

Request:

```json
{"tx_hash": "0xdddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd"}
```

Server behavior:
- Validate that the hash is a valid transaction hash on the configured chain.
- Fetch receipt and transaction details through the chain adapter.
- Verify target contract, `createCase` call/event, `case_id`, document hash and recovered transaction sender against the submitter's linked wallet.
- Record the transaction and return pending/confirmed state based on receipt and confirmation depth. The API does not trust the client-supplied transaction hash by itself.

Response `202` while pending or `200` if already confirmed:

```json
{
  "workflow_id": "9a8a6c11-9404-4cd4-9d7f-9eeb66da0c20",
  "application_state": "REGISTERING",
  "chain_state": "NOT_REGISTERED",
  "sync_status": "PENDING_CHAIN",
  "transaction": {"action": "CREATE_CASE", "status": "PENDING", "tx_hash": "0x...", "confirmations": 0, "explorer_url": null}
}
```

After the indexer confirms `CaseCreated`, the worker starts extraction. Errors: `CASE_ALREADY_REGISTERED`, `REGISTRATION_TX_NOT_FOUND`, `REGISTRATION_TX_WRONG_CHAIN`, `REGISTRATION_TX_WRONG_CONTRACT`, `REGISTRATION_EVENT_MISMATCH`, `REGISTRATION_SENDER_MISMATCH`, `REGISTRATION_TX_REVERTED`.

#### `GET /documents/{document_id}`

**Auth:** required. **Role:** user with case/document access.

Returns document metadata and current processing summary. It does not return private storage bucket/path. A short-lived signed viewing URL may be included only after case authorization.

Response `200`:

```json
{
  "document_id": "a1e4cb64-3bd2-4f03-9e43-4558e0bf6c86",
  "workflow_id": "9a8a6c11-9404-4cd4-9d7f-9eeb66da0c20",
  "filename": "lab-equipment-invoice.pdf",
  "mime_type": "application/pdf",
  "size_bytes": 184320,
  "scan_status": "CLEAN",
  "document_hash": "0x...",
  "view_url": "https://private-storage.example/signed-url",
  "view_url_expires_at": "2026-10-31T04:16:00Z"
}
```

Do not cache signed URLs for longer than their expiry. Errors: `DOCUMENT_NOT_FOUND`, `DOCUMENT_ACCESS_DENIED`, `DOCUMENT_NOT_AVAILABLE`.

### 7.4 Workflow reads

#### `GET /workflows`

**Auth:** required. **Role:** all roles with list scope. Query parameters:

| Parameter | Type | Meaning |
|---|---|---|
| `application_state` | enum, optional | Filter application state |
| `chain_state` | enum, optional | Filter confirmed chain state |
| `sync_status` | enum, optional | Filter sync state |
| `assigned_to_me` | boolean, optional | Only assigned cases |
| `submitted_by_me` | boolean, optional | Only current submitter's cases |
| `approval_tier` | `STANDARD`/`SENIOR`, optional | Approval-tier filter |
| `created_after`, `created_before` | RFC 3339, optional | Date range |
| `cursor` | opaque string, optional | Cursor pagination |
| `limit` | integer, 1–100, default 20 | Page size |

Authorization scope is enforced server-side; query parameters cannot widen a user's access. Sort order is stable `created_at DESC, id DESC`.

Response `200`:

```json
{"items": [/* WorkflowSummary */], "next_cursor": null, "has_more": false}
```

#### `GET /workflows/{workflow_id}`

**Auth:** required. **Role:** permitted case viewer.

Returns `WorkflowDetail`:

```json
{
  "summary": {/* WorkflowSummary */},
  "latest_extraction": {/* ExtractedDataVersion or null */},
  "extraction_versions": [/* ExtractedDataVersion summaries; full version history is also available below */],
  "latest_validation": {/* ValidationResult or null */},
  "planner_recommendation": {
    "route": "REQUEST_APPROVAL",
    "policy_version": 1,
    "explanation": "All mandatory checks passed. The case is ready for the standard approval queue."
  },
  "assignment": {"reviewer_user_id": null, "approver_user_id": null, "approval_tier": "STANDARD"},
  "latest_transaction": {/* TransactionStatus or null */},
  "available_actions": ["VIEW_AUDIT", "PREPARE_DECISION"]
}
```

`available_actions` is a UI hint only; every action is re-authorized when submitted. Errors: `WORKFLOW_NOT_FOUND`, `WORKFLOW_ACCESS_DENIED`.

#### `GET /workflows/{workflow_id}/versions`

Returns ordered, append-only data versions and associated validation/review metadata. Does not return encrypted salts or raw signing material.

Response `200`:

```json
{"workflow_id": "9a8a6c11-9404-4cd4-9d7f-9eeb66da0c20", "items": [/* ExtractedDataVersion */], "next_cursor": null}
```

#### `GET /workflows/{workflow_id}/events`

**Purpose:** SSE progress/timeline stream. Query parameter `after` is optional event cursor; reconnect may use `Last-Event-ID` header.

Event names include:
- `workflow.stage_changed`
- `workflow.extraction_completed`
- `workflow.validation_completed`
- `workflow.needs_review`
- `workflow.ready_for_approval`
- `chain.transaction_pending`
- `chain.transaction_confirmed`
- `chain.transaction_failed`
- `chain.discrepancy_detected`
- `workflow.completed`
- `heartbeat`

Example:

```text
id: 17442
event: workflow.stage_changed
data: {"workflow_id":"9a8a6c11-9404-4cd4-9d7f-9eeb66da0c20","application_state":"VALIDATING","chain_state":"EXTRACTED","sync_status":"IN_SYNC"}

```

Implementation note: no leading whitespace before the `event:` or `data:` fields. SSE is a notification channel; clients should refetch the workflow resource after a material event. Use polling as a fallback if SSE is unavailable.

### 7.5 Review and correction

Review semantics are strict:
- `REVIEWABLE_WARNING` may be confirmed after human inspection.
- `CORRECTION_REQUIRED` requires a new data version and deterministic revalidation.
- `HARD_BLOCK` cannot be cleared by generic confirmation or correction in the MVP; an eligible approver may reject the case from `NEEDS_REVIEW`.
- Manual entry after extraction failure is a new version and must go through the same validator.

#### `POST /workflows/{workflow_id}/review/prepare`

**Auth:** required. **Role:** assigned `REVIEWER`.

Request for warning confirmation:

```json
{"action": "CONFIRM_WARNING", "data_version": 1, "reason_text": "Verified the low-confidence vendor suffix against the source invoice."}
```

Request for correction:

```json
{
  "action": "CORRECT_DATA",
  "base_data_version": 1,
  "corrected_fields": {"total_amount": "11800.00"},
  "reason_text": "Corrected the total using the printed invoice total."
}
```

Request for manual entry after extraction failure:

```json
{
  "action": "MANUAL_ENTRY",
  "fields": {
    "document_type": "invoice",
    "vendor_name": "ABC Technologies",
    "invoice_number": "INV-1042",
    "invoice_date": "2026-10-28",
    "subtotal": "10000.00",
    "tax": "1800.00",
    "total_amount": "11800.00",
    "currency": "INR"
  },
  "reason_text": "Extraction failed after bounded retries; fields entered from the original invoice."
}
```

Response `200`:

```json
{
  "review_action_id": "5a8ec3b0-8b8e-4df3-a0d9-cc7d6889e001",
  "action": "CORRECT_DATA",
  "base_data_version": 1,
  "proposed_data_version": 2,
  "proposed_data_hash": "0x...",
  "reason_hash": "0x...",
  "chain_action": {
    "mode": "DIRECT_WALLET",
    "chain_id": 91562037,
    "contract_address": "0x4444444444444444444444444444444444444444",
    "function": "recordCorrection",
    "args": {"caseId": "0x...", "baseVersion": 1, "newDataHash": "0x...", "reasonHash": "0x..."}
  },
  "expires_at": "2026-10-31T04:20:00Z"
}
```

The prepare operation must not mark the review as complete or advance workflow state. Proposed correction data must be held in a protected pending-action record until the matching chain transaction is confirmed. **Database schema amendment required:** add a `review_drafts`/pending-action table (or an equivalent explicitly modeled protected record) with expiry, reviewer, proposed fields, version/hash, reason hash and status. Do not put invoice fields into the generic worker job payload.

#### `POST /workflows/{workflow_id}/review`

**Auth:** required. **Role:** same reviewer who prepared the action.

Request:

```json
{
  "review_action_id": "5a8ec3b0-8b8e-4df3-a0d9-cc7d6889e001",
  "tx_hash": "0xdddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd"
}
```

For the MVP, reviewer actions use a direct BridgeKey transaction. The API validates the transaction receipt, contract address, function/event, actor wallet, case ID, version and hashes. It does not trust the client-supplied hash. After the chain event reaches configured confirmation depth, the backend finalizes the review action, persists the new immutable `extracted_data` version (for correction/manual entry), and schedules deterministic validation. For warning confirmation, the confirmed `CaseReviewed` event is required before setting the workflow as approval-ready.

Response `202` pending or `200` confirmed:

```json
{
  "review_action_id": "5a8ec3b0-8b8e-4df3-a0d9-cc7d6889e001",
  "status": "PENDING",
  "transaction": {/* TransactionStatus */},
  "workflow": {"application_state": "NEEDS_REVIEW", "chain_state": "NEEDS_REVIEW", "sync_status": "PENDING_CHAIN"}
}
```

After correction/manual entry is confirmed, expected next application state is `VALIDATING`; the worker reruns rules. For warning confirmation, expected chain state becomes `READY_FOR_APPROVAL` after confirmation. Errors: `REVIEW_NOT_ASSIGNED`, `REVIEW_ACTION_NOT_FOUND`, `REVIEW_ACTION_EXPIRED`, `REVIEW_DISPOSITION_NOT_ALLOWED`, `HARD_BLOCK_CANNOT_BE_CONFIRMED`, `STALE_DATA_VERSION`, `REVIEW_TX_MISMATCH`, `REVIEW_TX_REVERTED`.

### 7.6 Approval and rejection

#### `GET /approvals`

**Auth:** required. **Role:** `APPROVER` or `SENIOR_APPROVER`.

Query parameters: `approval_tier`, `cursor`, `limit`.

Returns cases in `READY_FOR_APPROVAL` assigned/visible to the user's tier, with `sync_status = IN_SYNC`. Cases with `DISCREPANCY` are not actionable and must be shown separately as blocked if the user has visibility.

#### `POST /workflows/{workflow_id}/decisions/prepare`

**Auth:** required. **Role:** eligible `APPROVER` or `SENIOR_APPROVER`.

Request:

```json
{"decision": "APPROVE", "data_version": 2, "reason_text": "Invoice verified and approved."}
```

Allowed `decision`: `APPROVE` or `REJECT`.

Server behavior:
1. Verify application role, assignment/tier and wallet link.
2. Read case state, approval tier, current data version, evidence hash and current signer nonce from MST chain through `ChainGateway`.
3. Verify state is eligible; sync is not `DISCREPANCY`; user is not submitter or reviewer on this case.
4. Canonicalize and hash optional reason text; reason text remains off-chain.
5. Prepare signed intent with a short configurable deadline.
6. Select `EIP712_RELAYED` only if the active environment has passed the BridgeKey/EIP-712 POC. Otherwise select `DIRECT_WALLET` and return structured call details.

Response `200`:

```json
{
  "intent_id": "2f5f1f20-9021-4f83-a0bf-97b44c8cd020",
  "decision": "APPROVE",
  "case_id": "0x...",
  "data_version": 2,
  "evidence_hash": "0x...",
  "signer_wallet": "0x3333333333333333333333333333333333333333",
  "submission_mode": "EIP712_RELAYED",
  "typed_data": {/* ApprovalIntent typed-data domain/types/message */},
  "direct_call": null,
  "summary": {"vendor_name": "ABC Technologies", "invoice_number": "INV-1042", "amount": "11800.00", "currency": "INR", "decision": "Approve", "data_version": 2},
  "expires_at": "2026-10-31T04:15:00Z"
}
```

If direct mode is selected, `typed_data` is null and `direct_call` contains `chain_id`, `contract_address`, function name and strongly typed ABI arguments for `approveCase` or `rejectCase`. The frontend sends that call through BridgeKey; it does not send arbitrary calldata constructed by user input.

Errors: `APPROVAL_ROLE_REQUIRED`, `APPROVAL_TIER_MISMATCH`, `SELF_APPROVAL_FORBIDDEN`, `REVIEWER_CANNOT_APPROVE_OWN_REVIEW`, `WORKFLOW_INVALID_STATE`, `WORKFLOW_SYNC_DISCREPANCY`, `STALE_DATA_VERSION`, `CHAIN_UNAVAILABLE`, `WALLET_NOT_LINKED`, `SIGNING_MODE_UNAVAILABLE`.

#### `POST /workflows/{workflow_id}/approve`

**Auth:** required. **Role:** eligible approver. Accepts one of the two submission modes.

Relayed request:

```json
{
  "submission_mode": "EIP712_RELAYED",
  "intent_id": "2f5f1f20-9021-4f83-a0bf-97b44c8cd020",
  "signature": "0x..."
}
```

Direct-wallet request:

```json
{
  "submission_mode": "DIRECT_WALLET",
  "intent_id": "2f5f1f20-9021-4f83-a0bf-97b44c8cd020",
  "tx_hash": "0xdddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd"
}
```

The server verifies the signer/linked wallet or direct transaction sender, exact intent, current evidence/version, chain/contract/function and decision. Relayed intents are persisted and queued in the transactional outbox; the relayer submits unchanged signature data. Direct wallet transactions are observed and verified by transaction/receipt and indexed event. The API response does not itself make the case approved.

Response `202`:

```json
{
  "workflow_id": "9a8a6c11-9404-4cd4-9d7f-9eeb66da0c20",
  "application_state": "DECISION_PENDING",
  "chain_state": "READY_FOR_APPROVAL",
  "sync_status": "PENDING_CHAIN",
  "transaction": {"action": "APPROVE_CASE", "status": "PENDING", "tx_hash": "0x...", "confirmations": 0, "explorer_url": null}
}
```

The final `APPROVED` state is set only after the indexer confirms the `CaseApproved` event at the configured confirmation depth.

#### `POST /workflows/{workflow_id}/reject`

Same schema and behavior as `/approve`, but the intent decision must be `REJECT`, and the confirmed chain event must be `CaseRejected`. Rejection from `NEEDS_REVIEW` is allowed only under the contract's specified rule/tier checks. A rejected case is terminal.

#### `GET /workflows/{workflow_id}/audit`

Returns merged timeline assembled from application audit logs, transaction lifecycle and indexed chain events. Events are sorted by their authoritative timestamp/block order; application events must be visually distinguished from on-chain events.

Response `200`:

```json
{
  "workflow_id": "9a8a6c11-9404-4cd4-9d7f-9eeb66da0c20",
  "items": [/* AuditTimelineEvent */],
  "next_cursor": null
}
```

### 7.7 Blockchain verification

#### `GET /blockchain/verify/{workflow_id}`

**Auth:** required; role/case visibility policy applies. **Purpose:** backend recomputation of off-chain evidence and a report that the frontend can compare with direct on-chain reads.

Query parameters:
- `data_version`: optional; defaults to the data version used for final decision.
- `include_recomputed_hashes`: optional boolean, default `true` for authorized case viewers.

Backend behavior:
1. Read the deployed contract case through the configured chain adapter.
2. Load exact source file and selected extraction/validation version from authorized private storage/database.
3. Recompute SHA-256 file hash.
4. Recompute canonical structured-data and validation hashes using the correct canonicalization version and protected salt material.
5. Compare hashes/version/state with chain storage and confirmed event history.
6. Evaluate legal event ordering and decision transaction confirmation.
7. Evaluate historical signer role at decision block if role grant/revocation event history is available.

Response `200`: `VerificationReport`.

The frontend should separately read contract state/events directly through viem/MST SDK and compare them with the backend report. A backend report alone is not an independent trustless proof. If private data or salt cannot be accessed, return `UNAVAILABLE` for that check rather than `PASS`.

Errors: `WORKFLOW_NOT_FOUND`, `CHAIN_UNAVAILABLE`, `EVIDENCE_NOT_AVAILABLE`, `DEPLOYMENT_NOT_FOUND`, `VERIFICATION_INCOMPLETE`.

### 7.8 Dashboard

#### `GET /dashboard/summary`

**Auth:** required. All roles receive counts limited to their scope.

Response `200`:

```json
{
  "counts": {
    "total_visible": 24,
    "processing": 3,
    "needs_review": 4,
    "ready_for_approval": 5,
    "decision_pending": 2,
    "approved": 8,
    "rejected": 2,
    "sync_discrepancies": 0
  },
  "pending_actions": [{"type": "REVIEW", "count": 2}, {"type": "APPROVAL", "count": 3}],
  "recent_activity": [/* AuditTimelineEvent summary */]
}
```

Dashboard counts are convenience projections, not blockchain proof.

### 7.9 Administration

All `/admin/*` routes require `ADMIN` role and an active wallet where an on-chain role grant/revoke transaction is required.

| Method + path | Purpose |
|---|---|
| `GET /admin/users` | List organization users and role status |
| `POST /admin/users/invite` | Invite/create a user through approved identity flow |
| `GET /admin/roles` | List app and on-chain role assignments |
| `POST /admin/roles/prepare` | Prepare role grant/revoke transaction details |
| `POST /admin/roles/confirm` | Submit/verify admin-signed role grant/revoke transaction hash |
| `GET /admin/policies` | List immutable policy versions |
| `POST /admin/policies` | Create a new immutable policy version |
| `POST /admin/policies/{version}/activate` | Activate policy version for new cases |
| `GET /admin/vendors` | List registered vendors |
| `POST /admin/vendors` | Add vendor |
| `PATCH /admin/vendors/{vendor_id}` | Create a new vendor record version/update active status according to DB policy |
| `GET /admin/chain-deployments` | List configured deployments without secret RPC credentials |

Policy response must clearly say that monetary thresholds are organization-configured demo policies, not legal/college policy defaults. Policy versions already used by a workflow remain immutable.

Role grant/revocation is not complete until the corresponding contract transaction is confirmed and `role_assignments.chain_status = CONFIRMED` (or `REVOKED`). An admin API write alone does not grant an on-chain role.

---

## 8. Internal Relayer API (Service-to-Service)

The relayer is not a public browser API. It is called only by the backend outbox dispatcher over an authenticated private service channel.

### 8.1 `POST /internal/v1/decision-submissions`

Request:

```json
{
  "submission_id": "c2a35f91-a5d0-4d9f-8b79-6cc7e2d009c0",
  "chain_id": 91562037,
  "contract_address": "0x4444444444444444444444444444444444444444",
  "function": "approveWithSignature",
  "typed_data": {/* Exact approved EIP-712 domain/types/message */},
  "signature": "0x...",
  "idempotency_key": "case:0x...:approve:v2:signer:0x...:nonce:0"
}
```

Relayer obligations:
- Allow-list chain ID, contract address and only `approveWithSignature` / `rejectWithSignature` selectors for MVP.
- Verify domain, signature, signer, deadline and current on-chain nonce.
- Simulate transaction where supported.
- Enforce rate/gas/spend/balance limits.
- Submit signed intent unchanged.
- Never hold `APPROVER_ROLE` or `SENIOR_APPROVER_ROLE`.

Response `202`:

```json
{
  "submission_id": "c2a35f91-a5d0-4d9f-8b79-6cc7e2d009c0",
  "status": "SUBMITTED",
  "tx_hash": "0xdddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd"
}
```

This response means submission occurred, not confirmation. The backend indexer owns final confirmation.

### 8.2 Relayer status/read endpoint

`GET /internal/v1/decision-submissions/{submission_id}` returns service transaction lifecycle and safe error codes. It must require service-to-service authentication. The public frontend never calls this endpoint.

### 8.3 Relayer errors

`RELAYER_CHAIN_NOT_ALLOWED`, `RELAYER_CONTRACT_NOT_ALLOWED`, `RELAYER_FUNCTION_NOT_ALLOWED`, `RELAYER_SIGNATURE_INVALID`, `RELAYER_NONCE_MISMATCH`, `RELAYER_INTENT_EXPIRED`, `RELAYER_SIMULATION_REVERTED`, `RELAYER_RATE_LIMITED`, `RELAYER_BALANCE_LOW`, `RELAYER_SUBMISSION_FAILED`.

---

## 9. API Error Catalogue

| Error code | HTTP | Meaning / frontend behavior |
|---|---:|---|
| `AUTH_REQUIRED` | 401 | Sign in again |
| `AUTH_SESSION_EXPIRED` | 401 | Refresh session/sign in |
| `USER_INACTIVE` | 403 | Contact administrator |
| `ROLE_REQUIRED` | 403 | User lacks required application role |
| `CASE_ACCESS_DENIED` | 404/403 | Case not visible to user |
| `WALLET_NOT_LINKED` | 409 | Link wallet before on-chain action |
| `WALLET_SIGNATURE_INVALID` | 422 | Reconnect wallet and sign challenge again |
| `FILE_TYPE_NOT_ALLOWED` | 415 | Select PDF/JPG/PNG |
| `FILE_TOO_LARGE` | 413 | Reduce file size |
| `FILE_INVALID` | 422 | Replace corrupt/unsupported file |
| `FILE_SCAN_REJECTED` | 422 | Unsafe file rejected |
| `DUPLICATE_DOCUMENT` | 409 | Show matching case/reference; do not auto-approve |
| `WORKFLOW_INVALID_STATE` | 409 | Refresh case; explain required state |
| `WORKFLOW_SYNC_DISCREPANCY` | 409 | Block action and show reconciliation warning |
| `STALE_DATA_VERSION` | 409 | Reload latest evidence and prepare new intent |
| `VALIDATION_HARD_BLOCK` | 409 | Cannot clear via reviewer confirmation |
| `REVIEW_DISPOSITION_NOT_ALLOWED` | 409 | Explain whether correction or confirmation is allowed |
| `HARD_BLOCK_CANNOT_BE_CONFIRMED` | 409 | Route to investigation/eligible rejection |
| `APPROVAL_TIER_MISMATCH` | 403 | Show required senior tier |
| `SELF_APPROVAL_FORBIDDEN` | 403 | Submitter cannot decide own case |
| `REVIEWER_CANNOT_APPROVE_OWN_REVIEW` | 403 | Separate reviewer and approver |
| `CHAIN_NOT_CONFIGURED` | 503 | Deployment/configuration issue; no simulated fallback |
| `CHAIN_UNAVAILABLE` | 503 | Keep pending; retry reads safely |
| `CHAIN_TX_REVERTED` | 409 | Map contract custom error; do not auto-retry deterministic revert |
| `TRANSACTION_NOT_FOUND` | 404 | Verify network/hash; retry lookup |
| `TRANSACTION_PENDING` | 202 | Continue polling/streaming |
| `RELAYER_UNAVAILABLE` | 503 | Keep outbox item retryable; show pending |
| `RELAYER_BALANCE_LOW` | 503 | Notify operator; do not claim transaction submitted |
| `IDEMPOTENCY_KEY_CONFLICT` | 409 | Same key was reused for a different request |
| `RATE_LIMITED` | 429 | Retry after `Retry-After` |
| `UPSTREAM_LLM_UNAVAILABLE` | 502/503 | Bounded retry then manual-entry review path |
| `EXTRACTION_SCHEMA_INVALID` | 422/202 | Bounded retry; then human manual entry |
| `INTERNAL_ERROR` | 500 | Show correlation ID; never display trace |

The backend maps contract custom errors to stable API codes. It must not expose raw revert data directly to end users.

---

## 10. Idempotency, Concurrency and Transaction Semantics

### 10.1 Idempotency

- Required for `POST /documents/upload`, `/register`, `/review/prepare`, `/review`, `/decisions/prepare`, `/approve`, `/reject`, and all admin mutations.
- Key scope is `(organization_id, user_id, route, idempotency_key)`.
- Same key + same normalized request returns the original response/result.
- Same key + different request returns `409 IDEMPOTENCY_KEY_CONFLICT`.
- Store request fingerprint and resulting resource/operation ID. Do not store raw files in idempotency metadata.

### 10.2 Optimistic concurrency

- Review/correction and decision requests must include the expected `data_version`.
- Decision preparation reads current on-chain state/evidence/nonce; if any changes before submission, contract or backend rejects the stale intent.
- Include `ETag` on workflow detail responses if implemented. Mutations may require `If-Match`; this is optional for MVP if the explicit version guard is consistently enforced.

### 10.3 Outbox

For relayed approvals/rejections:
1. Verify and persist the signed intent.
2. In the same database transaction, create the approval record and outbox transaction row.
3. Dispatcher claims row using `FOR UPDATE SKIP LOCKED` and calls the relayer outside the DB lock.
4. Persist returned tx hash/status.
5. Indexer independently observes and confirms the matching contract event.
6. Only then set chain projection and approval result to confirmed.

A failed HTTP call between API and relayer must not lose the signed intent. Retrying the outbox dispatch uses the same idempotency key. The relayer also deduplicates `submission_id`/idempotency key.

### 10.4 Direct wallet transaction observation

For direct BridgeKey actions (MVP registration and reviewer actions; decision fallback if EIP-712 is unavailable), the API receives a transaction hash and independently validates:
- Correct chain ID.
- Correct contract address.
- Expected function/event and ABI arguments.
- Correct linked sender wallet.
- Correct case ID, data version and evidence hash.
- Receipt success and confirmation depth.

The API never treats a client-provided `tx_hash` as proof until it verifies the transaction and event.

---

## 11. API-to-Worker and API-to-Chain Boundaries

### 11.1 API to worker

The API inserts durable `jobs` rows in PostgreSQL. Worker claims jobs with leases/row locking and executes LangGraph. HTTP request handlers must not synchronously wait for full LLM extraction, validation, chain confirmation or reconciliation.

Job payloads contain identifiers and routing metadata only. The worker loads document/evidence data from private storage/database under service credentials.

### 11.2 Worker to chain adapter

The workflow worker calls a single `ChainGateway` interface, not SDK-specific functions directly. Logical operations include `record_extraction`, `record_extraction_failure`, `record_validation`, `route_validated_case`, reads and event/receipt helpers. The adapter can use the MST Python SDK if POC-tested; otherwise use a tested JSON-RPC implementation such as `web3.py`. The final choice is an implementation decision gated by MST POC results.

### 11.3 API to relayer

Only the outbox dispatcher calls the internal relayer endpoint. Browser clients never call the relayer directly and cannot supply arbitrary target contract or calldata.

---

## 12. Rate Limits, Upload Limits and Timeouts

Exact values must be configurable and tuned during demo testing. Proposed initial defaults:

| Operation | Initial limit (proposal) | Notes |
|---|---:|---|
| Upload size | 10 MiB per file | Confirm against actual test invoices and hosting limits |
| Upload requests | 10 per user/minute | Separate from general API limit |
| General API | 120 requests/user/minute | Read-heavy demo workload |
| Wallet challenge | 5 per user/10 minutes | Avoid signature spam |
| Review/decision submissions | 10 per user/minute | Idempotency still required |
| LLM extraction | Worker-controlled concurrency | Backoff and bounded retries |
| Relayer | Per signer + global spend/gas caps | Enforced by relayer service, not only API |

Do not treat proposed limits as fixed policy until team approval. Use connection/read timeouts for external RPC, storage and LLM calls. Async transaction confirmation is polled/indexed; do not hold an HTTP request open waiting for a block.

---

## 13. OpenAPI and Code Generation Requirements

FastAPI is the source of truth for the actual API schema. The implementation must publish `/openapi.json` in local/staging environments and generate a committed schema artifact for frontend codegen.

Required artifacts:

```text
packages/
  api-schema/
    openapi.json
    typescript/          # generated TypeScript DTO/client types
    README.md
```

Rules:
1. Use Pydantic request/response models for every endpoint.
2. Keep API DTOs separate from SQLAlchemy/database models.
3. Use explicit enums for state fields.
4. Use decimal strings for INR amounts.
5. Generate frontend types from OpenAPI; do not hand-maintain duplicated types.
6. Treat breaking API changes as versioned interface changes reviewed by all workstreams.
7. Add OpenAPI diff checks to CI.
8. Never publish internal relayer endpoints or admin-only internals as public browser operations.

---

## 14. Testing and Acceptance Criteria

### 14.1 Authentication and role tests

- Missing/expired/invalid JWT rejected.
- Inactive user rejected.
- Cross-organization workflow access denied.
- Submitter cannot call reviewer/approver/admin APIs.
- Reviewer can only review assigned cases.
- Approver cannot decide own submitted case.
- Reviewer cannot approve a case they reviewed/corrected.
- Admin API role operations require ADMIN and chain confirmation before showing role as active.

### 14.2 Document and workflow tests

- Accept valid PDF/JPG/PNG.
- Reject invalid MIME/signature, oversized and malformed files.
- Compute same SHA-256 for exact bytes across backend and browser test vector.
- Store only private storage path; do not expose it.
- Duplicate upload/idempotent retry does not create a second workflow.
- Registration transaction with wrong chain/contract/case/hash/sender is rejected.
- Worker starts only after confirmed `CaseCreated` event.

### 14.3 AI/review tests

- Valid extraction creates schema-valid result.
- Schema-invalid model output retries a bounded number of times and routes to manual entry.
- `REVIEWABLE_WARNING` supports confirmation.
- `CORRECTION_REQUIRED` cannot be confirmed; correction creates a new version and triggers revalidation.
- `HARD_BLOCK` cannot be cleared by confirmation or ordinary correction.
- Manual entry is validated through the same deterministic rules.
- No AI endpoint can invoke chain/relayer actions.

### 14.4 Approval/relayer tests

- Prepare endpoint rejects wrong role/tier/state/sync status.
- Typed data binds current case, decision, version, evidence, nonce and deadline.
- Stale evidence or expired signature is rejected.
- Relayer rejects wrong chain, contract, selector, signature, nonce or expired intent.
- Relayer has no contract role.
- Relayer submits exact signed intent unchanged.
- Direct tx hash is verified against chain, contract, sender, function and event.
- API never sets APPROVED/REJECTED before confirmed chain event.

### 14.5 Verification and reliability tests

- Document/data/validation hash match and mismatch paths.
- Event sequence illegal path returns FAIL.
- Historical role check returns UNAVAILABLE if event history cannot prove it; never false PASS.
- RPC outage returns pending/unavailable safely.
- Reorg/invalidation returns a previously confirmed event to pending/non-canonical as designed.
- Reconciler mismatch blocks new actions and returns `WORKFLOW_SYNC_DISCREPANCY`.
- Transaction outbox retry does not duplicate state transition.
- SSE reconnect resumes using event ID or polling fallback.

### 14.6 API definition of done

API v0.1 is accepted when:
1. OpenAPI schema is generated and committed.
2. All routes have request/response Pydantic models.
3. Auth, role and organization-scope tests pass.
4. Upload and document read APIs work with private storage.
5. Workflow read/list/version/timeline APIs work with seeded test data.
6. Review flow enforces dispositions and versioning.
7. Decision prepare/submit flow enforces the user's signer and the contract's state.
8. Relayer dispatch is internal and outbox-backed.
9. Transaction statuses distinguish pending/confirmed/failed.
10. Verification API returns individual checks with honest PASS/FAIL/PENDING/UNAVAILABLE states.

---

## 15. Implementation Order and Ownership

| Sequence | API work item | Owner | Depends on |
|---:|---|---|---|
| 1 | Pydantic DTOs, error envelope, auth middleware | Backend/AI | PRD + architecture |
| 2 | OpenAPI skeleton and generated TypeScript types | Backend + Frontend | DTOs |
| 3 | Upload, hashing, private storage, draft workflow | Backend/AI | DB migrations |
| 4 | Workflow register transaction verification | Backend + Blockchain | MST POC + ABI |
| 5 | Workflow list/detail/version/audit read models | Backend/AI | DB schema |
| 6 | Jobs/SSE progress | Backend/AI | Worker/checkpointer setup |
| 7 | Review prepare/submit + draft storage | Backend + Frontend + Blockchain | Review contract interface |
| 8 | Decision prepare/submit + relayer outbox | Backend + Blockchain + Frontend | EIP-712/BridgeKey POC |
| 9 | Verification read API | Backend + Frontend + Blockchain | ABI/events/hash vectors |
| 10 | Admin policy/vendor/role APIs | Backend + Blockchain | Role/policy decisions |
| 11 | OpenAPI contract tests and staging integration | All | Endpoints implemented |

### Team boundaries

- **Backend/AI workstream:** FastAPI routers, Pydantic DTOs, auth, data repositories, jobs, workflow services, read models, OpenAPI.
- **Blockchain workstream:** published ABI/event/typed-data artifacts, MST adapter compatibility findings, relayer internal API schema, contract-call verification expectations.
- **Frontend workstream:** generated API client, BridgeKey adapter, upload/review/approval/verification screens, SSE/polling behavior.
- **Architecture/integration owner:** reviews interface changes, keeps error/status vocabulary aligned and coordinates end-to-end acceptance.

---

## 16. Open Decisions and Required Schema Amendments

| ID | Decision / issue | Proposed handling | Blocks |
|---|---|---|---|
| API-01 | BridgeKey wallet-link signing method | Prove provider method/signature recovery in MST POC; use only verified method | Wallet linking |
| API-02 | EIP-712 typed-data compatibility | Enable `EIP712_RELAYED` only after BridgeKey + MST contract recovery test passes; otherwise use direct-wallet decision fallback | Relayer approval |
| API-03 | MST RPC/SDK choice | Evaluate `mst-sdk-python` and `@mstblockchain/mst-sdk`; fallback to standards-compatible RPC adapter after tests | Chain adapter |
| API-04 | MSTScan URL path format | Verify transaction/contract URL pattern; allow `explorer_url = null` until verified | Explorer links |
| API-05 | Confirmation depth/finality | Configure after MST Testnet observation/coordinator confirmation; never hard-code guessed value | Confirmed UI state |
| API-06 | Review draft persistence | Add `review_drafts` or equivalent protected pending-action schema; do not persist correction fields in generic job payload | Review/correction API |
| API-07 | Case ID derivation | Use one shared, versioned deterministic 32-byte derivation implementation and hash test vectors | Upload/register |
| API-08 | Historical role proof | Index `RoleGranted`/`RoleRevoked` events and check role at decision block; otherwise show `UNAVAILABLE` | Verification |
| API-09 | Upload size and retention | Proposed 10 MiB upload cap; final values and retention reviewed by team | Production-style upload |
| API-10 | Direct-wallet transaction verification | Prove receipt/sender/event decode behavior on MST POC | Registration and reviewer actions |
| API-11 | Endpoint error compatibility | Freeze API error enums with frontend before integration | Frontend integration |

### 16.1 Required database schema amendment

The current Database Schema v0.1 does not define a persisted record for a prepared human correction/manual-entry payload that must exist off-chain before a direct-wallet transaction is confirmed. Add a restricted `review_drafts` table (or an equivalently explicit table) with at least:

- `id`, `organization_id`, `workflow_id`, `reviewer_user_id`, `reviewer_wallet`;
- `action`, `base_data_version`, proposed fields JSON, proposed data hash, reason text/hash;
- `status` (`PREPARED`, `TX_PENDING`, `CONFIRMED`, `FAILED`, `EXPIRED`);
- expiry, associated transaction, timestamps and idempotency key.

The row must be protected by role/organization checks. Only after the chain event is confirmed should the system insert the canonical immutable `extracted_data` version and `review_actions` record. This avoids treating a not-yet-confirmed proposal as completed human review.

---

## 17. Version History

| Version | Date | Changes |
|---|---|---|
| 0.1 | October 31, 2026 | Initial API contract aligned with PRD v0.1, System Architecture v0.2, Database Schema v0.1, Smart Contract Specification v0.1 and MST guide. |

---

## 18. Sign-off

| Workstream | Reviewer | Status |
|---|---|---|
| Product / Architecture | Aditya Shelke | Pending team review |
| Backend / AI | Assigned team member | Pending team review |
| Blockchain | Assigned team member | Pending team review |
| Frontend | Assigned team member | Pending team review |

**Freeze rule:** API route and DTO names can be implemented in parallel as drafts, but the OpenAPI contract is not frozen until the database review-draft amendment, contract ABI/event interface and MST POC compatibility decisions are reviewed by all workstream owners.
