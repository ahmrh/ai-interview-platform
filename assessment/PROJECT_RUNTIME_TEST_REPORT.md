# Senior engineering assessment: runtime, product coverage, and reliability

**Project:** AI Interview Platform
**Assessment date:** 27 October 2026
**Scope:** Local API and web application, checked against the project wiki and supplied full-stack brief
**Overall assessment:** The application services are available and the web production build succeeds. The assessor setup path and selected API workflows have passed local smoke checks. The candidate journey is not ready to be considered reliable: its Retry control was reproduced failing, the preflight depends on third-party speed-test hosts, and the end-to-end Gemini interview and result workflow was not verified. The repository currently has no effective automated regression suite.

This is a focused engineering assessment, not a penetration test or a certification of production readiness. It records what was exercised, what was observed in source, and what remains unknown.

## Executive summary

| Area | Assessment |
|---|---|
| API runtime | **Available.** Both health routes returned HTTP 200 in the final check. PostgreSQL, Redis, and Sidekiq had also been confirmed connected in the preceding setup checks. |
| Web application | **Build passes.** The Vite production build and TypeScript check completed successfully. Assessor screens loaded in the earlier smoke test using synthetic data. |
| Candidate preflight | **Not reliable in this run.** External speed-test requests failed in the browser, and Retry did not restart all checks. |
| Gemini-dependent features | **Unknown.** Stubbed client-contract checks passed earlier, but no real provider request or live interview was made. |
| Security review | **No confirmed vulnerability identified in the reviewed changes.** This was a bounded review, not exhaustive security testing. |
| Automated tests | **Not adequate for ongoing development.** Rails reports zero tests; RSpec fails loading a missing `spec_helper`; the web package has no test script. |

### Highest-priority actions

1. Make candidate preflight retries restart every check deterministically, and cover that behavior with an automated browser test.
2. Fix microphone and animation-frame cleanup paths; then verify them with a browser test using a controlled media-device mock.
3. Ensure a browser reconnect invalidates the previous disconnect-grace callback, and test the race with a short, controlled timer.
4. Replace hard-coded third-party bandwidth probes with a configurable or first-party check and a clear fallback policy.
5. Establish API and web regression suites before extending the full simulation workflow.

## Scope and method

The assessment used:

- The project wiki PRDs and the supplied brief, with emphasis on the assessor-to-candidate-to-results simulation.
- Local runtime checks against the API health routes and the available web application.
- Existing synthetic-data smoke checks for sign-in, skill lookup, assessment/session routes, and vacancy CRUD. Temporary synthetic records were removed after those checks.
- Browser screenshots documenting the candidate preflight and assessor setup states.
- A source-level review of the affected web media lifecycle and API WebSocket reconnect logic.
- A bounded security review. No active exploit payloads were run and no external target was tested.
- The available project test commands and a fresh web production build.

The local candidate browser session opened for this final pass used a synthetic candidate response. The browser automation environment denied microphone permission and the internet preflight did not finish in the allotted run. This is not a representative test of a real candidate device; it is recorded as a limitation, not treated as proof that all devices fail.

No Gemini key was printed, copied into this report, or sent to Google. The earlier Gemini checks used local stubs only. This report does not establish real model access, free-tier quota, provider billing behavior, or live audio compatibility.

## Findings

### P1 — Retry does not restart the candidate hardware checks

**Service:** Web
**Type:** Reproduced functional defect
**Confidence:** High
**User impact:** A candidate who encounters a recoverable preflight failure can be left unable to start the interview. The Start Interview button remains disabled while required checks are incomplete.

The browser first showed an Internet check failure. After Retry, it showed OS/browser stuck at “Checking...” and Internet still “Waiting”; the checks did not return to a completed state. The implementation resets check statuses, but the OS/browser check is started by a mount-only effect. Resetting the status does not cause that effect to run again. See [`HardwareCheck.tsx`](../web/src/components/HardwareCheck.tsx).

**Evidence**

- [Candidate internet check failed](./evidence/candidate-internet-check-failed.png)
- [Retry leaves checks waiting](./evidence/candidate-retry-stuck.png)

**Recommended correction:** Model each attempt explicitly (for example, with an attempt ID consumed by every check) or centralize the checks in a function that Retry invokes. Ensure that stale results from an earlier attempt cannot overwrite the current one. Add a browser regression test that fails before the correction and asserts all checks settle after Retry.

### P1 — Candidate readiness is coupled to external speed-test services

**Service:** Web
**Type:** Runtime blocker observed in this environment; scope of production impact is unknown
**Confidence:** High for the observed failure; medium for impact outside this environment
**User impact:** When the browser cannot reach the chosen external CDN and echo hosts, the internet check reports failing values and keeps Start Interview disabled—even if the application API itself is reachable.

The probe uses fixed third-party download URLs and default external upload hosts. Only ping and upload endpoint overrides are available in the inspected utility; the download sources are still hard-coded. See [`internetSpeedTest.ts`](../web/src/utils/internetSpeedTest.ts). Browser requests to the external test hosts failed during the earlier run, and the candidate screenshots record the resulting blocked state.

This does not prove the check fails on every network. It does show the preflight can become unavailable due to external host access, browser policy, or network filtering rather than a failure in this application.

**Recommended correction:** Prefer a small first-party endpoint or an explicit configurable probe set. Separate “measurement unavailable” from “connection below threshold,” and explain the difference to candidates. Keep the start decision tied to product-defined audio requirements rather than a single third-party download result.

### P1 — A stale browser-disconnect callback may end a reconnected interview

**Service:** API
**Type:** Reconnect-race concern identified by source review; not dynamically reproduced
**Confidence:** High from the reviewed control flow
**User impact:** If a candidate reconnects before the browser-disconnect grace period expires, an older callback may still run and terminate a session that is active again.

The disconnect handler records the disconnection and manages a grace period in [`audio_websocket_middleware.rb`](../api/app/channels/audio_websocket_middleware.rb). The reviewed flow does not clearly bind the delayed end action to the particular disconnect generation or invalidate it on a successful browser reconnect. This should be treated as a high-priority race to verify, not as a confirmed production incident: a controlled timing test was not run.

**Recommended correction:** Keep a cancellable timer or disconnect-generation token in session state. On reconnect, cancel/invalidate the pending disconnect action. Before ending the session, have the callback re-check that the browser is still disconnected and that the callback belongs to the current disconnect period.

**Regression test:** Use a short fake grace period: disconnect, reconnect before expiry, advance the clock, and assert the session stays active. Also test the opposite case—no reconnect, then expiry ends the session once.

### P2 — Microphone and audio-level monitoring cleanup is incomplete

**Service:** Web
**Type:** Resource/privacy lifecycle concern identified by source review; runtime cleanup was not verified
**Confidence:** High from the inspected cleanup paths; dynamic result unavailable
**User impact:** A microphone stream acquired for a mic-only check may remain active when the check is retried or the component is unmounted. The audio-level animation loop also schedules another frame without a corresponding cancellation path. A lingering stream can leave the microphone indicator active after the candidate leaves the check, while an uncancelled animation loop can continue doing work.

In [`HardwareCheck.tsx`](../web/src/components/HardwareCheck.tsx), the visible cleanup effect stops tracks from `videoStream`, while the mic-only stream is not clearly retained in that ref/state; the animation loop schedules `requestAnimationFrame` without a matching `cancelAnimationFrame` in its cleanup. This assessment is based on source inspection. The automated browser could not grant microphone access, so no claim is made that an actual device remained active in this run.

**Recommended correction:** Retain every acquired `MediaStream` in a ref and stop all its tracks on retry, completion, and unmount. Store the animation-frame ID and cancel it in the effect cleanup. Close any `AudioContext` created for monitoring. Test this with a fake stream whose tracks record calls to `stop()` and a fake animation-frame scheduler.

### P2 — No effective automated regression suite is available

**Services:** API and web
**Type:** Verification gap confirmed by project commands
**Confidence:** High
**User impact:** The observed retry behavior and the API lifecycle risks have no reliable CI checks to prevent recurrence.

- `bundle exec rails test` completed with **0 runs and 0 assertions**.
- `bundle exec rspec` failed before examples ran because [`.rspec`](../api/.rspec) requires a `spec_helper` that could not be loaded.
- The web build succeeds, but the web package does not expose a test script.

**Recommended correction:** Choose and document the intended test runners. Make the Rails suite execute meaningful model, request, and channel/service tests; configure RSpec correctly if it is the intended runner. Add a web component/browser suite focused first on hardware-check state and media cleanup. Wire both into CI and fail CI if a test command silently discovers zero tests.

### Security review — no confirmed vulnerabilities found

The bounded security review did not identify a confirmed security vulnerability in the reviewed changes. No active exploit payloads were run. This is not a claim that the whole application is secure: authentication/authorization boundaries, dependency exposure, production configuration, and the complete API surface still need systematic, repeatable coverage.

| # | Severity | File | Lines | Vulnerability | Confidence |
|---|----------|------|-------|---------------|------------|
| — | — | — | — | No confirmed vulnerabilities identified in the bounded review. | — |

## Runtime and test results

### API

- `GET /health`: **HTTP 200**, `{"status":"ok"}`.
- `GET /api/v1/health`: **HTTP 200**, `{"status":"ok"}`.
- PostgreSQL query and Redis ping: **passed** in the preceding runtime setup checks.
- Sidekiq registration: **one process connected to Redis** in the preceding setup checks.
- Synthetic admin login and skill taxonomy lookup: **passed** in the earlier smoke test; 22 seeded skills were returned.
- Assessment/session API checks and vacancy create/read/update/delete: **passed** with temporary synthetic data; those temporary records were removed.
- `bundle exec rails test`: **0 runs, 0 assertions, 0 failures, 0 errors**.
- `bundle exec rspec`: **failed before examples** with `LoadError: cannot load such file -- spec_helper`.

The health routes establish that the API process responds; they do not prove every business workflow or background job is healthy.

### Web

- `npm run build` from `web/`: **passed**, including the TypeScript check and Vite production build.
- Assessor login, assessment list, new-assessment form, and seeded skill picker: **passed** in the earlier browser/API smoke checks with synthetic data.
- Candidate interview route: rendered, but the external-network preflight did not complete successfully in the earlier test. Retry reproduced the stuck state.
- In the final browser automation pass, a synthetic candidate response loaded, but the internet check remained in progress and the browser returned `NotAllowedError` for microphone access. Audio readiness and interview start were therefore **not verified** in that pass.

## PRD workflow coverage

| PRD stage | Result | Evidence or limitation |
|---|---|---|
| Assessor sign-in and assessment listing | **Partially verified** | Loaded with a temporary synthetic account and assessment. |
| Skill selection | **Verified for the UI interaction** | Seeded taxonomy picker displayed 22 skills and accepted a selection. |
| Assessment submission and Gemini prompt generation | **Not tested** | Avoided triggering a Gemini-backed job. |
| Session invite and candidate-info lookup | **Verified through local API smoke checks** | Synthetic session only; no live interview was started. |
| Candidate hardware readiness and Retry | **Failed / not reliable** | Internet check was blocked in the earlier run; Retry reproduced the waiting-state defect. |
| Microphone/audio preflight cleanup | **Not dynamically tested** | Browser automation denied microphone access; source review found cleanup gaps. |
| Gemini Live connection, adaptive follow-ups, transcript, and coverage updates | **Not tested** | Requires a real provider session and successful candidate preflight. |
| Session end and portfolio generation | **Not tested** | Depends on a completed live session and provider processing. |
| Fit/gap review and export | **Not tested** | No generated portfolio or vacancy-linked result was available for this workflow. |
| WebSocket reconnect/resumption | **Not tested** | The stale disconnect callback is a source-level race concern; a controlled reconnect test remains necessary. |

The wiki simulation references Gemini 2.0 model IDs that were retired. Model configuration was updated in the preceding work, but model availability and behavior were not confirmed against Google. Stubbed REST and Live protocol checks are useful contract checks only; they are not evidence of provider access or free-tier quota.

## Evidence

The screenshots are browser captures from the earlier local run. Assessor views use synthetic data; they do not show real candidate records or credentials.

1. [Candidate internet check failed](./evidence/candidate-internet-check-failed.png)
2. [Retry leaves checks waiting](./evidence/candidate-retry-stuck.png)
3. [Assessor assessment list](./evidence/assessor-assessment-list.png)
4. [New assessment form](./evidence/assessment-form-loaded.png)
5. [Assessment skill selection](./evidence/assessment-skill-form.png)

The microphone cleanup and WebSocket reconnect findings are source-review findings, not screenshot-backed runtime reproductions. Capturing a screenshot would not demonstrate those lifecycle conditions; they need automated controlled tests.

## Improvement recommendations

### Stabilize the candidate entry path

- Make each preflight attempt an explicit unit of work. All checks should start, complete, time out, or fail against the same attempt identifier.
- Provide cancellation when a retry supersedes an in-flight check, and ignore late results from the superseded attempt.
- Remove hard-coded third-party download dependencies or provide a first-party/configurable probe with a documented fallback.
- Distinguish device permission denial, unsupported hardware, probe unavailability, and measured poor connectivity in both UI and telemetry.
- Do not enable the interview until required devices are ready, but do not make a candidate fail solely because a nonessential external measurement host is unreachable.

### Make media ownership explicit

- Centralize acquisition and disposal of microphone streams so every path stops every track.
- Cancel animation frames, timers, event listeners, and audio contexts during retry and unmount.
- Add user-visible confirmation when the microphone is active and when it is released.
- Exercise success, permission denial, retry, navigation away, and repeated start/stop in automated tests.

### Make reconnect behavior race-safe

- Treat each disconnect as a distinct generation and guard delayed callbacks against stale generations.
- Make the disconnect grace timer cancellable and observable.
- Test browser reconnect, Gemini reconnect, repeated disconnects, and session-end races with a fake clock and local fake WebSocket peers.
- Record structured session events around disconnect/reconnect without logging secrets, transcripts, or unnecessary personal data.

### Build a dependable test and delivery foundation

- Repair or replace the current API test setup and ensure zero-test discovery fails CI.
- Add Rails request/service/channel tests for authentication, assessment/session lifecycle, job enqueueing, and disconnect/reconnect behavior.
- Add web component tests for preflight state transitions and browser tests for candidate and assessor happy paths.
- Use deterministic fixtures and disposable test databases; do not make routine CI depend on Gemini, a real microphone, external CDNs, or email services.
- Keep provider contract tests separate from provider smoke tests. Make any real provider smoke test opt-in, explicit, quota-aware, and redacted.

## Enhancement strategy

The following sequence prioritizes user trust and a testable end-to-end foundation before adding more AI behavior.

### Phase 1 — Establish a trustworthy baseline

1. Document one supported local setup path for API, web, PostgreSQL, Redis, and Sidekiq.
2. Fix the test runner configuration and add a CI job that proves it actually ran tests.
3. Add health/readiness checks that distinguish process liveness from database, Redis, and queue readiness.
4. Create deterministic synthetic fixtures for an assessor, assessment, candidate session, and vacancy. Keep fixtures separate from development seed data.
5. Add an end-to-end workflow map based on the PRDs, with each step linked to an automated check or an explicitly documented manual validation.

**Exit criteria:** A new developer can start both projects from the documented instructions; CI runs a nonzero API suite and web suite; readiness accurately reports required dependencies.

### Phase 2 — Remove candidate-entry blockers

1. Correct Retry state orchestration and prevent stale preflight results from racing newer attempts.
2. Repair stream, audio-context, and animation-frame disposal.
3. Replace or configure the speed-test infrastructure and define sensible behavior when a probe is blocked.
4. Add browser coverage for network failure/recovery, microphone permission denied/granted, retry, and navigation away.
5. Improve candidate-facing diagnostics so the next action is obvious without exposing implementation details.

**Exit criteria:** A candidate can recover from a transient preflight failure; every acquired media resource is released on retry or exit; the application does not depend on an undocumented third-party host.

### Phase 3 — Verify the assessor-to-session lifecycle without external AI

1. Automate assessment creation, skill selection, invite creation, candidate-info lookup, session state changes, and authorization checks.
2. Use a fake Gemini adapter and fake WebSocket peer to test transcript/coverage events and reconnect handling deterministically.
3. Test background job retry, duplicate delivery, and failure reporting.
4. Validate that generated or partial results remain consistent when a job or connection fails midway.

**Exit criteria:** The complete local workflow can be exercised repeatedly without real Gemini access or manual database cleanup.

### Phase 4 — Validate Gemini integration deliberately

1. Confirm the chosen model IDs and supported Live API configuration against current provider documentation.
2. Add request/response contract tests that do not use the real key.
3. Provide a manual, opt-in smoke command that reports only redacted status, model, latency, and provider error category.
4. Test rate limits, quota exhaustion, malformed responses, timeouts, and provider outages using stubs first.
5. Run a short synthetic interview only when the account’s quota and billing constraints are understood; never rely on live provider calls in normal CI.

**Exit criteria:** The team can distinguish local integration defects from model availability, quota, and billing issues without disclosing credentials or candidate data.

### Phase 5 — Complete results, export, and operational readiness

1. Exercise session completion through portfolio generation, fit/gap analysis, vacancy matching, and export.
2. Add validation for incomplete sessions and partial AI output so users see a useful recoverable state rather than a broken result.
3. Define structured metrics for preflight completion, connection recovery, job latency, AI failure categories, and export success.
4. Add privacy-aware retention rules for audio/transcripts and verify access controls for every result and export route.
5. Run a release checklist covering migration safety, background workers, observability, backups, and rollback.

**Exit criteria:** The PRD’s end-to-end simulation succeeds with synthetic data; failures are observable and recoverable; production operations and data handling are documented.

## Bottom line

The project is runnable at the service/build level, but that is not yet equivalent to a verified product flow. The most concrete user-facing defect is candidate Retry. Candidate media cleanup and WebSocket disconnect handling also deserve fixes and targeted tests before relying on live interviews. The next milestone should be a deterministic, fully automated synthetic journey—from assessor setup through candidate session and results—before treating Gemini-backed production behavior as verified.
