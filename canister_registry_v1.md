# Registry Canister — Specification v1
> **Status:** Draft · **Last updated:** 2026-04-13  
> **Language:** Motoko · **Platform:** Internet Computer Protocol (ICP)

---

## 1. Overview & Vision

The **Registry Canister** is the identity and onboarding backbone of the Hydra Mobility Protocol. It manages driver and rider registration, performs liveness verification, and routes users to the correct city-level Matching canister via deterministic regional mapping. This canister functions with other canisters. It don't have subaccount logic and relies on Governance canister. Cycles are managed by Governance canister.

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
    public let SUSPENSION_EXPIRY_PERIOD_SECONDS  : Nat64 = 30 * 24 * 60 * 60;   // 30 days in seconds

    // Limits, must be adjusted buy country/city
    public let MAX_ACTIVE_DRIVERS_PER_WEEK : Nat64 = 500;
    public let MAX_ACTIVE_DRIVERS_PER_MONTH : Nat64 = 1000;

    public let MAX_FOREIGNERS_REGISTERED_PERCENT : Float = 20.0;

    //Cleanup
    public let INACTIVE_DRIVER_PRUNING_PERIOD_SECONDS : Nat64 = 90 * 24 * 60 * 60;   // 90 days in seconds
    public let INACTIVE_DRIVER_MINIMAL_PERCENTVOTING_POWER : Float = 0.1; // 0.1% of total voting power

    // Registration expiry — drivers stuck in #Pending (fee unpaid) are deleted after this period.
    public let PENDING_REGISTRATION_EXPIRY_PERIOD_SECONDS  : Nat64 = 30 * 24 * 60 * 60;  // 30 days in seconds

    // Liveness expiry — drivers stuck in #PendingLiveness (fee paid, liveness not completed) are deleted.
    // Registration fee is NOT refunded. Driver re-registers from scratch.
    public let PENDING_LIVENESS_EXPIRY_PERIOD_SECONDS : Nat64 = 30 * 24 * 60 * 60;  // 30 days in seconds

    // Inspection expiry — drivers stuck in #PendingInspection without confirmation are deleted.
    public let PENDING_INSPECTION_EXPIRY_PERIOD_SECONDS : Nat64 = 30 * 24 * 60 * 60;  // 30 days in seconds
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
        pendingLivenessAt  : ?Timestamp;     // Set when status transitions to #PendingLiveness (payment confirmed)
        pendingInspectionAt: ?Timestamp;     // Set when status transitions to #PendingInspection (liveness passed)
        lastActiveAt       : Timestamp;
        suspendedUntil     : ?Timestamp;     // Null if not suspended
        suspendReason      : Text;           // Empty if not suspended
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
    // --- 5. REGION MAPPING TYPES            ---
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
    // --- 6. SUPPORT TYPES                   ---
    // ==========================================

    /// Snapshot of call counters. Exposed via supportCounters() for external monitoring.
    /// Counters are reset every 24H by worker.mo.
    public type SupportCounters = {
        driverRegister : Nat;
        riderRegister  : Nat;
        driverLookup   : Nat;
        riderLookup    : Nat;
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


    // ==========================================
    // --- 3. GOVERNANCE CONFIG TYPE          ---
    // ==========================================

    /// All live configurable values for the Registry canister.
    /// Returned by governanceGetConfig() composite query via Governance.
    public type RegistryGovernanceConfig = {
        driverRegistrationFee : Nat;
        riderRegistrationFee  : Nat;
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
```

---

```motoko
// main.mo
// --- Call counters (reset every 24H by worker.mo) ---
var _counterDriverRegister : Nat = 0;
var _counterRiderRegister  : Nat = 0;
var _counterDriverLookup   : Nat = 0;
var _counterRiderLookup    : Nat = 0;

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
// Public. msg.caller must match a driver with status = #PendingLiveness.
// Step 3 — driver submits their DecideAI liveness hash after registration fee has been paid.
// Liveness check can be retried unlimited times while status = #PendingLiveness.
driverSubmitLiveness(livenessHash : LivenessHash) -> Result<(), Text>

  Validators (in order):
    1. msg.caller is registered as a driver with status = #PendingLiveness.
    2. livenessHash is not already registered by another driver (prevents duplicate proof).
    3. Calls KYC_PROVIDER_CANISTER_ID with livenessHash; must return verified = true.

  On success:
    - Update driver.livenessHash.
    - Update driver.status = #PendingInspection.
    - Update driver.pendingInspectionAt = now().
    - Collect all #Active drivers in the same region (excluding candidate) from state.drivers.
    - Call REPUTATION_CANISTER_ID.inspectionAssign(candidateId, eligibleDrivers[]).
      Reputation selects inspectors, assigns records, manages the full inspection lifecycle.
    - Return #ok.

  On any validation failure:
    - Return #err with descriptive Text.
```

---

### 5.4 Registry Driver Payment Confirmed (Governance contract call)

```
// Internal. msg.caller must match Governance canister principal.
// Called by Governance after it verifies the driver registration fee payment on the ICP Ledger.
registryDriverPaymentConfirmed(driver : AccountId) -> Result<(), Text>

  Validators (in order):
    1. msg.caller must match GOVERNANCE_CANISTER_ID.
    2. Driver exists and status is #Pending.

  On success:
    - Set driver.status = #PendingLiveness.
    - Set driver.pendingLivenessAt = now().
    - Return #ok.

  On any validation failure:
    - Return #err with descriptive Text.
```

---

### 5.5 Registry Activate Driver (Reputation contract call)

```
// Internal. msg.caller must match Reputation canister principal.
// Called by Reputation canister when inspectionConfirmedCount >= _minConfirmations.
registryActivateDriver(candidateId : AccountId) -> Result<(), Text>

  Validators (in order):
    1. msg.caller must match REPUTATION_CANISTER_ID.
    2. Driver exists and status is #PendingInspection.

  On success:
    - Set driver.status = #Active.
    - Set driver.matchingCanisterId from regionMappings[region].
    - Call GOVERNANCE_CANISTER_ID.depositRefundDriverRegistration(candidateId).
      Governance returns the registration fee ICP back to the driver's principal.
    - Return #ok.

  On any validation failure:
    - Return #err with descriptive Text.
```

---

### 5.6 Registry Inspection Failed (Reputation contract call)

```
// Internal. msg.caller must match Reputation canister principal.
// Called by Reputation canister when an inspection definitively fails
// (e.g., all responded but confirmedCount < _minConfirmations).
registryInspectionFailed(candidateId : AccountId) -> Result<(), Text>

  Validators (in order):
    1. msg.caller must match REPUTATION_CANISTER_ID.
    2. Driver exists and status is #PendingInspection.

  On success:
    - Delete driver from state.drivers.
    - (Registration fee is left locked in Governance; Governance worker sweeps it on expiry.)
    - Return #ok.

  On any validation failure:
    - Return #err with descriptive Text.
```

---

### 5.7 Rider Registration

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
// query — called by Governance canister composite query. No caller check (public query).
// Returns all live configurable values in a single typed record.
governanceGetConfig() -> RegistryGovernanceConfig
```

Returns the current live state of all governance-controlled constants. Used by `Governance.governanceGetAllConfigs()` composite query.

---

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
| 2 | Driver | UI calls `Governance.depositSubaccountGet(#DriverRegistration)`. Governance reads fee from `registryGovernanceGetConfig()`. UI shows subaccount + amount. |
| 3 | Driver | Send registration fee to displayed subaccount. Press "I have paid". UI calls `Governance.depositConfirmDriverRegistration(depositId)`. |
| 4 | Governance | Verifies payment on ICP Ledger. Calls `Registry.registryDriverPaymentConfirmed(driver)`. Status → `#PendingLiveness`. |
| 5 | Driver | Complete liveness check on [decide.ai](https://id.decideai.xyz/). Submit `livenessHash`. Status → `#PendingInspection`. |
| 6 | System | Randomly select 1–3 active drivers with highest voting power. Call `Reputation.inspectionAssign()`. |
| 7 | Inspectors | Each inspector contacts the candidate via OpenChat video call. Verifies language and car. No documents checked. |
| 8 | Inspectors | Call `driverInspectionConfirm()` with outcome. |
| 9 | System | If `confirmedCount >= _minConfirmations`: status → `#Active`. Inspector fee distributed. |

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

### 6.6 Reputation Canister Delegation

| Concern | Handled by |
|---------|------------|
| Inspector selection | Registry selects active drivers by voting power, passes list to `REPUTATION_CANISTER_ID.inspectionAssign()` |
| Inspection records | Reputation canister owns all `DriverInspectionDao` state |
| Inspector confirmation (`driverInspectionConfirm`) | Reputation canister public API |
| Driver activation on threshold | Reputation calls `registryActivateDriver()` on Registry |
| Inspector fee distribution | Reputation canister |
| Inspection status query | Registry calls `REPUTATION_CANISTER_ID.inspectionStatus(candidateId)` |

> See `canister_reputation_v1.md` for the full Reputation canister specification.

---

### 6.7 Registration Fees

| Rule | Detail |
|------|--------|
| Driver fee | 0 ICP default (decided via governance) |
| Rider fee | 0 ICP (always free) |
| Inspector fee | Managed by Reputation canister. See `canister_reputation_v1.md`. |
| Change authority | Governance canister only |
| Fee held by | Governance canister. Status = `#Paid` while driver completes registration. |
| Fee returned | Only when driver reaches `#Active`. Governance `depositRefundDriverRegistration()` returns ICP to driver. |
| Fee forfeited | If driver record is deleted at `#Pending` or `#PendingLiveness` expiry — ICP is swept to Governance treasury. |

> **Anti-spam:** Fee is non-refundable if registration is abandoned at any stage before `#Active`.

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

2. UI calls depositSubaccountGet(#DriverRegistration) → Governance
   Governance reads driverRegistrationFee from Registry.governanceGetConfig().
   Returns { depositId, subaccount, amount }.
   UI displays subaccount address and required amount to the driver.

3. Driver sends registration fee to the displayed subaccount on the ICP Ledger.
   Driver presses "I have paid".
   UI calls depositConfirmDriverRegistration(depositId) → Governance
   Governance verifies ICP Ledger balance of deposit.subaccount >= deposit.amount.
   Governance calls registryDriverPaymentConfirmed(driver) → Registry.
   Status → #PendingLiveness.

4. Driver completes liveness check on decide.ai (off-chain). Can retry unlimited times.
   Driver submits liveness proof:
   driverSubmitLiveness(livenessHash) → Registry
   Registry calls KYC_PROVIDER_CANISTER_ID to confirm livenessHash = verified.
   Status → #PendingInspection.

5. Registry collects all #Active drivers in the same region (excluding candidate).
   Calls REPUTATION_CANISTER_ID.inspectionAssign(candidateId, eligibleDrivers[]).
   Reputation selects inspectors by voting power, assigns inspection records,
   manages the full inspection lifecycle including OpenChat notifications.

6. Each inspector initiates a video call with the candidate via OpenChat.
   Inspector checks:
     · Candidate speaks required language(s).
     · Car is physically the car described in the registration form.
     · No documents are requested; no privacy data is collected.

7. Inspector submits outcome:
   driverInspectionConfirm({ inspectionId, outcome: #Confirmed | #Rejected }) → Reputation

8. If confirmed count >= _minConfirmations:
   · Reputation calls registryActivateDriver(driver) → Registry. Status → #Active.
   · matchingCanisterId set from regionMappings[region].
   · Registry calls GOVERNANCE_CANISTER_ID.depositRefundDriverRegistration(driver).
   · Governance returns registration fee ICP to driver.
   · Inspector fee distributed by Reputation (if _inspectionFeePerInspector > 0).

✅ Driver is registered and can receive ride requests in their mapped region.
   Registration fee is returned.
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
Inactive drivers will be pruned with worker.mo after 90 Days (INACTIVE_DRIVER_PRUNING_PERIOD_SECONDS) if he has voting power less when 0.1% (INACTIVE_DRIVER_MINIMAL_VOTING_POWER)
```

### 7.4 Expired Pending Registration Cleanup
```
Drivers stuck in #Pending (form submitted but registration fee never paid) are deleted by worker.mo
after PENDING_REGISTRATION_EXPIRY_PERIOD_SECONDS (30 days).

Governance worker.mo also marks the #DriverRegistration deposit #Expired and sweeps any
paid-but-unconfirmed funds to the Governance treasury.

After deletion the driver principal is treated as never having applied.
Validator 1 of driverRegister() passes and the driver may re-register from scratch.
```

### 7.5 Expired PendingLiveness Cleanup
```
Drivers stuck in #PendingLiveness (fee paid, liveness not completed) are deleted by worker.mo
after PENDING_LIVENESS_EXPIRY_PERIOD_SECONDS (30 days from pendingLivenessAt).

The registration fee is NOT refunded. The Governance deposit remains #Paid and is swept to
Governance treasury by Governance worker.mo on expiry.

After deletion the driver principal is treated as never having applied.
Validator 1 of driverRegister() passes and the driver may re-register from scratch (paying the fee again).
```

### 7.6 Expired PendingInspection Cleanup
```
Drivers stuck in #PendingInspection (passed liveness, failed/timeout on inspection) are deleted by worker.mo
after PENDING_INSPECTION_EXPIRY_PERIOD_SECONDS (30 days from pendingInspectionAt).

The registration fee is NOT refunded. The Governance deposit remains #Paid and is swept to
Governance treasury by Governance worker.mo on expiry.

After deletion the driver principal is treated as never having applied.
Validator 1 of driverRegister() passes and the driver may re-register from scratch (paying the fee again).

Note: Reputation canister will also clean up matching staled DriverInspectionDao records on its side.
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
                          · Delete #Pending driver registrations older than PENDING_REGISTRATION_EXPIRY_PERIOD_SECONDS
                            (fee was never paid; record removed; driver may re-register from scratch)
                          · Delete #PendingLiveness driver registrations older than PENDING_LIVENESS_EXPIRY_PERIOD_SECONDS
                            (checked via pendingLivenessAt; fee is forfeited; driver re-registers from scratch)
                          · Delete #PendingInspection driver registrations older than PENDING_INSPECTION_EXPIRY_PERIOD_SECONDS
                            (checked via pendingInspectionAt; fee is forfeited; driver re-registers from scratch)
                          · TODO: mark inactive drivers (lastActiveAt > INACTIVE_THRESHOLD_DAYS)

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
| Rider governance suspend/ban | Medium | Mirror driver suspend/ban flow |
| Monitoring dashboard integration | Low | `supportCounters()` endpoint is ready; consumer to be defined |
