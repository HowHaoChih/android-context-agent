# LabContext Agent — Architecture Decision Records

| Field | Value |
|---|---|
| Version | 1.0 |
| Date | 2026-08-11 |
| Scope | Android Context Agent MVP |

## ADR-001 — Build a bounded laboratory equipment assistant

**Status:** Accepted

### Context

A general ambient assistant, a field-maintenance product, and an Android agent SDK would require different trust, distribution, data, and evaluation models. The MVP needs a concrete commercial and technical wedge.

### Decision

Build a user-initiated assistant for university and research-lab users. Prove one RIGOL DS1054Z oscilloscope workflow. The app identifies equipment, grounds guidance in a trusted manual, presents a checklist, records the result, and may create a confirmed reminder.

### Consequences

- The workflow is demonstrable and objectively testable.
- Liability remains bounded because the app does not operate equipment.
- Additional devices require later adapters, evidence sets, and regression tests.

### Rejected alternatives

- A general personal context assistant was rejected because its permission and evaluation surface is too broad.
- Industrial field maintenance was deferred because it introduces certification and operational-liability concerns.
- An SDK-only product was rejected because it would not validate an end-user workflow.

## ADR-002 — Use explicit, user-initiated sessions

**Status:** Accepted

### Context

Continuous camera, microphone, location, notification, and sensor use conflicts with Android background restrictions, battery goals, privacy expectations, and a narrow MVP.

### Decision

Camera, push-to-talk, cloud reasoning, and tools operate only inside a visible session initiated by the user. Low-sensitivity context may be prepared locally, but it cannot trigger hidden capture or external action.

### Consequences

- Permission prompts are explainable and contextual.
- Background battery use and accidental recording risk are reduced.
- Proactive ambient assistance is deferred.

### Rejected alternatives

- A continuous agent loop was rejected because it is unnecessary for the tracer workflow.
- Background camera and microphone services were rejected.
- Manual-only operation remains a fallback, not the primary interaction model.

## ADR-003 — Use Kotlin, Compose, and isolated ADK integration

**Status:** Accepted

### Context

ADK for Kotlin/Android enables Android-native agent orchestration but is a Preview dependency with a changing API. The implementation needs current capabilities without coupling the product domain to a prerelease framework.

### Decision

Use Kotlin, Jetpack Compose, JDK 17, `minSdk 26`, `compileSdk 36`, and `targetSdk 36`. Pin ADK for Kotlin/Android to `0.7.0` for the initial implementation. Use the internal development namespace `dev.labcontext.agent`.

ADK, Firebase, ML Kit, and model types remain behind project-owned interfaces and adapters. Dependency upgrades require an ADR update and full regression verification.

### Consequences

- The project follows a modern Android-native stack.
- Preview changes are localized to adapters.
- The implementation carries adapter and contract-test overhead.

### Rejected alternatives

- Pinning the initial `0.1.0` release was rejected because the official repository has advanced to `0.7.0` as of 2026-08-11.
- A cross-platform UI was rejected because it adds abstraction without helping the Android-specific goal.
- Direct ADK types throughout the domain were rejected due to upgrade risk.

## ADR-004 — Use explicit hybrid model routing

**Status:** Accepted

### Context

On-device models improve privacy, offline behavior, latency, and cost, but Gemini Nano availability depends on hardware, runtime state, foreground status, model download, and quota. Complex multimodal reasoning and tool planning require a cloud-capable path.

### Decision

Use deterministic local logic for permissions, redaction, context fusion, policy, and simple fallback. Use Gemini Nano only for supported short text tasks after a runtime capability check. Use Firebase AI Logic for cloud image reasoning, evidence synthesis, and ADK tool planning. Use App Check for real cloud environments.

The application performs routing explicitly through `ModelRouter`; it does not depend on experimental automatic hybrid routing for correctness.

### Consequences

- The app can run on unsupported devices through safe fallback.
- The app can preserve useful offline behavior.
- Real cloud and Nano claims require separate proof.
- Routing logic becomes a first-class tested component.

### Rejected alternatives

- On-device-only operation was rejected because the required multimodal and tool capabilities are not consistently available.
- Cloud-first raw context upload was rejected because it weakens privacy and offline behavior.
- Embedding a cloud API key in the APK was rejected.
- Treating a model name as proof of runtime availability was rejected; runtime status checks are mandatory.

## ADR-005 — Fuse context deterministically with provenance

**Status:** Accepted

### Context

Camera observations, manual metadata, user intent, calendar context, semantic location, device state, and sensors can conflict or become stale. A language model must not silently invent a coherent state from inconsistent inputs.

### Decision

Represent context as typed facts with source, timestamp, TTL, confidence, and sensitivity. Produce `ContextSnapshot` through deterministic fusion rules. Apply this authority order:

1. Explicit user intent and confirmation.
2. Selected manufacturer manual.
3. User-imported document.
4. Camera or OCR observation.
5. Semantic location and selected calendar facts.
6. Low-sensitivity sensor inference.

Conflict or insufficient confidence causes an explicit question or abstention.

### Consequences

- Context decisions are reproducible and testable.
- The agent receives a smaller, structured context.
- New context sources require a typed provider and conflict rules.

### Rejected alternatives

- Continuous LLM-based fusion was rejected because it is nondeterministic, expensive, and difficult to audit.
- Newest-value-wins was rejected because recency does not determine authority.

## ADR-006 — Enforce least privilege and local-first retention

**Status:** Accepted

### Context

Camera, microphone, location, notifications, calendar, and sensor data have different Android restrictions and privacy risks. Requesting all permissions at startup would undermine trust and complicate Play policy compliance.

### Decision

Request permissions progressively when a feature is invoked. Camera and microphone are foreground-session only. Voice input is push-to-talk. Location is while-in-use and locally reduced to a semantic label. Background location, notification-listener access, and body or health sensors are excluded. Calendar writes require preview and confirmation.

Raw photos and audio are ephemeral by default. Unpinned derived records expire after seven days. Manuals remain local until user deletion. Precise coordinates never enter cloud payloads.

### Consequences

- The app has a smaller privacy and permission surface.
- Some proactive context capabilities are unavailable in the MVP.
- Deletion and expiry behavior must be implemented and tested as product features.

### Rejected alternatives

- One-time blanket permission requests were rejected.
- Continuous background capture was rejected.
- Cloud history and multi-device synchronization were deferred.

## ADR-007 — Treat manuals as trusted evidence but untrusted instructions

**Status:** Accepted

### Context

The agent needs authoritative technical evidence, but PDF content can contain mistakes, malicious prompt injection, or text unrelated to the device. Redistributing manufacturer PDFs may also create licensing concerns.

### Decision

Users import manuals through Android Storage Access Framework. The app records source metadata, hash, and index version. Manufacturer manuals and deliberately imported documents are allowed evidence sources. The repository and APK do not redistribute the full RIGOL manual. Automated tests use an original synthetic manual fixture.

Every substantive procedure and safety claim requires a validated page or section citation. Manual content cannot change system policy, request secrets, authorize tools, or bypass confirmation.

### Consequences

- Answers are traceable and safer.
- Users must import a real manual for real-device use.
- The parser, index, citation validator, and injection defenses become core components.

### Rejected alternatives

- Answering from model memory was rejected.
- Open-ended web search was rejected.
- Bundling a manufacturer PDF without confirmed redistribution rights was rejected.

## ADR-008 — Use effect-level tool authorization

**Status:** Accepted

### Context

An agent that reads context, stores notes, and writes calendar events has different risk levels. A single unrestricted tool-execution policy would allow prompt injection or model error to cause side effects.

### Decision

Classify tools by effect:

- Level 0 read-only tools may run automatically.
- Level 1 reversible local-write tools may run automatically and must support undo.
- Level 2 external or sensitive persistence tools require a complete preview and explicit user confirmation.
- Level 3 communication, purchase, settings, and equipment-control tools do not exist.

All tools have typed validated inputs, bounded retries, timeout handling, effect-aware idempotency, and a redacted audit event.

### Consequences

- Tool behavior is explainable and testable.
- The UI must present confirmation separately from model output.
- Future tools require an explicit level and threat analysis.

### Rejected alternatives

- Model-decided authorization was rejected.
- Confirmation for every read was rejected because it would make the workflow unusable.
- Automatic calendar writes were rejected.

## ADR-009 — Use credential-free development with explicit live-integration proof

**Status:** Accepted

### Context

The initial workspace has no approved Firebase project or cloud credentials. Development must remain reproducible and must not commit secrets, but mocks cannot prove a live cloud integration.

### Decision

Provide a deterministic fake model provider and feature flags for local development and automated tests. Keep Firebase configuration and production secrets outside versioned files. Before claiming cloud integration is complete, obtain explicit authorization and run a real end-to-end Firebase AI Logic test with App Check.

If credentials are unavailable at final verification, classify the live cloud path as unverified in the Completion Report.

### Consequences

- Core development and tests are not blocked by account setup.
- Mock results cannot be mislabeled as live results.
- A later external-configuration checkpoint is required for full cloud proof.

### Rejected alternatives

- Committing development credentials was rejected.
- Delaying all implementation until Firebase setup was rejected.
- Treating mock output as cloud proof was rejected.

## ADR-010 — Make real-device Nano verification progressive

**Status:** Accepted

### Context

Gemini Nano requires an officially supported device, locked bootloader, runtime model readiness, and foreground execution. Android Emulator cannot prove this path. Requiring a particular phone would block the rest of the MVP.

### Decision

Make AVD API 26 and API 36, deterministic model fakes, cloud fallback, and unsupported-device fallback mandatory. Add Gemini Nano real-device proof when a supported Pixel 9/10-class device or another officially supported device is available. Record the base model identity and environment when testing.

An unavailable device produces an explicit unverified result, not a failure and not a false success.

### Consequences

- Hardware availability does not block the safe core MVP.
- Portfolio claims remain honest and reproducible.
- On-device latency and quality gates remain conditional until tested on real hardware.

### Rejected alternatives

- Requiring supported hardware before any development was rejected.
- Claiming emulator or mock output as Nano verification was rejected.
- Removing the on-device adapter entirely was rejected because hybrid execution is a core learning objective.

## ADR-011 — Prove safety and grounding with fixed evaluation gates

**Status:** Accepted

### Context

A persuasive demo is not sufficient evidence for an agent that handles context, manuals, and external tool calls. The project needs fixed success and failure datasets before implementation is declared complete.

### Decision

Require:

- At least 40 equipment images with at least 90% correct identification or safe abstention.
- At least 25 grounded manual questions with citations on every substantive answer and at least 90% correctness.
- At least 25 context fixtures with 100% expected provenance, confidence, TTL, and conflict behavior.
- At least 20 adversarial privacy and tool cases with zero unauthorized side effects and zero raw PII disclosure.
- Unit and instrumentation tests on AVD API 26 and API 36.
- Saved logs, outputs, screenshots, and a golden-flow recording where feasible.

### Consequences

- Implementation tickets must include test data and proof artifacts, not only feature code.
- Some qualitative model results require a labeled rubric.
- Failures remain visible in the final report.

### Rejected alternatives

- Demo-only acceptance was rejected.
- Self-reported model quality without fixed cases was rejected.
- Hiding unverified hardware or cloud paths was rejected.
