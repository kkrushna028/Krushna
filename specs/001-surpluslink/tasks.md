# Tasks: SurplusLink Autonomous Food-Rescue Platform

**Spec Reference**: `specs/001-surpluslink/spec.md`  
**Plan Reference**: `specs/001-surpluslink/plan.md`  
**Status**: Ready for Implementation

---

## Group 1: Foundation, Firebase and Auth
- [ ] Task 1.1: Verify Next.js project setup, environment variables (`.env.local`), and tailwind theme configuration (Emerald/Teal palette).
- [ ] Task 1.2: Implement `lib/firebaseClient.ts` with Firebase Auth and optional App Check configuration.
- [ ] Task 1.3: Implement `lib/firebaseAdmin.ts` with Firebase Admin initialization and mock-safe fallback for local development.
- [ ] Task 1.4: Build Auth context (`lib/authContext.tsx`) with anonymous/demo login, email/password, and role detection (`donor`, `ngo`, `admin`).
- [ ] Task 1.5: Build Login & Onboarding view (`app/(auth)/login/page.tsx`) with 1-click Demo credentials.

## Group 2: Schemas, Tools, and Gemini Client
- [ ] Task 2.1 [P]: Create Zod schemas in `lib/schemas/`: `intakeSchema.ts`, `safetySchema.ts`, `matchingSchema.ts`, `logisticsSchema.ts`, `planSchema.ts`, and `auditSchema.ts`.
- [ ] Task 2.2 [P]: Implement `lib/gemini.ts` with `@google/genai` client, model selection (`gemini-2.5-flash` vs `gemini-1.5-pro`/`gemini-2.5-pro`), and structured JSON response configuration.
- [ ] Task 2.3 [P]: Implement typed tool registry `lib/tools/toolRegistry.ts` and tool implementations:
  - `findNGOs.ts` (geographic distance & dietary/capacity matching)
  - `estimateRoute.ts` (distance/duration with Maps or Haversine fallback)
  - `shelfLifeRules.ts` (microbiological safety heuristics)
  - `draftMessage.ts` (courteous SMS/WhatsApp draft generator)

## Group 3: Individual Agents with Unit Tests
- [ ] Task 3.1: Implement Intake Agent (`lib/agents/intakeAgent.ts`) extracting items, quantities, cooking time, and storage condition from multimodal input (photo, text, audio transcript, CSV).
- [ ] Task 3.2: Implement Safety Agent (`lib/agents/safetyAgent.ts`) assessing biological risk, calculating safe-until timestamp, confidence score, and distribution caveats.
- [ ] Task 3.3: Implement Matching Agent (`lib/agents/matchingAgent.ts`) invoking `findNGOs` tool to score and rank top 3 recipient organizations.
- [ ] Task 3.4: Implement Logistics Agent (`lib/agents/logisticsAgent.ts`) invoking `estimateRoute` and `draftMessage` to schedule pickup window and communications.
- [ ] Task 3.5: Implement Verifier Agent (`lib/agents/verifierAgent.ts`) performing cross-agent validation, time sanity checks, and schema verification.
- [ ] Task 3.6: Write unit tests in `tests/agents.test.ts` verifying all 5 agents produce valid Zod schemas and handle edge cases.

## Group 4: Orchestrator with SSE Streaming
- [ ] Task 4.1: Implement `lib/agents/orchestrator.ts` managing sequential agent execution, 14s timeout budget, 8-step execution budget, and event streaming.
- [ ] Task 4.2: Implement Server-Sent Events route handler `app/api/agents/orchestrate/route.ts` streaming typed events (`AGENT_START`, `AGENT_PROGRESS`, `AGENT_COMPLETE`, `PIPELINE_COMPLETE`, `PIPELINE_ERROR`).
- [ ] Task 4.3: Implement Human-in-the-Loop plan action endpoint `app/api/plans/[id]/action/route.ts` (`APPROVE`, `REJECT`, `MODIFY`).
- [ ] Task 4.4: Run Vitest tests on orchestrator and verify streaming output. *(Checkpoint 1: Preview after Group 4)*

## Group 5: Security Layer (Rules, Guard, Rate Limit, Audit Log)
- [ ] Task 5.1 [P]: Implement `lib/security/promptGuard.ts` with regex pattern sanitizer and `<untrusted_user_data>` isolation.
- [ ] Task 5.2 [P]: Implement `lib/security/rateLimiter.ts` (token bucket per IP/UID).
- [ ] Task 5.3 [P]: Implement `lib/security/cache.ts` (SHA-256 hash response cache).
- [ ] Task 5.4 [P]: Implement `lib/security/auditLog.ts` writing immutable records to Firestore / local log.
- [ ] Task 5.5: Write security rules in `firestore.rules` enforcing per-user data isolation and server-only audit log mutations.
- [ ] Task 5.6: Write security unit tests in `tests/security.test.ts` (prompt injection payloads, rate limit overflow, unauthorized plan modifications).

## Group 6: Donor UI (Input Studio, Live Agent Trace, Plan Approval)
- [ ] Task 6.1: Build Donor Studio (`app/(dashboard)/page.tsx`) with Multimodal Input Selector (Photo upload/webcam mock, voice note recorder mock, free text, and CSV upload).
- [ ] Task 6.2: Build 5-Scenario One-Click Demo Bar (Mess Leftovers, Bakery Surplus, Wedding Banquet, Blurry Photo, Adversarial Payload).
- [ ] Task 6.3: Build Live Agent Reasoning Timeline with Framer Motion animated states (Intake, Safety, Match, Route, Verify) showing live thoughts, durations, and token counts.
- [ ] Task 6.4: Build Verified Rescue Plan Card with safety badge, safe-until countdown, interactive NGO selection, editable draft message, and "Approve Rescue Plan" CTA.
- [ ] Task 6.5: Implement multilingual language toggle (English, Hindi, Marathi) with dynamic UI strings.
- [ ] Task 6.6: Add dark mode toggle and skeleton loaders for all asynchronous states. *(Checkpoint 2: Preview after Group 6)*

## Group 7: NGO Portal and Admin Dashboard with BigQuery
- [ ] Task 7.1: Build NGO Coordinator Portal (`app/(dashboard)/ngos/page.tsx`) with live capacity slider, accepted categories checkboxes, incoming rescue offer alerts, and Accept/Decline actions.
- [ ] Task 7.2: Implement BigQuery client `lib/bigquery.ts` with `plan_metrics` streaming buffer and offline fallback.
- [ ] Task 7.3: Build Admin Impact & Observability Dashboard (`app/(dashboard)/admin/page.tsx`) with Recharts visual graphs:
  - Kg rescued & Meals provided (KPIs)
  - CO2e Avoided (KPI)
  - Agent Success Rate & Latency Trends (Line / Area charts)
  - Token Usage & Cost ($USD) breakdown by model.
- [ ] Task 7.4: Implement mock dataset seeder with realistic historical data for high-impact demo visualization.

## Group 8: Eval Harness and Benchmark Test Suite
- [ ] Task 8.1: Construct `evals/testCases.json` with 20+ comprehensive benchmark scenarios across food types, regional languages, blurry inputs, extreme temperatures, and injection vectors.
- [ ] Task 8.2: Implement `evals/runEvals.ts` CLI harness measuring schema validity, accuracy, latency, token consumption, and injection defense pass rate.
- [ ] Task 8.3: Add `npm run eval` to `package.json` and execute eval suite, persisting benchmark metrics to `evals/results.json`.

## Group 9: Responsive Design & Accessibility Polish
- [ ] Task 9.1: Test and optimize viewport responsiveness at 360px (mobile), 768px (tablet), and 1440px (desktop) ensuring zero horizontal scroll.
- [ ] Task 9.2: Ensure WCAG 2.1 AA compliance: accessible color contrast ratios on Emerald/Teal palette, ARIA attributes, keyboard tab ordering, and focus indicators.
- [ ] Task 9.3: Add sound/haptic feedback or visual toast alerts for approvals and warnings. *(Checkpoint 3: Preview after Group 9)*

## Group 10: Docker & Cloud Run Deployment
- [ ] Task 10.1: Write multi-stage `Dockerfile` with standalone Next.js output.
- [ ] Task 10.2: Add `.dockerignore` and Cloud Run deployment configuration (`cloudbuild.yaml` or service specification).
- [ ] Task 10.3: Update `README.md` with comprehensive disclosure of all libraries, APIs, pretrained models, architecture diagram, and local running instructions.
- [ ] Task 10.4: Conduct Step 7 analysis and hardening audit.
