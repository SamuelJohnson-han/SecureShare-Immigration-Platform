 # SecureShare Immigration Platform

## Overview

SecureShare is a Web3 decentralized application (dApp) built on the Stacks blockchain using Clarity smart contracts. The platform empowers users (e.g., immigrants, travelers, or citizens) to maintain full control over their personal data when interacting with immigration authorities. Users can store encrypted immigration-related data (such as visas, passports, travel history, or residency proofs) off-chain (e.g., on IPFS for decentralization) and use smart contracts to grant granular, revocable access to specific data fields. This solves real-world problems like:

- **Privacy Erosion**: Traditional systems force users to share excessive data with authorities, leading to potential misuse, data breaches, or surveillance.
- **Lack of Consent and Revocation**: Once data is shared, users can't easily revoke access or audit who viewed it.
- **Inefficiency and Trust Issues**: Centralized databases are prone to hacks and require blind trust in intermediaries; SecureShare uses blockchain for transparency and immutability.
- **Compliance Challenges**: Authorities need verifiable data without overreach, while users want minimal disclosure (zero-knowledge proofs could be integrated in future versions).

The platform involves authorities registering and requesting data, users approving/rejecting requests, and an audit trail for accountability. Data sharing is pseudonymous, with optional KYC for authorities. Incentives could include STX micropayments for data sharing.

This project uses 6 core smart contracts written in Clarity:
1. **UserRegistry.clar**: Handles user registration and profile management.
2. **AuthorityRegistry.clar**: Registers and verifies immigration authorities.
3. **DataVault.clar**: Manages storage of encrypted data hashes (e.g., IPFS CIDs).
4. **AccessControl.clar**: Defines and enforces access permissions with revocation.
5. **RequestManager.clar**: Processes data requests from authorities to users.
6. **AuditLog.clar**: Records all interactions for transparency and disputes.

The dApp frontend (not included here; assume React + Hiro Wallet) interacts with these contracts via the Stacks.js library.

## Real-World Impact

- **For Users**: Enhances privacy by allowing selective disclosure (e.g., share only visa expiry date, not full passport scan). Revoke access if authorities overstep.
- **For Authorities**: Streamlines verification with tamper-proof data, reducing fraud. Requests are logged, ensuring accountability.
- **Broader Problems Solved**: Reduces identity theft risks in immigration processes, supports GDPR-like compliance in a decentralized way, and could extend to other sectors like healthcare or finance.
- **Web3 Benefits**: No single point of failure; users own their data via wallets; smart contracts automate trustless sharing.

## Architecture

- **Off-Chain Components**:
  - Data encrypted and stored on IPFS (or similar decentralized storage).
  - Frontend dApp for user/authority interfaces.
- **On-Chain Components**: The 6 Clarity contracts deployed on Stacks.
- **Workflow**:
  1. User registers and uploads encrypted data hash to DataVault.
  2. Authority registers and sends a request via RequestManager.
  3. User approves/denies via AccessControl, granting temporary access.
  4. Authority views decrypted data off-chain (user shares key via secure channel).
  5. All actions logged in AuditLog.
- **Security**: Use Clarity's safety features (no reentrancy, explicit errors). Data is never stored plaintext on-chain.

## Installation and Deployment

### Prerequisites
- Stacks CLI (install via `npm install -g @stacks/cli`).
- Hiro Wallet for testing.
- Node.js for any scripts.

### Steps
1. Clone the repo: `git clone <repo-url>`.
2. Navigate to the contracts directory: `cd contracts`.
3. Deploy contracts using Stacks CLI:
   - `stacks deploy UserRegistry.clar --testnet` (repeat for each contract, handling dependencies).
4. Set up dependencies in deployment scripts (e.g., UserRegistry must be deployed before DataVault).
5. Build and run the frontend (assuming you add one).
6. Test with sample data: Register a user, upload a mock IPFS hash, simulate a request.

### Dependencies
- Clarity standard libraries (e.g., for principals, maps).
- No external packages; pure Clarity.

## Smart Contracts

Below is a high-level description of each contract. Full code is in the `contracts/` directory (simulated here as code blocks).

### 1. UserRegistry.clar
Registers users and stores basic profiles (e.g., principal to user ID).

```clarity
;; UserRegistry.clar

(define-map users principal { user-id: uint, registered-at: uint })

(define-public (register-user (user-id uint))
  (let ((caller tx-sender))
    (if (is-none (map-get? users caller))
      (begin
        (map-set users caller { user-id: user-id, registered-at: block-height })
        (ok true))
      (err u100))))  ;; Already registered

(define-read-only (get-user (user principal))
  (map-get? users user))
```

### 2. AuthorityRegistry.clar
Registers authorities (e.g., government entities) with verification (mock KYC via admin).

```clarity
;; AuthorityRegistry.clar

(define-constant admin 'ST1PQHQKV0RJXZHJYZRCKZ7TGM2CJ0XAV1R6RNMJB)  ;; Example admin principal

(define-map authorities principal { auth-id: uint, verified: bool })

(define-public (register-authority (auth-id uint))
  (let ((caller tx-sender))
    (if (is-eq tx-sender admin)  ;; Only admin can verify
      (begin
        (map-set authorities caller { auth-id: auth-id, verified: true })
        (ok true))
      (err u200))))  ;; Not admin

(define-read-only (is-authority (auth principal))
  (match (map-get? authorities auth)
    some-entry (ok (get verified some-entry))
    none (err u201)))
```

### 3. DataVault.clar
Stores data hashes per user (e.g., IPFS CID for encrypted files).

```clarity
;; DataVault.clar

(define-map data-vault { user: principal, data-id: uint } { hash: (buff 32), timestamp: uint })

(define-public (store-data (data-id uint) (hash (buff 32)))
  (let ((caller tx-sender))
    (map-set data-vault { user: caller, data-id: data-id } { hash: hash, timestamp: block-height })
    (ok true)))

(define-read-only (get-data (user principal) (data-id uint))
  (map-get? data-vault { user: user, data-id: data-id }))
```

### 4. AccessControl.clar
Manages permissions: Grant, revoke access to specific data.

```clarity
;; AccessControl.clar

(define-map permissions { user: principal, auth: principal, data-id: uint } { granted: bool, expiry: uint })

(define-public (grant-access (auth principal) (data-id uint) (expiry uint))
  (let ((caller tx-sender))
    (map-set permissions { user: caller, auth: auth, data-id: data-id } { granted: true, expiry: (+ block-height expiry) })
    (ok true)))

(define-public (revoke-access (auth principal) (data-id uint))
  (let ((caller tx-sender))
    (map-delete permissions { user: caller, auth: auth, data-id: data-id })
    (ok true)))

(define-read-only (check-access (user principal) (auth principal) (data-id uint))
  (match (map-get? permissions { user: user, auth: auth, data-id: data-id })
    some-permission (if (and (get granted some-permission) (> (get expiry some-permission) block-height))
                      (ok true)
                      (err u300))
    none (err u301)))
```

### 5. RequestManager.clar
Handles requests from authorities to users.

```clarity
;; RequestManager.clar

(define-map requests { request-id: uint } { auth: principal, user: principal, data-id: uint, status: (string-ascii 10) })

(define-public (send-request (request-id uint) (user principal) (data-id uint))
  (let ((caller tx-sender))
    (try! (contract-call? .AuthorityRegistry is-authority caller))  ;; Verify authority
    (map-set requests { request-id: request-id } { auth: caller, user: user, data-id: data-id, status: "pending" })
    (ok true)))

(define-public (approve-request (request-id uint))
  (let ((caller tx-sender))
    (match (map-get? requests { request-id: request-id })
      some-req (if (is-eq (get user some-req) caller)
                 (begin
                   (map-set requests { request-id: request-id } (merge some-req { status: "approved" }))
                   ;; Grant access automatically
                   (try! (contract-call? .AccessControl grant-access (get auth some-req) (get data-id some-req) u100))  ;; 100 blocks expiry
                   (ok true))
                 (err u400))
      none (err u401))))

(define-read-only (get-request (request-id uint))
  (map-get? requests { request-id: request-id }))
```

### 6. AuditLog.clar
Logs all events for auditing.

```clarity
;; AuditLog.clar

(define-map logs uint { event: (string-ascii 50), actor: principal, timestamp: uint, details: (buff 128) })

(define-data-var log-counter uint u0)

(define-private (add-log (event (string-ascii 50)) (details (buff 128)))
  (let ((id (var-get log-counter)))
    (map-set logs id { event: event, actor: tx-sender, timestamp: block-height, details: details })
    (var-set log-counter (+ id u1))
    (ok id)))

;; Example: Call from other contracts, e.g., after grant-access
(define-public (log-event (event (string-ascii 50)) (details (buff 128)))
  (add-log event details))

(define-read-only (get-log (log-id uint))
  (map-get? logs log-id))
```

## Usage Examples

- Register as user: Call `register-user` in UserRegistry.
- Store data: Upload to IPFS off-chain, then call `store-data`.
- Authority requests: Call `send-request`.
- User approves: Call `approve-request`, which triggers access grant.
- Check access: Authority calls `check-access` before viewing.
- Audit: Query logs for any event.

## Testing

Use Stacks testnet. Write unit tests in Clarity (e.g., assert success/failure).

## Future Enhancements

- Integrate zero-knowledge proofs for verifiable computations without revealing data.
- Add governance token for community decisions.
- Mobile app integration for easy data uploads.

## License

MIT License. See LICENSE file.