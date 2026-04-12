# Governance Canister — Specification v1
> **Status:** Draft · **Last updated:** 2026-04-12  
> **Language:** Motoko · **Platform:** Internet Computer Protocol (ICP)

## 1. Overview & Vision
The Governance Canister is the local brain of a Hydra Mobility instance. It manages the **DRV (Voting Point)** ledger, handles driver-submitted proposals, and executes changes on the regional Gateway and Matching canisters. 

Key design properties:
- **Meritocratic:** Power is earned by driving, not buying.
- **Self-Decaying:** Influence dilutes if you stop driving (10% monthly inflation).
- **Hard-Guarded:** Requires a 10 ICP deposit to prevent proposal spam.

--------------------------------------------------------------------------------
2. Glossary
Term
Definition
DRV Token / Voting Point
The non-tradable governance points earned daily by active drivers. 1 completed, verified trip equals 1 Voting Point
.
Monthly Inflation
A 10% monthly inflation mechanism applied to the total supply of voting points, ensuring that recent activity holds more weight than past activity
.
Founder Security Faucet
An initial allocation of voting power given to founders to protect the nascent DAO from malicious takeovers. This allocation extracts zero profit and mathematically decays to zero over a 5-year period
.
Proposal Deposit
A deposit of 10 ICP required to submit a voting proposal, preventing bad actors from spamming the network
.
Protocol Treasury
The central pool of funds collected from the 0.1% - 1% sustainability fees and rejected proposal deposits, used to cover compute cycles and fund future developments
.

--------------------------------------------------------------------------------
3. Core Definitions & Types (Motoko)
module GovernanceTypes {
    
    public type AccountId = Principal;
    public type ProposalId = Nat64;
    
    // --- 1. VOTING POINT (DRV) LEDGER TYPES ---
    public type DRVAccount = {
        owner : AccountId;
        balance : Nat; 
        lastActive : Int;
    };

    // --- 2. PROPOSAL TYPES ---
    public type ProposalStatus = {
        #Active;
        #Accepted;
        #Rejected;
        #Executed;
    };

    public type ProposalPayload = {
        // Defines the target canister and the action to be executed if passed
        targetCanister : Principal;
        methodName : Text;
        encodedArguments : Blob; 
    };

    public type Proposal = {
        id : ProposalId;
        proposer : AccountId;
        title : Text;
        description : Text;
        payload : ProposalPayload;
        depositSubaccount : Blob; // For the 10 ICP deposit verification
        status : ProposalStatus;
        votesFor : Nat;
        votesAgainst : Nat;
        createdAt : Int;
        votingEndsAt : Int;
    };

    // --- 3. VOTE TYPES ---
    public type VoteReceipt = {
        voter : AccountId;
        proposalId : ProposalId;
        voteType : { #Approve; #Reject };
        votingPower : Nat; // Snapshot of their DRV balance at the time of voting
    };
    
    public type GovernanceAction = {
    	#SetSustainabilityFee : Nat64;
    	#UpdateMatchingAlgorithm : Blob; // WASM hash
    	#BlacklistDriver : Principal;
    	#TransferTreasury : { to: Principal; amount: Nat64 };
	};

}

--------------------------------------------------------------------------------
4. Internal State Variables
drvLedger : TrieMap<AccountId, DRVAccount> - Maps driver principals to their earned voting points.
proposals : TrieMap<ProposalId, Proposal> - Stores all historical and active network proposals.
totalDrvSupply : Nat - Tracks the global supply to accurately calculate the 10% monthly inflation
.
founderFaucetBalance : Nat - Tracks the remaining security voting power allocated to founders, which decays over 5 years
.

--------------------------------------------------------------------------------
5. Public API
5.1 submitProposal(title, description, payload, paymentSubaccount) -> Result<ProposalId, Error>
Allows any participant to submit a governance proposal. Validators:
The canister verifies that exactly 10 ICP has been deposited into the single-use paymentSubaccount
. Logic:
Registers the proposal as #Active and sets the voting period.
5.2 castVote(proposalId, voteType) -> Result<VoteReceipt, Error>
Allows a driver to vote on an active proposal using their accumulated DRV balance. Validators:
Caller must have a DRV balance > 0.
Proposal must be #Active.
5.3 recordTripActivity(driverId, tripDetails) -> Result<Nat, Error>
Called securely by the Ride Matching Canister upon successful trip completion. Logic:
Mints 1 DRV point and adds it to the driver's ledger balance (1 Trip = 1 Point)
.
5.4 executeProposal(proposalId) -> Result<(), Error>
Can be triggered by a worker or any user once a proposal's voting period ends. Logic:
If #Accepted, the payload is executed via inter-canister call to the target (e.g., updating the fee rate in the Gateway canister). The 10 ICP deposit is returned to the proposer
.
If #Rejected, the 10 ICP deposit is swept into the protocol treasury
.

--------------------------------------------------------------------------------
6. Business Rules & Invariants
6.1 The 10% Monthly Inflation Mechanism
To ensure that the network is always governed by the people actively driving today, the total supply of voting points inflates by 10% every month
.
Invariant: A background worker/timer triggers distributeInflation() monthly.
Result: A new active driver will gain about 50% of peak relative voting power in two months, 70% in four months, and 95% in six months, while the influence of veteran drivers who stop driving naturally dilutes
.
6.2 Founder Security Decay
Founders receive an initial voting allocation to prevent bad actors from hijacking the protocol before the community grows
.
Invariant: This balance decays mathematically to 0 over a 5-year period
. Founders extract absolutely zero profit from this; it is strictly for network security
.
6.3 Unlimited Customization
The governance canister can execute updates to any parameter in the protocol. Drivers can vote to adjust pricing based on high demand, bad road conditions, or the number of passengers
. Furthermore, protocol rules can be adapted for specific local city requirements (e.g., Vilnius city protocol rules vs. Kaunas city protocol rules)
