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

This CIP adds an optional transaction-body field containing the expected latest
enacted governance action ID for each of four governance categories. The ledger
checks these IDs against its existing state during phase 1 when `is_valid` is
`true`. Plutus scripts receive the supplied values through a new script-context
field.

This gives contracts direct evidence of enactment without adding governance
history or new persistent ledger state. Reads can be repeated. Parents of enacted
actions are outside the scope of this proposal.

## Motivation: Why is this CIP necessary?

A contract may need evidence that a particular governance action has been enacted.
Knowing that an action was proposed, received votes, or changed a parameter does
not directly prove that the specific action ID was enacted.

The ledger already keeps the latest enacted action ID for each governance category
with a parent chain. Plutus scripts cannot currently read these IDs. Exposing them
would let a contract check enactment in the same transaction that performs its
operation, without a separate transaction to register the proof.

The optional current treasury value provides a useful model: a transaction supplies
an expected value, the ledger checks it, and the script context carries the
supplied value. This CIP applies that model to the existing enacted action IDs.

## Specification

### Scope

The new field covers the four governance purposes that have a previous action ID
under [CIP-1694](../CIP-1694/README.md):

| Key | Category | Included governance actions |
| --- | --- | --- |
| `0` | Protocol parameters | Parameter changes |
| `1` | Hard fork | Hard fork initiations |
| `2` | Committee | Committee updates and motions of no confidence |
| `3` | Constitution | New constitutions |

These keys identify categories in this field. They are separate from the numeric
tags used to encode governance action constructors.

Treasury withdrawals can enact, but have no parent chain or stored latest enacted
action ID. They are omitted to keep the added complexity small: supporting them
would require new ledger state and update rules. Information actions do not enact.

An action ID consists of the transaction ID that proposed it and its proposal
index within that transaction. This proposal exposes IDs only; it does not expose
action contents, parents, voting results, or ratification status.

### Source of truth

For each category, use the latest enacted action ID in the committed governance
state, or absence if that category has no enacted action ID.

In the reference ledger implementation, these are the proposal roots returned by
[`govStatePrevGovActionIds`][committed-roots]. Although their names use "previous",
they identify the latest enacted actions, which serve as the predecessors for
future proposals. This CIP adds no new persistent ledger state.

The comparison uses the ledger state immediately before applying the transaction,
after any epoch transition due for its block. It must not use pending ratification
results, the DRep pulser's intermediate state, or an indexer's view of the chain.

If several actions in one category enact at the same epoch boundary, only the last
one in the ledger's enactment order is available. Existing
[`proposalsApplyEnactment`][apply-enactment] processing already updates these roots
in that order.

### Transaction-body field

Add the following optional entry to the transaction-body map. Key `30` is an
illustrative placeholder and must be assigned before implementation.

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
```

The existing `gov_action_id` type is reused:

```cddl
gov_action_id = [
  transaction_id : transaction_id,
  gov_action_index : uint .size 2
]

transaction_id = bytes .size 32
```

The complete fragment is available in [enactment.cddl](./enactment.cddl).
`enactment_fields` is a group to insert into the existing transaction-body map,
not a replacement for that map.

The following meanings apply:

| Supplied value | Meaning |
| --- | --- |
| Field omitted | Make no enactment assertions. |
| Empty map | Make no enactment assertions. |
| Category omitted | Make no assertion about that category. |
| Category mapped to `nil` | Assert that the category has no enacted action ID. |
| Category mapped to an action ID | Assert that this is the latest enacted action in that category. |

For example, `{0: [h, 2], 3: nil}` asserts that proposal index `2` in transaction
`h` is the latest enacted parameter change, and that no constitution action has
been enacted. It makes no claim about hard forks or committee changes. Here `h`
stands for a 32-byte transaction ID.

Initial protocol parameters, a committee, or a constitution can exist without a
corresponding enacted governance action. A `nil` assertion concerns the action ID,
not the existence of those initial values.

Duplicate category keys, unknown categories, and malformed action IDs must be
rejected. These structural checks apply regardless of `is_valid`.

### Phase-1 validation

When `is_valid = true`, every supplied category must match the committed ledger
value, including absence:

```text
assertions = transaction.body.latest_enacted_actions, defaulting to {}

if transaction.is_valid:
    for each (category, supplied) in assertions:
        expected = committed_latest_enacted_action(category)
        require supplied == expected
```

A mismatch is a phase-1 failure. It must identify the category, supplied value,
and expected value. A mismatch does not cause collateral collection.

The check also applies to transactions that contain no Plutus scripts. An empty
or omitted field adds no assertions to check.

When `is_valid = false`, skip this equality check. This follows the existing
[`validateTreasuryValue` check][treasury-check], which runs only in the successful
transaction branch of the ledger rule. Existing script-failure and collateral
rules still apply; setting the flag to `false` alone does not make a transaction
acceptable.

No new redeemer type or script purpose is required.

### Plutus script context

Add a field to `TxInfo` with the following logical type, using the existing
`GovernanceActionId` type:

```haskell
txInfoLatestEnactedActions :: Map Integer (Maybe GovernanceActionId)
```

Its keys use the category numbers above. Populate it from the transaction body:

- An omitted field becomes an empty map.
- An omitted category has no map entry.
- A category containing `nil` becomes an entry containing `Nothing`.
- A category containing an action ID becomes an entry containing `Just` that ID.

Encode the field as a Plutus Data map, with integer keys in ascending numeric
order. Reuse the target language version's existing `Maybe` and
`GovernanceActionId` encodings. The field's position in `TxInfo` and the language
version introducing it must be assigned before implementation.

The context builder must copy the supplied data. It must not fetch live governance
state, fill in omitted entries, or replace a stale ID. This is the same separation
used for the [current treasury value in `TxInfo`][treasury-context]: the ledger
checks the assertion, while context construction uses the transaction body.

Context construction must also be independent of `is_valid`. A transaction marked
`false` may therefore carry unchecked enactment assertions during script
evaluation. The enactment guarantee applies to transactions accepted with
`is_valid = true`, whose ordinary effects are applied. Transactions accepted with
`is_valid = false` follow the existing collateral path and cannot use these
unchecked assertions to spend ordinary inputs or apply ordinary outputs.

A contract checks that its expected category has a `Just` entry equal to the
action ID it requires. Missing entries supply no evidence.

### Repeated reads and availability

Reading an action ID does not consume it or change governance state. Any number
of transactions may use the same ID while it remains the latest in its category.
Contracts that need single use can enforce it through their own state.

Once a newer action in that category enacts, the old ID no longer passes this
check. There is no time-based expiry, parent lookup, or historical lookup. A
contract that needs to remember earlier enactment can record a successful check
in its own state.

### Versioning and activation

This is a change to the ledger-script interface under
[CIP-0035](../CIP-0035/README.md). Activation requires a hard fork and a new Plutus
ledger language version carrying the revised script context. The transaction-body
key, activating protocol version, and target ledger language version remain
unassigned in this draft.

After activation, changes to the specified encoding or validation rules require a
new CIP that supersedes this one. The replacement CIP must define its protocol
activation and any further ledger language change. This versions the protocol
specification; edits to this document remain tracked by Git history.

At activation, assertions use the existing committed roots, including actions
enacted before the upgrade. No history reconstruction or new governance-state
initialization is needed. The new body field is accepted only from the activating
protocol version onward.

This draft specifies the field for ordinary top-level transactions. Extending it
to nested transaction bodies would require an explicit definition of their
validation and context mapping.

## Rationale: How does this CIP achieve its goals?

### Small ledger change

The ledger already stores and updates the four values being asserted. The main
changes are transaction-body serialization, a phase-1 comparison, and a Plutus
context field. At most four IDs are compared per transaction. Existing transaction
size and script execution limits apply to the additional data.

Exposing all action ancestors would require retaining history. Even retaining
just one parent would add state and require a rule for actions enacted before
activation, whose parents are no longer stored. Leaving parents out allows this
proposal to use the current ledger state directly.

### Deterministic script inputs

The supplied values are fixed in the transaction body. An enactment between
transaction construction and inclusion can make the assertion stale, but it
cannot silently change what the script sees. For `is_valid = true`, a stale
assertion causes phase-1 rejection and the transaction must be rebuilt.

Two nodes validating the same transaction against the same ledger state obtain
the same result. Epoch transitions, rollback, and mempool revalidation use the
ordinary ledger rules. A rollback restores the roots with the rest of the state;
there is no separate enactment cache to keep in sync.

This preserves fixed script inputs while making overall transaction validity
depend on an explicit state assertion, as the current treasury value already
does.

### Meaning of the proof

For a transaction accepted with `is_valid = true`, equality with a committed root
establishes that the supplied ID was enacted and is the latest in that category.
Proposal submission, votes, ratification, and deposit returns are insufficient
substitutes for that equality. The field makes no claim that all effects of an
action remain in force indefinitely.

Repeated use is intentional. No claim registry or global consumption rule is
needed to provide evidence of enactment.

### Compatibility

Existing nodes need an upgrade to accept the new transaction-body field. Scripts
that read it must target the Plutus ledger language version introducing the new
context. Adding the field to a deployed context layout would change the arguments
received by existing scripts, so the layout must be versioned.

The policy for transactions that combine this field with older script versions
remains an open design decision below.

### Open Questions

- Allocate the transaction-body key and the protocol version that activates it.
- Select the Plutus context version and field position. Specify how the chosen
  version handles transactions that also use older script-context versions;
  this draft does not assume that deployed context layouts can be changed.

These decisions are required before implementation. Community review and an
implementation commitment are also pending.

## Path to Active

### Acceptance Criteria

- [ ] Assign the body key, activation version, and Plutus context encoding.
- [ ] Implement body codecs and structural validation.
- [ ] Implement equality checks against committed roots for `is_valid = true`.
- [ ] Expose the supplied values through the chosen Plutus context version.
- [ ] Update the `plutus` repository's interface specification and implementation.
- [ ] Support transaction construction and retrieval of the committed roots in
      node queries and transaction-building tools.
- [ ] Publish support in external transaction builders and script libraries.
- [ ] Publish serialization and context translation test vectors.
- [ ] Verify the cases below in ledger integration tests.
- [ ] Implementation present within block producing nodes used by 80%+ of stake.
- [ ] Activate the protocol change on Cardano mainnet.

Required test cases:

| Case | Expected result from this feature |
| --- | --- |
| Field omitted or empty | No assertion; empty context map. |
| Matching latest ID, `is_valid = true` | Check passes; supplied ID appears in context. |
| Matching `nil`, `is_valid = true` | Check passes; category appears as `Nothing`. |
| Omitted category | No check or context entry for that category. |
| Wrong ID, category, or absence, `is_valid = true` | Phase-1 failure; no collateral collection. |
| One mismatch among several categories | Phase-1 failure for the transaction. |
| Stale ID after a new enactment | Phase-1 failure when marked `true`. |
| Proposed or ratified action that is not the committed root | Phase-1 failure when marked `true`. |
| Several same-category enactments at one boundary | Only the final root matches. |
| Committee update followed by no confidence, or the reverse | Both use category `2`; only its latest ID matches. |
| Repeated reads of a matching ID | Each check passes while the ID remains current. |
| Wrong ID with `is_valid = false` and a failing script | Equality check skipped; existing collateral rules apply. |
| `is_valid = false` when all scripts succeed | Existing validation-tag mismatch rule rejects it. |
| Malformed ID, duplicate key, or unknown category with either flag | Structural validation rejects it. |
| Different CBOR map entry orders | Same ascending key order in the Plutus context. |
| Rollback, node restart, or a different pulser progress point | Same result for the same committed state and transaction. |
| Upgrade with existing enacted roots | Existing IDs are immediately assertable. |

### Implementation Plan

1. Obtain Plutus and Ledger review and resolve the version and encoding
   assignments above.
2. Add the transaction-body field and its codecs to the target ledger era.
3. Add the equality check alongside the current treasury value check, using the
   committed roots in the transaction's input ledger state.
4. Update the Plutus interface specification and implementation with the new field
   and deterministic translation from the transaction body.
5. Update node queries where needed, transaction builders, and script libraries.
6. Add the test vectors and integration cases and measure size and evaluation costs.
7. Release the node and Plutus changes, then activate through a hard fork enabling
   the selected protocol version and new Plutus ledger language version.

## References

- [CIP-0001: CIP Process](../CIP-0001/README.md).
- [CIP-0035: Plutus Core Evolution](../CIP-0035/README.md).
- [CIP-0084: Cardano Ledger Evolution](../CIP-0084/README.md), including its guidance
  to use the Plutus category for changes to the ledger-script interface.
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
