# Architecture — ASHA Sahayak

> Comprehensive system architecture documentation for the ASHA Sahayak maternal health assistant.

---

## Table of Contents

- [System Overview](#system-overview)
- [Architecture Diagram](#architecture-diagram)
- [Layer Details](#layer-details)
  - [Presentation Layer](#1-presentation-layer)
  - [Service Layer](#2-service-layer)
  - [AI Provider Layer](#3-ai-provider-layer)
  - [Data & Storage Layer](#4-data--storage-layer)
- [Key Architectural Decisions](#key-architectural-decisions)
- [Data Flow Diagrams](#data-flow-diagrams)
- [Schema Organization](#schema-organization)
- [Provider Resolution](#provider-resolution)
- [RAG Pipeline Architecture](#rag-pipeline-architecture)
- [Security & Compliance](#security--compliance)

---

## System Overview

ASHA Sahayak follows a **4-layer architecture** designed for dual deployment — a **local SQLite + FAISS** mode for development/demo and a **Databricks Lakehouse** mode for production.

```
                    ┌─────────────────────────┐
                    │      ASHA Workers        │
                    │   (Mobile / Desktop)     │
                    └────────────┬────────────┘
                                 │
                    ┌────────────▼────────────┐
                    │   Presentation Layer     │
                    │  Gradio UI / React SPA   │
                    │  FastAPI REST API         │
                    └────────────┬────────────┘
                                 │
                    ┌────────────▼────────────┐
                    │    Service Layer         │
                    │  9 Domain Services       │
                    │  (Business Logic)        │
                    └────────────┬────────────┘
                                 │
                    ┌────────────▼────────────┐
                    │   AI Provider Layer      │
                    │  5 Abstract Providers    │
                    │  (Reasoning, Translation │
                    │   Speech, Vision, Embed) │
                    └────────────┬────────────┘
                                 │
                    ┌────────────▼────────────┐
                    │   Data & Storage Layer   │
                    │  SQLite / Delta Lake     │
                    │  FAISS / Vector Search   │
                    └─────────────────────────┘
```

---

## Architecture Diagram

```
┌───────────────────────────────────────────────────────────────────┐
│                      PRESENTATION LAYER                           │
│  ┌──────────────────────┐    ┌─────────────────────────────────┐  │
│  │     Gradio UI        │    │     React SPA (Optional)        │  │
│  │  ┌──────┐ ┌────────┐ │    │  React 19 + Vite + Tailwind    │  │
│  │  │ Home │ │Patients│ │    │  ┌──────┐ ┌──────┐ ┌────────┐  │  │
│  │  ├──────┤ ├────────┤ │    │  │ Home │ │ Chat │ │ Dashbd │  │  │
│  │  │ Chat │ │Dashbrd │ │    │  └──────┘ └──────┘ └────────┘  │  │
│  │  └──────┘ └────────┘ │    └──────────────┬──────────────────┘  │
│  └──────────┬───────────┘                   │                     │
│             │            FastAPI REST API    │                     │
│             └───────────────┬───────────────┘                     │
│                    20+ REST endpoints (app/api.py)                 │
└────────────────────────────┬──────────────────────────────────────┘
                             │
┌────────────────────────────┴──────────────────────────────────────┐
│                       SERVICE LAYER (9 Services)                   │
│                                                                    │
│  ┌──────────────┐  ┌──────────────────┐  ┌─────────────────────┐  │
│  │ PatientSvc   │  │ ConversationSvc  │  │  DocumentSvc        │  │
│  │ • CRUD       │  │ • Multilingual   │  │  • EHR upload       │  │
│  │ • Auto-fields│  │ • RAG pipeline   │  │  • OCR extraction   │  │
│  │ • Search     │  │ • Risk check     │  │  • Abnormality flag │  │
│  └──────────────┘  └──────────────────┘  └─────────────────────┘  │
│  ┌──────────────┐  ┌──────────────────┐  ┌─────────────────────┐  │
│  │ RiskSvc      │  │ RetrievalSvc     │  │  RationSvc          │  │
│  │ • 16 rules   │  │ • Guideline RAG  │  │  • POSHAN 2.0       │  │
│  │ • Scoring    │  │ • Patient memory │  │  • Condition adjust  │  │
│  │ • Bands      │  │ • FAISS/VS       │  │  • Village aggregate │  │
│  └──────────────┘  └──────────────────┘  └─────────────────────┘  │
│  ┌──────────────┐  ┌──────────────────┐  ┌─────────────────────┐  │
│  │ ScheduleSvc  │  │ DashboardSvc     │  │  AuditSvc           │  │
│  │ • 7 ANC vis. │  │ • Village summary│  │  • Action logging   │  │
│  │ • PMSMA align│  │ • Risk queue     │  │  • Model call track │  │
│  │ • High-risk  │  │ • Ration summary │  │  • Compliance trail  │  │
│  └──────────────┘  └──────────────────┘  └─────────────────────┘  │
│                                                                    │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ db.py — Database Abstraction (SQLite local / Databricks SQL)│   │
│  └─────────────────────────────────────────────────────────────┘   │
└────────────────────────────┬──────────────────────────────────────┘
                             │
┌────────────────────────────┴──────────────────────────────────────┐
│                    AI / ML PROVIDER LAYER                          │
│                                                                    │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐     │
│  │  Reasoning   │  │ Translation  │  │  Speech-to-Text      │     │
│  │  ┌─────────┐ │  │  ┌─────────┐ │  │  ┌────────────────┐  │     │
│  │  │Sarvam-m │ │  │  │ Sarvam  │ │  │  │ Sarvam saaras  │  │     │
│  │  │Llama 70B│ │  │  │Translate│ │  │  │ :v3            │  │     │
│  │  │Mock     │ │  │  │Mock     │ │  │  │ Mock           │  │     │
│  │  └─────────┘ │  │  └─────────┘ │  │  └────────────────┘  │     │
│  └──────────────┘  └──────────────┘  └──────────────────────┘     │
│  ┌──────────────┐  ┌──────────────────────────────────────────┐   │
│  │  Embeddings  │  │  Vision / OCR                            │   │
│  │  ┌─────────┐ │  │  ┌───────────┐ ┌─────────┐ ┌──────┐    │   │
│  │  │e5-small │ │  │  │pytesseract│ │ Sarvam  │ │ Mock │    │   │
│  │  │Mock     │ │  │  │(local)    │ │ Vision  │ │      │    │   │
│  │  └─────────┘ │  │  └───────────┘ └─────────┘ └──────┘    │   │
│  └──────────────┘  └──────────────────────────────────────────┘   │
│                                                                    │
│  providers/base.py — Abstract Base Classes (ABC)                   │
│  providers/config.py — Factory pattern + env-driven resolution     │
└────────────────────────────┬──────────────────────────────────────┘
                             │
┌────────────────────────────┴──────────────────────────────────────┐
│                    DATA & STORAGE LAYER                             │
│                                                                    │
│  LOCAL MODE                      │  PRODUCTION MODE                │
│  ┌──────────────────────┐        │  ┌───────────────────────────┐  │
│  │  SQLite (WAL mode)   │        │  │  Delta Lake               │  │
│  │  demo.db             │        │  │  Unity Catalog            │  │
│  └──────────────────────┘        │  │  5 schemas, 20+ tables    │  │
│  ┌──────────────────────┐        │  └───────────────────────────┘  │
│  │  FAISS (in-memory)   │        │  ┌───────────────────────────┐  │
│  │  Guideline index     │        │  │  Databricks Vector Search │  │
│  │  Patient memory idx  │        │  │  Managed endpoints        │  │
│  └──────────────────────┘        │  └───────────────────────────┘  │
│                                  │  ┌───────────────────────────┐  │
│                                  │  │  Volumes (PDFs, images,   │  │
│                                  │  │   audio, exports)         │  │
│                                  │  └───────────────────────────┘  │
│                                  │  ┌───────────────────────────┐  │
│                                  │  │  Databricks Jobs/         │  │
│                                  │  │  Workflows (daily, wkly)  │  │
│                                  │  └───────────────────────────┘  │
└───────────────────────────────────────────────────────────────────┘
```

---

## Layer Details

### 1. Presentation Layer

**Dual frontend architecture** supporting both a Gradio-based mobile-first UI (default) and an optional React SPA.

| Component | File | Technology | Purpose |
|-----------|------|-----------|---------|
| App launcher | `app/main.py` | FastAPI + Gradio | Starts server, mounts API + UI |
| REST API | `app/api.py` | FastAPI | 20+ REST endpoints |
| Home page | `app/pages/home.py` | Gradio | Overview stats, alerts, quick actions |
| Patients page | `app/pages/patients.py` | Gradio | List, search, create patients |
| Patient Detail | `app/pages/patient_detail.py` | Gradio | Full profile, vitals, history |
| AI Assistant | `app/pages/assistant.py` | Gradio | Multilingual chat with text/audio/image |
| Dashboard | `app/pages/dashboard.py` | Gradio | Village operational planning |
| React SPA | `frontend/src/` | React 19 + Vite | Optional modern SPA alternative |

**Request flow**: Browser → FastAPI (Uvicorn) → API Router or Gradio mount → Service Layer

### 2. Service Layer

Nine domain services encapsulate all business logic. Each service is a Python class that receives dependencies via constructor injection.

| Service | File | Responsibility | Key Methods |
|---------|------|---------------|-------------|
| **PatientService** | `patient_service.py` | Patient CRUD, auto-computed pregnancy fields | `create_patient()`, `update_patient()`, `search_patients()` |
| **RiskService** | `risk_service.py` | 16-rule deterministic risk assessment | `evaluate_patient()`, `evaluate_all_patients()` |
| **ScheduleService** | `schedule_service.py` | ANC scheduling, PMSMA alignment | `generate_schedule()`, `get_due_today()`, `get_overdue()` |
| **RationService** | `ration_service.py` | POSHAN 2.0 nutrition recommendations | `generate_recommendation()`, `aggregate_village_rations()` |
| **ConversationService** | `conversation_service.py` | Multilingual RAG-based AI chat | `process_message()`, `get_patient_history()` |
| **DocumentService** | `document_service.py` | EHR upload, OCR, abnormality detection | `process_upload()`, `get_patient_reports()` |
| **RetrievalService** | `retrieval_service.py` | Dual-index RAG retriever | `retrieve()`, `add_guideline_chunk()`, `add_patient_memory()` |
| **DashboardService** | `dashboard_service.py` | Village-level operational dashboard | `get_village_dashboard()`, `create_snapshot()` |
| **AuditService** | `audit_service.py` | Compliance logging, model tracking | `log()`, `log_model_call()` |

**Database abstraction** (`db.py`): Provides a unified interface over SQLite (local) and Databricks SQL (production) with connection pooling, WAL mode, JSON serialization, and row-as-dict results.

### 3. AI Provider Layer

Five AI capabilities are abstracted behind Python abstract base classes (ABCs), enabling runtime provider swapping via environment variables or config.

| Capability | Interface | Implementations | Default |
|-----------|-----------|-----------------|---------|
| **Reasoning** | `ReasoningProvider` | Sarvam-m (24B), Databricks Llama 3.1 70B, Mock | Sarvam |
| **Translation** | `TranslationProvider` | Sarvam Translate (22 Indian languages), Mock | Sarvam |
| **Speech-to-Text** | `SpeechToTextProvider` | Sarvam saaras:v3, Mock | Sarvam |
| **Vision/OCR** | `VisionExtractionProvider` | pytesseract (local), Sarvam Vision (3B), Mock | pytesseract |
| **Embeddings** | `EmbeddingProvider` | multilingual-e5-small (384-dim), Mock | Local (e5) |

**Factory pattern** (`providers/config.py`):
```python
# Resolution order: Explicit arg → Env var → Config YAML → Default
provider = get_reasoning_provider()       # Auto-resolved
provider = get_reasoning_provider("mock") # Explicit override
```

### 4. Data & Storage Layer

**Local mode**: SQLite database (`demo.db`) with WAL mode + FAISS in-memory vector indices. Zero-configuration, works offline.

**Production mode**: Databricks Lakehouse with Delta Lake tables under Unity Catalog, Databricks Vector Search for RAG, and Volumes for binary assets.

---

## Key Architectural Decisions

### ADR-1: Databricks-First Design
**Decision**: All application state lives in Delta Lake tables under Unity Catalog with 5 schemas (core, clinical, ops, reference, serving).  
**Rationale**: Enables ACID transactions, time travel, schema evolution, and unified governance for patient data.  
**Fallback**: SQLite provides identical schema locally for development and demo.

### ADR-2: Provider Abstraction Pattern
**Decision**: Every AI capability (reasoning, translation, speech, vision, embeddings) is behind an abstract Python interface.  
**Rationale**: Enables (a) mock providers for offline/demo use, (b) Sarvam AI as default India-built provider, (c) Databricks FM API as alternative, (d) easy addition of future providers without changing service logic.  
**Implementation**: Abstract base classes in `providers/base.py`, factory functions in `providers/config.py`.

### ADR-3: Deterministic Risk Engine (Rules-First)
**Decision**: 16 deterministic rules from MCP Card 2018 and Safe Motherhood Booklet run first, before any ML scoring.  
**Rationale**: Clinical safety requires predictable, auditable risk detection. ML can complement but never override deterministic rules for emergency detection.  
**Extension point**: Risk scoring layer is designed to accept ML model scores as an additive signal.

### ADR-4: Dual-Index RAG Architecture
**Decision**: Two separate vector indices — one for government guidelines, one for patient-specific memory.  
**Rationale**: Guidelines are shared across all patients (static); patient memory is per-patient (dynamic). Separate indices allow different update cadences, filtering, and relevance scoring.  
**Implementation**: FAISS locally, Databricks Vector Search in production.

### ADR-5: Scheme-Aligned Ration Engine
**Decision**: Nutrition recommendations follow POSHAN 2.0 / Saksham / MCP Card with explicit rule basis.  
**Rationale**: Government scheme alignment ensures recommendations are actionable (ASHA workers can reference specific scheme items). Every recommendation includes `rule_basis` and `source_refs` for auditability.

### ADR-6: English Pivot for Multilingual Processing
**Decision**: All user inputs are translated to English before processing; responses are generated in English then translated back.  
**Rationale**: Sarvam-m reasoning model performs best with English prompts. Source language is preserved in the encounter record for audit. Translation quality is handled by Sarvam Translate which supports 22 Indian languages natively.

### ADR-7: Assistive-Only Design
**Decision**: The system recommends actions and provides evidence but never autonomously makes clinical decisions.  
**Rationale**: ASHA workers are not clinicians. The system must support their work without creating liability or bypassing the medical referral chain. Every output includes evidence, source references, and recommended next steps for the worker.

---

## Data Flow Diagrams

### Patient Registration Flow

```
ASHA Worker fills form (name, age, village, LMP, phone)
        │
        ▼
┌──────────────────────────────────────────────┐
│  PatientService.create_patient()              │
│  • Validate inputs                            │
│  • Auto-compute: gestational_weeks (LMP→now)  │
│  • Auto-compute: trimester (weeks→1st/2nd/3rd)│
│  • Auto-compute: edd_date (LMP + 280 days)    │
│  • Persist to core.patients                    │
└──────────────────────┬───────────────────────┘
                       │
        ┌──────────────┼──────────────┐
        ▼              ▼              ▼
┌──────────────┐ ┌───────────┐ ┌──────────────┐
│ScheduleSvc   │ │ RiskSvc   │ │ RationSvc    │
│generate_     │ │evaluate_  │ │generate_     │
│schedule()    │ │patient()  │ │recommendation│
│• 7 ANC visits│ │• 16 rules │ │• Trimester   │
│• PMSMA align │ │• Scoring  │ │  baseline    │
│• High-risk   │ │• Band     │ │• Condition   │
│  extras      │ │           │ │  adjustments │
└──────┬───────┘ └─────┬─────┘ └──────┬───────┘
       ▼               ▼              ▼
   ops.schedules  patients.risk  ops.ration_plans
```

### Multilingual AI Chat Flow

```
Input: text (Hindi) / audio (Kannada) / image (lab report)
        │
        ▼
┌─────────────────────────────────────────────┐
│  ConversationService.process_message()       │
│                                              │
│  1. INPUT NORMALIZATION                      │
│     audio → SpeechProvider.transcribe()      │
│     image → VisionProvider.extract()         │
│     text  → use directly                     │
│                                              │
│  2. LANGUAGE HANDLING                        │
│     detect_language(input)                   │
│     translate(input → English)  [pivot]      │
│                                              │
│  3. PATIENT CONTEXT BUILDING                 │
│     • Recent vitals (Hb, BP, weight)         │
│     • Known conditions & medications         │
│     • Last 3 encounter summaries             │
│     • Pregnancy stage & risk band            │
│                                              │
│  4. RAG RETRIEVAL                            │
│     RetrievalService.retrieve(query)         │
│     → Top 5 guideline chunks                 │
│     → Top 3 patient memory chunks            │
│                                              │
│  5. RISK ASSESSMENT                          │
│     Extract symptoms from query              │
│     RiskService.evaluate_patient()           │
│     → Check all 16 rules                     │
│     → Flag emergencies                       │
│                                              │
│  6. PROMPT COMPOSITION                       │
│     System: "You are ASHA Sahayak..."        │
│     Context: patient profile + vitals        │
│     Guidelines: retrieved passages           │
│     Risk: triggered rules + actions          │
│     Query: translated user question          │
│                                              │
│  7. RESPONSE GENERATION                      │
│     ReasoningProvider.generate()             │
│     (temp=0.3, max_tokens=1024)              │
│                                              │
│  8. TRANSLATE BACK                           │
│     translate(response → source_language)    │
│                                              │
│  9. PERSIST & RETURN                         │
│     Store Encounter record                   │
│     Add to patient memory index              │
│     Return full response + evidence          │
└─────────────────────────────────────────────┘
```

### Daily Refresh Pipeline

```
Triggered: Databricks Job at 00:15 UTC daily
        │
        ▼
┌───────────────────────────────┐
│  1. Risk Recomputation        │
│  RiskService.evaluate_all()   │
│  • Run 16 rules for ALL       │
│    active patients             │
│  • Update risk_band/score     │
│  • Create EMERGENCY alerts    │
└───────────────┬───────────────┘
                │
                ▼
┌───────────────────────────────┐
│  2. Schedule Overdue Update   │
│  UPDATE ops.schedules         │
│  SET status='overdue'         │
│  WHERE due_date < today       │
│  AND status='scheduled'       │
└───────────────┬───────────────┘
                │
                ▼
┌───────────────────────────────┐
│  3. Dashboard Snapshots       │
│  For each village:            │
│  • Summary (totals, risk dist)│
│  • Today's visits             │
│  • Overdue visits             │
│  • High-risk queue            │
│  • Ration summary             │
│  Store as snapshot for history│
└───────────────┬───────────────┘
                │
                ▼
┌───────────────────────────────┐
│  4. Audit Trail               │
│  Log all actions with         │
│  timestamps & entity refs     │
└───────────────────────────────┘
```

### EHR Upload & OCR Pipeline

```
Input: PDF or JPEG/PNG image of lab report
        │
        ▼
┌─────────────────────────────────────────────┐
│  DocumentService.process_upload()            │
│                                              │
│  1. FILE INGESTION                           │
│     Detect type: PDF → pdf2image → image     │
│                  Image → direct              │
│                                              │
│  2. OCR / VISION EXTRACTION                  │
│     VisionProvider.extract_from_image()      │
│     → JSON: {hemoglobin, systolic_bp,        │
│       diastolic_bp, blood_sugar_fasting,     │
│       urine_protein, report_type, ...}       │
│                                              │
│  3. ABNORMALITY DETECTION                    │
│     Check extracted values against thresholds│
│     → Hb < 7: CRITICAL severe anemia         │
│     → Hb 7-10: WARNING moderate anemia       │
│     → BP > 160/110: CRITICAL hypertension    │
│     → BP > 140/90: WARNING elevated BP       │
│     → Glucose > 126: WARNING GDM risk        │
│     → Urine protein +: WARNING pre-eclampsia │
│                                              │
│  4. OBSERVATION CREATION                     │
│     Auto-create clinical.observations record │
│     with extracted values                    │
│                                              │
│  5. PATIENT MEMORY                           │
│     Build summary text of extraction         │
│     Add to patient memory RAG index          │
│                                              │
│  6. PERSIST                                  │
│     Store report in clinical.reports         │
│     Store observation, link to report        │
│     Return extraction + flags                │
└─────────────────────────────────────────────┘
```

---

## Schema Organization

All data is organized under 5 schemas (Unity Catalog in production, flat SQLite tables locally):

```
asha_sahayak (catalog)
├── core                    # Primary entities
│   ├── patients            # 23 columns, primary entity
│   ├── asha_workers        # Worker profiles
│   └── villages            # Village metadata
│
├── clinical                # Health records
│   ├── observations        # 18 columns, clinical measurements
│   ├── reports             # EHR documents + OCR results
│   ├── encounters          # 17 columns, conversation history
│   ├── medications         # Patient medications
│   └── patient_flags       # Risk markers
│
├── ops                     # Operational tracking
│   ├── schedules           # 13 columns, ANC visit schedule
│   ├── alerts              # Active alerts (severity-sorted)
│   ├── ration_plans        # 12 columns, nutrition plans
│   └── appointments        # Facility appointments
│
├── reference               # Lookup / configuration data
│   ├── guidelines          # Source guideline documents
│   ├── medical_thresholds  # Normal ranges, warning/critical values
│   ├── nutrition_rules     # Trimester + condition baselines
│   ├── schedule_rules      # ANC visit definitions
│   └── facilities          # PHC, CHC, DH, FRU, AWC master
│
└── serving                 # RAG & audit
    ├── guideline_chunks    # Embedded guideline text (384-dim vectors)
    ├── patient_memory_chunks # Embedded patient history
    ├── retrieval_logs      # RAG query logs
    └── audit_log           # Full compliance audit trail
```

---

## Provider Resolution

```
get_reasoning_provider(name=None)
        │
        ▼
  name provided? ──YES──→ Use explicit name
        │ NO
        ▼
  Env var set? ──YES──→ Use $REASONING_PROVIDER
  (REASONING_PROVIDER)
        │ NO
        ▼
  Config YAML? ──YES──→ Use app_config.yaml → providers.reasoning.default
  (app_config.yaml)
        │ NO
        ▼
  Use hardcoded default ("sarvam")
```

Same pattern applies for all 5 provider types.

---

## RAG Pipeline Architecture

```
┌──────────────────────────────────────────────────────┐
│                  INDEXING PIPELINE                     │
│                                                       │
│  data/sample_reference/*.md                           │
│       │                                               │
│       ▼                                               │
│  Chunk text (500 chars, 100-char overlap)             │
│       │                                               │
│       ▼                                               │
│  EmbeddingProvider.embed(chunks)                      │
│  → 384-dim vectors (multilingual-e5-small)            │
│       │                                               │
│       ▼                                               │
│  Store in guideline_chunks table + FAISS index        │
│  Categories: antenatal_care, danger_signs,            │
│              nutrition, pmsma, risk_assessment         │
└──────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────┐
│                 RETRIEVAL PIPELINE                     │
│                                                       │
│  User query (translated to English)                   │
│       │                                               │
│       ▼                                               │
│  EmbeddingProvider.embed_single(query)                │
│       │                                               │
│       ├──────────────────────────┐                    │
│       ▼                          ▼                    │
│  Guideline Index             Patient Memory Index     │
│  (top_k=5)                   (top_k=3, filtered       │
│                               by patient_id)          │
│       │                          │                    │
│       ▼                          ▼                    │
│  Merge + format context                               │
│       │                                               │
│       ▼                                               │
│  Inject into LLM prompt as "Evidence" section         │
└──────────────────────────────────────────────────────┘
```

---

## Security & Compliance

| Aspect | Implementation |
|--------|---------------|
| **Data Consent** | `consent_status` field on every patient (pending/granted/revoked) |
| **Audit Trail** | Every action logged via AuditService with actor, timestamp, entity |
| **Model Tracking** | All LLM calls logged: provider, model, tokens, latency |
| **No PII in Logs** | Patient names are not included in audit log details |
| **Explainability** | Every AI response includes `retrieved_chunks[]`, `triggered_rules[]`, `rule_basis[]` |
| **Human-in-the-Loop** | System recommends; ASHA worker decides and acts |
| **Encryption** | Databricks-managed encryption at rest; HTTPS in transit |
| **Access Control** | Unity Catalog RBAC for production table access |

## Schema Organization

| Schema | Purpose |
|--------|---------|
| `core` | Patients, ASHA workers, villages |
| `clinical` | Observations, reports, encounters, medications |
| `ops` | Schedules, alerts, ration plans, appointments |
| `reference` | Guidelines, thresholds, nutrition/schedule rules |
| `serving` | Guideline/patient chunks, retrieval logs, model audit |
