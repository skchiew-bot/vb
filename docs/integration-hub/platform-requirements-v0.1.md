# Integration Hub & Voice Layer — Requirements

**Module scope:** 02 Integration Hub (telephony and AI providers), 05 Number/DID routing, 07 Conversational Voice Engine (talent, listening, pronunciation), 12 Observatory (voice-asset pre-recording)
**Version:** 0.2 (draft for design review)
**Date:** 17 October 2026
**Related:** `telephony-contract-spec.md` (v0.1), `number-registry-routing-data-model.md` (v0.3)
**Status:** Requirements captured from product owner. Provider behaviour, limits, pricing, Malaysian/Singapore regulatory rules and voice licensing terms must be validated before build commitments.

---

## 1. Objective

Let an authorised administrator connect telephony and AI voice providers by entering credentials, after which the platform:

1. connects and configures DIDs for inbound and outbound use across more than one carrier;
2. randomises outbound caller-ID from a pool, while keeping inbound numbers exclusive to inbound;
3. builds localised (Malaysia, Singapore) expressive voice talent from ElevenLabs and/or OpenAI, with AI assistance;
4. pre-records and tags scripts autonomously, gated by an AI quality council, so that pre-recorded and live speech match in tone and emotion;
5. listens robustly (noise, filler sounds, echo) so the bot is not wrongly interrupted;
6. pronounces names and terms correctly through a managed pronunciation dictionary;
7. switches between voice providers if one becomes unreachable.

---

## 2. Decision log

| # | Decision | Source |
|---|---|---|
| D1 | Daythree is owner of record of all numbers | Product owner |
| D2 | Fixed number assignment remains allowed per campaign | Product owner |
| D3 | No shared inbound numbers | Product owner |
| D4 | Number role is exclusive: `inbound_line` or `outbound_line`; outbound lines have a callback landing behaviour | Product owner (explained in §4.4) |
| D5 | Outbound pool spans providers: Twilio plus one other carrier; randomised | Product owner |
| D6 | Only admin / super admin may buy or release numbers | Product owner |
| D7 | Talent is created for Malaysia and Singapore, localised, from ElevenLabs and/or OpenAI, with expressive parameters | Product owner |
| D8 | Pre-recorded assets auto-publish, then are checked by an AI council for tone, pitch, conversation flow and emotional consistency | Product owner (see risk R-3) |
| D9 | Non-speech sounds (mm, um, uh, cough, low-frequency/background noise) must not interrupt the bot | Product owner |
| D10 | AI provider credentials are entered and stored in settings; the voice stack can fail over between ElevenLabs and OpenAI in both directions | Product owner |
| D11 | Second carrier is **Telnyx**; outbound pools span Twilio and Telnyx; both adapters are first release | Product owner |
| D12 | ElevenLabs and OpenAI each cover speech-to-text, language model and text-to-speech, so a full stack can run on either | Product owner (to be validated per language, see §6.3) |
| D13 | Outbound caller-ID is random per attempt; no sticky number per customer | Product owner |
| D14 | Two AI commercial modes: **BYOAI** (client's own keys; Daythree charges a 10% admin fee) and **Daythree AI** (platform keys; billed in credits) | Product owner (details and gaps in §5.5) |
| D15 | Languages: English, Bahasa Malaysia, Mandarin, Singlish, Manglish, Cantonese, Tamil | Product owner (phasing in §6.3) |

Open questions are listed in §12.

---

## 3. Provider onboarding (telephony and AI)

### 3.1 Common onboarding principles
- Credentials are entered once in **Settings → Providers** by a super admin, stored only in the secrets vault, shown masked, and never returned by any API after saving.
- Prefer scoped keys (least privilege). The platform declares the permissions it needs and rejects keys that are broader than necessary where the provider allows scoping.
- Onboarding is **idempotent and re-runnable**: it reconciles desired vs actual configuration; running it twice changes nothing.
- Every step is logged to the audit trail with a result. Failures are explicit, with the step, cause and remedy.
- Rotation and revocation do not drop live calls.

### 3.2 Telephony onboarding flow (Twilio and Telnyx)
| Step | Action | Result |
|---|---|---|
| 1 | Admin enters credentials and picks the data region | Stored in vault; reference only in database |
| 2 | **Validate**: authenticate, confirm key scope, read account status, balance/limits | Pass/fail with reasons |
| 3 | **Discover**: capabilities, rate limits (calls per second, concurrency), available number types for Malaysia and Singapore, existing numbers | Capability declaration saved |
| 4 | **Import** existing numbers (admin chooses role for each) | Numbers appear in registry as `stock` or allocated |
| 5 | **Configure**: inbound handling endpoints, status callbacks, carrier-level fallback, recording and media settings | Binding `in_sync` |
| 6 | **Verify**: test inbound call, test outbound call, forged-callback rejection, fallback test | Pass report stored |
| 7 | **Activate** provider account | Status `active`; visible to routing |
| 8 | **Monitor**: drift detection, limit and balance alerts | Ongoing |

**Buying numbers (D6):** restricted to admin / super admin. Recommended safeguards: per-purchase and monthly spend limits, a visible cost preview, regulatory-document checklist (Malaysia/Singapore requirements differ and may need human-supplied documents), and an audit record. Purchase is never triggered automatically by onboarding.

### 3.3 AI provider onboarding (ElevenLabs, OpenAI)
Same flow shape: enter → validate → discover → configure defaults → test → activate → monitor.

**Settings captured per AI provider account** (all editable by super admin only):

| Group | Fields |
|---|---|
| Identity | provider (ElevenLabs / OpenAI), account label, environment (sandbox / production), commercial mode (`daythree_ai` platform-managed, billed in credits; or `byoai` tenant-owned, billed at cost plus 10% admin fee; see §5.5), tenant scope |
| Credentials | API key (vault reference, masked, last-4 shown, created/rotated date), optional organisation/project identifier, optional additional keys (e.g. separate key for realtime or admin) |
| Endpoint | base URL / region, data-residency region, proxy settings if any |
| Allowed capabilities | speech-to-text, language model, text-to-speech, speech-to-speech/realtime, voice design/cloning, pronunciation dictionaries, sound generation |
| Model catalogue | allowed models per capability, default model per capability, pinned versions |
| Voice defaults | default voice talent mapping, default stability/expressiveness/speed ranges (provider-specific) |
| Limits | requests per minute, concurrent streams, daily/monthly token or character caps, per-call cost budget |
| Spend control | soft threshold, hard cap, action on cap (alert, stop generation, switch provider), currency |
| Data handling | retention setting (zero-retention where available), whether customer audio may be used for provider training (default: no), region restrictions |
| Health | health-check interval, timeout, latency thresholds, error-rate thresholds |
| Status | `pending_verification`, `active`, `degraded`, `suspended`, `revoked`; last check, last error |

A **Test Connection** action performs a cheap, bounded call per enabled capability and reports latency, model access and any permission gaps. It is metered and counted against spend limits.

---

## 4. Telephony and number requirements

### 4.1 R1 — Connect by credentials
Acceptance: after entering valid credentials, an admin can complete §3.2 without engineering help; a test inbound and outbound call succeed; the provider appears in the routing and pool configuration.

### 4.2 R2 — Exclusive number roles (D4)
- Every number is exactly one of `inbound_line` or `outbound_line`.
- Inbound lines are never in a pool and never presented as outbound caller-ID.
- Outbound lines never have an inbound route; they have a callback landing policy (§4.4).
- Changing a role is a controlled change: release to stock, cooling period, re-activation.

Acceptance: system rejects any configuration that violates the above; an audit query lists every number with its role and shows no violations.

### 4.3 R3 — Randomised outbound caller-ID across providers (D5)

**Selection (per outbound attempt):**
1. Start from the campaign's pool (numbers on any active provider account).
2. Remove ineligible numbers: quarantined, over per-number hourly/daily volume ceiling, below minimum reputation, provider degraded or at rate limit, CLI approval missing or expired.
3. Choose randomly from what remains, applying provider weights if configured.
4. Route the call through the provider that holds the chosen number.
5. Log: chosen number, pool, strategy, eligible-set size, provider, policy versions.

**Options:** sticky number per customer within a campaign (default off); weighting by provider, cost or reputation; automatic redistribution when a provider is degraded.

**Guardrails**
- Rotation changes only the number shown. Suppression, consent, calling windows and retry counters are keyed on the customer contact point and are unaffected.
- Rotation must not be used to evade carrier controls or regulatory limits; volume ceilings per number apply.
- If the eligible set is empty, the attempt is refused with `cli_unavailable` and rescheduled by the Campaign module.

**Customer-number contactability tracking.** Each attempt's outcome updates a **contact-point status** held by the Campaign module (not the telephony registry):

| Status | Inference |
|---|---|
| `contactable` | Answered by human at least once recently |
| `no_answer_pattern` | Repeated no-answer within policy windows |
| `unreachable` | Network/unallocated/invalid number outcomes |
| `rejected_pattern` | Immediate rejects or very short rings across attempts and across different caller-IDs (suggests screening or blocking by the customer) |
| `suspected_blocked` | Pattern consistent with the customer blocking Daythree-presented numbers; **an inference with a confidence level, never a fact** |
| `confirmed_wrong_number` / `do_not_contact` | From call content or customer request, per Blueprint §5.2 |
| `data_review_required` | Conflicting signals |

Each status stores evidence (counts, window, which numbers were tried), confidence and the date. It feeds retry policy and list hygiene, and aggregated counts feed number reputation. Telephony cannot reliably tell that a customer has blocked a number, so the platform must not state it as fact to the customer or in reports.

### 4.4 R2a — Callback landing behaviour (explained)

**What it is.** Because outbound numbers rotate, a customer will often call back the number that just rang them. That number is an outbound line, so it has no inbound route. A **callback landing behaviour** defines what the customer experiences when they call it, so they never reach a dead line, a "number not in service" tone or silence.

**Example.** The bot calls Mr Tan from +60 3-xxxx 1234 (outbound pool). He misses it and calls back. Instead of nothing happening, he hears an approved message such as "Thank you for returning our call. Please call us on our service line, 03-xxxx 5678, or stay on the line for assistance", and the call is directed accordingly. The platform also logs the callback against his earlier attempt.

| Option | Behaviour | Release |
|---|---|---|
| `message_and_redirect` | Play approved message pointing to the client's inbound line; end or transfer | **First release (default)** |
| `message_and_end` | Play approved message, end | First release |
| `transfer_to_destination` | Send to a configured human or queue | First release, if destination exists |
| `ai_with_context` | Connect to the voicebot using the recent outbound attempt as context; identity must still be verified before any account disclosure | Later release |

Why it matters: unanswered callbacks lose payments, create complaints, and can fail caller-ID reachability expectations in some jurisdictions (to be confirmed by compliance). Callbacks also give the best signal that a customer is willing to engage.

Rules: every active outbound line has an approved landing policy; callbacks respect suppression and DNC; the matched earlier call is a hint, not authentication; callbacks are logged and linked to the original attempt.

### 4.5 R6a — Number purchase control
Only admin / super admin may order or release numbers. The UI shows cost preview, regulatory checklist and remaining spend allowance before confirming.

---

## 5. Voice provider settings and failover (D10)

### 5.1 Concepts
- **Voice stack profile:** a named combination of providers and models for each function the bot needs: speech-to-text, language model, text-to-speech (or one speech-to-speech/realtime service). Examples: *Stack A — ElevenLabs voice + OpenAI reasoning*, *Stack B — OpenAI realtime end to end*. Exact capability combinations depend on what each provider supports and must be validated.
- **Failover group:** an ordered pair (or list) of stack profiles. One is primary, the other is standby.
- **Talent twin:** the same persona represented on both providers (matching voice, accent, pace, emotional range), linked together and approved as equivalent. Failover requires a twin; a talent without a twin cannot fail over and the platform warns at configuration time.

### 5.2 Health detection
Continuous synthetic probes plus live-traffic signals per provider and per function:
- connection failure or timeout;
- error rate above threshold in a rolling window;
- time-to-first-audio or turn latency above threshold (p95);
- rate-limit exhaustion or quota exhaustion;
- audio quality failure (empty or corrupted audio);
- provider status feed where available.

Thresholds are configurable; defaults are conservative. A provider moves `active → degraded → unreachable` with hysteresis to avoid flapping.

### 5.3 Switching behaviour
| Scenario | Behaviour |
|---|---|
| Provider unreachable **before** a call starts | New calls start on the standby stack automatically |
| Provider fails **mid-call** | Retry once within a short budget. If it persists, switch for the next turn to the standby stack using the talent twin; resume from the checkpoint; use pre-recorded assets or a neutral bridging phrase to cover the gap; if the switch also fails, offer human transfer or callback |
| Degraded but working | Shift a configurable percentage of new calls to standby, or alert only, per policy |
| Recovery | Fail back only after a sustained healthy period (hysteresis) and, by default, only for new calls, not mid-call |
| Manual override | Super admin can force a stack, drain a stack, or pause failover; audited |

Rules
- A mid-call voice change is audible, so the twin must be matched and approved in advance; the council scores twin similarity.
- Pre-recorded assets belong to a talent version; on failover, assets of the active talent remain usable only if the live voice is the same talent. Otherwise the stitching layer falls back to full synthesis on the standby (Blueprint §8.2) to avoid voice mismatch.
- Pronunciation dictionaries are provider-specific in mechanism; each stack carries its own compiled version of the same logical dictionary and is tested for parity.
- Failover events are recorded on the call (`voice.provider_degraded`, `voice.failover_started`, `voice.failover_completed`, `voice.failback`), and the call is flagged for QA review of the seam.
- Costs differ per stack; cost events record which stack served each turn.
- Spend cap behaviour is configurable: alert, stop generation, or switch to the cheaper stack.

### 5.4 Settings UI requirements
- Provider list with status, last health check, current latency/error rate, spend vs cap.
- Add / edit / rotate key (masked), Test Connection, enable/disable capabilities.
- Voice stack profile builder with validation that required capabilities are covered.
- Failover group editor: primary, standby, thresholds, hysteresis, manual override, simulate-failure test mode (sandbox only).
- Talent twin view showing equivalence scores and unmatched talents.
- Change history and audit trail.
- Permissions: super admin edits; admin views; changes to credentials and failover thresholds are high-risk changes requiring a second approver.

### 5.5 Commercial modes: BYOAI and Daythree AI (D14)

| | **Daythree AI** | **BYOAI** |
|---|---|---|
| Keys | Daythree's provider accounts (platform-managed) | Client's own keys, entered in the client's settings, stored in the vault under that tenant |
| Provider bill | Paid by Daythree | Paid by the client directly to the provider |
| Client is charged | **Credits** consumed per usage unit at the Daythree rate card | **10% admin fee** on the provider usage cost |
| Cost visibility | Daythree knows its true cost and margin | Cost must be taken from usage the platform meters and, where available, reconciled to the provider's usage data |
| Data terms | Daythree's provider agreements | Client's provider agreements; the client is responsible for its own retention and training settings |

**Required behaviour**
- Metering keeps the two modes in separate ledgers per tenant: a **credit ledger** (balance, top-ups, consumption, expiry) and an **admin-fee ledger** (provider cost base, 10%, invoice period).
- Every usage event carries `ai_commercial_mode`, provider account, stack profile and the rate card version, so the same call can be priced correctly even when it switches stack.
- Credits: pre-call balance check; reserve credits for active campaigns; low-balance alerts at configurable thresholds; defined behaviour at zero (stop new outbound dials, finish live calls, keep inbound service per policy).
- Credit price per unit is configured by Daythree and versioned; changes do not rewrite past usage.
- Fee base is a decision for you (Q11): 10% of metered provider cost, or of the provider invoice. Metered cost is available immediately; provider invoices are authoritative but arrive late and need reconciliation.

**Failover across modes — a billing gap to close**
If a client's BYOAI provider fails and the platform fails over to the standby, who pays? Options:
1. Standby must also be that client's own key (clean billing, but no protection if the client has only one key).
2. Fail over to Daythree AI and charge credits (better continuity; requires the client's consent in contract and sufficient credits).
3. Block failover across modes and hand off to a human or callback.
Default recommendation: option 1 where the client supplies two providers, otherwise option 2 only with prior opt-in and credits available; never switch modes silently. Each failover event records which mode served each turn.

**FinOps:** margin per tenant = credit revenue or admin fee minus Daythree's own provider cost (zero provider cost for BYOAI, so the admin fee must cover platform compute, QA, storage and support). Check that 10% covers those costs at scale; QA council, pre-recording and monitoring costs run on Daythree's own models and are not covered by the client's keys.

---

## 6. Voice talent (D7)

### 6.1 Definition
A **voice talent** is a governed persona: voice identity on one or more providers, language/accent coverage, style range, behaviour guidelines, pronunciation capability profile, licence, and approval state.

Fields (logical): `talent_id`, name, description, markets (Malaysia, Singapore), languages and accents (Malaysian English, Singapore English, Bahasa Malaysia; others to be confirmed), gender/age presentation (descriptive), provider representations (`provider`, `voice_id`, model, version), expressive parameters (provider-specific ranges for stability, expressiveness/style, speed, pitch tendencies), emotional range supported, approved use cases (collections, service, reminders), licence/consent evidence, twin links, test results, status, version, lineage.

### 6.2 AI-assisted creation flow
1. Admin states requirements (market, language mix, brand tone, use case, empathy level, pace, prohibited traits).
2. GPT drafts a **talent brief**: style guide, vocabulary preferences, sample scripts across emotions, pronunciation hot-list, and search/design parameters for the provider.
3. Platform generates candidate voices from ElevenLabs and/or OpenAI (stock voices, designed voices, or other provider-supported methods).
4. Automated screening: language accent fit, intelligibility, consistency over long text, emotional range, pronunciation test against the dictionary.
5. **Human audition and approval** before the talent can be used in production. Voice cloning of a real person requires documented consent and licence.
6. Create the **twin** on the second provider and score equivalence.
7. Publish the talent version; monitor in production.

### 6.3 Localisation
Malaysian and Singapore talents are separate personas, not one accent setting. Code-switching (for example English with Bahasa Malaysia, or Mandarin with English) and local usage must be covered in test scripts. The pronunciation hot-list covers local place names, banks, telcos, common names and terms.

**Language scope (D15) and how to treat it**
| Language | Nature | Key risk |
|---|---|---|
| English | Standard | Baseline |
| Bahasa Malaysia | Standard | Formal vs colloquial register |
| Manglish | English variety with Malay/Chinese/Tamil mixing | Recognition accuracy; no formal standard; speaking style must sound natural, not caricature |
| Singlish | English variety with Hokkien/Malay/Mandarin influence | Same; plus tone and respect conventions |
| Mandarin | Standard language | Accent: Malaysian vs Singaporean vs mainland; script (simplified/traditional) for any text |
| Cantonese | Spoken variety | Weaker provider support in recognition and synthesis; romanisation and name pronunciation vary |
| Tamil | Standard language | Provider support and voice quality may be limited; names and code-switching |

Per-language requirements: a **capability matrix** per provider and function (recognition, reasoning, synthesis, pronunciation control, expressiveness), measured on your own test sets rather than taken from provider claims; native-speaker reviewers for the council calibration set; separate pronunciation dictionary scope per language; language identification and a defined handling of code-switching within a call; a defined fallback when a language fails on one provider (switch provider for that language, transfer to a human, or offer another language).

**Recommended phasing** (to be validated by the capability matrix): English, Bahasa Malaysia and Manglish first; then Mandarin and Singlish; then Cantonese and Tamil once recognition and voice quality pass thresholds. Do not announce a language to clients as supported until it passes the scenario suite on at least one full stack.

### 6.4 Controls
- AI disclosure per regulation and client policy.
- Licensing and usage-rights tracking per talent and provider; talent expires or is blocked if rights lapse.
- Spend caps on generation and voice design.

---

## 7. Pre-recording, stitching and emotional continuity (D8)

### 7.1 Pipeline
1. **Mine** recurring phrases from approved scripts and anonymised conversations.
2. **Draft** script variants per language, with GPT assistance.
3. **Tag**: emotion/style register, pace, intent, whether sensitive or regulated, pronunciation items, allowed neighbours.
4. **Generate** the audio on the target talent and provider with the matching expressive parameters.
5. **Evaluate** automatically (loudness, clipping, silence edges, pronunciation, intelligibility, duration).
6. **Council review** (§7.3).
7. **Publish** to the asset library with full metadata and lineage.

### 7.2 Asset metadata
`asset_id`, text, language/accent, talent and talent version, provider, model and version, voice ID, emotion register, expressive parameters used, pace, loudness, pitch summary, pronunciation entries applied (dictionary version), start/end prosody (for stitching), duration, quality scores, council results, licence, checksum, cost, status, lineage (script version, parameters), compatible-neighbour hints.

### 7.3 Emotion register and continuity
- Use a **small controlled set of registers** (for example neutral-professional, warm, empathetic, apologetic, firm-calm, reassuring; set to be agreed) rather than free-form emotion labels.
- Define an **allowed transition matrix** between registers. A jump the matrix does not allow (for example neutral directly to very cheerful) is blocked or smoothed by choosing an intermediate register or a full re-synthesis.
- The stitching layer chooses asset and live-synthesis settings so that the live segment starts in the register the pre-recording ended in.
- Pre-recorded and live segments must use the same talent version, model and parameter set; if not, use full synthesis.
- A **Prosody Compatibility Scorer** checks pitch, pace, loudness, timbre and seam artefacts; below threshold means reject the stitch.

### 7.4 Council gate (D8) — recommended control
Product decision is **auto-publish, with council check**. The key design question is order:

| Model | Behaviour | Risk |
|---|---|---|
| Council **before** serving | Asset publishes automatically once the council passes; no human step | Safest; adds minutes of latency to publishing, which is acceptable for pre-recording |
| Council **after** serving | Asset goes live immediately; council reviews afterwards and withdraws failures | Customers may hear a defective asset before withdrawal |

**Recommendation:** auto-publish means *no human approval step*, but an asset becomes servable only after the council passes it (gate before serving). Add canary exposure (a small share of calls) with live monitoring, automatic withdrawal and rollback, and a hold list. Regulated or financially binding wording (disclosures, payment terms) should retain a human approval requirement even in an otherwise autonomous flow; this needs product-owner confirmation (open question Q6).

The council uses separate roles (compliance wording, naturalness, emotion/tone continuity, pronunciation, seam quality), must not be the same model family that generated the asset for high-risk items, and is calibrated against sampled human review.

### 7.5 Invalidations
A change to talent version, provider model, pronunciation entry, script wording or emotion matrix marks affected assets `stale`; stale assets are not served unless explicitly re-approved, and regeneration is queued under the spend cap.

---

## 8. Listening layer (D9)

### 8.1 Problems addressed
1. Background and low-frequency noise reaching speech recognition.
2. **Echo** of the bot's own voice leaking into the customer line and triggering self-interruption.
3. Non-speech vocalisations (mm, um, uh, cough, throat clearing, breath) interrupting the bot.
4. Genuine interruptions that must stop the bot.

### 8.2 Behaviour
- Apply noise suppression and echo cancellation before speech detection and recognition.
- Classify incoming sound as: noise, non-lexical vocalisation (filler, cough), backchannel word, or substantive speech. Only substantive speech (and meaningful short words) triggers barge-in.
- Treat **mm / um / uh / cough / background noise / low-frequency sound** as non-interrupting (per D9). Short lexical words such as "ya", "ok", "no", "wait", "boleh" are **not** filtered by default because they can carry meaning; a step may mark them as backchannel when appropriate.
- Barge-in sensitivity is a **versioned listening profile** with per-step overrides: minimum speech duration, minimum energy/confidence, frequency band, non-lexical sound list, grace period before stopping, and whether the step is **non-interruptible** (for example mandatory disclosures) or **high-sensitivity** (collecting a figure or an OTP).
- The bridge flushes queued audio immediately when a genuine interruption is accepted (Telephony Contract §9).

### 8.3 Measures and tuning
False-interruption rate, missed-interruption rate, echo-induced interruption rate, time-to-stop after real barge-in, percent of calls with talk-over, and recognition errors caused by noise. Profiles are tuned on real call data by carrier and channel; changes are versioned and A/B tested.

---

## 9. Pronunciation dictionary (R7)

### 9.1 Logical dictionary
Entry: `entry_id`, written form, language, category (person name, place, bank/telco/brand, product, acronym, number/date/currency format, other), phonetic representation (neutral form), **provider-specific renderings** (how each provider is told to say it), scope (global, client, talent, provider), example sentence, status, test results, version, owner, created/updated.

One logical dictionary is compiled into each provider's mechanism; provider support differs and must be verified (some providers offer pronunciation dictionaries or phoneme controls; some may not).

### 9.2 Talent pronunciation capability
For each talent and provider: which entries it pronounces correctly unaided, which need an override, which fail even with an override. Failures are visible, block assets that depend on them, and are reported to the talent owner.

### 9.3 Customer names
Names are the hard case (Malay, Chinese romanisation in various dialects, Indian names, abbreviations, honorifics, code-switching).
- Names come from campaign data, are personal data, and are tenant-scoped; they are not stored in shared dictionaries.
- **Pre-campaign name check:** for each list, generate and score names, flag low-confidence ones for review or a safe fallback (for example avoiding the name, or using a neutral form of address).
- Per-customer overrides are stored with the contact point under the tenant's retention rules.
- Numbers, amounts (RM / SGD), dates and times have standard formatting rules per language.

### 9.4 Regression
Each entry is audio-tested per talent after any change in dictionary, talent version or model. Changes invalidate dependent assets (§7.5). Dictionary versions are recorded on every call.

---

## 10. New and changed logical entities

| Entity | Purpose |
|---|---|
| `AiProviderAccount` | Credentials reference, capabilities, limits, health, data handling (§3.3) |
| `VoiceStackProfile` | Provider and model per function (§5.1) |
| `FailoverGroup`, `FailoverEvent` | Primary/standby, thresholds, history (§5) |
| `VoiceTalent`, `TalentVersion`, `TalentProviderVoice`, `TalentTwinLink` | Persona, per-provider voice, equivalence (§6) |
| `EmotionRegister`, `EmotionTransitionRule` | Controlled registers and allowed transitions (§7.3) |
| `VoiceAsset`, `AssetEvaluation`, `CouncilReview` | Pre-recordings, automatic scores, council outcomes (§7) |
| `ListeningProfile`, `ListeningProfileOverride` | Barge-in and noise rules, per-step overrides (§8) |
| `PronunciationEntry`, `PronunciationRendering`, `TalentPronunciationResult` | Dictionary and capability (§9) |
| `ContactPointStatus` (Campaign module) | Contactability inference with evidence and confidence (§4.3) |
| `CallbackLandingPolicy` (Number registry) | Callback behaviour for outbound lines (§4.4) |
| `Number.role`, `NumberPool` options | Exclusive role; cross-provider pool selection (data model v0.3) |

---

## 11. Telemetry, KPIs and abuse scenarios

| Area | Metrics |
|---|---|
| Providers | Health, latency, error rate, spend vs cap, failover count and duration |
| Pools | Selection distribution by provider and number, ceilings hit, empty-set refusals |
| Numbers | Answer rate, short-call rate, reject rate, quarantine events |
| Callbacks | Callback volume per outbound line, landing outcome, link to original attempt |
| Contactability | Share of list by status; `suspected_blocked` rate by caller-ID |
| Talent / assets | Council pass rate, stale assets, reuse hit rate, cost saved, defect withdrawals |
| Emotion continuity | Blocked transitions, seam rejections, listener-rated naturalness |
| Listening | False and missed interruption rates, echo events |
| Pronunciation | Entries failing per talent, name-check low-confidence rate, customer-name corrections |
| Failover | Mid-call failover success, customer impact (dropped/hand-off), failback time |

Abuse and risk scenarios
- Leaked or over-scoped API key → vault storage, scoped keys, rotation, high-risk change approval, no display after save.
- Runaway generation cost → per-provider caps, per-job budgets, kill switch.
- Cloning a real person without consent → licence/consent evidence required before activation.
- Number rotation to dodge blocking or limits → per-number ceilings, quarantine, customer-keyed suppression.
- Fabricated "blocked" claims → contactability is inferential, never stated to customers.
- Provider flapping → hysteresis, minimum dwell time, manual override.
- Defective asset served at scale → council gate before serving, canary, automatic withdrawal.
- Voice change mid-call confuses customer → talent twin approval, switch only at turn boundaries, QA flag.

---

## 12. Open questions

| # | Question | Impact |
|---|---|---|
| Q1 | ~~Who is the second carrier?~~ **Resolved: Telnyx (D11).** Still to validate: Malaysian/Singapore number availability and regulatory requirements, caller-ID rules, media transport options and rate limits | Second adapter scope |
| Q2 | ~~Languages~~ **Resolved (D15):** seven languages/varieties. Open: phasing in §6.3 and whether each client contracts for specific languages | Talent catalogue, dictionary, test sets |
| Q3 | Regulatory expectations on caller-ID reachability and callback in Malaysia and Singapore | Callback landing default |
| Q4 | ~~Function coverage~~ **Resolved (D12):** both cover all functions. Open: validation per language; whether to allow mixed stacks (one provider's voice with the other's reasoning) as well as single-provider stacks | Stack profile design, failover coverage |
| Q5 | ~~BYO keys~~ **Resolved (D14):** both modes exist | Tenancy of `AiProviderAccount`, metering |
| Q6 | Should regulated wording (disclosures, payment terms) be exempt from auto-publish? | Governance and compliance exposure |
| Q7 | Is the emotion register set (§7.3) acceptable, or should it be larger? | Metadata, council rubric |
| Q8 | Is mid-call provider switching acceptable to clients, or only between calls? | Failover design, customer experience |
| Q9 | Data retention and zero-retention terms with each AI provider | Privacy and contracts |
| Q10 | ~~Sticky caller-ID~~ **Resolved (D13):** random per attempt | Answer rates vs customer recognition |
| Q11 | Is the BYOAI 10% admin fee applied to Daythree's metered cost or to the client's provider invoice? Is there a minimum monthly fee? | Billing accuracy and margin |
| Q12 | Do credits also cover telephony (carrier minutes, numbers), or only AI usage? How are carrier costs charged to clients? | Credit design, unit economics |
| Q13 | When a BYOAI provider fails, may the platform fail over to Daythree AI and charge credits (opt-in), or must the client supply two providers? | Failover and billing (§5.5) |
| Q14 | Credit behaviour at zero balance: stop new outbound only, or also inbound? Grace allowance? | Service continuity, collections exposure |
| Q15 | Is the BYOAI client responsible for their provider's data terms (retention, training)? Is that stated in the contract? | Privacy and liability |

---

## 13. Suggested sequencing

| Wave | Content | Exit gate |
|---|---|---|
| A | Provider settings (telephony and AI), vault integration, Test Connection, onboarding for Twilio and Telnyx, number registry with exclusive roles and admin-only purchase | Both carriers connected; test inbound/outbound on each; keys never visible after save |
| B | Pools across providers, constrained random selection, callback landing (message and redirect), contact-point status, reputation and quarantine | Randomised outbound across both carriers with full trace; callback to an outbound line handled |
| C | Voice stack profiles, ElevenLabs and OpenAI connected, failover with talent twin, health detection, settings UI | Simulated provider outage fails over with a recorded seam and no dropped call |
| D | Talent creation flow, Malaysia/Singapore talents, pronunciation dictionary and name check | Approved talents with parity on both providers; dictionary regression passing |
| E | Pre-recording pipeline, emotion registers, council gate, stitching with continuity scoring | Asset published only after council pass; blocked transition demonstrated |
| F | Listening profiles and tuning with live data | False/missed interruption targets met |

Waves C and D are the highest-risk and should start provider capability validation early.

---

## 14. Change control

| Version | Date | Change |
|---|---|---|
| 0.2 | 17 Oct 2026 | D11–D15: Telnyx as second carrier, both AI providers cover all functions, random caller-ID, BYOAI vs Daythree AI commercial modes, seven-language scope; added §5.5 and language table |
| 0.1 | 17 Oct 2026 | Initial requirements from product owner; decisions D1–D10; AI provider settings and failover |
