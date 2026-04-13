# Driver-Owned Decentralized Ride Hailing Protocol on ICP

A fully decentralized ride-hailing protocol built on the Internet Computer (ICP) that replaces centralized intermediaries with a driver-owned governance and execution model. This protocol aims to eliminate the 10-30% commission rates seen in centralized platforms, opting instead for a near-zero sustainability fee to cover network cycles and development.

Group on FB: www.facebook.com/groups/ridehailing/

Page on ICP: https://4nbfz-saaaa-aaaas-qfbea-cai.icp0.io/

## Current progress

Markup documentation for AI generated code progress:
> 1. Gateway canister - done `canister_gateway_v1.md`
2. Governance canister - in progress
3. Registry canister - not started
4. Matching canister - not started
5. Treasury & Escrow canister - not started
6. Reputation & Arbitration canister - not started
7. AI Inference canister - not started

You can support this development by donating ICP (Internet Computer Protocol) tokens to the following address:

`cf538348dc493ade831bfdb138ac68d6299bbc964cee4f2a1984aa5123774765`

Donated: 0 ICP (last updated 2026-04-09)

## Project Structure

The protocol comprises several core components designed for infinite scalability, decentralized governance, and a secure operational flow.

### Frontend Progressive Web App (PWA)

*   **Technology Stack**: Built as a Single Page Application (SPA) using **SolidJS**, completely fully on-chain.
*   **Accessibility**: Engineered as a Progressive Web App, enabling direct browser installation across mobile operating systems (Android/iOS) and thereby bypassing centralized app store constraints and fees.
*   **Offline Resilience**: Employs a Service Worker strategy to cache the UI shell and essential ride data, ensuring drivers maintain functionality during intermittent connectivity losses.
*   **Authentication**: Exclusively utilizes **Internet Identity** (WebAuthn) for secure, device-bound authentication, eliminating centralized password databases.

### Backend Smart Contracts (Canisters)

The backend runs entirely on ICP smart contracts (canisters) to maintain a decentralized state.

1.  **Gateway Canister** is the directory and routing backbone of the decentralised Hydra Mobility Mesh. 
2.  **Governance Canister**: The central authority managing the Voting Point ledger. There is no tradable governance token. Active drivers earn "Voting Points" (1 trip = 1 point), with a 10% monthly inflation mechanism to prioritize recent activity.
3.  **Registry Canister**: Manages driver/rider registration, hashes KYC data to the chain (for liveness checks, is used: https://id.decideai.xyz/), and handles deterministic regional mapping.
4.  **Payment Canister**: Holds rider payments in escrow during transit and automates the deduction of the driver-governed Sustainability Fee (0.1%-1%) before settling funds.
5.  **Reputation Canister**: An immutable ledger of rating events. Employs a peer-jury model for decentralized dispute resolution with slashing mechanisms to penalize malicious behavior.
6.  **Matching Canister (City-Sharded)**: Dedicated canisters for specific geographic areas (e.g., Vilnius) to prevent state bloat. Manages the ride lifecycle (`RIDE_REQUESTED` to `SETTLED`) via update calls and incorporates lightweight proximity filtering algorithms.
7.  *(Optional)* **AI Canister**: Uses ICP’s on-chain AI to detect fraudulent ride patterns through geolocation hash deviations and compute dynamic surge pricing proposals.

## Technical Economics (The "Reverse Gas" Model)

Because ICP operates on a "reverse-gas" model, the application pays for its own computation; end users and drivers do not need to hold cryptocurrency. The protocol is economically highly viable:

*   **Cost Estimate**: The sharded architecture (assuming 1000 drivers) will burn roughly $10k–$30k worth of compute cycles annually.
*   **Treasury Revenue**: Even if drivers contribute a low flat fee (e.g., €20/month) rather than a 20% commission, the treasury over-collateralizes network compute costs by over 10x, with remaining funds allocated via DAO governance to development grants or system marketing.

## Security & Anti-Fraud Architecture

*   **Location Integrity**: Does not store full GPS traces. Drivers submit a cryptographic hash (`hash(lat, lon, timestamp)`) periodically to prove location compliance without overwhelming storage limits.
*   **Data Archival**: To ensure infinite scaling without canister memory limits, the Ride Matching canisters continuously push completed rides into time-sharded, read-only Archive Canisters in strongly agregrated form. No long term reporting data is stored to save cycles. 


