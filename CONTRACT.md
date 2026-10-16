# Feather Contracts: Contract 1

**Status:** proposed for freezing, awaiting the Feather team's approval. When it
is approved, Contract 1 is frozen and feather-contracts becomes 1.0.0.

This document is the public contract of feather-contracts: what other resources
may call and rely on. Everything listed here keeps working, with the same
meaning, for as long as `Contract = 1`. The README explains the concepts and
setup; this file is the reference a consumer codes against.

## 1. Compatibility promise

Within Contract 1:

- **May be added** without a new contract version: optional request fields,
  result fields, type definition fields with a default, exports, client routes,
  events, payload fields, error codes and capability features. Consumers must
  ignore unknown result fields and treat an unknown error code as a failure.
- **Never changes**: the name, required fields, result shape or meaning of
  anything below; an error code's meaning; an event's name or the meaning of its
  fields; the lifecycle and its transitions; the policy action names;
  idempotency, revision and settlement guarantees.
- **Needs Contract 2**: removing or renaming anything, making an optional field
  required, changing a default, or changing what a result, status or event
  means.

Consumers check `GetCapabilities().value.contract == 1` and the features they
use before relying on them.

## 2. Lifecycle

| Export | Result |
|---|---|
| `GetCapabilities()` | `{ contract = 1, version, state, features, ... }` |
| `GetHealth()` | Health snapshot (diagnostic; fields may be added) |
| `AwaitReady(timeoutMs)` | Ready, or `timeout`; `timeoutMs` 0 to 60000 |

Features in Contract 1 (value `1` unless noted): `lifecycle`, `health`,
`migrations`, `results`, `obligationTypes`, `drafts`, `issuance`, `acceptance`,
`queries`, `settlement`, `settlementRecovery`, `reconciliation`,
`economyReconciliation`, `cancellation`, `disputes`, `dueProcessing`,
`defaults`, `idempotency`, `auditActions`, `outbox`. `clientRoutes`,
`organizationCreditors` and `organizationDebtors` are `1` when enabled in the
configuration. `amendments` is `0`.

Contracts publishes `contracts.ready.v1` when it is ready. Owners register
their obligation types again after every Contracts or owner restart.

## 3. Calling rules

- **Results.** Every export returns `{ ok = true, value = ... }` or
  `{ ok = false, code, message, details }`. Branch on `code`.
- **Caller.** The calling resource is taken from Cfx (`GetInvokingResource`).
  No field can claim another resource, account or character.
- **Context.** The optional second argument accepts only `actorSource` (a
  connected player; Contracts reads the account and character from Core) and
  `correlationId`. Actor UUIDs are never accepted.
- **Request IDs.** Mutations take `requestId`
  (`^[A-Za-z0-9][A-Za-z0-9._:-]*$`, at most 128; client routes at most 64,
  scoped to the character). Repeating it with the same payload returns the
  stored result with `replayed = true`; reusing it with a different payload
  returns `idempotency_conflict`. Scope: the calling resource and the
  operation.
- **Revisions.** Issuer operations accept an optional `expectedRevision`; a
  stale value returns `revision_conflict` with `details.revision`.
- **Amounts** are integers in the currency's minor units (2500 = 25.00).
- **Policy.** With authorization enabled, these go through Core policy with the
  issuing resource as `subject.resource`: issuing
  (`contracts.obligation.issue`), cancelling and cancelling through a dispute
  (`contracts.obligation.cancel`), marking defaults
  (`contracts.obligation.default`). A missing or failing provider denies.

## 4. Obligation types

`RegisterObligationType(definition)`, caller in `Config.Access.trustedIssuers`.
The first resource to register a type key owns it permanently
(`type_owner_conflict` for anyone else). Registering again with a changed
definition updates it; existing obligations keep the flags they were created
with.

| Field | Default | Meaning |
|---|---|---|
| `typeKey` | required | `^[a-z][a-z0-9_]*$`, at most 48, permanent |
| `label` | required | Display label, at most 64 |
| `issuerTypes` | required | `character`, `organization`, `system` |
| `creditorTypes` | required | `character`, `organization`, `system` (each enabled in `Config.Settlement.creditorTypes`) |
| `debtorTypes` | `{ 'character' }` | `character`, `organization` (each enabled in `Config.Settlement.debtorTypes`) |
| `requiresAcceptance` | `false` | The debtor accepts before it can be paid or fall due |
| `debtorMaySettle` | `true` | The debtor may pay |
| `issuerMaySettle` | `false` | The owner may collect from a character debtor without the debtor acting |
| `debtorMayDispute` | `false` | The debtor may dispute |
| `currencies` | any enabled | Allowed currency codes |
| `minimumAmount` / `maximumAmount` | 1 / `Config.Limits.maximumAmount` | Bounds in minor units |
| `maximumDueSeconds` | `Config.Limits.maximumDueSeconds` | Longest due period (at least 60) |
| `defaultAfterSeconds` | none | A due obligation defaults automatically this long after its due date |

At least one of `debtorMaySettle` and `issuerMaySettle` must be true.
`GetObligationType({ typeKey })` returns the definition with `owner`,
`revision`, `active` (registered since the owner started) and `transitions`.

## 5. Lifecycle of an obligation

```text
draft ──issue──▶ issued ──accept*──▶ accepted
                   │                    │
                   └──── due date ──────┴──▶ due ──(defaultAfterSeconds or MarkDefaulted)──▶ defaulted
issued/accepted/due/defaulted ──settle──▶ settled           (* only when requiresAcceptance)
issued/accepted/due ──dispute──▶ disputed ──resolve: upheld──▶ previous status
                                          └─resolve: cancelled─▶ cancelled
draft/issued/accepted/due/disputed ──cancel──▶ cancelled
```

`settled` and `cancelled` are terminal. A `defaulted` obligation can still be
paid. Amount, currency, debtor and creditor are fixed in the draft; the
creditor's Economy account is fixed at issuance. Nothing changes the terms of
an issued obligation (no amendments in Contract 1).

## 6. Exports

| Export | Caller | Fields | Notes |
|---|---|---|---|
| `CreateDraft(request, context?)` | type owner | `requestId`, `obligationType`, `issuer`, `debtor`, `creditor`, `amount`, `currency`, `dueInSeconds`, optional `referenceType` + `referenceId`, `memo` (at most 255), `reasonCode` | Parties are `{ type, id }`; a `system` party has no `id` |
| `IssueObligation(request, context?)` | type owner | `requestId`, `obligationId`, optional `reasonCode`, `expectedRevision` | Policy issue; fixes the creditor account; `dueAt` = now + `dueInSeconds` |
| `AcceptObligation(request, context)` | type owner with `actorSource` = the character debtor; for an organization debtor, the resource operating its treasury | `requestId`, `obligationId` | |
| `DisputeObligation(request, context)` | as for accept | `requestId`, `obligationId`, `reasonCode` | Type must allow disputes |
| `ResolveDispute(request, context?)` | type owner | `requestId`, `obligationId`, `outcome` (`upheld`, `cancelled`), `reasonCode`, optional `expectedRevision` | `cancelled` needs the cancel policy |
| `SettleObligation(request, context?)` | type owner; for an organization debtor, only the resource operating its treasury | `requestId`, `obligationId` | See section 7 |
| `CancelObligation(request, context?)` | type owner | `requestId`, `obligationId`, `reasonCode`, optional `expectedRevision` | Policy cancel |
| `MarkDefaulted(request, context?)` | type owner | `requestId`, `obligationId`, `reasonCode`, optional `expectedRevision` | Policy default; from `due` only |
| `GetObligation({ obligationId })` | type owner, issuer, trusted reader | | Others get `obligation_not_found` |
| `ListPartyObligations({ party, role?, statuses?, obligationType?, limit?, cursor? })` | trusted reader; a type owner filtering its own type; the resource operating the organization party's treasury | `role` `debtor` (default) or `creditor`; `limit` 1 to 50, default 20 | `{ items, nextCursor }`, newest first |
| `ReconcileSettlements()` | trusted reconciler | | `{ processed, journal, audit, healthy }` |

### Obligation snapshot

`obligationId`, `obligationType`, `issuerResource`, `issuer`, `debtor`,
`creditor` (parties `{ type, id }`), `creditorAccountId`, `amount`, `currency`,
`status`, `requiresAcceptance`, `debtorMaySettle`, `issuerMaySettle`,
`debtorMayDispute`, `dueInSeconds`, `defaultAfterSeconds`, `dueAt`,
`referenceType`, `referenceId`, `memo`, `settlementState` (`none`, `pending`),
`settlementTransactionId`, `revision`, `createdAt`, `issuedAt`, `acceptedAt`,
`settledAt`, `cancelledAt`, `disputedAt`, `defaultedAt`, `transitions`.
Mutations add `replayed`. Client routes return the same snapshot without
`creditorAccountId`, `issuerResource` and `transitions`.

## 7. Settlement guarantees

- Money moves only in Feather Economy, from the debtor's account (character
  wallet or organization treasury) to the creditor account fixed at issuance,
  with reason `contracts.settlement` and reference
  `contract_obligation` / `obligationId`.
- An obligation has at most one pending or committed settlement. It is never
  paid twice, and a settlement is never marked failed while its outcome is
  unknown.
- A definitive Economy refusal returns its code (`insufficient_funds`,
  `account_not_found`, `account_closed`, `currency_mismatch`,
  `authorization_denied`, `invalid_input`, `idempotency_conflict`,
  `transaction_conflict`); the obligation stays payable and a new `requestId`
  may try again.
- An unknown outcome returns `settlement_pending`. Contracts replays the same
  Economy request until it knows the result; repeating the caller's
  `requestId` resumes it.
- `initiatedBy` is `debtor` (the character debtor as actor, a client route, or
  an organization's treasury operator) or `issuer` (the owner collecting with
  `issuerMaySettle` from a character debtor). The issuer can never spend an
  organization's treasury.
- Reconciliation checks every committed settlement against Economy's journal
  when Economy offers `GetTransaction`.

## 8. Error codes

| Code | Meaning |
|---|---|
| `invalid_input` | A field is missing, unknown or malformed |
| `not_ready`, `timeout` | Contracts is not ready; `AwaitReady` timed out |
| `dependency_unavailable` | Core or Economy is unavailable |
| `authorization_denied` | Not trusted, not the owner or debtor, or policy denied |
| `idempotency_conflict` | Request ID reused with a different payload |
| `revision_conflict` | Stale `expectedRevision`; `details.revision` |
| `invalid_state` | The transition is not allowed from the current status; `details.status` |
| `settlement_pending` | A settlement is in progress or its outcome is unknown |
| `settlement_not_allowed` | The caller may not pay this obligation |
| `type_not_found`, `type_inactive`, `type_owner_conflict` | Obligation type missing, not registered since its owner started, or owned by another resource |
| `limit_exceeded` | The obligation type limit is reached |
| `party_type_not_allowed` | The party type is not allowed by the obligation type |
| `creditor_type_unsupported`, `debtor_type_unsupported` | The installation cannot settle that party type |
| `invalid_amount`, `currency_not_allowed`, `currency_not_found`, `currency_disabled` | Amount or currency rejected |
| `creditor_account_unavailable`, `debtor_account_unavailable` | The party has no open account in that currency |
| `obligation_not_found` | Missing, or not visible to the caller |
| `invalid_cursor` | The cursor is not from this listing |
| `character_required` | Client route without an active character |
| Economy refusal codes | See section 7 |
| `transaction_failed`, `transaction_conflict`, `invalid_persistence`, `internal_error` | Retry with the same request ID, or report it |

## 9. Events

Published through Core after commit, from an outbox; delivery is at least once,
deduplicate by `eventId`.

```text
contracts.ready.v1
contracts.obligation.issued.v1        contracts.obligation.accepted.v1
contracts.obligation.due.v1           contracts.obligation.settled.v1
contracts.obligation.cancelled.v1     contracts.obligation.disputed.v1
contracts.obligation.dispute_resolved.v1
contracts.obligation.defaulted.v1     contracts.settlement.failed.v1
```

Obligation payloads carry `eventId`, `obligationId`, `obligationType`,
`issuerResource`, `status`, `previousStatus`, `revision`, `debtor`, `creditor`,
`amount`, `currency`, `dueAt`, `sourceResource`, `reasonCode` and
`correlationId`. `settled` adds `settlementId`, `economyTransactionId` and
`initiatedBy`; `settlement.failed` adds `settlementId`, `code` and
`initiatedBy`; `dispute_resolved` adds `outcome`.

## 10. Client routes

Read and debtor actions for the calling player's own character; identity comes
from the Core session. Rate and size limited. Disabled together with
`Config.ClientRoutes.enabled = false`.

| Route | Payload |
|---|---|
| `contracts.obligations.list.v1` | `{ role?, statuses?, obligationType?, limit?, cursor? }` (own obligations, drafts hidden) |
| `contracts.obligation.get.v1` | `{ obligationId }` (when the caller is the debtor or the character creditor) |
| `contracts.obligation.accept.v1` | `{ requestId, obligationId }` |
| `contracts.obligation.settle.v1` | `{ requestId, obligationId }` (pays from the caller's own wallet) |
| `contracts.obligation.dispute.v1` | `{ requestId, obligationId, reasonCode }` |

No client route acts for an organization.

## 11. Configuration

A server may change: the access groups (`trustedIssuers`, `trustedReaders`,
`trustedReconcilers`), `Config.Settlement.creditorTypes` and `debtorTypes`
(enabling organizations needs Economy's `organizationSettlement`), retry and
scheduler timing, outbox timing, client routes on or off, and the limits.

A server must not: change `Config.Settlement.reasonCode` or `referenceType`
once settlements exist (reconciliation matches them), or disable a party type
that existing obligations use.

## 12. Not in Contract 1

Amendments (changing the terms of an issued obligation), partial payments,
instalments and automatic refunds. They would arrive as new operations and
capability features.
