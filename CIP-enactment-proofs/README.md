---
CIP: "?"
Title: Latest Enacted Governance Actions in the Script Context
Category: Plutus
Status: Proposed
Authors:
    - keyanm <keyanmaskoot@gmail.com> # Replace the Git username with the author's name.
Implementors: []
Discussions: [] # Add the original PR link when submitted.
Created: 2026-09-28
License: CC-BY-4.0
---

## Abstract

This CIP exposes the latest enacted governance action IDs through an optional
transaction-body field and a new Plutus `TxInfo` field. Phase-1 checks against
existing governance state provide contracts with evidence of enactment without
adding persistent ledger state.

## Motivation: Why is this CIP necessary?

The ledger already keeps the latest enacted action IDs, but Plutus scripts cannot
read them. Exposing them lets a contract verify enactment and perform its operation
in one transaction.

The proposal follows the optional current treasury value: the transaction supplies
an expected value, the ledger checks it, and the script context carries that value.

## Specification

### Scope

The new field covers the four governance purposes that have a previous action ID
under [CIP-1694](../CIP-1694/README.md):

| Key | Governance category |
| --- | --- |
| `0` | Protocol parameter changes |
| `1` | Hard fork initiations |
| `2` | Committee updates and motions of no confidence |
| `3` | New constitutions |

These are new category keys, separate from governance action constructor tags.

Treasury withdrawals can enact, but have no parent chain or stored latest enacted
action ID. They are omitted to keep the added complexity small: supporting them
would require new ledger state and update rules. Information actions do not enact.

### Ledger state

The expected values are the committed proposal roots returned by
[`govStatePrevGovActionIds`][committed-roots]: the latest enacted ID in each
category, or absence if none has enacted.

Use the state immediately before applying the transaction, after any epoch
transition due for its block. Pending ratification results must not be used.

When several actions in one category enact at the same boundary, only the final
root is available, as determined by
[`proposalsApplyEnactment`][apply-enactment].

### Transaction-body field

Insert `enactment_fields` into the transaction-body map. Key `30` is a placeholder;
`gov_action_id` reuses the existing ledger type. The fragment is also available in
[enactment.cddl](./enactment.cddl).

```cddl
enactment_fields = (
  ? 30 : latest_enacted_actions
)

latest_enacted_actions = {
  ? 0 : gov_action_id / nil, ; protocol parameters
  ? 1 : gov_action_id / nil, ; hard fork
  ? 2 : gov_action_id / nil, ; committee, including no confidence
  ? 3 : gov_action_id / nil  ; constitution
}

gov_action_id = [
  transaction_id : transaction_id,
  gov_action_index : uint .size 2
]

transaction_id = bytes .size 32
```

| Supplied value | Meaning |
| --- | --- |
| Field omitted or empty map | Make no enactment assertions. |
| Category omitted | Make no assertion about that category. |
| Category mapped to `nil` | Assert that the category has no enacted action ID. |
| Category mapped to an action ID | Assert that this is the latest enacted action in that category. |

Duplicate category keys, unknown categories, and malformed action IDs are always
rejected.

### Phase-1 validation

When `is_valid = true`, compare every supplied category against its committed
ledger value, including absence. A mismatch is a phase-1 failure reporting the
category, supplied value, and expected value. The check also applies to
transactions without Plutus scripts.

When `is_valid = false`, skip this equality check. This follows the existing
[`validateTreasuryValue` check][treasury-check]. In that case, inclusion in a block
does not authenticate the supplied IDs.

### Plutus script context

Add the following field to `TxInfo`:

```haskell
data EnactmentAssertion
  = Omitted
  | NoneEnacted
  | LatestEnacted GovernanceActionId

data LatestEnactedActions = LatestEnactedActions
  { protocolParameters :: EnactmentAssertion
  , hardFork           :: EnactmentAssertion
  , committee          :: EnactmentAssertion
  , constitution       :: EnactmentAssertion
  }

txInfoLatestEnactedActions :: LatestEnactedActions
```

Transaction-body keys `0`–`3` populate the record fields in declaration order.
Omitted categories become `Omitted`, `nil` becomes `NoneEnacted`, and an ID becomes
`LatestEnacted id`. An omitted or empty body field produces four `Omitted` values.

Encode the record as Plutus Data constructor `0`, with fields in declaration
order. For `EnactmentAssertion`, use constructor indices `0`, `1`, and `2` for
`Omitted`, `NoneEnacted`, and `LatestEnacted`, respectively. Only `LatestEnacted`
has a field, using the existing `GovernanceActionId` encoding.

As with the [current treasury value][treasury-context], construct this field solely
from the transaction body.

### Repeated reads and availability

An ID can be read repeatedly until a newer action in its category enacts. Parents
and historical IDs are unavailable. Contracts can enforce single use or retain
evidence of an earlier successful check through their own state.

### Versioning and activation

This ledger-script interface change requires a hard fork and a new Plutus ledger
language version under [CIP-0035](../CIP-0035/README.md).

After activation, changes to the specified encoding or validation rules require a
new CIP that supersedes this one. The replacement CIP must define its protocol
activation and any further ledger language change.

The field is accepted from the activating protocol version onward and immediately
uses the existing roots, including IDs enacted before the upgrade.

This draft covers top-level transactions only.

## Rationale: How does this CIP achieve its goals?

Using existing roots limits validation to four comparisons and avoids new ledger
state. Retaining parents or history would require additional storage and a rule
for recovering information about actions enacted before activation.

Supplying IDs in the transaction keeps script inputs fixed. A later enactment can
make an assertion stale, but cannot change what a script sees.

### Compatibility

The new context requires scripts to target the new ledger language version.
Changing a deployed context layout would alter the arguments existing scripts
receive. The policy for combining this field with older scripts remains open.

### Open Questions

- Assign the transaction-body key and activation protocol version.
- Select the Plutus ledger language version and `TxInfo` field position, and
  define handling of transactions that also use older script versions.

These choices, community review, and an implementation commitment are pending.

## Path to Active

### Acceptance Criteria

- [ ] Assign the body key, activation version, and Plutus context encoding.
- [ ] Implement the body codecs, phase-1 checks, and context translation.
- [ ] Update the `plutus` repository's interface specification and implementation.
- [ ] Provide node queries for the committed roots and support in external
      transaction builders and script libraries.
- [ ] Publish encoding vectors and pass ledger integration tests covering the
      cases below.
- [ ] Implementation present within block producing nodes used by 80%+ of stake.
- [ ] Activate the protocol change on Cardano mainnet.

Required test cases:

- Omitted, empty, and `nil` values; matching, wrong, and stale IDs; a mismatch
  among multiple categories; transactions without Plutus scripts.
- Both validation branches and unconditional structural checks; deterministic
  context translation and record field ordering.
- Repeated reads, committee/no-confidence changes, and multiple enactments in one
  category at the same boundary, where only the final root matches.
- Activation with existing roots, rollback, node restart, and independence from
  pending ratification or pulser progress.

### Implementation Plan

1. Obtain Plutus and Ledger review and resolve the version and encoding
   assignments above.
2. Add the body field and codecs, and place the equality check alongside the
   current treasury value check.
3. Update the Plutus specification and implementation, node queries where needed,
   transaction builders, and script libraries.
4. Add test vectors and integration tests; measure size and evaluation costs.
5. Release the node and Plutus changes, then activate through a hard fork enabling
   the selected protocol version and new Plutus ledger language version.

## References

- [CIP-0001: CIP Process](../CIP-0001/README.md).
- [CIP-0035: Plutus Core Evolution](../CIP-0035/README.md).
- [CIP-0084: Cardano Ledger Evolution](../CIP-0084/README.md).
- [CIP-1694: A First Step Towards On-Chain Decentralized Governance](../CIP-1694/README.md).
- Reference ledger source below is pinned to commit
  [`d2e02427567ae650677ebf2e9c17f2a5e69c0dd4`][ledger-baseline].

[ledger-baseline]: https://github.com/IntersectMBO/cardano-ledger/tree/d2e02427567ae650677ebf2e9c17f2a5e69c0dd4
[committed-roots]: https://github.com/IntersectMBO/cardano-ledger/blob/d2e02427567ae650677ebf2e9c17f2a5e69c0dd4/eras/conway/impl/src/Cardano/Ledger/Conway/Governance.hs#L285-L286
[apply-enactment]: https://github.com/IntersectMBO/cardano-ledger/blob/d2e02427567ae650677ebf2e9c17f2a5e69c0dd4/eras/conway/impl/src/Cardano/Ledger/Conway/Governance/Proposals.hs#L492-L560
[treasury-check]: https://github.com/IntersectMBO/cardano-ledger/blob/d2e02427567ae650677ebf2e9c17f2a5e69c0dd4/eras/conway/impl/src/Cardano/Ledger/Conway/Rules/Ledger.hs#L375-L379
[treasury-context]: https://github.com/IntersectMBO/cardano-ledger/blob/d2e02427567ae650677ebf2e9c17f2a5e69c0dd4/eras/conway/impl/src/Cardano/Ledger/Conway/TxInfo.hs#L542-L543

## Copyright

This CIP is licensed under [CC-BY-4.0](https://creativecommons.org/licenses/by/4.0/legalcode).
