# Registry Canister — Specification v1
> **Status:** Draft · **Last updated:** 2026-04-13  
> **Language:** Motoko · **Platform:** Internet Computer Protocol (ICP)

---

## 1. Overview & Vision

The **Registry Canister** is the identity and onboarding backbone of the Hydra Mobility Protocol. It manages driver and rider registration, stores hashed KYC data on-chain for liveness verification, and performs deterministic regional mapping to route users to the correct city-level Matching canister.

Key design properties:

- **Privacy-first:** Only cryptographic hashes of KYC data are stored on-chain — no raw personal data is ever persisted.
- **Liveness-verified:** Uses [DecideAI](https://id.decideai.xyz/) for KYC liveness checks before registration is finalised.
- **Deterministic routing:** Each registered driver/rider is mapped to a geographic region (city/country) which determines their assigned Matching canister.
- **Role-separated:** Drivers and riders are distinct identity types with separate registration flows and data models.

---

## 2. Glossary

| Term | Definition |
|------|-----------|
| **Driver** | A registered transport provider who has completed KYC and is mapped to a regional Matching canister. |
| **Rider** | A registered end-user who has completed KYC and can request rides in their mapped region. |
| **KYC Hash** | A cryptographic hash of the user's identity document data. Raw data is never stored; only the hash is written to chain. |
| **Liveness Check** | A proof-of-humanity verification performed via DecideAI before registration is accepted. |
| **Region** | A city + country pair. Deterministically maps a user to a specific Matching canister. |
| **RegistryStatus** | The lifecycle state of a driver or rider record (pending, active, suspended, banned). |

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
}
```

## 3.2 Types
```motoko
// types.mo
module Types {

    // ==========================================
    // --- 1. CORE ALIASES                    ---
    // ==========================================

    public type CanisterId  = Principal;
    public type AccountId   = Principal;
    public type Timestamp   = Nat64;   // nanoseconds since Unix epoch (ICP Ledger standard)
    public type KycHash     = Blob;    // SHA-256 hash of the raw KYC document data
    public type Region      = { city : Text; country : Text };


    // ==========================================
    // --- 2. REGISTRY STATUS TYPES           ---
    // ==========================================

    public type RegistryStatus = {
        #Pending;     // KYC submitted but not yet verified
        #Active;      // Verified and operational
        #Suspended;   // Temporarily restricted — set by governance
        #Banned;      // Permanently removed — set by governance
    };


    // ==========================================
    // --- 3. DRIVER TYPES                    ---
    // ==========================================

    /// Internal state record. Stored per registered driver.
    public type DriverDao = {
        owner          : AccountId;
        kycHash        : KycHash;       // Hash of KYC document. Raw data never stored.
        region         : Region;        // City + country used to map to a Matching canister
        matchingCanisterId : CanisterId; // Assigned Matching canister for this driver's region
        status         : RegistryStatus;
        registeredAt   : Timestamp;
        lastActiveAt   : Timestamp;
        suspendedUntil : ?Timestamp;    // Null if not suspended
        suspendReason  : Text;          // Empty if not suspended
    };

    /// Public-facing record shared with other canisters.
    public type DriverContract = {
        owner              : AccountId;
        region             : Region;
        matchingCanisterId : CanisterId;
        status             : RegistryStatus;
        registeredAt       : Timestamp;
    };

    public type DriverRegisterContract = {
        kycHash  : KycHash;
        region   : Region;
    };

    public type DriverRegisterResponse = {
        #ok  : DriverContract;
        #err : Text;
    };


    // ==========================================
    // --- 4. RIDER TYPES                     ---
    // ==========================================

    /// Internal state record. Stored per registered rider.
    public type RiderDao = {
        owner          : AccountId;
        kycHash        : KycHash;       // Hash of KYC document. Raw data never stored.
        region         : Region;
        matchingCanisterId : CanisterId; // Assigned Matching canister for this rider's region
        status         : RegistryStatus;
        registeredAt   : Timestamp;
        lastActiveAt   : Timestamp;
        suspendedUntil : ?Timestamp;
        suspendReason  : Text;
    };

    /// Public-facing record shared with other canisters.
    public type RiderContract = {
        owner              : AccountId;
        region             : Region;
        matchingCanisterId : CanisterId;
        status             : RegistryStatus;
        registeredAt       : Timestamp;
    };

    public type RiderRegisterContract = {
        kycHash  : KycHash;
        region   : Region;
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

    /// Payload for governanceRegisterRegion — adds a new city/country → Matching canister mapping.
    public type RegistryGovernanceRegisterRegionContract = {
        region             : Region;
        matchingCanisterId : CanisterId;
    };

    public type RegistryGovernanceRegisterRegionResponse = {
        #ok  : RegionMappingContract;
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
regionMappings : TrieMap<Text, RegionMappingDao>  // key: "city:country"
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
// Public. No caller check.
// Called by a driver to register their identity.
driverRegister(DriverRegisterContract) -> DriverRegisterResponse

  Validators (in order):
    1. msg.caller is not already registered as a driver.
    2. kycHash is not already registered by another driver (prevents duplicate KYC).
    3. KYC liveness check passes: call KYC_PROVIDER_CANISTER_ID with kycHash; must return verified = true.
    4. region has an active mapping in regionMappings.

  On success:
    - Register driver with status = #Active.
    - Set matchingCanisterId from regionMappings[region].
    - Return DriverContract.

  On any validation failure:
    - Return #err with descriptive Text.
```

---

### 5.3 Rider Registration

```
// Public. No caller check.
// Called by a rider to register their identity.
riderRegister(RiderRegisterContract) -> RiderRegisterResponse

  Validators (in order):
    1. msg.caller is not already registered as a rider.
    2. kycHash is not already registered by another rider (prevents duplicate KYC).
    3. KYC liveness check passes: call KYC_PROVIDER_CANISTER_ID with kycHash; must return verified = true.
    4. region has an active mapping in regionMappings.

  On success:
    - Register rider with status = #Active.
    - Set matchingCanisterId from regionMappings[region].
    - Return RiderContract.

  On any validation failure:
    - Return #err with descriptive Text.
```

---

### 5.4 Driver Lookup

```
// query — called by Matching canister before accepting a ride request.
driverLookup(driver : AccountId) -> ?DriverContract

  Returns DriverContract if driver is found and status = #Active, otherwise null.
```

---

### 5.5 Rider Lookup

```
// query — called by Matching canister before accepting a ride request.
riderLookup(rider : AccountId) -> ?RiderContract

  Returns RiderContract if rider is found and status = #Active, otherwise null.
```

---

### 5.6 Region Lookup

```
// query — returns the Matching canister assigned to a given region.
regionLookup(region : Region) -> ?RegionMappingContract

  Returns RegionMappingContract if an active mapping exists for region, otherwise null.
```

---

### 5.7 Support

```
// Public query. No authentication required. Used for external monitoring and metrics dashboards.
supportCounters() -> SupportCounters
```

Returns a snapshot of all call counters. Counters are reset every 24H by `worker.mo`.

---

### 5.8 Governance

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


// Called from Governance canister to register a new city region mapping.
// msg.caller must match governance canister principal.
governanceRegisterRegion(RegistryGovernanceRegisterRegionContract) -> RegistryGovernanceRegisterRegionResponse

  Validators:
  1. msg.caller must match governance canister principal.
  2. Region is not already registered.

  Logic:
  - Insert new RegionMappingDao with isActive = true.
  - Return RegionMappingContract.
```

---

## 6. Business Rules & Invariants

### 6.1 KYC Rules

| Rule | Detail |
|------|--------|
| Storage | Only `kycHash` (SHA-256) is stored on-chain. Raw document data is never persisted. |
| Uniqueness | One KYC hash per identity type (driver or rider). Duplicate = rejection. |
| Liveness | DecideAI liveness verification must return `verified = true` before registration is finalised. |
| Provider | `KYC_PROVIDER_CANISTER_ID` defined in `constants.mo` |

---

### 6.2 Region Mapping Rules

| Rule | Detail |
|------|--------|
| Mapping authority | Only Governance canister may add or deactivate regions |
| Deterministic | Same region always maps to the same Matching canister |
| Active check | Registration is rejected if region has no active mapping |

---

### 6.3 Activity Lifecycle

| Event | Trigger | Actor |
|-------|---------|-------|
| `lastActiveAt` updated | Driver/rider completes a ride | Matching canister (inter-canister call) |
| Marked `#Suspended` | Governance proposal executed | `governance.mo` |
| Suspension lifted | `suspendedUntil` timestamp passed | `worker.mo` (daily check) |
| Marked `#Banned` | Governance proposal executed | `governance.mo` |

---

### 6.4 Registration Fees

| Rule | Detail |
|------|--------|
| Default | 0 ICP (TBD via governance) |
| Change authority | Governance canister only |
| Enforcement | Checked against deposit before registration is accepted |

> **Note:** Fee logic TBD. Mirror Gateway deposit pattern (`DepositDao`) if non-zero fees are introduced.

---

## 7. Bootstrap Flows

### 7.1 Driver Registration Flow

```
1. Operator (driver) obtains a KYC liveness proof from DecideAI (kycHash).
2. Driver calls: driverRegister(DriverRegisterContract) → Registry Canister
3. Registry validates:
     a. Driver is not already registered.
     b. kycHash is unique.
     c. Calls KYC_PROVIDER_CANISTER_ID for liveness confirmation.
     d. region has an active mapping.
4. Registry registers driver with status = #Active.
5. Registry returns DriverContract with matchingCanisterId.
6. Driver is now discoverable by the Matching canister via driverLookup().

✅ Driver is registered and can receive ride requests in their mapped region.
```

### 7.2 Region Mapping Flow

```
1. A governance proposal is submitted: #Registry(#RegisterRegion(args)).
2. Proposal is voted on and accepted by the driver DAO.
3. executeProposal() calls governanceRegisterRegion(RegistryGovernanceRegisterRegionContract) → Registry Canister.
4. Registry inserts new RegionMappingDao with isActive = true.
5. New registrations for that region are now accepted.

✅ New city/country region is live and accepting driver/rider registrations.
```

---

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
| KYC provider inter-canister API | High | DecideAI canister interface not yet defined — placeholder in validators.mo |
| Registration fee mechanism | High | Mirror Gateway `DepositDao` pattern if non-zero fee is introduced |
| Rider governance suspend/ban | Medium | Mirror driver suspend/ban flow |
| `ProposalTarget` extension | High | Add `#Registry` variant to Governance canister `ProposalTarget` type |
| Inactive driver pruning | Low | `worker.mo` task to mark drivers inactive after 90 days without a trip |
| Monitoring dashboard integration | Low | `supportCounters()` endpoint is ready; consumer to be defined |
