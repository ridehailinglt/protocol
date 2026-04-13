# Registry Canister — Specification v1
> **Status:** Draft · **Last updated:** 2026-04-13  
> **Language:** Motoko · **Platform:** Internet Computer Protocol (ICP)

---

## 1. Overview & Vision

The **Registry Canister** is the identity and onboarding backbone of the Hydra Mobility Protocol. It manages driver and rider registration, performs liveness verification, and routes users to the correct city-level Matching canister via deterministic regional mapping.

Key design properties:

- **Privacy-first:** No documents, no personal data stored on-chain. Only a liveness proof hash from DecideAI is stored per driver. Riders store no sensitive data at all.
- **Community-verified (Driver):** After liveness check, 1–3 randomly selected active drivers conduct a video call inspection via OpenChat before the new driver is admitted.
- **Free for Riders:** Rider registration requires only a simple form — no liveness check, no inspection, no fee.
- **Internet Identity 2:** All users authenticate via Internet Identity 2. No username/password.
- **Deterministic routing:** Each registered driver/rider is mapped to a geographic region (city + country) which determines their assigned Matching canister.
- **Role-separated:** Drivers and riders are distinct identity types with separate registration flows, lifecycle states, and data models.

---

## 2. Glossary

| Term | Definition |
|------|-----------|
| **Driver** | A registered transport provider who has passed liveness check, peer inspection, and is mapped to a regional Matching canister. |
| **Rider** | A registered end-user who has filled the registration form and can request rides in their mapped region. No liveness or document checks. |
| **Liveness Hash** | A cryptographic proof-of-humanity hash issued by DecideAI. No documents or raw personal data are ever stored on-chain. |
| **Liveness Check** | A proof-of-humanity verification performed via DecideAI. Required for drivers only. |
| **Peer Inspection** | A video call via OpenChat where 1–3 active drivers verify: (a) the candidate speaks required languages, (b) the car matches the form. No documents checked. |
| **Inspector** | An active driver with voting power, randomly selected to conduct a peer inspection video call. |
| **Internet Identity 2** | The authentication provider used by all participants. No username/password flows. |
| **Region** | A city + country pair. Deterministically maps a user to a specific Matching canister. |
| **DriverRegistryStatus** | The lifecycle state of a driver record. Pending → PendingLiveness → PendingInspection → Active → Suspended / Banned. |
| **RiderRegistryStatus** | The lifecycle state of a rider record. Active → Suspended / Banned. |

---

## 3. Definitions

## 3.1 Constants
```motoko
// constants.mo
module Constants {
    public let GOVERNANCE_CANISTER_ID : Principal = Principal.fromText("...");
    public let GATEWAY_CANISTER_ID    : Principal = Principal.fromText("...");

    // KYC provider
    public let KYC_PROVIDER_CANISTER_ID : Principal = Principal.fromText("...");  // DecideAI

    // Registration
    public let DRIVER_REGISTRATION_FEE : Nat64 = 0;  // TODO: decide fee via governance
    public let RIDER_REGISTRATION_FEE  : Nat64 = 0;  // TODO: decide fee via governance

    // Activity lifecycle
    public let INACTIVE_THRESHOLD_DAYS   : Nat64 = 90 * 24 * 60 * 60;   // 90 days in seconds
    public let SUSPENSION_EXPIRY_PERIOD  : Nat64 = 30 * 24 * 60 * 60;   // 30 days in seconds

    // Limits, must be adjusted buy country/city
    public let MAX_ACTIVE_DRIVERS_PER_WEEK : Nat64 = 500;
    public let MAX_ACTIVE_DRIVERS_PER_MONTH : Nat64 = 1000;

    public let MAX_FOREIGNERS_REGISTERED_PERCENT : Float = 20.0;

    //Cleanup
    public let INACTIVE_DRIVER_PRUNING_PERIOD : Nat64 = 90 * 24 * 60 * 60;   // 90 days in seconds
    public let INACTIVE_DRIVER_MINIMAL_PERCENTVOTING_POWER : Float = 0.1; // 0.1% of total voting power
}
```

## 3.2 Types
```motoko
// types.mo
module Types {

    // ==========================================
    // --- 1. CORE ALIASES                    ---
    // ==========================================

    public type CanisterId    = Principal;
    public type AccountId     = Principal;
    public type Timestamp     = Nat64;   // nanoseconds since Unix epoch (ICP Ledger standard)
    public type LivenessHash  = Blob;    // DecideAI proof-of-humanity hash. No raw document data stored.
    public type Region        = { city : Text; country : Text };


    // ==========================================
    // --- 2. REGISTRY STATUS TYPES           ---
    // ==========================================

    public type DriverRegistryStatus = {
        #Pending;            // Form submitted; awaiting liveness check on DecideAI
        #PendingLiveness;    // Liveness check initiated; awaiting DecideAI confirmation
        #PendingInspection;  // Liveness passed; awaiting peer inspection video calls
        #Active;             // Fully verified and operational
        #Suspended;          // Temporarily restricted — set by governance
        #Banned;             // Permanently removed — set by governance
    };

    public type RiderRegistryStatus = {
        #Active;     // Registered and operational
        #Suspended;  // Temporarily restricted — set by governance
        #Banned;     // Permanently removed — set by governance
    };


    // ==========================================
    // --- 3. DRIVER TYPES                    ---
    // ==========================================

    /// Internal state record. Stored per registered driver.
    public type DriverDao = {
        owner              : AccountId;
        livenessHash       : LivenessHash;   // DecideAI proof-of-humanity hash. No raw personal data stored.
        // --- Registration form fields ---
        firstName          : Text;
        lastName           : Text;
        email              : Text;
        phone              : Text;
        address            : Text;
        carMake            : Text;
        carModel           : Text;
        carYear            : Nat;
        carPlate           : Text;
        openChatHandle     : Text;           // Required: used by inspectors to initiate video calls
        // --- Region & routing ---
        region             : Region;         // City + country used to map to a Matching canister
        matchingCanisterId : CanisterId;     // Assigned Matching canister for this driver's region
        // --- Lifecycle ---
        status             : DriverRegistryStatus;
        registeredAt       : Timestamp;
        lastActiveAt       : Timestamp;
        suspendedUntil     : ?Timestamp;     // Null if not suspended
        suspendReason      : Text;           // Empty if not suspended
        // --- Inspection ---
        inspectionIds      : [InspectionId]; // IDs of peer inspection records for this driver
    };

    /// Public-facing record shared with other canisters.
    public type DriverContract = {
        owner              : AccountId;
        region             : Region;
        matchingCanisterId : CanisterId;
        status             : DriverRegistryStatus;
        registeredAt       : Timestamp;
    };

    /// Submitted by the driver to begin registration.
    public type DriverRegisterContract = {
        firstName      : Text;
        lastName       : Text;
        email          : Text;
        phone          : Text;
        address        : Text;
        carMake        : Text;
        carModel       : Text;
        carYear        : Nat;
        carPlate       : Text;
        openChatHandle : Text;
        region         : Region;
    };

    public type DriverRegisterResponse = {
        #ok  : DriverContract;
        #err : Text;
    };


    // ==========================================
    // --- 4. RIDER TYPES                     ---
    // ==========================================

    /// Internal state record. Stored per registered rider.
    /// Rider registration is free and requires no liveness check or peer inspection.
    public type RiderDao = {
        owner              : AccountId;
        // --- Registration form fields ---
        firstName          : Text;
        lastName           : Text;
        email              : Text;
        phone              : Text;
        // --- Region & routing ---
        region             : Region;
        matchingCanisterId : CanisterId;
        // --- Lifecycle ---
        status             : RiderRegistryStatus;
        registeredAt       : Timestamp;
        lastActiveAt       : Timestamp;
        suspendedUntil     : ?Timestamp;
        suspendReason      : Text;
    };

    /// Public-facing record shared with other canisters.
    public type RiderContract = {
        owner              : AccountId;
        region             : Region;
        matchingCanisterId : CanisterId;
        status             : RiderRegistryStatus;
        registeredAt       : Timestamp;
    };

    /// Submitted by the rider to begin registration. Simple form — no documents, no liveness.
    public type RiderRegisterContract = {
        firstName : Text;
        lastName  : Text;
        email     : Text;
        phone     : Text;
        region    : Region;
    };

    public type RiderRegisterResponse = {
        #ok  : RiderContract;
        #err : Text;
    };


    // ==========================================
    // --- 5. DRIVER INSPECTION TYPES         ---
    // ==========================================

    public type InspectionId = Nat64;

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
        #ok  : { inspectionId : InspectionId; candidateStatus : DriverRegistryStatus };
        #err : Text;
    };


    // ==========================================
    // --- 6. REGION MAPPING TYPES            ---
    // ==========================================

    /// Maps a region (city + country) to the Principal of the responsible Matching canister.
    public type RegionMappingDao = {
        region             : Region;
        matchingCanisterId : CanisterId;
        isActive           : Bool;
        registeredAt       : Timestamp;
    };

    public type RegionMappingContract = {
        region             : Region;
        matchingCanisterId : CanisterId;
    };


    // ==========================================
    // --- 7. SUPPORT TYPES                   ---
    // ==========================================

    /// Snapshot of call counters. Exposed via supportCounters() for external monitoring.
    /// Counters are reset every 24H by worker.mo.
    public type SupportCounters = {
        driverRegister     : Nat;
        riderRegister      : Nat;
        driverLookup       : Nat;
        riderLookup        : Nat;
        inspectionAssigned : Nat;
        inspectionConfirmed: Nat;
    };
}
```

```motoko
// types.governance.mo
module Types {

    // ==========================================
    // --- 1. REGISTRY GOVERNANCE TYPES       ---
    // ==========================================

    /// Payload for governanceUpdateFees — adjusts registration fees.
    public type RegistryGovernanceUpdateFeesContract = {
        driverRegistrationFee : Nat;
        riderRegistrationFee  : Nat;
    };

    public type RegistryGovernanceUpdateFeesResponse = {
        #ok  : { fees : RegistryGovernanceUpdateFeesContract };
        #err : Text;
    };

    /// Payload for governanceSuspendDriver — temporarily restricts a driver.
    public type RegistryGovernanceSuspendDriverContract = {
        driver         : AccountId;
        suspendedUntil : Timestamp;
        reason         : Text;
    };

    public type RegistryGovernanceSuspendDriverResponse = {
        #ok  : { driver : AccountId };
        #err : Text;
    };

    /// Payload for governanceBanDriver — permanently removes a driver.
    public type RegistryGovernanceBanDriverContract = {
        driver : AccountId;
        reason : Text;
    };

    public type RegistryGovernanceBanDriverResponse = {
        #ok  : { driver : AccountId };
        #err : Text;
    };

    /// Payload for governanceUpdateInspectorCount — adjusts required confirmation count.
    public type RegistryGovernanceUpdateInspectorCountContract = {
        minConfirmations : Nat;  // Min inspectors required (1–3)
        maxInspectors    : Nat;  // Max inspectors assigned per candidate (1–3)
    };

    public type RegistryGovernanceUpdateInspectorCountResponse = {
        #ok  : { minConfirmations : Nat; maxInspectors : Nat };
        #err : Text;
    };

    /// Payload for governanceUpdateInspectionFee — adjusts inspector payout.
    public type RegistryGovernanceUpdateInspectionFeeContract = {
        inspectionFeePerInspector : Nat;  // ICP e8s paid to each confirming inspector
    };

    public type RegistryGovernanceUpdateInspectionFeeResponse = {
        #ok  : { inspectionFeePerInspector : Nat };
        #err : Text;
    };
}
```

---

## 4. Internal State Variables

```motoko
// state.mo
drivers        : TrieMap<AccountId, DriverDao>
riders         : TrieMap<AccountId, RiderDao>
regionMappings : TrieMap<Text, RegionMappingDao>    // key: "city:country"
inspections    : TrieMap<InspectionId, DriverInspectionDao>
```

---

```motoko
// main.mo
// --- Call counters (reset every 24H by worker.mo) ---
var _counterDriverRegister      : Nat = 0;
var _counterRiderRegister       : Nat = 0;
var _counterDriverLookup        : Nat = 0;
var _counterRiderLookup         : Nat = 0;
var _counterInspectionAssigned  : Nat = 0;
var _counterInspectionConfirmed : Nat = 0;

// --- Inspection settings (adjustable via governance) ---
var _minConfirmations        : Nat = 1;          // Min inspector confirmations required to activate driver
var _maxInspectors           : Nat = 3;          // Max inspectors assigned per candidate
var _inspectionFeePerInspector : Nat = 0;        // ICP e8s paid per confirming inspector (TBD via governance)

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

### 5.2 Driver Registration

```
// Public. No caller check. Called by a candidate driver.
// Step 1 of driver onboarding — submits the initial registration form.
driverRegister(DriverRegisterContract) -> DriverRegisterResponse

  Validators (in order):
    1. msg.caller is not already registered as a driver (any status).
    2. region has an active mapping in regionMappings.
    3. openChatHandle is not empty (required for inspector-initiated video calls).

  On success:
    - Register driver with status = #Pending.
    - Form data (name, car details, region, OpenChat handle) stored in DriverDao.
    - Form editing is disabled after this point — driver cannot modify submitted data.
    - Return DriverContract (status = #Pending).

  On any validation failure:
    - Return #err with descriptive Text.
```

---

### 5.3 Driver Liveness Submission

```
// Public. msg.caller must match a driver with status = #Pending.
// Step 2 — driver submits their DecideAI liveness hash.
driverSubmitLiveness(livenessHash : LivenessHash) -> Result<(), Text>

  Validators (in order):
    1. msg.caller is registered as a driver with status = #Pending.
    2. livenessHash is not already registered by another driver (prevents duplicate proof).
    3. Calls KYC_PROVIDER_CANISTER_ID with livenessHash; must return verified = true.

  On success:
    - Update driver.livenessHash.
    - Update driver.status = #PendingInspection.
    - Randomly select 1–_maxInspectors active drivers with highest voting power from the same region.
    - Create a DriverInspectionDao for each selected inspector.
    - Notify each inspector's OpenChat handle (off-chain; handled by frontend/notification layer).
    - Return #ok.

  On any validation failure:
    - Return #err with descriptive Text.
```

---

### 5.4 Inspector Confirm

```
// Public. msg.caller must be an assigned inspector.
// Step 3 — inspector records the outcome of their video call inspection.
driverInspectionConfirm(DriverInspectionConfirmContract) -> DriverInspectionConfirmResponse

  Validators (in order):
    1. msg.caller is assigned as inspector for the given inspectionId.
    2. Inspection status is #Pending.
    3. Candidate driver status is #PendingInspection.

  On success:
    - Update DriverInspectionDao.status = contract.outcome.
    - Count all #Confirmed inspections for the candidate.
    - If confirmedCount >= _minConfirmations:
        · Set driver.status = #Active.
        · Set driver.matchingCanisterId from regionMappings[region].
        · Pay _inspectionFeePerInspector ICP e8s to each confirming inspector (if fee > 0).
    - If all inspectors responded and confirmedCount < _minConfirmations:
        · Keep driver in #PendingInspection (future governance or retry logic TBD).
    - Return DriverInspectionConfirmResponse.

  On any validation failure:
    - Return #err with descriptive Text.
```

---

### 5.5 Rider Registration

```
// Public. No caller check.
// Rider registration is free. No liveness check. No peer inspection.
riderRegister(RiderRegisterContract) -> RiderRegisterResponse

  Validators (in order):
    1. msg.caller is not already registered as a rider (any status).
    2. region has an active mapping in regionMappings.

  On success:
    - Register rider with status = #Active immediately.
    - Set matchingCanisterId from regionMappings[region].
    - Return RiderContract.

  On any validation failure:
    - Return #err with descriptive Text.
```

---

### 5.6 Driver Lookup

```
// query — called by Matching canister before accepting a ride request.
driverLookup(driver : AccountId) -> ?DriverContract

  Returns DriverContract if driver is found and status = #Active, otherwise null.
```

---

### 5.7 Rider Lookup

```
// query — called by Matching canister before accepting a ride request.
riderLookup(rider : AccountId) -> ?RiderContract

  Returns RiderContract if rider is found and status = #Active, otherwise null.
```

---

### 5.8 Region Lookup

```
// query — returns the Matching canister assigned to a given region.
regionLookup(region : Region) -> ?RegionMappingContract

  Returns RegionMappingContract if an active mapping exists for region, otherwise null.
```

---

### 5.9 Support

```
// Public query. No authentication required. Used for external monitoring and metrics dashboards.
supportCounters() -> SupportCounters
```

Returns a snapshot of all call counters. Counters are reset every 24H by `worker.mo`.

---

### 5.10 Governance

```
// Called from Governance canister to adjust registration fees.
// msg.caller must match governance canister principal.
governanceUpdateFees(RegistryGovernanceUpdateFeesContract) -> RegistryGovernanceUpdateFeesResponse

  Validators:
  1. msg.caller must match governance canister principal.

  Logic:
  - Update driver and rider registration fees in state.


// Called from Governance canister to temporarily suspend a driver.
// msg.caller must match governance canister principal.
governanceSuspendDriver(RegistryGovernanceSuspendDriverContract) -> RegistryGovernanceSuspendDriverResponse

  Validators:
  1. msg.caller must match governance canister principal.
  2. Driver exists and is not already #Banned.

  Logic:
  - Set driver.status = #Suspended, suspendedUntil, suspendReason.


// Called from Governance canister to permanently ban a driver.
// msg.caller must match governance canister principal.
governanceBanDriver(RegistryGovernanceBanDriverContract) -> RegistryGovernanceBanDriverResponse

  Validators:
  1. msg.caller must match governance canister principal.
  2. Driver exists.

  Logic:
  - Set driver.status = #Banned. suspendedUntil = null.


// Called from Governance canister to adjust inspector count requirements.
// msg.caller must match governance canister principal.
governanceUpdateInspectorCount(RegistryGovernanceUpdateInspectorCountContract) -> RegistryGovernanceUpdateInspectorCountResponse

  Validators:
  1. msg.caller must match governance canister principal.
  2. minConfirmations >= 1 and maxInspectors <= 3 and minConfirmations <= maxInspectors.

  Logic:
  - Update _minConfirmations and _maxInspectors in state.


// Called from Governance canister to adjust the fee paid to each confirming inspector.
// msg.caller must match governance canister principal.
governanceUpdateInspectionFee(RegistryGovernanceUpdateInspectionFeeContract) -> RegistryGovernanceUpdateInspectionFeeResponse

  Validators:
  1. msg.caller must match governance canister principal.

  Logic:
  - Update _inspectionFeePerInspector in state.
```

---

## 6. Business Rules & Invariants

### 6.1 Liveness & Privacy Rules

| Rule | Detail |
|------|--------|
| No documents | No identity documents or personal data are ever stored on-chain or off-chain. |
| No raw data | Only a `LivenessHash` (DecideAI proof-of-humanity) is stored per driver. |
| Uniqueness | One liveness hash per driver. Duplicate = rejection. |
| Liveness | DecideAI liveness verification must return `verified = true`. Drivers only. |
| Riders | Riders undergo no liveness or document checks. Registration = form only. |
| Provider | `KYC_PROVIDER_CANISTER_ID` defined in `constants.mo` |

---

### 6.2 Driver Registration Process

| Step | Who | Description |
|------|-----|-------------|
| 0 | Driver | Authenticate via Internet Identity 2 |
| 1 | Driver | Fill registration form (name, car details, region, OpenChat handle). Status → `#Pending`. Form locked. |
| 2 | Driver | Complete liveness check on [decide.ai](https://id.decideai.xyz/). Submit `livenessHash`. Status → `#PendingInspection`. |
| 3 | System | Randomly select 1–3 currently active drivers with highest voting power. Create `DriverInspectionDao` records. |
| 4 | Inspectors | Each inspector contacts the candidate via OpenChat video call. Verifies: (a) speaks required language, (b) car matches form. No documents checked. |
| 5 | Inspectors | Call `driverInspectionConfirm()` with outcome. |
| 6 | System | If `confirmedCount >= _minConfirmations`: status → `#Active`. Inspector fee distributed. |

> **Privacy guarantee:** Inspectors only see the candidate's OpenChat handle and form data (name, car details). No identity documents are exchanged.

---

### 6.3 Rider Registration Process

| Step | Who | Description |
|------|-----|-------------|
| 0 | Rider | Authenticate via Internet Identity 2 |
| 1 | Rider | Fill simple registration form (name, email, phone, region). Status → `#Active` immediately. |

> **Free:** No fee, no liveness check, no inspection. Rider is operational immediately.

---

### 6.4 Region Mapping Rules

| Rule | Detail |
|------|--------|
| Mapping authority | Only Gateway canister may add or deactivate regions (single source of truth) |
| Deterministic | Same region always maps to the same Matching canister |
| Active check | Registration is rejected if region has no active mapping |

---

### 6.5 Activity Lifecycle

| Event | Trigger | Actor |
|-------|---------|-------|
| `lastActiveAt` updated | Driver/rider completes a ride | Matching canister (inter-canister call) |
| Marked `#Suspended` | Governance proposal executed | `governance.mo` |
| Suspension lifted | `suspendedUntil` timestamp passed | `worker.mo` (daily check) |
| Marked `#Banned` | Governance proposal executed | `governance.mo` |

---

### 6.6 Inspection Inspector Selection Rules

| Rule | Detail |
|------|--------|
| Pool | Only `#Active` drivers from the same region are eligible |
| Selection | Ordered by voting power (highest first); top `_maxInspectors` selected |
| Count | 1 to `_maxInspectors` inspectors assigned (default max: 3) |
| Confirmation threshold | `_minConfirmations` confirmations required (default: 1) |
| Form lock | Driver form data is read-only once submitted; inspectors see exactly what was filed |
| No documents | Inspectors do NOT check any identity documents; visual and language check only |

---

### 6.7 Registration Fees

| Rule | Detail |
|------|--------|
| Driver fee | 0 ICP default (TBD via governance) |
| Rider fee | 0 ICP (always free) |
| Inspector fee | `_inspectionFeePerInspector` ICP e8s paid per confirming inspector. Default 0. |
| Change authority | Governance canister only |

> **Note:** Inspector fee distribution is triggered automatically on driver activation. Mirror Gateway `DepositDao` pattern if non-zero fees are introduced.

---

## 7. Bootstrap Flows

### 7.1 Driver Registration Flow

```
0. Driver authenticates via Internet Identity 2.

1. Driver fills and submits registration form:
     · Name, email, phone, address
     · Car make, model, year, number plate
     · Country and city of operations
     · OpenChat handle (required for inspector video calls)
   driverRegister(DriverRegisterContract) → Registry
   Status → #Pending. Form is locked — no edits allowed after submission.

2. Driver completes liveness check on decide.ai (off-chain).
   Driver submits liveness proof:
   driverSubmitLiveness(livenessHash) → Registry
   Registry calls KYC_PROVIDER_CANISTER_ID to confirm livenessHash = verified.
   Status → #PendingInspection.

3. Registry selects 1–3 active drivers with highest voting power from the same region.
   Creates DriverInspectionDao for each inspector.
   Frontend notifies inspectors via their OpenChat handles.

4. Each inspector initiates a video call with the candidate via OpenChat.
   Inspector checks:
     · Candidate speaks Lithuanian (or other required languages).
     · Car is physically the car described in the registration form.
     · No documents are requested; no privacy data is collected.

5. Inspector submits outcome:
   driverInspectionConfirm({ inspectionId, outcome: #Confirmed | #Rejected }) → Registry

6. If confirmed count >= _minConfirmations:
   · Status → #Active.
   · matchingCanisterId set from regionMappings[region].
   · Inspector fee distributed (if _inspectionFeePerInspector > 0).

✅ Driver is registered and can receive ride requests in their mapped region.
```

### 7.2 Rider Registration Flow

```
0. Rider authenticates via Internet Identity 2.

1. Rider fills and submits simple registration form:
     · Name, email, phone
     · Country and city of operations
   riderRegister(RiderRegisterContract) → Registry
   Status → #Active immediately.

✅ Rider is registered and can request rides in their mapped region. Free, no checks.
```

---

### 7.3 Cleanup / pruning
```
Inactive drivers will be pruned with worker.mo after 90 Days (INACTIVE_DRIVER_PRUNING_PERIOD) if he has voting power less when 0.1% (INACTIVE_DRIVER_MINIMAL_VOTING_POWER)

```


## 8. Architecture

```
canister_registry/
├── main.mo               — Actor entry point. Exposes all public endpoints. Holds _version.
├── state.mo              — Stable state definitions. All maps and counters. Upgraded via postupgrade.
├── constants.mo          — Protocol-wide constants (KYC provider, fees, etc.)
├── types.mo              — All shared type definitions (module Types from Section 3).
├── types.governance.mo   — Governance contract types and inter-canister call logic.
├── logic.mo              — All business logic (registration, lookup, region mapping).
├── governance.mo         — Governance canister interface and logic.
├── validators.mo         — Pure validation functions called from main.mo before endpoint logic.
├── math.mo               — Pure helper functions (timestamp arithmetic, hash utilities, etc.)
└── worker.mo             — Heartbeat-driven background tasks:
                          · Lift expired suspensions (suspendedUntil < now → #Active)
                          · Reset call-rate counters every 24H
                          · TODO: mark inactive drivers (lastActiveAt > INACTIVE_THRESHOLD_DAYS)
                          · TODO: expire stale inspections (driver stays in #PendingInspection; retry logic TBD)

tests/
├── tests.[module_name].mops       — Unit tests (per module)
└── integration.[flow].mops        — End-to-end integration tests simulating registration flows
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
| DecideAI liveness inter-canister API | High | DecideAI canister interface not yet defined — placeholder in validators.mo |
| Inspector notification mechanism | High | Frontend must notify inspectors via OpenChat; on-chain trigger to be defined |
| Inspector fee distribution | High | Mirror Gateway `DepositDao` pattern when `_inspectionFeePerInspector > 0` |
| Inspection expiry / retry logic | Medium | What happens if all inspectors reject or fail to respond? Retry flow TBD |
| Rider governance suspend/ban | Medium | Mirror driver suspend/ban flow |
| `ProposalTarget` extension | High | Add `#Registry` variant to Governance canister `ProposalTarget` type |
| Monitoring dashboard integration | Low | `supportCounters()` endpoint is ready; consumer to be defined |
