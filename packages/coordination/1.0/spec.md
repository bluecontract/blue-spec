# Blue Timelines and Conversation Specification 1.0

> **Status.** Draft specification for the Blue Timelines and Conversation package.
>
> **Positioning.** Blue Timelines provide the external ordering, attribution, and completeness layer for Blue document processing. The Conversation package provides standard Blue types for timeline-delivered events, operations, actor attribution, user/action requests, and deterministic workflow handlers.
>
> **Scope.** This document defines Timeline concepts, Timeline Provider responsibilities, completeness guarantees, conversation event delivery, operation invocation, target-document concurrency rules, actor/source attribution, workflow handler semantics, and a canonical type catalog for the Conversation package. It depends on the Blue Language Specification, the Blue Contracts and Processor Specification, and Blue BEX. It does not redefine BlueId, type resolution, canonicalization, patch semantics, processor lifecycle, gas accounting, or the full BEX language.

## Conventions

The key words **MUST**, **MUST NOT**, **REQUIRED**, **SHOULD**, **SHOULD NOT**, **MAY**, and **OPTIONAL** are to be interpreted as normative requirement levels.

Sections marked **normative** define required behavior for conforming Blue Timelines and Conversation 1.0 implementations. Sections marked **informative** explain intent, examples, or implementation guidance.

Canonical source nodes in this document are intended to be identity-bearing Blue content. As in the Blue Language core registry, the `description` fields of canonical Conversation nodes define semantics and affect BlueId. Editing a canonical `description` is a type-identity change, not a documentation-only edit.

---

## 0. Overview

Blue document processing is deterministic once the processor receives one document state and one event. Timelines answer the question that the core processor intentionally leaves outside its scope: **which external events should be delivered, in what order, and when is that order complete enough to trust?**

A Timeline is an append-only, hash-linked sequence of entries maintained by a Timeline Provider. A Timeline Entry is a Blue event envelope containing the timeline, the previous entry reference, timestamp, message payload, actor attribution, and source attribution. Conversation documents receive Timeline Entries through Timeline Channels.

A Conversation document exposes actions through `Conversation/Operation` contracts. Callers invoke those operations by posting `Conversation/Operation Request` messages to an authorized Timeline. A Sequential Workflow or Sequential Workflow Operation handles matching events and produces patch and event effects through the Blue Contracts processor.

The combined model is:

```text
Timeline Provider(s)
  -> append and attest Timeline Entries
  -> issue completeness guarantees
  -> feeder collects complete window
  -> feeder sorts entries deterministically
  -> PROCESS(document, timelineEntry) for each entry
  -> Conversation Timeline Channel accepts matching entry
  -> Conversation Workflow handles entry.message
  -> processor applies patches/events/checkpoints deterministically
```

The result is a verifiable business conversation: actions are attributed to actors, ordered by Timeline completeness, constrained by document contracts, and applied only through deterministic Blue processor rules.

---

## 1. Dependency Boundaries

### 1.1 Blue Language boundary

Conversation source nodes are Blue content. Their `name`, `description`, fields, and nested contracts participate in Blue Language identity. The language implementation parses, preprocesses, resolves, canonicalizes, and hashes them. It does not execute them.

### 1.2 Blue Contracts boundary

The Conversation package defines concrete Channel, Handler, Marker, and event types used by a Blue Contracts processor. It does not replace the core processor rules. Patches, cascades, embedded scopes, checkpoints, termination, gas accounting, and post-patch type soundness are governed by the Blue Contracts and Processor Specification.

### 1.3 BEX boundary

`Conversation/Compute` embeds Blue BEX programs as workflow steps. BEX computes values, changesets, and events. BEX does not mutate the document by itself. A workflow must use `Conversation/Update Document` to apply computed changesets and `Conversation/Trigger Event` or `emitEvents` semantics to emit events.

### 1.4 Feeder boundary

The Timeline Feeder is outside the core processor. The feeder queries Timeline Providers, obtains completeness guarantees, chooses a safe processing window, sorts Timeline Entries, and invokes the core processor with one Timeline Entry at a time. The feeder MUST preserve the observable deterministic behavior defined here.

---

## 2. Timeline Concepts

### 2.1 Timeline

A **Timeline** is the ordered, append-only record of one actor, account, organization, service, agent, device, blockchain account, or other perspective. A Timeline is not a global ledger. It is one verified perspective that can be composed with other verified perspectives.

A Timeline Provider MUST maintain the Timeline so that:

- entries are append-only;
- each non-first entry references the previous entry;
- timestamps are monotonic under the provider's guarantee rules;
- entries are queryable by timestamp window and by cursor;
- the provider can attest completeness for a requested timestamp boundary;
- actor and source attribution are supplied for every entry accepted by the provider.

### 2.2 Timeline Entry

A **Timeline Entry** is the event envelope delivered to Conversation Timeline Channels. The `message` field is the domain payload. Conversation processors normally match handlers against `entry.message` while preserving the full entry for actor/source/persistence decisions.

The `prevEntry` field forms a hash chain. A provider or verifier MUST reject a Timeline Entry whose `prevEntry` does not match the preceding accepted entry for that Timeline, except for the first entry where `prevEntry` is absent.

### 2.3 Timeline Provider

A **Timeline Provider** is a trust anchor for a Timeline. It stores entries, verifies identity, attributes actions to actors, records source information, assigns timestamps, and issues completeness guarantees. A provider may be a commercial service, bank, government identity service, internal enterprise system, or blockchain-derived provider.

Provider trust is explicit. BlueId verification proves content integrity. It does not prove that the provider correctly attributed the actor, respected revocation, or honored completeness. Those are provider responsibilities and business/legal trust assumptions.

### 2.4 Completeness Guarantee

The normative completeness guarantee is a half-open timestamp boundary:

```text
No entries exist in this Timeline with timestamp < T,
and any future entry accepted into this Timeline will have timestamp >= T.
```

A processor may safely close the window below `T`. Portable processing SHOULD use half-open windows:

```text
process entries with lastWatermark <= timestamp < nextWatermark
```

This avoids ambiguity for entries whose timestamp equals the guarantee boundary. To process a triggering entry with timestamp `S`, a feeder SHOULD request a boundary strictly greater than `S`, for example `S + 1` for integer microsecond timestamps, or a provider-defined next boundary that includes all entries at `S`.

A provider MUST NOT later accept or reveal an entry with timestamp below a boundary it has guaranteed. If an append request arrives late, the provider MUST either reject it or assign it a timestamp greater than or equal to every outstanding guarantee boundary it has issued for that Timeline.

### 2.5 Ordering across multiple Timelines

For a Conversation document with multiple Timeline Channels, the feeder MUST choose a processing boundary that is complete across all relevant timelines. The feeder collects all entries in the safe half-open window and sorts them deterministically.

The default portable ordering key is:

```text
timestamp ascending,
channel key ascending by Unicode code point,
timelineId ascending by Unicode code point,
sequence ascending when present,
entry BlueId ascending as final tie-breaker
```

A blockchain timeline profile MAY insert chain-specific tie-breakers, such as block number and transaction index, before the entry BlueId tie-breaker when those values are part of the verified entry content or proof.

### 2.6 Conversation delivery model

The core processor still receives one event per invocation. A feeder that has collected a batch of Timeline Entries MUST invoke the processor once per entry in the deterministic order above, carrying the updated document from one invocation to the next.

A conforming Timeline Feeder MUST NOT skip an entry merely because no handler is expected to match it. The core processor and channel checkpoint rules decide local acceptance and idempotency.

### 2.7 Blockchain timelines

A blockchain can be used as Timeline storage only for finalized history. A blockchain-based Timeline Provider or feeder MUST expose a safe completeness boundary based on finalized blocks or equivalent finality. When a document includes a slow-finality timeline, the safe processing boundary for the entire conversation is limited by the slowest required timeline.

---

## 3. Conversation Processing Profile

### 3.1 External channel restriction

A Blue Conversation 1.0 document SHOULD use Timeline Channels as its only external user/application event channels. Processor-managed channels from the Blue Contracts core remain available: Document Update, Triggered Event, Lifecycle Event, and Embedded Node.

A `Conversation/Composite Timeline Channel` may compose several same-scope Timeline Channels into a single union channel, but it is still a Timeline-derived external channel.

### 3.2 Timeline Channel delivery

A `Conversation/Timeline Channel` accepts a Timeline Entry when the entry belongs to the configured timeline. The channelized payload is the full Timeline Entry. Handler event filters and workflow matchers SHOULD match against `message`, `actor`, `source`, or other fields of that full entry.

### 3.3 Checkpoints

Timeline Channels use the Blue Contracts external-channel checkpoint mechanism. The default checkpoint subject is the accepted Timeline Entry. A Timeline profile MAY use a `Timeline Cursor` as the checkpoint subject, but it MUST preserve idempotency and deterministic newness. A stale or duplicate entry MUST NOT cause handler execution.

### 3.4 Operation model

A Conversation document exposes callable behavior with `Conversation/Operation` contracts. An Operation names the channel on which callers invoke it and defines the request payload shape.

An Operation is not a handler. It is a marker describing a supported action. A handler such as `Conversation/Sequential Workflow Operation` implements that operation.

### 3.5 Operation Request event

A caller invokes an operation by posting a Timeline Entry whose `message` is `Conversation/Operation Request`. The request identifies:

- the operation name;
- the request payload;
- the target document snapshot/version;
- whether a newer descendant version may be used.

The `document` field is not advisory. It is the target identity constraint for the invocation.

### 3.6 Target document and `allowNewerVersion`

Let `D_expected` be the Content BlueId identified by `Operation Request.document`. The target resolution layer MUST first identify the document session, conversation, or stored document lineage to which the request is addressed. It then applies this rule:

- If `allowNewerVersion` is absent or false, the current target document's Content BlueId MUST equal `D_expected` before the operation is processed.
- If `allowNewerVersion` is true, the runtime MAY process the request on the latest committed version of the same target document, but only if it can prove that `D_expected` was a previous committed state in that document's lineage.

`allowNewerVersion: true` does **not** permit execution on an unrelated latest document. It only relaxes exact-version equality to lineage inclusion. If the runtime cannot prove the target lineage, it MUST reject the operation before contract processing and MUST NOT mutate the document.

A target document reference may be a pure BlueId reference or a complete Blue document. If a complete document is supplied, its Content BlueId is `D_expected`.

### 3.7 Workflow operation binding

`Conversation/Sequential Workflow Operation` implements an Operation named by its `operation` field. Its effective handler channel is the referenced Operation's `channel` field. If the workflow instance also provides an explicit `channel`, it MUST equal the referenced Operation's channel. If the referenced Operation does not exist, has an invalid channel, or the effective channel cannot be derived deterministically, the handler is invalid.

This is a Conversation profile recognition rule for this concrete handler type. Plain `Conversation/Sequential Workflow` handlers still bind through their own `channel` field under the Blue Contracts core rules.

### 3.8 Workflow steps

A Sequential Workflow executes its steps in list order. Each step receives an immutable context including:

- the full Timeline Entry payload as `event`;
- `event.message` as the primary domain payload;
- the current selected document view, read-only;
- current handler contract content;
- previous named step results;
- deterministic compute/binding helpers supported by the step type.

Step effects are accumulated and returned to the core processor as a normal Contract Execution Result. The workflow MUST NOT mutate the document except by returning patches to the processor.

---

## 4. Canonical Source Node Rules

The YAML nodes in §5 are intended registry source nodes. Descriptions are deliberately semantic and identity-bearing. A registry release SHOULD publish each node's exact source, preprocessed/canonical form, calculated BlueId, specification version, and fixture package identity.

Names such as `Text`, `Integer`, `Boolean`, `List`, `Dictionary`, `Core/Channel`, `Core/Handler`, `Core/Marker`, and `Core/Json Patch Entry` are aliases used for readability. A canonical registry release MUST resolve them through the declared Blue Language and Blue Contracts preprocessing environments.

---

## 5. Conversation and Timeline Type Catalog

### 5.1 Timeline and provider types

#### Timeline

```yaml
name: Timeline
description: >
  Conversation package type representing an append-only, provider-maintained
  log of Timeline Entry nodes. A Timeline is the verified perspective of one
  owner or authority: a person, organization, service, AI agent, blockchain
  account, government identity, bank account, enterprise system, or other
  entity that can make statements or take actions. It should be read as a
  personal or institutional ledger rather than as a global ledger. Alice may
  have one Timeline, Bob may have another Timeline, a bank may have another,
  and a Blue document can receive entries from all of them through Timeline
  Channels.

  The fundamental purpose of a Timeline is to make independent document
  processors reach the same result without coordinating with each other. Each
  processor observes entries from the Timelines configured in a document,
  obtains provider completeness guarantees for the relevant timestamp window,
  sorts the complete set of entries deterministically, and then applies them to
  the document in that order. The Timeline does not claim that there is one
  universal order for all events in the world. Instead, it provides one
  authenticated perspective that can be merged with other authenticated
  perspectives by the rules in this package.

  Entries in a Timeline are append-only: new entries are added after existing
  entries and historical entries are not rewritten. Each non-first entry links
  to the previous entry through prevEntry, normally as a BlueId reference. Since
  the BlueId of an entry is calculated from its content, including the previous
  entry reference, this creates a tamper-evident hash chain. If an older entry
  is modified, its BlueId changes and every later prevEntry link that depended
  on it becomes invalid.

  A Timeline Provider is responsible for the operational facts that BlueId
  hashing alone cannot prove: which identity owns the Timeline, which actor
  performed the action, which source submitted it, what timestamp was assigned,
  and which completeness guarantees have been issued. Managed providers can
  provide near-real-time guarantees; blockchain-backed providers can provide
  guarantees only for finalized history. In both cases, the Timeline is useful
  because processors can ask where the safe processing boundary is and avoid
  applying entries that might later be reordered by newly discovered earlier
  entries.

  The timelineId is stable within the provider namespace and is the primary
  value used by Timeline Channel matching unless a concrete provider subtype
  defines stronger matching fields such as account, tenant, chain, contract,
  owner, or finality profile.
timelineId:
  type: Text
  description: >
    Stable provider-scoped identifier of this Timeline. The identifier is the
    value most Timeline Channels use to decide whether an incoming Timeline
    Entry belongs to the channel. It is application-defined, but it MUST remain
    stable for the lifetime of the Timeline so checkpoints, cursors, audit
    trails, and provider guarantees continue to refer to the same append-only
    sequence.
```

#### Timeline Entry

```yaml
name: Timeline Entry
description: >
  Conversation package event envelope representing one immutable fact recorded
  in a Timeline. A Timeline Entry is the unit that document processors receive,
  order, checkpoint, audit, and pass into handlers. The entry belongs to exactly
  one Timeline, references the previous entry in that Timeline, carries a
  provider-assigned timestamp, identifies what happened in message, and records
  provider-asserted actor and source attribution.

  The message field is the domain event, such as Set Price, Confirm Shipment,
  Operation Request, Chat Message, payment response, consent grant, or any
  application-specific Blue node. The surrounding Timeline Entry is the
  coordination envelope. It tells processors whose Timeline the message came
  from, where the entry sits in that Timeline's hash chain, when the provider
  placed it, who or what performed the action, and how the request reached the
  provider. A handler will often match event.message, but the channel payload is
  the full Timeline Entry because actor policy, source policy, deterministic
  ordering, audit, and idempotency need the envelope fields too.

  The prevEntry field creates the tamper-evident chain. The first entry in a
  Timeline has no prevEntry. Every later entry points to the immediately
  previous accepted entry, normally by BlueId. Because a BlueId is a content
  identity, changing an old entry changes its BlueId and invalidates the later
  entry that referenced it. This gives verifiers a simple way to detect
  rewritten history without requiring global consensus.

  The timestamp enables deterministic ordering across multiple independent
  Timelines. When a processor learns about a new entry from one Timeline, it
  cannot immediately assume that entry is next for the document. Another
  participant may have posted an earlier entry on a different Timeline that the
  processor has not seen yet. The feeder therefore asks every other relevant
  Timeline Provider for a completeness guarantee up to the processing boundary.
  Only after all relevant providers have either returned older entries or
  guaranteed that no older entries exist may the feeder sort and deliver the
  entries. This is what lets two processors observing the same Timelines reach
  the same document state without talking to each other.

  Timeline Entry ordering is deterministic. Entries are primarily ordered by
  timestamp. If timestamps collide, processors use the package-defined
  tie-breakers, such as channel key and timeline identity, and provider profiles
  may add append-order proofs such as sequence, block number, or transaction
  index. Blockchain-backed Timelines commonly have timestamp collisions because
  block timestamps have coarse precision; such profiles MUST expose enough
  finalized ordering data for all processors to make the same choice.
timeline:
  type: Conversation/Timeline
  description: >
    The Timeline this entry belongs to. This value is not merely metadata; it is
    the address of the perspective from which the entry was asserted and the
    place a feeder queries for adjacent entries, hash-chain validation, and
    completeness guarantees. A provider-specific subtype MAY include account,
    tenant, chain, contract, owner, or finality information. Timeline Channel
    matching compares this value according to the channel's concrete provider
    semantics.
prevEntry:
  description: >
    Reference to the previous Timeline Entry in the same Timeline. The first
    entry omits prevEntry. Non-first entries SHOULD use a pure BlueId reference
    to the immediately preceding accepted entry. This field is what turns a list
    of events into a hash chain: each entry's BlueId commits to its own content
    and to the identity of the entry before it. A Timeline Provider or verifier
    MUST reject an entry whose prevEntry does not match the immediately
    preceding accepted entry for the Timeline, because accepting it would create
    a fork or silently rewrite history.
sequence:
  type: Integer
  description: >
    Optional monotonically increasing provider-assigned entry number within the
    Timeline. Timestamp is the portable cross-timeline ordering key, but some
    providers need an additional append-order proof for entries that share the
    same timestamp. When present, sequence is used as an ordering tie-breaker
    after the package-level keys such as timestamp, channel key, and timelineId.
    Providers SHOULD expose sequence or an equivalent deterministic ordering
    proof. Blockchain profiles may use block number and transaction index
    instead.
timestamp:
  type: Integer
  description: >
    Provider-assigned timestamp in microseconds since Unix epoch
    (1970-01-01T00:00:00Z). The timestamp establishes temporal order both
    within one Timeline and across the set of Timelines used by a document.
    Timestamps in a single Timeline MUST be monotonic under the provider's
    completeness rules. A managed provider should assign timestamps that reflect
    physical time as accurately as its profile promises, but the decisive rule
    is the binding guarantee: once the provider has guaranteed completeness for
    boundary T, it MUST NOT later accept or reveal an entry with timestamp less
    than T. It may reject a late append request or assign it a timestamp greater
    than or equal to the outstanding boundary.

    For deterministic document processing, a timestamp alone is not enough. A
    processor must also know that all relevant Timelines are complete below the
    chosen boundary. Without that knowledge, an older unseen event from another
    Timeline could arrive later and force a different state transition order.
    Timeline Completeness Guarantee closes that gap.
message:
  description: >
    Domain payload of this Timeline Entry: the action, statement, observation,
    or request being recorded. The payload may be any valid Blue node, such as
    Chat Message, Operation Request, Status Change, a PayNote response, or an
    application-specific event. Handlers commonly match and process
    event.message, but message content MUST NOT be treated as a trusted source
    of timeline identity, timestamp, actor attribution, or source attribution.
    Those facts come from the provider-asserted envelope fields.
actor:
  type: Conversation/Actor
  description: >
    Actor attribution asserted by the Timeline Provider for the action recorded
    by this entry. The actor says who or what performed the action: a principal
    acting directly, an AI or automated agent acting under delegated authority,
    or a provider-specific subtype with richer audit details. Actor attribution
    is the foundation for policies that distinguish human approval from
    automation. It MUST be assigned from authenticated provider knowledge and
    MUST NOT be self-asserted by untrusted message content.
source:
  type: Conversation/Source
  description: >
    Optional source attribution describing how the request reached the Timeline
    Provider, such as Browser Session, API Call, or Document Request. Source
    complements actor: actor identifies who or what exercised authority, while
    source identifies the channel or mechanism through which the provider
    received the request. Source is policy-relevant and SHOULD be
    provider-asserted whenever a document may care whether an action came from
    an interactive session, a programmatic integration, or another document.
```

#### Timeline Provider

```yaml
name: Timeline Provider
description: >
  Conversation package type describing the trust anchor that maintains a
  Timeline and makes it usable for deterministic multi-party processing. A
  Timeline Provider stores entries, verifies the Timeline owner, accepts append
  requests, assigns timestamps, records actor and source attribution, exposes
  queries for entries and cursors, and issues Timeline Completeness Guarantee
  attestations.

  The provider is trusted for facts that are outside pure content hashing.
  BlueId verification can show that an entry's bytes have not changed and that
  prevEntry forms a valid hash chain, but it cannot prove who authenticated to
  the service, whether an action was performed by a person or by an agent,
  whether an agent still had a valid delegation, whether a timestamp was
  assigned under the provider's clock rules, or whether a completeness guarantee
  has been honored. Those are provider responsibilities.

  A managed provider, such as a commercial platform, bank, government identity
  service, or enterprise system, MUST preserve append-only history and MUST
  never violate an outstanding completeness guarantee. If it has guaranteed that
  no entries exist before boundary T, any later append that would otherwise have
  received an earlier timestamp must be rejected or assigned a timestamp greater
  than or equal to T. This binding promise is what lets processors close the
  window on past events and process a document without waiting for all parties
  to coordinate directly.

  A blockchain-derived provider or feeder plays the same logical role but can
  only provide guarantees for finalized history. It derives entries from chain
  state, derives timestamps from finalized blocks or equivalent consensus data,
  and exposes a safe boundary based on the finality profile. When such a
  Timeline participates in a document, every other Timeline in that document is
  limited by the slowest finality boundary for safe deterministic processing.
providerId:
  type: Text
  description: >
    Stable identifier of the provider namespace. Provider IDs are used in audit
    trails, profile selection, and Timeline addressing. They SHOULD identify the
    institution, service, chain profile, or provider class whose guarantees and
    attribution semantics a processor is relying on.
assuranceLevel:
  type: Text
  description: >
    Optional human-readable or profile-defined assurance category. Examples
    include email-verified, enterprise-verified, bank-KYC,
    government-verified, hardware-backed, or blockchain-finalized. The value
    communicates how strongly the provider verifies identity and attribution; it
    does not replace the concrete provider profile or the cryptographic or
    institutional proof attached to entries and guarantees.
```

#### Timeline Completeness Guarantee

```yaml
name: Timeline Completeness Guarantee
description: >
  Provider attestation that closes a half-open timestamp window for one
  Timeline. A guarantee with timestamp T is a binding statement that no entries
  exist in the Timeline with timestamp less than T and that every future entry
  accepted into that Timeline will have timestamp greater than or equal to T.
  It is a promise about both the past and the future.

  This type is the mechanism that solves the determinism problem for documents
  with multiple Timeline Channels. If Alice posts an entry at timestamp A, a
  processor must know whether Bob, a bank, or another participant has an entry
  with timestamp less than A before applying Alice's entry. The feeder queries
  every other relevant Timeline Provider for a guarantee at the chosen boundary.
  Providers return any older entries that must be processed first or attest that
  no such entries exist. Once every relevant Timeline is covered, the feeder can
  sort the collected entries and deliver them to the processor in a final order.

  The guarantee is half-open: entries with timestamp less than T are complete;
  entries at timestamp T are not covered by this particular guarantee. This
  avoids ambiguity at the boundary and lets profiles use deterministic
  tie-breakers for equal timestamps under a later safe window.

  Blockchain-backed Timelines satisfy this concept only for finalized history.
  Their proof normally identifies the finalized block, block hash,
  confirmations or finality proof, and the entries read from the Timeline smart
  contract up to that block. They cannot safely guarantee the recent present
  because pending transactions and non-final blocks may still change.

  This type is an attested provider response; it is not itself a processor
  channel and does not cause handler execution.
guarantee:
  type: Text
  description: >
    Required guarantee kind. For this specification the portable value is
    no-entries-before, meaning that the provider attests there are no entries in
    the identified Timeline with timestamp less than the timestamp field and
    that future accepted entries will not be assigned an earlier timestamp.
  schema:
    required: true
    enum: [no-entries-before]
timeline:
  type: Conversation/Timeline
  description: >
    Timeline to which the guarantee applies. The guarantee is scoped to this
    exact Timeline identity and provider profile. It MUST NOT be reused for a
    different account, tenant, chain, contract, owner, or provider namespace,
    even if those Timelines are controlled by the same organization.
timestamp:
  type: Integer
  description: >
    Half-open completeness boundary T, expressed in microseconds since Unix
    epoch. Entries with timestamp less than T are complete under this guarantee.
    Entries with timestamp equal to T are outside this window and require a
    later boundary to process safely. Feeders choose safe processing windows by
    obtaining compatible boundaries from every Timeline Channel relevant to the
    document and then processing only entries covered by all of them.
entries:
  type: List
  itemType: Conversation/Timeline Entry
  description: >
    Optional list of entries below the boundary returned together with the
    guarantee. If a provider discovers entries that the feeder had not yet seen,
    it returns them here so they can be collected, sorted with the triggering
    entry and other discovered entries, and processed in deterministic order.
    If present, every returned entry MUST belong to timeline and have timestamp
    less than timestamp.
proof:
  description: >
    Provider-specific cryptographic or institutional proof of the guarantee,
    such as a signed attestation, certificate chain, audit proof, bank or
    enterprise audit record, or blockchain finality proof. The proof format is
    defined by the provider profile. Consumers rely on this proof to audit that
    the provider made the binding statement at the boundary they processed.
```

#### Timeline Cursor

```yaml
name: Timeline Cursor
description: >
  Conversation package checkpoint subject representing how far a Timeline has
  been consumed by a channel, feeder, or provider profile. A cursor records the
  last processed entry and the completeness watermark used to decide future
  queries. It is the durable memory that prevents duplicate handler execution
  and allows processing to resume after interruption without reordering or
  skipping entries.

  A cursor is tied to one Timeline identity and one processing context. It MUST
  NOT be used to skip entries from a different Timeline, different provider
  namespace, or different channel semantics. The cursor can be stored as a Blue
  Contracts Channel Event Checkpoint subject for timeline-aware channels or
  maintained by an external feeder that invokes the processor with one complete
  Timeline Entry at a time.

  The watermark is a completeness boundary, not merely the timestamp of the
  last event seen. Entries below the watermark are known to be complete under
  provider guarantees and have either been processed or intentionally ignored by
  channel semantics. Entries at or above the watermark may still need later
  queries before they are safe to process.
timeline:
  type: Conversation/Timeline
  description: >
    Timeline governed by this cursor. The cursor's lastEntry and watermark are
    meaningful only for this Timeline identity and provider profile.
lastEntry:
  description: >
    Optional reference to the last Timeline Entry processed for this Timeline
    and channel. SHOULD be a pure BlueId reference when known. The reference
    helps a feeder resume from a concrete hash-chain position and detect
    provider or storage inconsistencies.
watermark:
  type: Integer
  description: >
    Half-open completeness boundary up to which this channel or feeder has
    processed entries. Entries with timestamp less than watermark are complete
    and either processed or intentionally ignored by channel semantics. A feeder
    MUST NOT advance this value beyond the boundary covered by every relevant
    Timeline Provider guarantee for the document processing window.
```

### 5.2 Actor, source, and policy types

#### Actor

```yaml
name: Actor
description: >
  Conversation package base type for provider-asserted attribution of who or
  what performed a Timeline Entry action. Actor is policy-relevant attribution,
  not executable behavior and not a self-description supplied by the message
  payload. A Timeline Provider MUST assign actor values from authenticated
  provider knowledge: login sessions, API credentials, delegated agent
  sessions, enterprise identity, blockchain account control, or another
  provider-profile mechanism.

  Actor attribution is the foundation for trusting automation in Blue
  documents. A processor that receives a Timeline Entry needs to know whether
  the action was performed directly by the principal or by an AI agent or other
  automated system acting under delegated authority. Documents can then enforce
  Actor Policy: routine operations may allow any actor, while critical
  operations such as authorizing funds, canceling a contract, or granting
  consent may require a Principal Actor.

  Principal Actor and Agent Actor are the required portable categories.
  Providers SHOULD specialize them with implementation details when those
  details are useful for audit or policy, such as UI session strength, API key
  identifier, agent document reference, delegation grant reference, principal
  account, or revocation state. Processors classify every subtype of Principal
  Actor as principal and every subtype of Agent Actor as agent unless a stricter
  policy requires a specific subtype.
```

#### Principal Actor

```yaml
name: Principal Actor
type: Conversation/Actor
description: >
  Base actor category for a human principal or account owner exercising their
  own authority. The action may have been submitted through an interactive user
  interface, a strongly authenticated browser session, a device-bound
  credential, or a programmatic credential that the provider treats as being
  under the principal's direct control. The important semantic claim is that the
  provider attributes the action to the principal, not to a delegated agent.

  Provider-specific subtypes may record UI session details, authentication
  method, multi-factor or biometric strength, session identifier, API key ID,
  hardware-backed credential, risk signals, or other audit evidence. Actor
  Policy treats every subtype of Principal Actor as a principal action unless a
  stricter policy requires a specific subtype, for example a rule requiring a
  strongly authenticated UI Principal Actor for payment authorization.
```

#### Agent Actor

```yaml
name: Agent Actor
type: Conversation/Actor
description: >
  Base actor category for a non-human or automated agent exercising delegated
  authority. This includes AI agents, workflow agents, service integrations, or
  other automation that acts on behalf of a Timeline owner. The provider is
  asserting that the entry was not a direct principal action; it was performed
  by an agent whose authority must come from a delegation, policy, credential,
  or other provider-recognized grant.

  A provider-specific subtype SHOULD identify the agent, the principal on whose
  behalf it acted, and the delegation grant or permission document that made the
  action valid. This gives observers an audit path: confirm the entry was made
  by an agent, inspect the agent definition or session, inspect the delegation
  scope, and verify that the requested operation was inside that scope when the
  entry was accepted.

  Actor Policy treats every subtype of Agent Actor as an agent action unless a
  stricter policy requires a specific subtype. This distinction is what lets a
  Blue document grant bounded autonomy to agents while reserving high-risk
  operations for direct principal approval.
onBehalfOf:
  type: Conversation/Actor
  description: >
    Actor or principal whose delegated authority this agent is using. A
    provider-specific subtype MAY replace or refine this with account IDs,
    principal identifiers, delegation references, agent-session documents, or
    permission-grant Blue documents. This field should make the chain of
    authority auditable from principal to delegation to agent action.
```

#### Source

```yaml
name: Source
description: >
  Conversation package base type for provider-asserted source attribution. A
  Source describes how a Timeline Entry reached the Timeline Provider, such as
  through an authenticated browser session, an API call, a document-generated
  request, an enterprise integration, or a blockchain transaction. Actor and
  Source answer different questions: actor says whose authority was exercised;
  source says through which mechanism the provider received the action.

  Source is policy-relevant and SHOULD be supplied by the provider whenever the
  distinction can affect authorization or audit. A document may allow a
  principal to add notes through an API key, but require an interactive browser
  session for authorizing funds. Another document may accept a document request
  as evidence of a workflow handoff while still requiring the target document's
  own Operation and Actor Policy to authorize state changes.
```

#### Browser Session

```yaml
name: Browser Session
type: Conversation/Source
description: >
  Source subtype describing an authenticated interactive browser or user
  interface session. This source is used when the provider accepted the action
  through a session where the principal or operator was present in a UI flow.
  Provider-specific subtypes SHOULD include authentication strength, session
  reference, multi-factor or biometric evidence, device binding, and risk
  signals when those details are intended to be policy-relevant. Actor Policy
  can use this to require direct human presence for sensitive operations.
uiSessionNonce:
  type: Text
  description: Opaque provider-issued nonce identifying the UI session.
```

#### API Call

```yaml
name: API Call
type: Conversation/Source
description: >
  Source subtype describing a programmatic API request accepted by the Timeline
  Provider. API Call identifies the submission mechanism, not by itself the
  authority behind the action. The actor field still determines whether the
  provider attributed the action to a Principal Actor, an Agent Actor, or a
  provider-specific subtype. Documents that require human presence SHOULD NOT
  treat API Call as equivalent to an interactive browser session unless their
  policy explicitly allows that provider-specific credential model.
apiKeyId:
  type: Text
  description: Identifier of the API credential used for the request.
```

#### Document Request

```yaml
name: Document Request
type: Conversation/Source
description: >
  Source subtype describing a request emitted by another Blue document
  workflow. It links the Timeline Entry to the document and document version
  that caused the request, so observers can trace why the entry was posted and
  which workflow produced it. This is useful for multi-document coordination,
  follow-up requests, customer-action workflows, and auditable handoffs between
  documents.

  Processors MUST treat Document Request as attribution and evidence, not as
  authority to bypass the target document's own Operation and Actor Policy. The
  target document still decides whether the operation exists, whether the
  request shape is valid, whether the actor is allowed, and whether the target
  document version check succeeds.
documentId:
  type: Text
  description: Identifier or BlueId of the document that triggered the request.
epoch:
  type: Integer
  description: Optional committed document epoch or version number at emission time.
```

#### Actor Policy

```yaml
name: Actor Policy
type: Core/Marker
description: >
  Conversation marker that restricts which actor and source categories may
  invoke named operations. Actor Policy is evaluated against the Timeline
  Entry's provider-asserted actor and source fields before an operation handler
  applies state changes. If the required actor or source category is not
  satisfied, the operation delivery is rejected or produces a deterministic
  policy failure according to the Conversation runtime profile.

  Actor Policy is how Blue documents turn Timeline Provider attribution into
  enforceable automation boundaries. A document can allow agents to perform
  low-risk operations such as adding notes, requesting information, or preparing
  drafts while requiring a Principal Actor for operations such as authorizing
  funds, capturing funds, canceling a contract, granting consent, or changing
  delegation scope. This avoids two unsafe extremes: giving agents shared
  credentials that erase attribution, and requiring human approval for every
  low-risk automated action.

  The portable policy categories are intentionally simple: principal, agent, or
  any for actors, and browserSession, apiCall, documentRequest, or any for
  sources. Provider profiles may define richer subtypes and future profiles may
  express more granular constraints, such as requiring a strongly authenticated
  UI principal session or rejecting API-key based authorization for a specific
  operation. Unsupported marker rules follow Blue Contracts must-understand
  semantics.
operations:
  type: Dictionary
  keyType: Text
  valueType:
    requiresActor:
      type: Text
      description: >
        Required actor category: principal, agent, or any.
      schema:
        enum: [principal, agent, any]
    requiresSource:
      type: Text
      description: >
        Required source category: browserSession, apiCall, documentRequest, or
        any.
      schema:
        enum: [browserSession, apiCall, documentRequest, any]
    excludeSource:
      type: Text
      description: >
        Disallowed source category: browserSession, apiCall, or
        documentRequest.
      schema:
        enum: [browserSession, apiCall, documentRequest]
  description: >
    Per-operation constraints keyed by Operation contract key. Each entry states
    the minimum actor/source category required before the operation may run.
    The most specific provider subtype policy may be expressed by using richer
    actor or source shapes in a future profile.
```

### 5.3 Event, request, response, and message types

#### Event

```yaml
name: Event
description: >
  Conversation package base type for domain events carried in Timeline Entry
  message fields or emitted by workflows. Event is not a contract and has no
  runtime behavior by itself. Concrete event subtypes define payload semantics.
```

#### Request

```yaml
name: Request
type: Conversation/Event
description: >
  Base type for a specific, trackable request to another participant, service,
  document, or institution. A Request carries requestId so a later Response can
  correlate itself to the exact request.
requestId:
  type: Text
  description: >
    Caller-generated identifier for this request. The recipient uses it to
    correlate responses. The identifier need only be unique within the relevant
    conversation or participant context unless a concrete subtype requires more.
```

#### Response

```yaml
name: Response
type: Conversation/Event
description: >
  Base type for an event that directly responds to a prior Request. A Response
  links to the original request through inResponseTo and may also include the
  incoming event that triggered the workflow, usually as a pure blueId reference
  or compact correlation object.
inResponseTo:
  type:
    name: Correlation
    description: >
      Structured reference linking this response back to the original request
      and, when useful, the event that initiated the workflow producing the
      response.
    requestId:
      type: Text
      description: requestId from the Request this Response answers.
    incomingEvent:
      description: Optional reference to the event that initiated the workflow.
  description: Correlation data for the response.
```

#### Chat Message

```yaml
name: Chat Message
type: Conversation/Event
description: >
  Human-readable conversational message carried in a Timeline Entry. Chat
  Message is for communication and audit. It does not invoke an operation unless
  a document explicitly defines a handler that treats it as such.
message:
  type: Text
  description: Message text exactly as supplied after Blue parsing.
```

#### Named Event

```yaml
name: Named Event
type: Conversation/Event
description: >
  Generic event with a human-readable name. It is useful for lightweight
  application events before a more specific typed event is standardized. Systems
  SHOULD prefer domain-specific event subtypes for portable automation.
```

#### Lifecycle Event

```yaml
name: Lifecycle Event
type: Conversation/Event
description: >
  Base event category for significant lifecycle changes in a document or
  process. Conversation Lifecycle Event is distinct from Core processor
  lifecycle events, although concrete documents may bridge or translate between
  them.
```

#### Status Change

```yaml
name: Status Change
type: Conversation/Event
description: >
  Event indicating that a document's semantic status transitioned. A Status
  Change records the new status node. It does not by itself mutate status; a
  workflow must apply an Update Document step if the document state should
  change.
status:
  type: Conversation/Document Status
  description: New status node.
```

### 5.4 Status types

#### Document Status

```yaml
name: Document Status
description: >
  Base type for semantic status indicators used by Conversation documents.
  Concrete status subtypes describe whether a document is pending, active,
  terminated successfully, or terminated with failure. Status is domain state,
  not processor termination state.
mode:
  type: Text
  description: Portable status category such as pending, active, or terminated.
```

#### Status Pending

```yaml
name: Status Pending
type: Conversation/Document Status
mode: pending
description: >
  Status indicating that the document or process has not yet started or is
  waiting for required input before active work begins.
```

#### Status In Progress

```yaml
name: Status In Progress
type: Conversation/Document Status
mode: active
description: >
  Status indicating that the document or process is active and has not reached a
  terminal successful or failed state.
```

#### Status Completed

```yaml
name: Status Completed
type: Conversation/Document Status
mode: terminated
description: >
  Status indicating that the document or process reached its intended successful
  terminal state. This is semantic completion and does not necessarily mean the
  Blue Contracts processor scope has a Processing Terminated Marker.
```

#### Status Failed

```yaml
name: Status Failed
type: Conversation/Document Status
mode: terminated
description: >
  Status indicating that the document or process reached a failed terminal
  state. This is semantic failure and does not necessarily mean a processor
  runtime fatal occurred.
```

### 5.5 Channels and operations

#### Timeline Channel

```yaml
name: Timeline Channel
type: Core/Channel
description: >
  Conversation external channel that delivers Timeline Entry events belonging
  to one configured Timeline. Timeline Channels are the way a Conversation
  document receives actions from participants and systems. A purchase agreement
  might have one Timeline Channel for the buyer and one for the seller; a
  payment document might have channels for a customer, merchant, payment
  service provider, and bank. Each channel binds the document to one verified
  perspective.

  The channel accepts an incoming event only when the event is a Timeline Entry
  and its timeline matches this channel's configured timeline under the
  concrete provider matching rules. Matching is about Timeline identity, not
  about message content. A forged message claiming to be from Alice does not
  match Alice's channel unless the Timeline envelope itself belongs to Alice's
  configured Timeline.

  The channelized payload is the full Timeline Entry. Handlers SHOULD match the
  fields they care about, commonly event.message.type for business behavior and
  event.actor or event.source for policy. The channel MUST NOT strip the
  envelope down to message, because deterministic ordering, idempotency,
  actor/source enforcement, timestamp audit, and prevEntry verification all
  require the full entry.

  Timeline Channel is checkpoint-gated as an external Blue Contracts channel,
  but the feeder is responsible for delivering only entries that are safe under
  the Timeline completeness algorithm. When a document has multiple Timeline
  Channels, the feeder obtains guarantees from all relevant providers, collects
  any older entries discovered, sorts the complete set, and invokes the core
  processor one Timeline Entry at a time. Processor-managed channels such as
  Document Update, Triggered Event, Lifecycle Event, and Embedded Node are not
  Timeline Channels.
timelineId:
  type: Text
  description: >
    Provider-scoped timeline identifier accepted by this channel. Concrete
    provider channel subtypes MAY add account, provider, tenant, chain, owner,
    contract, storage, or finality fields. If both timeline and timelineId are
    present, they MUST identify the same Timeline. This value participates in
    channel matching and checkpoint identity, so changing it changes which
    participant perspective the document is listening to.
timeline:
  type: Conversation/Timeline
  description: >
    Optional full Timeline descriptor accepted by this channel. Use this when a
    provider profile needs more than timelineId to address the Timeline or to
    define matching and completeness semantics.
```

#### Composite Timeline Channel

```yaml
name: Composite Timeline Channel
type: Core/Channel
description: >
  Conversation channel that accepts an incoming Timeline Entry if any named
  child channel in the same scope would accept it. It is a union channel for
  handler convenience and does not create a new Timeline, new provider
  guarantee, or new ordering domain. The child Timeline Channels still define
  the participant perspectives, completeness requirements, checkpoint subjects,
  and provider matching rules.

  A Composite Timeline Channel delivers at most one payload for a single
  incoming event even if more than one child channel would match. Its
  channelized payload is the original full Timeline Entry. This lets a handler
  listen to a group of participants while retaining the envelope fields needed
  to determine which Timeline produced the event, which actor performed it, and
  which source submitted it.
channels:
  type: List
  itemType: Text
  description: >
    Contract-map keys of same-scope channels to compose. Each key MUST name an
    existing supported Timeline-derived Channel in the same scope. The list
    order is not a cross-timeline delivery order and MUST NOT override the
    feeder's deterministic ordering by timestamp and tie-breakers; normal
    channel and handler ordering rules still apply after the feeder has chosen
    the next safe Timeline Entry to deliver.
```

#### Operation

```yaml
name: Operation
type: Core/Marker
description: >
  Conversation marker defining a callable action exposed by a document. An
  Operation declares the channel on which Operation Request messages are
  accepted and the request shape that invocation payloads must satisfy. An
  Operation is not executable by itself; a handler such as Sequential Workflow
  Operation implements it. Operation contract keys are operation names for
  Operation Request.operation unless a concrete profile defines aliases.
request:
  description: >
    Request schema or Blue shape for this operation. Operation Request.request
    MUST conform to this shape before the implementation handler applies state
    changes. A missing request shape means the operation accepts any valid Blue
    node as request payload.
channel:
  type: Text
  description: >
    Contract-map key of the same-scope Timeline Channel or Composite Timeline
    Channel through which Operation Request messages invoke this operation.
```

#### Operation Request

```yaml
name: Operation Request
description: >
  Conversation package event sent as a Timeline Entry message to invoke a named
  Operation on a specific target document snapshot or a proven newer descendant
  of that snapshot. The operation field names the Operation contract to invoke;
  request carries the payload; document identifies the exact document version
  for which the caller created the request; allowNewerVersion controls whether
  a later version in the same verified document lineage may be used. The
  document field is a binding target constraint, not a hint. A processor or
  feeder MUST reject the request before contract processing if it cannot prove
  that the selected target document is the supplied document version or, when
  allowNewerVersion is true, a newer committed version whose lineage includes
  the supplied document identity.
operation:
  type: Text
  description: >
    Name of the Operation to invoke. It MUST name an Operation contract in the
    target document scope. The caller is invoking that specific operation and no
    other operation.
request:
  description: >
    Payload for the operation. The payload MUST conform to the target
    Operation's request shape before state-changing workflow steps are applied.
document:
  description: >
    Required target document snapshot. The value MUST identify the exact Blue
    document version for which the request was created, either as a pure blueId
    reference to that document's Content BlueId or as a complete Blue document
    whose Content BlueId is the target identity. The operation MUST be applied
    only to that document version when allowNewerVersion is false, or to a
    newer committed version of the same document lineage that provably includes
    this identity when allowNewerVersion is true. A runtime MUST NOT ignore this
    field and MUST NOT apply the operation to an unrelated latest document.
allowNewerVersion:
  type: Boolean
  description: >
    Optional concurrent-modification flag. Missing is false. When false, the
    current selected target document Content BlueId MUST equal the identity
    supplied in document. When true, the runtime may operate on the latest
    known version of the same target document only after proving that the
    supplied document identity existed at some earlier committed point in that
    document's lineage. This flag never waives the document target check.
```

### 5.6 Workflow types

#### Sequential Workflow Step

```yaml
name: Sequential Workflow Step
description: >
  Abstract base type for one step in a Conversation Sequential Workflow.
  Concrete step types include Compute, JavaScript Code, Trigger Event, and
  Update Document. A step executes only inside its containing workflow and has
  no external runtime behavior by itself. Step names, when present, are used as
  keys in the workflow steps result map for later steps.
name:
  type: Text
  description: >
    Optional step-local name. If present, the step result is stored under this
    key and can be read by later steps. Duplicate names in one workflow are
    invalid unless a profile explicitly defines replacement semantics.
```

#### Sequential Workflow

```yaml
name: Sequential Workflow
type: Core/Handler
description: >
  Conversation Handler that executes an ordered list of Sequential Workflow Step
  nodes when it receives a matching channelized payload from its bound channel.
  The workflow evaluates any handler event matcher against the full channelized
  payload, normally a Timeline Entry. If the matcher accepts, steps run in list
  order. Each step receives immutable event, document, currentContract, and
  previous step-result context. Effects requested by steps are collected into a
  Blue Contracts Contract Execution Result and applied by the processor in the
  standard order. A Sequential Workflow MUST NOT perform external side effects
  and MUST NOT mutate the document except through Update Document patches
  returned to the processor.
steps:
  type: List
  itemType: Conversation/Sequential Workflow Step
  description: Ordered list of steps to execute in positional order.
```

#### Sequential Workflow Operation

```yaml
name: Sequential Workflow Operation
type: Conversation/Sequential Workflow
description: >
  Conversation handler pattern for implementing an Operation as a Sequential
  Workflow. The operation field names an Operation contract in the same scope.
  The effective handler channel is the referenced Operation's channel. If this
  workflow also supplies a channel field, it MUST equal the referenced
  Operation's channel. The workflow runs only for Operation Request messages
  whose operation field names the referenced Operation and whose request payload
  conforms to the Operation's request shape. This type is a Conversation
  profile extension over ordinary handler binding; processors that do not
  support it must treat it as unsupported under must-understand rules.
operation:
  type: Text
  description: >
    Contract-map key of the Operation this workflow implements. The referenced
    Operation MUST exist in the same scope and MUST define the invocation
    channel and request shape.
```

#### Update Document

```yaml
name: Update Document
type: Conversation/Sequential Workflow Step
description: >
  Workflow step that applies a list of Blue Contracts Json Patch Entry objects
  to the current selected document by returning them to the processor as
  handler patches. Patches are applied in list order using the Blue Contracts
  patch, boundary, cascade, checkpoint, termination, gas, and post-patch type
  soundness rules. This step is the only standard Conversation workflow step
  that directly requests persistent document mutation. If a changeset is
  computed by an earlier Compute or JavaScript Code step, this step is what
  turns that computed data into processor-applied patches.
changeset:
  type: List
  itemType: Core/Json Patch Entry
  description: >
    Ordered patch entries to apply. Each entry MUST be valid under the Blue
    Contracts Json Patch Entry rules. Implementations MAY support deterministic
    template or BEX substitution before validation, but the final changeset
    supplied to the processor MUST contain concrete patch entries.
```

#### Trigger Event

```yaml
name: Trigger Event
type: Conversation/Sequential Workflow Step
description: >
  Workflow step that emits one event through the current scope's Triggered
  event mechanism. The emitted event is recorded under the executing scope and
  enqueued into that scope's Triggered FIFO according to the Blue Contracts
  processor rules. Trigger Event does not deliver the event immediately during
  Document Update cascades.
event:
  description: >
    Event payload to emit. The final event MUST be a valid Blue node for event
    delivery. Implementations MAY support deterministic template or BEX
    substitution before validation.
```

#### Compute

```yaml
name: Compute
type: Conversation/Sequential Workflow Step
description: >
  Workflow step that evaluates Blue Expression Objects (BEX) using the current
  workflow context and returns a deterministic BEX result. Compute may read the
  current document view, event, current contract, previous step results,
  constants, functions, and host bindings exposed by the Conversation runtime.
  Compute does not mutate the document directly. If the computed result contains
  a changeset, that changeset remains data until an Update Document step applies
  it. If emitEvents is true and the computed result contains events, the
  Conversation workflow may emit those events as Triggered events according to
  processor rules. BEX execution MUST remain deterministic and MUST NOT perform
  external side effects.
expr:
  description: Optional BEX expression used as the Compute program root.
do:
  type: List
  description: Ordered list of BEX statements used as the Compute program root.
definition:
  description: >
    Optional reference or contract key for reusable Compute Definition content.
entry:
  type: Text
  description: Optional BEX entry function name.
constants:
  type: Dictionary
  description: Named constants available to BEX through $const.
functions:
  type: Dictionary
  description: Named reusable BEX functions.
emitEvents:
  type: Boolean
  value: true
  description: >
    If true, events returned by the BEX result may be emitted by the workflow.
returnResult:
  type: Boolean
  value: true
  description: >
    If true, the Compute result is stored under the step name for later steps.
gasLimit:
  type: Integer
  description: Optional maximum BEX gas for this Compute step.
```

#### Compute Definition

```yaml
name: Compute Definition
type: Core/Marker
description: >
  Conversation marker containing reusable BEX constants and functions for
  Compute steps in the same scope. A Compute step may reference this definition
  by contract key or by provider reference according to the Conversation
  runtime profile. Compute Definition is identity-bearing content and does not
  execute by itself.
constants:
  type: Dictionary
  description: Named constants made available to referencing Compute steps.
functions:
  type: Dictionary
  description: Named BEX function definitions made available to referencing Compute steps.
```

#### JavaScript Code

```yaml
name: JavaScript Code
type: Conversation/Sequential Workflow Step
description: >
  Compatibility workflow step that executes deterministic sandboxed JavaScript
  as part of a Sequential Workflow. Portable Conversation 1.0 workflows SHOULD
  prefer Compute/BEX. A JavaScript Code implementation MUST expose only
  deterministic APIs, MUST disable clocks, randomness, network, filesystem,
  eval-like dynamic code creation, nondeterministic sorting, and hidden mutable
  host state, and MUST meter execution deterministically. Returned patches and
  events are data until the workflow converts them into processor effects.
code:
  type: Text
  description: Deterministic JavaScript source for this step.
```

### 5.7 Change-management types

#### Change Request

```yaml
name: Change Request
description: >
  Payload for direct or proposed document-change operations. A Change Request
  may include ordinary document patches, structured section-based contract
  changes, and a human-readable summary. When a Contracts Change Policy is
  active, changes to /contracts MUST be represented through sectionChanges
  rather than raw changeset entries.
summary:
  type: Text
  description: Human-readable summary of the requested change.
changeset:
  type: List
  itemType: Core/Json Patch Entry
  description: >
    Ordered patch entries for ordinary document changes. A policy may reject
    patches targeting /contracts or reserved processor paths.
sectionChanges:
  type: Conversation/Document Section Changes
  description: Structured /contracts mutations grouped by document sections.
```

#### Change Operation

```yaml
name: Change Operation
type: Conversation/Operation
description: >
  Operation that applies a Change Request immediately to the document when
  authorized. Concrete workflows implementing this operation MUST validate
  policy, reserved paths, section changes, and patch legality before applying
  Update Document.
request:
  type: Conversation/Change Request
```

#### Propose Change Operation

```yaml
name: Propose Change Operation
type: Conversation/Operation
description: >
  Operation that stores a Change Request as a proposed change for later
  acceptance or rejection instead of applying it immediately. The proposed
  change is document state and should be visible for review.
request:
  type: Conversation/Change Request
```

#### Accept Change Operation

```yaml
name: Accept Change Operation
type: Conversation/Operation
description: >
  Operation that accepts a previously stored proposed change and applies the
  proposed changeset according to the document's current policies.
```

#### Reject Change Operation

```yaml
name: Reject Change Operation
type: Conversation/Operation
description: >
  Operation that rejects a previously stored proposed change and removes or
  marks the proposal state without applying its changeset.
```

#### Proposed Change Invalid

```yaml
name: Proposed Change Invalid
type: Conversation/Event
description: >
  Event emitted when a proposed or direct change operation cannot be applied
  because it violates policy, has invalid shape, attempts forbidden paths, lacks
  a required summary, duplicates section keys, or otherwise fails deterministic
  validation.
reason:
  type: Text
  description: Deterministic human-readable reason for invalidity.
```

#### Document Section

```yaml
name: Document Section
type: Core/Marker
description: >
  Declarative marker documenting one logical section of a document and linking
  that section to relevant fields and contracts. Document Section markers are
  used by sectionChanges to make /contracts changes reviewable and auditable as
  semantic sections rather than arbitrary raw contract-map mutations.
title:
  type: Text
  description: Short user-facing section title.
summary:
  type: Text
  description: Brief functional summary of the section's purpose and behavior.
relatedFields:
  type: List
  itemType: Text
  description: Absolute runtime pointer paths of ordinary fields covered by this section.
relatedContracts:
  type: List
  itemType: Text
  description: Same-scope contract keys that implement or affect this section.
```

#### Document Section Change Entry

```yaml
name: Document Section Change Entry
description: >
  One structured change to a document section and its related contracts. The
  sectionKey identifies the section marker contract key. The section field
  describes the Document Section marker to add or replace. The contracts map
  contains related contract entries that belong to the section.
sectionKey:
  type: Text
  description: Contract-map key of the Document Section marker.
section:
  type: Conversation/Document Section
  description: Section marker content.
contracts:
  type: Dictionary
  description: Related contract entries keyed by contract-map key.
```

#### Document Section Changes

```yaml
name: Document Section Changes
description: >
  Structured set of /contracts mutations grouped by logical document sections.
  This type lets change workflows add, modify, or remove section markers and
  their related contracts while preserving reviewable intent. It is used when a
  Contracts Change Policy requires section-based contract changes.
add:
  type: List
  itemType: Conversation/Document Section Change Entry
  description: New sections and related contracts to create.
modify:
  type: List
  itemType: Conversation/Document Section Change Entry
  description: Existing sections and related contracts to replace or update.
remove:
  type: List
  itemType: Text
  description: Section keys to remove, along with their related contracts.
```

#### Contracts Change Policy

```yaml
name: Contracts Change Policy
type: Core/Marker
description: >
  Conversation policy marker restricting mutation of /contracts. When enabled,
  raw Json Patch entries targeting /contracts are rejected by Conversation
  change workflows, and contract changes must be expressed as structured
  sectionChanges. This policy does not override Blue Contracts reserved-key
  write protection; processor-reserved paths remain protected.
requireSectionChanges:
  type: Boolean
  description: >
    When present and not false, /contracts changes must use sectionChanges.
    Setting false disables this Conversation-level policy while leaving core
    processor protections intact.
```

#### Change Workflow

```yaml
name: Change Workflow
type: Conversation/Sequential Workflow Operation
description: >
  Standard workflow pattern that validates a Change Request and applies it
  immediately when valid. The workflow MUST reject attempts to mutate protected
  processor paths and MUST honor Contracts Change Policy before applying an
  Update Document step.
request:
  type: Conversation/Change Request
  description: Expected operation request payload.
```

#### Propose Change Workflow

```yaml
name: Propose Change Workflow
type: Conversation/Sequential Workflow Operation
description: >
  Standard workflow pattern that validates a Change Request and stores it as a
  proposed change under a deterministic proposal path for later accept/reject.
  It MUST NOT apply the proposed changeset during proposal creation.
request:
  type: Conversation/Change Request
postfix:
  type: Text
  description: Optional postfix used to build the proposal state key.
```

#### Accept Change Workflow

```yaml
name: Accept Change Workflow
type: Conversation/Sequential Workflow Operation
description: >
  Standard workflow pattern that loads a stored proposed change, validates that
  it is still applicable to the current document and policies, applies its
  changeset through Update Document, and removes the proposal state.
postfix:
  type: Text
  description: Optional postfix used to locate the proposal state key.
```

#### Reject Change Workflow

```yaml
name: Reject Change Workflow
type: Conversation/Sequential Workflow Operation
description: >
  Standard workflow pattern that rejects a stored proposed change and removes
  the proposal state without applying the proposed changeset.
postfix:
  type: Text
  description: Optional postfix used to locate the proposal state key.
```

### 5.8 Bootstrap, user-action, and consent types

#### Document Bootstrap Requested

```yaml
name: Document Bootstrap Requested
type: Conversation/Request
description: >
  Request to create or bootstrap a new Blue document conversation from a
  provided document template. The request may include channel bindings,
  invitation messages, and the participant responsible for performing the
  bootstrap. Bootstrapping is outside the core processor unless a supported
  workflow or provider profile implements it.
document:
  description: Target Blue document or template to bootstrap.
channelBindings:
  type: Dictionary
  keyType: Text
  valueType: Core/Channel
  description: Mapping from channel names in the new document to participant/provider channel descriptors.
initialMessages:
  description: Invitation messages for participants.
  defaultMessage:
    type: Text
    description: Default invitation message.
  perChannel:
    type: Dictionary
    keyType: Text
    valueType: Text
    description: Channel-specific invitation messages.
bootstrapAssignee:
  type: Text
  description: Channel name of participant asked to bootstrap the document.
onBehalfOf:
  type: Text
  description: Account or participant on whose behalf the document is created.
```

#### Document Bootstrap Responded

```yaml
name: Document Bootstrap Responded
type: Conversation/Response
description: >
  Response accepting or rejecting a Document Bootstrap Requested event.
status:
  type: Text
  description: Decision for the bootstrap request.
  schema:
    enum: [accepted, rejected]
reason:
  type: Text
  description: Optional reason or context, especially when rejected.
```

#### Document Bootstrap Completed

```yaml
name: Document Bootstrap Completed
type: Conversation/Response
description: >
  Response confirming that document bootstrap completed and identifying the
  created document.
documentId:
  type: Text
  description: Identifier or BlueId of the bootstrapped document.
```

#### Document Bootstrap Failed

```yaml
name: Document Bootstrap Failed
type: Conversation/Response
description: >
  Response indicating that document bootstrap failed deterministically or at the
  provider level.
reason:
  type: Text
  description: Failure reason.
```

#### Inform User About Pending Action

```yaml
name: Inform User About Pending Action
type: Conversation/Event
description: >
  Event notifying a participant or provider that a user action is required to
  continue a document process. It names the operation the user is expected to
  run, provides user-facing text, identifies the channel to watch for the
  follow-up Timeline Entry, and describes the expected request shape.
operation:
  type: Text
  description: Operation name the user should invoke.
title:
  type: Text
  description: Short user-facing title.
message:
  type: Text
  description: Human-readable explanation of the required action.
channel:
  type: Text
  description: Channel expected to receive the follow-up operation request.
expectedRequest:
  description: Expected request payload shape for the operation.
```

#### Customer Action Requested

```yaml
name: Customer Action Requested
type: Conversation/Request
description: >
  Event by which a document asks a bank, provider, or UI host to request a
  specific customer decision or input. It is a request for out-of-band human UI
  action, not proof that the customer already acted.
title:
  type: Text
  description: Short title for the pending action.
message:
  type: Text
  description: Human-readable message shown to the customer.
actions:
  type: List
  itemType:
    label:
      type: Text
      description: User-facing action label.
    variant:
      type: Text
      description: Optional button style such as primary, secondary, or reject.
    inputSchema:
      description: Optional input schema for additional customer input.
    inputRequired:
      type: Boolean
      description: Whether input is required for this action.
    inputTitle:
      type: Text
      description: Title displayed above the input control.
    inputPlaceholder:
      type: Text
      description: Placeholder or help text for input.
  description: Available user actions.
```

#### Customer Action Responded

```yaml
name: Customer Action Responded
type: Conversation/Response
description: >
  Provider or bank response reporting the customer's decision for a prior
  Customer Action Requested event. The provider is responsible for authenticating
  the customer interaction and attributing the resulting Timeline Entry.
actionLabel:
  type: Text
  description: Label of the action selected by the user.
input:
  description: Optional user-provided payload for the selected action.
respondedAt:
  type: Common/Timestamp
  description: Provider-recorded response timestamp.
```

#### Customer Consent

```yaml
name: Customer Consent
description: >
  Generic document recording a customer's consent granted to a grantee, such as
  a merchant, bank, platform, or agent. Consent documents may themselves be
  Conversation documents with Timeline Channels and revocation operations.
consentKind:
  type: Text
  description: Classification of consent, such as cardTransactionMonitoring.
consentDetails:
  type: Dictionary
  keyType: Text
  valueType: Text
  description: Scope details for the consent.
consentStatus:
  type: Text
  description: Consent state, commonly granted or revoked.
grantedAt:
  type: Common/Timestamp
  description: Timestamp when consent was granted.
revokedAt:
  type: Common/Timestamp
  description: Timestamp when consent was revoked.
revocationReason:
  type: Text
  description: Reason for revocation.
```

#### Customer Consent Revoked

```yaml
name: Customer Consent Revoked
type: Conversation/Event
description: >
  Event indicating that a Customer Consent document was revoked.
reason:
  type: Text
  description: Revocation reason.
revokedAt:
  type: Common/Timestamp
  description: Revocation timestamp.
```

### 5.9 Common support types

#### Timestamp

```yaml
name: Timestamp
type: Text
description: >
  Common text type for ISO 8601 timestamps with timezone offset, for example
  2025-01-10T09:30:00Z, 2025-01-10T10:30:00+01:00, or
  2025-01-10T09:30:00.123456Z. This is a Text subtype for semantic clarity;
  portable Blue Language 1.0 has no core timestamp scalar.
```

#### Currency

```yaml
name: Currency
type: Text
description: >
  Common text type for ISO 4217 currency codes such as USD, EUR, GBP, or PLN.
  This type identifies currency code text; numeric money amount semantics are
  defined by concrete payment or document types.
```

#### Document

```yaml
name: Document
description: >
  Common base type for semantic documents that describe their business meaning
  directly. A concrete document type should define its own fields and contracts.
  The kind field provides a human-readable business classification in addition
  to the document's Blue type identity.
kind:
  type: Text
  description: Human-readable business meaning of this document.
```

#### Document Anchor

```yaml
name: Document Anchor
description: >
  Common type for a named connection point where another matching document may
  link into this document. Anchors describe expected document shape through an
  optional template and are used by agents to compose multi-document processes.
template:
  description: Optional Blue document-shaped template describing what matches this anchor.
```

#### Document Anchors

```yaml
name: Document Anchors
type: Dictionary
keyType: Text
valueType: Common/Document Anchor
description: >
  Dictionary of named Document Anchor entries. Keys are anchor names. Values
  describe the expected matching document shape or relationship point.
```

#### PermissionGrant

```yaml
name: PermissionGrant
type: Common/Document
description: >
  Common document family for documents whose main purpose is granting,
  constraining, or revoking authority. Concrete subtypes should define the
  grantee, grantor, scope, revocation rules, and evidence required to rely on
  the permission.
```

#### Profile

```yaml
name: Profile
type: Common/Document
description: >
  Common document family for identity, business, service, or participant
  profiles. A Profile is descriptive content and carries no authority unless a
  concrete contract or provider policy gives it authority.
```

#### Service

```yaml
name: Service
type: Common/Document
description: >
  Common document family for a service offered by a participant. Services may
  expose anchors, terms, categories, or Conversation operations in concrete
  subtypes.
categories:
  type: List
  itemType: Text
  description: Optional service categories.
```

#### Space

```yaml
name: Space
type: Common/Document
description: >
  Common document family for a participant-controlled space, such as a business
  presence, collaboration area, or marketplace surface. Spaces may expose
  anchors for agents to connect related documents.
website:
  type: Text
  description: Optional public website.
anchors:
  type: Common/Document Anchors
  description: Optional connection points for matching documents.
```

#### Task

```yaml
name: Task
type: Common/Document
description: >
  Common document family for a unit of work, obligation, or requested action.
  Concrete task types should define assignee, requester, status, evidence, and
  completion conditions.
```

#### Payment

```yaml
name: Payment
type: Common/Document
description: >
  Common document family for payment-related documents. Concrete payment types,
  such as PayNote or rail-specific payment envelopes, define participants,
  amounts, lifecycle events, authorization, settlement, cancellation, and
  reversal semantics.
```

#### Record

```yaml
name: Record
type: Common/Document
description: >
  Common document family for factual records or attestations. A Record should
  state what is recorded, who recorded it, source evidence, and any lifecycle or
  correction rules in concrete subtypes.
```

#### Relationship

```yaml
name: Relationship
type: Common/Document
description: >
  Common document family for a relationship between two or more participants,
  documents, services, or accounts. Concrete subtypes define the parties,
  terms, permissions, and termination rules.
```

#### Common Request

```yaml
name: Request
type: Common/Document
description: >
  Common document family for persistent request documents. This is distinct from
  Conversation/Request, which is an event type. A Common Request document may
  itself contain Conversation contracts for approval, rejection, fulfillment, or
  escalation.
```

#### Common Response

```yaml
name: Response
type: Common/Document
description: >
  Common document family for persistent response documents. This is distinct
  from Conversation/Response, which is an event type.
```

---

## 6. Normative Processing Algorithms

### 6.1 Timeline feeder algorithm

```text
function PROCESS_TIMELINE_NOTIFICATION(document, notifiedChannelKey, notifiedEntry):
    S = notifiedEntry.timestamp
    W = provider_next_boundary_after(S)  # commonly S + 1 microsecond

    channels = discover_conversation_timeline_channels(document)

    entries = []
    for channel in channels:
        timeline = channel.timeline
        guarantee = provider(timeline).guarantee_no_entries_before(W)
        verify_guarantee(guarantee, timeline, W)
        entries += provider(timeline).entries_in_window(channel.cursor.watermark, W)

    entries = remove_entries_at_or_below_already_processed_cursors(entries)
    sort entries by (timestamp, channelKey, timelineId, sequence?, entryBlueId)

    for entry in entries:
        document, outbox, gas = PROCESS(document, entry)
        persist(document, outbox, gas)

    return document
```

A feeder MAY choose a larger safe boundary than the triggering entry requires. It MUST NOT process entries at timestamps not covered by a completeness guarantee for every relevant Timeline Channel.

### 6.2 Operation request handling

```text
function RESOLVE_OPERATION_REQUEST(request, documentStore):
    D_expected = contentBlueId(request.document)
    target = documentStore.resolve_target_session(request.document)

    if request.allowNewerVersion is true:
        current = target.latest()
        require target.lineage_contains(D_expected)
        return current
    else:
        current = target.current()
        require contentBlueId(current) == D_expected
        return current
```

Only after this target check succeeds may the Timeline Entry be delivered to the document processor.

### 6.3 Sequential Workflow Operation recognition

```text
function RECOGNIZE_WORKFLOW_OPERATION(scope, handler):
    opKey = handler.operation
    op = scope.contracts[opKey]
    require op is Conversation/Operation
    effectiveChannel = op.channel
    if handler.channel exists:
        require handler.channel == effectiveChannel
    bind handler to effectiveChannel
```

---

## 7. Conformance Checklist

A conforming Blue Timelines and Conversation 1.0 implementation MUST:

- treat Timeline Entry as the external event envelope for Conversation Timeline Channels;
- verify Timeline Channel matching against provider-defined Timeline identity;
- preserve the full Timeline Entry as channelized payload;
- evaluate Operation Request target document identity before applying operation effects;
- treat `allowNewerVersion` as lineage relaxation only, never as permission to ignore `document`;
- derive Sequential Workflow Operation channel binding from the referenced Operation or reject invalid bindings;
- apply Operation request-shape validation before state-changing steps;
- enforce Actor Policy when supported and reject unsupported Actor Policy under must-understand rules when active;
- execute Sequential Workflow steps in deterministic order;
- apply Update Document steps only through Blue Contracts patch semantics;
- ensure Compute/BEX does not directly mutate the document;
- keep JavaScript Code, when supported, deterministic and sandboxed;
- use half-open completeness windows for Timeline Provider guarantees;
- sort multi-timeline entries deterministically;
- never process entries outside a completeness-guaranteed safe window;
- checkpoint accepted Timeline Channel entries or cursors deterministically;
- preserve Blue Language identity boundaries for all canonical Conversation source nodes.

---

## 8. Behavior-Defining Test Vectors

**C1 — Timeline channel accepts matching entry**  
Given a Timeline Channel for `timelineId: alice`, when a Timeline Entry with `timeline.timelineId: alice` is delivered, the channel accepts and delivers the full entry as payload.

**C2 — Timeline channel rejects different timeline**  
A Timeline Channel for `alice` rejects an entry from `bob` and does not create/update a checkpoint unless another channel accepts it.

**C3 — Completeness is half-open**  
A guarantee `no entries before T` permits processing entries with timestamp `< T` and does not permit processing entries with timestamp `T` unless a later boundary covers them.

**C4 — Triggering entry included by next boundary**  
For an entry timestamp `S`, a feeder queries boundary `S + 1` or a provider-defined next boundary before processing the entry.

**C5 — Deterministic multi-timeline ordering**  
Entries from two timelines in the same safe window are sorted by timestamp, channel key, timelineId, sequence when present, then entry BlueId.

**C6 — Operation Request exact document**  
With `allowNewerVersion: false`, an Operation Request whose `document` identifies an older document is rejected if the current target document has a different Content BlueId.

**C7 — Operation Request newer lineage allowed**  
With `allowNewerVersion: true`, an Operation Request for document `D1` may run on current document `D3` only if the document store proves `D1 -> ... -> D3` in the same lineage.

**C8 — Operation Request unrelated latest rejected**  
With `allowNewerVersion: true`, a request for document `D1` is rejected if the latest document does not have `D1` in its lineage.

**C9 — Sequential Workflow Operation channel derivation**  
A workflow operation naming `approve` binds to `contracts.approve.channel`. If it also declares a different `channel`, recognition fails.

**C10 — Compute does not mutate**  
A Compute step returning a changeset does not change the document unless a later Update Document step applies that changeset.

**C11 — Actor Policy principal required**  
An operation requiring `principal` rejects a Timeline Entry whose actor is a subtype of Agent Actor.

**C12 — Composite channel single delivery**  
A Composite Timeline Channel whose two child channels both match a Timeline Entry delivers one payload to its handlers, not two.

---

## Appendix A — Common Implementer Mistakes

### A.1 Do not process `<= T` for a `no entries before T` guarantee

The portable guarantee closes timestamps strictly less than `T`. Use a later boundary to include entries at `T`.

### A.2 Do not treat `Operation Request.document` as a UI hint

The field binds the operation to a specific document identity or proven newer descendant. Ignoring it breaks concurrency safety.

### A.3 Do not let `allowNewerVersion` jump documents

It permits only newer versions in the same verified lineage.

### A.4 Do not deliver only `message` as Timeline Channel payload

Handlers may need actor, source, timestamp, timeline, and prevEntry. The channel payload is the full Timeline Entry.

### A.5 Do not let Compute patches mutate automatically

BEX returns data. Update Document applies patches.

### A.6 Do not use actor fields supplied only inside message payload as authority

Actor attribution must be asserted by the Timeline Provider in the Timeline Entry actor field.

---

## Appendix B — Release Artifacts

A Blue Timelines and Conversation 1.0 release SHOULD publish:

- this prose specification;
- canonical Conversation source node files;
- a manifest listing each canonical source node, preprocessed/canonical form, BlueId, and fixture package identity;
- conformance fixtures for Timeline Channel matching, completeness windows, operation request document targeting, actor policy, workflow operation binding, Compute, and Update Document;
- provider-profile fixtures for at least one managed Timeline Provider and, if supported, one blockchain-finality Timeline profile.

---

*End of Blue Timelines and Conversation Specification 1.0.*
