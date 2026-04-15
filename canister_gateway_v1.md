# Gateway Canister — Specification v1
> **Status:** Draft · **Last updated:** 2026-04-09  
> **Language:** Motoko · **Platform:** Internet Computer Protocol (ICP)

---

## 1. Overview & Vision

The **Gateway Canister** is the directory and routing backbone of the decentralised Hydra Mobility Mesh. A globally distributed mesh of independent Gateway canisters acts as a resilient, peer-to-peer registry with no single point of failure — analogous to a gossip-protocol overlay network. It can function independently of other canisters, but can be governed by the Governance canister. It has build in subaccount logic for deposits and fees and processes deposits and fees via worker.mo. It manages its cycles independly, but Governance canister acts as a fallback for cycle management.

Key design properties:

- A new Gateway only needs to know **one existing Gateway** to bootstrap and discover the entire mesh.
- Gateways propagate each other's registrations laterally via `gatewayRegisterInternal`, so the full mesh stays consistent.
- Clone canisters (city-level user-facing canisters) register with any Gateway and discover the full mesh via periodic sync.
- If a Gateway or Clone fails, clients know all alternative endpoints and can migrate seamlessly.

---

## 2. Glossary

| Term | Definition |
|------|-----------|
| **Gateway** | A canister that acts as a directory node in the mesh. Stores the full list of Gateways and Clones. |
| **Clone** | A city/country-level canister pair (frontend + backend) that serves end users. Registered with and discovered via Gateways. |
| **Quarantine** | A trust restriction applied to a newly registered Gateway. During quarantine the Gateway cannot add new Gateways, and other Gateways will not sync with it. Duration: **6 months** from registration. |
| **Sync** | The process by which a Gateway or Clone fetches the latest registry state from a peer Gateway. |
| **Deposit** | ICP tokens locked by a Gateway or Clone operator as an anti-spam economic commitment before registration is accepted. |
| **Controller** | The ICP principal that controls (owns) a canister. Each Gateway owner is responsible for funding their canister's cycle balance. |

---

## 3. Definitions

## 3.1 Constants
```motoko
// constants.mo
module Constants {
    public let GOVERNANCE_CANISTER_ID : Principal = Principal.fromText("...");

    public let DEPOSIT_EXPIRY_PERIOD : Nat64 = 30 * 24 * 60 * 60;  // 30 days in seconds
    public let GATEWAY_REGISTRATION_FEE : Nat64 = 10 * 100_000_000;  // 10 ICP in e8s
    public let CANISTER_REGISTRATION_FEE : Nat64 = 10 * 100_000_000;  // 10 ICP in e8s

    public let QUARANTINE_EXPIRY_PERIOD : Nat64 = 6 * 30 * 24 * 60 * 60;  // 6 months in seconds
    
    public let GATEWAY_SYNC_PERIOD : Nat64 = 24 * 60 * 60;  // 24 hours in seconds
    public let GATEWAY_SYNC_ALL_PERIOD : Nat64 = 168 * 60 * 60;  // 168 hours in seconds
    public let CLONE_SYNC_PERIOD : Nat64 = 24 * 60 * 60;  // 24 hours in seconds
    
    public let GATEWAY_SYNC_LIMIT : Nat64 = 4;  // 4 calls in 24 hours
    public let GATEWAY_SYNC_ALL_LIMIT : Nat64 = 2;  // 2 calls in 168 hours
    public let CLONE_SYNC_LIMIT : Nat64 = 1;  // 1 call in 24 hours
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
    public type City        = Text;
    public type Country     = Text;
    public type Timestamp   = Nat64;   // nanoseconds since Unix epoch (ICP Ledger standard)
    public type Subaccount  = Blob;    // 32-byte unique subaccount identifier


    // ==========================================
    // --- 2. DEPOSIT TYPES                   ---
    // ==========================================

    /// Economic commitment required before a Gateway or Clone is accepted.
    /// If registration is never completed, the deposited funds are transferred
    /// to the canister owner after DEPOSIT_EXPIRY_PERIOD (30) days (spam prevention, processed by worker.mo).
    public type DepositStatus = {
        #pending;
        #paid;
    };

    public type DepositDao = {
        depositor  : CanisterId;    // The canister that made the deposit
        subaccount : Subaccount;    // Unique subaccount for this deposit
        amount     : Nat64;         // Deposit amount in e8s (ICP smallest unit)
        status     : DepositStatus; // Default: #pending
        consumed   : Bool;          // Set to true when deposit is used for registration. Default: false
        createdAt  : Timestamp;     // When the deposit record was created
    };


    // ==========================================
    // --- 3. REGISTRATION INSTRUCTIONS TYPE  ---
    // ==========================================

    /// Returned inside #err responses on first registration attempt.
    /// Provides machine-readable, structured guidance so operators can automate
    /// the onboarding process. Subject to review in a future iteration.
    public type RegistrationInstructions = {
        depositSubaccount : Subaccount;   // Where to send the deposit
        requiredAmount    : Nat64;        // Amount in e8s (defined in constants.mo)
        ledgerCanisterId  : CanisterId;   // ICP Ledger canister to transfer to
        nextStep          : Text;         // Human-readable description of what to do next
        checkEndpoint     : Text;         // Endpoint to call to verify deposit status
    };


    // ==========================================
    // --- 4. GATEWAY-TO-GATEWAY (G2G) TYPES  ---
    // ==========================================

    /// Internal state record. Stored per known Gateway.
    public type GatewayDao = {
        gateway          : CanisterId;
        country          : Country;
        city             : City;
        registeredAt     : Timestamp;
        quarantineUntil  : Timestamp;  // Quarantine lasts 6 months from registeredAt. Cannot add Gateways during this period.
        quarantineReason : Text;       // Initial value: "first registration"
        lastSyncAt       : Timestamp;
        counterNotSynced : Nat;        // Incremented +1 per day by worker.mo if no sync received. Max meaningful value: 30.
        isActive         : Bool;
    };

    /// Public-facing record shared during sync and registration responses.
    public type GatewayContract = {
        gateway         : CanisterId;
        country         : Country;
        city            : City;
        registeredAt    : Timestamp;
        quarantineUntil : Timestamp;
    };

    /// Sent by a Gateway to initiate a sync with a peer Gateway.
    /// Single cursor covers both Gateways and Clones — sync is always performed together.
    public type GatewaySyncContract = {
        gateway             : CanisterId;
        lastRegistrationAt  : Timestamp;  // Timestamp of the last known registration (Gateway or Clone)
    };

    public type GatewaySyncResponse = {
        #ok : {
            gateways           : [GatewayContract];
            clones             : [CloneContract];
            lastRegistrationAt : Timestamp;
        };
        #err : Text;
    };

    public type GatewayRegisterRequest = {
        gateway : CanisterId;
        country : Country;
        city    : City;
    };

    public type GatewayRegisterResponse = {
        #ok  : GatewayContract;
        #err : RegistrationInstructions;  // Always return structured instructions on failure
    };


    // ==========================================
    // --- 5. CLONE-TO-GATEWAY (C2G) TYPES    ---
    // ==========================================

    /// Internal state record. Stored per known Clone.
    public type CloneDao = {
        cloneFrontend    : CanisterId;
        cloneBackend     : CanisterId;
        country          : Country;
        city             : City;
        registeredAt     : Timestamp;
        lastSyncAt       : Timestamp;
        counterNotSynced : Nat;        // Incremented +1 per day by worker.mo if no sync received. Max meaningful value: 30.
        isActive        : Bool;
    };

    /// Public-facing record shared during sync and registration responses.
    public type CloneContract = {
        cloneFrontend : CanisterId;
        cloneBackend  : CanisterId;
        country       : Country;
        city          : City;
        registeredAt  : Timestamp;
    };

    /// Sent by a Clone to register itself with a Gateway.
    public type CloneRegisterContract = {
        cloneFrontend : CanisterId;
        cloneBackend  : CanisterId;
        country       : Country;
        city          : City;
    };

    public type CloneRegisterResponse = {
        #ok  : CloneContract;
        #err : RegistrationInstructions;  // Always return structured instructions on failure
    };

    /// Sent by a Clone (msg.caller = cloneBackend principal) to sync registry state.
    /// Single cursor covers both Gateways and Clones — sync is always performed together.
    public type CloneSyncContract = {
        cloneFrontend      : CanisterId;
        cloneBackend       : CanisterId;
        lastRegistrationAt : ?Timestamp;  // Null on first sync — returns full state
    };

    public type CloneSyncResponse = {
        #ok : {
            gateways           : [GatewayContract];
            clones             : [CloneContract];
            lastRegistrationAt : Timestamp;
        };
        #err : Text;
    };


    // ==========================================
    // --- 6. SUPPORT TYPES                   ---
    // ==========================================

    /// Snapshot of call counters. Exposed via supportCounters() for external monitoring.
    /// Counters are reset every 24H by worker.mo.
    public type SupportCounters = {
        gatewayRegister         : Nat;
        gatewayRegisterInternal : Nat;
        gatewaySync             : Nat;
        gatewaySyncAll          : Nat;
        cloneRegister           : Nat;
        cloneSync               : Nat;
    };
}
```

```motoko
// governance.mo

module Types {
  // ==========================================
  // --- 1. GATEWAY GOVERNANCE TYPES  ---
  // ==========================================

  // Used from Governance canister to change gateway canister fees for gateway and for canister registration
  GatewayGovernanceUpdateFeeContract = {
      gatewayRegistrationFee : Nat;
      canisterRegistrationFee : Nat;
      depositExpiryPeriod : Nat;
  }

  GatewayGovernanceUpdateFeeResponse = {
      #ok : { fees : GatewayGovernanceUpdateFeeContract }
      #err : Text;
  }

  // Used from Governance canister to change gateway canister periods for gateway and for canister registration
  GatewayGovernanceUpdatePeriodsContract = {
      quarantineExpiryPeriod : Nat;
      gatewaySyncPeriod : Nat;
      gatewaySyncAllPeriod : Nat;
      cloneSyncPeriod : Nat;
  }

  GatewayGovernanceUpdatePeriodsResponse = {
      #ok : { periods : GatewayGovernanceUpdatePeriodsContract }
      #err : Text;
  }

  GatewayGovernanceUpdateLimitsContract = {
      gatewaySyncLimit : Nat;
      gatewaySyncAllLimit : Nat;
      cloneSyncLimit : Nat;
  }

  GatewayGovernanceUpdateLimitsResponse = {
      #ok : { limits : GatewayGovernanceUpdateLimitsContract }
      #err : Text;
  }

  // Used from Governance canister to quarantine/unquarantine a gateway
  GatewayGovernanceQuarantineContract = {
      gateway : CanisterId;
      quarantineUntil : Timestamp;
      reason : Text;
  }

  GatewayGovernanceQuarantineResponse = {
      #ok : { gateway : CanisterId }
      #err : Text;
  }
}
```
---

## 4. Internal State Variables

```motoko
// --- Call counters (reset every 24H by worker.mo) ---
var _counterGatewayRegister         : Nat = 0;
var _counterGatewayRegisterInternal : Nat = 0;
var _counterGatewaySync             : Nat = 0;
var _counterGatewaySyncAll          : Nat = 0;
var _counterCloneRegister           : Nat = 0;
var _counterCloneSync               : Nat = 0;

// --- Version (incremented on every canister upgrade) ---
// Used in postupgrade migrations to detect state schema changes.
var _version : Nat = 1;
```

---

## 5. Public API

### 5.1 Health

```
// query — no cycle cost, called by peer Gateways before accepting registration
getStatus() -> #ok
```
Returns `#ok` if the canister is alive and responsive. No fields are returned.  
If the call fails or times out, the calling Gateway treats this canister as **not alive** and skips it.

---

### 5.2 Gateway Registration

```
// Public. No caller check. No call-rate limit.
// Called by a new Gateway operator to join the mesh.
gatewayRegister(GatewayRegisterRequest) -> GatewayRegisterResponse

  Validators (in order):
    1. Gateway is not already registered.
    2. Gateway has deposited the required ICP amount (see constants.mo). Deposit status = #paid, consumed = false.
    3. Gateway is alive: call getStatus() on the candidate Gateway; must return #ok.

  On success:
    - Register Gateway with quarantineUntil = now + 6 months, quarantineReason = "first registration".
    - Mark deposit as consumed = true.
    - Call gatewayRegisterInternal on all currently registered, active, non-quarantined Gateways
      to propagate the new member across the mesh. Failures are silently skipped.
    - Return GatewayContract.

  On any validation failure:
    - Return #err : RegistrationInstructions with all steps needed to complete registration.
```

```
// Internal. Caller must be a registered, active, non-quarantined Gateway (msg.caller check).
// No call-rate limit — called as part of mesh propagation, not by operators directly.
gatewayRegisterInternal(GatewayRegisterRequest) -> GatewayRegisterResponse

  Validators (in order):
    1. msg.caller is a registered, active, non-quarantined Gateway.
    2. Gateway to register is not already registered.
    3. Gateway to register is alive: call getStatus(); must return #ok.

  On success:
    - Register Gateway with quarantineUntil = now + 6 months, quarantineReason = "first registration".
    - Return GatewayContract.

  On failure:
    - Return #err with descriptive Text.
```

---

### 5.3 Gateway Sync

```
// Called daily by peer Gateways to stay in sync.
// Quarantined Gateways are NOT called by this Gateway (we don't trust them).
// Quarantined Gateways MAY call this Gateway and receive a response.
gatewaySync(GatewaySyncContract) -> GatewaySyncResponse

  Validators:
    1. msg.caller is a registered Gateway.
    2. Calling Gateway has not exceeded 4 calls in the last 24H (enforced via per-caller counter; reset by worker.mo).

  Logic:
    - If caller's lastRegistrationAt == this Gateway's lastRegistrationAt:
        Return #ok with empty arrays (fully in sync, save cycles).
    - If different:
        Return #ok with full gateways[] and clones[] arrays.
        Update lastSyncAt for the calling Gateway.
        Update lastRegistrationAt if caller's value is more recent.
```

```
// Called weekly as a full-state fallback sync. Ignores cursor, always returns everything.
// Quarantined Gateways are NOT called by this Gateway.
// Quarantined Gateways MAY call this Gateway and receive a response.
gatewaySyncAll(GatewaySyncContract) -> GatewaySyncResponse

  Validators:
    1. msg.caller is a registered Gateway.
    2. Calling Gateway has not exceeded 2 calls in the last 168 hours (enforced via per-caller counter; reset by worker.mo).

  Logic:
    - Always return full gateways[] and clones[] arrays without cursor check.
    - Update lastSyncAt for the calling Gateway.
```

---

### 5.4 Clone Registration

```
// Public. No caller check. No call-rate limit.
// Called by a new Clone operator to join the mesh.
cloneRegister(CloneRegisterContract) -> CloneRegisterResponse

  Validators (in order):
    1. Clone (cloneBackend) is not already registered.
    2. Clone has deposited the required ICP amount (see constants.mo). Deposit status = #paid, consumed = false.
    3. Clone is alive: call getStatus() on cloneBackend; must return #ok.

  On success:
    - Register Clone. No quarantine applies to Clones.
    - Mark deposit as consumed = true.
    - Clone propagation to peer Gateways happens passively via the next gatewaySync / gatewaySyncAll cycle.
      There is no cloneRegisterInternal — this is by design.
    - Return CloneContract.

  On any validation failure:
    - Return #err : RegistrationInstructions with all steps needed to complete registration.
```

> **Note:** Clones are propagated to peer Gateways passively through the existing `gatewaySync` / `gatewaySyncAll` flow. There is no `cloneRegisterInternal` equivalent. This is intentional — Clone registration latency across the mesh is acceptable, as Clones sync with Gateways independently.

---

### 5.5 Clone Sync

```
// Called daily by Clones to stay in sync with the mesh.
// msg.caller must match cloneBackend principal.
cloneSync(CloneSyncContract) -> CloneSyncResponse

  Validators:
    1. msg.caller is a registered Clone (matched via cloneBackend).
    2. Calling Clone has not exceeded 1 call in 24H (enforced via per-caller counter; reset by worker.mo).

  Logic:
    - If lastRegistrationAt is provided and matches this Gateway's value:
        Return #ok with empty arrays (fully in sync).
    - Otherwise:
        Return #ok with full gateways[] and clones[] arrays and current lastRegistrationAt.
    - Update lastSyncAt for the calling Clone.
```

---

### 5.6 Support

```
// Public query. No authentication required. Used for external monitoring and metrics dashboards.
supportCounters() -> SupportCounters
```

Returns a snapshot of all call counters. Counters are reset every 24H by `worker.mo`.

---

### 5.7 Governance

```
// Called from Governance canister to change gateway canister fees
// msg.caller must match governance canister principal
governanceUpdateFee(GatewayGovernanceUpdateFeeContract) -> GatewayGovernanceUpdateFeeResponse

  Validators:
  1. msg.caller must match governance canister principal

  Logic:
  - Update gateway canister fees in state
  

governanceUpdatePeriods(GatewayGovernanceUpdatePeriodsContract) -> GatewayGovernanceUpdatePeriodsResponse

  Validators:
  1. msg.caller must match governance canister principal

  Logic:
  - Update gateway canister periods in state

governanceQuarantine(GatewayGovernanceQuarantineContract) -> GatewayGovernanceQuarantineResponse

  Validators:
  1. msg.caller must match governance canister principal

  Logic:
  - Update gateway canister quarantine status in state

governanceUpdateLimits(GatewayGovernanceUpdateLimitsContract) -> GatewayGovernanceUpdateLimitsResponse

  Validators:
  1. msg.caller must match governance canister principal

  Logic:
  - Update gateway canister limits in state

```
## 6. Business Rules & Invariants

### 6.1 Quarantine Rules

| Rule | Detail |
|------|--------|
| Duration | 6 months from `registeredAt` (`quarantineUntil = registeredAt + 6 months in nanoseconds`) |
| Initial reason | `"first registration"` |
| Quarantined Gateway: can **receive** syncs | ✅ Yes — other Gateways may sync with it |
| Quarantined Gateway: can **initiate** `gatewayRegisterInternal` | ❌ No — rejected by peers |
| Gateways sync **to** quarantined Gateways | ❌ No — quarantined Gateways are excluded from outgoing sync targets |
| Clones: quarantine applies | ❌ No — Clones are never quarantined |
| Gateways calls quarantined gateways getStatus() | ✅ Yes — used for liveness validation |


### 6.2 Rate Limits

| Endpoint | Limit | Window | Reset |
|----------|-------|--------|-------|
| `gatewaySync` | 4 calls | 24 hours | `worker.mo` heartbeat |
| `gatewaySyncAll` | 2 calls | 168 hours | `worker.mo` heartbeat |
| `cloneSync` | 1 call | 24 hours | `worker.mo` heartbeat |

Rate limits are enforced per `msg.caller`. Counters are stored in state and reset by `worker.mo`.

### 6.3 Deposit Rules

| Rule | Detail |
|------|--------|
| Required amount | Defined in `constants.mo` (applies to both Gateways and Clones) |
| `consumed` default | `false` |
| Set to `consumed = true` | On successful registration |
| Unclaimed deposits (not consumed after 30 days) | Transferred to the Gateway canister owner by `worker.mo`. Spam prevention mechanism. |

### 6.4 Activity Lifecycle

| Event | Trigger | Actor |
|-------|---------|-------|
| `counterNotSynced` incremented | No sync received in a given day | `worker.mo` (daily) |
| Marked `isActive = false` | `counterNotSynced > 14` | `worker.mo` |
| Deleted from state | Inactive for 30 days | `worker.mo` |
| `getStatus()` fails during registration | Candidate skipped | Calling Gateway |

### 6.5 Cycle Management

The owner of each Gateway canister is responsible for maintaining its cycle balance. There is no shared cycle pool. Gateways make inter-canister calls (to peers and to Clones during `getStatus()` checks); all cycles for those calls are drawn from the calling Gateway's balance.

---

## 7. Bootstrap Flows

### 7.1 Gateway Bootstrap

```
1. Operator deploys Gateway B (e.g., in Lithuania).
2. Operator configures Gateway B with the CanisterId of an existing Gateway A.
3. Operator deposits the required ICP to Gateway B's subaccount on the ICP Ledger.

4. Gateway B calls: gatewayRegister(GatewayRegisterRequest) → Gateway A
5. Gateway A validates:
     a. B is not already registered
     b. B's deposit is paid and not consumed
     c. getStatus() on B returns #ok (B is alive)
6. Gateway A registers B (quarantineUntil = now + 6 months, quarantineReason = "first registration").
7. Gateway A propagates B to all active, non-quarantined peer Gateways (C, D, …) via gatewayRegisterInternal.
     - Failures are silently skipped.
8. Gateway A returns GatewayRegisterResponse #ok : GatewayContract to B.

9. Gateway B calls: gatewaySync(GatewaySyncContract) → Gateway A
10. Gateway A responds with full gateways[] and clones[] lists.
11. Gateway B is now aware of C, D, ...

12. Gateway B calls: gatewaySync(GatewaySyncContract) → C, D, ...
    (B is in quarantine, so C and D will not propagate any registrations B attempts. B can still receive syncs.)

✅ All Gateways are now aware of each other. B is quarantined for 6 months.
```

### 7.2 Clone Bootstrap

```
1. Operator deploys Clone B (frontend + backend canisters) in a target city.
2. Operator configures Clone B with the CanisterId of any known Gateway (e.g., Gateway A).
3. Operator deposits the required ICP to Clone B's subaccount on the ICP Ledger.

4. Clone B calls: cloneRegister(CloneRegisterContract) → Gateway A
5. Gateway A validates:
     a. B is not already registered
     b. B's deposit is paid and not consumed
     c. getStatus() on cloneBackend returns #ok
6. Gateway A registers Clone B. No quarantine.
7. Gateway A returns CloneRegisterResponse #ok : CloneContract to Clone B.

8. Clone B calls: cloneSync(CloneSyncContract) → Gateway A
9. Gateway A responds with full gateways[] and clones[] lists + lastRegistrationAt.
10. Clone B is now aware of all active Gateways and alternative Clones.

11. Other Gateways discover Clone B passively via the next gatewaySync / gatewaySyncAll cycle.
    There is no fast propagation path for Clones — this is by design.

✅ Clone B is registered and synced. Users can be directed to it or away from it on failure.
```

---

## 8. Architecture

```
canister_gateway/
├── main.mo               — Actor entry point. Exposes all public endpoints. Holds _version.
├── state.mo              — Stable state definitions. All maps and counters. Upgraded via postupgrade.
├── constants.mo          — Protocol-wide constants (deposit amounts, quarantine duration, rate limits, etc.)
├── types.mo              — All shared type definitions (module Types from Section 3).
├── types.governance.mo   — Governance canister type definitions.
├── logic.mo              — All business logic.
├── governance.mo         — Governance canister interface and logic.
├── validators.mo         — Pure validation functions called from main.mo before endpoint logic.
├── math.mo               — Pure helper functions (timestamp arithmetic, duration checks, etc.)
└── worker.mo             — Heartbeat-driven background tasks:
                          · Increment counterNotSynced for stale Gateways/Clones
                          · Mark inactive (counterNotSynced > 14)
                          · Delete from state (inactive > 30 days)
                          · Reset call-rate counters every 24H / 168 hours
                          · Transfer unclaimed deposits to canister owner after 30 days

tests/
├── tests.[module_name].mops       — Unit tests (per module)
└── tests.[module_name].mops — End-to-end integration tests simulating Gateway and Clone bootstrap flows
```

**Key technology choices:**
- `Tier.Map` — for ordered, stable `TrieMap`-compatible state storage in state.mo
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
| Finalise `RegistrationInstructions` type fields | High | Author to review and approve field set |
| Extended quarantine rules | Medium | Conditions under which quarantine can be extended beyond 6 months |
| Monitoring dashboard integration | Low | `supportCounters()` endpoint is ready; consumer to be defined |
| Refund / unclaimed deposit UI | Low | worker.mo handles transfer; operator notification mechanism TBD |
