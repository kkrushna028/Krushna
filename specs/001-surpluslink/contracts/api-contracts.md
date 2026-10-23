# API Contracts: SurplusLink

This document defines the HTTP endpoints, input payloads, validation schemas, and streaming contracts.

---

## 1. Multi-Agent Orchestration (Server-Sent Events)

### `POST /api/agents/orchestrate`
Executes the sequential 5-agent pipeline (`Intake` -> `Safety` -> `Match` -> `Route` -> `Verify`) and streams real-time step events to the client.

- **Headers**:
  - `Content-Type: application/json`
  - `Authorization: Bearer <Firebase_ID_Token>`
- **Request Body (JSON)**:
  ```typescript
  {
    inputType: "TEXT" | "PHOTO" | "AUDIO" | "CSV" | "DEMO";
    textPayload?: string;
    imageBase64?: string;
    audioBase64?: string;
    csvContent?: string;
    donorLocation?: {
      address: string;
      lat: number;
      lng: number;
    };
    preferredLanguage?: "en" | "hi" | "mr";
  }
  ```
- **Response Format**: `text/event-stream; charset=utf-8`
- **Stream Event Format**:
  ```
  event: AGENT_EVENT
  data: {
    stage: "INTAKE" | "SAFETY" | "MATCH" | "LOGISTICS" | "VERIFIER" | "DONE" | "ERROR",
    stepIndex: number, // 1 to 5
    title: string,
    reasoningTrace: string,
    durationMs: number,
    data?: any,
    metrics?: {
      promptTokens: number,
      completionTokens: number,
      costUsd: number
    }
  }
  ```
- **Terminal Event**:
  ```
  event: DONE
  data: {
    planId: string,
    plan: RescuePlanObject,
    totalLatencyMs: number,
    totalCostUsd: number
  }
  ```

---

## 2. Plan Human-in-the-Loop Action

### `POST /api/plans/[id]/action`
Processes explicit donor approval, rejection, or field modifications.

- **Request Body**:
  ```typescript
  {
    action: "APPROVE" | "REJECT" | "MODIFY";
    donorNotes?: string;
    modifiedItems?: Array<{ id: string; quantityKg: number }>;
  }
  ```
- **Response (200 OK)**:
  ```json
  {
    "success": true,
    "planId": "plan_abc123",
    "status": "APPROVED",
    "dispatchedAt": "2026-10-23T12:00:00Z",
    "notificationSent": true,
    "auditLogId": "audit_xyz789"
  }
  ```
- **Error Response (403 Forbidden)**:
  ```json
  {
    "error": "UNAUTHORIZED_PLAN_ACCESS",
    "message": "Only the originating donor can approve this plan."
  }
  ```

---

## 3. NGO Registry & Capacity Updates

### `GET /api/ngos`
Lists vetted NGOs filtered by coordinates or dietary categories.
- **Query Params**:
  - `lat`: number
  - `lng`: number
  - `radiusKm`: number (default 25)
  - `category`: string

### `PATCH /api/ngos/[id]`
Updates an NGO's live capacity or acceptance preferences.
- **Request Body**:
  ```typescript
  {
    currentOccupancyKg?: number;
    acceptingDonations?: boolean;
    openHours?: { open: string; close: string };
  }
  ```

---

## 4. Admin Observability & Impact Dashboard

### `GET /api/analytics`
Fetches aggregated metrics from BigQuery / Firestore cache.
- **Response (200 OK)**:
  ```json
  {
    "totalRescuedKg": 4820.5,
    "mealsRescued": 12051,
    "co2eAvoidedKg": 12051.25,
    "totalPlansCount": 184,
    "agentSuccessRate": 0.983,
    "avgLatencyMs": 4200,
    "avgCostPerPlanUsd": 0.0014,
    "recentPlans": [...]
  }
  ```
