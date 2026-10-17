# Number Registry & Routing — Logical Data Model

**Module:** 02 Integration Hub → Telephony (supports 05 Campaign/Inbound Router)
**Version:** 0.2 (draft for design review)
**Date:** 17 October 2026
**Depends on:** Telephony Capability Contract v0.1 (`telephony-contract-spec.md`)
**Status:** Logical model. Physical schema, storage engine and serialisation are deferred. Carrier-specific facts and regulatory rules must be validated per provider and per client.

---

## 1. Purpose and design goals

Define the system of record for **numbers (DIDs/CLIs), provider accounts, routes, destinations, schedules and policies** that govern inbound answering and outbound caller-ID selection, plus the audit and reputation data needed to operate them.

| Goal | Implication |
|---|---|
| Vendor-neutral | No carrier concepts as keys; carrier references are attributes of an adapter binding |
| Multi-tenant, white-label ready | Every row is tenant-scoped; isolation enforceable at the data layer |
| Answer fast | Route resolution must work from a cached, versioned snapshot; the database is never on the answer path |
| Safe change | Routes and policies are versioned, approved, effective-dated and reversible |
| Auditable | Every change is attributable, diffable and replayable |
| Keyed on the customer, not the number | Suppression, consent and retry state never live on or depend on a number record |
| Cost-aware | Every number and route links to a rate card and to cost attribution |
| Event-driven | Registry changes emit events; read models are projections |

### 1.1 Decisions applied in v0.2

| # | Decision | Model consequence |
|---|---|---|
| D1 | **Daythree is owner of record for every number.** | `owner_of_record` is fixed to `daythree`. Numbers are held on Daythree-managed provider accounts and **allocated** to a tenant/client as a right of use. Regulatory submissions are made by Daythree. BYO / client-owned numbers and `import_number` are out of scope for v1. |
| D2 | **Number pools are modelled, but any campaign or route may use a fixed number instead.** | A campaign's outbound CLI source is exactly one of: a fixed number or a pool. Pools are an optional capability per tenant; `fixed` is the default strategy. |
| D3 | **No shared inbound numbers.** | An inbound number has exactly one active route, belonging to exactly one tenant and client. Enforced at write time, publish validation and snapshot compile. |

Assumption to confirm (flagged, not decided): outbound CLI numbers are also allocated to a single client, so that reputation and complaints are attributable. Pools are therefore client-scoped, never cross-client.

---

## 2. Conceptual overview

```
Tenant ─┬─ Client ─┬─ Campaign ──────────────┐
        │          │                          │ (uses)
        │          └─ Route ─ RouteVersion ─ RouteRule ─ Destination
        │                         │                          ▲
        ├─ ProviderAccount ─ AdapterBinding                  │
        │         │                                          │
        │         └─ Number ─┬─ NumberAssignment ── Route / Campaign / Pool
        │                    ├─ NumberRegulatory
        │                    ├─ NumberCliApproval
        │                    ├─ NumberReputationSnapshot
        │                    └─ NumberLifecycleEvent
        ├─ NumberPool ─ NumberPoolMember
        ├─ Schedule ─ ScheduleWindow / ScheduleException
        ├─ Destination (transfer / overflow / fallback)
        ├─ CallerIdPolicy, CapacityPolicy, SpendPolicy
        └─ RoutingSnapshot (compiled, immutable, cached)
```

**Two planes**
- **Authoring plane (relational, transactional):** the editable entities above.
- **Serving plane (compiled, cached):** `RoutingSnapshot` — an immutable, versioned, denormalised bundle used for answer-time resolution and outbound CLI selection.

---

## 3. Conventions

- **Keys:** surrogate opaque IDs (e.g. `number_id`, `route_id`); never the E.164 string as a key. Natural uniqueness is enforced by constraints.
- **Tenancy:** `tenant_id` on every table. `client_id` where a client sub-scope applies. Composite uniqueness includes `tenant_id`.
- **Phone numbers:** stored in E.164 with a normalised form; a separate masked display form; indexed by a keyed hash for lookup without exposing raw values in secondary systems.
- **Time:** all timestamps are stored with UTC offset; business schedules store an IANA timezone.
- **Versioning:** versioned entities have an immutable version row plus a mutable pointer to the current/published version.
- **Soft retirement:** records are retired, not deleted; deletion follows retention policy by data class.
- **Audit columns (all entities):** `created_at`, `created_by`, `updated_at`, `updated_by`, `row_version` (optimistic concurrency), `change_request_id`.
- **Status vocabulary:** controlled enumerations, versioned; unknown values rejected.

---

## 4. Entity catalogue

### 4.1 Provider and adapter

#### `ProviderAccount`
A tenant's account or sub-account at a carrier.
| Field | Notes |
|---|---|
| `provider_account_id` (PK) | |
| `tenant_id`, `client_id?` | Scope |
| `adapter_id` | Which adapter implements it |
| `provider` | Carrier name (descriptive) |
| `provider_ref` | Carrier's account identifier |
| `ownership` | `platform_managed` \| `tenant_owned` (BYO) |
| `credential_ref` | Pointer to secrets vault — **never** the secret |
| `credential_version`, `credential_rotated_at` | |
| `data_region` | Signalling/media/recording region |
| `capability_declaration_ref` | Adapter capability snapshot |
| `limits` | Declared CPS, concurrency, daily caps |
| `status` | `pending_verification`, `active`, `suspended`, `revoked` |
| `verified_at`, `verified_by` | Ownership/capability verification |

Constraint: at most one active account per (`tenant_id`, `adapter_id`, `provider_ref`).

#### `AdapterBinding`
Maps a platform resource to the carrier-side resource, keeping carrier vocabulary out of core entities.
| Field | Notes |
|---|---|
| `binding_id` (PK) | |
| `resource_type` | `number`, `route`, `destination` |
| `resource_id` | Platform ID |
| `provider_account_id` | |
| `provider_ref` | Carrier resource ID |
| `provider_config_hash` | Detects drift between platform intent and carrier configuration |
| `sync_status` | `in_sync`, `pending`, `drift`, `failed` |
| `last_synced_at`, `last_error` | |

Drift detection compares desired vs actual carrier configuration on a schedule and raises an alert; it never silently overwrites without a change record.

---

### 4.2 Numbers

#### `Number`
| Field | Notes |
|---|---|
| `number_id` (PK) | |
| `holder_account_id` | Daythree-managed provider account that holds the number (see `ProviderAccount`) |
| `allocated_tenant_id`, `allocated_client_id?` | Right-of-use allocation; null while in the unallocated Daythree stock. Exactly one tenant/client at a time |
| `allocation_state` | `stock`, `allocated`, `reclaiming` |
| `e164`, `e164_hash` | Globally unique among non-released records (Daythree owns all numbers, so uniqueness is platform-wide, not per tenant) |
| `country`, `number_type` | geographic, mobile, toll-free, shared-cost, other |
| `capabilities` | voice_in, voice_out, sms, whatsapp |
| `provider_account_id` | Current holder |
| `purpose` | `inbound`, `outbound_cli`, `both` |
| `lifecycle_state` | See 5.1 |
| `restriction_reason` | If restricted/quarantined |
| `owner_of_record` | Fixed `daythree` (D1). Field retained so a future BYO model does not need a schema change; writes of any other value are rejected in v1 |
| `acquisition_type` | `ordered`, `ported_in` (`imported` disabled in v1) |
| `rate_card_ref` | Rental and usage rates |
| `monthly_cost`, `currency` | Estimated rental |
| `activated_at`, `released_at` | |
| `labels` | Free tags (non-semantic) |

#### `NumberRegulatory`
Regulatory/identity requirements the carrier or regulator imposes.
| Field | Notes |
|---|---|
| `regulatory_id` (PK), `number_id` | |
| `requirement_type` | address, identity document, business registration, use-case declaration, other |
| `status` | `not_required`, `required`, `submitted`, `under_review`, `approved`, `rejected`, `expired` |
| `evidence_ref` | Pointer to restricted document store |
| `submitted_at`, `decided_at`, `expires_at`, `rejection_reason` | |
| `submitted_by`, `reviewed_by` | |

A number cannot reach `active` while any required `NumberRegulatory` is not `approved`. Expiry triggers a pre-expiry alert and, if lapsed, `restricted`.

#### `NumberCliApproval`
Authorises a number to be **presented** to callees.
| Field | Notes |
|---|---|
| `cli_approval_id` (PK), `number_id` | |
| `client_id` | Which client may present it |
| `scope` | `all_campaigns`, `campaign_list`, `vertical`, `route_only` |
| `campaign_ids?` | When scoped |
| `ownership_evidence_ref` | Daythree's holding evidence for the number (D1) |
| `client_authorisation_ref` | Client's written authorisation for calls to be made in its name from this number (letter of authority or equivalent) |
| `verification_status` | Carrier-side verification where required |
| `approved_by`, `approved_at`, `expires_at` | Approver must hold the approval permission and differ from requester |
| `status` | `pending`, `approved`, `revoked`, `expired` |

Rule: a number is eligible as outbound CLI only if it has an `approved`, unexpired `NumberCliApproval` covering the campaign.

#### `NumberAssignment`
Binds a number to a purpose. A number can hold several assignments over time but only one **active inbound** assignment at a time.
| Field | Notes |
|---|---|
| `assignment_id` (PK), `number_id` | |
| `assignment_type` | `inbound_route`, `outbound_campaign`, `outbound_pool` |
| `target_id` | `route_id`, `campaign_id` or `pool_id` |
| `effective_from`, `effective_to` | |
| `status` | `scheduled`, `active`, `ended` |
| `change_request_id` | |

Constraints
- Unique active inbound assignment per number (no two routes answer one DID at the same time).
- **No shared inbound (D3):** the route targeted by an inbound assignment must belong to the same tenant and client as the number's allocation. An inbound number is never referenced by routes of two clients, and a route rule cannot hand off to another client's route.
- An assignment target must belong to the number's allocated tenant/client; moving a number between clients requires `reclaiming` → `stock` → re-allocation with all assignments ended first.
- An outbound CLI assignment requires an approved `NumberCliApproval`.
- Assignment overlap rules are enforced at write time and again at snapshot compile time.

#### `NumberPool` and `NumberPoolMember`
Groups outbound CLIs for selection.
| `NumberPool` | Notes |
|---|---|
| `pool_id`, `tenant_id`, `client_id` | |
| `name`, `purpose` | |
| `selection_strategy` | `fixed`, `round_robin`, `weighted`, `reputation_weighted`, `local_presence` (reserved) |
| `max_calls_per_number_per_day/hour` | Volume ceiling to protect reputation |
| `min_reputation` | Below this a member is skipped |
| `status` | |

| `NumberPoolMember` | Notes |
|---|---|
| `pool_id`, `number_id` | Composite key |
| `weight`, `priority` | |
| `status` | `active`, `paused`, `quarantined` |

**Fixed vs pool (D2):** a campaign's `CallerIdPolicy` names either a fixed `number_id` (via an `outbound_campaign` assignment) or a `pool_id`, never both. `selection_strategy = fixed` is the default and the only strategy enabled in the first release; other strategies are gated by a per-tenant capability flag. Pools are scoped to one client.

Rotation changes which number is presented only. It has **no effect** on suppression, consent, windows or retry counters (those key on customer/contact point).

#### `NumberReputationSnapshot`
Periodic rollup per outbound number.
| Field | Notes |
|---|---|
| `number_id`, `window_start`, `window_end`, `granularity` | hour/day |
| `attempts`, `answered`, `short_calls`, `rejected`, `blocked_indicators` | Counts |
| `answer_rate`, `short_call_rate`, `reject_rate` | Derived |
| `volume_rank`, `score`, `flags` | Computed |
| `scoring_version` | Reproducibility |

Used for quarantine decisions and pool selection. Append-only.

#### `NumberLifecycleEvent`
Immutable history of state changes.
`event_id`, `number_id`, `from_state`, `to_state`, `reason_code`, `actor` (user/system/rule), `evidence_ref`, `occurred_at`, `change_request_id`.

---

### 4.3 Destinations and schedules

#### `Destination`
Where a call can be sent: transfer, overflow, fallback.
| Field | Notes |
|---|---|
| `destination_id` (PK), `tenant_id`, `client_id` | |
| `destination_type` | `pstn`, `sip_uri`, `queue_endpoint`, `voicemail`, `recorded_message`, `ai_route` |
| `address` / `address_ref` | E.164 or SIP URI; sensitive, access-controlled |
| `display_name`, `purpose` | e.g. collections specialist, hardship team, complaints |
| `schedule_id` | When it is staffed/available |
| `capacity` | Concurrent limit (optional) |
| `health_status`, `last_health_check_at` | Optional probe |
| `fallback_destination_id` | Next if unavailable |
| `verification_status` | Ownership/consent of the destination verified |
| `status` | |

Destinations form an allow-list. A transfer to anything not in the allow-list is refused by the Telephony Gateway. Fallback chains must be acyclic and terminate at a `recorded_message`, `voicemail` or callback/ticket action.

#### `Schedule`, `ScheduleWindow`, `ScheduleException`
| Entity | Fields |
|---|---|
| `Schedule` | `schedule_id`, `tenant_id`, `name`, `timezone` (IANA), `status` |
| `ScheduleWindow` | `schedule_id`, `day_of_week`, `start_time`, `end_time` (multiple per day allowed) |
| `ScheduleException` | `schedule_id`, `date_or_range`, `type` (closed, special hours), `reason`, `source` (holiday calendar, manual) |

Schedules are evaluated in the schedule's timezone. Customer-timezone **calling windows** for outbound compliance belong to the policy engine (Module 05), not this schedule, and are not duplicated here.

---

### 4.4 Routes

#### `Route`
Stable identity of an inbound handling rule.
| Field | Notes |
|---|---|
| `route_id` (PK), `tenant_id`, `client_id` | |
| `name`, `description`, `purpose` | |
| `current_published_version_id` | Pointer |
| `status` | `draft`, `active`, `suspended`, `retired` |

#### `RouteVersion` (immutable once approved)
| Field | Notes |
|---|---|
| `route_version_id` (PK), `route_id`, `version_no` | |
| `state` | `draft`, `validated`, `approved`, `published`, `superseded`, `rolled_back` |
| `workflow_ref` | `workflow_id` + `workflow_version` **or** alias (e.g. `production`) |
| `schedule_id` | Default hours |
| `capacity_limit` | Max concurrent AI sessions for this route |
| `language_hint` | |
| `recording_policy_ref`, `disclosure_policy_ref` | |
| `failure_action` | Used when AI/platform unavailable |
| `overflow_action` | Used when over capacity |
| `after_hours_action` | |
| `effective_from`, `effective_to` | |
| `validation_report_ref` | Result of pre-publish checks |
| `approved_by`, `approved_at`, `published_by`, `published_at` | Approver differs from author |
| `parent_version_id` | For diff and rollback |

#### `RouteRule`
Ordered rules inside a version; first match wins, with an explicit default.
| Field | Notes |
|---|---|
| `rule_id` (PK), `route_version_id`, `position` | |
| `match_conditions` | Structured: caller segment/hint, anonymous, time/schedule state, language, priority list match, custom attributes from context lookup |
| `decision` | `ACCEPT_AI`, `TRANSFER`, `PLAY_AND_END`, `REJECT`, `OVERFLOW` |
| `target` | `workflow_ref` and/or `destination_id` and/or `message_asset_ref` |
| `priority_class` | e.g. standard, priority — affects capacity reservation |
| `capacity_override` | Optional |
| `reason_code` | Recorded in `call.inbound_decision` |

Validation (pre-publish): exactly one default rule; no unreachable rules; every referenced workflow version exists and is published; every destination is active and its fallback chain terminates safely; every schedule resolves; no rule can route to a retired resource; decision/target combinations are legal.

#### `RouteFallback`
Carrier-level fallback independent of the platform being reachable.
| Field | Notes |
|---|---|
| `number_id`, `fallback_destination_id` | The destination the carrier uses if the platform webhook/endpoint fails |
| `fallback_message_ref` | Optional preface |
| `last_pushed_at`, `sync_status` | Must be in sync before a number is activated for inbound |

A number cannot be activated for inbound without a verified `RouteFallback`.

---

### 4.5 Policies

#### `CallerIdPolicy` (outbound CLI selection)
`policy_id`, `tenant_id`, `client_id`, `scope` (tenant/client/campaign), `allowed_pool_ids`, `allowed_number_types`, `require_local_country`, `forbid_withheld`, `min_reputation`, `max_per_number_daily/hourly`, `version`, `status`.

#### `CapacityPolicy`
`policy_id`, `tenant_id`, `scope` (tenant/client/campaign/route/provider_account), `max_concurrent_calls`, `max_cps`, `inbound_reserved_share`, `fair_share_weight`, `burst_allowance`, `degradation_mode` (reject, queue-limited, play-and-end), `version`.

#### `SpendPolicy`
`policy_id`, `tenant_id`, `scope`, `period` (day/month), `soft_threshold`, `hard_cap`, `currency`, `action_on_soft` (alert), `action_on_hard` (stop admissions), `kill_switch_authority`, `version`.

#### `DestinationPolicy`
`policy_id`, `tenant_id`, `allowed_countries`, `blocked_prefixes`, `premium_rate_blocked`, `exception_list`, `version`.

All policies are versioned, effective-dated and referenced by version from route versions, pools and snapshots so any call can state which policy version governed it.

---

### 4.6 Compiled serving data

#### `RoutingSnapshot`
Immutable bundle compiled from published entities.
| Field | Notes |
|---|---|
| `snapshot_id` (PK), `tenant_id`, `snapshot_no` | Monotonic |
| `compiled_at`, `compiled_by` (system or user trigger), `source_change_requests` | |
| `contents_hash` | Integrity |
| `contents_ref` | Stored bundle (routes, rules, destinations, schedules, number-to-route map, pools, CLI eligibility, policy versions) |
| `validation_status` | Must pass the same checks as publish |
| `activated_at`, `deactivated_at` | |
| `rollback_of_snapshot_id` | If created by rollback |

Behaviour
- Gateway nodes hold the active snapshot in memory; answer-time lookup is key-based on `e164_hash` → `route_version_id` → ordered rules.
- A new snapshot becomes active atomically; nodes keep the previous one for instant rollback.
- If a node cannot load a new snapshot it continues on the last good one and raises an alert. It never serves an unvalidated snapshot.
- Each inbound call records `snapshot_id`, `route_version_id`, `rule_id`, `policy_versions` in `call.inbound_decision`.

---

### 4.7 Change control and audit

#### `ChangeRequest`
`change_request_id`, `tenant_id`, `type` (order number, release number, approve CLI, edit route, edit destination, change caps, rotate credential, kill switch), `requested_by`, `requested_at`, `risk_class`, `diff_ref`, `approvers_required`, `approvals[]`, `status`, `scheduled_for`, `applied_at`, `rolled_back_at`, `reason`.

Risk classes drive approval: low (label edits), medium (route edits, destination edits), high (CLI approval, number release, spend cap changes, credential rotation, kill-switch reset). High risk requires a second approver.

#### `AuditEvent`
Append-only, tamper-evident: `audit_id`, `tenant_id`, `actor`, `actor_type`, `action`, `resource_type`, `resource_id`, `before_ref`, `after_ref`, `change_request_id`, `occurred_at`, `source_ip/session`, `hash_chain_prev`, `hash`.

#### `KillSwitchState`
`kill_switch_id`, `scope_type` (call, campaign, number, tenant, provider_account), `scope_id`, `state` (engaged/released), `engaged_by`, `engaged_at`, `reason`, `released_by`, `released_at`. Gateways check engaged switches on every admission.

---

## 5. State machines

### 5.1 Number lifecycle
```
searched → ordered → regulatory_pending → active ⇄ restricted → released
                          │                  │
                          └→ rejected        └→ quarantined → active | released
```
| Transition | Guard |
|---|---|
| `ordered → regulatory_pending` | Carrier indicates requirements outstanding |
| `regulatory_pending → active` | All required `NumberRegulatory` approved; carrier confirms; for inbound: `RouteFallback` in sync |
| `active → restricted` | Manual hold, regulatory lapse, spend event, legal hold |
| `active → quarantined` | Reputation rule breach or complaint |
| `quarantined → active` | Manual review with recorded reason |
| `* → released` | High-risk change request approved; no active assignments; no calls in the last N minutes; retention rules satisfied |

Released numbers are retained as records and their E.164 is not reusable by another tenant within a configured cooling period (avoids misrouted legacy callers and misattributed reputation).

### 5.2 Route version lifecycle
`draft → validated → approved → published → superseded`, with `rolled_back` as a recorded outcome. Only `published` versions are compiled into snapshots. Rollback re-publishes a prior approved version as a new snapshot; it never edits history.

### 5.3 Adapter binding sync
`pending → in_sync`; `in_sync → drift → pending`; any → `failed`. A number cannot be activated while its binding is not `in_sync`.

---

## 6. Resolution algorithms (logical)

### 6.1 Inbound resolution
1. Normalise dialled number; compute `e164_hash`.
2. Look up the active snapshot: number → active inbound assignment → `route_version`.
3. If no match: apply tenant-level unknown-number action (play-and-end), emit anomaly.
4. Evaluate rules in order against call context (caller hint, anonymous flag, schedule state, capacity state, kill switches).
5. Apply the first matching rule; if capacity is exceeded apply overflow; if the target is unavailable apply the failure action.
6. Emit `call.route_resolved` and `call.inbound_decision` with snapshot, version, rule and reason.

### 6.2 Outbound CLI selection
1. Receive an approved dial job (policy decision already attached).
2. Determine candidate set: campaign's allowed pools/numbers → filter by `active`, not `quarantined`, approved `NumberCliApproval` covering the campaign, `CallerIdPolicy`, destination-country rules.
3. Remove numbers over volume ceilings or below minimum reputation.
4. Apply pool strategy; if the dial job is a retry or continuation, prefer continuity only where policy allows and never to circumvent limits.
5. If the candidate set is empty: refuse with `cli_unavailable`; the Campaign module reschedules.
6. Record the selected `number_id`, pool, strategy and policy versions on the attempt.

---

## 7. Integrity constraints (summary)

1. Tenant-scoped uniqueness of active E.164.
2. Exactly one active inbound assignment per number.
3. No `active` number with unapproved required regulatory items.
4. No `active` inbound number without a synced carrier fallback.
5. No outbound CLI use without an approved, unexpired, correctly-scoped CLI approval.
6. A route version must pass full validation before approval; approver ≠ author.
7. Destination fallback chains are acyclic and terminate in a safe end state.
8. Published route versions and approved CLI approvals are immutable; changes create new versions.
9. Credentials are references only; no secrets in this model.
10. Suppression, consent, wrong-party and retry state are **not** attributes of any entity in this model.
11. Every mutation produces an `AuditEvent` linked to a `ChangeRequest`.
12. Released number E.164 cannot be reassigned within the cooling period.
13. `owner_of_record` is `daythree` for every number in v1.
14. A number has at most one allocated tenant/client; all its assignments, CLI approvals and pool memberships belong to that same tenant/client.
15. A campaign's CLI source is exactly one of a fixed number or a pool.
16. No inbound number is shared across routes, clients or tenants.

---

## 8. Events emitted by the registry

| Domain | Events |
|---|---|
| Provider | `provider_account.verified`, `.suspended`, `.credential_rotated`, `.limits_changed` |
| Number | `number.ordered`, `.regulatory_updated`, `.activated`, `.restricted`, `.quarantined`, `.released`, `.assignment_changed`, `.cli_approved`, `.cli_revoked`, `.reputation_changed`, `.drift_detected` |
| Pool | `pool.created`, `.member_added`, `.member_paused`, `.strategy_changed` |
| Route | `route.version_drafted`, `.validated`, `.approved`, `.published`, `.rolled_back`, `.retired` |
| Destination | `destination.created`, `.health_changed`, `.fallback_changed`, `.retired` |
| Policy | `policy.version_published` (per policy type) |
| Snapshot | `snapshot.compiled`, `.activated`, `.load_failed`, `.rolled_back` |
| Control | `killswitch.engaged`, `.released`, `spend.threshold_crossed`, `change_request.opened/approved/applied/rolled_back` |

Events carry the common envelope from the Telephony Contract (§11.1) plus `resource_type`, `resource_id`, `change_request_id`.

---

## 9. Access control (permissions)

| Permission | Typical holder |
|---|---|
| `number.view` | Operations, support |
| `number.order` / `number.import` | Telephony admin |
| `number.release` | Telephony admin + second approver |
| `cli.request` / `cli.approve` | Requester / compliance or client owner (separate people) |
| `route.edit` / `route.approve` / `route.publish` | Author / reviewer / release manager (separate) |
| `destination.edit` | Telephony admin; client owner for client destinations |
| `policy.edit.capacity/spend/destination/cli` | Platform ops / tenant admin by policy type |
| `killswitch.engage` | Operations, tenant admin |
| `killswitch.release` | Higher authority, audited |
| `credential.rotate` | Security admin |
| `audit.view` | Compliance, security |

Sensitive attributes (`address`, destination addresses, evidence references) are field-level restricted and masked by default.

---

## 10. Scale, performance and storage guidance

| Concern | Guidance |
|---|---|
| Answer-path latency | Memory-resident snapshot keyed by hashed number; no synchronous database call on answer |
| Snapshot size | Per-tenant snapshots; incremental compile; size budget with alerts |
| Reputation data | Time-partitioned; hourly rollups retained shorter than daily; raw call data lives in the interaction store |
| Audit/lifecycle | Append-only, partition by tenant and month; archived per retention policy |
| Hot paths | Number lookup by `e164_hash`; pool candidate selection (cached per campaign); destination health |
| Multi-region | Registry authoritative in a primary region; snapshots replicated; gateways serve from local regional copy; failover rules defined with data-residency constraints |
| Consistency | Authoring plane strongly consistent; serving plane eventually consistent with bounded staleness (alert if exceeded) |
| Concurrency | Optimistic concurrency via `row_version`; publishes serialised per tenant |
| Cardinality planning | Numbers per tenant, rules per route, pools per client — set and test limits, publish them |

---

## 11. Retention and privacy

| Data class | Retention guidance (to be set per client/legal) |
|---|---|
| Number, assignment, route versions | Life of tenant relationship plus audit period |
| CLI approvals and regulatory evidence | Per regulatory requirement; restricted store |
| Reputation snapshots | Hourly short, daily longer |
| Audit events | Per compliance policy; tamper-evident |
| Released number history | Retained for cooling period and audit |
| Destination addresses | Treated as personal/organisational data; masked in logs |

Customer phone numbers are **not** stored in this model. Only the tenant's own numbers and destinations are held here.

---

## 12. Telemetry and operational views

| View | Purpose |
|---|---|
| Number health board | State, reputation score, answer/short-call rates, volume vs ceiling, quarantine candidates |
| Regulatory tracker | Pending/expiring items by number, age, owner |
| Route inventory | Number → route → workflow version, last published, drift status |
| Routing decision audit | Per call: snapshot, version, rule, reason, overridden by capacity or kill switch |
| Destination health | Availability, transfer success, abandonment |
| Capacity and spend | Usage vs limits per tenant/campaign/provider |
| Change calendar | Scheduled changes, approvals pending, recent rollbacks |
| Drift and sync | Bindings out of sync, snapshot load failures |
| Cost per number | Rental plus usage by number and pool |

Alerts: regulatory expiry (T-30/T-7), quarantine, sustained answer-rate drop, drift detected, snapshot load failure, destination unavailable, spend threshold, kill switch engaged, route validation failure at compile.

---

## 13. Abuse and risk scenarios addressed

| Scenario | Control in this model |
|---|---|
| Unapproved caller-ID presented | CLI approval entity, scope, approver ≠ requester, expiry |
| Number rotation used to evade suppression or limits | Suppression keyed on customer; pool rotation affects only presentation; volume ceilings per number |
| Misrouted inbound calls after bad edit | Versioned routes, full validation, approval, instant snapshot rollback |
| Transfer to an unapproved external number | Destination allow-list and verification |
| Dead air when platform is down | Mandatory carrier-level fallback before activation |
| Hijacked or leaked credentials | Vault references, rotation entity, audit, high-risk approval |
| Silent carrier-side drift | Adapter binding drift detection |
| Number recycled to another tenant, misdirecting callers | Cooling period, retained history |
| Cost runaway | Spend policies, kill switch state, admission checks |
| Insider tampering | Hash-chained audit, separation of duties, change requests |

---

## 14. Traceability to the Telephony Contract

| Contract area | Data model support |
|---|---|
| §5 Number management | `Number`, `NumberRegulatory`, `NumberCliApproval`, `NumberAssignment`, `NumberPool`, `NumberReputationSnapshot`, `NumberLifecycleEvent` |
| §6 Inbound routing | `Route`, `RouteVersion`, `RouteRule`, `RouteFallback`, `Schedule*`, `Destination`, `RoutingSnapshot` |
| §7 Outbound admission and CLI | `CallerIdPolicy`, `CapacityPolicy`, `SpendPolicy`, `DestinationPolicy`, `NumberPool*` |
| §8 Transfer | `Destination`, fallback chains |
| §12 Resilience and kill switches | `KillSwitchState`, `RouteFallback`, snapshot rollback |
| §13 Security and tenancy | `ProviderAccount.credential_ref`, permissions, `AuditEvent`, `ChangeRequest` |
| §14 Cost | `rate_card_ref`, `monthly_cost`, spend policy links |
| §16 Adapters | `ProviderAccount`, `AdapterBinding` |

---

## 15. Open questions

1. ~~Number ownership of record~~ — **Resolved (D1):** Daythree. Still to define: offboarding policy (a client leaving cannot take the number unless the contract and carrier allow porting out), client authorisation wording, and whether Daythree can pass regulatory-holder obligations to clients where a regulator requires the end user's details.
2. ~~Pools in the first release~~ — **Resolved (D2):** pools modelled; fixed assignment allowed and default.
3. **Caller recognition lists** (VIP, priority, blocklists): are they route-rule conditions referencing an external list, or a new entity? Privacy impact because they hold customer numbers.
4. **Workflow reference by alias vs pinned version:** aliases allow fast fixes but weaken change control. Recommend pinned versions for regulated routes, aliases only for non-regulated.
5. **Destination verification:** how is consent/ownership of a client's transfer number evidenced?
6. **Number cooling period** length and whether reputation history follows a number across tenants (recommend no).
7. **Multi-region and data residency:** single primary region or per-region registries per tenant?
8. **Unknown-number policy:** default action for calls to a number with no active route.
9. ~~Shared numbers across clients~~ — **Resolved (D3):** forbidden. Open sub-question: may one client share one inbound number across its own campaigns/lines? Current model says yes only through a single route with rules, never through multiple routes.
11. **Outbound CLI sharing across clients:** assumed single-client allocation (see 1.1). Confirm.
12. **Number inventory ahead of demand:** Daythree stock holding cost vs speed of onboarding; who bears the rental for unallocated stock?
10. **Carrier-side fallback capability** varies by carrier; the adapter capability declaration must state what it can enforce.

---

## 16. Suggested build order

1. `ProviderAccount`, `AdapterBinding`, secrets reference, audit and change-request foundation
2. `Number`, lifecycle, regulatory, assignment (fixed inbound and fixed outbound CLI)
3. `Destination`, `Schedule`, `Route`/`RouteVersion`/`RouteRule`, validation, `RoutingSnapshot`
4. `RouteFallback`, `KillSwitchState`, policies (capacity, spend, destination, CLI)
5. `NumberCliApproval` workflow
6. Reputation snapshots, quarantine, pools (after first production traffic)

**Exit gate:** publish a route, take a real inbound call resolved from a snapshot with full decision trace; roll back to the prior version within one minute; refuse an unapproved CLI; fall back to a human destination when the platform is deliberately disabled; reproduce every change from the audit trail.

---

## 17. Change control

| Version | Date | Change |
|---|---|---|
| 0.1 | 17 Oct 2026 | Initial draft |
| 0.2 | 17 Oct 2026 | Applied decisions D1 (Daythree owner of record), D2 (pools modelled, fixed assignment allowed), D3 (no shared inbound); added allocation fields, client authorisation, constraints 13–16 |
