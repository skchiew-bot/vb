# First Release — Build Plan, Sequencing, Team Split and Sandbox Tests

**Scope:** Integration Hub (telephony and AI providers), number registry and routing, outbound pools, callback landing, AI provider settings and failover, metering foundations
**Version:** 0.1 (draft for planning review)
**Date:** 17 October 2026
**Related:** `telephony-contract-spec.md` · `number-registry-routing-data-model.md` v0.3 · `platform-requirements-v0.1.md` v0.2 · `event-schemas-v0.1.md`
**Status:** Planning document. Durations are indicative and assume the team in §5; they must be re-estimated after Sprint 0 spikes. Carrier and AI-provider behaviour, limits, pricing and Malaysian/Singapore regulatory rules have not been verified and are validated in Sprint 0.

---

## 1. Release definition

### 1.1 Release 1 objective
An administrator can connect **Twilio and Telnyx** and **ElevenLabs and OpenAI** from Settings, activate numbers with exclusive inbound/outbound roles, run **randomised outbound** across both carriers, receive **inbound** calls routed to a workflow stub, handle **callbacks** to outbound numbers, and have the voice stack **fail over** between the two AI providers — with every action traced, metered and auditable.

### 1.2 In scope (Release 1)
| Area | Included |
|---|---|
| Foundation | Tenancy, roles, secrets vault, audit, change requests, event bus with the standard envelope, kill switches, spend caps |
| Telephony | Telephony Gateway, adapter SPI, **Twilio adapter**, **Telnyx adapter**, canonical states/events, conformance suite |
| Numbers | Registry, lifecycle, regulatory tracking, exclusive roles, Daythree ownership and client allocation, admin-only purchase, CLI approval |
| Routing | Routes, versions, snapshots, destinations, schedules, carrier-level fallback, rollback |
| Outbound | Dial jobs with policy decision reference, admission control, pools spanning both carriers, constrained random selection, reputation and quarantine, answering-machine handling modes |
| Callback | Callback landing policy (message-and-redirect, message-and-end, transfer) |
| Contactability | Contact-point status with evidence and confidence (Campaign module) |
| Call control | Blind transfer, recording control and upload confirmation, DTMF (non-sensitive), play audio |
| AI providers | ElevenLabs and OpenAI settings, key storage, Test Connection, stack profiles, health detection, failover and failback, manual override |
| Voice (minimum) | Media bridge to the voice engine on the chosen transport, a small set of stock talents, a basic listening profile (noise/echo handling, non-speech sounds ignored) |
| Metering | Usage events for telephony and AI, credit ledger (Daythree AI), BYOAI admin-fee ledger, spend thresholds, reconciliation jobs |
| UI | Provider settings, number inventory, pools, routes, health and spend boards, failover controls |
| Test | Sandbox environment, call-sink test harness, conformance and parity suites |

### 1.3 Out of scope (Release 2+)
Talent creation with GPT, autonomous pre-recording and council, emotion register and stitching continuity, full pronunciation dictionary and name pre-check, warm transfer, `ai_with_context` callbacks, SMS/WhatsApp, Cantonese and Tamil production support, sticky caller-ID, advanced machine-detection tuning, additional carriers. (Release 1 reserves the data and events for these.)

### 1.4 Release 1 exit criteria
1. Both carriers pass the full conformance suite and the parity suite (§8).
2. Randomised outbound across Twilio and Telnyx numbers with full trace; no policy-refused dial reaches a carrier.
3. Inbound and callback calls resolved from a signed snapshot, with carrier-level fallback proven.
4. Simulated ElevenLabs outage and OpenAI outage each fail over (both directions) with recorded seam and no dropped call in the test scenarios.
5. Credit and admin-fee ledgers reconcile to usage events within agreed tolerance.
6. Kill switches and spend caps stop traffic within the agreed time.
7. Security review passed; no credentials or sensitive values appear in logs, events or UI after save.
8. Pilot readiness review signed by product, security and compliance.

---

## 2. Sprint 0: de-risking spikes (must finish before committing dates)

| # | Spike | Question answered | Output | Owner |
|---|---|---|---|---|
| S0-1 | **Telnyx validation** | Can Telnyx supply Malaysian and Singapore numbers for our roles? Regulatory documents? Caller-ID rules? Rate limits? Recording, machine detection, transfer, callback authentication? | Capability declaration draft; go/no-go; contingency | Telephony lead |
| S0-2 | **Twilio validation** | Same questions for Twilio, plus sub-account model and key scoping | Capability declaration draft | Telephony lead |
| S0-3 | **Media transport** | Carrier-hosted stream vs SIP to our media server: latency, barge-in flush, audio quality, ops burden, cost, parity across both carriers | Recommendation and decision | Voice/AI lead + SRE |
| S0-4 | **AI provider capability matrix** | For each language and function, what do ElevenLabs and OpenAI actually deliver (recognition accuracy, voice quality, latency, streaming, expressiveness, pronunciation control)? | Language × function × provider scorecard; phasing | Voice/AI lead |
| S0-5 | **Failover feasibility** | Can the same persona be approximated on both providers (talent twin)? Cost and latency of switching mid-call | Twin scorecard; mid-call switch measured | Voice/AI lead |
| S0-6 | **Compliance baseline** | Calling hours, recording consent, caller-ID presentation, callback expectations, data residency for Malaysia and Singapore | Policy pack v0 for Module 01 | Compliance + product |
| S0-7 | **Platform choices** | Hosting regions, event transport, vault, database, CI/CD, observability stack | Architecture decision records | Tech lead + SRE |
| S0-8 | **Sandbox accounts and budgets** | Accounts, test numbers, spend caps, trial restrictions on both carriers and both AI providers | Working credentials in vault; budget approved | SRE + Finance |

**Decisions needed by end of Sprint 0:** media transport; second-carrier go/no-go; hosting/regions; compliance baseline; BYOAI fee base and credit scope (Requirements Q11, Q12); mid-call vs between-call failover (Q8).

---

## 3. Workstreams

| WS | Name | Deliverables | Depends on |
|---|---|---|---|
| WS1 | **Platform foundation** | Tenancy and RBAC, vault integration, audit log (hash chained), change-request workflow, event bus and envelope validation, dead-letter store, feature flags | S0-7 |
| WS2 | **Telephony Gateway and adapter SPI** | Command validation, idempotency, adapter routing, event normaliser, admission controller, reconciler, kill switches | WS1 |
| WS3 | **Twilio adapter** | Implements SPI; inbound/outbound/transfer/recording/media; callback authentication; capability declaration | WS2 |
| WS4 | **Telnyx adapter** | Same, to parity | WS2 (starts once SPI stable) |
| WS5 | **Number registry and routing** | Provider accounts, numbers, lifecycle, regulatory, allocation, CLI approval, routes/versions/snapshots, destinations, schedules, fallback, callback landing | WS1, WS2 |
| WS6 | **Campaign dial engine (minimum)** | Dial jobs, policy gate hook, pools and constrained random selection, retries, contact-point status, reputation | WS2, WS5 |
| WS7 | **AI provider hub and failover** | ElevenLabs and OpenAI connectors, settings, Test Connection, stack profiles, health probes, failover/failback, overrides | WS1 |
| WS8 | **Voice engine minimum** | Media bridge client, stock talents, basic listening profile, turn events, hand-off to workflow stub | S0-3, WS3/WS4, WS7 |
| WS9 | **Metering and FinOps** | Usage events, rate cards, credit ledger, admin-fee ledger, spend policies, reconciliation | WS1, WS2, WS7 |
| WS10 | **Admin UI** | Settings (providers, keys, test), inventory, pools, routes, health/spend boards, failover controls | APIs from WS2, WS5, WS7, WS9 |
| WS11 | **Quality engineering** | Call-sink harness, fake adapter, conformance and parity suites, load and failure injection, security tests | Runs from Sprint 1 |
| WS12 | **Security and compliance** | Threat model, vault review, redaction audit, penetration test, policy pack | Continuous |

---

## 4. Sequencing

Two-week sprints. Indicative total of about 22 weeks including Sprint 0, assuming the §5 team; re-baseline after Sprint 0.

| Sprint | Weeks | Focus | Key deliverables | Gate |
|---|---|---|---|---|
| **0** | 1–3 | Spikes and setup | S0-1 to S0-8; repo, CI, environments skeleton; decisions logged | Decisions signed off |
| **1** | 4–5 | Foundation | Tenancy, vault, audit, change requests; event bus and envelope validation; adapter SPI definition; **fake adapter**; call-sink harness v0 | Fake adapter passes SPI tests |
| **2** | 6–7 | Registry core and Twilio inbound | Provider accounts, number lifecycle, roles, allocation; Twilio onboarding flow (validate, discover, configure, verify); Twilio inbound answer/reject/fallback; canonical events | Inbound test call on Twilio with full trace |
| **3** | 8–9 | Twilio outbound and routing | Dial jobs, admission control, policy-gate enforcement; Twilio outbound outcomes; routes, versions, snapshot compiler and rollback; destinations and schedules | Outbound test call, refusal path, snapshot rollback |
| **4** | 10–11 | Telnyx adapter | Telnyx onboarding and calls to parity; conformance suite on both; parity tests | Both pass conformance 1–10 |
| **5** | 12–13 | Pools, callback, contactability | Pools across carriers, constrained random, reputation and quarantine; callback landing; contact-point status; transfer and recording control | Randomised outbound across both carriers; callback handled |
| **6** | 14–15 | AI providers and failover | ElevenLabs and OpenAI settings, key storage, Test Connection; stack profiles; health probes; failover/failback; overrides | Simulated outage fails over in both directions |
| **7** | 16–17 | Voice minimum and metering | Media bridge on chosen transport; stock talents; basic listening profile; usage events; credit and admin-fee ledgers; spend caps | End-to-end AI call with metering |
| **8** | 18–19 | UI completion and integration | Admin UI complete; end-to-end scenarios; reconciliation jobs; kill switch drills | Scenario suite passes |
| **9** | 20–21 | Hardening | Load tests, failure injection, security test, redaction audit, runbooks, dashboards and alerts | Exit criteria 1–7 met |
| **10** | 22 | Pilot readiness | Go/no-go review; pilot plan with internal numbers then one limited client | Exit criterion 8 |

Parallel tracks: WS10 (UI) starts Sprint 2 against stubbed APIs; WS11 (test) runs from Sprint 1; WS12 (security) reviews each sprint; WS9 (metering) starts Sprint 3.

Critical path: Sprint 0 decisions → SPI/Gateway → Twilio adapter → admission/dial → Telnyx parity → pools → media bridge → end-to-end AI call. The **Telnyx validation (S0-1)** and **media transport (S0-3)** are the two items most likely to move the plan.

---

## 5. Team split

Assumed team of about 14 engineers plus specialists. Adjust if smaller; the largest compromise would be sequencing Telnyx after the first release.

| Squad | Roles | Owns | Interfaces provided |
|---|---|---|---|
| **Product & architecture** | Product owner (you), solution architect | Scope, decisions, acceptance, contract documents | Decision log, acceptance sign-off |
| **Platform** | 3 backend engineers, SRE | WS1, WS9 foundations, environments, vault, event bus | Tenancy, events, audit, secrets, metering APIs |
| **Telephony** | 4 backend engineers (2 per adapter), telephony specialist | WS2, WS3, WS4, media bridge integration | Telephony Gateway commands/events, adapter SPI |
| **Registry & Campaign** | 2–3 backend engineers | WS5, WS6 | Number/route/pool APIs, dial job intake, contact-point status |
| **Voice & AI** | 2 backend/ML engineers, voice QA specialist | WS7, WS8 | AI provider hub, stack/failover APIs, voice turn events |
| **Frontend** | 2 engineers, designer | WS10 | Admin UI |
| **Quality engineering** | 2 SDET, test-bench engineer | WS11 | Harness, suites, reports, gates |
| **Security & compliance** | Security engineer (part-time), compliance advisor | WS12 | Threat model, reviews, policy pack |
| **FinOps** | Analyst (part-time) | Rate cards, reconciliation rules, margin | Rate cards, fee/credit rules |

### 5.1 Ownership rules
- **Contract-first:** schemas and the adapter SPI are owned by the architect; changes go through change control (event-schemas §2.5). Squads build against mocks until contracts are stable.
- **One owner per workstream**; shared services have named maintainers.
- **Definition of done** for every story: tests including conformance where applicable; events emitted and schema-validated; audit entries; metrics and alerts; runbook note; no secrets or raw PII in logs.
- **Cross-squad dependencies** tracked on a single board; a blocked dependency of more than two days is escalated.

### 5.2 Cadence
Sprint demos with real calls; weekly risk review; decision log updated within one day of any decision; end-of-sprint gate review against §4.

---

## 6. Environments and sandbox strategy

| Environment | Purpose | Telephony | AI providers | Data |
|---|---|---|---|---|
| **Dev** | Daily development | Fake adapter, replayed carrier payloads | Recorded/stubbed responses | Synthetic |
| **Sandbox** | Real-carrier and real-provider testing | Real Twilio and Telnyx test accounts with low spend caps; test numbers; destinations restricted to the call-sink allow-list | Real ElevenLabs and OpenAI accounts with low caps | Synthetic only |
| **Staging** | Prod-like, load and failure injection | Production-grade accounts, restricted traffic | Production-grade, capped | Synthetic |
| **Pilot / Production** | Live | Live accounts | Live accounts | Real, under policy |

Sandbox guardrails
- **Destination allow-list:** only call-sink numbers and internal numbers can be dialled. Anything else is refused in code, not by convention.
- **Hard spend caps** per carrier and per AI provider; automatic stop and alert.
- **Synthetic customers only.** No real customer data in sandbox or staging.
- **Trial/sandbox tier limits:** trial or unverified accounts commonly restrict destinations, inject announcements, or cap features. Plan on upgraded accounts with small balances. Confirm each carrier's behaviour in S0-8.
- Credentials for sandbox are separate from production and rotated on schedule.

### 6.1 Test harness components (WS11)
| Component | Function |
|---|---|
| **Call sink** | A set of platform-owned test endpoints (numbers or SIP endpoints) that answer or refuse calls on command: answer after N seconds, ring-no-answer, busy, reject, voicemail greeting, fax tone, silence, noise, DTMF, long call, drop mid-call, speak scripted audio, inject interruption audio at set offsets |
| **Call source** | Generates inbound calls to platform numbers with scripted audio, withheld/anonymous callers, bursts |
| **Fake adapter** | Implements the SPI with deterministic behaviour and failure injection for dev and CI |
| **Callback replay** | Replays recorded real carrier callbacks, including duplicates, out-of-order and forged (bad signature) variants |
| **AI provider simulator** | Stubs ElevenLabs/OpenAI endpoints with controllable latency, errors, rate limits, empty/corrupt audio, timeouts |
| **Fault injector** | Kills provider connections, delays webhooks, drops media, exhausts quotas, rotates credentials mid-test |
| **Metering verifier** | Compares emitted usage/credit/fee events to carrier and provider usage records |
| **Trace assertions** | Asserts the expected event chain per scenario (event-schemas §6) |

---

## 7. Conformance and parity approach

- The 15 conformance tests in the Telephony Contract (§16.3) are the acceptance suite for **each** adapter.
- The **parity suite** runs every scenario against Twilio, Telnyx and the fake adapter and compares the resulting event chains and field values. Differences must be explained by the capability declaration or fixed.
- A scenario passes only when the **event trace**, **state transitions**, **cost estimate** and **audit entries** all match expectations, not just when the call connects.

---

## 8. Sandbox test plan: Twilio and Telnyx

"Both" means the test runs separately on Twilio and on Telnyx, and results are compared. IDs are stable for the test report.

### 8.1 Onboarding and accounts (O)
| ID | Scenario | Expected | Carriers |
|---|---|---|---|
| O-01 | Enter valid credentials; run validation, discovery, configuration, verification | Account `active`; capability declaration saved; key never retrievable after save | Both |
| O-02 | Enter invalid or over-scoped key | Rejected with specific step and remedy; nothing stored beyond error | Both |
| O-03 | Re-run onboarding on an active account | No changes (idempotent) | Both |
| O-04 | Import existing numbers; assign roles | Numbers in registry with correct role and state | Both |
| O-05 | Rotate credentials during an active call | Call unaffected; new key in use; audit and approval recorded | Both |
| O-06 | Revoke credentials | Provider account moves to `revoked`; admissions stop; alert raised | Both |
| O-07 | Drift: change configuration directly at the carrier | `drift_detected`; no silent overwrite; resolved through change request | Both |

### 8.2 Numbers and routing (N)
| ID | Scenario | Expected | Carriers |
|---|---|---|---|
| N-01 | Admin orders a number within spend allowance | Order, regulatory state, activation path; non-admin refused | Both |
| N-02 | Order beyond spend allowance | Refused with reason; audit entry | Both |
| N-03 | Activate inbound line without verified carrier fallback | Blocked | Both |
| N-04 | Activate outbound line without callback landing policy | Blocked | Both |
| N-05 | Try to use an inbound line in a pool or as CLI | Rejected at write time and at snapshot compile | Both |
| N-06 | Assign one inbound number to two routes or two clients | Rejected | Both |
| N-07 | Publish route, then roll back | New snapshot active within target time; previous behaviour restored; decision trace shows version | Both |
| N-08 | Release a number with active assignments | Refused until assignments end; cooling period enforced after release | Both |

### 8.3 Inbound (I)
| ID | Scenario | Expected | Carriers |
|---|---|---|---|
| I-01 | Call a routed inbound number | `call.inbound_received` → `route_resolved` → `inbound_decision(ACCEPT_AI)`; workflow stub reached; clean end | Both |
| I-02 | Outside business hours | Configured after-hours action | Both |
| I-03 | Over route capacity | Overflow or play-and-end; no dead air | Both |
| I-04 | Platform ingress disabled | Carrier-level fallback reaches human number or recorded message | Both |
| I-05 | Withheld caller number | Treated as anonymous; no context lookup | Both |
| I-06 | Forged callback (bad signature) | Rejected; `callback_rejected` event; no state change | Both |
| I-07 | Duplicate and out-of-order callbacks | Deduplicated; ordered by sequence; anomaly counted | Both |
| I-08 | Unknown dialled number | Tenant unknown-number action; anomaly event | Both |

### 8.4 Outbound (D)
| ID | Scenario | Expected | Carriers |
|---|---|---|---|
| D-01 | Dial to call-sink: answered human, clean hang-up | Full chain from `dial.requested` to `call.ended`; outcome `answered_human`; cost estimated | Both |
| D-02 | Sink: no answer / busy / reject / network fail | Correct telephony outcome and end reason for each | Both |
| D-03 | Sink: voicemail greeting | Machine detected per mode; correct action; confidence logged | Both |
| D-04 | Dial without valid policy decision | Refused before carrier contact (`dial.refused`) | Both |
| D-05 | Suppressed/DNC contact at dial time | Refused with reason; no carrier call | Both |
| D-06 | Outside calling window | Refused; rescheduled by campaign | Both |
| D-07 | Duplicate dial command (same idempotency key) | One call only | Both |
| D-08 | Carrier accepts dial but callback never arrives | Reconciler resolves state in time; `unreconciled_call` if not | Both |
| D-09 | Destination not on allow-list (sandbox) or in blocked prefix | Refused in code | Both |
| D-10 | Max duration reached | Call ended with `max_duration` | Both |

### 8.5 Pools and providers (P)
| ID | Scenario | Expected | Carriers |
|---|---|---|---|
| P-01 | Pool of numbers across Twilio and Telnyx; 500 attempts | Distribution near expectation given weights; each call routed via holder provider; selection logged | Cross-provider |
| P-02 | Quarantine a number mid-run | Excluded within target time; others continue | Cross-provider |
| P-03 | Per-number hourly ceiling | Number skipped at ceiling | Cross-provider |
| P-04 | One provider degraded or rate-limited | Weight redistributed; no stalled queue | Cross-provider |
| P-05 | All eligible numbers excluded | `pool.selection_failed`, `cli_unavailable`, rescheduled | Cross-provider |
| P-06 | Campaign configured with fixed number | Always that number; pool ignored | Both |
| P-07 | Rotation does not reset suppression or retry counters | Counters unchanged after caller-ID changes | Cross-provider |
| P-08 | CLI approval expired | Number excluded; refusal if none remain | Both |

### 8.6 Callback landing and contactability (C)
| ID | Scenario | Expected | Carriers |
|---|---|---|---|
| C-01 | Call back an outbound line (message-and-redirect) | Landing message played; redirect info; callback matched to earlier attempt as hint; events logged | Both |
| C-02 | Callback from a suppressed contact | No collections content; policy respected | Both |
| C-03 | Callback outside hours / over capacity | Configured behaviour | Both |
| C-04 | Repeated rejects across different caller-IDs | `rejected_pattern` with evidence and confidence; never stated as fact | Both |
| C-05 | Wrong-number statement (simulated) | Contact suppressed; status updated | Both |

### 8.7 Transfer, recording, DTMF, media (T/M)
| ID | Scenario | Expected | Carriers |
|---|---|---|---|
| T-01 | Blind transfer to allow-listed destination | `transfer.initiated` → `answered` → `completed` | Both |
| T-02 | Transfer to non-listed destination | Refused | Both |
| T-03 | Transfer target busy/no answer | `transfer.failed`; fallback action; customer not left unattended | Both |
| T-04 | Recording start, pause for sensitive input, resume, stop | States and offsets correct; sensitive segment absent; upload confirmed with checksum | Both |
| T-05 | Recording upload fails | Retries; `upload_failed` alert after limit | Both |
| T-06 | DTMF non-sensitive menu; sensitive input | Digits only for menu; none in events for sensitive | Both |
| M-01 | Media session open; first audio both ways | Latency recorded; within budget | Both |
| M-02 | Barge-in: injected speech during bot audio | Audio flushed; time-to-stop within target; `audio_heard_ratio` correct | Both |
| M-03 | Media session loss and recovery | Bounded reconnect; fallback if exceeded | Both |
| M-04 | Echo and noise injection | No self-interruption beyond threshold; non-speech sounds ignored | Both |

### 8.8 Resilience and security (R/S)
| ID | Scenario | Expected | Carriers |
|---|---|---|---|
| R-01 | Carrier API errors on dial | Retry within idempotency; circuit breaker opens at threshold; admissions stop | Both |
| R-02 | Full carrier outage (simulated) | Alert; inbound fallback; outbound shifts to the other carrier's numbers where pool spans | Cross-provider |
| R-03 | Kill switch per call, campaign, number, tenant, provider | Traffic stops within target time; audited | Both |
| R-04 | Spend cap reached | Admissions stop; alert; live calls finish | Both |
| R-05 | Retry storm / queue backlog | Backpressure and dead-letter handling; no duplicate calls | Both |
| S-01 | Secrets scan on logs, events, UI, API responses | Zero credentials or OTP values | Both |
| S-02 | Permission matrix | Each role limited to its permissions; high-risk changes need second approver | Both |
| S-03 | Tenant isolation | Tenant A cannot see or use Tenant B numbers, routes, events | Both |
| S-04 | Replay attack on callbacks | Rejected | Both |

### 8.9 AI providers and failover (A)
| ID | Scenario | Expected | Providers |
|---|---|---|---|
| A-01 | Enter ElevenLabs and OpenAI keys; Test Connection | Capabilities tested; latency and permission gaps reported; key masked | Both |
| A-02 | Stack A (ElevenLabs voice) → simulated outage before call | New calls start on standby; `stack_selected(reason=standby)` | EL→OAI and OAI→EL |
| A-03 | Mid-call failure | Retry once; failover at turn boundary using talent twin; seam flagged for QA; customer experience within budget | Both directions |
| A-04 | Both providers down | Transfer, callback or polite close per policy | — |
| A-05 | Flapping provider | Hysteresis prevents oscillation; minimum dwell time respected | Both |
| A-06 | Failback | Only after stable period; new calls only by default | Both |
| A-07 | Manual override (force, drain, pause failover) | Honoured and audited | — |
| A-08 | Talent without twin | Warning at configuration; failover blocked for that talent | — |
| A-09 | Cross-mode failover without opt-in or credits | `failover_blocked`; no silent switch | — |
| A-10 | Spend cap on an AI provider | Per policy: alert, stop generation, or switch | Both |

### 8.10 Metering and FinOps (F)
| ID | Scenario | Expected | Carriers/Providers |
|---|---|---|---|
| F-01 | Telephony usage per call vs carrier usage records | Estimate within tolerance; discrepancy flagged beyond tolerance | Both carriers |
| F-02 | AI usage per turn vs provider usage | Reconciled; mode recorded | Both providers |
| F-03 | Credit consumption on Daythree AI tenant | Balance decrements; ledger links to usage and rate-card version | — |
| F-04 | BYOAI tenant | No credits consumed; admin fee accrues on agreed base | — |
| F-05 | Low balance and zero balance | Alerts; configured stop behaviour (per Q14) | — |
| F-06 | Rate-card change mid-period | New version applies forward only | — |
| F-07 | Failover call spanning two stacks | Each turn priced on the stack that served it | — |

### 8.11 Parity (X) and performance (L)
| ID | Scenario | Expected |
|---|---|---|
| X-01 | Run D-01…D-10 and I-01…I-08 on Twilio, Telnyx and fake adapter | Event chains and fields equivalent; differences explained by capability declaration |
| X-02 | Run P-01 with providers swapped | No behavioural change besides provider labels |
| L-01 | Ramp to declared CPS and concurrency on each carrier (staging) | No dropped admissions beyond rule; admission controller holds limits |
| L-02 | Time-to-first-audio and turn latency by carrier and transport | Within agreed p50/p95 budgets (set in Sprint 0) |
| L-03 | Snapshot compile and load time at target number/route counts | Within target; rollback time met |
| L-04 | Event bus throughput and lag at peak | Lag within target; no loss |

---

## 9. Quality gates per sprint

| Gate | Requirement |
|---|---|
| Sprint gate | Demo of real calls; sprint scenarios green on fake adapter and on at least one real carrier |
| Contract gate | No schema or SPI change without impact list and consumer sign-off |
| Security gate | Secrets scan clean; permission tests pass |
| Cost gate | Sandbox spend within budget; unexpected usage investigated |
| Release gate | Exit criteria 1–8 (§1.4) |

---

## 10. Risks and mitigations

| Risk | Impact | Mitigation |
|---|---|---|
| Telnyx cannot supply required Malaysia/Singapore numbers or caller-ID | Cross-provider pool not possible at launch | S0-1 go/no-go; contingency carrier shortlist; pool design already provider-agnostic |
| Media transport choice wrong | Rework, latency | S0-3 spike; keep behind contract; measure on both carriers |
| Regulatory documents slow number activation | Delays pilot | Start document collection in Sprint 0; use imported numbers for testing |
| AI provider language quality below expectation | Language rollout slips | S0-4 matrix; phase languages; per-language fallback |
| Mid-call failover audible or unreliable | Poor customer experience | Twin scoring, turn-boundary switching, bridging phrases, QA flag; between-call option if unacceptable |
| Sandbox cost overrun | Budget | Hard caps, allow-listed destinations, daily review |
| Adapter divergence | Lock-in, inconsistent data | Conformance and parity suites as release gates |
| Scope creep from Release 2 features | Delay | Strict scope table; reserve fields only |
| Single point of failure in gateway | Outage | Redundant deployment, snapshots, carrier-level fallback, kill-switch drills |
| Team smaller than assumed | Longer timeline | Sequence Telnyx after first-release go-live; keep contracts stable |
| Billing rules unsettled | Rework in WS9 | Decide Q11/Q12/Q14 by Sprint 6 |

---

## 11. Decisions needed and by when

| Decision | Needed by |
|---|---|
| Media transport | End of Sprint 0 |
| Second carrier go/no-go (Telnyx) | End of Sprint 0 |
| Hosting regions, event transport, vault | End of Sprint 0 |
| Compliance baseline for Malaysia and Singapore | End of Sprint 0 (v0), refine by Sprint 4 |
| Mid-call vs between-call failover | Sprint 1 |
| Admin-fee base (metered vs invoice) and minimum fee | Sprint 5 |
| Do credits cover telephony | Sprint 5 |
| Zero-credit behaviour | Sprint 6 |
| Cross-mode failover (BYOAI → Daythree AI) opt-in policy | Sprint 6 |
| Emotion register set | Before Release 2 planning |
| Sampling of listening events | Sprint 7 |

---

## 12. Assumptions

- The team and cadence in §5; funding for sandbox spend.
- Both carriers and both AI providers can be accessed from the chosen hosting regions.
- A compliance owner is available for policy decisions; legal review of recording, caller-ID and callback practices is done by each client's compliance team.
- Event transport, serialisation and registry tooling are chosen in Sprint 0 and do not change the logical schemas.
- Release 1 serves a pilot (internal numbers, then one limited client) before broader launch.

---

## 13. Change control

| Version | Date | Change |
|---|---|---|
| 0.1 | 17 Oct 2026 | Initial first-release build plan |
