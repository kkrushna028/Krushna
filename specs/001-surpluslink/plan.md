# Technical Plan: SurplusLink Autonomous Food-Rescue Platform

**Spec Reference**: `specs/001-surpluslink/spec.md`  
**Architecture Style**: Serverless Next.js App Router + Multi-Agent Orchestration + Server-Sent Events (SSE)

---

## 1. System Architecture & Tech Stack

### Frontend & UI Layer
- **Framework**: Next.js 16 (App Router), React 19, TypeScript
- **Styling**: Tailwind CSS v4 with custom Emerald / Teal palette (`#059669`, `#0d9488`, `#0f766e`, `#f0fdf4`)
- **Animation**: Framer Motion for smooth agent step timeline, layout transitions, and alert banners
- **Visualizations**: Recharts for impact metrics (kg rescued over time, carbon offset trends, model latency)
- **Internationalization**: Lightweight client context supporting English, Hindi (हिंदी), and Marathi (मराठी)
- **Responsive Breakpoints**: Explicit mobile-first testing at 360px, 768px, and 1440px

### Google & Cloud Services Layer
- **Gemini AI**: `@google/genai` SDK executed exclusively on server route handlers:
  - Fast Model (`gemini-2.5-flash`): Intake multimodal extraction, Safety inspection, Verifier audit.
  - Advanced Model (`gemini-1.5-pro` / `gemini-2.5-pro`): Matching multi-factor scoring, Logistics optimization.
  - Function Calling: Typed registry (`findNGOs`, `estimateRoute`, `getShelfLifeRules`, `draftMessage`).
  - Structured Outputs: Native JSON output via `responseSchema` mapped to Zod schemas.
- **Firebase Authentication & Firestore**:
  - Firebase Auth: Email/Password, Anonymous/Demo, Google Sign-in.
  - Firestore Collections: `donations`, `ngos`, `plans`, `auditLog`.
  - Security Rules: Enforce strict user-level tenancy (`donorId == request.auth.uid`).
  - Firebase App Check: Attestation token verification on mutation endpoints.
- **Google Cloud BigQuery**:
  - Dataset: `surpluslink_analytics`
  - Table: `plan_metrics` (schema: `plan_id`, `donor_id`, `timestamp`, `food_category`, `total_weight_kg`, `matched_ngo_id`, `tokens_prompt`, `tokens_completion`, `latency_ms`, `estimated_cost_usd`, `status`)
- **Google Maps API**:
  - Distance Matrix / Routes API for real travel time computation with high-speed haversine / mock fallback in demo mode.
- **Cloud Run Deployment**:
  - Multi-stage `Dockerfile`, standalone Next.js build, port 8080, autoscaling to 0 when idle.
  - Secrets injected via Google Cloud Secret Manager.

---

## 2. Directory Structure

```
surpluslink/
├── app/
│   ├── (auth)/login/page.tsx
│   ├── (dashboard)/
│   │   ├── page.tsx (Donor Intake & Planning Studio)
│   │   ├── ngos/page.tsx (NGO Coordinator Portal)
│   │   └── admin/page.tsx (Impact Analytics & Agent Health)
│   ├── api/
│   │   ├── agents/
│   │   │   └── orchestrate/route.ts (SSE Streaming Orchestrator)
│   │   ├── donations/route.ts
│   │   ├── plans/[id]/action/route.ts (Human-in-the-loop Approval)
│   │   ├── ngos/route.ts
│   │   └── analytics/route.ts (BigQuery reporting)
│   ├── globals.css
│   └── layout.tsx
├── lib/
│   ├── agents/
│   │   ├── orchestrator.ts (Main coordination engine with SSE & timeout budget)
│   │   ├── intakeAgent.ts
│   │   ├── safetyAgent.ts
│   │   ├── matchingAgent.ts
│   │   ├── logisticsAgent.ts
│   │   └── verifierAgent.ts
│   ├── tools/
│   │   ├── toolRegistry.ts (Allow-listed function calling definitions)
│   │   ├── findNGOs.ts
│   │   ├── estimateRoute.ts
│   │   └── shelfLifeRules.ts
│   ├── schemas/
│   │   ├── index.ts
│   │   ├── intakeSchema.ts
│   │   ├── safetySchema.ts
│   │   ├── matchingSchema.ts
│   │   ├── logisticsSchema.ts
│   │   ├── planSchema.ts
│   │   └── auditSchema.ts
│   ├── security/
│   │   ├── promptGuard.ts (Regex & XML data isolation)
│   │   ├── rateLimiter.ts (Token bucket memory & Redis-ready limiter)
│   │   └── cache.ts (SHA-256 keyed LRU response cache)
│   ├── gemini.ts (Client initialization with fallback handling)
│   ├── firebaseAdmin.ts (Server-side admin SDK initialization)
│   ├── firebaseClient.ts (Client-side auth & App Check initialization)
│   └── bigquery.ts (Metric ingestion client with offline queue)
├── evals/
│   ├── runEvals.ts (CLI runner for agent eval suite)
│   ├── testCases.json (20+ structured benchmark scenarios)
│   └── results.json
├── firestore.rules
├── Dockerfile
└── .env.local
```

---

## 3. Orchestration Engine (`lib/agents/orchestrator.ts`)

The orchestrator operates as an asynchronous generator streaming typed SSE events:
- **Event Types**:
  - `AGENT_START`: `{ agent: "INTAKE" | "SAFETY" | "MATCH" | "LOGISTICS" | "VERIFIER", step: number, timestamp: string }`
  - `AGENT_PROGRESS`: `{ agent: string, reasoning: string, toolCall?: { name: string, args: any } }`
  - `AGENT_COMPLETE`: `{ agent: string, output: any, durationMs: number }`
  - `PIPELINE_COMPLETE`: `{ plan: RescuePlan, metrics: PlanMetrics }`
  - `PIPELINE_ERROR`: `{ error: string, stage: string, recoverable: boolean }`

### Resilience & Guardrails
- **Timeout Budget**: Total execution budget capped at 14,000ms. If an agent step exceeds its budget (3,000ms), it activates deterministic rule-based fallbacks.
- **Max Steps Budget**: Capped at 8 total agent-tool transitions to prevent cyclic tool calls.
- **Schema Validation & Self-Healing**: Every agent response is validated via Zod. If parsing throws, the orchestrator triggers a single repair prompt with temperature 0.0 before failing over.
- **Idempotency Cache**: If `hash(input)` is in memory cache (<10 mins), cached plan is returned instantly.

---

## 4. Prompt Injection Defense Architecture

1. **Layer 1: Sanitization & Pattern Filter (`promptGuard.ts`)**
   - Strips command injection sequences (`ignore previous instructions`, `roleplay as`, `system override`, `you are now`, `drop database`).
2. **Layer 2: Structured Data Wrapping**
   - External text and OCR tokens are isolated in XML enclosures:
     ```
     <untrusted_user_input role="data_only">
     ${sanitizedInput}
     </untrusted_user_input>
     ```
3. **Layer 3: Strict Tool Allow-Listing**
   - The model can only call predefined functions declared in `toolRegistry.ts`. Arbitrary code execution or system shell invocation is physically impossible.

---

## 5. Food Safety Heuristics Engine

The Safety Agent implements biological food safety standards:
- **Danger Zone**: 4°C to 60°C (40°F to 140°F).
- **Cooked Perishable (Rice, Dal, Curries, Meat)**: Max 4 hours at room temperature, up to 24–48 hours if chilled immediately (<4°C).
- **Bakery / High Sugar / Dry Grains**: 24–72 hours safe window.
- **High-Risk Proteins (Seafood, Dairy, Cut Melons)**: High risk multiplier; safe window reduced by 50%.
- **Output Requirements**:
  - `safeUntil`: ISO 8601 string.
  - `confidence`: Float `0.0 - 1.0`.
  - `riskLevel`: `"LOW" | "MEDIUM" | "HIGH" | "HAZARD"`.
  - `distributionCaveat`: Mandatory verification checklist (Smell, Sight, Temperature check).

---

## 6. Observability & BigQuery Integration

Every completed or failed plan produces a `PlanMetrics` record:
```typescript
interface PlanMetrics {
  planId: string;
  donorId: string;
  timestamp: string;
  foodCategory: string;
  totalWeightKg: number;
  matchedNgoId: string;
  totalTokens: number;
  promptTokens: number;
  completionTokens: number;
  latencyMs: number;
  estimatedCostUsd: number;
  status: "DRAFT" | "APPROVED" | "REJECTED" | "DISPATCHED";
}
```
If BigQuery credentials are not configured in local development, metrics stream seamlessly to Firestore and local JSON logs without throwing unhandled exceptions.

---

## 7. Eval Harness & Quality Gates (`evals/`)

- 20+ benchmark cases covering:
  - Clean restaurant surplus
  - Hindi & Marathi transliterated notes
  - Adversarial prompt injection payloads
  - Extremely perishable foods near expiry
  - Remote locations with single / no matching NGOs
  - Blurry images with missing weights
- CLI runner `npm run eval` executes against the orchestrator and outputs:
  - Accuracy / Success Rate (%)
  - Average Latency (target < 8s)
  - Avg Tokens & Cost per Run
  - Injection Defense Pass Rate (must be 100%)
