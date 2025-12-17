# Previous Governance Action ID

## Overview

In Conway era, governance actions that modify protocol state form **chains by purpose**. Each new proposal must reference the last enacted action of the same type (or nothing if it's the first action of that type).

**Goal**: Prevent conflicts and ensure **sequential application** of changes of the same type.

## Which Proposals Require Previous Gov Action ID?

| GovAction | Requires Previous? | Purpose | Shares Chain With |
|-----------|-------------------|---------|-------------------|
| **ParameterChange** | Yes | PParamUpdatePurpose | - |
| **HardForkInitiation** | Yes | HardForkPurpose | - |
| **NoConfidence** | Yes | CommitteePurpose | UpdateCommittee |
| **UpdateCommittee** | Yes | CommitteePurpose | NoConfidence |
| **NewConstitution** | Yes | ConstitutionPurpose | - |
| **TreasuryWithdrawals** | No | - | - |
| **InfoAction** | No | - | - |

## Four Independent Chains

The ledger maintains 4 independent chains (roots), one for each purpose:

```mermaid
graph LR
    subgraph "GovRelation (4 independent roots)"
        PP[grPParamUpdate] --> PP1[ParameterChange]
        HF[grHardFork] --> HF1[HardForkInitiation]
        CM[grCommittee] --> CM1[NoConfidence]
        CM --> CM2[UpdateCommittee]
        CN[grConstitution] --> CN1[NewConstitution]
    end

    style CM1 fill:#FFD700
    style CM2 fill:#FFD700
```

**Important**: `NoConfidence` and `UpdateCommittee` share the **same chain** (CommitteePurpose)!

## Validation Process

```mermaid
flowchart TD
    subgraph "1. Submission (GOV rule)"
        S1[Proposal submitted] --> S2{parent == root?}
        S2 -->|Yes| S3[Added to Proposals]
        S2 -->|No| S5{parent in Proposals?}
        S5 -->|Yes| S3
        S5 -->|No| S4[InvalidPrevGovActionId error]
        style S4 fill:#FF6B6B
        style S3 fill:#90EE90
    end

    subgraph "2. Ratification (RATIFY rule at epoch boundary)"
        R1[Check proposal] --> R2{parent == currentRoot?}
        R2 -->|Yes| R3{Threshold reached?}
        R3 -->|Yes| R4[ENACTED]
        R3 -->|No| R5[Remains in Proposals]
        R2 -->|No| R6[Skipped, may expire later]
        style R4 fill:#90EE90
        style R6 fill:#FFD700
    end
```

### 1. When Proposal is Submitted (GOV rule)

The proposal is added to the Proposals forest. The parent reference is validated:

```
Valid if:
  - parent == currentRoot (Nothing for first action, or last enacted)
  - OR parent exists in Proposals (points to unratified proposal)
```

If neither condition is met → `InvalidPrevGovActionId` error, transaction fails.

### 2. When Proposal is Ratified (RATIFY rule)

At epoch boundary, for ratification to succeed:

```
proposal.parent == ensPrevGovActionIds[purpose]
```

The proposal's parent must match the **currently enacted** action of the same purpose.

If it doesn't match → proposal is NOT ratified (skipped, may expire later).

## Example: ParameterChange Chain

**Initial state**: `prevGovActionIds.PParamUpdate = Nothing`

**Epoch 1**:
- Proposal A submitted: `ParameterChange(parent=Nothing, minFee=100)`
- Validation: parent == root (Nothing) ✓
- Added to Proposals

**Epoch boundary 1→2**:
- A ratified, A enacted
- `prevGovActionIds.PParamUpdate = Just(A)` (root updated)
- A removed from Proposals

**Epoch 2**:
- Proposal B submitted: `ParameterChange(parent=Just(A), minFee=200)`
- Validation: parent == root (Just A) ✓
- Proposal C submitted: `ParameterChange(parent=Nothing, minFee=300)` ❌
- Validation: parent(Nothing) != root(Just A) → **InvalidPrevGovActionId**
- **Transaction FAILS, C is never added to Proposals!**

**Epoch boundary 2→3**:
- B ratified, B enacted
- `prevGovActionIds.PParamUpdate = Just(B)`

## Tree Structure of Proposals

Proposals are stored as a **forest** (4 trees, one per purpose):

```mermaid
graph TD
    subgraph "PParamUpdate tree"
        R1[root/enacted]
        R1 --> A[Proposal A]
        R1 --> B[Proposal B]
        A --> C[Proposal C]
    end

    subgraph "Committee tree"
        R2[root/enacted]
        R2 --> X[Proposal X]
        R2 --> Y[Proposal Y]
        X --> Z[Proposal Z]
    end
```

When A is enacted:
- A becomes the new root
- A's deposit is returned ✓
- B (sibling) and all its descendants are **removed** (they're no longer valid)
- Deposits for all removed proposals are returned ✓

```mermaid
graph TD
    subgraph "Before: A ratified"
        R1[root] --> A1[A ✓ ratified]
        R1 --> B1[B]
        A1 --> C1[C]
        style A1 fill:#90EE90
    end

    subgraph "After: A enacted"
        A2[A = new root] --> C2[C]
        B2[B removed]
        style A2 fill:#90EE90
        style B2 fill:#FF6B6B,stroke-dasharray: 5 5
    end
```

## Competing Proposals (Siblings)

Multiple proposals can reference the same parent, creating "siblings":

```mermaid
graph TD
    X[enacted X] --> A[Proposal A]
    X --> B[Proposal B]
    X --> C[Proposal C]
    A --> D[Proposal D]

    style X fill:#87CEEB
```

**Scenario 1**: When voting completes and A is ratified:

```mermaid
graph TD
    subgraph "Result"
        A2[A = new root] --> D2[D valid]
        B2[B removed]
        C2[C removed]
        style A2 fill:#90EE90
        style B2 fill:#FF6B6B,stroke-dasharray: 5 5
        style C2 fill:#FF6B6B,stroke-dasharray: 5 5
    end
```

1. A is enacted
2. B and C are removed (siblings)
3. D remains valid (child of A)

**Scenario 2**: If B was ratified instead:

```mermaid
graph TD
    subgraph "Result"
        B3[B = new root]
        A3[A removed]
        C3[C removed]
        D3[D removed]
        style B3 fill:#90EE90
        style A3 fill:#FF6B6B,stroke-dasharray: 5 5
        style C3 fill:#FF6B6B,stroke-dasharray: 5 5
        style D3 fill:#FF6B6B,stroke-dasharray: 5 5
    end
```

1. B is enacted
2. A and C are removed
3. D is also removed (child of removed proposal A)

## Why TreasuryWithdrawals and InfoAction Don't Need Previous?

- **TreasuryWithdrawals**: Each withdrawal is independent, they don't conflict with each other. Multiple withdrawals can be enacted in the same epoch.

- **InfoAction**: Purely informational, doesn't change any protocol state. It's just a signal/poll.

These actions don't form chains and can be enacted in parallel.

## Errors Related to Previous Gov Action ID

| Error | When It Occurs |
|-------|----------------|
| `InvalidPrevGovActionId` | Proposal submitted with invalid/non-existent parent |
| `GovActionsDoNotExist` | Voting on a proposal that was removed (sibling was enacted) |

## Proposal Lifecycle

```mermaid
stateDiagram-v2
    [*] --> Submitted: Submit proposal
    Submitted --> InProposals: Valid parent

    Submitted --> Rejected: InvalidPrevGovActionId

    InProposals --> Ratified: Threshold reached + parent == root
    InProposals --> InProposals: Not enough votes
    InProposals --> Expired: gasExpiresAfter < currentEpoch
    InProposals --> Removed: Sibling was enacted

    Ratified --> Enacted: Apply changes

    Enacted --> [*]: Becomes new root + Deposit returned ✓
    Expired --> [*]: Deposit returned ✓
    Removed --> [*]: Deposit returned ✓
    Rejected --> [*]: Tx fails (deposit never taken)
```

**Note**: Deposit is **always returned** when a proposal leaves the Proposals set, regardless of the reason (enacted, expired, or removed due to sibling enactment).

## Summary

1. **6 action types require chaining**: ParameterChange, HardForkInitiation, NoConfidence, UpdateCommittee, NewConstitution
2. **2 action types are independent**: TreasuryWithdrawals, InfoAction
3. **4 independent chains** exist (Committee purpose is shared by NoConfidence and UpdateCommittee)
4. **Parent must match current root** at ratification time
5. **Siblings are removed** when one of them is enacted
6. **Deposits are ALWAYS returned** when proposal is removed (enacted, expired, or sibling enacted)
