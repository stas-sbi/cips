## CIP-00XX Mint Canton Coin from Unminted/Unclaimed Pool
<pre>
  Title: Enable escrow mechanisms and governance votes to allow Canton Coin minting from the unminted pool
  Author:
    Jose Velasco
  Status: Draft
  Type: Tokenomics
  Created: 2025-06-18
  Approved: 
  License: CC-1.0
</pre>

## Abstract

This proposal introduces a new governance mechanism allowing Super Validators (SVs) to initiate a vote to authorize the minting of Canton Coin from an unclaimed (unminted) reward pool, helping to operationalize the milestone-based CIP rewards structure and aligning with the first deliverable of CIP-0058: Enable escrow mechanisms to ensure Super Validators deliver value in return for rewards.

## Copyright

This CIP is licensed under CC0-1.0: [Creative Commons CC0 1.0 Universal](https://creativecommons.org/publicdomain/zero/1.0/)

## Specification

This CIP introduces a new on-ledger voting-based governance flow that enables Super Validators (SVs) to propose the creation of `UnallocatedUnclaimedActivityRecord` contracts, representing authorized but not yet allocated Amulet rewards.

### High-level overview

To contextualize the protocol and governance changes introduced by this CIP, we outline both a generalized workflow and a specific escrow-based scenario. These flows illustrate how unminted rewards transition through governance-controlled minting, and how this mechanism enables milestone-based issuance of Canton Coin while preserving decentralization, auditability, and escrow guarantees.

#### General Flow

- Weighted activity records are assigned to a party based on their contributions within a round.

- If the assigned party fails to mint their rewards before the round concludes, those rewards are moved into the unminted pool.

- At a later stage—following an off-chain agreement—someone may propose that a specified party should receive a portion of the unminted pool.

- An on-chain vote is initiated to authorize the minting. Super Validators review and either approve or reject the proposal.



#### Specific Flow

- The GSF node hosts an SV weight on behalf of an SV in an escrow agreement.
- The GSF hosts a party for that SV on a separate GSF operated Validator, and names that party as the beneficiary of the SV weight.
- The SVs, in each round, credit the GSF SV node for work/activity, and create a weighted activity records assigned coupon, which the GSF node splits among all the SV rights hosted on that node.
- The GSF specifically disables the final minting action for all escrowed SV parties.
- All the coin earned by escrowed parties goes into the unminted pool and loses all connection to any party.
- When the SV passes a milestone, it then searches all the history to find weighted activity records assigned coupons that had been assigned to it, normalized / trimmed down to the weight it wants to claim.
- The SV presents these records, and the resulting escrow claim, to the GSF.
- GSF does its own calculation and makes a recommendation via an onchain vote.
- SVs accept or reject the vote.


### Governance Specification

- A new action requiring confirmation (`SRARC_CreateUnallocatedUnclaimedActivityRecord`) is added to `DsoRules_ActionRequiringConfirmation`.
- A new choice `DsoRules_CreateUnallocatedUnclaimedActivityRecord` allows the DSO to propose a beneficiary, reward amount, and reason for the reward.
- Upon successful vote confirmation, an `UnallocatedUnclaimedActivityRecord` contract is created. This contract has two expiry parameters:
  - `expiresAt`: Duration from creation, derived from the configurable DSO field `unallocatedUnclaimedActivityRecordTimeout` (defaulting to 24 hours).
  - `unclaimedActivityRecordExpiresAt`: Passed to the resulting activity record once allocation occurs.

### Allocation and Expiration

- A new SV-App trigger, `UnallocatedUnclaimedActivityRecordAllocatorTrigger`, locates eligible `UnclaimedReward` contracts, archives them up to the specified amount, and creates an `UnclaimedActivityRecord`.
- A second trigger, `UnallocatedUnclaimedActivityRecordExpiryTrigger`, expires unallocated records after their TTL.
- A third trigger, `UnclaimedActivityRecordExpiryTrigger`, expires unused `UnclaimedActivityRecord` contracts.

### Minting Mechanism

- `UnclaimedActivityRecord` is introduced as a new type of activity record that can be minted.
- It is added as a new variant (`InputUnclaimedActivityRecord`) within the existing `TransferInput` type.
- This input is consumed by `AmuletRules_Transfer`, which mints the corresponding Amulet during a self-transfer executed by automation in the Wallet App (`CollectRewardsAndMergeAmuletsTrigger`).

### Off-ledger Implementation Requirements

- **SV App**
  - Implement and deploy the three new triggers listed above.
  - Extend the Treasury Service to support the new activity record lifecycle.

- **Wallet App**
  - Extend the `CollectRewardsAndMergeAmuletsTrigger` to auto-collect `UnclaimedActivityRecord` using the new transfer input.

- **CLI App**
  - Introduce a Python CLI utility to calculate minted and unclaimed rewards by party ID and time range, reusing logic from `scan_txlog.py`.

- **SV App Voting UI**
  - Update the SV App voting UI to support proposals using `DsoRules_CreateUnallocatedUnclaimedActivityRecord`.

- **SV App Governance UI**
  - Add support in the SV App for configuring `unallocatedUnclaimedActivityRecordTimeout`.

## Motivation

This change is required to implement the reward distribution mechanism described in CIP-0058. 
The CIP defines a process whereby, upon successful milestone verification, the Tokenomics Working Group approves a 
reward amount and the Super Validators assign a portion of unclaimed rewards to be minted. 
This functionality leverages the standard voting process and introduces new ledger, off-ledger, and UI components to 
support reward calculation, archival of unclaimed rewards, and minting of the corresponding Amulet.
While motivated by the need to support milestone-based escrowed rewards, the proposed mechanism is more general.
It enables Super Validators to mint any type of unclaimed reward for any reason, not limited to escrowed incentives.
This flexibility supports future governance decisions and additional reward use cases beyond the scope of CIP-0058.

This aligns with the first deliverable of [CIP-0058](https://github.com/global-synchronizer-foundation/cips/blob/main/cip-0058/cip-0058.md): _Enable escrow mechanisms to ensure Super Validators deliver value in return for rewards_.


## Rationale

This proposal introduces a governance-backed, automated reward allocation mechanism that ensures Super Validators (SVs) never act as direct issuers of Amulets. Instead, it relies on existing voting infrastructure, explicit reward tracking, and on-ledger automation. The design satisfies the CIP-0058 objective of enabling escrowed value transfer with network approval, while preserving clear separation of duties between reward authorization and execution.

Super Validators need a way to reward parties for milestone-based work without directly issuing Amulets. The proposed CIP process allows:

- Proposing reward allocations through a formal vote explicitly approving the proposal.
- Tracking and escrowing those rewards in a verifiable, on-ledger structure.
- Allowing beneficiaries to receive their rewards without needing manual approval or signature from SVs.
- Ensuring archival of previous reward entitlements (UnclaimedReward contracts) during the minting process.

The final design introduces `UnallocatedUnclaimedActivityRecord` as a bridge between governance and execution. It supports clear auditability by representing approved but not yet funded reward intents. Once funded, these result in the creation of an `UnclaimedActivityRecord`, which serves as a transferable representation of reward entitlement. The `UnclaimedActivityRecord` is then consumed during a self-transfer initiated by automation in the Wallet App, resulting in the minting of the corresponding Amulet.

### Alternatives Considered

#### Approach 1 – Mint in One Step

This approach involved exercising a single voting choice (`AmuletRules_MintFromUnmintedPool`) that would simultaneously archive `UnclaimedReward` contracts and mint the Amulet.

Rejected due to:

- Risk of stale contract references caused by background reward merging.
- Inability to collect beneficiary signatures, which are required for minting.

#### Approach 2 – Post-Vote Execution with Extra Consequences

This design embedded additional side effects in the vote’s success path: archiving `UnclaimedReward`s and creating an `AmuletMintOffer` as part of the vote confirmation.

Rejected due to:

- Inflexibility in the current voting infrastructure, which expects one-to-one mapping between action and execution.
- Implementation would be non-trivial and hard to maintain.

#### Approach 3 – Parallel Offer and Burn Request Flow

Upon a successful vote, two contracts would be created:
1. An `AmuletMintOffer` for the beneficiary to accept.
2. A `BurnUnclaimedRewardsRequest` for the DSO to handle archival.

Rejected due to:

- Lack of atomicity, leading to potential inconsistencies.

#### Approach 4 – AmuletMintOffer Claimed Manually by Beneficiary

This design introduced `UnallocatedAmuletMintOffer`, accepted by an SV trigger that creates an `AmuletMintOffer`. The beneficiary would manually accept the final offer.

Rejected due to:

- The design would violate a core requirement of CIP-0058: SVs must not act as issuers.

### Justification for the Final Design

The chosen approach introduces a clean separation of concerns:

- Voting captures intent and approval.
- SVs allocate funding (`UnclaimedReward` contracts) without minting.
- The `UnclaimedActivityRecord` encapsulates minting rights in a transferable, on-ledger structure.
- Beneficiaries receive `Amulet` through a self-transfer initiated by automation in the Wallet App, maintaining signature and stakeholder integrity.
- The model avoids redundant templates or triggers and fits well into Splice’s composable architecture.

This design was refined through multiple iterations, peer reviews, and discussions across the engineering and product teams, converging on a solution that meets both technical constraints and governance objectives.


## Backwards compatibility

All choices and backend code involved are backward compatible.

## Reference implementation

The new implementation includes backend and Daml changes. 
An early version of the Daml code is available in the following forked Splice branch: 
https://github.com/jose-velasco-ieu/splice/tree/josevelasco/daml-minting-escrowed-canton-coin-2/cip-0058
