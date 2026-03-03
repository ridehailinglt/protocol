## Prompt 1: SolidJS Progressive Web App (Asset Canister)

**Context for AI:**  
Build a Web3 frontend for a decentralized ride-hailing protocol deployed 100% on-chain on the Internet Computer using the :contentReference[oaicite:0]{index=0}. Do not use external Web2 databases or centralized CDNs.

### Architecture & Stack
- Use SolidJS for a Single Page Application (SPA).
- Configure as a Progressive Web App (PWA):
  - Include `manifest.json`.
  - Implement a Service Worker for offline caching of:
    - UI shell.
    - Active ride state.
    - Last known driver list.
- Authentication must strictly use :contentReference[oaicite:1]{index=1} (WebAuthn), generating a Principal ID.

### Core Client State (Stores)
Create reactive SolidJS stores:
- `authStore`
- `rideStore`
- `driverStore`
- `governanceStore`
- `reputationStore`

### Network Logic
- Implement polling via lightweight query calls every 2–3 seconds for status updates.
- If the connection drops:
  - Batch-submit queued update calls upon reconnection.
- All economic and state-changing actions (e.g., accepting a ride) must strictly use update calls.


---

## Prompt 2: Root Governance & Voting Point Canister

**Context for AI:**  
Build the core governance brain of the protocol.

### Voting Point Ledger Logic
- No hard-capped tokens.
- Implement a Voting Point ledger:
  - 1 completed trip = 1 Voting Point minted to the driver’s Principal ID.
- Implement a 10% monthly inflation function:
  - Inflate total supply mathematically.
  - Ensure recent activity carries more voting weight than historical activity.
- Implement a Security Faucet for founders:
  - Grants initial voting power.
  - Voting weight decays to exactly 0 over 5 years using:
    - `weight = base_tokens × e^(-kt)`

### Governance Logic
Implement proposal submission structs for:
- Protocol parameter changes.
- Sustainability fee adjustments (0.1%–1%).
- Algorithm modifications.
- Driver bans.
- Treasury allocations.

Additional requirements:
- Maximum voting caps per identity.
- Slashing functions for malicious proposals.


---

## Prompt 3: Registry & Identity Canister

**Context for AI:**  
Build the canonical registry for network participants.

### State & Functions
- Maintain mapping:
  - `Principal -> DriverProfile`
- Store driver KYC and license verifications strictly as on-chain hashes.
  - Do not store raw PII.
- Implement deterministic regional sharding:
  - Example shard: `VilniusMatching`
  - Driver assignment must be algorithmically reproducible.


---

## Prompt 4: Ride Matching Canister (Sharded by City)

**Context for AI:**  
Build the high-frequency ride matching engine. Duplicate per city to prevent state bloat.

### Ride State Machine
Strict transition flow via update calls only:

`RIDE_REQUESTED → DRIVER_RESERVED → DRIVER_ACCEPTED → DRIVER_ARRIVED → RIDE_STARTED → RIDE_COMPLETED → SETTLED`

### Matching Logic
- Radius filter algorithm.
- Use:
  - Driver availability flags.
  - Minimum reputation thresholds.

### Location Privacy
- Do not store full GPS traces.
- Accept periodic location updates as:
  - `hash(lat, lon, timestamp)`

### Data Management
- Store only active rides in memory.
- Upon `SETTLED`:
  - Trigger inter-canister call.
  - Move data to a time-sharded Archive Canister.


---

## Prompt 5: Treasury & Escrow Canister

**Context for AI:**  
Build the financial settlement layer.

### Core Logic
- Implement escrow:
  - Rider locks estimated fare before `RIDE_STARTED`.
- Upon `SETTLED`:
  - Release payment instantly to driver.
  - Deduct Sustainability Fee (0.1%–1%) defined by governance.
- Pool Sustainability Fees:
  - Public treasury balance.
  - Used for:
    - Automated cycle top-ups.
    - Developer grants.


---

## Prompt 6: Reputation & Arbitration Canister

**Context for AI:**  
Build the immutable reputation and community justice system.

### Reputation State
- Append-only log of rating events.
- Weighted score algorithm:
  - Emphasize recent ride history.

### Arbitration State
- Dispute case creation:
  - Evidence references stored as hashes only.
- Randomized jury selection:
  - Select active drivers.
- Include:
  - Slashing mechanism.
  - Time-bounded voting windows.


---

## Prompt 7: On-Chain AI Canister (Optional but Recommended)

**Context for AI:**  
Build a canister utilizing DFINITY’s on-chain AI execution environment on the :contentReference[oaicite:2]{index=2}.

### Inference Tasks

**Fraud Detection**
- Inputs:
  - Ride pattern anomalies.
  - Cancellation frequency.
  - Rating variance.
  - Geolocation hash deviations.
- Output:
  - Continuous fraud score.
  - Automatic flagging of suspicious accounts.

**Surge Pricing**
- Detect zone imbalance.
- Output dynamic surge multipliers.
- Must respect limits set by Governance Canister.
