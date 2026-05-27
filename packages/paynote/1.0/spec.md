# Blue PayNote Specification 1.0

> **Status.** Draft specification for the Blue PayNote package.
>
> **Positioning.** A PayNote is a standard Blue document that attaches to exactly one transaction and governs what should happen with that transaction. It is not a payment rail, processor, bank ledger, card network, wallet, escrow account, or settlement system. It is the shared, verifiable process document around the rail.
>
> **Scope.** This document explains the PayNote concept, participant model, lifecycle vocabulary, request/response vocabulary, triggered-transaction model, control locks, final-amount resolution, deferred recipient selection, and canonical PayNote package source nodes. It intentionally does **not** define compute steps, JavaScript, BEX programs, workflow implementation bodies, payment-rail APIs, authorization protocols, compliance rules, or fund movement itself.

## Conventions

The key words **MUST**, **MUST NOT**, **REQUIRED**, **SHOULD**, **SHOULD NOT**, **MAY**, and **OPTIONAL** are to be interpreted as normative requirement levels.

Sections marked **normative** define required behavior for PayNote package compatibility. Sections marked **informative** explain intent, examples, or implementation guidance.

Canonical source nodes in this document are intended to be identity-bearing Blue content. Their `description` fields deliberately define semantics. A registry release SHOULD publish the exact source node, preprocessed/canonical form, calculated BlueId, specification version, and fixture package identity for every canonical PayNote type.

Because Blue type nodes can fix values inherited by descendants, mutable lifecycle fields such as `status`, `amount.secured`, `amount.completed`, and `controls.completionLocked` are described as fields but are **not** assigned fixed default `value` fields in the canonical `PayNote` type. A PayNote instance may initialize those values, and a runtime profile may initialize them, but the type itself must not make mutable state impossible to change.

---

## 0. Overview

A PayNote is a document attached to a transaction. The transaction is where money moves, is authorized to move, is held, is released, is captured, is reversed, or is otherwise represented on a rail. The PayNote is the shared document that defines the conditions and evidence around that transaction.

```text
TRANSACTION = money moves, or is authorized/held on a rail
PAYNOTE     = rules, evidence, participants, conditions, and lifecycle around it
```

A PayNote answers questions that a raw payment transaction usually cannot answer by itself:

- who is the payer, payee, and guarantor;
- what amount and currency are expected;
- what details identify or attach the underlying transaction;
- what conditions must be satisfied before funds are secured, completed, cancelled, or reversed;
- whether the final amount has been resolved;
- whether completion, reversal, or transaction-detail updates are locked;
- which participant requested an action;
- which trusted party confirmed the actual rail outcome;
- whether new independent transactions should be triggered.

The central rule is simple:

```text
one PayNote -> exactly one governed transaction
```

A PayNote may trigger more transactions, but every triggered transaction is independent. A triggered transaction may have its own PayNote. There are no child PayNotes in the core model; there are only independent transactions connected by evidence and events.

---

## 1. Core Concepts

### 1.1 One PayNote attaches to one transaction

A PayNote governs exactly one transaction. That transaction may be a card authorization, card capture, bank-transfer initiation, account hold, internal ledger transfer, account-to-card payout, wallet transfer, payment mandate spend, escrow release, or another rail-specific transaction.

If a PayNote causes another payment action, that action is a new transaction. It may be linked to the original PayNote by event evidence, but it is not part of the same PayNote's transaction.

This model avoids ambiguity:

```text
PayNote A governs Transaction A.
PayNote A may request Transaction B.
Transaction B may have PayNote B.
PayNote B governs Transaction B, not Transaction A.
```

### 1.2 PayNote is optional

Not every transaction needs a PayNote. A small immediate card purchase, such as a coffee payment, may proceed without one. PayNotes are useful when a transaction needs conditions, delayed completion, multiple parties, evidence, dispute protection, agent constraints, split settlement, reversals, or triggered follow-on transactions.

A rail or guarantor MAY treat a transaction without a PayNote as a normal payment. A transaction with an accepted PayNote is governed according to the PayNote's rules and the guarantor's supported rail semantics.

### 1.3 Participants

Every PayNote has three conceptual participant roles.

**Payer** is the party providing funds or value.

**Payee** is the party intended to receive funds or value.

**Guarantor** is the trusted party that controls, executes, or authoritatively reports the underlying transaction. Examples include a bank, card issuer, payment processor, wallet provider, internal ledger operator, platform operator, or other payment guarantor.

The Guarantor is the source of truth for what actually happened on the underlying rail. Participants may request actions; the Guarantor confirms, rejects, declines, fails, or records the actual result.

### 1.4 Accepted before initiated

A PayNote can be accepted before the underlying transaction is fully initiated or fully described.

**Accepted** means the Guarantor agrees that this PayNote governs a particular transaction flow or candidate transaction.

**Initiated** means the underlying rail-specific transaction has actually started, and the Guarantor has recorded attachment details such as provider reference, rail type, initiated amount, authorization code, transaction id, hold id, or account reference.

This distinction matters because a PayNote may be accepted while final recipient details, final amount, or rail-specific details are still unresolved.

### 1.5 Deferred recipient and transaction-detail selection

A PayNote may be accepted or even secured before the final payee, merchant, account, recipient target, or provider-specific transaction details are known.

This is useful for agentic and pre-approval scenarios:

- a bank secures funds before the final merchant is selected;
- an agent shops within a pre-approved budget;
- a marketplace reserves value before a seller is confirmed;
- a travel agent assembles hotel, restaurant, and transport commitments;
- a recipient account is filled in later by a trusted participant.

The PayNote therefore includes participant requests and Guarantor responses for payee assignment and transaction-detail updates.

### 1.6 Rail-neutral lifecycle vocabulary

PayNote uses rail-neutral terms:

| PayNote term | Card example | Bank transfer example | Internal ledger example |
|---|---|---|---|
| secure funds | authorize / hold approved | hold / commit under provider control | reserve internally |
| complete payment | capture | settle / release | book transfer |
| cancel before completion | void / release auth | stop before settlement | release reservation |
| reverse after completion | refund / reversal | return / compensating transfer | compensating transfer |

Rail-specific packages MAY expose specialized terms, but canonical PayNote vocabulary SHOULD stay rail-neutral.

### 1.7 Final amount resolution

Many transactions begin with an expected amount and later need the final completion amount to be resolved.

Examples include tips, mileage, hotel incidentals, partial fulfillment, delivery adjustments, marketplace partial shipment, and usage-based billing.

PayNote uses **final amount resolution** rather than older settlement-amount language. The concept is the final amount this PayNote should complete for, not a provider's back-office settlement accounting.

### 1.8 Control locks

A PayNote may restrict actions that the rail would otherwise allow. The standard control locks are:

- completion lock / unlock;
- reversal lock / unlock;
- transaction-details-update lock / unlock.

Locks make PayNote more than an observer. A card authorization might exist, but the PayNote can require that capture be rejected until delivery is confirmed. A reversal may be locked during dispute handling. Transaction details may be locked after final approval.

### 1.9 Triggered independent transactions

A PayNote may request or trigger new transactions. Examples include voucher credits, split payouts, milestone releases, recurring subscription renewals, agent-budget sub-transactions, factoring advances, refunds, bonuses, insurance staged payouts, or marketplace partner settlements.

Each triggered transaction is independent and may have its own PayNote.

---

## 2. PayNote Lifecycle Model

### 2.1 Recommended lifecycle states

PayNote `status` is rail-neutral and intentionally simple. Recommended values are:

| Status | Meaning |
|---|---|
| `Pending` | PayNote exists but has not yet been accepted or rejected by the Guarantor. |
| `Accepted` | Guarantor accepts that this PayNote governs the transaction flow. |
| `Initiated` | The underlying transaction has been initiated and attachment details are recorded. |
| `Secured` | Funds/value are secured or controlled by the Guarantor. |
| `Completed` | Payment has been finalized according to this PayNote. |
| `Cancelled` | Transaction was cancelled before completion. |
| `Reversed` | Value was reversed after completion. |
| `Rejected` | Guarantor rejected the PayNote or its acceptance. |
| `Failed` | A technical or provider-level failure prevents normal progression. |

Provider-specific packages MAY add additional statuses, but portable PayNote logic SHOULD be able to reason about these canonical states.

### 2.2 Requests and responses

PayNote distinguishes participant requests from Guarantor responses.

A **Request** asks that something happen. It may be posted by the payer, payee, agent, platform, or another permitted participant.

A **Response** records the authoritative result of a request, usually from the Guarantor.

For example:

```text
Secure Funds Requested  -> participant asks Guarantor to secure funds
Funds Secured           -> Guarantor confirms secured amount
Funds Securing Declined -> Guarantor declines for business/risk/policy reason
Funds Securing Failed   -> Guarantor attempted but technical/provider failure occurred
```

### 2.3 Declined vs failed

PayNote separates **declined** from **failed**.

A declined response means the Guarantor considered the request and refused it as a valid business/risk/policy decision. A failed response means the Guarantor attempted or could not complete the action because of a technical, provider-level, integration, rail, or operational failure.

This distinction matters for audit and retries. A declined action may require a different request or human approval. A failed action may be retried after the external issue is resolved.

### 2.4 Source of truth

Participants may request actions. The Guarantor response is authoritative for rail facts. A PayNote SHOULD NOT treat a participant's request as proof that money moved, was secured, was completed, or was reversed.

---

## 3. Canonical Source Node Rules

The YAML nodes in this document are intended as PayNote package source nodes. The descriptions are intentionally semantic and identity-bearing.

Names such as `Text`, `Integer`, `Boolean`, `List`, `Dictionary`, `Common/Currency`, `Common/Timestamp`, `Conversation/Request`, `Conversation/Response`, `Conversation/Event`, `Conversation/Operation`, and `Conversation/Timeline Channel` are aliases used for readability. A registry release must resolve them through the declared Blue Language, Conversation, and package preprocessing environment.

The PayNote package SHOULD avoid embedding implementation code in canonical type nodes. Workflow implementations, BEX programs, JavaScript, provider API calls, and rail adapters belong in implementation packages or profile-specific contracts, not in the provider-neutral PayNote semantic registry.

---

## 4. PayNote Type Catalog

### 4.1 Core document types

#### PayNote

```yaml
name: PayNote
description: >
  PayNote package base type for a Blue document attached to exactly one
  transaction. The transaction is where value is authorized, held, moved,
  completed, cancelled, or reversed on a payment rail. The PayNote is the
  shared process document around that transaction: it records the participants,
  expected and final amounts, currency, transaction attachment details,
  control locks, lifecycle status, requests, Guarantor responses, and evidence
  explaining why the transaction should be secured, completed, cancelled,
  reversed, or used to trigger other independent transactions. A PayNote is
  not itself a bank account, payment rail, ledger entry, authorization token,
  settlement instruction, or legal identity system. A PayNote may trigger new
  independent transactions, each of which may have its own PayNote, but this
  PayNote governs only the single transaction to which it is attached.
kind:
  type: Text
  description: >
    Fixed package discriminator for PayNote documents. This field helps search,
    indexing, and profile selection identify the document as a PayNote without
    relying only on its type chain.
  value: PayNote
status:
  type: Text
  description: >
    Current rail-neutral lifecycle state of this PayNote. Recommended portable
    values are Pending, Accepted, Initiated, Secured, Completed, Cancelled,
    Reversed, Rejected, and Failed. A PayNote instance normally starts as
    Pending, but the canonical PayNote type does not fix a default value because
    status is mutable lifecycle state.
  schema:
    enum: [Pending, Accepted, Initiated, Secured, Completed, Cancelled, Reversed, Rejected, Failed]
currency:
  type: Common/Currency
  description: >
    Currency of the governed transaction, normally an ISO 4217 currency code.
    Amount fields use minor units of this currency unless a rail-specific
    extension explicitly defines another unit model.
participants:
  description: >
    Optional participant references for the conceptual Payer, Payee, and
    Guarantor roles. Concrete deployments may represent participants through
    Timeline channels, account references, identities, documents, or provider
    records. These references are descriptive and policy-relevant; they do not
    by themselves prove identity or authorization.
  payer:
    description: >
      Party providing funds or value for the transaction. The payer may be a
      person, organization, account holder, wallet owner, agent principal, or
      other funding party.
  payee:
    description: >
      Party intended to receive value. The payee may be unresolved when the
      PayNote is accepted, for example in deferred recipient selection or
      agent-shopping flows.
  guarantor:
    description: >
      Trusted party that controls, executes, or authoritatively reports the
      underlying transaction. Examples include a bank, card issuer, payment
      processor, wallet, platform operator, or internal ledger operator.
amount:
  description: >
    Amount values associated with the governed transaction. All values are in
    minor currency units unless a rail-specific extension says otherwise.
    The fields are cumulative state, not independent instructions.
  expectedTotal:
    type: Integer
    description: >
      Initially expected total amount for the governed transaction in minor
      units. This is the amount participants expect before final amount
      resolution, partial completion, reversal, or rail-specific adjustment.
  finalResolved:
    type: Integer
    description: >
      Final amount in minor units that this PayNote should complete for, when
      known. It may differ from expectedTotal because of tip, mileage, partial
      fulfillment, hotel final charge, usage, adjustment, or other final amount
      resolution.
  finalAmountResolved:
    type: Boolean
    description: >
      Whether finalResolved has been explicitly resolved by the Guarantor or by
      a process the Guarantor accepts. False means finalResolved should not be
      treated as authoritative even if a numeric value is present.
  secured:
    type: Integer
    description: >
      Amount currently secured, reserved, authorized, held, committed, or
      otherwise controlled by the Guarantor for this transaction.
  completed:
    type: Integer
    description: >
      Amount already completed, captured, settled, released, booked, or
      otherwise finalized under this PayNote.
  reversed:
    type: Integer
    description: >
      Amount already reversed, refunded, returned, compensated, or otherwise
      moved back after completion.
transactionDetails:
  description: >
    Transaction attachment information confirmed by the Guarantor. The base
    PayNote type intentionally leaves this opaque so provider-specific packages
    can define stronger rail-specific shapes, such as card authorization codes,
    bank transfer references, wallet transaction ids, or ledger entry ids.
controls:
  description: >
    PayNote-level control locks that restrict sensitive transitions even when
    the underlying rail might otherwise allow them. Guarantors and rail adapters
    should consult these locks before completion, reversal, or transaction
    detail mutation.
  completionLocked:
    type: Boolean
    description: >
      When true, payment completion should be rejected until explicitly
      unlocked. This can protect a customer until delivery, approval, or another
      condition is satisfied.
  reversalLocked:
    type: Boolean
    description: >
      When true, reversal after completion should be rejected until explicitly
      unlocked. This can protect dispute handling, fraud review, or settlement
      finality requirements.
  transactionDetailsUpdateLocked:
    type: Boolean
    description: >
      When true, transaction-detail or payee-recipient updates should be
      rejected until explicitly unlocked. This can freeze recipient details
      after approval or after a risk decision.
payNoteInitialStateDescription:
  description: >
    Human-readable explanation of what this PayNote is for. This text is
    identity-bearing Blue content and should describe the intended process,
    conditions, participants, and special rules clearly enough for participants
    and auditors to understand the PayNote without external documentation.
  summary:
    type: Text
    description: >
      Short explanation of the PayNote purpose, suitable for lists, approvals,
      and participant-facing summaries.
  details:
    type: Text
    description: >
      Full explanation of the process, conditions, participant expectations,
      special rules, and relevant context. Markdown is recommended when human
      readability matters.
contracts:
  payerChannel:
    type: Conversation/Timeline Channel
    description: >
      Timeline channel for actions, requests, or confirmations from the payer
      or payer-side agent. Concrete PayNotes bind this channel to a provider
      and account or timeline.
  payeeChannel:
    type: Conversation/Timeline Channel
    description: >
      Timeline channel for actions, requests, or confirmations from the payee
      or payee-side agent. This channel may remain unbound initially when the
      payee or recipient target is unresolved.
  guarantorChannel:
    type: Conversation/Timeline Channel
    description: >
      Timeline channel for authoritative Guarantor responses and rail outcome
      events. Guarantor messages are the source of truth for what happened on
      the underlying transaction rail.
  acceptPayNote:
    type: Conversation/Operation
    channel: guarantorChannel
    request:
      type: PayNote/PayNote Accepted
    description: >
      Operation through which the Guarantor records acceptance of this PayNote
      as governing the referenced transaction flow.
  rejectPayNote:
    type: Conversation/Operation
    channel: guarantorChannel
    request:
      type: PayNote/PayNote Rejected
    description: >
      Operation through which the Guarantor records rejection of this PayNote.
  recordTransactionInitiated:
    type: Conversation/Operation
    channel: guarantorChannel
    request:
      type: PayNote/Transaction Initiated
    description: >
      Operation through which the Guarantor records that the underlying
      transaction has been initiated and identifies its rail attachment details.
  recordTransactionInitiationFailed:
    type: Conversation/Operation
    channel: guarantorChannel
    request:
      type: PayNote/Transaction Initiation Failed
    description: >
      Operation through which the Guarantor records failure to initiate the
      underlying transaction.
  confirmPayeeAssignment:
    type: Conversation/Operation
    channel: guarantorChannel
    request:
      type: PayNote/Payee Assignment Confirmed
    description: >
      Operation through which the Guarantor confirms an assigned payee or
      recipient target for a PayNote that previously had unresolved recipient
      details.
  rejectPayeeAssignment:
    type: Conversation/Operation
    channel: guarantorChannel
    request:
      type: PayNote/Payee Assignment Rejected
    description: >
      Operation through which the Guarantor rejects a proposed payee or
      recipient target.
  confirmTransactionDetailsUpdated:
    type: Conversation/Operation
    channel: guarantorChannel
    request:
      type: PayNote/Transaction Details Updated
    description: >
      Operation through which the Guarantor confirms and records updated
      transaction details.
  rejectTransactionDetailsUpdate:
    type: Conversation/Operation
    channel: guarantorChannel
    request:
      type: PayNote/Transaction Details Update Rejected
    description: >
      Operation through which the Guarantor rejects a proposed transaction
      details update as a business, risk, policy, or validation decision.
  failTransactionDetailsUpdate:
    type: Conversation/Operation
    channel: guarantorChannel
    request:
      type: PayNote/Transaction Details Update Failed
    description: >
      Operation through which the Guarantor records that a transaction-details
      update failed because of a technical or provider-level issue.
  secureFunds:
    type: Conversation/Operation
    channel: guarantorChannel
    request:
      type: PayNote/Funds Secured
    description: >
      Operation through which the Guarantor confirms that value is secured or
      under its control.
  declineSecureFunds:
    type: Conversation/Operation
    channel: guarantorChannel
    request:
      type: PayNote/Funds Securing Declined
    description: >
      Operation through which the Guarantor declines a request to secure funds.
  failSecureFunds:
    type: Conversation/Operation
    channel: guarantorChannel
    request:
      type: PayNote/Funds Securing Failed
    description: >
      Operation through which the Guarantor records technical or provider-level
      failure while trying to secure funds.
  completePayment:
    type: Conversation/Operation
    channel: guarantorChannel
    request:
      type: PayNote/Payment Completed
    description: >
      Operation through which the Guarantor confirms that the payment was
      completed or finalized.
  declineCompletePayment:
    type: Conversation/Operation
    channel: guarantorChannel
    request:
      type: PayNote/Payment Completion Declined
    description: >
      Operation through which the Guarantor declines a request to complete the
      payment.
  failCompletePayment:
    type: Conversation/Operation
    channel: guarantorChannel
    request:
      type: PayNote/Payment Completion Failed
    description: >
      Operation through which the Guarantor records technical or provider-level
      failure while trying to complete the payment.
  cancelBeforeCompletion:
    type: Conversation/Operation
    channel: guarantorChannel
    request:
      type: PayNote/Payment Cancelled Before Completion
    description: >
      Operation through which the Guarantor confirms cancellation before
      completion.
  declineCancelBeforeCompletion:
    type: Conversation/Operation
    channel: guarantorChannel
    request:
      type: PayNote/Payment Cancellation Declined
    description: >
      Operation through which the Guarantor declines a cancellation request.
  failCancelBeforeCompletion:
    type: Conversation/Operation
    channel: guarantorChannel
    request:
      type: PayNote/Payment Cancellation Failed
    description: >
      Operation through which the Guarantor records technical or provider-level
      failure while trying to cancel before completion.
  reverseAfterCompletion:
    type: Conversation/Operation
    channel: guarantorChannel
    request:
      type: PayNote/Payment Reversed After Completion
    description: >
      Operation through which the Guarantor confirms reversal after completion.
  declineReverseAfterCompletion:
    type: Conversation/Operation
    channel: guarantorChannel
    request:
      type: PayNote/Payment Reversal Declined
    description: >
      Operation through which the Guarantor declines a reversal request.
  failReverseAfterCompletion:
    type: Conversation/Operation
    channel: guarantorChannel
    request:
      type: PayNote/Payment Reversal Failed
    description: >
      Operation through which the Guarantor records technical or provider-level
      failure while trying to reverse after completion.
  resolveFinalAmount:
    type: Conversation/Operation
    channel: guarantorChannel
    request:
      type: PayNote/Final Amount Resolved
    description: >
      Operation through which the Guarantor confirms the final amount this
      PayNote should complete for.
  rejectFinalAmountResolution:
    type: Conversation/Operation
    channel: guarantorChannel
    request:
      type: PayNote/Final Amount Resolution Rejected
    description: >
      Operation through which the Guarantor rejects a proposed final amount.
  lockPaymentCompletion:
    type: Conversation/Operation
    channel: guarantorChannel
    request:
      type: PayNote/Payment Completion Locked
    description: >
      Operation through which the Guarantor records that payment completion is
      locked.
  unlockPaymentCompletion:
    type: Conversation/Operation
    channel: guarantorChannel
    request:
      type: PayNote/Payment Completion Unlocked
    description: >
      Operation through which the Guarantor records that payment completion is
      unlocked.
  failPaymentCompletionLockChange:
    type: Conversation/Operation
    channel: guarantorChannel
    request:
      type: PayNote/Payment Completion Lock Change Failed
    description: >
      Operation through which the Guarantor records failure to change the
      payment completion lock.
  lockPaymentReversal:
    type: Conversation/Operation
    channel: guarantorChannel
    request:
      type: PayNote/Payment Reversal Locked
    description: >
      Operation through which the Guarantor records that payment reversal is
      locked.
  unlockPaymentReversal:
    type: Conversation/Operation
    channel: guarantorChannel
    request:
      type: PayNote/Payment Reversal Unlocked
    description: >
      Operation through which the Guarantor records that payment reversal is
      unlocked.
  failPaymentReversalLockChange:
    type: Conversation/Operation
    channel: guarantorChannel
    request:
      type: PayNote/Payment Reversal Lock Change Failed
    description: >
      Operation through which the Guarantor records failure to change the
      payment reversal lock.
  lockTransactionDetailsUpdate:
    type: Conversation/Operation
    channel: guarantorChannel
    request:
      type: PayNote/Transaction Details Update Locked
    description: >
      Operation through which the Guarantor records that transaction details
      updates are locked.
  unlockTransactionDetailsUpdate:
    type: Conversation/Operation
    channel: guarantorChannel
    request:
      type: PayNote/Transaction Details Update Unlocked
    description: >
      Operation through which the Guarantor records that transaction details
      updates are unlocked.
  failTransactionDetailsUpdateLockChange:
    type: Conversation/Operation
    channel: guarantorChannel
    request:
      type: PayNote/Transaction Details Update Lock Change Failed
    description: >
      Operation through which the Guarantor records failure to change the
      transaction-details-update lock.
```

#### Transaction Status

```yaml
name: Transaction Status
description: >
  PayNote package supporting type describing the observable status of an
  underlying rail transaction. It may be used inside transactionDetails,
  provider reports, or rail-specific extensions. It is descriptive state about
  the rail, not a request by itself. PayNote status remains the process state;
  Transaction Status represents rail-facing status.
status:
  type: Text
  description: >
    Provider or rail status label for the underlying transaction, such as
    authorized, held, captured, settled, failed, cancelled, voided, refunded,
    or reversed. Concrete rail profiles SHOULD define their permitted values.
authorizedAmountMinor:
  type: Integer
  description: Amount authorized or held by the rail in minor units.
capturedAmountMinor:
  type: Integer
  description: Amount captured, settled, or completed by the rail in minor units.
currency:
  type: Common/Currency
  description: Currency used by the amount fields.
```

#### Card Transaction Details

```yaml
name: Card Transaction Details
description: >
  PayNote package rail-specific detail type for identifying a card transaction
  or card authorization. These fields are not required by the provider-neutral
  PayNote type, but card PayNote profiles can use them to attach a PayNote to
  a card authorization, capture, void, or reversal flow.
retrievalReferenceNumber:
  type: Text
  description: Card-network retrieval reference number when available.
systemTraceAuditNumber:
  type: Text
  description: Card-network system trace audit number when available.
transmissionDateTime:
  type: Common/Timestamp
  description: Network or provider timestamp associated with the card message.
authorizationCode:
  type: Text
  description: Authorization code assigned by the issuer or card processor.
```

#### Card Transaction PayNote

```yaml
name: Card Transaction PayNote
type: PayNote/PayNote
description: >
  PayNote subtype for a PayNote attached to a card transaction. It preserves
  the base invariant that one PayNote governs one transaction, while adding a
  cardTransactionDetails field for card-specific attachment evidence. Card
  rail packages may extend this type with issuer, acquirer, network, merchant,
  authorization, capture, void, or refund details.
cardTransactionDetails:
  type: PayNote/Card Transaction Details
  description: Card-specific details identifying the governed card transaction.
```

#### Merchant To Customer PayNote

```yaml
name: Merchant To Customer PayNote
type: PayNote/PayNote
description: >
  PayNote subtype for a transaction where a merchant, platform, or seller moves
  value to a customer. Examples include voucher credit, refund, goodwill credit,
  cashback, customer compensation, or post-purchase benefit. It is still a
  PayNote for exactly one transaction; any further spend of the credited value
  is represented by independent transactions and optional additional PayNotes.
```

---

## 5. Canonical Participant Request Types

Participant request types are messages asking the Guarantor or another responsible participant to perform an action. A request does not by itself prove the action happened.

#### PayNote Acceptance Requested

```yaml
name: PayNote Acceptance Requested
type: Conversation/Request
description: >
  PayNote package request by a participant asking the Guarantor to accept this
  PayNote as governing the referenced transaction or transaction flow. The
  request expresses the participant's desire to attach process rules to a rail
  transaction; acceptance is authoritative only when the Guarantor later emits
  PayNote Accepted.
```

#### Payee Assignment Requested

```yaml
name: Payee Assignment Requested
type: Conversation/Request
description: >
  PayNote package request asking the Guarantor to assign, confirm, or bind the
  payee or recipient target for this PayNote. This is especially useful when a
  PayNote is accepted before the final merchant, payee, account, or recipient
  is known, such as in agent shopping, marketplace sourcing, travel booking,
  or pre-approved spend flows.
payeeRef:
  type: Text
  description: >
    Proposed payee or recipient reference. The interpretation is provider-
    specific and may identify a person, merchant, account, wallet, document,
    external recipient target, or internal customer profile.
recipientTargetRef:
  type: Text
  description: >
    Optional more specific recipient target, such as account number, wallet
    address, merchant target, payout destination, or provider-specific rail
    endpoint.
reason:
  type: Text
  description: Optional explanation supplied by the requester.
```

#### Transaction Details Update Requested

```yaml
name: Transaction Details Update Requested
type: Conversation/Request
description: >
  PayNote package request asking the Guarantor to update transaction-specific
  attachment details. This is the standard request for deferred recipient
  selection, agent shopping flows, merchant detail completion, account target
  changes, or other transaction details that are unresolved when the PayNote is
  first accepted. The Guarantor remains responsible for confirming, rejecting,
  or failing the update.
transactionDetails:
  description: >
    Opaque proposed transaction details payload. Provider-specific PayNote
    subtypes SHOULD give this field a stronger shape when possible.
```

#### Secure Funds Requested

```yaml
name: Secure Funds Requested
type: Conversation/Request
description: >
  PayNote package request asking the Guarantor to secure value for the governed
  transaction. Depending on the rail, securing may mean authorizing, reserving,
  holding, committing, or otherwise placing value under Guarantor control before
  final completion.
amount:
  type: Integer
  description: Requested amount to secure in minor units.
```

#### Complete Payment Requested

```yaml
name: Complete Payment Requested
type: Conversation/Request
description: >
  PayNote package request asking the Guarantor to finalize the governed
  transaction. Depending on the rail, completion may mean capture, settlement,
  release, booking, payout, or another finalization action. A completion request
  should respect final amount resolution and completion locks.
amount:
  type: Integer
  description: Requested completion amount in minor units.
```

#### Cancel Before Completion Requested

```yaml
name: Cancel Before Completion Requested
type: Conversation/Request
description: >
  PayNote package request asking the Guarantor to cancel the governed
  transaction before completion. Depending on the rail, cancellation may mean
  voiding an authorization, releasing a hold, stopping a transfer before
  settlement, or cancelling a reservation.
reason:
  type: Text
  description: Reason for requesting cancellation.
```

#### Reverse After Completion Requested

```yaml
name: Reverse After Completion Requested
type: Conversation/Request
description: >
  PayNote package request asking the Guarantor to reverse value after the
  governed transaction has been completed. Depending on the rail, reversal may
  mean refund, return, charge reversal, or compensating transfer. A reversal
  request should respect reversal locks and already reversed amounts.
amount:
  type: Integer
  description: Requested reversal amount in minor units.
reason:
  type: Text
  description: Reason for requesting reversal.
```

#### Final Amount Resolution Requested

```yaml
name: Final Amount Resolution Requested
type: Conversation/Request
description: >
  PayNote package request proposing the final amount this PayNote should
  complete for. The proposed amount is not authoritative until the Guarantor or
  accepted process emits Final Amount Resolved. This type is used for tips,
  mileage, partial fulfillment, hotels, restaurants, courier flows, usage-based
  pricing, and any other flow where expectedTotal is not final.
finalAmount:
  type: Integer
  description: Proposed final amount in minor units.
reason:
  type: Text
  description: Explanation for the proposed final amount.
```

#### Payment Completion Lock Requested

```yaml
name: Payment Completion Lock Requested
type: Conversation/Request
description: >
  PayNote package request asking the Guarantor to lock payment completion so
  subsequent completion attempts are rejected until explicitly unlocked. This
  can protect customers or counterparties while delivery, fulfillment,
  inspection, dispute handling, or other preconditions remain unresolved.
reason:
  type: Text
  description: Reason for requesting the completion lock.
```

#### Payment Completion Unlock Requested

```yaml
name: Payment Completion Unlock Requested
type: Conversation/Request
description: >
  PayNote package request asking the Guarantor to unlock payment completion so
  completion attempts may be processed again according to the PayNote and rail
  rules.
reason:
  type: Text
  description: Reason for requesting the completion unlock.
```

#### Payment Reversal Lock Requested

```yaml
name: Payment Reversal Lock Requested
type: Conversation/Request
description: >
  PayNote package request asking the Guarantor to lock reversal after
  completion so subsequent reversal attempts are rejected until explicitly
  unlocked. This can protect dispute processes, fraud review, settlement
  windows, or other risk controls.
reason:
  type: Text
  description: Reason for requesting the reversal lock.
```

#### Payment Reversal Unlock Requested

```yaml
name: Payment Reversal Unlock Requested
type: Conversation/Request
description: >
  PayNote package request asking the Guarantor to unlock reversal after
  completion so reversal attempts may be processed again according to the
  PayNote and rail rules.
reason:
  type: Text
  description: Reason for requesting the reversal unlock.
```

#### Transaction Details Update Lock Requested

```yaml
name: Transaction Details Update Lock Requested
type: Conversation/Request
description: >
  PayNote package request asking the Guarantor to lock transaction-detail
  updates so payee, recipient, merchant, account, or rail-attachment details
  cannot be changed until explicitly unlocked.
reason:
  type: Text
  description: Reason for requesting the transaction-details-update lock.
```

#### Transaction Details Update Unlock Requested

```yaml
name: Transaction Details Update Unlock Requested
type: Conversation/Request
description: >
  PayNote package request asking the Guarantor to unlock transaction-detail
  updates so payee, recipient, merchant, account, or rail-attachment details
  may be updated again according to the PayNote and rail rules.
reason:
  type: Text
  description: Reason for requesting the transaction-details-update unlock.
```

---

## 6. Canonical Guarantor Response Types

Guarantor response types are authoritative lifecycle events for the governed transaction or PayNote process.

### 6.1 Acceptance and initiation

#### PayNote Accepted

```yaml
name: PayNote Accepted
type: Conversation/Response
description: >
  PayNote package response by which the Guarantor accepts this PayNote as
  governing the referenced transaction or transaction flow. Acceptance means the
  Guarantor recognizes the PayNote as the process document for subsequent
  supported actions, but it does not by itself prove that the underlying rail
  transaction has been initiated, secured, or completed.
```

#### PayNote Rejected

```yaml
name: PayNote Rejected
type: Conversation/Response
description: >
  PayNote package response by which the Guarantor rejects this PayNote for the
  referenced transaction or transaction flow. Rejection means the transaction is
  not governed by this PayNote unless a later accepted PayNote or provider
  policy says otherwise.
reason:
  type: Text
  description: Guarantor-provided reason for rejection.
```

#### Transaction Initiated

```yaml
name: Transaction Initiated
type: Conversation/Response
description: >
  PayNote package response by which the Guarantor confirms that the underlying
  transaction has been initiated and records how this PayNote attaches to it.
  The details are rail-neutral at the base level and may be extended by card,
  bank, wallet, platform, ledger, or provider-specific packages.
railType:
  type: Text
  description: >
    Rail or transaction family, such as card, bank-transfer, wallet, internal-
    ledger, payout, mandate-spend, escrow, or provider-specific value.
attachmentPoint:
  type: Text
  description: >
    Provider-specific point at which the PayNote attached, such as card
    authorization, transfer initiation, SCA confirmation, account hold, internal
    ledger transfer, payout initiation, or settlement instruction.
providerReference:
  type: Text
  description: Provider-specific transaction, authorization, transfer, hold, or ledger reference.
initiatedAmount:
  type: Integer
  description: Amount initiated on the rail in minor units.
```

#### Transaction Initiation Failed

```yaml
name: Transaction Initiation Failed
type: Conversation/Response
description: >
  PayNote package response by which the Guarantor records that it attempted to
  initiate the underlying transaction, or could not initiate it, and the
  initiation failed for a technical, provider-level, rail, integration, or
  operational reason.
reason:
  type: Text
  description: Failure reason.
```

### 6.2 Payee and transaction detail responses

#### Payee Assignment Confirmed

```yaml
name: Payee Assignment Confirmed
type: Conversation/Response
description: >
  PayNote package response by which the Guarantor confirms a payee or recipient
  target for this PayNote. This is authoritative evidence that deferred
  recipient selection has been accepted for the governed transaction.
payeeRef:
  type: Text
  description: Confirmed payee reference.
recipientTargetRef:
  type: Text
  description: Optional confirmed recipient target or rail endpoint.
```

#### Payee Assignment Rejected

```yaml
name: Payee Assignment Rejected
type: Conversation/Response
description: >
  PayNote package response by which the Guarantor rejects a proposed payee or
  recipient assignment.
reason:
  type: Text
  description: Reason for rejection.
```

#### Transaction Details Updated

```yaml
name: Transaction Details Updated
type: Conversation/Response
description: >
  PayNote package response by which the Guarantor confirms and records updated
  transaction details. The confirmed transactionDetails payload becomes the
  Guarantor-accepted attachment information for the governed transaction.
transactionDetails:
  description: >
    Opaque transaction details payload confirmed by the Guarantor. Provider-
    specific packages SHOULD give this field a stronger shape.
payeeRef:
  type: Text
  description: Optional confirmed payee reference when the update includes payee assignment.
recipientTargetRef:
  type: Text
  description: Optional confirmed recipient target or rail endpoint.
merchantReference:
  type: Text
  description: Optional merchant, seller, or provider reference.
displayLabel:
  type: Text
  description: Optional human-readable label for the updated transaction details.
```

#### Transaction Details Update Rejected

```yaml
name: Transaction Details Update Rejected
type: Conversation/Response
description: >
  PayNote package response by which the Guarantor rejects proposed transaction
  details as a business, risk, policy, validation, authorization, or process
  decision.
reason:
  type: Text
  description: Reason for rejection.
```

#### Transaction Details Update Failed

```yaml
name: Transaction Details Update Failed
type: Conversation/Response
description: >
  PayNote package response by which the Guarantor records that it attempted to
  apply a transaction-details update, or could not apply it, and the update
  failed for a technical, provider-level, rail, integration, or operational
  reason.
reason:
  type: Text
  description: Failure reason.
```

### 6.3 Securing funds

#### Funds Secured

```yaml
name: Funds Secured
type: Conversation/Response
description: >
  PayNote package response by which the Guarantor confirms that value is now
  secured, reserved, authorized, held, committed, or otherwise under Guarantor
  control for the governed transaction.
amountSecured:
  type: Integer
  description: Amount secured in minor units.
```

#### Funds Securing Declined

```yaml
name: Funds Securing Declined
type: Conversation/Response
description: >
  PayNote package response by which the Guarantor declines a request to secure
  funds as a business, risk, policy, authorization, limit, or validation
  decision.
reason:
  type: Text
  description: Reason for decline.
```

#### Funds Securing Failed

```yaml
name: Funds Securing Failed
type: Conversation/Response
description: >
  PayNote package response by which the Guarantor records that securing funds
  failed because of a technical, provider-level, rail, integration, or
  operational issue.
reason:
  type: Text
  description: Failure reason.
```

### 6.4 Completing payment

#### Payment Completed

```yaml
name: Payment Completed
type: Conversation/Response
description: >
  PayNote package response by which the Guarantor confirms that the governed
  transaction was completed, captured, settled, released, booked, paid out, or
  otherwise finalized according to the rail and PayNote rules.
amountCompleted:
  type: Integer
  description: Amount completed in minor units.
```

#### Payment Completion Declined

```yaml
name: Payment Completion Declined
type: Conversation/Response
description: >
  PayNote package response by which the Guarantor declines a request to complete
  payment as a business, risk, policy, authorization, lock, limit, or validation
  decision.
reason:
  type: Text
  description: Reason for decline.
```

#### Payment Completion Failed

```yaml
name: Payment Completion Failed
type: Conversation/Response
description: >
  PayNote package response by which the Guarantor records that completion failed
  because of a technical, provider-level, rail, integration, or operational
  issue.
reason:
  type: Text
  description: Failure reason.
```

### 6.5 Cancelling before completion

#### Payment Cancelled Before Completion

```yaml
name: Payment Cancelled Before Completion
type: Conversation/Response
description: >
  PayNote package response by which the Guarantor confirms that the governed
  transaction was cancelled before completion. Depending on the rail, this may
  correspond to void, authorization release, stopped transfer, hold release, or
  reservation cancellation.
```

#### Payment Cancellation Declined

```yaml
name: Payment Cancellation Declined
type: Conversation/Response
description: >
  PayNote package response by which the Guarantor declines a request to cancel
  before completion as a business, risk, policy, authorization, rail-state, or
  validation decision.
reason:
  type: Text
  description: Reason for decline.
```

#### Payment Cancellation Failed

```yaml
name: Payment Cancellation Failed
type: Conversation/Response
description: >
  PayNote package response by which the Guarantor records that cancellation
  before completion failed because of a technical, provider-level, rail,
  integration, or operational issue.
reason:
  type: Text
  description: Failure reason.
```

### 6.6 Reversing after completion

#### Payment Reversed After Completion

```yaml
name: Payment Reversed After Completion
type: Conversation/Response
description: >
  PayNote package response by which the Guarantor confirms that value was
  reversed after the governed transaction had completed. Depending on the rail,
  this may correspond to refund, reversal, return, chargeback-related movement,
  or compensating transfer.
amountReversed:
  type: Integer
  description: Amount reversed in minor units.
```

#### Payment Reversal Declined

```yaml
name: Payment Reversal Declined
type: Conversation/Response
description: >
  PayNote package response by which the Guarantor declines a reversal request as
  a business, risk, policy, authorization, rail-state, lock, limit, or validation
  decision.
reason:
  type: Text
  description: Reason for decline.
```

#### Payment Reversal Failed

```yaml
name: Payment Reversal Failed
type: Conversation/Response
description: >
  PayNote package response by which the Guarantor records that reversal failed
  because of a technical, provider-level, rail, integration, or operational
  issue.
reason:
  type: Text
  description: Failure reason.
```

### 6.7 Final amount resolution

#### Final Amount Resolved

```yaml
name: Final Amount Resolved
type: Conversation/Response
description: >
  PayNote package response by which the Guarantor confirms the final amount this
  PayNote should complete for. After this response, finalResolved and
  finalAmountResolved can be treated as authoritative according to the PayNote
  and rail profile.
finalAmount:
  type: Integer
  description: Final resolved amount in minor units.
```

#### Final Amount Resolution Rejected

```yaml
name: Final Amount Resolution Rejected
type: Conversation/Response
description: >
  PayNote package response by which the Guarantor rejects a proposed final
  amount as a business, risk, policy, validation, evidence, or authorization
  decision.
reason:
  type: Text
  description: Reason for rejection.
```

### 6.8 Control lock responses

#### Payment Completion Locked

```yaml
name: Payment Completion Locked
type: Conversation/Response
description: >
  PayNote package response by which the Guarantor confirms that payment
  completion is locked. Subsequent completion attempts should be rejected until
  Payment Completion Unlocked is recorded.
lockedAt:
  type: Common/Timestamp
  description: Timestamp when the completion lock was recorded.
```

#### Payment Completion Unlocked

```yaml
name: Payment Completion Unlocked
type: Conversation/Response
description: >
  PayNote package response by which the Guarantor confirms that payment
  completion is unlocked. Subsequent completion attempts may be considered again
  according to the PayNote and rail rules.
unlockedAt:
  type: Common/Timestamp
  description: Timestamp when the completion unlock was recorded.
```

#### Payment Completion Lock Change Failed

```yaml
name: Payment Completion Lock Change Failed
type: Conversation/Response
description: >
  PayNote package response by which the Guarantor records that a requested
  completion-lock or completion-unlock change failed because of a technical,
  provider-level, rail, integration, or operational issue.
reason:
  type: Text
  description: Failure reason.
```

#### Payment Reversal Locked

```yaml
name: Payment Reversal Locked
type: Conversation/Response
description: >
  PayNote package response by which the Guarantor confirms that payment reversal
  is locked. Subsequent reversal attempts should be rejected until Payment
  Reversal Unlocked is recorded.
lockedAt:
  type: Common/Timestamp
  description: Timestamp when the reversal lock was recorded.
```

#### Payment Reversal Unlocked

```yaml
name: Payment Reversal Unlocked
type: Conversation/Response
description: >
  PayNote package response by which the Guarantor confirms that payment reversal
  is unlocked. Subsequent reversal attempts may be considered again according to
  the PayNote and rail rules.
unlockedAt:
  type: Common/Timestamp
  description: Timestamp when the reversal unlock was recorded.
```

#### Payment Reversal Lock Change Failed

```yaml
name: Payment Reversal Lock Change Failed
type: Conversation/Response
description: >
  PayNote package response by which the Guarantor records that a requested
  reversal-lock or reversal-unlock change failed because of a technical,
  provider-level, rail, integration, or operational issue.
reason:
  type: Text
  description: Failure reason.
```

#### Transaction Details Update Locked

```yaml
name: Transaction Details Update Locked
type: Conversation/Response
description: >
  PayNote package response by which the Guarantor confirms that transaction
  details updates are locked. Subsequent updates to payee, recipient, merchant,
  account, or rail attachment details should be rejected until Transaction
  Details Update Unlocked is recorded.
lockedAt:
  type: Common/Timestamp
  description: Timestamp when the transaction-details-update lock was recorded.
```

#### Transaction Details Update Unlocked

```yaml
name: Transaction Details Update Unlocked
type: Conversation/Response
description: >
  PayNote package response by which the Guarantor confirms that transaction
  details updates are unlocked. Subsequent transaction details updates may be
  considered again according to the PayNote and rail rules.
unlockedAt:
  type: Common/Timestamp
  description: Timestamp when the transaction-details-update unlock was recorded.
```

#### Transaction Details Update Lock Change Failed

```yaml
name: Transaction Details Update Lock Change Failed
type: Conversation/Response
description: >
  PayNote package response by which the Guarantor records that a requested
  transaction-details-update lock or unlock failed because of a technical,
  provider-level, rail, integration, or operational issue.
reason:
  type: Text
  description: Failure reason.
```

---

## 7. Triggered Transaction and Linked PayNote Types

Triggered transaction types describe requests and responses for starting new independent transactions related to a PayNote. They do not create child PayNotes. They create or refer to separate transactions, each of which may have its own PayNote.

#### Linked Card Charge Requested

```yaml
name: Linked Card Charge Requested
type: Conversation/Request
description: >
  PayNote package request to initiate a new independent card charge related to
  an existing PayNote or payment mandate, with a linked PayNote document that
  will govern the new charge. This does not create a child PayNote of the
  current PayNote. It requests a separate transaction that may have its own
  PayNote.
amount:
  type: Integer
  description: Requested amount for the new card charge in minor units.
paymentMandateDocumentId:
  type: Text
  description: Optional document id of the Payment Mandate authorizing this charge.
paynote:
  description: >
    PayNote document or PayNote reference intended to govern the new independent
    card charge.
```

#### Linked Card Charge and Capture Immediately Requested

```yaml
name: Linked Card Charge and Capture Immediately Requested
type: Conversation/Request
description: >
  PayNote package request to initiate a new independent card charge and complete
  or capture it immediately, subject to payment mandate, rail, and Guarantor
  rules. The linked PayNote governs that new transaction, not the transaction
  governed by the requesting PayNote.
amount:
  type: Integer
  description: Requested amount for the new card charge in minor units.
paymentMandateDocumentId:
  type: Text
  description: Optional document id of the Payment Mandate authorizing this charge.
paynote:
  description: PayNote document or PayNote reference intended to govern the new card charge.
```

#### Reverse Card Charge Requested

```yaml
name: Reverse Card Charge Requested
type: Conversation/Request
description: >
  PayNote package request to initiate a new independent reversal or refund flow
  for a card charge, optionally governed by its own linked PayNote. This type is
  card-specific; provider-neutral PayNotes SHOULD use Reverse After Completion
  Requested unless card-specific semantics are required.
amount:
  type: Integer
  description: Requested reversal amount in minor units.
paymentMandateDocumentId:
  type: Text
  description: Optional mandate document authorizing the reversal request.
paynote:
  description: Optional PayNote document or reference for the new reversal transaction.
```

#### Reverse Card Charge and Capture Immediately Requested

```yaml
name: Reverse Card Charge and Capture Immediately Requested
type: Conversation/Request
description: >
  PayNote package card-specific request to reverse a card charge and complete
  the related movement immediately when the rail supports that behavior. This is
  a rail-specific extension and should not replace provider-neutral reversal
  vocabulary in generic PayNote documents.
amount:
  type: Integer
  description: Requested amount in minor units.
paymentMandateDocumentId:
  type: Text
  description: Optional mandate document authorizing the reversal request.
paynote:
  description: Optional PayNote document or reference for the new transaction.
```

#### Linked PayNote Started

```yaml
name: Linked PayNote Started
type: Conversation/Response
description: >
  PayNote package response confirming that startup of a linked independent
  PayNote document was requested successfully for a newly triggered transaction.
  The linked PayNote governs the triggered transaction and is not a child
  PayNote of the current PayNote.
payNoteSessionId:
  type: Text
  description: Session id of the bootstrapped linked PayNote document.
payNoteDocumentId:
  type: Text
  description: Document id of the bootstrapped linked PayNote when available.
```

#### Linked PayNote Start Responded

```yaml
name: Linked PayNote Start Responded
type: Conversation/Response
description: >
  PayNote package response reporting the status of a linked PayNote startup
  attempt for a triggered independent transaction. A successful status may be
  followed by Linked PayNote Started with concrete document/session identifiers.
status:
  type: Text
  description: Provider-defined status of the linked PayNote startup attempt.
reason:
  type: Text
  description: Optional reason when startup was rejected, failed, delayed, or otherwise not completed.
```

#### Linked PayNote Start Failed

```yaml
name: Linked PayNote Start Failed
type: Conversation/Response
description: >
  PayNote package response reporting that startup of a linked independent
  PayNote document failed for a technical, provider-level, rail, integration,
  or operational reason.
reason:
  type: Text
  description: Failure reason.
```

---

## 8. Payment Mandate Extension Types

A Payment Mandate is not a PayNote. It is an authorization document that may allow another party or document to initiate payment requests under limits. PayNotes can use mandates for agent budgets, delegated merchant credits, voucher spending, and linked transaction startup.

#### Payment Mandate

```yaml
name: Payment Mandate
description: >
  PayNote package supporting document that authorizes a grantee to execute
  payment requests on behalf of a granter under explicit limits, with
  deterministic spend tracking and optional linked PayNote startup rules. A
  Payment Mandate is not itself a PayNote and does not govern exactly one
  transaction. It is an authority and limit document that may be referenced by
  PayNotes or payment requests. Spending under a mandate may create independent
  transactions, and those transactions may have their own PayNotes.
granterType:
  type: Text
  description: >
    Type of party granting authority to use funds or payment capacity. Expected
    portable values include merchant and customer; provider profiles may define
    additional granter types.
granterId:
  type: Text
  description: >
    Identifier of the granting party, interpreted according to granterType. For
    example, merchantId when granterType is merchant or customerId when
    granterType is customer.
granteeType:
  type: Text
  description: >
    Type of party allowed to invoke this mandate. Expected portable values
    include documentId, merchantId, and customerId; provider profiles may define
    additional grantee types.
granteeId:
  type: Text
  description: Identifier of the grantee, interpreted according to granteeType.
amountLimit:
  type: Integer
  description: Maximum amount in minor currency units that may be authorized under this mandate.
currency:
  type: Common/Currency
  description: Currency used for amountLimit and tracked amounts.
amountReserved:
  type: Integer
  description: >
    Running amount currently reserved or authorized under this mandate in minor
    units. It increases after successful authorize/reserve actions and decreases
    after release or capture according to mandate settlement rules.
amountCaptured:
  type: Integer
  description: Running amount captured, settled, or completed under this mandate in minor units.
sourceAccount:
  type: Text
  description: >
    Funds source to use when this mandate executes. A provider profile may use
    values such as root, a specific account number, a wallet id, a credit line,
    or another funding source reference.
expiresAt:
  type: Common/Timestamp
  description: Optional timestamp after which the mandate is inactive.
revokedAt:
  type: Common/Timestamp
  description: Optional timestamp at which the mandate was revoked. When set, the mandate is inactive.
allowLinkedPayNote:
  type: Boolean
  description: Whether this mandate allows startup of linked PayNotes for triggered transactions.
allowedPayNotes:
  type: List
  itemType:
    typeBlueId:
      type: Text
      description: >
        Optional BlueId of an allowed PayNote type. Mutually exclusive with
        documentBlueId.
    documentBlueId:
      type: Text
      description: >
        Optional BlueId of an allowed concrete PayNote document. Mutually
        exclusive with typeBlueId.
  description: >
    Optional allowlist for linked PayNotes. Missing list means any linked
    PayNote is allowed when allowLinkedPayNote is true. Each item should set
    exactly one selector: typeBlueId or documentBlueId.
allowedPaymentCounterparties:
  type: List
  itemType:
    counterpartyType:
      type: Text
      description: Counterparty identifier type, such as merchantId, customerId, or accountNumber.
    counterpartyId:
      type: Text
      description: Counterparty identifier interpreted according to counterpartyType.
  description: >
    Optional allowlist of permitted payment counterparties. Missing list means
    any counterparty is allowed subject to provider policy.
chargeAttempts:
  type: Dictionary
  keyType: Text
  valueType:
    amountMinor:
      type: Integer
      description: Requested amount for the charge attempt in minor units.
    currency:
      type: Common/Currency
      description: Currency requested for the charge attempt.
    counterpartyType:
      type: Text
      description: Counterparty identifier type.
    counterpartyId:
      type: Text
      description: Counterparty identifier interpreted according to counterpartyType.
    chargeMode:
      type: Text
      description: Requested charge mode, such as authorize_only or authorize_and_capture.
    authorizationStatus:
      type: Text
      description: Authorization decision status, such as approved or rejected.
    authorizationReason:
      type: Text
      description: Optional authorization rejection or information reason.
    authorizationRespondedAt:
      type: Common/Timestamp
      description: Timestamp of authorization decision response.
    authorizedAmountMinor:
      type: Integer
      description: Amount authorized and reserved against mandate capacity for this attempt.
    settled:
      type: Boolean
      description: Whether mandate settlement was accepted for this attempt.
    lastSettlementRequestStatus:
      type: Text
      description: Last settlement request status received from the bank or provider.
    lastSettlementProcessingStatus:
      type: Text
      description: Mandate processing status for the last settlement response.
    settlementReason:
      type: Text
      description: Optional settlement rejection, failure, or information reason.
    settlementRespondedAt:
      type: Common/Timestamp
      description: Timestamp of last settlement processing response.
    reservedDeltaMinor:
      type: Integer
      description: Reserved amount delta applied for the accepted settlement.
    capturedDeltaMinor:
      type: Integer
      description: Captured amount delta applied for the accepted settlement.
    holdId:
      type: Text
      description: Optional hold id associated with settlement.
    transactionId:
      type: Text
      description: Optional transaction id associated with settlement.
    lastSettlementId:
      type: Text
      description: Optional last applied settlement id used for idempotency tracking.
  description: >
    Stateful map of authorization id to authorization and settlement state. The
    exact update rules are defined by the mandate implementation profile, not by
    the PayNote semantic type alone.
```

#### Payment Mandate Spend Authorization Requested

```yaml
name: Payment Mandate Spend Authorization Requested
type: Conversation/Request
description: >
  PayNote package request asking the mandate Guarantor to authorize a spend
  attempt under a Payment Mandate. This request is used by delegated parties,
  agents, documents, or merchants that need to consume mandate capacity for a
  new transaction or rail action.
authorizationId:
  type: Text
  description: Idempotency and tracking identifier for this authorization attempt.
amountMinor:
  type: Integer
  description: Requested amount in minor currency units.
currency:
  type: Common/Currency
  description: Requested currency.
counterpartyType:
  type: Text
  description: Counterparty identifier type, such as merchantId, customerId, or accountNumber.
counterpartyId:
  type: Text
  description: Counterparty identifier interpreted according to counterpartyType.
requestingDocumentId:
  type: Text
  description: Optional document id of the requesting PayNote or process document.
requestingSessionId:
  type: Text
  description: Optional session id of the requesting document instance.
requestedAt:
  type: Common/Timestamp
  description: Timestamp at which authorization was requested.
```

#### Payment Mandate Spend Authorization Responded

```yaml
name: Payment Mandate Spend Authorization Responded
type: Conversation/Response
description: >
  PayNote package response reporting whether a spend attempt under a Payment
  Mandate was authorized. Approval reserves capacity according to the mandate;
  rejection leaves capacity unchanged except for any provider-defined audit
  state.
authorizationId:
  type: Text
  description: Authorization id this response refers to.
status:
  type: Text
  description: Authorization response status, such as approved or rejected.
reason:
  type: Text
  description: Optional rejection or information reason.
remainingAmountMinor:
  type: Integer
  description: Remaining available amount after the response, in minor units.
respondedAt:
  type: Common/Timestamp
  description: Timestamp of the response.
```

#### Payment Mandate Spend Settled

```yaml
name: Payment Mandate Spend Settled
type: Conversation/Response
description: >
  PayNote package response recording settlement of a previously authorized
  mandate spend. Settlement may adjust reserved and captured mandate amounts and
  may attach hold or transaction identifiers returned by a bank, card processor,
  platform, or ledger.
authorizationId:
  type: Text
  description: Authorization id being settled.
settlementId:
  type: Text
  description: Idempotency and tracking identifier for the settlement.
status:
  type: Text
  description: Settlement status, such as succeeded, failed, accepted, or rejected.
reservedDeltaMinor:
  type: Integer
  description: Change to reserved amount in minor units.
capturedDeltaMinor:
  type: Integer
  description: Change to captured amount in minor units.
holdId:
  type: Text
  description: Optional hold identifier.
transactionId:
  type: Text
  description: Optional transaction identifier.
reason:
  type: Text
  description: Optional failure, rejection, or information reason.
respondedAt:
  type: Common/Timestamp
  description: Timestamp of settlement response.
```

#### Payment Mandate Spend Settlement Responded

```yaml
name: Payment Mandate Spend Settlement Responded
type: Conversation/Response
description: >
  PayNote package response summarizing mandate-side processing of a settlement
  request. It records the current reserved and captured totals after processing.
authorizationId:
  type: Text
  description: Authorization id associated with the settlement.
settlementId:
  type: Text
  description: Settlement id this response refers to.
status:
  type: Text
  description: Processing status, such as accepted, rejected, succeeded, or failed.
reason:
  type: Text
  description: Optional reason.
amountReserved:
  type: Integer
  description: Current reserved amount after processing, in minor units.
amountCaptured:
  type: Integer
  description: Current captured amount after processing, in minor units.
respondedAt:
  type: Common/Timestamp
  description: Timestamp of response.
```

#### Payment Mandate Attached

```yaml
name: Payment Mandate Attached
type: Conversation/Event
description: >
  PayNote package event indicating that a Payment Mandate document was attached
  to a PayNote, delivery, transaction, or related process. The mandate may be
  used as authority for later spend or triggered transaction requests.
paymentMandateDocumentId:
  type: Text
  description: Document id of the attached Payment Mandate.
```

#### Payment Mandate Attachment Failed

```yaml
name: Payment Mandate Attachment Failed
type: Conversation/Event
description: >
  PayNote package event indicating that attaching a Payment Mandate failed for
  a technical, provider-level, authorization, validation, or operational reason.
reason:
  type: Text
  description: Failure reason.
```

---

## 9. Card Transaction Extension Types

These types are card-rail extensions. They should not replace the provider-neutral PayNote vocabulary unless card-specific semantics are required.

#### Card Transaction Report

```yaml
name: Card Transaction Report
type: Conversation/Event
description: >
  PayNote package card extension event reporting a card transaction observed by
  a card issuer, bank, processor, monitoring service, or other card-rail
  participant. It can be used to identify a transaction and start, attach, or
  update a Card Transaction PayNote.
transactionId:
  type: Text
  description: Provider transaction id.
merchantId:
  type: Text
  description: Merchant identifier associated with the card transaction.
amountMinor:
  type: Integer
  description: Transaction amount in minor units.
currency:
  type: Common/Currency
  description: Transaction currency.
occurredAt:
  type: Common/Timestamp
  description: Timestamp at which the card transaction occurred or was observed.
status:
  type: Text
  description: Card transaction status reported by the provider.
cardTransactionDetails:
  type: PayNote/Card Transaction Details
  description: Card-network or processor details associated with the transaction.
```

#### Start Card Transaction Monitoring Requested

```yaml
name: Start Card Transaction Monitoring Requested
type: Conversation/Request
description: >
  PayNote package card extension request asking a provider to monitor card
  transactions for a target merchant or card-account context and report matching
  events for PayNote attachment or delivery.
targetMerchantId:
  type: Text
  description: Merchant id to monitor.
events:
  type: List
  itemType: Text
  description: Card transaction event kinds requested for monitoring.
requestedAt:
  type: Common/Timestamp
  description: Timestamp when monitoring was requested.
```

#### Card Transaction Monitoring Started

```yaml
name: Card Transaction Monitoring Started
type: Conversation/Response
description: >
  PayNote package card extension response confirming that card transaction
  monitoring has started for the requested target and event set.
targetMerchantId:
  type: Text
  description: Merchant id being monitored.
events:
  type: List
  itemType: Text
  description: Event kinds being monitored.
startedAt:
  type: Common/Timestamp
  description: Timestamp when monitoring started.
consentDocumentId:
  type: Text
  description: Optional customer consent document id authorizing monitoring.
consentSessionId:
  type: Text
  description: Optional customer consent session id authorizing monitoring.
```

#### Card Transaction Monitoring Request Rejected

```yaml
name: Card Transaction Monitoring Request Rejected
type: Conversation/Response
description: >
  PayNote package card extension response rejecting a request to monitor card
  transactions.
targetMerchantId:
  type: Text
  description: Merchant id that was requested for monitoring.
events:
  type: List
  itemType: Text
  description: Requested event kinds.
reason:
  type: Text
  description: Reason for rejection.
rejectedAt:
  type: Common/Timestamp
  description: Timestamp of rejection.
```

#### Card Transaction Monitoring Stopped

```yaml
name: Card Transaction Monitoring Stopped
type: Conversation/Event
description: >
  PayNote package card extension event indicating that card transaction
  monitoring stopped for a target merchant or event set.
targetMerchantId:
  type: Text
  description: Merchant id whose monitoring stopped.
events:
  type: List
  itemType: Text
  description: Event kinds that stopped being monitored.
stoppedAt:
  type: Common/Timestamp
  description: Timestamp when monitoring stopped.
reason:
  type: Text
  description: Optional reason monitoring stopped.
```

#### Card Charge Responded

```yaml
name: Card Charge Responded
type: Conversation/Response
description: >
  PayNote package card extension response reporting the result of a card charge
  request associated with a PayNote or Payment Mandate.
status:
  type: Text
  description: Card charge status, such as approved, rejected, declined, failed, or pending.
reason:
  type: Text
  description: Optional reason for non-success or additional information.
paymentMandateDocumentId:
  type: Text
  description: Optional Payment Mandate document id used for the charge.
```

#### Card Charge Completed

```yaml
name: Card Charge Completed
type: Conversation/Response
description: >
  PayNote package card extension response confirming that a card charge was
  completed or reached a final provider-reported state.
status:
  type: Text
  description: Completion status reported by the card provider.
holdId:
  type: Text
  description: Optional hold or authorization id.
transactionId:
  type: Text
  description: Optional card transaction id.
reason:
  type: Text
  description: Optional reason or information text.
```

#### Card Transaction Capture Lock Requested

```yaml
name: Card Transaction Capture Lock Requested
type: Conversation/Request
description: >
  PayNote package card extension request asking the Guarantor to lock capture
  on a card transaction. This is the card-rail analogue of Payment Completion
  Lock Requested.
cardTransactionDetails:
  type: PayNote/Card Transaction Details
  description: Card transaction to lock.
```

#### Card Transaction Capture Unlock Requested

```yaml
name: Card Transaction Capture Unlock Requested
type: Conversation/Request
description: >
  PayNote package card extension request asking the Guarantor to unlock capture
  on a card transaction. This is the card-rail analogue of Payment Completion
  Unlock Requested.
cardTransactionDetails:
  type: PayNote/Card Transaction Details
  description: Card transaction to unlock.
```

#### Card Transaction Capture Locked

```yaml
name: Card Transaction Capture Locked
type: Conversation/Response
description: >
  PayNote package card extension response confirming that capture is locked for
  a card transaction.
lockedAt:
  type: Common/Timestamp
  description: Timestamp when capture was locked.
```

#### Card Transaction Capture Unlocked

```yaml
name: Card Transaction Capture Unlocked
type: Conversation/Response
description: >
  PayNote package card extension response confirming that capture is unlocked
  for a card transaction.
unlockedAt:
  type: Common/Timestamp
  description: Timestamp when capture was unlocked.
```

#### Card Transaction Capture Lock Change Failed

```yaml
name: Card Transaction Capture Lock Change Failed
type: Conversation/Response
description: >
  PayNote package card extension response reporting that a card capture lock or
  unlock change failed for a technical, provider-level, rail, integration, or
  operational reason.
reason:
  type: Text
  description: Failure reason.
```

---

## 10. PayNote Delivery and Client Decision Types

These types support delivery of PayNotes to clients or participants and recording client decisions. They are not payment-rail events by themselves.

#### PayNote Delivery

```yaml
name: PayNote Delivery
description: >
  PayNote package supporting document for delivering a PayNote or PayNote
  bootstrap request to a client, attaching optional payment mandate information,
  collecting a client acceptance or rejection decision, and recording
  transaction identification status. PayNote Delivery is a delivery and consent
  coordination document; it is not itself the governed transaction.
payNoteBootstrapRequest:
  description: PayNote document or bootstrap request being delivered.
paymentMandateBootstrapRequest:
  description: Optional Payment Mandate bootstrap request delivered with the PayNote.
cardTransactionDetails:
  type: PayNote/Card Transaction Details
  description: Optional card transaction details associated with delivery.
deliveryStatus:
  type: Text
  description: Delivery status, such as pending, delivered, failed, accepted, rejected, or discarded.
transactionIdentificationStatus:
  type: Text
  description: Status of identifying the transaction to which the PayNote should attach.
clientDecisionStatus:
  type: Text
  description: Client decision status, such as pending, accepted, rejected, or discarded.
clientAcceptedAt:
  type: Common/Timestamp
  description: Timestamp when the client accepted.
clientRejectedAt:
  type: Common/Timestamp
  description: Timestamp when the client rejected.
clientDecisionDiscardedAt:
  type: Common/Timestamp
  description: Timestamp when a prior client decision was discarded.
```

#### PayNote Accepted By Client

```yaml
name: PayNote Accepted By Client
type: Conversation/Event
description: >
  PayNote package event recording that the client accepted the delivered
  PayNote. Client acceptance is distinct from Guarantor acceptance; Guarantor
  acceptance is recorded with PayNote Accepted.
acceptedAt:
  type: Common/Timestamp
  description: Timestamp when the client accepted.
```

#### PayNote Rejected By Client

```yaml
name: PayNote Rejected By Client
type: Conversation/Event
description: >
  PayNote package event recording that the client rejected the delivered
  PayNote. Client rejection may prevent optional PayNote attachment, depending
  on rail and provider policy.
reason:
  type: Text
  description: Client-provided or provider-recorded rejection reason.
rejectedAt:
  type: Common/Timestamp
  description: Timestamp when the client rejected.
```

#### PayNote Client Decision Discarded

```yaml
name: PayNote Client Decision Discarded
type: Conversation/Event
description: >
  PayNote package event recording that a prior client decision about a delivered
  PayNote was discarded, superseded, expired, or made irrelevant by a later
  state transition.
decision:
  type: Text
  description: Prior decision that was discarded.
reason:
  type: Text
  description: Reason the decision was discarded.
decisionAt:
  type: Common/Timestamp
  description: Timestamp associated with the discarded decision.
```

#### PayNote Delivery Failed

```yaml
name: PayNote Delivery Failed
type: Conversation/Event
description: >
  PayNote package event recording that delivering a PayNote to a client or
  participant failed.
reason:
  type: Text
  description: Failure reason.
```

#### Transaction Identified

```yaml
name: Transaction Identified
type: Conversation/Event
description: >
  PayNote package event recording that the transaction intended to be governed
  by a PayNote has been identified. This event is evidence for attachment; it
  does not by itself mean the Guarantor has accepted the PayNote.
```

#### Transaction Identification Failed

```yaml
name: Transaction Identification Failed
type: Conversation/Event
description: >
  PayNote package event recording that identifying the transaction intended to
  be governed by a PayNote failed.
reason:
  type: Text
  description: Failure reason.
```

---

## 11. Legacy Compatibility Types

The following types may exist in older PayNote packages or card-specific packages. New provider-neutral PayNote documents SHOULD prefer the canonical vocabulary above.

### 11.1 Reserve/capture aliases

`Reserve Funds Requested`, `Funds Reserved`, `Reservation Release Requested`, and `Reservation Released` are older or rail-specific forms of the secure/cancel vocabulary.

```yaml
name: Reserve Funds Requested
type: Conversation/Request
description: >
  Legacy PayNote request asking a Guarantor to reserve funds. New provider-
  neutral PayNote documents SHOULD use Secure Funds Requested, because secure
  covers reserve, authorize, hold, and commit across rails.
amount:
  type: Integer
  description: Amount requested to reserve in minor units.
```

```yaml
name: Funds Reserved
type: Conversation/Response
description: >
  Legacy PayNote response confirming funds were reserved. New provider-neutral
  PayNote documents SHOULD use Funds Secured.
amountReserved:
  type: Integer
  description: Amount reserved in minor units.
```

```yaml
name: Reservation Release Requested
type: Conversation/Request
description: >
  Legacy PayNote request asking the Guarantor to release a reservation or hold.
  New provider-neutral PayNote documents SHOULD use Cancel Before Completion
  Requested when releasing value before completion.
amount:
  type: Integer
  description: Amount requested to release in minor units.
```

```yaml
name: Reservation Released
type: Conversation/Response
description: >
  Legacy PayNote response confirming a reservation or hold was released. New
  provider-neutral PayNote documents SHOULD use Payment Cancelled Before
  Completion or a rail-specific release response when more precise.
amountReleased:
  type: Integer
  description: Amount released in minor units.
```

```yaml
name: Reservation Declined
type: Conversation/Response
description: >
  Legacy PayNote response declining a reserve/hold request. New provider-
  neutral PayNote documents SHOULD use Funds Securing Declined.
reason:
  type: Text
  description: Reason for decline.
```

```yaml
name: Reservation Release Declined
type: Conversation/Response
description: >
  Legacy PayNote response declining a reservation release request. New provider-
  neutral PayNote documents SHOULD use Payment Cancellation Declined when the
  release is cancellation before completion.
reason:
  type: Text
  description: Reason for decline.
```

`Capture Funds Requested`, `Funds Captured`, `Capture Declined`, and `Capture Failed` are older or card-specific forms of completion.

```yaml
name: Capture Funds Requested
type: Conversation/Request
description: >
  Legacy or card-specific PayNote request asking the Guarantor to capture funds.
  New provider-neutral PayNote documents SHOULD use Complete Payment Requested.
amount:
  type: Integer
  description: Amount requested to capture in minor units.
```

```yaml
name: Funds Captured
type: Conversation/Response
description: >
  Legacy or card-specific PayNote response confirming captured funds. New
  provider-neutral PayNote documents SHOULD use Payment Completed.
amountCaptured:
  type: Integer
  description: Amount captured in minor units.
```

```yaml
name: Capture Declined
type: Conversation/Response
description: >
  Legacy or card-specific PayNote response declining capture. New provider-
  neutral PayNote documents SHOULD use Payment Completion Declined.
reason:
  type: Text
  description: Reason for decline.
```

```yaml
name: Capture Failed
type: Conversation/Response
description: >
  Legacy or card-specific PayNote response reporting capture failure. New
  provider-neutral PayNote documents SHOULD use Payment Completion Failed.
reason:
  type: Text
  description: Failure reason.
```

### 11.2 Immediate reserve/capture aliases

```yaml
name: Reserve Funds and Capture Immediately Requested
type: Conversation/Request
description: >
  Legacy PayNote request asking the Guarantor to reserve or authorize funds and
  then immediately capture or complete them. New provider-neutral PayNote
  documents SHOULD represent this as Secure Funds Requested followed by Complete
  Payment Requested or as a rail-specific combined operation profile.
amount:
  type: Integer
  description: Amount requested in minor units.
```

### 11.3 Child PayNote aliases

The old child-PayNote model is deprecated. Use triggered independent transactions and linked PayNotes instead.

```yaml
name: Issue Child PayNote Requested
type: Conversation/Request
description: >
  Deprecated PayNote request from the older child-PayNote model. New PayNote
  documents SHOULD NOT use child PayNotes. They should trigger a new independent
  transaction and, if needed, start a linked PayNote for that transaction.
childPayNote:
  description: Deprecated child PayNote payload.
```

```yaml
name: Child PayNote Issued
type: Conversation/Response
description: >
  Deprecated PayNote response from the older child-PayNote model. New PayNote
  documents SHOULD use Linked PayNote Started for a new independent triggered
  transaction.
childPayNote:
  description: Deprecated child PayNote payload.
```

```yaml
name: Child PayNote Issuance Declined
type: Conversation/Response
description: >
  Deprecated PayNote response from the older child-PayNote model. New PayNote
  documents SHOULD use linked transaction or linked PayNote rejection/failure
  responses.
reason:
  type: Text
  description: Reason for decline.
```

### 11.4 Settlement amount aliases

Older packages used settlement amount language. New PayNote documents SHOULD use final amount resolution.

```yaml
name: Settlement Amount Specified
type: Conversation/Response
description: >
  Legacy PayNote response specifying an amount previously called settlement
  amount. New provider-neutral PayNote documents SHOULD use Final Amount
  Resolved, because the concept is the final amount this PayNote should complete
  for, not provider back-office settlement.
finalAmount:
  type: Integer
  description: Final amount in minor units.
```

```yaml
name: Settlement Amount Rejected
type: Conversation/Response
description: >
  Legacy PayNote response rejecting a proposed settlement amount. New provider-
  neutral PayNote documents SHOULD use Final Amount Resolution Rejected.
reason:
  type: Text
  description: Reason for rejection.
```

### 11.5 PayNote approval/cancellation aliases

```yaml
name: PayNote Approved
type: Conversation/Response
description: >
  Legacy PayNote response indicating approval. New provider-neutral PayNote
  documents SHOULD use PayNote Accepted when the Guarantor accepts the PayNote
  as governing a transaction.
```

```yaml
name: PayNote Cancelled
type: Conversation/Response
description: >
  Legacy PayNote response indicating cancellation of the PayNote or its governed
  transaction. New provider-neutral PayNote documents SHOULD use Payment
  Cancelled Before Completion when the transaction is cancelled before
  completion, or another specific lifecycle response when applicable.
```

```yaml
name: PayNote Cancellation Requested
type: Conversation/Request
description: >
  Legacy PayNote request asking to cancel a PayNote. New provider-neutral
  PayNote documents SHOULD use Cancel Before Completion Requested for the
  governed transaction, unless the request is truly about cancelling optional
  PayNote attachment before acceptance.
reason:
  type: Text
  description: Reason for cancellation request.
```

```yaml
name: PayNote Cancellation Rejected
type: Conversation/Response
description: >
  Legacy PayNote response rejecting PayNote cancellation. New provider-neutral
  PayNote documents SHOULD use Payment Cancellation Declined when the requested
  cancellation concerns the governed transaction before completion.
reason:
  type: Text
  description: Reason for rejection.
```

---

## 12. Worked Examples

These examples explain the PayNote model. They intentionally omit workflow implementation bodies, compute steps, rail APIs, and provider adapter details.

### 12.1 Simple direct bank transfer

Alice wants to pay Bob 250.00 USD. The Guarantor initiates and completes the transfer immediately if possible.

```yaml
name: Payment for Invoice Q3-SERVICES
type: PayNote/PayNote
status: Pending
currency: USD
participants:
  payer: Alice
  payee: Bob
  guarantor: Example Bank
amount:
  expectedTotal: 25000
  finalResolved: 25000
  finalAmountResolved: true
  secured: 0
  completed: 0
  reversed: 0
controls:
  completionLocked: false
  reversalLocked: false
  transactionDetailsUpdateLocked: false
payNoteInitialStateDescription:
  summary: Direct payment of 250.00 USD from Alice to Bob.
  details: >
    This PayNote governs a simple bank transfer. The Guarantor should initiate
    and complete the transfer immediately if compliance, balance, and rail rules
    permit.
```

Typical evidence sequence:

```text
PayNote Accepted
Transaction Initiated
Payment Completed
```

### 12.2 Delivery-protected card payment

A customer buys a refrigerator for 1,400.00 USD. The card is authorized now, but capture should happen only after delivery and a seven-day dispute window.

```text
Card authorization (1,400.00 USD)
└── PayNote: capture only after delivery confirmed + 7-day window passes
```

Typical evidence sequence:

```text
PayNote Accepted
Transaction Initiated
Funds Secured
Payment Completion Locked
Delivery Confirmed by carrier or merchant process
Payment Completion Unlocked
Payment Completed
```

If the customer disputes within the window, the PayNote can request cancellation before completion instead of completion.

### 12.3 Deferred recipient selection for an agent

Alice pre-approves 1,000.00 USD. Her agent shops for a merchant and later proposes recipient details. The Guarantor confirms or rejects the update.

```yaml
name: Shopping Budget with Deferred Recipient
type: PayNote/PayNote
status: Pending
currency: USD
participants:
  payer: Alice
  guarantor: Example Bank
amount:
  expectedTotal: 100000
  secured: 0
  completed: 0
  reversed: 0
controls:
  completionLocked: true
  reversalLocked: false
  transactionDetailsUpdateLocked: false
payNoteInitialStateDescription:
  summary: Pre-approved shopping budget where final merchant is selected later.
  details: >
    Alice authorizes an agent to search for a merchant within the budget. Funds
    may be secured before the payee is known. Completion remains locked until
    the Guarantor confirms transaction details and any required human approval.
```

Typical evidence sequence:

```text
PayNote Accepted
Transaction Initiated
Funds Secured
Transaction Details Update Requested
Transaction Details Updated
Payment Completion Unlocked
Complete Payment Requested
Payment Completed
```

### 12.4 Agent travel budget with independent triggered transactions

Alice secures 2,000.00 USD for a travel agent. The PayNote governs the top-level budget transaction. As choices are made, separate transactions are triggered.

```text
Card auth / account hold (2,000.00 USD) — agent budget
└── PayNote A: agent may trigger approved travel transactions within budget
    ├── triggers: Card auth (400.00 USD) — flight
    │   └── PayNote B: capture immediately if ticket issued; refund within 24h
    ├── triggers: Card auth (500.00 USD) — hotel
    │   └── PayNote C: capture at check-in; cancel if flight cancelled
    ├── triggers: Card auth (80.00 USD) — taxi
    │   └── PayNote D: capture actual fare
    └── triggers: Card auth (60.00 USD) — restaurant
        └── optional PayNote
```

Each arrow is a separate transaction. Each PayNote governs exactly one transaction.

### 12.5 Marketplace split

A marketplace purchase authorizes 100.00 USD. The PayNote requests a 15.00 USD platform amount immediately and an 85.00 USD seller amount after delivery.

```text
Card authorization (100.00 USD)
└── PayNote: marketplace split
    ├── immediately triggers/completes platform transaction (15.00 USD)
    └── delivery confirmed triggers/completes seller transaction (85.00 USD)
```

The platform and seller transactions are independent and may have their own PayNotes.

### 12.6 Final amount resolution

A courier quote starts at 15.00 USD, but actual mileage determines final amount.

```yaml
name: Mileage-Based Courier Payment
type: PayNote/PayNote
status: Pending
currency: USD
amount:
  expectedTotal: 1500
  finalResolved: 0
  finalAmountResolved: false
  secured: 0
  completed: 0
  reversed: 0
payNoteInitialStateDescription:
  summary: Courier payment with final amount resolved after mileage is known.
  details: >
    The expected amount is an estimate. Completion should use Final Amount
    Resolved once the Guarantor accepts actual mileage and final amount.
```

Typical evidence sequence:

```text
PayNote Accepted
Transaction Initiated
Funds Secured
Final Amount Resolution Requested
Final Amount Resolved
Complete Payment Requested
Payment Completed
```

---

## 13. Design Guidance

### 13.1 Keep the base PayNote provider-neutral

The base PayNote should not hard-code card-only concepts like capture, void, refund, or auth. Use secure, complete, cancel before completion, and reverse after completion in provider-neutral sources. Rail-specific packages can define aliases and stronger detail types.

### 13.2 Do not use fixed values for mutable state in base types

A Blue type fixed value is inherited as an invariant. Therefore the base PayNote type SHOULD NOT define `status.value: Pending`, `amount.completed.value: 0`, or `controls.completionLocked.value: false`. Those are mutable instance values. They belong in PayNote instances, initialization profiles, or implementation-specific runtime logic, not as fixed values on the canonical base type.

Safe fixed values are stable discriminators such as `kind: PayNote` on the base PayNote type.

### 13.3 Separate process truth from rail truth

A participant request is process intent. A Guarantor response is rail evidence. PayNote documents SHOULD avoid treating participant requests as proof that funds moved or were secured.

### 13.4 Prefer independent triggered transactions over child PayNotes

Do not model a complex commercial flow as a tree of child PayNotes. Model each money movement as an independent transaction, and attach one PayNote to each transaction that needs process governance.

### 13.5 Make human-readable descriptions strong

A PayNote's `payNoteInitialStateDescription` should be good enough for a participant, auditor, bank operator, or agent to understand the process without reading hidden code. The best PayNotes are inspectable commitments.

---

## 14. Summary

A PayNote is a document attached to one transaction that governs what should happen with that transaction.

Its canonical concepts are:

- accept or reject the PayNote;
- initiate the underlying transaction;
- secure funds;
- complete payment;
- cancel before completion;
- reverse after completion;
- resolve the final amount;
- assign payee or update transaction details;
- lock and unlock sensitive transitions;
- trigger independent new transactions when needed.

This makes PayNote simpler than the older child-PayNote model, more general than card-only terminology, and better suited to agentic, multi-party, cross-rail business processes.

*End of Blue PayNote Specification 1.0.*
