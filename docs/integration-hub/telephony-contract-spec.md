# Telephony Capability Contract — Specification

**Module:** 02 Integration Hub → Telephony
**Version:** 0.1 (draft for design review)
**Date:** 17 October 2026
**Status:** Proposed. Provider behaviour, Malaysian number availability and regulatory requirements must be validated with each carrier and each client's compliance team before build commitments.
**Parent document:** Daythree AI Voice Platform Blueprint v1.0

---

## 1. Purpose and scope

This contract defines how the platform controls telephony **independently of any carrier**. Modules 05 (Campaign/Inbound Router), 08 (Workflow Execution), 07 (Voice Engine) and 09 (Interaction/Recording) depend only on this contract. Each carrier is an **adapter** that implements it. The first adapter is Twilio. NICE CXone is **not** in the first release; the contract must nevertheless allow it, any SIP trunk provider, or a client's own PBX to be added later with no change to Modules 05, 07, 08 or 09.

### In scope
- Number (DID) inventory, lifecycle, caller-ID governance, routing bindings
- Inbound call control and routing
- Outbound dial control, admission control, and call outcomes
- Media bridge (audio transport to the Voice Engine)
- Transfer, recording control, and call-leg handling
- Canonical call states, events, reason codes and identifiers
- Cost/usage events and reconciliation hooks
- Security, tenancy, resilience, telemetry and conformance requirements

### Out of scope (v0.1)
- SMS, WhatsApp and email channels (the contract reserves a channel field, nothing more)
- Predictive/power dialling for human agents
- Multi-carrier least-cost routing and automatic failover (contract must not block it)
- Contact-centre agent desktop and queue management
- Carrier commercial negotiation and regulatory filings

### Terminology
| Term | Meaning |
|---|---|
| **Adapter** | Carrier-specific implementation of this contract |
| **Provider account** | A carrier-side account or sub-account holding numbers and credentials for one tenant |
| **Leg** | One party's connection in a call (customer leg, AI/media leg, transfer leg) |
| **DID** | A telephone number owned/leased for inbound or caller-ID use |
| **CLI** | Calling line identity — the number presented to the callee |
| **Dial job** | An approved instruction to attempt one outbound call |
| **Attempt** | One dial of one dial job; has an `attempt_id` |
| **Session** | The AI-side live conversation session bound to a call |

---

## 2. Design principles

1. **Capability-based, not provider-shaped.** The contract names what is needed (answer, dial, transfer, stream audio), never carrier artefacts (markup documents, SIDs, TwiML-style verbs).
2. **Compliance decided before the carrier.** The adapter never decides whether a call is permitted. The platform's policy gate approves a dial job; the adapter only executes it.
3. **Canonical in, canonical out.** All inbound carrier callbacks are normalised to canonical events; all platform commands use canonical commands.
4. **Idempotent and replayable.** Every command carries an idempotency key; every event is deduplicated and ordered per call.
5. **Keyed on the customer, not the DID.** Suppression, consent, windows and retry limits never depend on which number is used (Blueprint §5.2).
6. **Fail safe for the customer.** If AI, adapter or carrier fails mid-call, the customer reaches a human, a callback or a clear closing message — never silence.
7. **Cost-aware.** Every chargeable telephony action produces a usage/cost event.
8. **Tenant-isolated.** Credentials, numbers, events and spend are scoped per tenant.

---

## 3. Architecture

```
Campaign / Inbound Router (05)   Workflow Executor (08)   Voice Engine (07)   Recording (09)
              │                          │                       │                   │
              ▼                          ▼                       ▼                   ▼
   ┌───────────────────────── TELEPHONY CAPABILITY CONTRACT ─────────────────────────────┐
   │  Number Mgmt │ Inbound Control │ Outbound Control │ Call Control │ Media Bridge │ Usage │
   └───────────────────────────────────┬─────────────────────────────────────────────────┘
                                       │ Adapter SPI
           ┌───────────────┬───────────┴───────────┬──────────────────┐
        Twilio (v1)   Generic SIP trunk (v2)   Other carrier   Client PBX / contact centre
```

### Platform-side components behind the contract
| Component | Responsibility |
|---|---|
| **Telephony Gateway** | Single entry point for commands; validates, authorises, applies idempotency, routes to the correct adapter by tenant/number/route |
| **Event Normaliser** | Receives carrier callbacks, authenticates them, deduplicates, orders, maps to canonical events, publishes to the event bus |
| **Number Registry** | System of record for DIDs, lifecycle, bindings, reputation |
| **Admission Controller** | Enforces carrier rate, carrier concurrency, AI session capacity, tenant fair-share and spend caps before any dial |
| **Reconciler** | Detects calls with missing events/costs and repairs state from carrier records |

---

## 4. Identifiers and correlation

Uses the platform identifiers defined in Blueprint §3.1. Telephony adds:

| Identifier | Scope | Rule |
|---|---|---|
| `number_id` | Platform | Stable ID of a DID in the Number Registry. Never the number string itself as a key |
| `route_id` | Platform | Inbound routing rule |
| `dial_job_id` | Platform | One approved outbound instruction |
| `attempt_id` | Platform | One dial of a dial job; generated before the carrier is called |
| `call_id` | Platform | One telephony session; new on resume (Blueprint §3.1) |
| `leg_id` | Platform | One leg within a call |
| `provider_ref` | Adapter | Carrier's identifier for a call/leg/number. Stored, indexed, **never** a primary key |
| `provider_account_id` | Platform | Tenant's account/sub-account at the carrier |
| `idempotency_key` | Command | Unique per intended effect |

**Rules**
- `attempt_id` and `call_id` are created **before** the carrier is contacted, so a lost response can be reconciled.
- Every event carries `tenant_id`, `call_id`, `trace_id`, `provider_ref` where known.
- Phone numbers are stored in E.164 and masked in logs and traces by default.

---

## 5. Capability A — Number management

### 5.1 Operations
| Operation | Purpose | Notes |
|---|---|---|
| `search_numbers` | Find available numbers by country, type, capability, pattern | Read-only, no cost |
| `order_number` | Acquire a number | Returns `ordered` or `regulatory_pending`; idempotent |
| `submit_regulatory_info` | Supply documents/addresses a carrier requires | Status tracked asynchronously |
| `get_number` / `list_numbers` | Read registry and reconcile with carrier | |
| `bind_number` | Bind to purpose, tenant, route or campaign | Platform-side; adapter configures carrier routing |
| `unbind_number` | Remove a binding | Requires no active calls or explicit drain |
| `release_number` | Return to carrier | Requires approval; irreversible; audited |
| `import_number` | Bring an existing/BYO number under management | Requires ownership evidence |
| `verify_caller_id` | Verify a number as presentable CLI where carrier requires | Async |

### 5.2 Number record (logical fields)
`number_id`, `e164`, `country`, `number_type` (geographic, mobile, toll-free, shared-cost, other), `capabilities` (voice in/out, sms, whatsapp), `provider`, `provider_account_id`, `provider_ref`, `tenant_id`, `client_id`, `purpose` (inbound, outbound_cli, both), `lifecycle_state`, `regulatory_status`, `bound_route_id`, `bound_campaign_ids`, `cli_approval` (approver, date, scope, expiry), `monthly_cost`, `rate_card_ref`, `reputation_score`, `reputation_flags`, `created_at`, `activated_at`, `released_at`, `owner_of_record` (Daythree / client / third party).

### 5.3 Lifecycle
```
searched → ordered → regulatory_pending → active ⇄ restricted → released
                                  │            │
                                  └→ rejected  └→ quarantined (auto/manual) → active | released
```
| State | Meaning | Allowed use |
|---|---|---|
| `regulatory_pending` | Awaiting carrier/regulator approval | None |
| `active` | Usable | Per `purpose` and approvals |
| `restricted` | Admin hold (e.g. pending CLI approval, spend, legal) | Inbound only, or none, per hold type |
| `quarantined` | Reputation or complaint trigger | Excluded from outbound selection; inbound unaffected unless ruled otherwise |
| `released` | Returned | None; kept for audit |

### 5.4 Caller-ID governance
- A number may be **presented as CLI** only if `cli_approval` is valid, the number is `active`, and the tenant/client is permitted to use it.
- CLI selection is a platform decision made by the Admission Controller from an approved pool. The adapter receives the CLI explicitly and must not substitute another.
- Every CLI presented is recorded on the attempt.
- Changing, rotating or adding numbers **must not** reset or bypass suppression, consent, calling windows or retry counters, because those are keyed on customer and contact point.

### 5.5 Reputation and quarantine
Per outbound number, track rolling answer rate, short-call rate (answered then ended under N seconds), reject rate, complaint/blocked-label indicators where available, and volume per hour/day. Thresholds trigger `quarantined` and an alert. Thresholds are tenant-configurable with platform minimum safeguards. A quarantined number is never auto-returned to service without review.

---

## 6. Capability B — Inbound control

### 6.1 Flow
```
Carrier receives call to DID
   → Event Normaliser authenticates and normalises → call.inbound_received
   → Router resolves number_id → route_id → tenant/client/workflow_version
   → Policy checks (hours, capacity, blocklist, emergency/priority rules)
   → Decision: ACCEPT_AI | TRANSFER | PLAY_AND_END | REJECT | OVERFLOW
   → Media Bridge opens session → Voice Engine takes over
```

### 6.2 Routing rule (`route`) fields
`route_id`, `tenant_id`, `client_id`, `match` (number_id or set; optional caller pattern or segment), `workflow_id` + `workflow_version` (or alias such as "production"), `schedule` (business hours, holidays, timezone), `capacity_limit`, `overflow_action`, `failure_action`, `recording_policy_ref`, `disclosure_policy_ref`, `language_hint`, `priority`, `status`, `version`, `effective_from`.

Routes are versioned and changes are audited. The platform resolves routes at answer time using a cached snapshot so that a database slowdown cannot delay answering.

### 6.3 Decisions
| Decision | Behaviour |
|---|---|
| `ACCEPT_AI` | Answer, open media session, start workflow |
| `TRANSFER` | Direct to configured destination without AI (e.g. outside AI hours, VIP, regulatory) |
| `PLAY_AND_END` | Play a configured message then end (closed, blocked, over capacity) |
| `REJECT` | Decline before answer with a reason (blocked, invalid) |
| `OVERFLOW` | Redirect to secondary route or destination |

### 6.4 Failure behaviour (mandatory)
| Failure | Required behaviour |
|---|---|
| Platform routing unavailable | Carrier-level fallback destination (human number or recorded message) configured per DID |
| Voice Engine at capacity | Apply `overflow_action`; never answer then leave dead air |
| Workflow version missing/retired | Use last known good alias; alert |
| Callback authentication fails | Reject, log security event |

### 6.5 Caller recognition
The inbound caller number is a **hint** for context retrieval and previous-conversation linking (missed-callback, resume). It is never evidence of identity (Blueprint §7). Withheld or invalid numbers are handled as anonymous.

---

## 7. Capability C — Outbound control

### 7.1 Responsibility split
| Step | Owner |
|---|---|
| Eligibility, DNC/suppression, consent, calling window, retry limits, wrong-party status | **Campaign module (05) + policy engine** |
| CLI selection, capacity, rate, spend admission | **Admission Controller** |
| Place the call, report progress, report outcome | **Adapter** |

The adapter **must refuse** a dial command that lacks a valid `policy_decision_ref` (see 7.2). This makes bypass structurally impossible.

### 7.2 Dial job (command) required fields
`dial_job_id`, `attempt_id`, `tenant_id`, `client_id`, `campaign_id`, `customer_ref`, `contact_point_ref`, `destination_e164`, `cli_number_id`, `workflow_id`, `workflow_version`, `conversation_id` (new or continuing), `policy_decision_ref` (id + version + timestamp of the dial-time check), `window_expires_at` (do not start after this time), `max_ring_seconds`, `machine_detection_mode`, `recording_policy_ref`, `disclosure_policy_ref`, `max_call_duration`, `cost_budget`, `idempotency_key`.

**Dial-time re-check:** suppression, DNC, wrong-party, consent and window are evaluated again at the moment of dialling, not only at list load or scheduling. `policy_decision_ref` must be fresh (configurable maximum age, default short).

### 7.3 Admission control
A dial job is admitted only if **all** of the following hold:
1. Carrier rate (calls per second) headroom for the provider account
2. Carrier concurrent-call headroom
3. Voice Engine concurrent-session headroom, including a reserved buffer for inbound
4. Tenant fair-share and per-campaign concurrency limit
5. Tenant/campaign spend and daily attempt caps not exceeded
6. Selected CLI is `active`, approved and not quarantined
7. Destination is permitted by the country/route policy (no premium-rate or unapproved destinations)

Rejected jobs return a reason code and are re-queued by the Campaign module; the adapter never queues silently.

**Inbound protection:** a configurable share of AI capacity is reserved for inbound; outbound yields first under load.

### 7.4 Outcomes
Each attempt ends with exactly one **telephony outcome** and one **end reason**:

| Outcome | Typical end reasons |
|---|---|
| `not_placed` | policy_refused, capacity_refused, invalid_destination, cli_invalid, spend_cap, window_expired, provider_error |
| `no_answer` | ring_timeout |
| `busy` | busy |
| `rejected` | declined, blocked_by_callee |
| `failed` | network_error, number_unreachable, congestion, provider_error |
| `answered_machine` | voicemail_detected, fax_detected |
| `answered_human` | customer_hangup, ai_completed, transfer_completed, max_duration, system_disconnect, error |
| `answered_unknown` | detection_inconclusive |

Telephony outcomes are **not** business dispositions. The Workflow Executor assigns dispositions such as wrong party or DNC request (Blueprint §5.2). Telephony may supply hints.

### 7.5 Answering-machine handling
Configurable per dial job: `off`, `detect_and_end`, `detect_and_leave_approved_message`, `detect_and_flag`. For regulated collections, default is `detect_and_end` or an approved non-disclosing message. No account information may be spoken to a machine or unverified party. Detection confidence and time-to-decision are recorded; low confidence is treated as `answered_unknown` and handled by the workflow's wrong-party-safe opening.

### 7.6 Disconnection vs hang-up
The contract distinguishes `customer_hangup`, `ai_completed`, `system_disconnect`, `network_error` and `max_duration` so that Blueprint §5.3 (interruption and continuation) can act correctly.

---

## 8. Capability D — Call control (live)

### 8.1 Commands
| Command | Purpose | Notes |
|---|---|---|
| `answer` | Accept an inbound call | |
| `reject` | Refuse inbound before answer | With reason |
| `hangup` | End a call or leg | Reason required |
| `transfer_blind` | Hand customer to destination | Destination validated against allow-list |
| `transfer_warm` | Connect destination, hand over summary, then drop AI leg | Requires destination answer confirmation |
| `play_audio` | Play a stored/approved asset (e.g. disclosure, hold, closing) | Used when AI is unavailable |
| `send_dtmf` / `collect_dtmf` | Send or capture tones | Collected digits are treated as sensitive input |
| `hold` / `resume` | Hold a leg | |
| `record_start` / `record_pause` / `record_resume` / `record_stop` | Recording control | See 8.3 |
| `get_call` | Current state and legs | |

All commands carry `call_id`, `idempotency_key`, `requested_by` (workflow node, supervisor, system) and reason.

### 8.2 Transfer rules
- Destinations come from a **managed allow-list** per tenant: type (PSTN, SIP URI, queue endpoint), hours, capacity, fallback.
- Required result states: `transfer_initiated`, `transfer_answered`, `transfer_failed` (+ reason), `transfer_abandoned`.
- On failure the customer is not left unattended: fall back to callback offer, ticket, or message per workflow.
- Context passed to the receiving party is delivered out of band (case/ticket/summary), referenced by `case_id`; the contract does not require the carrier to carry data.
- Do not promise callback times that are not reserved (Blueprint §6.4).

### 8.3 Recording control
- Recording policy is supplied per call (`recording_policy_ref`): required, optional, forbidden; notice/consent requirement; pause-on-sensitive-input.
- Recording states follow Blueprint §10.1 (`NOT_STARTED → ACTIVE → PAUSED/RESUMED → STOPPED → UPLOAD_PENDING → STORED | UPLOAD_FAILED`).
- Recording is started only after any required notice has been played, if the policy demands it.
- Pause/resume events are timestamped so audio offsets align with the interaction timeline.
- Dual-channel (customer and AI separated) is preferred for QA and evaluation.
- Storage destination and retention are governed by Module 09, not the adapter. The adapter delivers audio to the configured platform ingest endpoint and confirms integrity.

---

## 9. Capability E — Media bridge

Transports call audio between the carrier and the Voice Engine.

### 9.1 Contract
| Aspect | Requirement |
|---|---|
| Direction | Full duplex, per call, per leg |
| Session setup | Platform issues a short-lived, single-use session token bound to `call_id`; the Voice Engine connects using it |
| Audio format | Declared at session setup (codec, sample rate, channels, frame size). Adapter converts if needed; the Voice Engine receives a normalised format |
| Timing | Frames carry sequence numbers and timestamps for latency measurement |
| Barge-in | Bridge must support immediate **flush of queued outbound audio** on command so interruptions are honoured |
| Marks | Voice Engine can place markers in outbound audio and receive confirmation when played, to know what the customer actually heard |
| Backpressure | Bridge signals buffer state; the Voice Engine must not over-produce |
| Reconnect | Short media-session loss triggers a bounded reconnect window; beyond it, the call follows failure behaviour (see 12) |
| Security | Encrypted transport; token expiry; no reuse across calls |

### 9.2 Transport options (adapter-internal)
| Option | Characteristics |
|---|---|
| **Carrier-hosted audio stream** | Fastest to integrate; telephony-grade audio; carrier in the audio path |
| **SIP media to platform media server** | More control and flexibility; more infrastructure to operate |

The contract is identical for both. The choice is per deployment and can differ by tenant. **Decision required before build:** which is used for the first release (see Section 17).

### 9.3 Audio-heard accounting
The bridge must report what portion of each AI utterance was actually played before interruption. This is required for accurate transcripts, QA of delivered audio (AIRIS) and cost accuracy.

---

## 10. Canonical call state model

```
                     ┌──────────── inbound ────────────┐
                     ▼                                 │
 INBOUND_RECEIVED → ROUTING → ANSWERED ─────────────────┤
                                                        ▼
 DIAL_REQUESTED → ADMITTED → DIALLING → RINGING → CONNECTED → (IN_AI) ⇄ (TRANSFERRING) → ENDED
        │             │          │          │
        ▼             ▼          ▼          ▼
   NOT_PLACED     NOT_PLACED   FAILED   NO_ANSWER / BUSY / REJECTED
```

State rules
- States only move forward except `IN_AI ⇄ TRANSFERRING ⇄ ON_HOLD`.
- `ENDED` is terminal and carries outcome, end reason, duration breakdown and who ended it.
- Out-of-order carrier callbacks are reordered by provider sequence/timestamp; a late event that conflicts with a terminal state is logged as an anomaly, not applied.

---

## 11. Canonical events

### 11.1 Envelope
Every event includes: `event_id`, `event_type`, `occurred_at` (with UTC offset), `received_at`, `tenant_id`, `client_id`, `conversation_id`, `call_id`, `attempt_id` (outbound), `leg_id`, `trace_id`, `provider`, `provider_ref`, `sequence`, `status`, `reason_code`, `policy_version`, `redacted_payload_ref`, `schema_version`.

Events are idempotent by `event_id` (derived deterministically from provider event identity) and ordered by `sequence` per call.

### 11.2 Event catalogue
| Group | Events |
|---|---|
| Number | `number.ordered`, `number.regulatory_updated`, `number.activated`, `number.bound`, `number.unbound`, `number.restricted`, `number.quarantined`, `number.released`, `number.reputation_changed` |
| Route | `route.created`, `route.updated`, `route.activated`, `route.retired` |
| Inbound | `call.inbound_received`, `call.route_resolved`, `call.inbound_decision`, `call.answered` |
| Outbound | `dial.requested`, `dial.admitted`, `dial.refused`, `dial.started`, `call.ringing`, `call.connected`, `call.machine_detected`, `call.no_answer`, `call.busy`, `call.rejected`, `call.failed` |
| Live control | `call.hold`, `call.resume`, `call.dtmf_collected`, `call.audio_played` |
| Transfer | `transfer.initiated`, `transfer.answered`, `transfer.failed`, `transfer.abandoned`, `transfer.completed` |
| Recording | `recording.started`, `recording.paused`, `recording.resumed`, `recording.stopped`, `recording.upload_pending`, `recording.upload_confirmed`, `recording.upload_failed` |
| Media | `media.session_opened`, `media.session_degraded`, `media.session_lost`, `media.session_restored`, `media.first_audio_received`, `media.first_audio_sent` |
| End | `call.ended` (with outcome, end reason, durations, ender) |
| Usage | `telephony.usage_estimated`, `telephony.usage_reconciled`, `telephony.usage_discrepancy` |
| Anomaly | `telephony.event_out_of_order`, `telephony.event_duplicate`, `telephony.unreconciled_call`, `telephony.callback_rejected` |

### 11.3 Reason codes
Maintain a controlled vocabulary mapped from each adapter. Unmapped carrier codes map to `provider_other` and retain the raw code in the restricted payload. The vocabulary is versioned; adapters declare which version they implement.

---

## 12. Resilience and failure behaviour

| Scenario | Required behaviour |
|---|---|
| Carrier API error on dial | Mark `not_placed/provider_error`; retry only if idempotent and within window; surface to Campaign module |
| Dial sent, no callback received | Reconciler queries carrier; resolves state within a defined timeout; emits `unreconciled_call` if unresolved |
| Duplicate callbacks | Deduplicate silently; count in telemetry |
| Voice Engine fails mid-call | Play approved holding/closing message, attempt transfer to fallback destination, else end politely and trigger callback; preserve checkpoint for resume |
| Media session lost | Bounded reconnect; then the same fallback as above |
| Platform ingress down | Carrier-level fallback per DID (human number or recorded message) |
| Carrier outage | Alert; stop admissions for that provider; inbound fallback; (future) route to secondary adapter |
| Circuit breaker | Per provider account; opens on sustained error rate; admission controller stops dialling |
| Duplicate dial command | Idempotency key returns original result; no second call |
| Retry storm | Capped retries with jitter; dead-letter queue for failed commands |
| Spend runaway | Per-tenant and per-campaign hard caps; automatic admission stop; kill switch |

**Kill switches** (all audited): per call, per campaign, per number, per tenant, per provider.

---

## 13. Security and tenancy

| Area | Requirement |
|---|---|
| Credentials | Stored in the secrets vault; scoped keys preferred over master credentials; per-tenant; rotation and revocation without downtime; never in logs, traces or model prompts |
| Tenant isolation | Separate provider accounts or sub-accounts per tenant where the carrier supports it; otherwise strict logical isolation with tenant tags on every resource |
| BYO carrier | Tenant may supply its own account or trunk; the same contract applies; platform verifies ownership and capabilities on connection |
| Inbound callback authentication | Verify carrier signature or equivalent for every callback; reject and log failures; replay protection; per-tenant endpoint identity |
| Destination controls | Country allow-list, premium/high-risk destination block, per-tenant destination policy |
| Number/CLI controls | Approval workflow, ownership evidence, audit |
| Data minimisation | Mask numbers in logs; recordings and transcripts follow Module 09 policy; DTMF for sensitive data is never logged |
| Audit | Immutable record of who changed numbers, routes, destinations, caps, credentials |
| RBAC | Separate permissions for: view numbers, order/release numbers, approve CLI, edit routes, edit destinations, set spend caps, trigger kill switch |
| Data residency | Each adapter declares where signalling, media and recordings are processed; policy can restrict adapters by region |

---

## 14. Usage, cost and reconciliation

1. **Estimated cost** emitted at `call.ended` from the effective rate card (by destination, number type, date).
2. **Reconciled cost** obtained later from carrier usage/billing records; discrepancy beyond tolerance emits `telephony.usage_discrepancy`.
3. **Cost lines:** connected time per leg, per-destination rate, number rental, recording, transfer leg, messaging (future), carrier fees and taxes where separable.
4. Every cost event links to `tenant_id`, `campaign_id`, `call_id`, `attempt_id`, `number_id`, `provider`, `rate_card_version`.
5. **Unreconciled detector:** any call without an end event or without a cost match after a configurable period is flagged.
6. Rate cards are effective-dated, currency-aware and never hard-coded into adapters.

---

## 15. Telemetry and KPIs

| Metric | Dimensions | Use |
|---|---|---|
| Answer rate; short-call rate; ring time | number, campaign, carrier route, hour | Reputation, data quality |
| Dial-to-connect failure mix | reason, destination prefix | Separate data issues from carrier issues |
| Attempts blocked at dial time | reason (DNC, window, suppression, consent) | Compliance evidence |
| Post-answer silence; time-to-first-audio (both directions) | tenant, adapter, media transport | Customer-perceived dead air |
| Barge-in flush latency | adapter | Interruption quality |
| Admission rejections | reason (carrier, AI, tenant, spend) | Capacity planning |
| Concurrency used vs limit | carrier, AI, tenant | Capacity planning |
| Callback latency, duplicates, out-of-order rate | adapter | Ingress health |
| Transfer success, time-to-answer, abandon rate | destination | Handoff quality |
| Recording upload success and delay | tenant | Evidence integrity |
| Cost per connected minute / per connected call / per right-party contact | tenant, campaign, route | FinOps |
| Unreconciled calls; cost discrepancy | adapter | Financial integrity |

Alerts: number quarantined; sustained answer-rate drop; carrier error spike; capacity saturation; spend at 80/100% of cap; unreconciled calls over threshold; callback authentication failures.

---

## 16. Adapter specification

### 16.1 Adapter responsibilities
Implement every mandatory capability or declare it unsupported; authenticate to the carrier; translate commands; authenticate and normalise callbacks; map reason codes; expose rate and concurrency limits; support reconciliation queries; obey idempotency.

### 16.2 Capability declaration
Each adapter publishes a machine-readable declaration:
`adapter_id`, `provider`, `version`, `contract_version`, `supported_countries`, `number_types`, `inbound`, `outbound`, `machine_detection`, `warm_transfer`, `recording` (modes, dual-channel), `dtmf`, `media_transports`, `max_cps`, `max_concurrency`, `supports_subaccounts`, `supports_byo`, `callback_authentication_method`, `data_regions`, `reason_code_version`.

The platform uses this to refuse unsupported configurations at design time (e.g. a workflow requiring warm transfer on an adapter without it).

### 16.3 Conformance test suite (adapter acceptance)
An adapter is certified only when it passes, against a sandbox:
1. Inbound answer, reject, overflow, fallback
2. Outbound answered, no-answer, busy, rejected, failed, machine
3. Idempotent duplicate dial command
4. Duplicate and out-of-order callbacks
5. Forged/replayed callback rejected
6. Transfer blind, warm, failed, abandoned
7. Recording start/pause/resume/stop and upload confirmation
8. Barge-in flush and audio-heard accounting
9. Media session loss and recovery
10. Dial refused without valid policy decision
11. Spend cap and kill switch enforcement
12. Cost estimate and reconciliation
13. Number order, regulatory pending, bind, quarantine, release
14. Credential rotation without dropped calls
15. Load test at declared CPS and concurrency

---

## 17. First release scope and decisions

### 17.1 First release (Twilio adapter)
**In:** number registry and binding, inbound routing with fallback, single and batched outbound with admission control and dial-time policy check, canonical events and states, blind transfer to a managed destination, recording control, media bridge on one transport, cost estimate events, spend caps and kill switches, callback authentication, conformance tests 1–14.

**Out:** number pools and automatic rotation, warm transfer (unless required by the first use case), advanced machine-detection tuning, secondary-carrier failover, messaging channels.

**Exit gate:** a real inbound call and a real outbound call, each with complete canonical event trace, cost record, a dial attempt refused by policy, a failed transfer handled gracefully, and a forged callback rejected.

### 17.2 Decisions required before build
| # | Decision | Options | Impact |
|---|---|---|---|
| 1 | Media transport for first release | Carrier-hosted stream vs SIP to platform media server | Latency, cost, ops burden |
| 2 | Provider account topology | Sub-account per tenant vs BYO vs shared | Isolation, cost attribution, white-label |
| 3 | Number ownership of record | Daythree reseller vs client direct | Regulatory documents, portability |
| 4 | Malaysian number types and regulatory requirements | To be validated with carrier | Go-live feasibility |
| 5 | Human transfer destinations (first release) | Client PBX/SIP/PSTN numbers | Transfer design, warm vs blind |
| 6 | Machine-detection policy | End, approved message, flag | Disclosure risk |
| 7 | Peak concurrency and CPS targets | Per tenant and platform | Admission control sizing |
| 8 | Recording consent and notice policy | Per client | Recording start sequence |
| 9 | Data residency constraints | Per client | Adapter region selection |
| 10 | Spend cap defaults and kill-switch authority | Tenant admin vs platform operations | Financial risk |

---

## 18. Risks and mitigations

| Risk | Impact | Mitigation |
|---|---|---|
| Carrier lacks required Malaysian numbers/CLI rules | Cannot go live | Validate early; contract allows another adapter |
| Provider concepts leak into upstream modules | Lock-in | Conformance suite and code review gate on canonical vocabulary only |
| Dead air under load | Customer harm | Inbound reservation, admission control, play-and-end fallback |
| Number reputation decay | Falling answer rates, higher cost | Reputation tracking, quarantine, volume ramps |
| Compliance bypass via adapter | Regulatory exposure | Mandatory `policy_decision_ref`, dial-time re-check, audit |
| Cost runaway | Financial loss | Hard caps, kill switches, alerts |
| Callback forgery | Fraud, data corruption | Signature verification, replay protection |
| Recording gaps | Evidence loss | Upload confirmation, retry, `upload_failed` alerting |
| Event loss or disorder | Wrong state, wrong billing | Sequencing, dedupe, reconciler |
| Over-engineering v1 | Delay | Strict scope in 17.1 |

---

## 19. Open items and assumptions

- This specification assumes the carrier supplies authenticated callbacks, bidirectional audio and recording; confirm per adapter.
- Carrier limits, pricing, Malaysian coverage and regulatory steps have not been verified here and must be confirmed with the provider.
- Legal and regulatory requirements (calling hours, recording consent, caller-ID presentation, data protection) must be confirmed by each client's compliance team and encoded as policy, not adapter logic.
- Schema formats (event serialisation, API style) are intentionally not fixed in v0.1; they will be defined in the next revision once decisions in 17.2 are made.

---

## 20. Change control

| Version | Date | Change |
|---|---|---|
| 0.1 | 17 Oct 2026 | Initial draft |

Contract changes follow semantic versioning: additive changes are minor; any change to meaning of existing fields, states or events is major and requires adapter re-certification.
