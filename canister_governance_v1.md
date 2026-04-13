# Governance Canister — Specification v1
> **Status:** Draft · **Last updated:** 2026-04-13  
> **Language:** Motoko · **Platform:** Internet Computer Protocol (ICP)

---

## 1. Overview & Vision

The **Governance Canister** is the local brain of a Hydra Mobility instance. It manages the **DRV (Voting Point)** ledger, handles driver-submitted proposals, and executes approved changes on the regional Gateway and Matching canisters.

Key design properties:

- **Meritocratic:** Power is earned by driving, not buying.
- **Self-Decaying:** Influence dilutes if you stop driving (10% monthly inflation).
- **Hard-Guarded:** Requires a 10 ICP deposit to prevent proposal spam.
- **Type-Safe Execution:** Proposal targets are enforced at the type level — it is impossible to construct an invalid Canister → Method combination.

---

## 2. Glossary

| Term | Definition |
|------|-----------|
| **DRV Token / Voting Point** | The non-tradable governance points earned daily by active drivers. 1 completed, verified trip equals 1 Voting Point. |
| **Monthly Inflation** | A 10% monthly inflation mechanism applied to the total supply of voting points, ensuring that recent activity holds more weight than past activity. |
| **Founder Security Faucet** | An initial allocation of voting power given to founders to protect the nascent DAO from malicious takeovers. This allocation extracts zero profit and mathematically decays to zero over a 5-year period. |
| **Proposal Deposit** | A deposit of 10 ICP required to submit a voting proposal, preventing bad actors from spamming the network. |
| **Protocol Treasury** | The central pool of funds collected from sustainability fees and rejected proposal deposits, used to cover compute cycles and fund future developments. |
| **ProposalTarget** | A nested variant type that encodes both the target canister type and its allowed governance method in a single, compiler-enforced value. |

---

## 3. Definitions

## 3.1 Constants
```motoko
// constants.mo
module Constants {
   public let GATEWAY_CANISTER_ID : Principal = Principal.fromText("...");
   public let REGISTRY_CANISTER_ID : Principal = Principal.fromText("...");
   public let MATCHING_CANISTER_ID : Principal = Principal.fromText("...");
   public let PAYMENT_CANISTER_ID : Principal = Principal.fromText("...");
   public let REPUTATION_CANISTER_ID : Principal = Principal.fromText("...");
   public let AI_CANISTER_ID : Principal = Principal.fromText("...");

   public let VOTING_PERIOD_FAST : Nat64 = 3 * 24 * 60 * 60;  // 3 days in seconds
   public let VOTING_PERIOD_SLOW : Nat64 = 5 * 24 * 60 * 60;  // 5 days in seconds
   public let PROPOSAL_DEPOSIT : Nat64 = 10 * 100_000_000;  // 10 ICP in e8s
   public let INFLATION_RATE_WEEKLY : Float = 0.1/30 * 7;  // 10% monthly inflation distributed weekly 2.333%

   public let DEPOSIT_EXPIRY_PERIOD : Nat64 = 30 * 24 * 60 * 60;  // 30 days in seconds
   public let TOKENS_DISTRIBUTION_PERIOD : Nat64 = 24 * 60 * 60;  // 24 hours in seconds

   // Not allowed to change via governance. Fixed buy design for security reasons.
   public let FOUNDER_SECURITY_BASE_PERCENT : Float = 50.0;  // 50% of total distributed tokens
   public let FOUNDER_SECURITY_PERIOD : Nat64 = 5 * 365 * 24 * 60 * 60;  // 5 years in seconds
}
```

### 3.2 Types

```motoko
// types.mo
module Types {

    // ==========================================
    // --- 1. CORE ALIASES                    ---
    // ==========================================

    public type CanisterId  = Principal;
    public type AccountId   = Principal;
    public type ProposalId  = Nat64;
    public type Timestamp   = Nat64;   // nanoseconds since Unix epoch (ICP Ledger standard)
    public type Subaccount  = Blob;    // 32-byte unique subaccount identifier



    // ==========================================
    // --- 2. PROPOSAL TARGET TYPES           ---
    // ==========================================

    /// A single, compiler-enforced type that binds a target canister type
    /// to its set of allowed governance methods and their argument payloads.
    ///
    /// Rules:
    ///   - Only explicitly listed methods can be proposed for each canister.
    ///   - Adding a new governable canister requires adding a new variant branch here.
    ///   - It is impossible to construct an invalid Canister → Method combination.
    public type ProposalTarget = {
        #Governance : GovernanceMethod;
        #Gateway    : GatewayMethod;
        #Matching   : MatchingMethod;
        #Registry   : RegistryMethod;
    };

    /// All governance-callable methods on the Governance canister.
    public type GovernanceMethod = {
        #UpdateFee     : GovernanceGovernanceUpdateFeeContract;
    };

    /// All governance-callable methods on the Gateway canister.
    public type GatewayMethod = {
        #UpdateFee     : GatewayGovernanceUpdateFeeContract;
        #UpdatePeriods : GatewayGovernanceUpdatePeriodsContract;
        #UpdateLimits  : GatewayGovernanceUpdateLimitsContract;
        #Quarantine    : GatewayGovernanceQuarantineContract;
    };

    /// All governance-callable methods on the Matching canister.
    /// Extend as Matching governance endpoints are defined.
    public type MatchingMethod = {
        #UpdateRadius : { meters : Nat32 };
    };

    /// All governance-callable methods on the Registry canister.
    public type RegistryMethod = {
        #UpdateFees           : RegistryGovernanceUpdateFeesContract;
        #SuspendDriver        : RegistryGovernanceSuspendDriverContract;
        #BanDriver            : RegistryGovernanceBanDriverContract;
        #UpdateInspectorCount : RegistryGovernanceUpdateInspectorCountContract;
        #UpdateInspectionFee  : RegistryGovernanceUpdateInspectionFeeContract;
    };


    // ==========================================
    // --- 3. USER / VOTING POINT TYPES       ---
    // ==========================================

    public type UserTokensDao = {
        owner      : AccountId;
        balance    : Nat;
        tripCount  : Nat;
        lastActive : Timestamp;
    };


    // ==========================================
    // --- 4. PROPOSAL TYPES                  ---
    // ==========================================

    public type ProposalStatus = {
        #Pending;
        #Active;
        #Accepted;
        #Rejected;
        #Executed;
    };

    /// Defines what a proposal will execute if it passes.
    /// `target` carries the canister type, the method, and the typed arguments in one value.
    /// `targetCanister` is the concrete Principal of the canister to call.
    /// `encodedArguments` is the Candid-encoded argument blob passed to the inter-canister call.
    public type ProposalPayloadDao = {
        target           : ProposalTarget;  // Canister type + method + args (compiler-enforced)
        targetCanister   : CanisterId;      // Concrete Principal for the inter-canister call
        encodedArguments : Blob;            // Candid-encoded arguments for the actual IC call
    };

    public type DepositStatus = {
        #pending;
        #paid;
    };

    public type DepositDao = {
        id         : ProposalId;    // ID of the proposal this deposit is for
        proposer   : AccountId;     // The account that made the deposit
        subaccount : Subaccount;    // Unique subaccount for this deposit
        amount     : Nat64;         // Deposit amount in e8s (ICP smallest unit)
        status     : DepositStatus; // Default: #pending
        consumed   : Bool;          // Set to true when deposit is used for registration. Default: false
        createdAt  : Timestamp;     // When the deposit record was created
    };

    public type ProposalDao = {
        id                : ProposalId;
        proposer          : AccountId;
        title             : Text;
        description       : Text;
        payload           : ProposalPayloadDao;
        status            : ProposalStatus;
        votesFor          : Nat;
        votesAgainst      : Nat;
        createdAt         : Timestamp;
        votingEndsAt      : Timestamp;
    };

    // ==========================================
    // --- 5. VOTE TYPES                      ---
    // ==========================================

    public type VoteReceiptDao = {
        voter       : AccountId;
        proposalId  : ProposalId;
        voteType    : { #Approve; #Reject };
        votingPower : Nat;  // Snapshot of the voter's DRV balance at the time of voting
    };
}
```

---

### 3.2 Governance Contracts (Gateway)

These types are used in `ProposalTarget.#Gateway` method payloads and are also consumed directly by the Gateway canister endpoints.

```motoko
// types.governance.mo
module Types {
    // ==========================================
    // --- 1. GOVERNANCE GOVERNANCE TYPES     ---
    // ==========================================

    /// Payload for governanceUpdateFee — adjusts Gateway and Clone registration fees.
    public type GovernanceGovernanceUpdateFeeContract = {
        proposalDeposit : Nat;
    };

    public type GovernanceGovernanceUpdateFeeResponse = {
        #ok  : { fees : GovernanceGovernanceUpdateFeeContract };
        #err : Text;
    };

    // ==========================================
    // --- 2. REGISTRY GOVERNANCE TYPES       ---
    // ==========================================

    /// Payload for registryGovernanceUpdateFees — adjusts driver/rider registration fees.
    public type RegistryGovernanceUpdateFeesContract = {
        driverRegistrationFee : Nat;
        riderRegistrationFee  : Nat;
    };

    public type RegistryGovernanceUpdateFeesResponse = {
        #ok  : { fees : RegistryGovernanceUpdateFeesContract };
        #err : Text;
    };

    /// Payload for registryGovernanceSuspendDriver — temporarily restricts a driver.
    public type RegistryGovernanceSuspendDriverContract = {
        driver         : AccountId;
        suspendedUntil : Timestamp;
        reason         : Text;
    };

    public type RegistryGovernanceSuspendDriverResponse = {
        #ok  : { driver : AccountId };
        #err : Text;
    };

    /// Payload for registryGovernanceBanDriver — permanently removes a driver.
    public type RegistryGovernanceBanDriverContract = {
        driver : AccountId;
        reason : Text;
    };

    public type RegistryGovernanceBanDriverResponse = {
        #ok  : { driver : AccountId };
        #err : Text;
    };

    /// Payload for registryGovernanceUpdateInspectorCount — adjusts peer inspection parameters.
    public type RegistryGovernanceUpdateInspectorCountContract = {
        minConfirmations : Nat;  // Min inspectors that must confirm (1–3)
        maxInspectors    : Nat;  // Max inspectors assigned per candidate (1–3)
    };

    public type RegistryGovernanceUpdateInspectorCountResponse = {
        #ok  : { minConfirmations : Nat; maxInspectors : Nat };
        #err : Text;
    };

    /// Payload for registryGovernanceUpdateInspectionFee — adjusts fee paid to each confirming inspector.
    public type RegistryGovernanceUpdateInspectionFeeContract = {
        inspectionFeePerInspector : Nat;  // ICP e8s paid to each confirming inspector
    };

    public type RegistryGovernanceUpdateInspectionFeeResponse = {
        #ok  : { inspectionFeePerInspector : Nat };
        #err : Text;
    };

    // ==========================================
    // --- 3. GATEWAY GOVERNANCE TYPES        ---
    // ==========================================

    /// Payload for governanceUpdateFee — adjusts Gateway and Clone registration fees.
    public type GatewayGovernanceUpdateFeeContract = {
        gatewayRegistrationFee  : Nat;
        canisterRegistrationFee : Nat;
        depositExpiryPeriod     : Nat;
    };

    public type GatewayGovernanceUpdateFeeResponse = {
        #ok  : { fees : GatewayGovernanceUpdateFeeContract };
        #err : Text;
    };

    /// Payload for governanceUpdatePeriods — adjusts sync and quarantine durations.
    public type GatewayGovernanceUpdatePeriodsContract = {
        quarantineExpiryPeriod : Nat;
        gatewaySyncPeriod      : Nat;
        gatewaySyncAllPeriod   : Nat;
        cloneSyncPeriod        : Nat;
    };

    public type GatewayGovernanceUpdatePeriodsResponse = {
        #ok  : { periods : GatewayGovernanceUpdatePeriodsContract };
        #err : Text;
    };

    /// Payload for governanceUpdateLimits — adjusts call-rate limits.
    public type GatewayGovernanceUpdateLimitsContract = {
        gatewaySyncLimit    : Nat;
        gatewaySyncAllLimit : Nat;
        cloneSyncLimit      : Nat;
    };

    public type GatewayGovernanceUpdateLimitsResponse = {
        #ok  : { limits : GatewayGovernanceUpdateLimitsContract };
        #err : Text;
    };

    /// Payload for governanceQuarantine — quarantine or lift quarantine on a Gateway.
    public type GatewayGovernanceQuarantineContract = {
        gateway         : CanisterId;
        quarantineUntil : Timestamp;
        reason          : Text;
    };

    public type GatewayGovernanceQuarantineResponse = {
        #ok  : { gateway : CanisterId };
        #err : Text;
    };
}
```

---

## 4. Internal State Variables

```motoko
//state.mo
// Voting point ledger — maps driver principals to their earned DRV balance.
governanceBasePoint : Float64 = 1.0;
founderBasePercent : Float64 = 50.0;
userTokens : TrieMap<AccountId, UserTokensDao>

// Proposal store — all historical and active proposals.
proposals  : TrieMap<ProposalId, ProposalDao>

```
---

```
//main.mo
// --- Aggregate counters ---
var _governanceSubaccount : Subaccount;
var _totalUserTokensSupply : Nat = 0;  // Global DRV supply; used to calculate 10% monthly inflation
var _founderFaucetBalance  : Nat = 0;  // Remaining founder security balance; decays to 0 over 5 years

// --- Version (incremented on every canister upgrade) ---
// Used in postupgrade migrations to detect state schema changes.
var _version : Nat = 1;
```
---

## 5. Public API

### 5.1 Submit Proposal

```
// Public. Any participant may call. No rate limit.
submitProposal(title, description, payload, paymentSubaccount) -> Result<ProposalId, Error>

  Validators (in order):
    1. Caller has a DRV balance > 0.
    2. Exactly 10 ICP has been deposited into paymentSubaccount on the ICP Ledger.
    3. payload.target is a valid ProposalTarget value (enforced by type system).

  On success:
    - Register proposal with status = #Active.
    - Set votingEndsAt = now + voting period duration.
    - Return ProposalId.

  On any validation failure:
    - Return #err with descriptive Error.
```

---

### 5.2 Cast Vote

```
// Public. Caller must be a registered driver with DRV balance > 0.
castVote(proposalId, voteType) -> Result<VoteReceiptDao, Error>

  Validators (in order):
    1. Caller has a DRV balance > 0.
    2. Proposal with proposalId exists and is #Active.
    3. Caller has not already voted on this proposal.

  On success:
    - Record VoteReceiptDao with votingPower = caller's current DRV balance (snapshot).
    - Update proposal votesFor or votesAgainst accordingly.
    - Return VoteReceiptDao.
```

---

### 5.3 Record Trip Activity

```
// Internal. msg.caller must be the registered Ride Matching canister.
recordTripActivity(driverId, tripDetails) -> Result<Nat, Error>

  Validators (in order):
    1. msg.caller is the registered Ride Matching canister principal.

  On success:
    - Mint 1 DRV point and add it to the driver's ledger balance (1 Trip = 1 Point).
    - Increment _totalUserTokensSupply by 1.
    - Update driver's lastActive timestamp.
    - Return new DRV balance.
```

---

### 5.4 Execute Proposal

```
// Public. Can be triggered by a worker or any user once voting ends.
executeProposal(proposalId) -> Result<(), Error>

  Validators (in order):
    1. Proposal with proposalId exists.
    2. Proposal status is #Accepted or #Rejected (voting period has ended).

  On #Accepted:
    - Dispatch an inter-canister call to payload.targetCanister using the method
      encoded in payload.target. Arguments are taken from payload.encodedArguments.
    - Switch on payload.target to resolve the correct endpoint:

        switch (payload.target) {
            case (#Governance(#UpdateFee(args)))                    { /* call governanceGovernanceUpdateFee(args) */ };
            case (#Gateway(#UpdateFee(args)))                       { /* call governanceUpdateFee(args) */ };
            case (#Gateway(#UpdatePeriods(args)))                   { /* call governanceUpdatePeriods(args) */ };
            case (#Gateway(#UpdateLimits(args)))                    { /* call governanceUpdateLimits(args) */ };
            case (#Gateway(#Quarantine(args)))                      { /* call governanceQuarantine(args) */ };
            case (#Matching(#UpdateRadius(args)))                   { /* call matchingUpdateRadius(args) */ };
            case (#Registry(#UpdateFees(args)))                     { /* call registryGovernanceUpdateFees(args) */ };
            case (#Registry(#SuspendDriver(args)))                  { /* call registryGovernanceSuspendDriver(args) */ };
            case (#Registry(#BanDriver(args)))                      { /* call registryGovernanceBanDriver(args) */ };
            case (#Registry(#UpdateInspectorCount(args)))           { /* call registryGovernanceUpdateInspectorCount(args) */ };
            case (#Registry(#UpdateInspectionFee(args)))            { /* call registryGovernanceUpdateInspectionFee(args) */ };
        };

    - On success: mark proposal as #Executed. Return 10 ICP deposit to proposer.
    - On inter-canister failure: keep proposal as #Accepted for retry. Do not return deposit yet.

  On #Rejected:
    - Sweep 10 ICP deposit into the protocol treasury.
    - Mark proposal as #Executed.
```
---

### 5.5 Governance Governance Update Fee

```
// Public. Can be triggered by a worker or any user once voting ends.
governanceGovernanceUpdateFee(args : GovernanceGovernanceUpdateFeeContract) -> Result<(), Error>

  Validators (in order):
    1. Proposal with proposalId exists.
    2. Proposal status is #Accepted or #Rejected (voting period has ended).

  On success:
    - Update the fee in the Governance canister.
    - Mark proposal as #Executed.
```

## 6. Business Rules & Invariants

### 6.1 Monthly Inflation Mechanism

| Rule | Detail |
|------|--------|
| Inflation rate | 10% of `_totalUserTokensSupply` minted monthly |
| Distribution | Distributed proportionally across all active DRV holders |
| Trigger | Background worker/timer calls `distributeInflation()` monthly |
| Effect | A new active driver gains ~50% of peak relative voting power in 2 months, ~95% in 6 months |

> **Invariant:** `_totalUserTokensSupply` must always equal the sum of all `UserTokensDao.balance` values before inflation distribution.

---

### 6.2 Founder Security Decay

| Rule | Detail |
|------|--------|
| Initial allocation | Zero intial allocation. Founder gets voting points from `founderBasePercent` of total distributed tokens every week |
| Decay period | Linear decay to 0 over 5 years |
| Purpose | Prevents bad actors from hijacking the protocol before the community grows |
| Profit | Zero — founders extract no financial value from this balance |

> **Invariant:** `_founderFaucetBalance` must only decrease. It must reach 0 at or before the 5-year mark.

---

### 6.3 Proposal Deposit Rules

| Rule | Detail |
|------|--------|
| Required amount | 10 ICP (in e8s) |
| Verification | Canister checks ICP Ledger balance of `paymentSubaccount` before accepting |
| On acceptance | Deposit returned to proposer after `#Executed` |
| On rejection | Deposit swept into protocol treasury |

---

### 6.4 ProposalTarget Invariant

The `ProposalTarget` type is the **single source of truth** for which canister types have governable endpoints and what those endpoints accept.

| Rule | Detail |
|------|--------|
| Compiler-enforced | Invalid Canister → Method pairs cannot be constructed |
| Exhaustive switch | `executeProposal` must handle every branch; the compiler rejects incomplete switches |
| Extension pattern | Adding a new governable canister = adding a new variant to `ProposalTarget` and a new `*Method` type |

---

### 6.5 Governance points/token distribution

//distributeInflation() is called every Monday at about 00:00 UTC
Every Monday at about 00:00 UTC worker.mo will inflate governanceBasePoint by `(10%/30) * 7`. All active drivers will recieve `governanceBasePoint * tripCount` DRV points.
Founder faucet recieves voting tokens from total amount of distributed this week tokens. This amount decades every week untill reaches 0 in 5 years.

### 6.6 Voting timeline

Voting can only start after voting tokens distribution and must finish before next voting tokens distribution. So voting period is a bit less than 7 days.
This greatly simplifies and removes need for complex time based logic. We check balances at the moment of voting and it will not change untill vote finishes. This approach saves cycles. In case we have active vote and new token distribution should start, we will distribute them next Monday.

### 6.7 Founder faucet decay
founderBasePercent = 50.0% first week and decades linearly to 0 over 5 years. Decay is calculated every year.
1 year - 50%
2 year - 40%
3 year - 30%
4 year - 20%
5 year - 10%
6 year - 0%

## 7. Architecture

```
canister_governance/
├── main.mo         — Actor entry point. Exposes all public endpoints. Holds _version.
├── state.mo        — Stable state definitions. All maps and counters. Upgraded via postupgrade.
├── types.mo        — All shared type definitions (module Types from Section 3).
├── types.governance.mo   — Governance contract types (Section 3.2) and inter-canister call logic.
├── logic.mo        — All business logic (inflation, proposal lifecycle, vote tallying).
├── validators.mo   — Pure validation functions called from main.mo before endpoint logic.
├── math.mo         — Pure helper functions (rounding, timestamp arithmetic, inflation calculations, etc.)
└── worker.mo       — Heartbeat-driven background tasks:
                    · distributeInflation() weekly 
                    · Trigger executeProposal() for expired proposals
                    · Performs cleaning tasks to keep canister state small not to bloat

tests/
├── tests.[module_name].mops  — Unit tests (per module)
└── integration.[flow].mops   — End-to-end integration tests simulating proposal lifecycle flows
```

**Key technology choices:**
- `Tier.Map` — for ordered, stable `TrieMap`-compatible state storage in `state.mo`
- `--enhanced-orthogonal-persistence` — enables automatic stable memory management across upgrades without manual `preupgrade`/`postupgrade` serialisation of simple types.

---

## 8. Upgrade & Migration Strategy

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

## 9. Open Items / Future Work

| Item | Priority | Notes |
|------|----------|-------|
| Voting period duration constant | High | Define in `constants.mo`; currently unspecified |
| Monitoring / proposal dashboard | Low | `supportCounters()`-style endpoint to be added |
