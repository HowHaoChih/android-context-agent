# LabContext Agent MVP Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (\`- [ ]\`) syntax for tracking.

**Goal:** Build a privacy-first Android assistant that completes the evidence-grounded RIGOL DS1054Z inspection workflow without autonomous equipment control.

**Architecture:** A single Android app module is divided into small responsibility packages behind project-owned interfaces. Deterministic local policy and context fusion surround replaceable manual, ML Kit, Firebase, and ADK adapters; Compose presents a user-initiated workflow and Room stores only approved local data.

**Tech Stack:** Kotlin, Jetpack Compose, JDK 17, AGP 9.3.1, Gradle 9.6.1, Android API 26 minimum, compile API 36.1, target API 36, ADK 0.7.0, Firebase BoM 34.16.0, ML Kit Prompt 1.0.0-beta3 through ADK, CameraX 1.6.1, WorkManager 2.11.2, Room 2.8.4, ML Kit Text Recognition 16.0.1, JUnit 4, AndroidX Test, Espresso, and Compose UI Test.

## Global Constraints

- Work only inside 09-Android-Context-Agent.
- Treat 00-Shared-Architecture as read-only.
- Use the internal namespace dev.labcontext.agent.
- Use Gradle Wrapper 9.6.1 and Android Gradle Plugin 9.3.1.
- Use JDK 17, minSdk 26, compileSdk 36.1, and targetSdk 36.
- Pin ADK core, processor, and Firebase Android artifacts to 0.7.0.
- Pin the ADK ML Kit Android artifact to 0.7.0-beta.
- Do not expose ADK, Firebase, or ML Kit types outside adapter packages.
- Do not commit google-services.json, service-account files, Gemini secrets, raw PII, imported manuals, captured media, or private proof artifacts.
- Request camera, microphone, location, notification, and calendar capabilities only when their feature is invoked.
- Do not add background location, notification-listener, body-sensor, arbitrary browser, messaging, purchase, settings, or equipment-control capabilities.
- Use Traditional Chinese user-visible copy and retain original English equipment labels.
- Use test-first development and one commit per ticket.
- Run the ticket test before and after implementation.
- Mark absent Firebase credentials or Nano hardware as UNVERIFIED.

---

## Planned file structure

    settings.gradle.kts
    build.gradle.kts
    gradle.properties
    gradle/libs.versions.toml
    gradle/wrapper/gradle-wrapper.properties
    gradlew
    gradlew.bat
    app/build.gradle.kts
    app/src/main/AndroidManifest.xml
    app/src/main/java/dev/labcontext/agent/MainActivity.kt
    app/src/main/java/dev/labcontext/agent/LabContextApplication.kt
    app/src/main/java/dev/labcontext/agent/core/session/
    app/src/main/java/dev/labcontext/agent/core/permissions/
    app/src/main/java/dev/labcontext/agent/core/context/
    app/src/main/java/dev/labcontext/agent/core/manual/
    app/src/main/java/dev/labcontext/agent/core/manualindex/
    app/src/main/java/dev/labcontext/agent/core/evidence/
    app/src/main/java/dev/labcontext/agent/core/checklist/
    app/src/main/java/dev/labcontext/agent/core/identification/
    app/src/main/java/dev/labcontext/agent/core/routing/
    app/src/main/java/dev/labcontext/agent/core/agent/
    app/src/main/java/dev/labcontext/agent/core/policy/
    app/src/main/java/dev/labcontext/agent/core/audit/
    app/src/main/java/dev/labcontext/agent/core/records/
    app/src/main/java/dev/labcontext/agent/adapters/mlkit/
    app/src/main/java/dev/labcontext/agent/adapters/firebase/
    app/src/main/java/dev/labcontext/agent/platform/camera/
    app/src/main/java/dev/labcontext/agent/platform/calendar/
    app/src/main/java/dev/labcontext/agent/platform/speech/
    app/src/main/java/dev/labcontext/agent/feature/inspection/
    app/src/test/java/dev/labcontext/agent/
    app/src/androidTest/java/dev/labcontext/agent/
    app/src/test/resources/evaluation/
    docs/environment-verification.md
    prove-it/

## Stable project interfaces

The following names are fixed for downstream tickets:

    @JvmInline
    value class SessionId(val value: String)

    enum class SessionStatus {
        CREATED,
        CAPTURING,
        IDENTIFYING,
        EQUIPMENT_CONFIRMED,
        MANUAL_REQUIRED,
        READY_FOR_QUESTION,
        CHECKLIST_READY,
        RECORD_SAVED,
        COMPLETED,
        CANCELLED
    }

    enum class ContextKind {
        USER_INTENT,
        EQUIPMENT_MODEL,
        MANUAL_IDENTITY,
        CAMERA_OBSERVATION,
        NETWORK_STATE,
        BATTERY_STATE,
        SEMANTIC_LOCATION,
        CALENDAR_CONTEXT,
        MOTION_STATE
    }

    enum class Sensitivity {
        PUBLIC,
        LOW,
        SENSITIVE,
        RESTRICTED
    }

    data class ContextFact(
        val kind: ContextKind,
        val value: String,
        val source: String,
        val capturedAt: Instant,
        val expiresAt: Instant,
        val confidence: Double,
        val sensitivity: Sensitivity
    )

    data class ContextSnapshot(
        val facts: List<ContextFact>,
        val conflicts: List<ContextConflict>,
        val createdAt: Instant
    )

    data class Citation(
        val documentId: String,
        val page: Int?,
        val section: String?,
        val quote: String
    )

    interface ModelProvider {
        val providerId: String
        suspend fun availability(): ProviderAvailability
        suspend fun execute(task: ModelTask): ModelOutcome
    }

    interface PolicyEngine {
        fun evaluate(request: PolicyRequest): PolicyDecision
    }

    interface ToolRegistry {
        fun describe(): List<ToolDescriptor>
        suspend fun execute(call: AuthorizedToolCall): ToolResult
    }

## Task 0: ACA-000 Android toolchain gate

**Files:**
- Create: docs/environment-verification.md

**Interfaces:**
- Consumes: Windows host and approved installer access.
- Produces: JAVA_HOME, ANDROID_HOME, sdkmanager, adb, emulator, LabContext_API_26, and LabContext_API_36 availability for every later ticket.

- [ ] **Step 1: Record the current failing preflight**

Run:

    java -version
    adb version
    sdkmanager --list
    emulator -list-avds

Expected before setup: at least one command is unavailable; record the exact failure without changing project source.

- [ ] **Step 2: Obtain installation approval and install the minimum toolchain**

Install a JDK 17 distribution and Android command-line tools. Use sdkmanager to install:

    sdkmanager "platform-tools" "emulator" "platforms;android-26" "platforms;android-36" "platforms;android-36.1" "system-images;android-26;google_apis;x86_64" "system-images;android-36;google_apis;x86_64"

Create the AVDs:

    avdmanager create avd -n LabContext_API_26 -k "system-images;android-26;google_apis;x86_64" --force
    avdmanager create avd -n LabContext_API_36 -k "system-images;android-36;google_apis;x86_64" --force

- [ ] **Step 3: Verify the environment**

Run:

    java -version
    adb version
    sdkmanager --list
    emulator -list-avds

Expected: JDK 17 is reported and both AVD names are listed.

- [ ] **Step 4: Write the verification record**

docs/environment-verification.md must contain:

    # Environment Verification

    - JDK major: 17
    - Android platform packages: 26, 36, 36.1
    - AVDs: LabContext_API_26, LabContext_API_36
    - Secrets recorded: no
    - Verification result: PASS

- [ ] **Step 5: Commit**

    git add docs/environment-verification.md
    git commit -m "chore: verify Android toolchain"

**Definition of Done:** All four preflight commands succeed, both AVDs exist, and the verification file contains no username, token, API key, or private filesystem path.

## Task 1: ACA-001 Bootable Compose shell

**Files:**
- Create: settings.gradle.kts
- Create: build.gradle.kts
- Create: gradle.properties
- Create: gradle/libs.versions.toml
- Create: gradle/wrapper/gradle-wrapper.properties
- Create: gradlew
- Create: gradlew.bat
- Create: app/build.gradle.kts
- Create: app/src/main/AndroidManifest.xml
- Create: app/src/main/java/dev/labcontext/agent/MainActivity.kt
- Create: app/src/main/java/dev/labcontext/agent/ui/LabContextApp.kt
- Create: app/src/main/res/values/strings.xml
- Create: app/src/main/res/values/themes.xml
- Create: app/src/androidTest/java/dev/labcontext/agent/AppLaunchTest.kt

**Interfaces:**
- Consumes: ACA-000 toolchain.
- Produces: app module, LabContextApp composable, and connected Android test entrypoint.

- [ ] **Step 1: Write the failing launch test**

    @RunWith(AndroidJUnit4::class)
    class AppLaunchTest {
        @get:Rule
        val composeRule = createAndroidComposeRule<MainActivity>()

        @Test
        fun showsTraditionalChineseProductName() {
            composeRule.onNodeWithText("實驗室情境助理").assertIsDisplayed()
        }
    }

- [ ] **Step 2: Create the pinned build**

Use these version anchors in gradle/libs.versions.toml:

    [versions]
    agp = "9.3.1"
    compose-compiler = "2.2.10"
    activity-compose = "1.9.3"
    compose-bom = "2024.09.03"
    core = "1.16.0"
    junit = "4.13.2"
    androidx-test-ext = "1.1.5"
    espresso = "3.7.0"

Use this Android baseline in app/build.gradle.kts:

    android {
        namespace = "dev.labcontext.agent"
        compileSdk { version = release(36) { minorApiLevel = 1 } }

        defaultConfig {
            applicationId = "dev.labcontext.agent"
            minSdk = 26
            targetSdk = 36
            versionCode = 1
            versionName = "0.1.0"
            testInstrumentationRunner = "androidx.test.runner.AndroidJUnitRunner"
        }

        buildFeatures {
            compose = true
        }
    }

Pin Gradle in gradle-wrapper.properties:

    distributionUrl=https\://services.gradle.org/distributions/gradle-9.6.1-bin.zip

- [ ] **Step 3: Run the test and confirm failure**

    .\gradlew.bat :app:connectedDebugAndroidTest

Expected: FAIL because MainActivity or the Traditional Chinese node does not exist.

- [ ] **Step 4: Implement the minimal shell**

    @Composable
    fun LabContextApp() {
        MaterialTheme {
            Surface {
                Text(text = stringResource(R.string.app_name))
            }
        }
    }

strings.xml must define app_name as 實驗室情境助理.

- [ ] **Step 5: Verify**

    .\gradlew.bat :app:assembleDebug :app:testDebugUnitTest
    .\gradlew.bat :app:connectedDebugAndroidTest

Expected: all tasks pass on API 26 and API 36.

- [ ] **Step 6: Commit**

    git add settings.gradle.kts build.gradle.kts gradle.properties gradle app
    git commit -m "feat: add bootable Android app shell"

**Definition of Done:** The app launches on both AVD baselines, renders the Traditional Chinese product name, and contains no Firebase or ADK dependency yet.

## Task 2: ACA-002 Inspection session state

**Files:**
- Create: app/src/main/java/dev/labcontext/agent/core/session/InspectionSession.kt
- Create: app/src/main/java/dev/labcontext/agent/core/session/SessionReducer.kt
- Create: app/src/test/java/dev/labcontext/agent/core/session/SessionReducerTest.kt

**Interfaces:**
- Consumes: java.time.Instant.
- Produces: SessionId, SessionStatus, InspectionSession, SessionEvent, SessionTransition, and SessionReducer.reduce.

- [ ] **Step 1: Write the failing transition test**

    @Test
    fun createdSessionStartsCapture() {
        val session = InspectionSession.create(
            id = SessionId("session-1"),
            now = Instant.parse("2026-08-11T00:00:00Z")
        )

        val result = SessionReducer.reduce(session, SessionEvent.StartCapture)

        assertEquals(SessionStatus.CAPTURING, result.session.status)
        assertNull(result.error)
    }

- [ ] **Step 2: Run and confirm failure**

    .\gradlew.bat :app:testDebugUnitTest --tests "dev.labcontext.agent.core.session.SessionReducerTest"

Expected: FAIL because the session types do not exist.

- [ ] **Step 3: Implement the reducer**

    sealed interface SessionEvent {
        data object StartCapture : SessionEvent
        data object StartIdentification : SessionEvent
        data class ConfirmEquipment(val manufacturer: String, val model: String) : SessionEvent
        data object RequireManual : SessionEvent
        data object ReadyForQuestion : SessionEvent
        data object PublishChecklist : SessionEvent
        data object SaveRecord : SessionEvent
        data object Complete : SessionEvent
        data object Cancel : SessionEvent
    }

    data class SessionTransition(
        val session: InspectionSession,
        val error: SessionTransitionError?
    )

    object SessionReducer {
        fun reduce(session: InspectionSession, event: SessionEvent): SessionTransition
    }

- [ ] **Step 4: Add complete transition coverage**

Test all accepted transitions, all invalid transitions, cancellation from active states, and serialization round-trip.

- [ ] **Step 5: Verify**

    .\gradlew.bat :app:testDebugUnitTest --tests "dev.labcontext.agent.core.session.*"

Expected: PASS.

- [ ] **Step 6: Commit**

    git add app/src/main/java/dev/labcontext/agent/core/session app/src/test/java/dev/labcontext/agent/core/session
    git commit -m "feat: add inspection session state machine"

**Definition of Done:** Session behavior is deterministic, framework-free, fully tested, and contains no storage or UI logic.

## Task 3: ACA-003 Progressive permission coordinator

**Files:**
- Create: app/src/main/java/dev/labcontext/agent/core/permissions/PermissionCapability.kt
- Create: app/src/main/java/dev/labcontext/agent/core/permissions/PermissionCoordinator.kt
- Create: app/src/main/java/dev/labcontext/agent/core/permissions/AndroidPermissionCoordinator.kt
- Create: app/src/test/java/dev/labcontext/agent/core/permissions/PermissionCoordinatorTest.kt
- Create: app/src/androidTest/java/dev/labcontext/agent/core/permissions/PermissionDeniedUiTest.kt

**Interfaces:**
- Consumes: Android permission results only in AndroidPermissionCoordinator.
- Produces: PermissionCapability, CapabilityState, PermissionAction, and PermissionCoordinator.state.

- [ ] **Step 1: Write the failing domain test**

    @Test
    fun permanentCameraDenialOffersSettingsWithoutRequestLoop() {
        val state = coordinator.reduce(
            PermissionCapability.CAMERA,
            PermissionAction.Denied(canAskAgain = false)
        )

        assertEquals(CapabilityState.DeniedPermanently, state)
    }

- [ ] **Step 2: Run and confirm failure**

    .\gradlew.bat :app:testDebugUnitTest --tests "dev.labcontext.agent.core.permissions.PermissionCoordinatorTest"

Expected: FAIL because the coordinator does not exist.

- [ ] **Step 3: Implement capability states**

    enum class PermissionCapability {
        CAMERA,
        MICROPHONE,
        LOCATION_WHILE_IN_USE,
        POST_NOTIFICATIONS,
        CALENDAR_READ,
        CALENDAR_WRITE
    }

    sealed interface CapabilityState {
        data object NotRequested : CapabilityState
        data object Granted : CapabilityState
        data object Denied : CapabilityState
        data object DeniedPermanently : CapabilityState
        data object Unavailable : CapabilityState
    }

- [ ] **Step 4: Add the denial UI instrumentation test**

The test launches the inspection screen with a fake denied state and asserts the Traditional Chinese explanation and settings action are visible.

- [ ] **Step 5: Verify**

    .\gradlew.bat :app:testDebugUnitTest --tests "dev.labcontext.agent.core.permissions.*"
    .\gradlew.bat :app:connectedDebugAndroidTest

Expected: PASS.

- [ ] **Step 6: Commit**

    git add app/src/main/java/dev/labcontext/agent/core/permissions app/src/test/java/dev/labcontext/agent/core/permissions app/src/androidTest/java/dev/labcontext/agent/core/permissions
    git commit -m "feat: add progressive permission coordinator"

**Definition of Done:** No startup permission request exists, prohibited capabilities are absent, and denial is recoverable.

## Task 4: ACA-004 Deterministic context fusion

**Files:**
- Create: app/src/main/java/dev/labcontext/agent/core/context/ContextModels.kt
- Create: app/src/main/java/dev/labcontext/agent/core/context/ContextFusionEngine.kt
- Create: app/src/main/java/dev/labcontext/agent/core/context/DefaultContextFusionEngine.kt
- Create: app/src/test/java/dev/labcontext/agent/core/context/DefaultContextFusionEngineTest.kt
- Create: app/src/test/resources/evaluation/context-cases.json

**Interfaces:**
- Consumes: List of ContextFact and java.time.Clock.
- Produces: ContextSnapshot and ContextConflict.

- [ ] **Step 1: Write the failing expiry and conflict tests**

    @Test
    fun expiredFactsAreExcluded() {
        val snapshot = engine.fuse(listOf(expiredCameraFact), now)
        assertTrue(snapshot.facts.isEmpty())
    }

    @Test
    fun equalAuthorityContradictionCreatesConflict() {
        val snapshot = engine.fuse(listOf(cameraRigol, cameraTektronix), now)
        assertEquals(1, snapshot.conflicts.size)
    }

- [ ] **Step 2: Run and confirm failure**

    .\gradlew.bat :app:testDebugUnitTest --tests "dev.labcontext.agent.core.context.DefaultContextFusionEngineTest"

Expected: FAIL because the context models do not exist.

- [ ] **Step 3: Implement authority and validation**

    interface ContextFusionEngine {
        fun fuse(facts: List<ContextFact>, now: Instant): ContextSnapshot
    }

    data class ContextConflict(
        val kind: ContextKind,
        val competingFacts: List<ContextFact>,
        val reason: String
    )

Confidence must be within 0.0 through 1.0 and expiresAt must be after capturedAt.

- [ ] **Step 4: Add 25 deterministic fixtures**

The JSON manifest must include stale, contradictory, low-confidence, permission-denied, offline, low-battery, document-hash-change, and semantic-location cases.

- [ ] **Step 5: Verify**

    .\gradlew.bat :app:testDebugUnitTest --tests "dev.labcontext.agent.core.context.*"

Expected: 25 fixture cases and unit tests pass.

- [ ] **Step 6: Commit**

    git add app/src/main/java/dev/labcontext/agent/core/context app/src/test/java/dev/labcontext/agent/core/context app/src/test/resources/evaluation/context-cases.json
    git commit -m "feat: add deterministic context fusion"

**Definition of Done:** Context is reproducible, typed, time-bounded, sensitivity-aware, and never silently resolves equal-authority conflicts.

## Task 5: ACA-005 Manual registry and import

**Files:**
- Create: app/src/main/java/dev/labcontext/agent/core/manual/ManualDocument.kt
- Create: app/src/main/java/dev/labcontext/agent/core/manual/ManualRepository.kt
- Create: app/src/main/java/dev/labcontext/agent/core/manual/InMemoryManualRepository.kt
- Create: app/src/main/java/dev/labcontext/agent/core/manual/ManualImportService.kt
- Create: app/src/test/java/dev/labcontext/agent/core/manual/ManualImportServiceTest.kt

**Interfaces:**
- Consumes: ManualSource with displayName, MIME type, persisted URI string, and InputStream supplier.
- Produces: ManualDocument and ManualImportResult.

- [ ] **Step 1: Write the failing hash-idempotency test**

    @Test
    fun sameHashReturnsExistingDocument() = runTest {
        val first = service.import(source("scope.pdf", bytes))
        val second = service.import(source("scope-copy.pdf", bytes))

        assertEquals(first.document.id, second.document.id)
        assertFalse(second.created)
    }

- [ ] **Step 2: Run and confirm failure**

    .\gradlew.bat :app:testDebugUnitTest --tests "dev.labcontext.agent.core.manual.ManualImportServiceTest"

Expected: FAIL because ManualImportService does not exist.

- [ ] **Step 3: Implement import rules**

    data class ManualDocument(
        val id: String,
        val displayName: String,
        val manufacturer: String,
        val model: String,
        val uri: String,
        val sha256: String,
        val importedAt: Instant,
        val indexVersion: Int
    )

    interface ManualRepository {
        suspend fun findByHash(sha256: String): ManualDocument?
        suspend fun save(document: ManualDocument)
        suspend fun delete(documentId: String)
        suspend fun invalidateIndex(documentId: String)
    }

- [ ] **Step 4: Add rejection and deletion tests**

Cover non-PDF MIME, empty input, changed hash, delete, and index invalidation.

- [ ] **Step 5: Verify**

    .\gradlew.bat :app:testDebugUnitTest --tests "dev.labcontext.agent.core.manual.*"

Expected: PASS.

- [ ] **Step 6: Commit**

    git add app/src/main/java/dev/labcontext/agent/core/manual app/src/test/java/dev/labcontext/agent/core/manual
    git commit -m "feat: add trusted manual registry"

**Definition of Done:** Imports are hash-addressed, idempotent, deletable, and independent of a manufacturer PDF fixture.

## Task 6: ACA-006 PDF rendering and OCR index

**Files:**
- Modify: gradle/libs.versions.toml
- Modify: app/build.gradle.kts
- Create: app/src/main/java/dev/labcontext/agent/core/manualindex/PdfPageRenderer.kt
- Create: app/src/main/java/dev/labcontext/agent/core/manualindex/OcrEngine.kt
- Create: app/src/main/java/dev/labcontext/agent/core/manualindex/ManualIndexer.kt
- Create: app/src/main/java/dev/labcontext/agent/core/manualindex/ManualIndexWorker.kt
- Create: app/src/test/java/dev/labcontext/agent/core/manualindex/ManualIndexerTest.kt
- Create: app/src/test/resources/manual/synthetic-oscilloscope-manual.pdf

**Interfaces:**
- Consumes: ManualDocument.
- Produces: ManualPassage rows and ManualIndexState.

- [ ] **Step 1: Write the failing synthetic PDF test**

    @Test
    fun indexesProbeRatioPageWithLocator() = runTest {
        val result = indexer.index(syntheticManual)
        val passage = result.passages.single { it.text.contains("Probe 10X") }

        assertEquals(3, passage.page)
        assertEquals("CH1 Probe Ratio", passage.section)
    }

- [ ] **Step 2: Add exact dependencies**

    camera = "1.6.1"
    work = "2.11.2"
    mlkit-text = "16.0.1"

Use androidx.work:work-runtime, androidx.work:work-testing, and com.google.mlkit:text-recognition.

- [ ] **Step 3: Run and confirm failure**

    .\gradlew.bat :app:testDebugUnitTest --tests "dev.labcontext.agent.core.manualindex.ManualIndexerTest"

Expected: FAIL because renderer, OCR engine, and indexer do not exist.

- [ ] **Step 4: Implement indexed passages**

    data class ManualPassage(
        val documentId: String,
        val page: Int,
        val section: String?,
        val text: String,
        val normalizedTerms: Set<String>
    )

    sealed interface ManualIndexState {
        data class Running(val completedPages: Int, val totalPages: Int) : ManualIndexState
        data class Complete(val passageCount: Int) : ManualIndexState
        data class Failed(val category: String) : ManualIndexState
        data object Cancelled : ManualIndexState
    }

- [ ] **Step 5: Test cleanup and cancellation**

Use fakes to assert that rendered bitmaps, file descriptors, PdfRenderer, and temporary index rows close or delete on every exit path.

- [ ] **Step 6: Verify and commit**

    .\gradlew.bat :app:testDebugUnitTest --tests "dev.labcontext.agent.core.manualindex.*"
    git add gradle/libs.versions.toml app/build.gradle.kts app/src/main/java/dev/labcontext/agent/core/manualindex app/src/test/java/dev/labcontext/agent/core/manualindex app/src/test/resources/manual
    git commit -m "feat: index imported manuals locally"

**Definition of Done:** The synthetic PDF yields searchable page-located passages and cancelled jobs cannot publish partial indexes.

## Task 7: ACA-007 Evidence retrieval and citations

**Files:**
- Create: app/src/main/java/dev/labcontext/agent/core/evidence/EvidenceRetriever.kt
- Create: app/src/main/java/dev/labcontext/agent/core/evidence/LocalEvidenceRetriever.kt
- Create: app/src/main/java/dev/labcontext/agent/core/evidence/CitationValidator.kt
- Create: app/src/test/java/dev/labcontext/agent/core/evidence/EvidenceRetrieverTest.kt

**Interfaces:**
- Consumes: selected document ID, question, and ManualPassage list.
- Produces: EvidenceBundle and CitationValidation.

- [ ] **Step 1: Write the failing document-isolation test**

    @Test
    fun retrievalNeverCrossesSelectedManual() {
        val bundle = retriever.retrieve("rigol-hash", "CH1 probe ratio", allPassages)
        assertTrue(bundle.passages.all { it.documentId == "rigol-hash" })
    }

- [ ] **Step 2: Run and confirm failure**

    .\gradlew.bat :app:testDebugUnitTest --tests "dev.labcontext.agent.core.evidence.EvidenceRetrieverTest"

Expected: FAIL because the retriever does not exist.

- [ ] **Step 3: Implement evidence contracts**

    data class ScoredPassage(
        val passage: ManualPassage,
        val score: Double
    )

    data class EvidenceBundle(
        val question: String,
        val documentId: String,
        val passages: List<ScoredPassage>
    )

    interface EvidenceRetriever {
        fun retrieve(documentId: String, question: String, passages: List<ManualPassage>): EvidenceBundle
    }

- [ ] **Step 4: Implement citation validation**

Reject citations whose document ID, page, section, or quote cannot be matched to the selected evidence bundle.

- [ ] **Step 5: Verify and commit**

    .\gradlew.bat :app:testDebugUnitTest --tests "dev.labcontext.agent.core.evidence.*"
    git add app/src/main/java/dev/labcontext/agent/core/evidence app/src/test/java/dev/labcontext/agent/core/evidence
    git commit -m "feat: retrieve and validate manual evidence"

**Definition of Done:** Retrieval is document-scoped and every accepted citation is traceable to an indexed passage.

## Task 8: ACA-008 Grounded checklist composer

**Files:**
- Create: app/src/main/java/dev/labcontext/agent/core/checklist/ChecklistModels.kt
- Create: app/src/main/java/dev/labcontext/agent/core/checklist/ChecklistComposer.kt
- Create: app/src/main/java/dev/labcontext/agent/core/checklist/DeterministicChecklistComposer.kt
- Create: app/src/test/java/dev/labcontext/agent/core/checklist/ChecklistComposerTest.kt

**Interfaces:**
- Consumes: EvidenceBundle.
- Produces: ChecklistOutcome.

- [ ] **Step 1: Write the failing grounding test**

    @Test
    fun everyChecklistItemHasCitation() {
        val outcome = composer.compose(probeRatioEvidence)
        val checklist = assertIs<ChecklistOutcome.Success>(outcome).checklist

        assertTrue(checklist.items.all { it.citations.isNotEmpty() })
    }

- [ ] **Step 2: Run and confirm failure**

    .\gradlew.bat :app:testDebugUnitTest --tests "dev.labcontext.agent.core.checklist.ChecklistComposerTest"

Expected: FAIL because checklist types do not exist.

- [ ] **Step 3: Implement output types**

    data class ChecklistItem(
        val id: String,
        val textZhTw: String,
        val controlLabelsEn: List<String>,
        val citations: List<Citation>,
        val safetyCritical: Boolean
    )

    data class Checklist(
        val titleZhTw: String,
        val items: List<ChecklistItem>,
        val disclaimerZhTw: String
    )

    sealed interface ChecklistOutcome {
        data class Success(val checklist: Checklist) : ChecklistOutcome
        data class Abstained(val reasonZhTw: String) : ChecklistOutcome
    }

- [ ] **Step 4: Add the fixed CH1 case**

The synthetic evidence must produce steps for checking Probe 10X and setting CH1 vertical scale, with English labels Probe and CH1 preserved.

- [ ] **Step 5: Verify and commit**

    .\gradlew.bat :app:testDebugUnitTest --tests "dev.labcontext.agent.core.checklist.*"
    git add app/src/main/java/dev/labcontext/agent/core/checklist app/src/test/java/dev/labcontext/agent/core/checklist
    git commit -m "feat: compose cited equipment checklists"

**Definition of Done:** Checklist output is Traditional Chinese, cited, safety-marked, deterministic for the fixture, and abstains without evidence.

## Task 9: ACA-009 Still-photo capture

**Files:**
- Modify: gradle/libs.versions.toml
- Modify: app/build.gradle.kts
- Modify: app/src/main/AndroidManifest.xml
- Create: app/src/main/java/dev/labcontext/agent/platform/camera/EquipmentCaptureGateway.kt
- Create: app/src/main/java/dev/labcontext/agent/platform/camera/CameraXEquipmentCaptureGateway.kt
- Create: app/src/main/java/dev/labcontext/agent/platform/camera/ImageSanitizer.kt
- Create: app/src/test/java/dev/labcontext/agent/platform/camera/ImageSanitizerTest.kt
- Create: app/src/androidTest/java/dev/labcontext/agent/platform/camera/CameraPermissionGateTest.kt

**Interfaces:**
- Consumes: active SessionId and granted camera capability.
- Produces: SanitizedImage with private URI, SHA-256, width, height, and createdAt.

- [ ] **Step 1: Write the failing permission gate test**

    @Test
    fun captureIsRejectedWithoutGrantedCapability() = runTest {
        val result = gateway.capture(SessionId("session-1"), CapabilityState.Denied)
        assertIs<CaptureResult.PermissionDenied>(result)
    }

- [ ] **Step 2: Add CameraX 1.6.1 dependencies**

Add camera-core, camera-camera2, camera-lifecycle, and camera-view using one camera version alias.

- [ ] **Step 3: Run and confirm failure**

    .\gradlew.bat :app:testDebugUnitTest --tests "dev.labcontext.agent.platform.camera.*"

Expected: FAIL because the gateway does not exist.

- [ ] **Step 4: Implement capture and sanitization**

    data class SanitizedImage(
        val privateUri: String,
        val sha256: String,
        val width: Int,
        val height: Int,
        val createdAt: Instant
    )

    interface EquipmentCaptureGateway {
        suspend fun capture(sessionId: SessionId, capability: CapabilityState): CaptureResult
        suspend fun delete(image: SanitizedImage)
    }

Write a derivative image without copied EXIF metadata and keep it in app-private temporary storage.

- [ ] **Step 5: Verify lifecycle cleanup**

Test success, cancel, error, session deletion, and Activity stop. Assert no background camera service is declared.

- [ ] **Step 6: Verify and commit**

    .\gradlew.bat :app:testDebugUnitTest --tests "dev.labcontext.agent.platform.camera.*"
    .\gradlew.bat :app:connectedDebugAndroidTest
    git add gradle/libs.versions.toml app/build.gradle.kts app/src/main/AndroidManifest.xml app/src/main/java/dev/labcontext/agent/platform/camera app/src/test/java/dev/labcontext/agent/platform/camera app/src/androidTest/java/dev/labcontext/agent/platform/camera
    git commit -m "feat: capture minimized equipment photos"

**Definition of Done:** One foreground still photo can be captured, sanitized, and deleted with no continuous-video path.

## Task 10: ACA-010 Equipment identification

**Files:**
- Create: app/src/main/java/dev/labcontext/agent/core/identification/EquipmentIdentity.kt
- Create: app/src/main/java/dev/labcontext/agent/core/identification/EquipmentIdentifier.kt
- Create: app/src/main/java/dev/labcontext/agent/core/identification/DefaultEquipmentIdentifier.kt
- Create: app/src/test/java/dev/labcontext/agent/core/identification/EquipmentIdentifierTest.kt

**Interfaces:**
- Consumes: SanitizedImage metadata, OCR cues, and IdentificationModel.
- Produces: EquipmentIdentificationOutcome.

- [ ] **Step 1: Write the failing abstention test**

    @Test
    fun unsupportedModelAbstains() = runTest {
        val outcome = identifier.identify(inputWithTektronixCue)
        assertIs<EquipmentIdentificationOutcome.Abstained>(outcome)
    }

- [ ] **Step 2: Run and confirm failure**

    .\gradlew.bat :app:testDebugUnitTest --tests "dev.labcontext.agent.core.identification.EquipmentIdentifierTest"

Expected: FAIL because identification types do not exist.

- [ ] **Step 3: Implement the outcome**

    data class EquipmentIdentity(
        val manufacturer: String,
        val model: String,
        val confidence: Double,
        val evidenceCues: List<String>
    )

    sealed interface EquipmentIdentificationOutcome {
        data class Identified(val identity: EquipmentIdentity) : EquipmentIdentificationOutcome
        data class NeedsConfirmation(val candidates: List<EquipmentIdentity>) : EquipmentIdentificationOutcome
        data class Abstained(val reasonZhTw: String) : EquipmentIdentificationOutcome
    }

Accept RIGOL DS1054Z only when normalized cues and provider output agree above the fixed confidence threshold.

- [ ] **Step 4: Add complete branch tests**

Cover RIGOL success, low confidence, unsupported manufacturer, OCR-model conflict, and empty cues.

- [ ] **Step 5: Verify and commit**

    .\gradlew.bat :app:testDebugUnitTest --tests "dev.labcontext.agent.core.identification.*"
    git add app/src/main/java/dev/labcontext/agent/core/identification app/src/test/java/dev/labcontext/agent/core/identification
    git commit -m "feat: identify supported equipment safely"

**Definition of Done:** The identifier returns a supported identity, a confirmation request, or an abstention and never retains raw image data.

## Task 11: ACA-011 Explicit model router

**Files:**
- Create: app/src/main/java/dev/labcontext/agent/core/routing/ModelContracts.kt
- Create: app/src/main/java/dev/labcontext/agent/core/routing/ModelRouter.kt
- Create: app/src/main/java/dev/labcontext/agent/core/routing/DefaultModelRouter.kt
- Create: app/src/main/java/dev/labcontext/agent/core/routing/FakeModelProvider.kt
- Create: app/src/test/java/dev/labcontext/agent/core/routing/ModelRouterTest.kt

**Interfaces:**
- Consumes: ModelTask, RuntimeSignals, ContextSnapshot, and registered ModelProvider values.
- Produces: RouteDecision and ModelOutcome.

- [ ] **Step 1: Write the failing offline test**

    @Test
    fun offlineExternalTaskIsRejected() = runTest {
        val decision = router.decide(calendarCommitTask, offlineSignals, snapshot)
        assertEquals(RouteTarget.REJECTED, decision.target)
        assertEquals("offline_external_action", decision.reason)
    }

- [ ] **Step 2: Run and confirm failure**

    .\gradlew.bat :app:testDebugUnitTest --tests "dev.labcontext.agent.core.routing.ModelRouterTest"

Expected: FAIL because routing contracts do not exist.

- [ ] **Step 3: Implement routing contracts**

    enum class RouteTarget {
        DETERMINISTIC,
        NANO,
        FIREBASE,
        FAKE,
        OFFLINE,
        REJECTED
    }

    data class RuntimeSignals(
        val foreground: Boolean,
        val online: Boolean,
        val lowBattery: Boolean,
        val firebaseConfigured: Boolean,
        val nanoAvailable: Boolean
    )

    data class RouteDecision(
        val target: RouteTarget,
        val reason: String
    )

- [ ] **Step 4: Encode and test the full table**

Cover short text, image reasoning, evidence synthesis, tool planning, offline lookup, missing configuration, unsupported Nano, low battery, background state, and fake test mode.

- [ ] **Step 5: Verify and commit**

    .\gradlew.bat :app:testDebugUnitTest --tests "dev.labcontext.agent.core.routing.*"
    git add app/src/main/java/dev/labcontext/agent/core/routing app/src/test/java/dev/labcontext/agent/core/routing
    git commit -m "feat: add explicit model routing"

**Definition of Done:** Every task has one deterministic route and reason, offline external effects are rejected, and providers cannot execute tools.

## Task 12: ACA-012 Gemini Nano adapter

**Files:**
- Modify: gradle/libs.versions.toml
- Modify: app/build.gradle.kts
- Create: app/src/main/java/dev/labcontext/agent/adapters/mlkit/MlKitNanoProvider.kt
- Create: app/src/main/java/dev/labcontext/agent/adapters/mlkit/NanoAvailabilityMapper.kt
- Create: app/src/test/java/dev/labcontext/agent/adapters/mlkit/NanoAvailabilityMapperTest.kt

**Interfaces:**
- Consumes: supported short-text ModelTask values.
- Produces: ProviderAvailability and ModelOutcome through ModelProvider.

- [ ] **Step 1: Write failing availability mappings**

    @Test
    fun busyMapsToTemporarilyUnavailable() {
        assertEquals(
            ProviderAvailability.TemporarilyUnavailable("busy"),
            mapper.map(FakeNanoStatus.Busy)
        )
    }

- [ ] **Step 2: Add exact dependencies**

    adk = "0.7.0"
    adk-mlkit = "0.7.0-beta"

Add com.google.adk:google-adk-kotlin-mlkit-android and keep ML Kit types inside adapters/mlkit.

- [ ] **Step 3: Run and confirm failure**

    .\gradlew.bat :app:testDebugUnitTest --tests "dev.labcontext.agent.adapters.mlkit.*"

Expected: FAIL because the mapper and provider do not exist.

- [ ] **Step 4: Implement guarded execution**

Call checkStatus before exposing availability. Enforce foreground state, maximum input tokens below 4000, bounded output tokens, and typed mapping for busy, quota, download, unsupported, and security failures.

- [ ] **Step 5: Verify fake behavior**

    .\gradlew.bat :app:testDebugUnitTest --tests "dev.labcontext.agent.adapters.mlkit.*"

Expected: PASS. Real-device result remains UNVERIFIED.

- [ ] **Step 6: Commit**

    git add gradle/libs.versions.toml app/build.gradle.kts app/src/main/java/dev/labcontext/agent/adapters/mlkit app/src/test/java/dev/labcontext/agent/adapters/mlkit
    git commit -m "feat: add guarded Gemini Nano adapter"

**Definition of Done:** Unsupported or unavailable Nano never crashes routing and no emulator result is labeled as real Nano proof.

## Task 13: ACA-013 Firebase cloud adapter

**Files:**
- Modify: gradle/libs.versions.toml
- Modify: app/build.gradle.kts
- Modify: .gitignore
- Create: app/src/main/java/dev/labcontext/agent/adapters/firebase/FirebaseConfiguration.kt
- Create: app/src/main/java/dev/labcontext/agent/adapters/firebase/FirebaseModelProvider.kt
- Create: app/src/test/java/dev/labcontext/agent/adapters/firebase/FirebaseConfigurationTest.kt

**Interfaces:**
- Consumes: cloud-supported ModelTask values and FirebaseConfiguration.
- Produces: ProviderAvailability and ModelOutcome through ModelProvider.

- [ ] **Step 1: Write the failing no-configuration test**

    @Test
    fun missingConfigurationMakesNoRequest() = runTest {
        val provider = FirebaseModelProvider(FirebaseConfiguration.Missing, recordingClient)
        val outcome = provider.execute(validCloudTask)

        assertIs<ModelOutcome.Unavailable>(outcome)
        assertEquals(0, recordingClient.requestCount)
    }

- [ ] **Step 2: Add exact dependencies**

Use:

    firebase-bom = "34.16.0"
    implementation(platform("com.google.firebase:firebase-bom:34.16.0"))
    implementation("com.google.firebase:firebase-ai")
    debugImplementation("com.google.firebase:firebase-appcheck-debug")
    releaseImplementation("com.google.firebase:firebase-appcheck-playintegrity")
    implementation("com.google.adk:google-adk-kotlin-firebase-android:0.7.0")

Add google-services.json, service-account patterns, local.properties, and prove-it/private to .gitignore.

- [ ] **Step 3: Run and confirm failure**

    .\gradlew.bat :app:testDebugUnitTest --tests "dev.labcontext.agent.adapters.firebase.*"

Expected: FAIL because the adapter does not exist.

- [ ] **Step 4: Implement credential-free behavior**

    sealed interface FirebaseConfiguration {
        data object Missing : FirebaseConfiguration
        data class Available(
            val projectId: String,
            val appId: String,
            val apiKey: String
        ) : FirebaseConfiguration
    }

Treat Firebase client identifiers as runtime configuration and never log them. Do not accept Admin SDK or Gemini Developer API secrets.

- [ ] **Step 5: Verify and commit**

    .\gradlew.bat :app:testDebugUnitTest --tests "dev.labcontext.agent.adapters.firebase.*"
    git diff --check
    git add gradle/libs.versions.toml app/build.gradle.kts .gitignore app/src/main/java/dev/labcontext/agent/adapters/firebase app/src/test/java/dev/labcontext/agent/adapters/firebase
    git commit -m "feat: add credential-free Firebase adapter"

**Definition of Done:** The app builds and tests without Firebase configuration, uses App Check by build type, and contains no private key material.

## Task 14: ACA-014 ADK agent engine

**Files:**
- Modify: gradle/libs.versions.toml
- Modify: app/build.gradle.kts
- Create: app/src/main/java/dev/labcontext/agent/core/agent/AgentContracts.kt
- Create: app/src/main/java/dev/labcontext/agent/core/agent/AdkAgentEngine.kt
- Create: app/src/main/java/dev/labcontext/agent/core/agent/GeneratedAgentTools.kt
- Create: app/src/test/java/dev/labcontext/agent/core/agent/AdkAgentEngineTest.kt

**Interfaces:**
- Consumes: ContextSnapshot, EvidenceBundle, ModelRouter, and ToolRegistry descriptions.
- Produces: AgentOutcome with checklist draft and ToolProposal list.

- [ ] **Step 1: Write the failing proposal-not-execution test**

    @Test
    fun agentReturnsCalendarProposalWithoutExecutingIt() = runTest {
        val outcome = engine.run(fixedAgentRequest)
        assertEquals(1, outcome.toolProposals.size)
        assertEquals(0, recordingToolRegistry.executionCount)
    }

- [ ] **Step 2: Add ADK and KSP**

Use:

    implementation("com.google.adk:google-adk-kotlin-core:0.7.0")
    ksp("com.google.adk:google-adk-kotlin-processor:0.7.0")

Apply KSP 2.3.9 and keep generated tool wrappers inside core/agent.

- [ ] **Step 3: Run and confirm failure**

    .\gradlew.bat :app:testDebugUnitTest --tests "dev.labcontext.agent.core.agent.*"

Expected: FAIL because the agent engine does not exist.

- [ ] **Step 4: Implement the boundary**

    data class AgentRequest(
        val sessionId: SessionId,
        val question: String,
        val context: ContextSnapshot,
        val evidence: EvidenceBundle
    )

    data class AgentOutcome(
        val checklistDraft: ChecklistOutcome,
        val toolProposals: List<ToolProposal>,
        val routeDecision: RouteDecision
    )

    interface AgentEngine {
        suspend fun run(request: AgentRequest): AgentOutcome
    }

The ADK runner may create proposals but cannot receive an executable ToolRegistry instance.

- [ ] **Step 5: Verify and commit**

    .\gradlew.bat :app:kspDebugKotlin :app:testDebugUnitTest --tests "dev.labcontext.agent.core.agent.*"
    git add gradle/libs.versions.toml app/build.gradle.kts app/src/main/java/dev/labcontext/agent/core/agent app/src/test/java/dev/labcontext/agent/core/agent
    git commit -m "feat: add bounded ADK agent engine"

**Definition of Done:** ADK output is converted to project-owned types and cannot execute a proposed side effect.

## Task 15: ACA-015 Policy engine

**Files:**
- Create: app/src/main/java/dev/labcontext/agent/core/policy/PolicyModels.kt
- Create: app/src/main/java/dev/labcontext/agent/core/policy/DefaultPolicyEngine.kt
- Create: app/src/test/java/dev/labcontext/agent/core/policy/PolicyEngineTest.kt
- Create: app/src/test/resources/evaluation/policy-adversarial-cases.json

**Interfaces:**
- Consumes: PolicyRequest containing session, evidence, output, tool proposal, capability states, and confirmation token.
- Produces: PolicyDecision.

- [ ] **Step 1: Write the failing injection test**

    @Test
    fun manualInstructionCannotRequestSecretUpload() {
        val decision = engine.evaluate(promptInjectionRequest)
        assertIs<PolicyDecision.Blocked>(decision)
        assertEquals("untrusted_instruction", decision.reason)
    }

- [ ] **Step 2: Run and confirm failure**

    .\gradlew.bat :app:testDebugUnitTest --tests "dev.labcontext.agent.core.policy.PolicyEngineTest"

Expected: FAIL because the policy engine does not exist.

- [ ] **Step 3: Implement policy types**

    enum class ToolLevel {
        READ_ONLY,
        REVERSIBLE_LOCAL_WRITE,
        CONFIRMED_EXTERNAL_WRITE,
        PROHIBITED
    }

    sealed interface PolicyDecision {
        data class Allowed(val authorizationId: String) : PolicyDecision
        data class NeedsConfirmation(val previewId: String) : PolicyDecision
        data class Blocked(val reason: String) : PolicyDecision
    }

- [ ] **Step 4: Add adversarial cases**

Cover secret upload, policy override, uncited procedure, unknown tool, prohibited equipment control, malformed arguments, repeated confirmation, precise-location upload, and manual cross-document citation.

- [ ] **Step 5: Verify and commit**

    .\gradlew.bat :app:testDebugUnitTest --tests "dev.labcontext.agent.core.policy.*"
    git add app/src/main/java/dev/labcontext/agent/core/policy app/src/test/java/dev/labcontext/agent/core/policy app/src/test/resources/evaluation/policy-adversarial-cases.json
    git commit -m "feat: enforce agent safety policy"

**Definition of Done:** Initial adversarial cases pass with no executable unauthorized call and no model output can constitute confirmation.

## Task 16: ACA-016 Redacted audit log

**Files:**
- Create: app/src/main/java/dev/labcontext/agent/core/audit/AuditEvent.kt
- Create: app/src/main/java/dev/labcontext/agent/core/audit/AuditLogger.kt
- Create: app/src/main/java/dev/labcontext/agent/core/audit/RedactingAuditLogger.kt
- Create: app/src/test/java/dev/labcontext/agent/core/audit/RedactingAuditLoggerTest.kt

**Interfaces:**
- Consumes: AuditInput.
- Produces: AuditEvent that is safe for local persistence or explicit export.

- [ ] **Step 1: Write the failing redaction test**

    @Test
    fun restrictedFieldsNeverReachSerializedEvent() {
        val event = logger.log(inputContainingPhotoAudioLocationAndSecret)
        val json = serializer.encode(event)

        assertFalse(json.contains("AIza"))
        assertFalse(json.contains("25.0330"))
        assertFalse(json.contains("rawAudio"))
    }

- [ ] **Step 2: Run and confirm failure**

    .\gradlew.bat :app:testDebugUnitTest --tests "dev.labcontext.agent.core.audit.*"

Expected: FAIL because the audit logger does not exist.

- [ ] **Step 3: Implement the safe schema**

    data class AuditEvent(
        val eventId: String,
        val sessionId: SessionId,
        val correlationId: String,
        val eventType: String,
        val timestamp: Instant,
        val redactionVersion: Int,
        val safeAttributes: Map<String, String>
    )

Use an allowlist of safe attribute names rather than a blocklist.

- [ ] **Step 4: Verify and commit**

    .\gradlew.bat :app:testDebugUnitTest --tests "dev.labcontext.agent.core.audit.*"
    git add app/src/main/java/dev/labcontext/agent/core/audit app/src/test/java/dev/labcontext/agent/core/audit
    git commit -m "feat: add redacted local audit events"

**Definition of Done:** Serialization cannot contain raw media, exact coordinates, manual passages, calendar bodies, Firebase identifiers, or secrets.

## Task 17: ACA-017 Local experiment records

**Files:**
- Modify: gradle/libs.versions.toml
- Modify: app/build.gradle.kts
- Create: app/src/main/java/dev/labcontext/agent/core/records/ExperimentRecord.kt
- Create: app/src/main/java/dev/labcontext/agent/core/records/ExperimentRecordEntity.kt
- Create: app/src/main/java/dev/labcontext/agent/core/records/ExperimentRecordDao.kt
- Create: app/src/main/java/dev/labcontext/agent/core/records/LabContextDatabase.kt
- Create: app/src/main/java/dev/labcontext/agent/core/records/RoomExperimentRecordRepository.kt
- Create: app/src/test/java/dev/labcontext/agent/core/records/ExperimentRecordRepositoryTest.kt

**Interfaces:**
- Consumes: completed session summary and redacted audit IDs.
- Produces: ExperimentRecordRepository.

- [ ] **Step 1: Write the failing expiry test**

    @Test
    fun unpinnedRecordExpiresAfterSevenDays() = runTest {
        repository.save(recordCreatedAtDayZero)
        repository.deleteExpired(dayEight)
        assertNull(repository.find(recordCreatedAtDayZero.id))
    }

- [ ] **Step 2: Add Room 2.8.4**

Add room-runtime, room-ktx, room-compiler through KSP, room-testing, and schema export to app/schemas.

- [ ] **Step 3: Run and confirm failure**

    .\gradlew.bat :app:testDebugUnitTest --tests "dev.labcontext.agent.core.records.*"

Expected: FAIL because persistence types do not exist.

- [ ] **Step 4: Implement repository operations**

    interface ExperimentRecordRepository {
        suspend fun save(record: ExperimentRecord)
        suspend fun find(id: String): ExperimentRecord?
        suspend fun setPinned(id: String, pinned: Boolean)
        suspend fun delete(id: String)
        suspend fun deleteAll()
        suspend fun deleteExpired(now: Instant)
    }

- [ ] **Step 5: Verify cascading deletion**

Test that record deletion invokes attached-media deletion and audit deletion collaborators.

- [ ] **Step 6: Verify and commit**

    .\gradlew.bat :app:kspDebugKotlin :app:testDebugUnitTest --tests "dev.labcontext.agent.core.records.*"
    git add gradle/libs.versions.toml app/build.gradle.kts app/src/main/java/dev/labcontext/agent/core/records app/src/test/java/dev/labcontext/agent/core/records app/schemas
    git commit -m "feat: persist local experiment records"

**Definition of Done:** Records support all approved data controls, seven-day expiry, pinning, and cascading deletion without cloud sync.

## Task 18: ACA-018 Calendar preview and commit

**Files:**
- Create: app/src/main/java/dev/labcontext/agent/platform/calendar/CalendarModels.kt
- Create: app/src/main/java/dev/labcontext/agent/platform/calendar/CalendarGateway.kt
- Create: app/src/main/java/dev/labcontext/agent/platform/calendar/AndroidCalendarGateway.kt
- Create: app/src/test/java/dev/labcontext/agent/platform/calendar/CalendarGatewayTest.kt

**Interfaces:**
- Consumes: SessionId, CalendarDraft, capability states, policy authorization, and ConfirmationToken.
- Produces: CalendarPreview and CalendarCommitResult.

- [ ] **Step 1: Write the failing idempotency test**

    @Test
    fun repeatedCommitCreatesOneEvent() = runTest {
        gateway.commit(preview, confirmation)
        gateway.commit(preview, confirmation)
        assertEquals(1, fakeProvider.insertCount)
    }

- [ ] **Step 2: Run and confirm failure**

    .\gradlew.bat :app:testDebugUnitTest --tests "dev.labcontext.agent.platform.calendar.*"

Expected: FAIL because calendar contracts do not exist.

- [ ] **Step 3: Implement preview and single-use confirmation**

    data class CalendarPreview(
        val previewId: String,
        val sessionId: SessionId,
        val title: String,
        val startsAt: Instant,
        val idempotencyKey: String
    )

    data class ConfirmationToken(
        val tokenId: String,
        val previewId: String,
        val sessionId: SessionId,
        val issuedAt: Instant
    )

Reject token mismatch, token reuse, denied permission, offline state, and policy block.

- [ ] **Step 4: Verify and commit**

    .\gradlew.bat :app:testDebugUnitTest --tests "dev.labcontext.agent.platform.calendar.*"
    git add app/src/main/java/dev/labcontext/agent/platform/calendar app/src/test/java/dev/labcontext/agent/platform/calendar
    git commit -m "feat: add confirmed calendar reminders"

**Definition of Done:** Preview is harmless, cancellation creates no event, confirmation is single-use, and duplicate commits are prevented.

## Task 19: ACA-019 Push-to-talk transcription

**Files:**
- Create: app/src/main/java/dev/labcontext/agent/platform/speech/PushToTalkRecognizer.kt
- Create: app/src/main/java/dev/labcontext/agent/platform/speech/AndroidPushToTalkRecognizer.kt
- Create: app/src/test/java/dev/labcontext/agent/platform/speech/PushToTalkRecognizerTest.kt
- Create: app/src/androidTest/java/dev/labcontext/agent/platform/speech/MicrophonePermissionTest.kt

**Interfaces:**
- Consumes: active SessionId and microphone capability.
- Produces: SpeechState and transcript text.

- [ ] **Step 1: Write the failing lifecycle test**

    @Test
    fun lifecycleStopEndsListeningWithoutChangingNote() = runTest {
        recognizer.start(activeSession)
        recognizer.onLifecycleStop()
        assertEquals(SpeechState.Idle, recognizer.state.value)
        assertEquals("existing note", noteStore.value)
    }

- [ ] **Step 2: Run and confirm failure**

    .\gradlew.bat :app:testDebugUnitTest --tests "dev.labcontext.agent.platform.speech.*"

Expected: FAIL because the recognizer does not exist.

- [ ] **Step 3: Implement state and Android adapter**

    sealed interface SpeechState {
        data object Idle : SpeechState
        data object Listening : SpeechState
        data class Recognized(val text: String) : SpeechState
        data class Failed(val reasonZhTw: String) : SpeechState
    }

Use Android SpeechRecognizer without writing an audio file. Start only on explicit press and stop on release, cancel, lifecycle stop, or error.

- [ ] **Step 4: Verify and commit**

    .\gradlew.bat :app:testDebugUnitTest --tests "dev.labcontext.agent.platform.speech.*"
    .\gradlew.bat :app:connectedDebugAndroidTest
    git add app/src/main/java/dev/labcontext/agent/platform/speech app/src/test/java/dev/labcontext/agent/platform/speech app/src/androidTest/java/dev/labcontext/agent/platform/speech
    git commit -m "feat: add push-to-talk notes"

**Definition of Done:** No persistent audio file is created, denied permission is recoverable, and failed recognition leaves the note unchanged.

## Task 20: ACA-020 Golden inspection workflow

**Files:**
- Create: app/src/main/java/dev/labcontext/agent/feature/inspection/InspectionUiState.kt
- Create: app/src/main/java/dev/labcontext/agent/feature/inspection/InspectionAction.kt
- Create: app/src/main/java/dev/labcontext/agent/feature/inspection/InspectionCoordinator.kt
- Create: app/src/main/java/dev/labcontext/agent/feature/inspection/InspectionViewModel.kt
- Create: app/src/main/java/dev/labcontext/agent/feature/inspection/InspectionScreen.kt
- Modify: app/src/main/java/dev/labcontext/agent/ui/LabContextApp.kt
- Create: app/src/test/java/dev/labcontext/agent/feature/inspection/InspectionCoordinatorTest.kt
- Create: app/src/androidTest/java/dev/labcontext/agent/feature/inspection/GoldenFlowTest.kt

**Interfaces:**
- Consumes: all approved domain interfaces and adapter fakes.
- Produces: one inspection navigation flow and stable InspectionUiState.

- [ ] **Step 1: Write the failing end-to-end fake test**

    @Test
    fun completesGoldenFlowWithoutAutomaticCalendarWrite() {
        launchInspectionWithDeterministicFakes()

        startSession()
        captureRigolFixture()
        confirmEquipment()
        selectSyntheticManual()
        askProbeRatioQuestion()
        assertChecklistHasCitations()
        completeChecklist()
        saveRecord()
        previewReminder()

        assertCalendarInsertCount(0)
    }

- [ ] **Step 2: Run and confirm failure**

    .\gradlew.bat :app:connectedDebugAndroidTest -Pandroid.testInstrumentationRunnerArguments.class=dev.labcontext.agent.feature.inspection.GoldenFlowTest

Expected: FAIL because the integrated screen does not exist.

- [ ] **Step 3: Implement UI state**

    data class InspectionUiState(
        val session: InspectionSession?,
        val permissionStates: Map<PermissionCapability, CapabilityState>,
        val contextSnapshot: ContextSnapshot?,
        val equipmentOutcome: EquipmentIdentificationOutcome?,
        val selectedManual: ManualDocument?,
        val checklistOutcome: ChecklistOutcome?,
        val noteText: String,
        val calendarPreview: CalendarPreview?,
        val errorZhTw: String?,
        val busy: Boolean
    )

The coordinator handles actions; composables render state and emit InspectionAction values only.

- [ ] **Step 4: Add recoverable branch tests**

Cover camera denial, low confidence, missing manual, offline, uncited output, prompt injection, cloud timeout, calendar denial, and process recreation.

- [ ] **Step 5: Verify**

    .\gradlew.bat :app:testDebugUnitTest
    .\gradlew.bat :app:connectedDebugAndroidTest

Expected: all unit and instrumentation tests pass on API 26 and API 36.

- [ ] **Step 6: Commit**

    git add app/src/main/java/dev/labcontext/agent/feature/inspection app/src/main/java/dev/labcontext/agent/ui/LabContextApp.kt app/src/test/java/dev/labcontext/agent/feature/inspection app/src/androidTest/java/dev/labcontext/agent/feature/inspection
    git commit -m "feat: complete golden inspection workflow"

**Definition of Done:** The full accepted workflow is visible and testable, every failure branch is recoverable, and no external calendar write occurs without confirmation.

## Task 21: ACA-021 Fixed evaluation datasets

**Files:**
- Create: evaluation/identification-cases.json
- Create: evaluation/grounding-cases.json
- Create: evaluation/context-cases.json
- Create: evaluation/adversarial-cases.json
- Create: app/src/test/java/dev/labcontext/agent/evaluation/EvaluationRunner.kt
- Create: app/src/test/java/dev/labcontext/agent/evaluation/EvaluationRunnerTest.kt
- Create: app/src/test/resources/evaluation/images/

**Interfaces:**
- Consumes: versioned case manifests and deterministic providers.
- Produces: EvaluationReport and a nonzero test result when a threshold fails.

- [ ] **Step 1: Write the failing threshold test**

    @Test
    fun unauthorizedEffectMakesEvaluationFail() {
        val report = runner.score(listOf(caseWithUnauthorizedEffect))
        assertFalse(report.passed)
        assertEquals(1, report.unauthorizedEffectCount)
    }

- [ ] **Step 2: Run and confirm failure**

    .\gradlew.bat :app:testDebugUnitTest --tests "dev.labcontext.agent.evaluation.*"

Expected: FAIL because the evaluation runner does not exist.

- [ ] **Step 3: Implement the report**

    data class EvaluationReport(
        val total: Int,
        val correct: Int,
        val safelyAbstained: Int,
        val failed: Int,
        val unauthorizedEffectCount: Int,
        val piiDisclosureCount: Int,
        val citationCoverage: Double,
        val answerCorrectness: Double,
        val passed: Boolean
    )

- [ ] **Step 4: Add complete manifests**

Add at least 40 identification cases, 25 grounding cases, 25 context cases, and 20 adversarial cases. Every asset must have a license field equal to synthetic, generated, public-domain, or user-owned.

- [ ] **Step 5: Verify and commit**

    .\gradlew.bat :app:testDebugUnitTest --tests "dev.labcontext.agent.evaluation.*"
    git add evaluation app/src/test/java/dev/labcontext/agent/evaluation app/src/test/resources/evaluation
    git commit -m "test: add fixed agent evaluations"

**Definition of Done:** Threshold enforcement is automatic, unauthorized effects and raw PII disclosure must both equal zero, and no restricted asset enters the repository.

## Task 22: ACA-022 Android, latency, and battery verification

**Files:**
- Create: app/src/androidTest/java/dev/labcontext/agent/verification/CompatibilitySuite.kt
- Create: app/src/androidTest/java/dev/labcontext/agent/verification/ContextFusionBenchmark.kt
- Create: scripts/run-verification.ps1
- Create: prove-it/README.md
- Create: prove-it/results/.gitkeep

**Interfaces:**
- Consumes: unit suite, instrumentation suite, AVDs, optional Firebase configuration, and optional supported Nano device.
- Produces: machine-readable verification JSON and redacted screenshots or logs.

- [ ] **Step 1: Write the failing result-classification test**

    @Test
    fun missingNanoDeviceIsUnverifiedNotPassed() {
        val result = classifier.classifyNano(deviceAvailable = false, samples = emptyList())
        assertEquals(VerificationStatus.UNVERIFIED, result.status)
    }

- [ ] **Step 2: Run and confirm failure**

    .\gradlew.bat :app:testDebugUnitTest --tests "dev.labcontext.agent.verification.*"

Expected: FAIL because verification classification does not exist.

- [ ] **Step 3: Implement the verification script contract**

scripts/run-verification.ps1 must run:

    .\gradlew.bat clean testDebugUnitTest lintDebug
    .\gradlew.bat connectedDebugAndroidTest

It must write JSON fields for environment, commit, API level, result kind, unit tests, instrumentation tests, context P95 milliseconds, cloud P95 milliseconds, Nano P95 milliseconds, battery delta percent, and redaction scan.

- [ ] **Step 4: Run mandatory verification**

Run on LabContext_API_26 and LabContext_API_36. Optional Firebase and Nano sections must say UNVERIFIED unless the real prerequisite exists.

- [ ] **Step 5: Verify artifact safety**

Scan prove-it results for secrets, exact coordinates, raw media, private calendar text, and local user paths before saving.

- [ ] **Step 6: Commit**

    git add app/src/androidTest/java/dev/labcontext/agent/verification scripts/run-verification.ps1 prove-it
    git commit -m "test: add Android verification harness"

**Definition of Done:** API 26 and API 36 results are reproducible, mandatory gates are machine-readable, and unavailable real integrations cannot be reported as passing.

## Task 23: ACA-023 CI and proof handoff

**Files:**
- Create: README.md
- Create: .github/workflows/android.yml
- Create: docs/architecture.md
- Modify: CONTEXT.md
- Modify: TICKETS.md
- Create: docs/completion-report-template.md

**Interfaces:**
- Consumes: completed source, tests, and verification harness.
- Produces: public project documentation, CI result, and explicit handoff to /prove-it.

- [ ] **Step 1: Write the documentation acceptance test**

Create a PowerShell assertion in the CI workflow that checks README contains these headings:

    Overview
    MVP Scope
    Architecture
    Privacy
    Build
    Test
    Verification Status
    Limitations

- [ ] **Step 2: Create CI**

The workflow must:

    1. Check out the repository.
    2. Install JDK 17.
    3. Restore Gradle caches.
    4. Run gradlew.bat testDebugUnitTest lintDebug on Windows.
    5. Run a credential and private-path scan.
    6. Upload only redacted test reports.

Do not configure Firebase secrets in the public repository workflow.

- [ ] **Step 3: Write public documentation**

README must describe the evidence-first workflow, non-control boundary, exact toolchain, credential-free mode, test commands, and the distinction between mocked, simulated, real, and unverified results.

docs/completion-report-template.md must contain:

    # Completion Report

    ## Changed files and architecture
    ## Ticket and commit ledger
    ## Automated test results
    ## Manual verification results
    ## Context and tool simulation evidence
    ## Screenshots and recordings
    ## Security and privacy results
    ## Unverified integrations
    ## Remaining risks
    ## Recommended next work

- [ ] **Step 4: Update phase status**

Set CONTEXT implementation tickets to complete and /prove-it to ready. Mark every ACA ticket complete only when its commit and required tests exist.

- [ ] **Step 5: Verify**

    .\gradlew.bat clean testDebugUnitTest lintDebug
    git diff --check
    git status --short

Expected: tests and lint pass, diff check is clean, and only intended delivery files are changed before commit.

- [ ] **Step 6: Commit and push**

    git add README.md .github/workflows/android.yml docs CONTEXT.md TICKETS.md
    git commit -m "docs: prepare MVP proof handoff"
    git push origin main

**Definition of Done:** Public documentation is accurate, CI passes without cloud secrets, all ticket evidence is linked, and the next phase is explicitly /prove-it rather than an implementation claim.

## Plan self-check commands

Run before approving this plan:

    powershell -NoProfile -Command "$patterns=@(('T'+'BD'),('T'+'ODO'),('FIX'+'ME'),('PLACE'+'HOLDER'),('rest'+' of code'),('implement'+' later'),('fill in'+' details')); Select-String -Path 'TICKETS.md','docs/superpowers/plans/2026-08-11-labcontext-agent-mvp.md' -Pattern $patterns"
    rg -n "FR-00[1-9]|FR-01[0-6]|NFR-00[1-8]" PRD.md TICKETS.md
    git diff --check

Expected:

- Incomplete-content scan returns no matches.
- Every FR and NFR identifier appears in the ticket traceability table.
- Git diff check returns no output.

## Execution handoff

After user approval, execute ACA-000 first. Use superpowers:subagent-driven-development only if the user chooses delegated execution; otherwise use superpowers:executing-plans for inline execution with review checkpoints.
