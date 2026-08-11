# LabContext Agent — Product Requirements Document

| Field | Value |
|---|---|
| Version | 1.1 |
| Status | Approved |
| Date | 2026-08-11 |
| Product owner | User |
| Implementation owner | Android Context Agent workspace |

## 1. Summary

LabContext Agent is an Android technical prototype for research-lab users who need evidence-grounded help while operating electronic bench equipment. The MVP focuses on a single RIGOL DS1054Z oscilloscope workflow and demonstrates safe context acquisition, trusted manual retrieval, hybrid model routing, typed tool calls, offline degradation, and auditable behavior.

The MVP is not a general personal assistant and does not control equipment. It assists a human who remains responsible for every physical action.

## 2. Problem

Research-lab users often encounter unfamiliar instruments with model-specific menus, probe settings, and safety constraints. Searching a long manual interrupts the task, while a generic language model may identify the wrong model, invent steps, ignore local context, or make an unsafe recommendation without evidence.

The product must reduce the time required to find a relevant, cited procedure without introducing hidden recording, uncontrolled external actions, or unsupported claims.

## 3. Target user

The primary user is a university or research-institute student, researcher, or lab assistant working with electronic and communications bench equipment.

The MVP assumes:

- One user on one Android device.
- The user deliberately starts every inspection session.
- The user can legally obtain and import the relevant manufacturer manual.
- The user understands that the app is an assistant, not a safety-certified controller.

## 4. Goals

1. Complete one evidence-first oscilloscope assistance workflow end to end.
2. Demonstrate safe fusion of camera, user intent, local device state, optional semantic location, selected calendar context, and low-sensitivity sensor facts.
3. Keep raw sensitive data local whenever practical and minimize cloud payloads.
4. Route work explicitly between deterministic local logic, supported on-device inference, cloud inference, and offline fallback.
5. Prevent manual or model content from bypassing permission and tool policies.
6. Produce automated proof for functionality, grounding, privacy, tool safety, latency, and supported Android versions.

## 5. Non-goals

The MVP does not provide:

- General assistance across arbitrary devices.
- Autonomous equipment control.
- Continuous environmental monitoring.
- Notification-listener access.
- Background camera, microphone, or precise location access.
- Open web research.
- Accounts, cloud history, synchronization, teams, billing, or a web dashboard.
- Play Store publication or production branding.
- Medical, industrial-safety, or regulatory certification.

## 6. Product principles

### 6.1 User initiation

Sensitive capture and inference start only after an explicit user action in a visible session.

### 6.2 Evidence before fluency

A cited and limited answer is preferred to an uncited, fluent answer. No evidence means abstention.

### 6.3 Local before cloud

Permission enforcement, context fusion, redaction, storage, and policy checks run locally. Cloud inference receives only the minimum data required for the approved task.

### 6.4 Human authority

The user confirms external writes and performs every physical action. The model cannot grant itself authority.

### 6.5 Testable degradation

Unsupported hardware, denied permission, no network, model errors, conflicting context, and malicious documents have explicit behaviors that can be tested.

## 7. Golden workflow

### Preconditions

- The app is installed on Android API 26 or later.
- A synthetic test manual or a user-selected RIGOL DS1054Z manual is imported and indexed.
- Camera permission is available for image capture.
- Microphone permission is requested only if push-to-talk is used.
- A cloud configuration is optional; the deterministic fake provider is available in development.

### Main flow

1. The user taps `開始設備檢查`.
2. The app displays what data will be used and requests only permissions required by the selected action.
3. The user captures a still image of the oscilloscope front panel.
4. The app minimizes image metadata and generates an equipment-identification request.
5. The system returns RIGOL DS1054Z with a confidence score or asks the user for another image.
6. The user confirms the identified model.
7. The app selects the matching imported manual by model metadata and document hash.
8. The user asks how to verify the CH1 probe ratio and set the vertical scale using text or push-to-talk.
9. The agent retrieves relevant manual passages.
10. The policy layer validates source, citation, scope, and safety requirements.
11. The app displays a Traditional Chinese checklist with original English control labels and page or section citations.
12. The user performs each step and marks it complete.
13. The user may dictate a result note; raw audio is deleted after transcription.
14. The app stores a local experiment record.
15. The app may preview a calendar reminder.
16. The calendar is written only after explicit confirmation.

### Required failure branches

- Permission denied: explain the missing capability and offer text-only or manual-only behavior.
- Low identification confidence: abstain and request a clearer image or manual model selection.
- Conflicting model evidence: ask the user to confirm the model label.
- Missing manual: offer manual import; do not answer from model memory.
- Missing citation: block the substantive answer.
- Malicious manual instruction: ignore the instruction, record a security event, and preserve policy.
- Offline: search cached manual content and allow a local record; do not run cloud reasoning or external writes.
- Cloud timeout or quota: apply bounded retry, then degrade without duplicating tool effects.
- Calendar permission denied: preserve the reminder as a local draft.

## 8. Functional requirements

### FR-001 — Inspection session

The user can start, view, complete, cancel, and delete an inspection session. Sensitive context collection ends when the visible session ends.

**Acceptance:** A session has a unique identifier, start and end timestamps, status, selected equipment, selected manual, derived context snapshot, checklist, notes, and tool audit references.

### FR-002 — Progressive permissions

The app requests camera, microphone, notification, location, and calendar permissions only when the corresponding feature is invoked. Denial must not crash or dead-end the app.

**Acceptance:** Automated tests cover first denial, permanent denial, later grant, revocation, and process restart for every requested permission.

### FR-003 — Still-image capture

The app captures or imports one still image for equipment identification. Continuous video is excluded.

**Acceptance:** EXIF and unnecessary metadata are removed before any cloud request. The raw image is deleted after inference unless the user explicitly attaches it to a pinned local record.

### FR-004 — Equipment identification

The system identifies the supported equipment and produces model, confidence, evidence cues, and an abstention reason.

**Acceptance:** The system never silently maps an unsupported model to RIGOL DS1054Z. Identification or safe abstention meets the defined 90% test threshold.

### FR-005 — Manual import and registry

The user imports a PDF using Android Storage Access Framework. The app records display name, local URI grant, manufacturer, model, content hash, index version, and import time.

**Acceptance:** A changed document hash invalidates the old index. The user can delete a manual and all derived index data.

### FR-006 — Evidence retrieval and citations

The system retrieves manual passages for a user question and returns citations with page or section locators.

**Acceptance:** Every substantive procedure or safety claim has at least one validated citation. Missing evidence produces an abstention rather than an uncited answer.

### FR-007 — Text and push-to-talk input

The user can ask a question by text or a press-and-hold voice action.

**Acceptance:** The microphone is active only while the user holds or explicitly activates the control. Raw audio is deleted after transcription, including error and cancellation paths.

### FR-008 — Context fusion

The app fuses typed facts into a deterministic `ContextSnapshot`.

Required fact classes are:

- Explicit user intent.
- Confirmed equipment model.
- Selected manual identity.
- Camera or OCR observation.
- Network availability.
- Battery and power state.
- Optional local semantic location.
- Optional selected-calendar context.
- Optional low-sensitivity motion or device-state signal.

**Acceptance:** Every fact has source, timestamp, TTL, confidence, and sensitivity. Conflicts follow the approved authority order and cannot be silently resolved by an LLM.

### FR-009 — Model routing

The app selects deterministic local logic, Gemini Nano, Firebase cloud inference, or offline fallback based on task capability, foreground status, runtime model availability, privacy, network, and battery state.

**Acceptance:** Routing decisions and fallback reasons are recorded without raw PII. Unsupported Nano devices automatically use an allowed alternative.

### FR-010 — Grounded checklist

The app converts validated evidence into a Traditional Chinese checklist while preserving original English equipment labels.

**Acceptance:** Each checklist item is traceable to a cited passage. The checklist contains an assistance disclaimer and never claims the app performed the physical step.

### FR-011 — Local experiment record

The user can save a record containing equipment identity, manual hash, question, cited checklist, confirmations, text note, timestamps, and redacted audit references.

**Acceptance:** Unpinned records expire after seven days. Pinned records remain until deleted. The user can delete one record or all records.

### FR-012 — Calendar reminder

The app can create a preview of a follow-up reminder. It writes the event only after explicit confirmation.

**Acceptance:** Canceling or dismissing the preview creates no external event. Repeated confirmation cannot create duplicate events.

### FR-013 — Notifications

The app may post its own completion or reminder notifications after permission is granted.

**Acceptance:** The app does not request notification-listener access and cannot read other applications' notification content.

### FR-014 — Offline fallback

The app remains useful without network access through imported manual search, deterministic checklist behavior, and local records.

**Acceptance:** Offline mode is visibly identified and does not claim cloud or Nano output that was not produced. It may save a reminder as a local draft, but it does not commit any external calendar action.

### FR-015 — Audit trail

The app records model route, evidence locators, policy decisions, tool name, validated arguments, confirmation state, result, error category, duration, and retry count.

**Acceptance:** Audit entries contain no raw image, audio, precise coordinates, calendar description, manual passage beyond the minimum locator, API key, or authentication token.

### FR-016 — Data controls

The user can inspect and delete sessions, pinned records, imported manuals, derived indexes, and audit data.

**Acceptance:** Deletion removes both the primary record and derived local artifacts. The app provides a complete local data reset.

## 9. Permission matrix

| Capability | Android permission or mechanism | Trigger | MVP policy |
|---|---|---|---|
| Still photo | Camera permission or system picker | User taps capture/import | Foreground session only |
| Push-to-talk | Microphone permission | User activates voice input | No passive listening |
| Location | While-in-use coarse or fine location | User enables semantic-location context | No background location |
| App notifications | Post-notification permission where required | User enables reminders | App output only |
| Other app notifications | Notification listener | None | Prohibited |
| Calendar read | Calendar provider access | User requests context or preview | Minimum selected data only |
| Calendar write | Calendar provider or system intent | User confirms preview | Idempotent confirmed write |
| Low-sensitivity sensors | Platform sensor APIs where permissionless or minimally permissioned | User enables context | Event-driven only |
| Body or health sensors | Body/health permissions | None | Excluded |
| Manual import | Storage Access Framework | User selects PDF | Persist only selected URI grant |

## 10. Data lifecycle

| Data | Storage | Retention | Cloud policy |
|---|---|---|---|
| Raw equipment photo | Memory or temporary private file | Deleted after inference unless explicitly attached to a pinned record | Minimized image only when cloud identification is approved |
| Raw voice audio | Temporary private file or stream | Deleted after transcription on success, error, or cancellation | Not retained by the app |
| Voice transcript | Session record | Seven days unless pinned | Send only when required for the approved question |
| Precise location | Memory only | Current operation | Never uploaded as coordinates |
| Semantic location label | Context snapshot | Session TTL | Optional and minimized |
| ContextSnapshot | Local session state | Session lifetime; redacted summary may remain in the record | Only required fields |
| Imported manual | App-private access through user-selected URI | Until user deletion | Manual passages only when required and permitted |
| Manual index | App-private storage | Until hash change or deletion | No bulk upload by default |
| Experiment record | App-private database | Seven days unless pinned | No cloud history in MVP |
| Audit record | App-private database | Same as associated unpinned session | Redacted diagnostics only when explicitly exported |
| Calendar event ID | Experiment record | Same as record | No calendar body in cloud logs |

## 11. Non-functional requirements

### NFR-001 — Security

- All tools are allowlisted and typed.
- All external writes require policy validation.
- Level 2 actions require a complete preview and explicit confirmation.
- Level 3 tools do not exist.
- Manual, OCR, model, and network content are untrusted inputs.
- Secrets, tokens, and Firebase production configuration are excluded from versioned files.

### NFR-002 — Privacy

- Raw sensitive media is ephemeral by default.
- PII is minimized before cloud use.
- Precise location is never uploaded.
- The app exposes deletion controls and avoids hidden telemetry.

### NFR-003 — Performance

- P95 context fusion is below 200 ms on the documented test environment.
- P95 supported on-device decision latency is below 1.5 seconds on a tested supported device.
- P95 cloud recommendation latency is below 5 seconds on the documented test network.

### NFR-004 — Battery

The app uses event-driven work and performs no continuous sampling. Additional battery use during the defined eight-hour background-idle test must not exceed 3% on the documented reference device.

### NFR-005 — Reliability

- Retries are bounded and limited to transient operations.
- External writes use idempotency protection.
- Process restart preserves only approved local state.
- Every failure mode produces an actionable user message and a stable internal error category.

### NFR-006 — Compatibility

- Minimum Android API 26.
- Compile against Android API 36.1 and target Android API 36.
- UI and mock/cloud paths run on AVD API 26 and API 36.
- Gemini Nano is feature-gated by runtime capability checks and verified only on supported real devices.

### NFR-007 — Accessibility and language

- User-visible MVP content is Traditional Chinese.
- Original English equipment control names remain visible beside translations.
- Interactive controls have accessible labels and do not rely on color alone.

### NFR-008 — Maintainability

- ADK, Firebase, ML Kit, and model implementations remain behind project-owned interfaces.
- Preview dependency versions are pinned.
- Schema and provider boundaries have contract tests.

## 12. Tool catalog

| Tool | Level | Effect | Confirmation |
|---|---:|---|---|
| `read_context_snapshot` | 0 | Read approved local context facts | No |
| `search_manual` | 0 | Search selected trusted manual | No |
| `read_manual_passage` | 0 | Read a cited local passage | No |
| `create_experiment_draft` | 1 | Create reversible local draft | No |
| `update_experiment_draft` | 1 | Edit reversible local draft | No |
| `preview_calendar_reminder` | 1 | Create local preview | No |
| `commit_calendar_reminder` | 2 | Write external calendar event | Yes |
| `attach_photo_to_pinned_record` | 2 | Retain otherwise ephemeral media | Yes |

No arbitrary browser, shell, message, email, call, purchase, system-setting, or equipment-control tool is exposed.

## 13. Evaluation plan

### 13.1 Equipment identification

- At least 40 labeled images.
- Coverage includes angle, distance, lighting, glare, blur, partial occlusion, and confusing unsupported devices.
- Pass condition: at least 90% correct supported identification or safe abstention.

### 13.2 Manual grounding

- At least 25 labeled questions against the synthetic fixture and approved user-imported documents.
- Pass conditions: citations on 100% of substantive answers; at least 90% answer correctness; abstention when evidence is unavailable.

### 13.3 Context fusion

- At least 25 deterministic fixtures.
- Include stale facts, contradictory facts, permission denial, low confidence, offline state, low battery, foreground transitions, and document hash change.
- Pass condition: 100% expected provenance, confidence, TTL, and conflict behavior.

### 13.4 Safety and privacy

- At least 20 adversarial fixtures.
- Include prompt injection in PDFs, OCR instructions, secret requests, malformed tool arguments, repeated confirmations, denied permissions, precise-location leakage, and calendar duplication.
- Pass conditions: zero unauthorized side effects and zero raw PII disclosures.

### 13.5 Android verification

- Unit tests for domain and policy logic.
- Instrumentation tests on AVD API 26 and API 36.
- One unsupported real-device fallback test when hardware is available.
- Gemini Nano verification on a supported locked-bootloader device when available.

### 13.6 Evidence artifacts

`/prove-it` must preserve:

- Test reports and command outputs.
- Redacted context and tool simulation logs.
- Screenshots of required states.
- A recording of the golden workflow where feasible.
- Device, Android version, model runtime, network, and dependency versions.
- An explicit distinction between real, simulated, mocked, and unverified results.

## 14. MVP completion criteria

The MVP is complete only when:

1. Every included functional requirement has a passing test or documented manual proof.
2. Every quality gate passes or is explicitly classified as unverified because required hardware or credentials were not provided.
3. No excluded feature has entered the implementation scope.
4. No secret or raw PII appears in source control, test artifacts, screenshots, or logs.
5. The golden workflow completes without equipment control or an unauthorized external write.
6. The Completion Report lists changed files, architecture, test results, manual verification, remaining risks, and recommended next work.

## 15. Deferred backlog

Deferred work includes:

- Additional oscilloscope models and equipment categories.
- Background proactive suggestions.
- Notification-content context.
- Account, sync, organization, and shared equipment libraries.
- Cloud history and multi-device use.
- Production analytics and billing.
- Play Store publication and final package identity.
- Direct equipment connectivity or control.
- Safety certification.

Deferred items cannot enter the MVP without a PRD revision and explicit user approval.
