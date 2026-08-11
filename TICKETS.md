# LabContext Agent — Ticket Board

| Field | Value |
|---|---|
| Version | 1.0 |
| Date | 2026-08-11 |
| Source | PRD 1.1, ADR 1.1, CONTEXT 1.1 |
| Current phase | /to-tickets complete; implementation plan awaiting approval |
| Implementation status | Not started |
| Ticket count | 24 |

## Execution rules

1. Execute exactly one ticket at a time.
2. A ticket owns one responsibility package or one delivery concern.
3. Start with the named failing test.
4. Run the named test and confirm the expected failure before implementation.
5. Implement only the minimum behavior required by the ticket.
6. Run the ticket-specific test command immediately after the change.
7. Run the regression command listed in the ticket.
8. Commit only the ticket files with the specified commit message.
9. Do not start the next ticket until the current Definition of Done is satisfied.
10. Do not commit Firebase credentials, service-account keys, raw PII, imported manuals, captured photos, audio, or private test artifacts.
11. Real, simulated, mocked, and unverified results must remain explicitly distinguishable.
12. A failing hardware or credential checkpoint cannot be converted into a passing mock claim.

## Dependency waves

| Wave | Tickets | Result |
|---|---|---|
| 0 | ACA-000, ACA-001 | Reproducible Android toolchain and bootable Traditional Chinese app shell |
| 1 | ACA-002 through ACA-004 | Testable session, permission, and context foundation |
| 2 | ACA-005 through ACA-008 | Imported manual becomes a cited local checklist |
| 3 | ACA-009 through ACA-014 | Photo identification and explicit local, Nano, Firebase, and ADK routing |
| 4 | ACA-015 through ACA-019 | Policy, audit, records, calendar, and push-to-talk safety |
| 5 | ACA-020 | Complete golden workflow |
| 6 | ACA-021 through ACA-023 | Fixed evaluations, Android verification, CI, and proof-ready documentation |

## Ticket index

### ACA-000 — Android toolchain gate

**Owner:** Development environment
**Depends on:** None
**Outcome:** JDK 17, Android SDK 26 and 36/36.1, platform tools, emulator, and two named AVDs are available.

**Definition of Done**

- java -version reports JDK 17.
- adb version, sdkmanager --list, and emulator -list-avds succeed.
- LabContext_API_26 and LabContext_API_36 exist.
- docs/environment-verification.md records exact paths and versions without secrets.
- No project source code is created by this ticket.

### ACA-001 — Bootable Compose shell

**Owner:** Build and app shell
**Depends on:** ACA-000
**Outcome:** A Traditional Chinese LabContext Agent shell builds and launches on both AVD baselines.

**Definition of Done**

- Gradle Wrapper 9.6.1 and AGP 9.3.1 are pinned.
- minSdk is 26, compileSdk is 36.1, and targetSdk is 36.
- The app namespace is dev.labcontext.agent.
- assembleDebug and testDebugUnitTest pass.
- A launch instrumentation test passes on API 26 and API 36.

### ACA-002 — Inspection session state

**Owner:** core/session
**Depends on:** ACA-001
**Outcome:** A deterministic state machine can start, complete, cancel, expire, and delete an inspection session.

**Definition of Done**

- Invalid status transitions return a typed error.
- Session identifiers, timestamps, equipment confirmation, checklist progress, and note text are represented.
- Unit tests cover every transition and process-restoration serialization.
- No Android framework type enters the session domain model.

### ACA-003 — Progressive permission coordinator

**Owner:** core/permissions
**Depends on:** ACA-002
**Outcome:** Camera, microphone, location, notification, calendar read, and calendar write permissions produce explicit capability states.

**Definition of Done**

- First denial, permanent denial, grant, revocation, and unavailable capability are represented.
- No permission is requested during app startup.
- Background location and notification-listener capabilities do not exist.
- Unit tests pass and a Compose denial-state test passes.

### ACA-004 — Deterministic context fusion

**Owner:** core/context
**Depends on:** ACA-002
**Outcome:** Typed facts become a ContextSnapshot with provenance, confidence, TTL, sensitivity, and explicit conflicts.

**Definition of Done**

- The accepted authority order is encoded.
- Expired facts never appear as active facts.
- Equal-authority contradictions produce a conflict instead of silent selection.
- The 25-case context fixture format is established.
- Unit tests pass with a deterministic fake clock.

### ACA-005 — Manual registry and Storage Access Framework import

**Owner:** core/manual
**Depends on:** ACA-001
**Outcome:** The user can select a PDF and register its manufacturer, model, URI grant, SHA-256 hash, and index version.

**Definition of Done**

- Non-PDF input is rejected.
- Re-importing the same hash is idempotent.
- A changed hash invalidates the previous index.
- Deleting a manual deletes its derived index.
- Tests use in-memory documents and contain no manufacturer PDF.

### ACA-006 — PDF rendering and local OCR index

**Owner:** core/manualindex
**Depends on:** ACA-005
**Outcome:** A user-imported PDF is rendered page by page, OCRed locally, and indexed through an observable WorkManager job.

**Definition of Done**

- PdfRenderer resources close on success, failure, and cancellation.
- ML Kit Text Recognition 16.0.1 is used for bundled Latin OCR.
- Index progress and failure category are observable.
- Cancellation leaves no partial active index.
- A synthetic PDF fixture produces the expected page text.

### ACA-007 — Evidence retrieval and citation validation

**Owner:** core/evidence
**Depends on:** ACA-006
**Outcome:** A question retrieves ranked manual passages and validates page or section citations.

**Definition of Done**

- Retrieval is restricted to the selected manual hash.
- Each passage includes document ID, page, section, text, and score.
- A substantive answer with no valid citation is rejected.
- Quoted evidence cannot exceed the selected local passage.
- At least five initial grounding cases pass.

### ACA-008 — Grounded checklist composer

**Owner:** core/checklist
**Depends on:** ACA-007
**Outcome:** Retrieved evidence becomes a Traditional Chinese checklist with original English control labels.

**Definition of Done**

- Every checklist item has at least one validated citation.
- Safety-critical items are visually and structurally marked.
- Missing evidence produces abstention.
- The synthetic CH1 probe-ratio question produces the fixed expected checklist.
- No model or Android framework type appears in the composer interface.

### ACA-009 — Still-photo capture and metadata minimization

**Owner:** platform/camera
**Depends on:** ACA-003
**Outcome:** A visible session can capture one still image and hand a minimized private URI to identification.

**Definition of Done**

- CameraX 1.6.1 is lifecycle-bound.
- Capture cannot start without an active session and granted camera capability.
- EXIF and unnecessary metadata are removed from the derivative.
- Temporary media is deleted on success, cancellation, error, and session deletion.
- No continuous video or background camera service exists.

### ACA-010 — Equipment identification and safe abstention

**Owner:** core/identification
**Depends on:** ACA-009, ACA-004
**Outcome:** OCR cues and a model provider return RIGOL DS1054Z or a typed abstention.

**Definition of Done**

- Unsupported equipment is never silently mapped to RIGOL DS1054Z.
- Confidence below the accepted threshold produces abstention.
- Conflicting OCR and model evidence requires user confirmation.
- A deterministic fake provider covers correct, low-confidence, conflict, and unsupported outcomes.
- The result records evidence cues without retaining the raw photo.

### ACA-011 — Explicit model router

**Owner:** core/routing
**Depends on:** ACA-004, ACA-010
**Outcome:** Every task is routed to deterministic, Nano, Firebase, fake, or offline behavior with a recorded reason.

**Definition of Done**

- Routing considers task capability, foreground state, runtime availability, network, battery, privacy, and configured providers.
- Offline routing never commits an external tool action.
- Unsupported Nano falls back without crashing.
- Unit tests cover the complete decision table.
- No provider can bypass the policy layer.

### ACA-012 — Gemini Nano adapter

**Owner:** adapters/mlkit
**Depends on:** ACA-011
**Outcome:** Supported short text work can use ADK ML Kit 0.7.0-beta and ML Kit Prompt API while unsupported devices fail safely.

**Definition of Done**

- checkStatus runs before the capability is shown.
- Foreground-only, busy, quota, download, and unlocked-bootloader failures map to typed availability.
- Input and output limits are enforced.
- Unit tests use an adapter fake.
- Real-device results are marked unverified until run on a supported locked device.

### ACA-013 — Firebase cloud adapter

**Owner:** adapters/firebase
**Depends on:** ACA-011
**Outcome:** The app builds without credentials and can use Firebase AI Logic when an authorized configuration is supplied.

**Definition of Done**

- Firebase BoM 34.16.0 and firebase-ai are pinned through the version catalog.
- Missing configuration returns ConfigMissing and does not make a network request.
- Debug App Check and release Play Integrity providers are separated.
- No Gemini secret, Admin SDK key, or service-account file exists in the repository.
- Fake and configuration-missing tests pass.

### ACA-014 — ADK agent engine and typed tool proposals

**Owner:** core/agent
**Depends on:** ACA-008, ACA-011, ACA-013
**Outcome:** ADK 0.7.0 coordinates evidence-grounded checklist requests and returns typed tool proposals without executing them.

**Definition of Done**

- google-adk-kotlin-core 0.7.0, firebase-android 0.7.0, processor 0.7.0, and ML Kit Android 0.7.0-beta are pinned.
- Agent input contains only the minimized ContextSnapshot and selected evidence.
- Tool proposals are data until the policy and confirmation layers approve them.
- A fake-model agent test produces the expected checklist and reminder preview.
- ADK types remain inside the adapter package.

### ACA-015 — Policy engine and prompt-injection defense

**Owner:** core/policy
**Depends on:** ACA-014
**Outcome:** Evidence, model output, OCR, and tool proposals are evaluated under one deterministic policy.

**Definition of Done**

- Manual text cannot change policy or request secrets.
- Substantive uncited output is blocked.
- Level 2 actions require a separate user confirmation token.
- Level 3 tool names are rejected even when proposed by a model.
- At least ten initial adversarial tests pass with zero unauthorized effects.

### ACA-016 — Redacted audit log

**Owner:** core/audit
**Depends on:** ACA-015
**Outcome:** Routing, evidence locators, policy decisions, tool proposals, results, errors, timing, and retries are locally auditable.

**Definition of Done**

- Raw photos, audio, precise coordinates, calendar bodies, manual passages, and credentials cannot be serialized into audit rows.
- Every event has session ID, event type, timestamp, redaction version, and correlation ID.
- Audit rows expire with an unpinned session.
- Export is explicit and redacted.
- Schema and redaction unit tests pass.

### ACA-017 — Local experiment records and retention

**Owner:** core/records
**Depends on:** ACA-002, ACA-016
**Outcome:** Room stores local experiment records, pinned state, and seven-day expiry.

**Definition of Done**

- Room 2.8.4 and KSP schema export are configured.
- Create, read, pin, unpin, delete, and delete-all operations pass.
- Expiry uses an injected clock.
- Deleting a record also removes attached media and audit rows.
- No cloud synchronization code exists.

### ACA-018 — Calendar preview and idempotent confirmed write

**Owner:** platform/calendar
**Depends on:** ACA-003, ACA-015, ACA-017
**Outcome:** A reminder preview becomes one calendar event only after explicit confirmation.

**Definition of Done**

- Preview is a reversible local draft.
- Cancel and dismiss create no event.
- A confirmation token is single-use and session-bound.
- Repeated commit calls with the same idempotency key create at most one event.
- Offline mode preserves the local draft and does not commit.

### ACA-019 — Push-to-talk transcription

**Owner:** platform/speech
**Depends on:** ACA-003, ACA-017
**Outcome:** A visible session can capture a press-and-hold utterance and save only the resulting text.

**Definition of Done**

- Microphone use requires active foreground session and granted capability.
- Listening stops on release, cancellation, lifecycle stop, and error.
- The app does not create a persistent audio file.
- Empty or failed recognition leaves the existing note unchanged.
- Fake-recognizer unit tests and a permission instrumentation test pass.

### ACA-020 — Golden inspection workflow

**Owner:** feature/inspection
**Depends on:** ACA-008 through ACA-019
**Outcome:** The accepted 16-step golden flow is usable from one Compose navigation graph.

**Definition of Done**

- The user can start a session, capture or import a photo, confirm equipment, select a manual, ask the fixed question, review citations, complete checklist items, dictate a note, save a record, and preview or confirm a reminder.
- Permission denial, low confidence, missing manual, offline, policy rejection, and calendar denial are visible recoverable states.
- Process recreation restores only approved local state.
- A fake-provider end-to-end instrumentation test passes on API 26 and API 36.

### ACA-021 — Fixed evaluation datasets and runner

**Owner:** evaluation
**Depends on:** ACA-020
**Outcome:** Identification, grounding, context, and adversarial cases run from versioned manifests with reproducible scoring.

**Definition of Done**

- Manifests define at least 40 image cases, 25 grounding questions, 25 context scenarios, and 20 adversarial cases.
- Repository assets are synthetic, generated, or redistribution-safe.
- The runner reports correct, abstained, failed, unauthorized-effect, and PII-disclosure counts.
- Threshold failures return a nonzero process result.
- No real user data is included.

### ACA-022 — Android, latency, and battery verification

**Owner:** verification
**Depends on:** ACA-021
**Outcome:** Required compatibility and performance gates produce machine-readable evidence.

**Definition of Done**

- Unit and instrumentation suites pass on the configured environments.
- API 26 and API 36 AVD reports are saved.
- Context fusion P95, cloud latency P95, supported Nano latency P95, and eight-hour idle battery delta have explicit result fields.
- Missing real hardware or Firebase credentials produces UNVERIFIED, not PASS.
- Screenshots and logs are redacted.

### ACA-023 — CI, public documentation, and proof handoff

**Owner:** delivery
**Depends on:** ACA-022
**Outcome:** The public repository explains the project, runs safe automated checks, and is ready for the separate /prove-it phase.

**Definition of Done**

- README documents scope, architecture, setup, privacy, limitations, and test commands.
- GitHub Actions runs formatting, unit tests, lint, and credential scanning without Firebase secrets.
- prove-it directories and artifact naming conventions are documented.
- CONTEXT and ticket statuses identify implementation completion without claiming unverified integrations.
- The repository is clean and all commits are pushed.

## Requirements traceability

| Requirement | Tickets |
|---|---|
| FR-001 Inspection session | ACA-002, ACA-020 |
| FR-002 Progressive permissions | ACA-003, ACA-020 |
| FR-003 Still-image capture | ACA-009 |
| FR-004 Equipment identification | ACA-010, ACA-021 |
| FR-005 Manual import and registry | ACA-005, ACA-006 |
| FR-006 Evidence retrieval and citations | ACA-007 |
| FR-007 Text and push-to-talk input | ACA-019, ACA-020 |
| FR-008 Context fusion | ACA-004 |
| FR-009 Model routing | ACA-011 through ACA-014 |
| FR-010 Grounded checklist | ACA-008, ACA-020 |
| FR-011 Local experiment record | ACA-017 |
| FR-012 Calendar reminder | ACA-018 |
| FR-013 Notifications | ACA-003, ACA-020 |
| FR-014 Offline fallback | ACA-011, ACA-018, ACA-020 |
| FR-015 Audit trail | ACA-016 |
| FR-016 Data controls | ACA-005, ACA-016, ACA-017 |
| NFR-001 Security | ACA-015, ACA-018, ACA-021 |
| NFR-002 Privacy | ACA-003, ACA-009, ACA-016, ACA-021 |
| NFR-003 Performance | ACA-022 |
| NFR-004 Battery | ACA-022 |
| NFR-005 Reliability | ACA-011, ACA-018, ACA-020 |
| NFR-006 Compatibility | ACA-000, ACA-001, ACA-012, ACA-022 |
| NFR-007 Accessibility and language | ACA-001, ACA-008, ACA-020 |
| NFR-008 Maintainability | ACA-001, ACA-011 through ACA-014 |

## Execution gate

No ticket is authorized for implementation until this board and the detailed plan are approved. The first executable ticket is ACA-000.
