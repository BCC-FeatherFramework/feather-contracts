# Feather Contracts

Authoritative payable obligations for the Feather Framework: invoices, bills,
fines, rent, taxes, court awards and similar debts, with their lifecycle and
settlement through Feather Economy.

Contracts owns obligation identity, parties, the frozen amount, currency and
creditor account, lifecycle status, due dates, settlement references,
disputes, cancellations, and the audit trail. It does **not** own prices, tax
policy, court gameplay, bank UI, debt-collection gameplay, or balances. Money
moves only in Feather Economy. Contracts is not a quest or scripting engine.

Status: implements phases C1–C3 of the
[Contracts master plan](https://github.com/FeatherFramework/feather-framework-docs/blob/main/feather-contracts/Feather_Contracts_Master_Plan.md),
plus the type registry C4 needs to prove a second obligation type. It has not
yet passed live acceptance.

| Gate | State |
|---|---|
| `ContractsFoundationSmokeTest` (17 checks) | Passed 17/17 on the server 2026-10-13 |
| `ContractsAuditSmokeTest` (12 checks) | Passed 12/12 on the server 2026-10-13 |
| `ContractsReleaseSmokeTest` (6 checks) | Passed 6/6 on the server 2026-10-13 |
| `ContractsReconcile` | Healthy on the server 2026-10-13 (no settlements yet) |
| `ContractsLiveTest` (DevMode, two players) | Not yet run |
| Second obligation type (C4) through a real issuer | Not yet proven |

## Dependencies and start order

```text
feather-mysql
feather-core      (Contract 1)
feather-economy   (Contract 1: currencyCatalog, accounts, transfers, idempotency)
```

Start Contracts after Economy and before the resources that issue obligations:

```cfg
ensure feather-core
ensure feather-economy
# ... platform and governance resources ...
ensure feather-contracts
ensure feather-admin
# obligation issuers (for example bcc-society, bcc-banks) after this
```

Contracts waits for Core and Economy readiness and capabilities, then fails
closed if either is missing.

## Required configuration in other resources

Contracts reads accounts and posts transfers through Economy's public exports,
so Economy must trust it. Add `feather-contracts` to these lists in
`feather-economy/config.lua`:

```lua
trustedReaders      = { ['feather-contracts'] = true, ... }
trustedTransactors  = { ['feather-contracts'] = true, ... }
```

Contracts is never a provisioner, reverser, or supplier: it cannot create
wallets or currency.

With `Config.Authorization.enabled = true` (the default), issuing, cancelling
and marking defaults are checked through Core policy with the issuing
resource as `subject.resource`. With feather-admin as policy provider, grant
each issuer explicitly in `feather-admin/configs/service_policy.lua`:

```lua
Config.servicePolicy['feather-contracts'] = {
    ['bcc-society'] = {
        ['contracts.obligation.issue'] = true,
        ['contracts.obligation.cancel'] = true,
        ['contracts.obligation.default'] = true
    }
}
```

Finally, list each issuing resource in `Config.Access.trustedIssuers` in this
resource's `config.lua`.

## Concepts

**Obligation type.** A consumer registers each type it issues. The first
resource to register a type key owns it permanently; only the owner may issue,
cancel, resolve disputes, or mark defaults for that type. Registration is
needed again after every Contracts or owner restart (listen for
`contracts.ready.v1`). Definitions are stored, so changing one never changes
obligations that already exist: each obligation snapshots its lifecycle flags.

| Field | Default | Meaning |
|---|---|---|
| `typeKey` | required | Stable key, `^[a-z][a-z0-9_]*$`, max 48 |
| `label` | required | Display label (not an identifier) |
| `issuerTypes` | required | Allowed issuer parties: `character`, `organization`, `system` |
| `creditorTypes` | required | Allowed creditor parties (must be enabled in `Config.Settlement.creditorTypes`) |
| `requiresAcceptance` | `false` | Debtor must accept before the obligation can be paid or fall due |
| `debtorMaySettle` | `true` | Debtor may pay it (client route or owner with `actorSource`) |
| `issuerMaySettle` | `false` | The owning resource may collect from the debtor's wallet without the debtor acting |
| `debtorMayDispute` | `false` | Debtor may dispute it |
| `currencies` | any enabled | Allowed Economy currency codes |
| `minimumAmount` / `maximumAmount` | 1 / config limit | Bounds in minor units |
| `maximumDueSeconds` | config limit | Longest allowed due period |
| `defaultAfterSeconds` | none | When set, a due obligation defaults automatically this long after its due date |

**Parties.** In v1 the debtor is always a character. Creditors can be a
character (paid into their wallet) or the system (paid into Economy's system
sink, for example fines and taxes). Organization creditors are modelled but
disabled: Feather Economy currently allows wallet-to-treasury transfers only
for shop purchases, and Contracts refuses to issue an obligation it could
never settle.

**Lifecycle.**

```text
draft ──issue──▶ issued ──accept*──▶ accepted
                   │                    │
                   └──── due date ──────┴──▶ due ──(defaultAfterSeconds or MarkDefaulted)──▶ defaulted
issued/accepted/due/defaulted ──settle──▶ settled           (* only when requiresAcceptance)
issued/accepted/due ──dispute──▶ disputed ──resolve: upheld──▶ previous status
                                          └─resolve: cancelled─▶ cancelled
draft/issued/accepted/due/disputed ──cancel──▶ cancelled
```

`settled` and `cancelled` are terminal. A defaulted obligation can still be
paid late. Every obligation snapshot includes its `transitions` table.

**Frozen terms.** Amount, currency, debtor, and creditor are set in the draft.
The creditor's Economy account is resolved and frozen at issuance. There is no
amendment workflow: cancel and issue a new obligation instead.

## Server contract

All exports are server-only, use the Contract 1 result envelope, and derive
the caller with `GetInvokingResource()`. Mutations take a stable `requestId`
(`^[A-Za-z0-9][A-Za-z0-9._:-]*$`, max 128). Repeating a request with the same
payload replays the stored result with `replayed = true`; reusing it with a
different payload returns `idempotency_conflict`.

The optional second argument `context` accepts only `actorSource` (a player
source; Contracts resolves the account and character from Core) and
`correlationId`. Actor UUIDs are never accepted from callers.

```lua
local contracts = exports['feather-contracts']
contracts:AwaitReady(30000)

contracts:RegisterObligationType({
    typeKey = 'society_invoice', label = 'Invoice',
    requiresAcceptance = true, debtorMayDispute = true,
    issuerTypes = { 'character' }, creditorTypes = { 'character' },
    currencies = { 'dollars' }, defaultAfterSeconds = 604800
})

local draft = contracts:CreateDraft({
    requestId = 'bill:' .. billId,
    obligationType = 'society_invoice',
    issuer = { type = 'character', id = issuerCharacterId },
    debtor = { type = 'character', id = customerCharacterId },
    creditor = { type = 'character', id = issuerCharacterId },
    amount = 2500, currency = 'dollars', dueInSeconds = 86400,
    referenceType = 'society_bill', referenceId = billId,
    memo = 'Blacksmith repairs'
}, { actorSource = src })

local issued = contracts:IssueObligation({
    requestId = 'bill-issue:' .. billId, obligationId = draft.value.obligationId
})
```

| Export | Caller | Notes |
|---|---|---|
| `GetCapabilities()`, `GetHealth()`, `AwaitReady(ms)` | any | Lifecycle and features |
| `RegisterObligationType(definition)` | trusted issuer | Binds the key to the caller on first registration |
| `GetObligationType({ typeKey })` | any | Definition, owner, `active`, `transitions` |
| `CreateDraft(request, context?)` | type owner | Validates parties, amount, currency, due period |
| `IssueObligation({ requestId, obligationId, reasonCode?, expectedRevision? }, context?)` | type owner | Policy `contracts.obligation.issue`; freezes creditor account; sets `due_at` |
| `AcceptObligation({ requestId, obligationId }, { actorSource })` | type owner | Actor must be the debtor |
| `DisputeObligation({ requestId, obligationId, reasonCode }, { actorSource })` | type owner | Actor must be the debtor; type must allow disputes |
| `ResolveDispute({ requestId, obligationId, outcome, reasonCode, expectedRevision? }, context?)` | type owner | `upheld` or `cancelled` (policy `contracts.obligation.cancel`) |
| `SettleObligation({ requestId, obligationId }, context?)` | type owner | Debtor-initiated with `actorSource` = debtor, otherwise requires `issuerMaySettle` |
| `CancelObligation({ requestId, obligationId, reasonCode, expectedRevision? }, context?)` | type owner | Policy `contracts.obligation.cancel` |
| `MarkDefaulted({ requestId, obligationId, reasonCode, expectedRevision? }, context?)` | type owner | Policy `contracts.obligation.default`; from `due` only |
| `GetObligation({ obligationId })` | owner, issuer, trusted reader | Others receive `obligation_not_found` |
| `ListPartyObligations({ party, role?, statuses?, obligationType?, limit?, cursor? })` | trusted reader, or owner filtering its own type | Cursor-paginated, newest first |
| `ReconcileSettlements()` | trusted reconciler | Replays pending settlements and returns the integrity audit |

Amounts are integers in the currency's minor units (2500 = 25.00 dollars).

## Client routes

Registered through Core, require a current character session, and bound by
payload limits and rate limits. Identity comes from the session only.
Disable them with `Config.ClientRoutes.enabled = false` if every interaction
goes through a gameplay resource.

| Route | Payload |
|---|---|
| `contracts.obligations.list.v1` | `{ role?, statuses?, obligationType?, limit?, cursor? }` (own obligations; drafts hidden) |
| `contracts.obligation.get.v1` | `{ obligationId }` (only when the caller is debtor or character creditor) |
| `contracts.obligation.accept.v1` | `{ requestId, obligationId }` |
| `contracts.obligation.settle.v1` | `{ requestId, obligationId }` (pays from the caller's own wallet) |
| `contracts.obligation.dispute.v1` | `{ requestId, obligationId, reasonCode }` |

Client `requestId` values are limited to 64 characters and scoped to the
character, so two players cannot collide.

## Settlement workflow

Separate resources cannot share a database transaction, so settlement is an
explicit, durable workflow.

| Step | System of record | Behaviour |
|---|---|---|
| 1. Begin | Contracts | One transaction reserves the request receipt, locks the obligation, inserts a `pending` settlement with Economy key `contracts:settle:<settlementId>`, and flags the obligation `settlement_state = pending` |
| 2. Transfer | Economy | `Transfer` from the debtor's wallet to the frozen creditor account, reason `contracts.settlement`, reference `contract_obligation`/`<obligationId>` |
| 3a. Committed | Contracts | One transaction marks the settlement `committed` with the Economy transaction UUID, the obligation `settled`, records the audit action, and queues `contracts.obligation.settled.v1` |
| 3b. Rejected | Contracts | Definitive Economy rejection (`insufficient_funds`, `account_closed`, ...): settlement `failed`, obligation flag cleared, `contracts.settlement.failed.v1` queued. Retry with a new `requestId` |
| 3c. Unknown | Contracts | Timeout, provider failure, or unconfirmed commit: settlement stays `pending` |

- **Retry owner:** Contracts. The scheduler replays pending settlements with
  the *same* Economy key, so Economy returns the original transaction instead
  of moving money twice. A caller repeating its `requestId` resumes the same
  settlement.
- **Idempotency scope:** Contracts receipts are per `(calling resource,
  operation, requestId)`; the Economy key is per settlement.
- **No double payment:** an obligation has at most one `pending` or
  `committed` settlement (enforced by a unique index), cancellation, dispute,
  and default are refused while a settlement is pending, and a settlement is
  never marked failed while its outcome is unknown.
- **Compensation:** none is automatic. Refunding a settled obligation is a
  separate Economy operation that Contracts does not perform.
- **Operator reconciliation:** `ContractsReconcile` in the server console.

Economy does not infer that a transfer satisfies an obligation; Contracts
stores that correlation.

## Due and default processing

A scheduler pass (`Config.Scheduler.intervalMs`) moves obligations whose
`due_at` has passed to `due`, and moves `due` obligations past
`due_at + defaultAfterSeconds` to `defaulted`. Conditions are evaluated by the
database clock against stored timestamps, so restarts neither skip nor repeat
transitions. Obligations with a pending settlement are skipped. Unaccepted
obligations of a type that requires acceptance never fall due; the issuer
cancels them.

## Events

Published through Core after commit via a transactional outbox:

```text
contracts.ready.v1
contracts.obligation.issued.v1        contracts.obligation.accepted.v1
contracts.obligation.due.v1           contracts.obligation.settled.v1
contracts.obligation.cancelled.v1     contracts.obligation.disputed.v1
contracts.obligation.dispute_resolved.v1
contracts.obligation.defaulted.v1     contracts.settlement.failed.v1
```

Payloads carry `eventId`, `obligationId`, `obligationType`, `issuerResource`,
`status`, `previousStatus`, `revision`, parties, `amount`, `currency`,
`dueAt`, `sourceResource`, `reasonCode`, and `correlationId`. Settled events
add `settlementId` and `economyTransactionId`. Delivery is at-least-once;
deduplicate by `eventId`.

## Persistence

Checksummed migrations in `server/migrations/`, ledger
`contract_schema_migrations`. Tables, written only by this resource:

| Table | Contents |
|---|---|
| `contract_obligation_types` | Type key, permanent owner, stored definition |
| `contract_obligations` | Obligation terms, lifecycle, frozen creditor account, settlement reference |
| `contract_actions` | Append-only audit: every transition with actor, reason, correlation, request, Economy transaction |
| `contract_receipts` | Idempotency receipts |
| `contract_settlements` | Settlement workflow records |
| `contract_outbox` | Pending and published events |

Never edit applied migrations; add a new numbered migration. Back up these
tables together with Economy's, because settlements reference Economy
transaction UUIDs.

## Validation

Server console:

```text
ContractsFoundationSmokeTest     read-only
ContractsAuditSmokeTest          read-only integrity audit
ContractsReleaseSmokeTest        read-only production checks
ContractsReconcile               replays pending settlements, prints integrity report
```

With `Config.DevMode = true` on an isolated development server only:

```text
ContractsLiveTest <debtor source> <creditor source> <fresh runId> [currency]
```

The live test registers two development types owned by feather-contracts,
then exercises draft replay and payload conflict, issuance, acceptance,
settlement replay, the double-payment guard (or the insufficient-funds path
when the debtor has no funds), dispute and upheld resolution, stale-revision
rejection, cancellation, and scheduler due/default processing. It moves at
most 0.01 from debtor to creditor and leaves one defaulted 0.01 fine as
evidence. Issuance requires a policy grant for feather-contracts itself while
policy is enabled. Disable DevMode and restart before release.

## Known limits

- Organization creditors and debtors need an Economy contract addition
  (wallet-to-treasury and treasury-debit transfers for `contracts.settlement`).
- No amendment workflow (`amendments = 0` in capabilities).
- The integrity audit checks Contracts' own records. Confirming each
  settlement against Economy's journal needs an Economy read export for
  transactions, which does not exist yet.
- Settlement replay after Economy is permanently unavailable keeps the
  settlement pending indefinitely; `settlement.stuck` is logged after
  `Config.Settlement.maximumAttempts`.
