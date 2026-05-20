# Farm Service Orchestrator (VetiCall) 🐄🩺

An AI-driven veterinary dispatch and orchestration system tailored for rural and livestock farming regions. **VetiCall** leverages a sophisticated multi-agent pipeline powered by Google Gemini to parse, triage, match, and book local veterinary providers instantly using natural language (supporting English, Roman Urdu, Hindi, and Urdu scripts).

---

## 🏗️ System Architecture

The application is structured as a modern, high-performance monorepo separated into a React Native frontend and a fast Node.js/Express.js backend.

```mermaid
graph TD
    A[React Native / Expo Client] -->|1. Submit raw text| B[Express.js API Gateway]
    B -->|2. Invoke Multi-Agent Chain| C[Intent Agent]
    C -->|3. Structured Parse (Gemini 2.5)| B
    B -->|4. Query Vet Cohorts| D[Discovery Layer]
    B -->|5. Match & Rank candidates| E[Matching Agent]
    E -->|6. Score & Dynamic Reasoning (Gemini 2.5)| B
    B -->|7. Persist Trace & Return JSON| F[(PostgreSQL Database)]
    A -->|8. Render Live Pipeline Steps| A
    A -->|9. Select Slot & Book| G[POST /api/bookings]
    G -->|10. Write Booking & Generate Code| F
```

### Key Architectural Layers:
1. **Front-End (Mobile App)**: Built with **Expo (React Native)**. Features a gorgeous glassmorphism UI design, seamless Right-to-Left (RTL) localization for Urdu, live animated agent pipelines, and dynamic terminal logging showing real-time agent thought traces.
2. **Back-End (API Server)**: Built with **Express.js** and TypeScript. Coordinates requests, executes the agent sequence, and handles database operations.
3. **Database Layer**: **PostgreSQL** configured via **Drizzle ORM**. Persists user bookings and logs structured execution histories inside the `agent_traces` table.

---

## ♊ How Gemini is Used

Google Gemini lies at the core of the orchestration logic. We utilize the state-of-the-art **`gemini-2.5-flash`** model (via `@google/generative-ai` `v1beta`) configured with strict structured outputs utilizing modern `SchemaType` specifications.

### 1. Intent Classification Agent (`intentAgent.ts`)
Parses unstructured user input (including Romanized languages like *"meri gaay beemar hai, bukhar hai"*) and translates it into a rigid JSON structure.
* **Prompt Strategy**: Instructs Gemini to perform multi-lingual classification, detecting the primary animal type (e.g., cow, sheep, goat), the core symptom (e.g., fever, injury), overall urgency, and the detected language.
* **Schema Definition**:
  ```typescript
  const schema = {
    type: SchemaType.OBJECT,
    properties: {
      animal: { type: SchemaType.STRING, description: "Type of animal" },
      problem: { type: SchemaType.STRING, description: "Identified health concern" },
      urgency: { type: SchemaType.STRING, enum: ["low", "medium", "high", "critical"] },
      detected_language: { type: SchemaType.STRING, description: "Primary language used" },
    },
    required: ["animal", "problem", "urgency", "detected_language"],
  };
  ```

### 2. Veterinarian Matching Agent (`matchingAgent.ts`)
Ranks a cohort of local veterinarians based on their specialties, location, and availability against the parsed intent.
* **Prompt Strategy**: Gemini acts as an expert rural clinical coordinator. It compares the parsed intent (e.g., "cow fever") against candidate profiles and generates:
  - An eligibility rank.
  - A contextual, human-readable matching reasoning explaining exactly *why* this veterinarian is appropriate (e.g., *"Dr. Garcia has 10+ years in rural livestock management and is currently nearby"*).

---

## 🔌 API Endpoints

### 1. Orchestrate Request
* **Endpoint**: `POST /api/orchestrate`
* **Content-Type**: `application/json`
* **Request Body**:
  ```json
  {
    "text": "meri gaay ko bukhar hai"
  }
  ```
* **Response**:
  ```json
  {
    "session_id": "8a32b2e8-d6a0-4389-9e8c-8f9f83a45c32",
    "parsed_intent": {
      "animal": "cow",
      "problem": "fever",
      "urgency": "medium",
      "detected_language": "Roman Urdu"
    },
    "top_providers": [
      {
        "vetId": "v3",
        "name": "Dr. Garcia",
        "reasoning": "Specializes in large cattle and rural livestock with local availability."
      }
    ],
    "trace": [
      {
        "id": 402,
        "agentName": "intentAgent",
        "input": { "text": "meri gaay ko bukhar hai" },
        "output": { "animal": "cow", "problem": "fever", "urgency": "medium", "detected_language": "Roman Urdu" },
        "durationMs": 1420
      }
    ]
  }
  ```

### 2. Confirm Booking
* **Endpoint**: `POST /api/bookings`
* **Content-Type**: `application/json`
* **Request Body**:
  ```json
  {
    "animal": "cow",
    "problem": "Veterinary consultation with Dr. Garcia",
    "urgency": "medium",
    "time": "19 May 2026 at 11:00 AM",
    "location": "Pakpattan, Punjab",
    "detectedLanguage": "Roman Urdu",
    "status": "confirmed"
  }
  ```
* **Response**:
  ```json
  {
    "booking_code": "VC-20260519-0012",
    "booking": {
      "id": 12,
      "animal": "cow",
      "problem": "Veterinary consultation with Dr. Garcia",
      "urgency": "medium",
      "time": "19 May 2026 at 11:00 AM",
      "location": "Pakpattan, Punjab",
      "detectedLanguage": "Roman Urdu",
      "status": "confirmed",
      "createdAt": "2026-05-19T04:10:48.000Z"
    }
  }
  ```

### 3. Verify Agent Trace
* **Endpoint**: `GET /api/traces/:sessionId`
* **Purpose**: Retrieves the full, multi-agent execution traces matching a specific `session_id` directly from either the PostgreSQL database or the server's local in-memory fallback store so that judges can easily audit and verify the agent's real reasoning chain.
* **Path Parameters**:
  - `sessionId`: The session UUID generated during the orchestrator post request (e.g. `8a32b2e8-d6a0-4389-9e8c-8f9f83a45c32`).
* **Response**:
  ```json
  {
    "session_id": "8a32b2e8-d6a0-4389-9e8c-8f9f83a45c32",
    "trace": [
      {
        "id": 402,
        "agentName": "intentAgent",
        "input": { "text": "meri gaay ko bukhar hai", "sessionId": "8a32b2e8-d6a0-4389-9e8c-8f9f83a45c32" },
        "output": { "animal": "cow", "problem": "fever", "urgency": "medium", "detected_language": "Roman Urdu" },
        "durationMs": 1420
      },
      {
        "id": 403,
        "agentName": "matchingAgent",
        "input": { "intent": { ... }, "vets": [...], "sessionId": "8a32b2e8-d6a0-4389-9e8c-8f9f83a45c32" },
        "output": [ ... ],
        "durationMs": 1850
      }
    ]
  }
  ```

---

## 📊 Agent Trace System

The `agent_traces` table captures high-granularity records of every individual agent invocation. 

### Why We Trace:
1. **Observability**: Developers can monitor raw latency (`durationMs`) and exact model inputs/outputs to detect prompt drift or bottleneck operations.
2. **Audit Trails**: Retains records of intent parsing vs. vet scoring parameters.
3. **UI Integration**: The client parses the `trace` response directly, converting the raw execution logs into a live developer console and populating the reasonings inside collapsible steps.

### Drizzle Schema Definition:
```typescript
export const agentTracesTable = pgTable("agent_traces", {
  id: serial("id").primaryKey(),
  agentName: varchar("agent_name", { length: 255 }).notNull(),
  input: jsonb("input").notNull(),
  output: jsonb("output").notNull(),
  durationMs: integer("duration_ms").notNull(),
  createdAt: timestamp("created_at").defaultNow().notNull(),
});
```

---

## 🛠️ Getting Started

### Prerequisites
- Node.js `v18+`
- `pnpm` installed globally
- A Google Gemini API Key

### Installation

1. Clone the repository and install all workspace dependencies:
   ```bash
   pnpm install
   ```

2. Create a `.env` file in the root workspace and add your keys:
   ```env
   GEMINI_API_KEY=your_gemini_api_key_here
   DATABASE_URL=postgresql://your_db_connection_string
   ```

3. Spin up the API server:
   ```bash
   cd artifacts/api-server
   pnpm run build
   pnpm run start
   ```

4. Launch the Expo front-end:
   ```bash
   cd artifacts/mobile
   pnpm run dev
   ```

---

## 🛡️ Robust Fail-Safes

- **Database Timeout Catching**: The server intercepts slow or unreachable PostgreSQL network conditions, smoothly executing queries using in-memory fallbacks so that APIs never return `500` errors.
- **Client Resilience**: Front-end request screens include integrated `try-catch` structures that automatically execute high-fidelity local simulations if connection limits or server errors occur.
