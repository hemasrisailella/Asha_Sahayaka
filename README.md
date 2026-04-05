<div align="center">

# 🏥 ASHA Sahayak (आशा सहायक)

### AI-Powered Multilingual Maternal Health Assistant for India's ASHA Workers

[![Python 3.10+](https://img.shields.io/badge/Python-3.10%2B-blue.svg)](https://www.python.org/downloads/)
[![FastAPI](https://img.shields.io/badge/FastAPI-0.115-009688.svg)](https://fastapi.tiangolo.com/)
[![Gradio](https://img.shields.io/badge/Gradio-4.44-orange.svg)](https://gradio.app/)
[![Databricks](https://img.shields.io/badge/Databricks-Native-red.svg)](https://databricks.com/)
[![Sarvam AI](https://img.shields.io/badge/Sarvam%20AI-India%20Built-green.svg)](https://sarvam.ai/)
[![Tests](https://img.shields.io/badge/Tests-53%20passing-brightgreen.svg)](#testing)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](#-license)

**Built for the [BharatBricks Hackathon](https://bharatbricks.org) with Databricks**

### 🎬 [Watch the Demo Presentation](https://drive.google.com/file/d/1eNbwSAk4BDAAnUoTzonG4nUn3B5CrBD0/view?usp=sharing)

[Quick Start](#-quick-start) · [Architecture](#-architecture) · [Features](#-key-features) · [Demo](#-demo-scenarios) · [API Docs](docs/api_spec.md) · [Deployment](docs/deployment.md)

</div>

---

## 🎯 Problem Statement

India has over **1 million ASHA workers** serving as the first point of contact for maternal healthcare in rural areas. They manage pregnant women through antenatal care visits, nutrition counseling, risk identification, and government scheme enrollment — all with **paper-based tools** and **limited clinical training**.

**ASHA Sahayak** bridges this gap by providing an AI-powered, multilingual digital assistant that:
- Understands **10+ Indian languages** via text, voice, and image
- Applies **16 evidence-based risk rules** from India's MCP Card and Safe Motherhood Booklet
- Generates **scheme-aligned nutrition plans** (POSHAN 2.0, Saksham, ICDS)
- Schedules **ANC visits** with PMSMA (9th-of-month) alignment
- Extracts clinical data from **uploaded EHR reports** via OCR
- Provides **explainable, evidence-grounded responses** — never replacing clinicians

---

## ✨ Key Features

| Feature | Description | Technical Detail |
|---------|-------------|------------------|
| **Patient Management** | Register & track pregnant women | Auto-computes gestational weeks, trimester, EDD (Naegele's rule) |
| **EHR Upload & OCR** | Upload PDF/image reports | Extracts Hb, BP, blood sugar; auto-flags abnormalities |
| **Multilingual AI Chat** | Interact in 10+ Indian languages | Sarvam AI translation + STT + reasoning pipeline |
| **Patient-Aware RAG** | Grounded responses with citations | Dual-index retrieval: guidelines + patient history (FAISS/Vector Search) |
| **Risk Detection Engine** | 16 deterministic rules | MCP Card 2018 + Safe Motherhood Booklet; 4 severity bands |
| **Ration Planning** | Nutrition recommendations | POSHAN 2.0 / Saksham; trimester + condition-specific adjustments |
| **ANC Scheduling** | 7-visit ANC schedule | PMSMA 9th-of-month alignment; high-risk biweekly monitoring |
| **Village Dashboard** | Operational planning view | Risk queue, today's visits, ration aggregation, upcoming deliveries |
| **Audit Trail** | Full compliance logging | Every action, model call, and clinical decision is logged |

### Design Principles

- **Assistive, not autonomous** — recommends actions with evidence, never replaces clinicians
- **Explainable** — every output includes rule basis, evidence, and source references
- **Multilingual by default** — supports Hindi, Kannada, Telugu, Tamil, Marathi, Bengali, Gujarati, Malayalam, Punjabi, Odia, and English
- **Databricks-native** — Delta Lake, Unity Catalog, Vector Search, Workflows, Databricks Apps
- **India-built AI** — uses [Sarvam AI](https://sarvam.ai/) (sarvam-m, saaras:v3, sarvam-translate) as default providers
- **Offline-capable** — SQLite + FAISS + mock providers enable full functionality without internet

---

## 🏗️ Architecture

```
┌──────────────────────────────────────────────────────────┐
│                    Presentation Layer                      │
│  ┌────────────────────┐    ┌───────────────────────────┐  │
│  │   Gradio UI        │    │   React SPA (Optional)    │  │
│  │   (Mobile-first)   │    │   React 19 + Vite + TW    │  │
│  └────────┬───────────┘    └─────────┬─────────────────┘  │
│           │         FastAPI REST API  │                    │
└───────────┼───────────────────────────┼────────────────────┘
            │                           │
┌───────────┴───────────────────────────┴────────────────────┐
│                    Service Layer (9 Services)               │
│  ┌──────────┐ ┌──────────────┐ ┌──────────┐ ┌──────────┐  │
│  │ Patient  │ │ Conversation │ │ Document │ │Dashboard │  │
│  │ Service  │ │   Service    │ │ Service  │ │ Service  │  │
│  ├──────────┤ ├──────────────┤ ├──────────┤ ├──────────┤  │
│  │  Risk    │ │  Retrieval   │ │  Ration  │ │ Schedule │  │
│  │ Service  │ │   Service    │ │ Service  │ │ Service  │  │
│  ├──────────┤ └──────────────┘ └──────────┘ └──────────┘  │
│  │  Audit   │                                              │
│  │ Service  │                                              │
│  └──────────┘                                              │
└───────────────────────┬────────────────────────────────────┘
                        │
┌───────────────────────┴────────────────────────────────────┐
│              AI / ML Provider Layer (5 Providers)           │
│  ┌──────────┐ ┌───────────┐ ┌──────────┐ ┌────────────┐   │
│  │Reasoning │ │Translation│ │  Speech  │ │  Vision/   │   │
│  │(Sarvam-m)│ │ (Sarvam)  │ │(saaras)  │ │  OCR       │   │
│  └──────────┘ └───────────┘ └──────────┘ └────────────┘   │
│  ┌──────────────────────┐                                  │
│  │  Embeddings          │  Each provider: ABC interface    │
│  │  (multilingual-e5)   │  → Mock / Sarvam / Databricks   │
│  └──────────────────────┘                                  │
└───────────────────────┬────────────────────────────────────┘
                        │
┌───────────────────────┴────────────────────────────────────┐
│                  Data & Storage Layer                        │
│  ┌────────────┐ ┌─────────────┐ ┌──────────────────────┐   │
│  │ Delta Lake │ │   Volumes   │ │   Vector Search      │   │
│  │ (5 schemas,│ │ (PDFs, imgs,│ │  (guidelines +       │   │
│  │ 20+ tables)│ │  audio)     │ │   patient memory)    │   │
│  └────────────┘ └─────────────┘ └──────────────────────┘   │
│  Local: SQLite + FAISS │ Production: Databricks Lakehouse   │
└─────────────────────────────────────────────────────────────┘
```

See [docs/architecture.md](docs/architecture.md) for detailed architectural decisions, data flows, and component diagrams.

---

## 🚀 Quick Start

### Prerequisites

| Requirement | Version | Notes |
|-------------|---------|-------|
| Python | 3.10+ | Required |
| pip | Latest | Package manager |
| Sarvam AI API Key | — | Optional; mock providers work without it |
| Tesseract OCR | 4.0+ | Optional; for local OCR (`apt install tesseract-ocr`) |
| Node.js | 18+ | Optional; only for React frontend |

### 1. Clone & Install

```bash
git clone https://github.com/hemasrisailella/Asha_Sahayak.git
cd Asha_Sahayak

# Create virtual environment (recommended)
python -m venv .venv
source .venv/bin/activate  # Linux/Mac
# .venv\Scripts\activate   # Windows

# Install dependencies
pip install -r requirements.txt
```

### 2. Generate Synthetic Data & Launch

```bash
# Generate production-grade synthetic data (30 patients, 82 observations, 12 facilities)
python -m tools.generate_all

# Launch the app
python app/main.py
```

Open **http://localhost:8080**. The app auto-seeds the database on first launch with 30 patients across 3 villages.

### 3. Enable Real AI Providers (Optional)

```bash
# Copy the example env file and add your Sarvam API key
cp .env.example .env
# Edit .env:
#   SARVAM_API_KEY=your_key_here
#   REASONING_PROVIDER=sarvam
#   TRANSLATION_PROVIDER=sarvam
#   SPEECH_PROVIDER=sarvam
```

Without an API key, **mock providers** are used — all features work with simulated responses for demo purposes.

### 4. Run Tests

```bash
pytest tests/ -v
```

53 tests covering patient CRUD, risk engine (16 rules), scheduling, ration planning, RAG grounding, and dashboard aggregation.

### 5. Databricks Deployment

```bash
# Option A: GitHub → Databricks (recommended)
# 1. Push to GitHub, connect repo in Databricks Workspace
# 2. Run notebooks/setup_demo_data.py
# 3. Deploy as Databricks App

# Option B: CLI Bundle Deploy
databricks bundle validate
databricks bundle deploy
databricks bundle run asha_sahayak
```

See [docs/deployment.md](docs/deployment.md) for step-by-step instructions.

---

## 📋 Demo Scenarios

| # | Scenario | What It Demonstrates |
|---|----------|---------------------|
| 1 | **Normal Pregnancy** | Patient registration, auto-computed EDD/trimester, 7-visit ANC schedule |
| 2 | **Moderate Anemia** | Risk detection (Hb 8.5), ELEVATED band, double IFA + iron-rich ration |
| 3 | **High-Risk PIH** | Multiple triggers (BP 152/98 + prior C-section), emergency referral path |
| 4 | **Multilingual Chat** | Hindi/Kannada text + voice input, guideline-grounded response with citations |
| 5 | **Report Upload** | OCR extraction from lab report, abnormality auto-flagging, observation creation |
| 6 | **Village Dashboard** | Operational planning: risk queue, overdue visits, ration aggregation |

See [docs/demo_runbook.md](docs/demo_runbook.md) for step-by-step walkthrough with talking points.

---

## 🔬 Risk Engine — 16 Deterministic Rules

All rules are derived from India's **MCP Card 2018** and **Safe Motherhood Booklet**:

| ID | Rule | Category | Threshold | Band |
|----|------|----------|-----------|------|
| R001 | Severe Anemia | Lab | Hb < 7 g/dL | EMERGENCY |
| R002 | Severe Hypertension | Vitals | BP > 160/110 | EMERGENCY |
| R003 | Vaginal Bleeding | Symptom | Reported | EMERGENCY |
| R004 | Convulsions/Seizures | Symptom | Reported | EMERGENCY |
| R005 | Reduced Fetal Movement | Symptom | 3rd trimester | EMERGENCY |
| R006 | Severe Headache + Vision | Symptom | Both present | EMERGENCY |
| R007 | Pregnancy-Induced Hypertension | Vitals | BP > 140/90 | HIGH_RISK |
| R008 | Gestational Diabetes Risk | Lab | Fasting glucose > 126 | HIGH_RISK |
| R009 | Moderate Anemia | Lab | Hb 7–10 g/dL | ELEVATED |
| R010 | Adolescent Pregnancy | Demographic | Age < 18 | HIGH_RISK |
| R011 | Advanced Maternal Age | Demographic | Age > 35 | HIGH_RISK |
| R012 | Prior C-Section/Stillbirth | History | In known conditions | HIGH_RISK |
| R013 | High Fever | Symptom | > 38.5°C or reported | HIGH_RISK |
| R014 | Preterm Labour Signs | Symptom | Before 37 weeks | EMERGENCY |
| R015 | Urine Protein Elevated | Lab | Positive (not trace) | ELEVATED |
| R016 | Excessive Edema | Symptom | Generalized | HIGH_RISK |

**Scoring**: `NORMAL (0–15)` → `ELEVATED (16–40)` → `HIGH_RISK (41–85)` → `EMERGENCY (86–100)`

Each evaluation returns: `risk_band`, `risk_score`, `triggered_rules[]`, `reason_codes[]`, `suggested_next_action`, `emergency_flag`, `escalation_recommendation`.

---

## 🍽️ Ration Engine — POSHAN 2.0 / Saksham Aligned

| Trimester | Calories | Protein | Core Supplements | Key Foods |
|-----------|----------|---------|------------------|-----------|
| 1st | 2,200 | 55g | IFA + Folic acid | Rice/wheat, dal, greens, milk, fruits |
| 2nd | 2,500 | 65g | IFA + Calcium + Albendazole | + nuts, increased dal (120g), milk (600ml) |
| 3rd | 2,700 | 75g | IFA + Calcium + THR | + greens (150g), nuts (45g), extra grains |

**Condition-specific adjustments**: Anemia → +500 cal, double IFA, iron-rich foods; GDM → -300 cal, low GI diet; Underweight → +300 cal, extra ghee/THR; Hypertension → reduced sodium, DASH-style.

---

## 📅 ANC Scheduling — MCP Card Aligned

| Visit | Gestational Weeks | Key Tests |
|-------|------------------|-----------|
| ANC-1 | 0–12 | Blood group, Hb, Urine, HIV, VDRL, TT-1 |
| ANC-2 | 14–20 | Hb, BP, Urine, TT-2, Ultrasound, FHR |
| ANC-3 | 24–28 | Hb, BP, GDM screening, FHR |
| ANC-4 | 30–34 | Hb, BP, Urine, FHR, Fetal position |
| ANC-5 | 34–36 | BP, FHR, Fetal position, Birth prep |
| ANC-6 | 36–38 | BP, FHR, Delivery plan |
| ANC-7 | 38–40 | BP, FHR, Labour signs education |
| High-Risk | 28–40 | Biweekly: BP, Weight, FHR (if HIGH_RISK/EMERGENCY) |

All visits are aligned with **PMSMA** (Pradhan Mantri Surakshit Matritva Abhiyan) — clinic on the **9th of every month**.

---

## 🗂️ Repository Structure

```
Asha_Sahayak/
├── app/                        # Application entrypoint
│   ├── main.py                 # FastAPI + Gradio launcher with auto-seeding
│   ├── api.py                  # REST API endpoints (20+ routes)
│   ├── components/             # Shared UI components
│   └── pages/                  # Gradio pages (Home, Patients, Detail, Assistant, Dashboard)
│
├── services/                   # Business logic layer (9 services)
│   ├── patient_service.py      # Patient CRUD, profile management
│   ├── risk_service.py         # 16-rule deterministic risk engine
│   ├── ration_service.py       # POSHAN 2.0 nutrition recommendations
│   ├── schedule_service.py     # ANC scheduling with PMSMA alignment
│   ├── conversation_service.py # Multilingual RAG-based AI chat
│   ├── document_service.py     # EHR upload, OCR, abnormality detection
│   ├── retrieval_service.py    # Dual-index RAG retriever (FAISS/Vector Search)
│   ├── dashboard_service.py    # Village-level operational dashboard
│   ├── audit_service.py        # Compliance logging & model call tracking
│   └── db.py                   # Database abstraction (SQLite / Delta Lake)
│
├── providers/                  # AI provider abstractions (5 capabilities)
│   ├── base.py                 # Abstract interfaces for all providers
│   ├── config.py               # Factory pattern + env-driven provider selection
│   ├── reasoning/              # Sarvam-m / Databricks Llama 70B / Mock
│   ├── translation/            # Sarvam Translate (22 languages) / Mock
│   ├── speech/                 # Sarvam saaras:v3 STT / Mock
│   ├── vision/                 # pytesseract / Sarvam Vision 3B / Mock
│   └── embeddings/             # multilingual-e5-small (384-dim) / Mock
│
├── models/                     # Pydantic data models
│   ├── patient.py              # Patient, PatientSummary, AshaWorker
│   ├── clinical.py             # Observation, Report, Encounter, Medication
│   ├── risk.py                 # RiskRule, RiskEvaluation, AlertRecord
│   ├── ration.py               # RationItem, RationRecommendation, VillageRationSummary
│   ├── schedule.py             # ScheduleEntry, Appointment, DailyTaskList
│   └── common.py               # Enums (RiskBand, Trimester, Modality), utility functions
│
├── pipelines/                  # Batch processing pipelines
│   ├── seed_demo_data.py       # First-launch demo data seeding
│   ├── ingest_guidelines.py    # RAG guideline chunking & embedding
│   ├── daily_refresh.py        # Daily risk recomputation + overdue detection
│   ├── risk_refresh.py         # On-demand risk batch refresh
│   └── weekly_summary.py       # Weekly ration aggregation
│
├── tools/                      # Data generation & population tools
│   ├── generate_all.py         # Master script: generate everything + populate DB
│   ├── generate_synthetic_data.py  # 30 patients, 82 observations, facilities
│   ├── generate_sample_ehrs.py     # 5 sample EHR text reports
│   └── populate_production_db.py   # Load generated data into database
│
├── frontend/                   # React SPA (optional alternative UI)
│   ├── src/App.jsx             # Routes: Home, Patients, PatientDetail, Assistant, Dashboard
│   ├── package.json            # React 19, Vite, Tailwind CSS
│   └── vite.config.js          # Build configuration
│
├── sql/                        # Delta Lake DDL scripts
│   ├── 001_catalog.sql         # Catalog + 5 schemas + volumes
│   ├── 002_tables.sql          # 20+ tables (core, clinical, ops, reference, serving)
│   ├── 003_views.sql           # 8 reporting views
│   └── 004_seed_reference.sql  # Medical thresholds, nutrition rules, schedule rules
│
├── data/
│   ├── sample_reference/       # 5 maternal health guideline documents (RAG source)
│   ├── sample_ehr/             # 5 generated EHR text reports
│   ├── synthetic/              # Generated patient/observation/facility JSON
│   └── seed/                   # Minimal demo seed data (10 patients)
│
├── notebooks/                  # Databricks setup notebooks
│   ├── setup_demo_data.py      # One-click demo setup on Databricks
│   ├── create_vector_index.py  # Vector Search index creation
│   └── validate_demo_flow.py   # End-to-end demo validation
│
├── tests/                      # pytest test suite (53 tests, 6 suites)
│   ├── conftest.py             # Fixtures: fresh_db, sample patients, observations
│   ├── test_risk_rules.py      # All 16 risk rules + compound triggers
│   ├── test_schedule_generation.py  # ANC schedule + PMSMA alignment
│   ├── test_ration_engine.py   # Trimester baselines + condition adjustments
│   ├── test_patient_flow.py    # CRUD + auto-computation + search
│   ├── test_rag_grounding.py   # Retrieval quality + embedding
│   └── test_dashboard.py       # Aggregation + sorting + snapshots
│
├── docs/                       # Comprehensive documentation
│   ├── architecture.md         # System architecture + design decisions
│   ├── api_spec.md             # Full API specification with examples
│   ├── data_model.md           # Entity relationships + table schemas
│   ├── deployment.md           # Step-by-step deployment guide
│   └── demo_runbook.md         # 6 demo scenarios with talking points
│
├── config/
│   ├── app_config.yaml         # Application configuration
│   └── providers.example.yaml  # Provider API key reference
│
├── app.yaml                    # Databricks App entry point
├── databricks.yml              # Databricks Asset Bundle definition
├── requirements.txt            # Python dependencies (30 packages)
└── README.md                   # This file
```

---

## 🗄️ Data Model

5 schemas, 20+ tables organized under Unity Catalog:

| Schema | Tables | Purpose |
|--------|--------|---------|
| **core** | patients, asha_workers, villages | Primary entities |
| **clinical** | observations, reports, encounters, medications, patient_flags | Health records |
| **ops** | schedules, alerts, ration_plans, appointments | Operational tracking |
| **reference** | guidelines, medical_thresholds, nutrition_rules, schedule_rules, facilities | Lookup data |
| **serving** | guideline_chunks, patient_memory_chunks, retrieval_logs, audit_log | RAG + audit |

See [docs/data_model.md](docs/data_model.md) for complete entity relationships and column-level details.

---

## 🤖 AI Provider Architecture

All AI capabilities are abstracted behind provider interfaces, enabling swap between production APIs and offline mock providers:

| Capability | Default Provider | Alternative | Offline |
|-----------|-----------------|-------------|---------|
| **Reasoning** | Sarvam-m (24B) | Databricks Llama 3.1 70B | Mock (static responses) |
| **Translation** | Sarvam Translate | — | Mock (echo) |
| **Speech-to-Text** | Sarvam saaras:v3 | — | Mock (static text) |
| **Vision/OCR** | pytesseract (local) | Sarvam Vision (3B) | Mock (static extraction) |
| **Embeddings** | multilingual-e5-small | — | Mock (random vectors) |

**Provider resolution order**: Explicit argument → Environment variable → Config YAML → Default

```bash
# Configure via environment variables
export SARVAM_API_KEY=your_key
export REASONING_PROVIDER=sarvam    # sarvam | databricks | mock
export TRANSLATION_PROVIDER=sarvam  # sarvam | mock
export SPEECH_PROVIDER=sarvam       # sarvam | mock
export VISION_PROVIDER=pytesseract  # pytesseract | sarvam | mock
export EMBEDDING_PROVIDER=local     # local | mock
```

---

## 🏛️ Government Scheme Alignment

| Scheme | Integration |
|--------|------------|
| **PMSMA** (Pradhan Mantri Surakshit Matritva Abhiyan) | 9th-of-month ANC visit scheduling |
| **POSHAN 2.0 / Saksham** | Trimester-specific nutrition & ration recommendations |
| **MCP Card 2018** | 16 risk rules, ANC visit protocols, clinical thresholds |
| **JSSK** (Janani Shishu Suraksha Karyakram) | Free entitlements awareness in responses |
| **JSY / PMMVY** | Scheme eligibility tracking |
| **ICDS / Anganwadi** | THR (Take-Home Ration) distribution planning |

---

## 🧪 Testing

```bash
# Run all 53 tests
pytest tests/ -v

# Run specific test suite
pytest tests/test_risk_rules.py -v       # 16+ risk rule tests
pytest tests/test_schedule_generation.py  # ANC scheduling tests
pytest tests/test_ration_engine.py        # Nutrition engine tests
pytest tests/test_patient_flow.py         # Patient CRUD tests
pytest tests/test_rag_grounding.py        # RAG retrieval tests
pytest tests/test_dashboard.py            # Dashboard aggregation tests

# Run a single test
pytest -k "test_severe_anemia_triggers_emergency" -v
```

**Test coverage areas**: Patient lifecycle, all 16 risk rules (individual + compound), schedule generation, PMSMA alignment, ration baselines + condition adjustments, RAG chunk ingestion + retrieval, dashboard aggregation + sorting.

---

## 🛠️ Technology Stack

| Layer | Technology | Purpose |
|-------|-----------|---------|
| **Backend Framework** | FastAPI 0.115 + Uvicorn | REST API server |
| **UI Framework** | Gradio 4.44 | Mobile-first web interface |
| **Frontend (Alt)** | React 19 + Vite + Tailwind CSS | Optional modern SPA |
| **Data Models** | Pydantic 2.0 | Typed data validation |
| **Database (Local)** | SQLite (WAL mode) | Zero-config local storage |
| **Database (Prod)** | Delta Lake + Unity Catalog | Databricks Lakehouse |
| **Vector Store (Local)** | FAISS-cpu | In-memory similarity search |
| **Vector Store (Prod)** | Databricks Vector Search | Managed vector index |
| **LLM Reasoning** | Sarvam-m (24B India-built) | Maternal health Q&A |
| **Translation** | Sarvam Translate v1 | 22 Indian languages |
| **Speech-to-Text** | Sarvam saaras:v3 | Multilingual transcription |
| **OCR/Vision** | pytesseract / Sarvam Vision 3B | Document extraction |
| **Embeddings** | intfloat/multilingual-e5-small | 384-dim, 100+ languages |
| **Batch Processing** | Databricks Workflows | Daily/weekly pipelines |
| **Testing** | pytest + pytest-asyncio | 53 automated tests |
| **Experiment Tracking** | MLflow | Model versioning |

---

## 📊 Synthetic Data

The `tools/` folder generates production-grade synthetic data for demo and testing:

| Dataset | Count | Description |
|---------|-------|-------------|
| Patients | 30 | Across 3 villages (Hosahalli, Kuppam, Arjunpur) with varied demographics |
| Observations | 82 | Longitudinal vitals (2–3 per patient), realistic clinical ranges |
| ASHA Workers | 3 | One per village; Kannada, Telugu, Hindi speakers |
| Facilities | 12 | PHC, CHC, DH, FRU, AWC across all villages |
| EHR Reports | 5 | Antenatal report templates (normal, anemia, PIH, adolescent, GDM) |
| Reference Docs | 5 | ANC guidelines, danger signs, nutrition norms, PMSMA, risk protocols |

**Risk distribution**: ~40% normal, ~20% mild anemia, ~10% moderate anemia, ~10% hypertensive, ~10% high-risk, ~5% emergency, ~5% GDM

```bash
# Regenerate all synthetic data
python -m tools.generate_all

# Or generate individual datasets
python -m tools.generate_synthetic_data   # Patients + observations
python -m tools.generate_sample_ehrs      # EHR text reports
python -m tools.populate_production_db    # Load into database
```

**To transition to real data**: Replace `data/synthetic/*.json` with real patient exports and re-run `python -m tools.populate_production_db`. On Databricks, data moves to Delta Lake tables — see `sql/002_tables.sql`.

---

## ⚠️ Limitations & Future Work

### Current Limitations
- Single-user demo (no multi-tenant authentication)
- FAISS used instead of Databricks Vector Search in local mode
- No production ABDM/Aadhaar integration
- OCR accuracy depends on report quality and language

### Roadmap
- [ ] ML-based risk scoring to complement deterministic rules
- [ ] WhatsApp/SMS notification integration for ASHA workers
- [ ] Offline-capable Progressive Web App (PWA)
- [ ] Postnatal and neonatal care workflows
- [ ] ABDM Health ID (Ayushman Bharat Digital Mission) integration
- [ ] Multi-district scalability with role-based access
- [ ] Automated anomaly detection on longitudinal vitals
- [ ] Integration with eJanani / RCH portal

---

## 📚 Documentation

| Document | Description |
|----------|-------------|
| [Architecture](docs/architecture.md) | System design, data flows, architectural decisions |
| [API Specification](docs/api_spec.md) | Full REST API contract with request/response examples |
| [Data Model](docs/data_model.md) | Entity relationships, table schemas, RAG storage |
| [Deployment Guide](docs/deployment.md) | Step-by-step Databricks + local deployment |
| [Demo Runbook](docs/demo_runbook.md) | 6 interactive demo scenarios with talking points |

---

## 📄 License

This project is licensed under the **MIT License**.

---

## 🙏 Acknowledgements

- **[Sarvam AI](https://sarvam.ai/)** — India-built multilingual AI models (sarvam-m, saaras, sarvam-translate)
- **[Databricks](https://databricks.com/)** — Lakehouse platform, Unity Catalog, Vector Search, Databricks Apps
- **[BharatBricks Hackathon](https://bharatbricks.org)** — The platform that inspired this project
- **Ministry of Health & Family Welfare, India** — MCP Card, Safe Motherhood Booklet, PMSMA guidelines
- **Ministry of Women & Child Development** — POSHAN 2.0, ICDS/Saksham nutrition norms
- **India's 1M+ ASHA Workers** — The real heroes this tool is built to support

---

<div align="center">

**Built with ❤️ for India's maternal healthcare ecosystem**

*ASHA Sahayak — Because every pregnancy deserves evidence-based care, in every language.*

</div>
