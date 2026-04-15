# Reputation Canister — Specification v1
> **Status:** Draft · **Last updated:** 2026-04-14  
> **Language:** Motoko · **Platform:** Internet Computer Protocol (ICP)

---

## 1. Overview & Vision

The **Reputation Canister** is the peer-inspection and trust backbone of the Hydra Mobility Protocol. It manages the full lifecycle of driver peer inspections — assigning inspectors, recording outcomes, distributing inspector fees, and exposing driver inspection status to other canisters via contract calls. We inspect only drivers and never riders. It has no subaccounts. It has no cycles. It has no DRV tokens. It is governed by the Governance canister. And Governance canister manages subaccounts, cycles for Reputation canister.

Key design properties:

- **Single responsibility:** The Reputation Canister is the only canister that creates, manages, and resolves driver inspection records. The Registry Canister delegates all inspection logic here.
- **Contract-call interface:** Other canisters (Registry, Governance) query inspection status via inter-canister contract calls — they never read inspection state directly.
- **Community-verified:** 1–3 randomly selected active drivers conduct a video call inspection via OpenChat before a new driver is admitted.
- **Privacy-first:** Inspectors only see the candidate's OpenChat handle and form data (name, car details). No identity documents are exchanged.
- **Fee distribution:** When a driver is activated, each confirming inspector receives `_inspectionFeePerInspector` ICP e8s automatically.

---

## 2. Glossary

| Term | Definition |
|------|-----------|
| **Inspection** | A peer-review process where 1–3 active drivers verify a candidate via OpenChat video call. |
| **Inspector** | An active driver with voting power, randomly selected to conduct a peer inspection video call. |
| **InspectionId** | A unique `Nat64` identifier for a single inspection record. |
| **InspectionStatus** | The state of a single inspection record: `#Pending`, `#Confirmed`, `#Rejected`. |
| **Candidate** | The driver being inspected (identified by `AccountId`). |
| **Confirmation threshold** | `_minConfirmations` — the minimum number of `#Confirmed` outcomes required to activate a candidate driver. |
| **Inspector fee** | `_inspectionFeePerInspector` ICP e8s paid to each confirming inspector when the candidate driver is activated. |

---

## 3. Definitions

## 3.1 Constants
```motoko
// constants.mo
module Constants {
    public let GOVERNANCE_CANISTER_ID : Principal = Principal.fromText("...");
    public let REGISTRY_CANISTER_ID   : Principal = Principal.fromText("...");

    // Inspection settings (adjustable via governance)
    public let DEFAULT_MIN_CONFIRMATIONS          : Nat = 1;    // Min inspector confirmations required
    public let DEFAULT_MAX_INSPECTORS             : Nat = 3;    // Max inspectors assigned per candidate
    public let DEFAULT_INSPECTION_FEE_PER_INSPECTOR : Nat = 0; // ICP e8s paid per confirming inspector (TBD via governance)
}
```

## 3.2 Types
```motoko
// types.mo
module Types {

    // ==========================================
    // --- 1. CORE ALIASES                    ---
    // ==========================================

    public type CanisterId   = Principal;
    public type AccountId    = Principal;
    public type Timestamp    = Nat64;   // nanoseconds since Unix epoch (ICP Ledger standard)
    public type InspectionId = Nat64;


    // ==========================================
    // --- 2. INSPECTION TYPES                ---
    // ==========================================

    public type InspectionStatus = {
        #Pending;    // Inspector has been assigned; video call not yet completed
        #Confirmed;  // Inspector confirmed the candidate passes
        #Rejected;   // Inspector rejected the candidate
    };

    /// One peer inspection record — one inspector, one candidate driver.
    public type DriverInspectionDao = {
        id             : InspectionId;
        candidateId    : AccountId;       // The driver being inspected
        inspectorId    : AccountId;       // The active driver conducting the inspection
        status         : InspectionStatus;
        assignedAt     : Timestamp;
        completedAt    : ?Timestamp;      // Null until inspector submits result
    };

    /// Submitted by an inspector to record their inspection outcome.
    public type DriverInspectionConfirmContract = {
        inspectionId : InspectionId;
        outcome      : { #Confirmed; #Rejected };
    };

    public type DriverInspectionConfirmResponse = {
        #ok  : { inspectionId : InspectionId; candidateActivated : Bool };
        #err : Text;
    };

    /// Returned to Registry canister on status query.
    public type InspectionStatusContract = {
        candidateId         : AccountId;
        inspectionIds       : [InspectionId];
        confirmedCount      : Nat;
        rejectedCount       : Nat;
        pendingCount        : Nat;
        candidateActivated  : Bool;   // true if confirmedCount >= _minConfirmations
    };

    public type InspectionStatusResponse = {
        #ok  : InspectionStatusContract;
        #err : Text;
    };


    // ==========================================
    // --- 3. SUPPORT TYPES                   ---
    // ==========================================

    /// Snapshot of call counters. Exposed via supportCounters() for external monitoring.
    /// Counters are reset every 24H by worker.mo.
    public type SupportCounters = {
        inspectionAssigned : Nat;
        inspectionConfirmed: Nat;
        inspectionRejected : Nat;
        statusQueries      : Nat;
    };
}
```

```motoko
// types.governance.mo
module Types {

    // ==========================================
    // --- 1. REPUTATION GOVERNANCE TYPES     ---
    // ==========================================

    /// Payload for governanceUpdateInspectorCount — adjusts required confirmation count.
    public type ReputationGovernanceUpdateInspectorCountContract = {
        minConfirmations : Nat;  // Min inspectors required (1–3)
        maxInspectors    : Nat;  // Max inspectors assigned per candidate (1–3)
    };

    public type ReputationGovernanceUpdateInspectorCountResponse = {
        #ok  : { minConfirmations : Nat; maxInspectors : Nat };
        #err : Text;
    };

    /// Payload for governanceUpdateInspectionFee — adjusts inspector payout.
    public type ReputationGovernanceUpdateInspectionFeeContract = {
        inspectionFeePerInspector : Nat;  // ICP e8s paid to each confirming inspector
    };

    public type ReputationGovernanceUpdateInspectionFeeResponse = {
        #ok  : { inspectionFeePerInspector : Nat };
        #err : Text;
    };
}
```

---

## 4. Internal State Variables

```motoko
// state.mo
inspections : TrieMap<InspectionId, DriverInspectionDao>
```

---

```motoko
// main.mo
// --- Call counters (reset every 24H by worker.mo) ---
var _counterInspectionAssigned  : Nat = 0;
var _counterInspectionConfirmed : Nat = 0;
var _counterInspectionRejected  : Nat = 0;
var _counterStatusQueries       : Nat = 0;

// --- Inspection settings (adjustable via governance) ---
var _minConfirmations           : Nat = 1;   // Min inspector confirmations required to activate driver
var _maxInspectors              : Nat = 3;   // Max inspectors assigned per candidate
var _inspectionFeePerInspector  : Nat = 0;   // ICP e8s paid per confirming inspector (TBD via governance)

// --- Version (incremented on every canister upgrade) ---
// Used in postupgrade migrations to detect state schema changes.
var _version : Nat = 1;
```

---

## 5. Public API

### 5.1 Health

```
// query — no cycle cost
getStatus() -> #ok
```
Returns `#ok` if the canister is alive and responsive. No fields are returned.

---

### 5.2 Assign Inspections

```
// Internal. msg.caller must match Registry canister principal.
// Called by Registry after a candidate driver passes liveness check (status → #PendingInspection).
// Registry provides the candidate AccountId and the list of eligible inspector AccountIds.
inspectionAssign(candidateId : AccountId, inspectors : [AccountId]) -> Result<[InspectionId], Text>

  Validators (in order):
    1. msg.caller must match REGISTRY_CANISTER_ID.
    2. inspectors is not empty and length <= _maxInspectors.
    3. candidateId has no existing #Pending inspections (idempotency guard).

  On success:
    - Create a DriverInspectionDao for each inspector with status = #Pending.
    - Return the list of created InspectionIds.

  On any validation failure:
    - Return #err with descriptive Text.
```

---

### 5.3 Inspector Confirm

```
// Public. msg.caller must be an assigned inspector.
// Step 3 of driver onboarding — inspector records the outcome of their video call inspection.
driverInspectionConfirm(DriverInspectionConfirmContract) -> DriverInspectionConfirmResponse

  Validators (in order):
    1. msg.caller is assigned as inspector for the given inspectionId.
    2. Inspection status is #Pending.

  On success:
    - Update DriverInspectionDao.status = contract.outcome.
    - Count all #Confirmed inspections for the candidate.
    - If confirmedCount >= _minConfirmations:
        · Set candidateActivated = true.
        · Notify Registry canister (inter-canister call) to set driver.status = #Active.
        · Pay _inspectionFeePerInspector ICP e8s to each confirming inspector (if fee > 0).
    - If all inspectors responded and confirmedCount < _minConfirmations:
        · Keep candidateActivated = false (future governance or retry logic TBD).
    - Return DriverInspectionConfirmResponse.

  On any validation failure:
    - Return #err with descriptive Text.
```

---

### 5.4 Inspection Status (Registry contract call)

```
// Internal. msg.caller must match Registry canister principal.
// Called by Registry to query the current inspection status for a candidate driver.
inspectionStatus(candidateId : AccountId) -> InspectionStatusResponse

  Validators (in order):
    1. msg.caller must match REGISTRY_CANISTER_ID.

  On success:
    - Return InspectionStatusContract with full counts and activation flag.

  On any validation failure:
    - Return #err with descriptive Text.
```

---

### 5.5 Support

```
// Public query. No authentication required. Used for external monitoring and metrics dashboards.
supportCounters() -> SupportCounters
```

Returns a snapshot of all call counters. Counters are reset every 24H by `worker.mo`.

---

### 5.6 Governance

```
// Called from Governance canister to adjust inspector count requirements.
// msg.caller must match governance canister principal.
governanceUpdateInspectorCount(ReputationGovernanceUpdateInspectorCountContract) -> ReputationGovernanceUpdateInspectorCountResponse

  Validators:
  1. msg.caller must match governance canister principal.
  2. minConfirmations >= 1 and maxInspectors <= 3 and minConfirmations <= maxInspectors.

  Logic:
  - Update _minConfirmations and _maxInspectors in state.


// Called from Governance canister to adjust the fee paid to each confirming inspector.
// msg.caller must match governance canister principal.
governanceUpdateInspectionFee(ReputationGovernanceUpdateInspectionFeeContract) -> ReputationGovernanceUpdateInspectionFeeResponse

  Validators:
  1. msg.caller must match governance canister principal.

  Logic:
  - Update _inspectionFeePerInspector in state.
```

---

## 6. Business Rules & Invariants

### 6.1 Inspector Selection Rules

| Rule | Detail |
|------|--------|
| Pool | Only `#Active` drivers from the same region are eligible (selected by Registry, passed to `inspectionAssign`) |
| Selection | Ordered by voting power (highest first); top `_maxInspectors` selected |
| Count | 1 to `_maxInspectors` inspectors assigned (default max: 3) |
| Confirmation threshold | `_minConfirmations` confirmations required (default: 1) |
| Form lock | Driver form data is read-only once submitted; inspectors see exactly what was filed |
| No documents | Inspectors do NOT check any identity documents; visual and language check only |

---

### 6.2 Inspection Fees

| Rule | Detail |
|------|--------|
| Inspector fee | `_inspectionFeePerInspector` ICP e8s paid per confirming inspector. Default 0. |
| Change authority | Governance canister only |
| Distribution trigger | Automatically on driver activation (when `confirmedCount >= _minConfirmations`) |

> **Note:** Mirror Gateway `DepositDao` pattern if non-zero fees are introduced.

---

### 6.3 Canister Interaction

| Direction | Method | Purpose |
|-----------|--------|---------|
| Registry → Reputation | `inspectionAssign(candidateId, inspectors[])` | After liveness check passes; Registry delegates inspection creation |
| Registry → Reputation | `inspectionStatus(candidateId)` | Registry queries inspection state for status decisions |
| Reputation → Registry | `registryActivateDriver(candidateId)` | Reputation notifies Registry when `confirmedCount >= _minConfirmations` |

---

## 7. Bootstrap Flows

### 7.1 Driver Inspection Flow

```
1. Registry canister calls inspectionAssign(candidateId, [inspector1, inspector2, ...]) → Reputation
   Reputation creates DriverInspectionDao for each inspector with status = #Pending.

2. Frontend notifies each inspector via their OpenChat handle (off-chain notification layer).

3. Each inspector initiates a video call with the candidate via OpenChat.
   Inspector verifies:
     · Candidate speaks required language(s).
     · Car is physically the car described in the registration form.
     · No documents are requested; no privacy data is collected.

4. Inspector submits outcome:
   driverInspectionConfirm({ inspectionId, outcome: #Confirmed | #Rejected }) → Reputation

5. If confirmedCount >= _minConfirmations:
   · Reputation calls registryActivateDriver(candidateId) → Registry
   · Registry sets driver.status = #Active, matchingCanisterId from regionMappings[region].
   · Reputation pays _inspectionFeePerInspector ICP e8s to each confirming inspector (if fee > 0).

✅ Driver is activated and can receive ride requests in their mapped region.
```

---

### 7.2 Registry Status Query Flow

```
Registry calls inspectionStatus(candidateId) → Reputation
Reputation returns InspectionStatusContract:
  {
    candidateId         = candidateId,
    inspectionIds       = [id1, id2, ...],
    confirmedCount      = N,
    rejectedCount       = M,
    pendingCount        = K,
    candidateActivated  = (N >= _minConfirmations)
  }

Registry uses candidateActivated flag to determine driver lifecycle state.
```

---

## 8. Architecture

```
canister_reputation/
├── main.mo               — Actor entry point. Exposes all public endpoints. Holds _version.
├── state.mo              — Stable state definitions. All maps and counters. Upgraded via postupgrade.
├── constants.mo          — Protocol-wide constants (registry canister id, inspection defaults, etc.)
├── types.mo              — All shared type definitions (module Types from Section 3).
├── types.governance.mo   — Governance contract types.
├── logic.mo              — All business logic (inspection assignment, confirmation, fee distribution).
├── governance.mo         — Governance canister interface and logic.
├── validators.mo         — Pure validation functions called from main.mo before endpoint logic.
├── math.mo               — Pure helper functions (timestamp arithmetic, etc.)
└── worker.mo             — Heartbeat-driven background tasks:
                          · Reset call-rate counters every 24H
                          · TODO: expire stale inspections (candidates in #PendingInspection; retry logic TBD)

tests/
├── tests.[module_name].mops       — Unit tests (per module)
└── integration.[flow].mops        — End-to-end integration tests simulating inspection flows
```

**Key technology choices:**
- `Tier.Map` — for ordered, stable `TrieMap`-compatible state storage in `state.mo`
- `--enhanced-orthogonal-persistence` — enables automatic stable memory management across upgrades without manual `preupgrade`/`postupgrade` serialisation of simple types.

---

## 9. Upgrade & Migration Strategy

- `_version : Nat` is stored in stable state in `main.mo` and incremented on every canister upgrade.
- `postupgrade` system hook is used to perform explicit state migrations when the schema changes.
- Migration logic reads `_version` to determine which migration steps to apply.
- All stable variables use `--enhanced-orthogonal-persistence` where possible to minimise manual migration surface area.

```motoko
// Example pattern in main.mo
system func postupgrade() {
    if (_version == 1) {
        // apply v1 → v2 migration
        _version := 2;
    };
};
```

---

## 10. Open Items / Future Work

| Item | Priority | Notes |
|------|----------|-------|
| Inspector notification mechanism | High | Frontend must notify inspectors via OpenChat; on-chain trigger to be defined |
| Inspector fee distribution | High | Mirror Gateway `DepositDao` pattern when `_inspectionFeePerInspector > 0` |
| Inspection expiry / retry logic | Medium | What happens if all inspectors reject or fail to respond? Retry flow TBD |
| `ProposalTarget` extension | High | Add `#Reputation` variant to Governance canister `ProposalTarget` type |
| Monitoring dashboard integration | Low | `supportCounters()` endpoint is ready; consumer to be defined |
