# Event Schemas — Telephony, Voice Stack, Registry and Metering

**Module scope:** 02 Integration Hub, 05 Campaign/Inbound Router, 07 Voice Engine, 09 Interaction/Recording, 11 SAIGE/FinOps (consumers), 10 AIRIS (consumer), 12 Observatory (producer and consumer)
**Version:** 0.1 (draft for design review)
**Date:** 17 October 2026
**Related:** `telephony-contract-spec.md` v0.1 · `number-registry-routing-data-model.md` v0.3 · `platform-requirements-v0.1.md` v0.2
**Status:** Logical schemas in tables. Serialisation format, transport technology and registry tooling are not fixed here. Sections marked **PENDING** depend on open decisions listed in §10.

---

## 1. Purpose

Every module that produces or consumes platform events must agree on exactly what each event contains. These schemas are the single reference for field names, types, requiredness, sensitivity and ordering so that:

- the Twilio and Telnyx adapters emit identical events for the same real-world occurrence;
- SAIGE, AIRIS, FinOps and the Observatory read one consistent history;
- audit, billing and compliance can reconstruct any call from events alone;
- changes can be introduced without breaking consumers.

---

## 2. Conventions

### 2.1 Naming
- `<domain>.<subject>_<past-tense-verb>`, lower case, snake_case after the dot. Examples: `call.connected`, `number.quarantined`, `voice.failover_completed`.
- Events describe something that **has happened**. Commands and requests are not events; a request that was refused is an event (`dial.refused`).
- One real-world occurrence produces one event. Derived or summarised information is a separate projection, not a duplicate event.

### 2.2 Standard envelope (all events)
| Field | Type | Req | Sens. | Notes |
|---|---|---|---|---|
| `event_id` | string, globally unique | Y | P0 | Deterministic from source identity so replays and duplicate callbacks produce the same ID |
| `event_type` | string | Y | P0 | From the catalogue in §5 |
| `schema_version` | string (major.minor) | Y | P0 | Version of the event's payload schema |
| `occurred_at` | timestamp with UTC offset | Y | P0 | When it happened at the source |
| `received_at` | timestamp with UTC offset | Y | P0 | When the platform accepted it |
| `producer` | string | Y | P0 | Module or adapter (for example `adapter.twilio`, `voice_engine`) |
| `producer_version` | string | Y | P0 | Build/adapter version |
| `tenant_id` | string | Y | P0 | |
| `client_id` | string | C | P0 | When scoped to a client |
| `campaign_id` | string | C | P0 | When applicable |
| `conversation_id` | string | C | P0 | Call-related events |
| `call_id` | string | C | P0 | Call-related events |
| `attempt_id` | string | C | P0 | Outbound call events |
| `leg_id` | string | C | P0 | Leg-specific events |
| `turn_id` | string | C | P0 | Turn-level voice events |
| `trace_id` | string | Y | P0 | End-to-end trace |
| `span_id` | string | O | P0 | |
| `causation_id` | string | O | P0 | `event_id` or command ID that caused this event |
| `sequence` | integer | C | P0 | Monotonic per `call_id` (or per resource for registry events) |
| `provider` | string | C | P0 | Carrier or AI provider name |
| `provider_ref` | string | C | P1 | Provider's identifier; indexed, never a primary key |
| `status` | enum | Y | P0 | `succeeded`, `failed`, `partial`, `info` |
| `reason_code` | string | C | P0 | From the controlled vocabulary (§3.4) |
| `policy_version` | string | C | P0 | Policy bundle in force |
| `config_version` | string | C | P0 | Workflow, route, snapshot, listening profile or stack version, as applicable |
| `redacted_payload_ref` | string | O | P0 | Pointer to restricted detail (access controlled) |
| `data_class` | enum | Y | P0 | Retention class (§2.6) |

Req: Y required · C conditional (rule stated) · O optional.

### 2.3 Sensitivity classes
| Class | Meaning | Handling |
|---|---|---|
| P0 | Operational, no personal or secret data | Normal logging and analytics |
| P1 | Identifiers that link to a person or account indirectly (provider refs, customer reference) | Pseudonymous references only; access controlled |
| P2 | Personal data (phone numbers, names, transcripts text) | Masked by default; stored only in restricted payload; field-level access |
| P3 | Secrets and regulated values (credentials, OTP, raw card/ID values, DTMF digits for sensitive input) | **Never in events.** Reference only, or omitted |

Phone numbers in events appear as masked form plus a keyed hash. Raw E.164 lives only in the restricted payload.

### 2.4 Ordering, delivery and deduplication
- **Delivery:** at least once. Consumers must be idempotent on `event_id`.
- **Order:** guaranteed per `call_id` by `sequence`; per resource for registry events. No global order.
- **Late or out-of-order events:** accepted and reordered by `sequence`; an event that conflicts with a terminal state is stored and also raises `telephony.event_out_of_order`.
- **Duplicates:** discarded silently; counted in telemetry.
- **Dead letter:** events that fail schema validation or consumption retries go to a dead-letter store with the reason; they are replayable.
- **Replay:** any consumer can rebuild its projection by replaying events; schemas must remain readable for the retention period.

### 2.5 Versioning
- Additive optional fields: **minor** version.
- Changing meaning, type, requiredness or removing a field: **major** version, and producers must dual-publish for a defined transition period.
- Every consumer declares the schema versions it supports. Unknown major versions are rejected into the dead-letter store, not guessed.
- Enumerations only grow within a major version; consumers must tolerate unknown values by treating them as `other`.

### 2.6 Retention classes (`data_class`)
| Class | Content | Guidance (set per client and law) |
|---|---|---|
| `ops_short` | Health pings, high-frequency telemetry | Days to weeks |
| `ops_standard` | Call lifecycle, routing decisions, provider events | Months |
| `audit` | Registry changes, credential events, approvals, kill switches | Years; tamper-evident |
| `billing` | Usage, cost and credit events | Per finance and tax requirements |
| `evidence` | Recording and QA evidence references | Per client policy and law |

---

## 3. Shared types and vocabularies

### 3.1 Identifiers
As defined in Blueprint §3.1 and Telephony Contract §4, plus: `number_id`, `route_id`, `pool_id`, `landing_policy_id`, `provider_account_id`, `ai_account_id`, `stack_profile_id`, `failover_group_id`, `talent_id`, `talent_version_id`, `asset_id`, `dictionary_version`, `listening_profile_id`, `council_review_id`, `credit_ledger_id`, `usage_event_id`, `contact_point_ref`.

### 3.2 Enumerations
| Name | Values |
|---|---|
| `direction` | `inbound`, `outbound`, `callback_landing` |
| `number_role` | `inbound_line`, `outbound_line` |
| `carrier` | `twilio`, `telnyx`, `other` (adapter-declared) |
| `ai_provider` | `elevenlabs`, `openai`, `other` |
| `ai_function` | `stt`, `llm`, `tts`, `realtime` |
| `ai_commercial_mode` | `daythree_ai`, `byoai` |
| `telephony_outcome` | `not_placed`, `no_answer`, `busy`, `rejected`, `failed`, `answered_machine`, `answered_human`, `answered_unknown` |
| `ended_by` | `customer`, `ai`, `system`, `transfer_target`, `operator`, `unknown` |
| `provider_health` | `active`, `degraded`, `unreachable`, `suspended`, `revoked` |
| `emotion_register` | Controlled set, to be fixed with the product owner (working set: `neutral_professional`, `warm`, `empathetic`, `apologetic`, `firm_calm`, `reassuring`) |
| `language` | `en`, `ms`, `zh_cmn`, `zh_yue`, `ta`, `en_singlish`, `en_manglish` (extendable) |
| `severity` | `info`, `warning`, `critical` |

### 3.3 Reusable object types
| Object | Fields |
|---|---|
| `Money` | `amount` (decimal string), `currency` |
| `PhoneRef` | `masked`, `hash`, `country`, `number_type?` |
| `Duration` | `seconds` (decimal) |
| `UsageQuantity` | `unit` (`second`, `minute`, `character`, `token_in`, `token_out`, `token_cached`, `request`, `call`), `quantity` (decimal) |
| `PolicyRef` | `policy_id`, `policy_version`, `decision`, `decided_at` |
| `ProviderCostRef` | `rate_card_id`, `rate_card_version`, `estimated`, `reconciled` |

### 3.4 Reason code vocabulary
Controlled, versioned (`reason_vocab_version` carried in the payload when codes change). Groups:
- **Dial refusal:** `policy_refused`, `suppressed`, `dnc`, `window_closed`, `consent_missing`, `capacity_refused`, `spend_cap`, `cli_unavailable`, `cli_not_approved`, `destination_blocked`, `kill_switch`, `window_expired`, `invalid_destination`, `provider_error`.
- **End reasons:** `customer_hangup`, `ai_completed`, `transfer_completed`, `max_duration`, `system_disconnect`, `network_error`, `ring_timeout`, `busy`, `declined`, `blocked_by_callee`, `voicemail_detected`, `fax_detected`, `detection_inconclusive`.
- **Provider/voice:** `timeout`, `rate_limited`, `quota_exhausted`, `auth_failed`, `latency_breach`, `error_rate_breach`, `empty_audio`, `corrupt_audio`, `provider_status_incident`, `manual_override`.
- **Other:** `provider_other` (raw code kept in restricted payload).

---

## 4. Event flow overview

```
Registry & provider changes ─────────────┐
Campaign (05) → dial.requested → admitted → started → ringing → connected
Inbound / callback landing ──────────────┤
Live call: media · turns · listening · voice stack · failover · transfer · recording
End: call.ended → telephony.usage_estimated → (later) usage_reconciled
Metering: ai.usage_recorded · credit.consumed · adminfee.accrued
Assurance: qa.* (AIRIS)        Improvement: asset.* · council.* (Observatory)
```

---

## 5. Event catalogue

Payload tables list fields **beyond the envelope**. Sens. is the highest sensitivity of the field. "Req" follows §2.2. Producers and consumers are named by module.

### 5.1 Provider accounts (telephony and AI)

| Event | Trigger | Payload | Sens. | Producer → Consumers |
|---|---|---|---|---|
| `provider_account.created` | Credentials accepted and stored | `provider_account_id`, `kind` (`telephony`/`ai`), `provider`, `commercial_mode?` (AI only), `data_region`, `credential_ref`, `created_by` | P1 | Integration Hub → audit, FinOps |
| `provider_account.verified` | Validation/discovery passed | `capability_declaration_ref`, `limits` {cps, concurrency, rpm, streams}, `checks_passed[]` | P0 | Hub → audit, admin UI |
| `provider_account.verification_failed` | Validation failed | `failed_step`, `reason_code`, `remedy_hint` | P0 | Hub → admin UI |
| `provider_account.activated` / `.suspended` / `.revoked` | State change | `from_state`, `to_state`, `changed_by`, `reason` | P0 | Hub → routing, audit |
| `provider_account.credential_rotated` | Key replaced | `credential_ref_old`, `credential_ref_new`, `rotated_by`, `approved_by`, `live_calls_affected` (must be 0) | P1 | Hub → audit, security |
| `provider_account.limits_changed` | Limits updated | `before`, `after`, `source` (`provider`/`manual`) | P0 | Hub → admission control |
| `provider_account.health_changed` | Health probe or traffic signal changed state | `from`, `to`, `signal` (`probe`/`traffic`/`status_feed`), `error_rate`, `latency_p95_ms`, `window_seconds` | P0 | Hub → failover, SAIGE |
| `provider_account.test_connection_completed` | Admin ran Test Connection | `capabilities_tested[]`, `latency_ms` per capability, `permission_gaps[]`, `cost_estimate` | P0 | Hub → admin UI, FinOps |
| `provider_account.drift_detected` | Platform intent differs from provider config | `resource_type`, `resource_id`, `drift_summary` | P0 | Hub → ops |
| `provider_account.spend_threshold_crossed` | Soft/hard spend threshold hit | `scope`, `threshold` (`soft`/`hard`), `spent` (`Money`), `cap` (`Money`), `action_taken` | P0 | FinOps → admission, ops |

### 5.2 Numbers, pools, routes, snapshots

Registry events use `sequence` per `number_id` / `route_id`, and always carry `change_request_id` and `actor`.

| Event | Trigger | Payload | Sens. |
|---|---|---|---|
| `number.ordered` | Purchase submitted by admin | `number_id`, `number` (`PhoneRef`), `role`, `number_type`, `provider_account_id`, `estimated_cost` (`Money`), `approved_by`, `spend_allowance_remaining` | P2 |
| `number.regulatory_updated` | Requirement status changed | `requirement_type`, `from`, `to`, `evidence_ref`, `expires_at?` | P1 |
| `number.activated` | Lifecycle reached active | `role`, `allocated_tenant_id`, `allocated_client_id`, `binding_sync_status`, `fallback_verified` (inbound) or `landing_policy_id` (outbound) | P1 |
| `number.allocated` / `number.reclaimed` | Right-of-use change | `from_client`, `to_client`, `reason` | P0 |
| `number.restricted` / `number.quarantined` / `number.released` | State change | `from_state`, `to_state`, `reason_code`, `evidence_ref?`, `cooling_period_until?` | P0 |
| `number.reputation_changed` | Scoring window closed | `window`, `attempts`, `answered`, `short_calls`, `rejected`, `blocked_indicators`, `score`, `scoring_version`, `flags[]` | P0 |
| `number.cli_approved` / `number.cli_revoked` | CLI approval change | `scope`, `campaign_ids?`, `approved_by`, `client_authorisation_ref`, `expires_at?` | P1 |
| `pool.created` / `pool.member_added` / `pool.member_paused` / `pool.member_quarantined` / `pool.strategy_changed` | Pool change | `pool_id`, `number_id?`, `strategy`, `spans_providers`, `provider_weights` | P0 |
| `pool.number_selected` | An outbound attempt selected a number | `attempt_id`, `pool_id`, `number_id`, `provider_account_id`, `strategy`, `eligible_count`, `excluded_counts` by reason, `policy_versions`, `random_seed_ref` | P0 |
| `pool.selection_failed` | Eligible set empty | `attempt_id`, `pool_id`, `excluded_counts`, `reason_code=cli_unavailable` | P0 |
| `landing_policy.published` / `.retired` | Callback landing policy change | `landing_policy_id`, `behaviour`, `applies_to`, `version`, `redirect_inbound_number_id?` | P0 |
| `route.version_drafted` / `.validated` / `.approved` / `.published` / `.rolled_back` / `.retired` | Route lifecycle | `route_id`, `route_version_id`, `version_no`, `workflow_ref`, `validation_report_ref?`, `approved_by`, `published_by` | P0 |
| `destination.created` / `.health_changed` / `.fallback_changed` / `.retired` | Destination change | `destination_id`, `destination_type`, `health`, `fallback_destination_id?` | P1 |
| `policy.version_published` | Policy bundle published | `policy_type`, `policy_id`, `policy_version`, `diff_ref` | P0 |
| `snapshot.compiled` / `.activated` / `.load_failed` / `.rolled_back` | Serving snapshot change | `snapshot_id`, `snapshot_no`, `contents_hash`, `node_id?`, `load_error?` | P0 |
| `killswitch.engaged` / `.released` | Emergency control | `scope_type`, `scope_id`, `reason`, `engaged_by`/`released_by` | P0 |

### 5.3 Dial, call lifecycle and outcomes

**Outbound**
| Event | Trigger | Payload | Sens. |
|---|---|---|---|
| `dial.requested` | Campaign module submitted a dial job | `dial_job_id`, `contact_point_ref`, `destination` (`PhoneRef`), `workflow_ref`, `policy_decision` (`PolicyRef`), `window_expires_at`, `machine_detection_mode`, `max_ring_seconds` | P2 |
| `dial.admitted` | Passed admission control | `selected_number_id`, `provider_account_id`, `carrier`, `admission_checks` {cps, concurrency, ai_capacity, tenant_share, spend, cli}, `queue_wait_ms` | P0 |
| `dial.refused` | Refused before placing | `reason_code`, `failed_checks[]`, `retry_hint` (`reschedule`/`never`/`later`) | P0 |
| `dial.started` | Carrier accepted the dial | `provider_ref`, `cli` (`PhoneRef`), `carrier` | P1 |
| `call.ringing` | Ringing detected | `ring_started_at` | P0 |
| `call.connected` | Answered | `answer_delay_ms`, `answered_by_guess` (`human`/`machine`/`unknown`), `answer_confidence?` | P0 |
| `call.machine_detected` | Machine/voicemail/fax decision | `kind`, `confidence`, `time_to_decision_ms`, `action` (`end`/`leave_message`/`flag`) | P0 |
| `call.no_answer` / `.busy` / `.rejected` / `.failed` | Non-answer result | `ring_seconds`, `reason_code`, `provider_code_raw_ref?` | P0 |

**Inbound and callback landing**
| Event | Trigger | Payload | Sens. |
|---|---|---|---|
| `call.inbound_received` | Call arrived at a number | `direction` (`inbound`/`callback_landing`), `number_id`, `caller` (`PhoneRef` or `withheld`), `provider_ref`, `snapshot_id` | P2 |
| `call.route_resolved` | Route/landing resolved | `route_id` or `landing_policy_id`, `route_version_id`, `rule_id`, `resolution_ms` | P0 |
| `call.inbound_decision` | Decision taken | `decision` (`ACCEPT_AI`/`TRANSFER`/`PLAY_AND_END`/`REJECT`/`OVERFLOW`/`LANDING_MESSAGE`), `reason_code`, `capacity_state`, `overrides[]` (kill switch, capacity) | P0 |
| `call.callback_matched` | Callback linked to an earlier attempt | `matched_attempt_id`, `matched_conversation_id`, `lookback_days`, `match_confidence`; **match is a hint, not identity** | P1 |
| `call.answered` | Platform answered inbound | `answer_delay_ms` | P0 |

**Common to all calls**
| Event | Trigger | Payload | Sens. |
|---|---|---|---|
| `call.state_changed` | Canonical state change | `from_state`, `to_state` | P0 |
| `call.ended` | Terminal | `telephony_outcome`, `end_reason`, `ended_by`, `durations` {ring, connected, ai_active, transfer, hold}, `cli_number_id?`, `carrier`, `provider_ref` | P0 |
| `call.resumed_from` | New `call_id` continues a conversation | `previous_call_id`, `checkpoint_ref`, `resume_reason` | P0 |
| `call.hold` / `call.resume` | Hold toggled | `leg_id`, `reason` | P0 |
| `call.dtmf_collected` | Digits received | `purpose` (`menu`/`sensitive`), `digit_count`; **digits never in event when purpose = sensitive** | P3 omitted |
| `call.audio_played` | Platform audio played | `asset_id?`, `purpose` (`disclosure`/`hold`/`closing`/`landing`), `completed` | P0 |

### 5.4 Transfer, recording, media

| Event | Trigger | Payload | Sens. |
|---|---|---|---|
| `transfer.initiated` | Transfer started | `transfer_type` (`blind`/`warm`), `destination_id`, `reason_code`, `case_id?`, `context_ref` | P0 |
| `transfer.answered` / `.completed` | Destination answered / handover done | `time_to_answer_ms`, `ai_leg_dropped_at?` | P0 |
| `transfer.failed` / `.abandoned` | Failure or caller left | `reason_code`, `fallback_action` (`callback`/`ticket`/`message`), `wait_seconds` | P0 |
| `recording.started` | Recording began | `recording_id`, `policy_ref`, `notice_played`, `channels` (`single`/`dual`) | P0 |
| `recording.paused` / `.resumed` | Sensitive input window | `reason` (`sensitive_input`/`policy`), `offset_ms` | P0 |
| `recording.stopped` | Recording stopped | `duration`, `bytes?` | P0 |
| `recording.upload_pending` / `.upload_confirmed` / `.upload_failed` | Storage transfer | `storage_destination_ref`, `object_ref`, `checksum`, `encryption_state`, `attempt_no`, `error_code?` | P1 |
| `media.session_opened` | Audio session set up | `transport` (**PENDING**: `carrier_stream`/`sip`), `codec`, `sample_rate`, `session_token_ref` | P0 |
| `media.first_audio_received` / `.first_audio_sent` | First frame each direction | `latency_from_connect_ms` | P0 |
| `media.session_degraded` / `.lost` / `.restored` | Quality/continuity | `packet_loss_pct`, `jitter_ms`, `gap_ms`, `reconnect_attempt` | P0 |

### 5.5 Conversation, listening and voice turns

| Event | Trigger | Payload | Sens. |
|---|---|---|---|
| `conversation.intent_detected` | Intent classified | `intent`, `confidence`, `language`, `language_confidence` | P0 |
| `voice.language_detected` / `.language_switched` | Language identified or code-switch handled | `from_language?`, `to_language`, `confidence`, `handling` (`same_talent`/`switch_talent`/`transfer`) | P0 |
| `voice.turn_started` | Customer speech accepted as a turn | `turn_id`, `listening_profile_id`, `profile_version`, `step_override_id?` | P0 |
| `voice.turn_completed` | Bot response delivered | `turn_id`, `stack_profile_id`, `talent_version_id`, `time_to_first_audio_ms`, `turn_latency_ms`, `response_path` (`prerecorded`/`stitched`/`synthesised`), `asset_ids[]`, `emotion_register`, `dictionary_version`, `audio_heard_ratio` | P0 |
| `listening.sound_classified` | Non-trivial sound classification | `class` (`noise`/`non_lexical`/`backchannel`/`speech`), `duration_ms`, `energy`, `confidence`, `action` (`ignored`/`candidate_barge_in`/`barge_in_accepted`) | P0 |
| `listening.barge_in_accepted` | Genuine interruption stops the bot | `turn_id`, `time_to_stop_ms`, `bot_audio_played_ratio`, `step_interruptible` | P0 |
| `listening.false_interruption_flagged` | Suspected self-trigger (echo/noise) | `suspected_cause` (`echo`/`noise`/`filler`/`cough`), `profile_version` | P0 |
| `listening.missed_interruption_flagged` | Customer spoke over bot but bot continued | `overlap_ms`, `profile_version` | P0 |
| `voice.emotion_transition_checked` | Stitch/live join evaluated | `from_register`, `to_register`, `allowed`, `action` (`stitch`/`smooth`/`resynthesise`/`blocked`), `prosody_compat_score` | P0 |
| `voice.stitch_rejected` | Seam below threshold | `asset_ids[]`, `scores` {pitch, pace, loudness, timbre, seam}, `fallback` (`full_synthesis`) | P0 |
| `voice.pronunciation_applied` | Dictionary entry used | `entry_ids[]`, `dictionary_version`, `provider_rendering_version` | P0 |
| `voice.name_pronunciation_flagged` | Low-confidence name | `contact_point_ref`, `confidence`, `fallback` (`neutral_address`/`spell`/`skip`) | P1 |

Transcript text and customer names are **not** in these events; they live in Module 09 and are referenced by `turn_id`.

### 5.6 Voice stack health and failover (D10)

| Event | Trigger | Payload | Sens. |
|---|---|---|---|
| `voice.stack_selected` | Stack chosen for a call | `stack_profile_id`, `failover_group_id`, `ai_commercial_mode`, `ai_account_ids` by function, `reason` (`primary`/`standby`/`manual`) | P0 |
| `voice.provider_degraded` | Provider crossed thresholds | `ai_account_id`, `function`, `signal`, `value`, `threshold`, `scope` (`call`/`platform`) | P0 |
| `voice.provider_retry` | Single retry within budget | `function`, `attempt_no`, `outcome` | P0 |
| `voice.failover_started` | Switch to standby begins | `from_stack`, `to_stack`, `trigger_reason_code`, `mid_call`, `talent_twin_id?`, `checkpoint_ref?`, `cross_mode` (BYOAI to Daythree AI: `true`/`false`) | P0 |
| `voice.failover_completed` | Standby serving | `switch_duration_ms`, `bridge_used` (`asset`/`phrase`/`silence`), `first_audio_on_standby_ms`, `twin_similarity_score` | P0 |
| `voice.failover_failed` | Standby also failed | `fallback_action` (`transfer`/`callback`/`end_message`), `reason_code` | P0 |
| `voice.failback` | Primary restored | `after_stable_seconds`, `scope` (`new_calls`/`all`) | P0 |
| `voice.failover_blocked` | Policy prevented switch | `reason_code` (`cross_mode_not_opted_in`, `no_twin`, `credits_insufficient`, `manual_hold`) | P0 |
| `voice.stack_override_set` / `.released` | Manual control | `stack_profile_id`, `mode` (`force`/`drain`/`pause_failover`), `set_by` | P0 |

### 5.7 Talent, pronunciation, assets, council

| Event | Trigger | Payload | Sens. |
|---|---|---|---|
| `talent.brief_generated` | GPT draft of brief | `talent_id`, `requirements_ref`, `model`, `usage` (`UsageQuantity[]`) | P0 |
| `talent.candidate_generated` | Candidate voice produced | `talent_id`, `provider`, `voice_ref`, `method` (`stock`/`designed`/`cloned`), `consent_ref?` (required when `cloned`), `screening_scores` | P1 |
| `talent.approved` / `.rejected` / `.retired` | Governance decision | `talent_version_id`, `approver`, `markets[]`, `languages[]`, `licence_ref`, `expires_at?` | P0 |
| `talent.twin_linked` | Equivalent on other provider | `talent_id`, `provider_a_voice_ref`, `provider_b_voice_ref`, `similarity_score`, `languages_covered[]` | P0 |
| `pronunciation.entry_added` / `.updated` / `.retired` | Dictionary change | `entry_id`, `language`, `category`, `scope`, `dictionary_version`, `changed_by` | P0 (customer-name entries: P2, tenant scope only) |
| `pronunciation.test_completed` | Entry tested per talent | `entry_id`, `talent_version_id`, `provider`, `result` (`pass`/`override_needed`/`fail`), `score` | P0 |
| `pronunciation.assets_invalidated` | Dictionary change staled assets | `entry_id`, `asset_ids[]` | P0 |
| `asset.generated` | Candidate asset created | `asset_id`, `talent_version_id`, `provider`, `model`, `language`, `emotion_register`, `expressive_params`, `script_version`, `usage` (`UsageQuantity[]`), `cost` (`Money`) | P0 |
| `asset.auto_evaluated` | Automatic checks done | `asset_id`, `scores` {loudness, clipping, silence_edges, intelligibility, pronunciation}, `pass` | P0 |
| `council.review_started` / `.review_completed` | Council evaluation | `council_review_id`, `asset_id`, `roles` (compliance wording, naturalness, tone continuity, pronunciation, seam), `role_scores`, `outcome` (`pass`/`fail`/`needs_human`), `model_family_separation_ok` | P0 |
| `asset.published` | Servable (after council pass) | `asset_id`, `canary_share`, `regulated` (`true`/`false`), `human_approval_ref?` (required when regulated) | P0 |
| `asset.withdrawn` | Pulled from service | `asset_id`, `reason_code`, `trigger` (`council`/`canary_monitor`/`qa`/`manual`), `affected_calls_estimate` | P0 |
| `asset.stale` | Dependency changed | `asset_id`, `cause` (`dictionary`/`talent_version`/`model`/`script`/`matrix`) | P0 |
| `asset.reuse_hit` | Asset served | `asset_id`, `call_id`, `turn_id`, `saved_cost_estimate` | P0 |

### 5.8 Contact points (Campaign module)

| Event | Trigger | Payload | Sens. |
|---|---|---|---|
| `contact_point.status_changed` | Contactability inference updated | `contact_point_ref`, `from_status`, `to_status`, `evidence` {attempts, window, distinct_cli_count, outcomes_summary}, `confidence`, `inference` (`true` for `suspected_blocked`/`rejected_pattern`) | P1 |
| `contact_point.suppressed` | Wrong number / DNC / do-not-contact | `kind`, `source` (`call_content`/`customer_request`/`list`), `effective_from` | P1 |
| `contact_point.review_flagged` | Conflicting signals | `signals[]` | P1 |

Statuses reported to clients must be labelled inferential where applicable; `suspected_blocked` is never asserted as fact.

### 5.9 Usage, cost, credits and fees

All usage and cost events carry `ai_commercial_mode` (for AI usage), `rate_card_id`, `rate_card_version`, and the link to `call_id`/`turn_id`/`asset_id`/`job_id` as applicable.

| Event | Trigger | Payload | Sens. |
|---|---|---|---|
| `telephony.usage_estimated` | Call ended | `carrier`, `provider_account_id`, `legs[]` {leg_id, destination_country, billable_seconds, rate}, `number_rental_share?`, `recording_minutes?`, `estimated_cost` (`Money`) | P0 |
| `telephony.usage_reconciled` | Provider usage matched | `usage_estimated_event_id`, `reconciled_cost` (`Money`), `variance` (`Money`, %) | P0 |
| `telephony.usage_discrepancy` | Variance beyond tolerance | `variance`, `tolerance`, `suspected_cause` | P0 |
| `ai.usage_recorded` | AI request completed | `ai_account_id`, `ai_provider`, `function`, `model`, `model_version`, `usage` (`UsageQuantity[]`), `stack_profile_id`, `purpose` (`live_call`/`asset_generation`/`qa`/`talent_creation`/`test`), `ai_commercial_mode`, `estimated_cost` (`Money`) | P0 |
| `ai.usage_reconciled` | Matched to provider usage/invoice | `usage_event_id`, `reconciled_cost`, `variance` | P0 |
| `credit.topped_up` | Balance added | `credit_ledger_id`, `credits`, `source` (`purchase`/`grant`/`adjustment`), `approved_by?` | P0 |
| `credit.reserved` / `.released` | Campaign reservation | `campaign_id`, `credits`, `reason` | P0 |
| `credit.consumed` | Usage priced | `credit_ledger_id`, `usage_event_id`, `credits`, `unit_price_version`, `balance_after` | P0 |
| `credit.low_balance` / `.exhausted` | Threshold / zero | `threshold`, `balance`, `action` (`alert`/`stop_outbound`/`restrict_inbound`/`grace`), `grace_until?` | P0 |
| `adminfee.accrued` | BYOAI fee posted | `tenant_id`, `period`, `cost_base` (`Money`), `fee_rate`, `fee` (`Money`), `base_source` (`metered`/`invoice`) — **PENDING Q11** | P0 |
| `margin.snapshot` | Periodic unit economics | `scope`, `period`, `revenue`, `provider_cost`, `platform_cost`, `margin` | P0 |

### 5.10 Quality, security and anomalies

| Event | Trigger | Payload | Sens. |
|---|---|---|---|
| `qa.policy_checked` | AIRIS pre-action/during-call check | `check_id`, `rule_ids[]`, `decision` (`allow`/`block`/`reroute`/`escalate`), `latency_ms` | P0 |
| `qa.post_call_completed` | Post-call evaluation | `qa_run_id`, `scores` by role, `critical_violation`, `audio_evaluated`, `flags[]` (includes `failover_seam`, `stitch_reject`) | P0 |
| `telephony.callback_rejected` | Callback failed authentication | `reason_code`, `source_ref`, `provider` | P1 |
| `telephony.event_out_of_order` / `.event_duplicate` | Ingress anomaly | `original_event_id`, `expected_sequence`, `received_sequence` | P0 |
| `telephony.unreconciled_call` | No end event or cost match after threshold | `age_minutes`, `missing` (`end_event`/`cost`) | P0 |
| `security.anomaly_detected` | Suspicious pattern (spend spike, credential misuse, caller-ID misuse) | `kind`, `severity`, `scope`, `evidence_ref` | P1 |
| `change_request.opened` / `.approved` / `.applied` / `.rolled_back` | Governed change | `change_request_id`, `type`, `risk_class`, `requested_by`, `approvers[]` | P0 |

---

## 6. Example sequences (illustrative)

### 6.1 Outbound call answered, mid-call provider failover
```
dial.requested → pool.number_selected → dial.admitted → dial.started → call.ringing → call.connected
→ recording.started → voice.stack_selected → conversation.intent_detected
→ voice.turn_started → voice.turn_completed (stitched) …
→ voice.provider_degraded → voice.provider_retry → voice.failover_started
→ voice.failover_completed → voice.turn_completed (standby stack)
→ call.ended → recording.stopped → recording.upload_confirmed
→ telephony.usage_estimated → ai.usage_recorded (per stack) → credit.consumed
→ qa.post_call_completed (flag: failover_seam) → contact_point.status_changed
```

### 6.2 Callback to an outbound line
```
call.inbound_received (direction = callback_landing) → call.route_resolved (landing_policy_id)
→ call.callback_matched (hint) → call.inbound_decision (LANDING_MESSAGE)
→ call.audio_played (purpose = landing) → call.ended
→ contact_point.status_changed (customer engaged)
```

### 6.3 Pre-recorded asset, council gate, then withdrawal
```
asset.generated → asset.auto_evaluated → council.review_started → council.review_completed (pass)
→ asset.published (canary) → asset.reuse_hit … → qa.post_call_completed (defect)
→ asset.withdrawn (trigger = canary_monitor) → asset.stale (dependants)
```

### 6.4 Refused dial
```
dial.requested → dial.refused (reason_code = suppressed or window_closed)
```
No `dial.started`. The `policy_decision` reference is stored for audit.

---

## 7. Producer and consumer matrix

| Consumer | Needs |
|---|---|
| Campaign (05) | `dial.*`, `call.ended`, `call.callback_matched`, `contact_point.*`, `pool.selection_failed`, `credit.exhausted` |
| Workflow Executor (08) | `call.*`, `transfer.*`, `voice.failover_*`, `conversation.*` |
| Voice Engine (07) | `asset.published/withdrawn/stale`, `pronunciation.*`, `provider_account.health_changed`, `snapshot.activated` |
| Recording (09) | `recording.*`, `call.ended`, `voice.turn_completed` |
| AIRIS (10) | `qa.*`, `voice.*`, `call.*`, `asset.*`, `council.*` |
| SAIGE / FinOps (11) | all usage, credit, fee, call and failover events, reputation, spend |
| Observatory (12) | `qa.post_call_completed`, `listening.*`, `voice.stitch_rejected`, `asset.*`, `pronunciation.test_completed` |
| Admin UI | provider, registry, health, spend and approval events |
| Audit store | all `audit`-class events, `change_request.*`, `provider_account.credential_rotated`, `killswitch.*`, `talent.approved` |

---

## 8. Required validations

1. Every event validates against its schema version before acceptance; failures go to dead-letter with the reason.
2. `call.dtmf_collected` with `purpose=sensitive` must contain no digit values.
3. `talent.candidate_generated` with `method=cloned` must carry `consent_ref`.
4. `asset.published` for assets marked regulated must carry `human_approval_ref`.
5. `council.review_completed` requires `model_family_separation_ok = true` for high-risk assets.
6. `voice.failover_started` with `cross_mode = true` requires an opt-in reference and sufficient credits, otherwise `voice.failover_blocked` is emitted.
7. `pool.number_selected` must reference an `outbound_line` number with a valid CLI approval.
8. No event may contain provider credentials, OTPs or raw sensitive identifiers.
9. For every call, a `call.ended` (or a recorded anomaly) must appear within the allowed time or `telephony.unreconciled_call` fires.

---

## 9. Conformance testing for schemas

| Test | Purpose |
|---|---|
| Twilio vs Telnyx parity | Same scenario produces equivalent events in both adapters |
| Duplicate and out-of-order injection | Consumers stay correct |
| Replay | Projections rebuilt from events match live projections |
| Version compatibility | Old consumers tolerate new minor versions; reject unknown majors safely |
| Redaction audit | Automated scan proves no P3 data and no unmasked P2 data in events |
| Billing trace | Every `credit.consumed` and `adminfee.accrued` links to usage events and a rate card version |
| Failover trace | A simulated outage produces the full §6.1 chain with consistent `trace_id` |

---

## 10. Pending items and open questions

| # | Item | Affects |
|---|---|---|
| E1 | **Media transport** (carrier stream or SIP to platform media server) — undecided | `media.*` fields, latency metrics |
| E2 | Emotion register set (Requirements Q7) | `emotion_register` enumeration, council rubric |
| E3 | Admin-fee base: metered cost vs invoice (Q11) | `adminfee.accrued` |
| E4 | Do credits cover telephony as well (Q12)? | `credit.consumed` for `telephony.*` |
| E5 | Cross-mode failover policy (Q13) | `voice.failover_*` fields |
| E6 | Mid-call vs between-call switching (earlier question 3; assumed mid-call allowed with twin) | `voice.failover_started.mid_call` |
| E7 | Zero-credit behaviour (Q14) | `credit.exhausted.action` |
| E8 | Exact retention periods per `data_class` | Storage and compliance |
| E9 | Event transport, registry tooling and serialisation format | Implementation |
| E10 | Whether `listening.*` classification events are sampled to control volume | Cost of telemetry |

---

## 11. Change control

| Version | Date | Change |
|---|---|---|
| 0.1 | 17 Oct 2026 | Initial draft covering telephony, registry, voice stack and failover, talent/assets/council, contact points, metering and credits |

Schema changes follow §2.5. Every change requires a consumer impact list and, for major changes, a dual-publish plan.
