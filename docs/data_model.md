# Data Model — ASHA Sahayak

> Complete entity-relationship documentation covering all 5 schemas and 20+ tables.

---

## Table of Contents

- [Overview](#overview)
- [Entity Relationship Diagram](#entity-relationship-diagram)
- [Schema Organization](#schema-organization)
- [Core Schema](#core-schema)
- [Clinical Schema](#clinical-schema)
- [Operations Schema](#operations-schema)
- [Reference Schema](#reference-schema)
- [Serving Schema](#serving-schema)
- [Enums & Constants](#enums--constants)
- [Auto-Computed Fields](#auto-computed-fields)
- [Local vs Production Storage](#local-vs-production-storage)

---

## Overview

ASHA Sahayak uses a **5-schema architecture** organized under a Unity Catalog namespace. Locally, all tables are stored in a single SQLite database (`demo.db`); in production, each schema maps to a Databricks Unity Catalog schema with Delta Lake tables.

| Schema | # Tables | Purpose |
|--------|----------|---------|
| `core` | 3 | Primary entities (patients, workers, villages) |
| `clinical` | 5 | Health records (observations, reports, encounters) |
| `ops` | 4 | Operational tracking (schedules, alerts, rations) |
| `reference` | 5 | Lookup data (thresholds, rules, facilities) |
| `serving` | 4 | RAG indices + audit trail |
| **Total** | **21** | |

---

## Entity Relationship Diagram

```
                        ┌──────────────┐
                        │ asha_workers │
                        │  (core)      │
                        └──────┬───────┘
                               │ 1:N
                               ▼
┌──────────────┐       ┌──────────────┐       ┌──────────────┐
│  villages    │ 1:N   │   patients   │  1:N  │ observations │
│  (core)      │◄──────│   (core)     │──────►│ (clinical)   │
└──────────────┘       └──────┬───────┘       └──────────────┘
                              │
                    ┌─────────┼─────────┬──────────┬──────────┐
                    │ 1:N     │ 1:N     │ 1:N      │ 1:N      │
                    ▼         ▼         ▼          ▼          ▼
             ┌──────────┐ ┌────────┐ ┌──────────┐ ┌────────┐ ┌──────────────┐
             │ reports  │ │encntrs │ │schedules │ │alerts  │ │ ration_plans │
             │(clinical)│ │(clinicl│ │(ops)     │ │(ops)   │ │ (ops)        │
             └──────────┘ └────────┘ └──────────┘ └────────┘ └──────────────┘
                    │
                    │ 1:N
                    ▼
             ┌──────────────────────┐
             │ patient_memory_chunks│
             │ (serving)            │
             └──────────────────────┘

┌───────────────┐    ┌─────────────────┐
│  guidelines   │───►│guideline_chunks │     ┌────────────┐
│  (reference)  │    │(serving)        │     │ audit_log  │
└───────────────┘    └─────────────────┘     │ (serving)  │
                                              └────────────┘
Reference Tables (standalone):
  medical_thresholds, nutrition_rules, schedule_rules, facilities
```

---

## Schema Organization

### Unity Catalog Hierarchy (Production)
```
asha_sahayak           ← Catalog
├── core               ← Schema
│   ├── patients
│   ├── asha_workers
│   └── villages
├── clinical
│   ├── observations
│   ├── reports
│   ├── encounters
│   ├── medications
│   └── patient_flags
├── ops
│   ├── schedules
│   ├── alerts
│   ├── ration_plans
│   └── appointments
├── reference
│   ├── guidelines
│   ├── medical_thresholds
│   ├── nutrition_rules
│   ├── schedule_rules
│   └── facilities
└── serving
    ├── guideline_chunks
    ├── patient_memory_chunks
    ├── retrieval_logs
    └── audit_log
```

---

## Core Schema

### `core.patients`

Primary entity tracking pregnant women. Central table with foreign keys from most other tables.

| Column | Type | Nullable | Description |
|--------|------|----------|-------------|
| `patient_id` | STRING | PK | UUID, auto-generated |
| `asha_worker_id` | STRING | FK | Assigned ASHA worker |
| `full_name` | STRING | NOT NULL | Patient's full name |
| `age` | INT | NOT NULL | Age in years |
| `village` | STRING | NOT NULL | Village name (FK logical) |
| `phone` | STRING | | Contact phone number |
| `consent_status` | STRING | NOT NULL | `pending`, `granted`, `revoked` |
| `language_preference` | STRING | NOT NULL | ISO code: hi, en, kn, te, ta, mr, bn, gu, ml, pa, od |
| `lmp_date` | DATE | NOT NULL | Last menstrual period date |
| `edd_date` | DATE | | **Auto-computed**: LMP + 280 days (Naegele's rule) |
| `gestational_weeks` | INT | | **Auto-computed**: (today - LMP) / 7 |
| `trimester` | STRING | | **Auto-computed**: 1st (0-13w), 2nd (14-27w), 3rd (28+w) |
| `gravida` | INT | | Number of pregnancies including current |
| `parity` | INT | | Number of prior births |
| `known_conditions` | STRING (JSON) | | Array of conditions: `["mild_anemia", "prior_csection"]` |
| `current_medications` | STRING (JSON) | | Array: `["IFA", "Calcium"]` |
| `blood_group` | STRING | | A+, A-, B+, B-, AB+, AB-, O+, O- |
| `height_cm` | DOUBLE | | Height in centimeters |
| `risk_band` | STRING | NOT NULL | `NORMAL`, `ELEVATED`, `HIGH_RISK`, `EMERGENCY` |
| `risk_score` | DOUBLE | | 0-100 composite risk score |
| `created_at` | TIMESTAMP | NOT NULL | Record creation time |
| `updated_at` | TIMESTAMP | NOT NULL | Last update time |

**Indexes**: `patient_id` (PK), `village` (filter), `risk_band` (filter), `asha_worker_id` (FK)

### `core.asha_workers`

ASHA worker profiles.

| Column | Type | Description |
|--------|------|-------------|
| `worker_id` | STRING PK | UUID |
| `full_name` | STRING | Worker name |
| `village` | STRING | Assigned village |
| `phone` | STRING | Contact number |
| `language` | STRING | Primary language (ISO code) |

### `core.villages`

Village metadata.

| Column | Type | Description |
|--------|------|-------------|
| `village_id` | STRING PK | UUID |
| `village_name` | STRING | Village name |
| `block` | STRING | Administrative block |
| `district` | STRING | District |
| `state` | STRING | State |
| `primary_phc` | STRING | Nearest PHC name |
| `total_households` | INT | Household count |

---

## Clinical Schema

### `clinical.observations`

Clinical measurements recorded at each ANC visit or extracted from uploaded reports.

| Column | Type | Nullable | Description |
|--------|------|----------|-------------|
| `observation_id` | STRING | PK | UUID |
| `patient_id` | STRING | FK, NOT NULL | Reference to patients |
| `obs_date` | DATE | NOT NULL | Date of observation |
| `hemoglobin` | DOUBLE | | Hb in g/dL (normal: 11-15) |
| `systolic_bp` | INT | | Systolic BP in mmHg (normal: 90-120) |
| `diastolic_bp` | INT | | Diastolic BP in mmHg (normal: 60-80) |
| `blood_sugar_fasting` | DOUBLE | | Fasting glucose in mg/dL (normal: 70-95) |
| `blood_sugar_pp` | DOUBLE | | Post-prandial glucose in mg/dL |
| `weight_kg` | DOUBLE | | Weight in kilograms |
| `urine_protein` | STRING | | `nil`, `trace`, `+`, `++`, `+++` |
| `urine_sugar` | STRING | | `nil`, `trace`, `+`, `++` |
| `edema` | STRING | | `yes`, `no`, `present` |
| `fetal_movement` | STRING | | `normal`, `reduced`, `absent` |
| `fetal_heart_rate` | INT | | FHR in bpm (normal: 120-160) |
| `fundal_height_cm` | DOUBLE | | Fundal height in cm |
| `pallor` | STRING | | `nil`, `mild`, `moderate`, `severe` |
| `source_report_id` | STRING | FK | If extracted from a report |
| `notes` | STRING | | Free-text clinical notes |

### `clinical.reports`

Uploaded EHR documents with OCR extraction results.

| Column | Type | Description |
|--------|------|-------------|
| `report_id` | STRING PK | UUID |
| `patient_id` | STRING FK | Reference to patients |
| `file_path` | STRING | Storage path (local or Volume) |
| `file_type` | STRING | MIME type: `image/jpeg`, `image/png`, `application/pdf` |
| `report_date` | DATE | Date of the report |
| `extracted_json` | STRING (JSON) | OCR extraction: `{hemoglobin, systolic_bp, ...}` |
| `extracted_text` | STRING | Raw OCR text output |
| `extractor_confidence` | DOUBLE | 0.0-1.0 confidence score |
| `abnormality_flags` | STRING (JSON) | Array: `["WARNING: Moderate anemia (Hb 8.5)"]` |
| `created_at` | TIMESTAMP | Upload time |

### `clinical.encounters`

Conversation history — every AI interaction is stored as an encounter.

| Column | Type | Description |
|--------|------|-------------|
| `encounter_id` | STRING PK | UUID |
| `patient_id` | STRING FK | Reference to patients |
| `encounter_time` | TIMESTAMP | When the conversation happened |
| `modality` | STRING | `text`, `audio`, `image` |
| `source_language` | STRING | ISO code of user's language |
| `original_text` | STRING | User's input in original language |
| `normalized_text` | STRING | Cleaned/normalized input |
| `translated_text` | STRING | Input translated to English (pivot) |
| `summary` | STRING | Brief summary of the encounter |
| `symptoms` | STRING (JSON) | Extracted symptoms: `["headache", "blurred_vision"]` |
| `ai_response` | STRING | Generated response (English) |
| `translated_response` | STRING | Response translated to source language |
| `retrieved_chunks` | STRING (JSON) | Retrieved guideline passages |
| `risk_snapshot` | STRING (JSON) | Risk evaluation at time of encounter |
| `red_flag` | BOOLEAN | Whether any emergency rule triggered |
| `escalation_status` | STRING | `none`, `escalated`, `resolved` |
| `created_at` | TIMESTAMP | Record creation time |

### `clinical.medications`

Patient medication tracking.

| Column | Type | Description |
|--------|------|-------------|
| `medication_id` | STRING PK | UUID |
| `patient_id` | STRING FK | Reference to patients |
| `medication_name` | STRING | Drug name |
| `dosage` | STRING | e.g., "60mg + 500mcg" |
| `frequency` | STRING | e.g., "daily", "weekly" |
| `start_date` | DATE | When prescribed |
| `end_date` | DATE | When to stop (null = ongoing) |
| `prescribed_by` | STRING | Prescriber reference |
| `notes` | STRING | Additional notes |

### `clinical.patient_flags`

Risk markers and clinical flags for patients.

| Column | Type | Description |
|--------|------|-------------|
| `flag_id` | STRING PK | UUID |
| `patient_id` | STRING FK | Reference to patients |
| `flag_type` | STRING | e.g., `risk_rule`, `lab_abnormality`, `symptom` |
| `flag_code` | STRING | e.g., `R001`, `ANEMIA_SEVERE` |
| `severity` | STRING | `INFO`, `WARNING`, `CRITICAL`, `EMERGENCY` |
| `message` | STRING | Human-readable description |
| `created_at` | TIMESTAMP | When flagged |
| `resolved_at` | TIMESTAMP | When resolved (null = active) |

---

## Operations Schema

### `ops.schedules`

ANC visit schedule tracking per patient.

| Column | Type | Description |
|--------|------|-------------|
| `schedule_id` | STRING PK | UUID |
| `patient_id` | STRING FK | Reference to patients |
| `visit_type` | STRING | `ANC-1` through `ANC-7`, `PMSMA`, `High-Risk Monitoring` |
| `visit_number` | INT | Sequential visit number |
| `due_date` | DATE | When the visit is due |
| `suggested_slot` | STRING | Time slot: `09:00-11:00` |
| `facility_name` | STRING | PHC, CHC, DH, FRU, AWC name |
| `tests_due` | STRING (JSON) | Array: `["Hb", "BP", "GDM screening"]` |
| `status` | STRING | `scheduled`, `completed`, `missed`, `overdue`, `cancelled` |
| `is_pmsma_aligned` | BOOLEAN | True if due date is 9th of month |
| `escalation_flag` | BOOLEAN | True for high-risk extra visits |
| `notes` | STRING | Visit-specific notes |
| `completed_at` | TIMESTAMP | When marked completed |

### `ops.alerts`

Active patient alerts sorted by severity.

| Column | Type | Description |
|--------|------|-------------|
| `alert_id` | STRING PK | UUID |
| `patient_id` | STRING FK | Reference to patients |
| `severity` | STRING | `INFO`, `WARNING`, `CRITICAL`, `EMERGENCY` |
| `alert_type` | STRING | `risk_evaluation`, `overdue_visit`, `lab_abnormality` |
| `message` | STRING | Alert description |
| `reason_codes` | STRING (JSON) | Array of rule IDs: `["R001", "R002"]` |
| `created_at` | TIMESTAMP | Alert creation time |
| `resolved_at` | TIMESTAMP | Resolution time (null = active) |
| `resolved_by` | STRING | Who resolved the alert |

### `ops.ration_plans`

Weekly nutrition recommendations per patient.

| Column | Type | Description |
|--------|------|-------------|
| `ration_id` | STRING PK | UUID |
| `patient_id` | STRING FK | Reference to patients |
| `week_start` | STRING | Week start date |
| `trimester` | STRING | `1st`, `2nd`, `3rd` |
| `calorie_target` | INT | Daily calorie target (2200-3200) |
| `protein_target_g` | INT | Daily protein target in grams |
| `recommendation_json` | STRING (JSON) | Array of `{item_name, quantity, unit, frequency, category, rationale}` |
| `supplements` | STRING (JSON) | Array: `["IFA", "Calcium", "Vitamin C"]` |
| `special_adjustments` | STRING (JSON) | Condition-specific: `["Double IFA for moderate anemia"]` |
| `rationale` | STRING | Human-readable explanation |
| `rule_basis` | STRING (JSON) | Array: `["POSHAN_2.0_T3_BASELINE", "MCP_CARD_IFA"]` |
| `source_refs` | STRING (JSON) | Array: `["POSHAN 2.0 Guidelines", "MCP Card 2018"]` |

### `ops.appointments`

Facility appointments linked to schedule entries.

| Column | Type | Description |
|--------|------|-------------|
| `appointment_id` | STRING PK | UUID |
| `patient_id` | STRING FK | Reference to patients |
| `schedule_id` | STRING FK | Reference to schedules |
| `facility_id` | STRING FK | Reference to facilities |
| `appointment_date` | DATE | Date |
| `appointment_time` | STRING | Time slot |
| `status` | STRING | `booked`, `confirmed`, `completed`, `missed` |
| `notes` | STRING | Notes |

---

## Reference Schema

### `reference.guidelines`

Source guideline documents used for RAG retrieval.

| Column | Type | Description |
|--------|------|-------------|
| `guideline_id` | STRING PK | UUID |
| `title` | STRING | Document title |
| `source` | STRING | Source organization |
| `category` | STRING | `antenatal_care`, `danger_signs`, `nutrition`, `pmsma`, `risk_assessment` |
| `language` | STRING | Document language (ISO code) |
| `content` | STRING | Full text content |
| `version` | STRING | Document version |
| `effective_date` | DATE | When this version became effective |

### `reference.medical_thresholds`

Normal ranges, warning thresholds, and critical values for clinical parameters.

| Column | Type | Description |
|--------|------|-------------|
| `threshold_id` | STRING PK | UUID |
| `parameter` | STRING | `hemoglobin`, `systolic_bp`, `diastolic_bp`, `fasting_glucose`, `fetal_heart_rate` |
| `unit` | STRING | Measurement unit |
| `normal_min` | DOUBLE | Lower bound of normal range |
| `normal_max` | DOUBLE | Upper bound of normal range |
| `warning_min` | DOUBLE | Warning lower threshold |
| `warning_max` | DOUBLE | Warning upper threshold |
| `critical_min` | DOUBLE | Critical lower threshold |
| `critical_max` | DOUBLE | Critical upper threshold |
| `pregnancy_stage` | STRING | Which stage this applies to (all, 1st, 2nd, 3rd) |
| `notes` | STRING | Clinical notes |
| `source` | STRING | Source guideline |

**Seed data (key thresholds)**:

| Parameter | Normal | Warning | Critical | Source |
|-----------|--------|---------|----------|--------|
| Hemoglobin | 11-15 g/dL | < 10 | < 7 | MCP Card |
| Systolic BP | 90-120 mmHg | > 140 | > 160 | MCP Card |
| Diastolic BP | 60-80 mmHg | > 90 | > 110 | MCP Card |
| Fasting Glucose | 70-95 mg/dL | > 126 | > 200 | ANC Guidelines |
| Fetal Heart Rate | 120-160 bpm | 110-170 | <110 or >170 | ANC Guidelines |

### `reference.nutrition_rules`

Trimester and condition-specific nutrition rules based on POSHAN 2.0 and MCP Card.

| Column | Type | Description |
|--------|------|-------------|
| `rule_id` | STRING PK | UUID |
| `rule_name` | STRING | e.g., `T1_BASELINE`, `ANEMIA_MODERATE` |
| `trimester` | STRING | `1st`, `2nd`, `3rd`, or `all` |
| `condition` | STRING | `normal`, `anemia_moderate`, `anemia_severe`, `gdm_risk`, `underweight`, `hypertension` |
| `calorie_target` | INT | Daily calorie target |
| `protein_target_g` | INT | Daily protein target |
| `calorie_adjustment` | INT | Adjustment to baseline (+500, -300, etc.) |
| `supplements` | STRING (JSON) | Required supplements |
| `food_items` | STRING (JSON) | Recommended food items with quantities |
| `restrictions` | STRING (JSON) | Foods to avoid or limit |
| `rationale` | STRING | Clinical rationale |
| `source` | STRING | POSHAN 2.0, MCP Card, etc. |

### `reference.schedule_rules`

ANC visit definitions from MCP Card with timing and required tests.

| Column | Type | Description |
|--------|------|-------------|
| `rule_id` | STRING PK | UUID |
| `visit_type` | STRING | `ANC-1` through `ANC-7`, `HIGH_RISK_MONITORING` |
| `visit_number` | INT | Sequential number |
| `week_start` | INT | Earliest gestational week |
| `week_end` | INT | Latest gestational week |
| `ideal_week` | INT | Ideal scheduling week |
| `tests_required` | STRING (JSON) | Array of required tests |
| `counseling_topics` | STRING (JSON) | Topics to cover |
| `notes` | STRING | Additional guidance |
| `source` | STRING | MCP Card 2018, PMSMA, etc. |

### `reference.facilities`

Healthcare facility master data.

| Column | Type | Description |
|--------|------|-------------|
| `facility_id` | STRING PK | UUID |
| `facility_name` | STRING | Name of facility |
| `facility_type` | STRING | `PHC`, `CHC`, `DH`, `FRU`, `AWC` |
| `village` | STRING | Serving village |
| `address` | STRING | Physical address |
| `phone` | STRING | Contact number |
| `services` | STRING (JSON) | Available services |
| `operating_hours` | STRING | e.g., "08:00-17:00" |

---

## Serving Schema

### `serving.guideline_chunks`

Embedded text chunks from reference guidelines for RAG retrieval.

| Column | Type | Description |
|--------|------|-------------|
| `chunk_id` | STRING PK | UUID |
| `guideline_id` | STRING FK | Source guideline reference |
| `chunk_text` | STRING | 500-char text chunk with 100-char overlap |
| `source_name` | STRING | Source document name |
| `category` | STRING | `antenatal_care`, `danger_signs`, `nutrition`, `pmsma`, `risk_assessment` |
| `language` | STRING | Chunk language (ISO code) |
| `embedding` | ARRAY<DOUBLE> | 384-dimensional vector (multilingual-e5-small) |

**Chunking strategy**: 500 characters with 100-character overlap. Categories inferred from source filename.

### `serving.patient_memory_chunks`

Embedded patient-specific context (conversations, report summaries, observations) for personalized RAG.

| Column | Type | Description |
|--------|------|-------------|
| `chunk_id` | STRING PK | UUID |
| `patient_id` | STRING FK | Patient this memory belongs to |
| `chunk_text` | STRING | Summary text of encounter/observation/report |
| `chunk_type` | STRING | `encounter_summary`, `report_extraction`, `observation_summary` |
| `source_date` | STRING | Date of the source event |
| `language` | STRING | Language of the text |
| `embedding` | ARRAY<DOUBLE> | 384-dimensional vector |

### `serving.retrieval_logs`

RAG query logs for monitoring retrieval quality.

| Column | Type | Description |
|--------|------|-------------|
| `log_id` | STRING PK | UUID |
| `query_text` | STRING | User query (English) |
| `patient_id` | STRING | Patient context |
| `guideline_results` | STRING (JSON) | Retrieved guideline chunk IDs + scores |
| `patient_results` | STRING (JSON) | Retrieved patient memory chunk IDs + scores |
| `timestamp` | TIMESTAMP | Query time |

### `serving.audit_log`

Full compliance audit trail for every significant action.

| Column | Type | Description |
|--------|------|-------------|
| `log_id` | STRING PK | UUID |
| `timestamp` | TIMESTAMP | Event time |
| `action` | STRING | `patient_created`, `risk_evaluated`, `chat_response`, `report_uploaded`, etc. |
| `entity_type` | STRING | `patient`, `observation`, `encounter`, `report`, `schedule` |
| `entity_id` | STRING | ID of the affected entity |
| `actor` | STRING | `user`, `system`, `daily_refresh`, etc. |
| `details` | STRING (JSON) | Action details (no PII) |

---

## Enums & Constants

Defined in `models/common.py`:

```python
class RiskBand(str, Enum):
    NORMAL = "NORMAL"         # Score 0-15
    ELEVATED = "ELEVATED"     # Score 16-40
    HIGH_RISK = "HIGH_RISK"   # Score 41-85
    EMERGENCY = "EMERGENCY"   # Score 86-100

class Trimester(str, Enum):
    FIRST = "1st"    # Weeks 0-13
    SECOND = "2nd"   # Weeks 14-27
    THIRD = "3rd"    # Weeks 28+

class Modality(str, Enum):
    TEXT = "text"
    AUDIO = "audio"
    IMAGE = "image"

class VisitStatus(str, Enum):
    SCHEDULED = "scheduled"
    COMPLETED = "completed"
    MISSED = "missed"
    OVERDUE = "overdue"
    CANCELLED = "cancelled"

class AlertSeverity(str, Enum):
    INFO = "INFO"
    WARNING = "WARNING"
    CRITICAL = "CRITICAL"
    EMERGENCY = "EMERGENCY"

class ConsentStatus(str, Enum):
    PENDING = "pending"
    GRANTED = "granted"
    REVOKED = "revoked"
```

---

## Auto-Computed Fields

When `lmp_date` is set or updated on a patient, the following fields are automatically computed:

```python
# Gestational weeks: integer weeks since LMP
gestational_weeks = (date.today() - lmp_date).days // 7

# Trimester: derived from gestational weeks
if gestational_weeks <= 13:
    trimester = "1st"
elif gestational_weeks <= 27:
    trimester = "2nd"
else:
    trimester = "3rd"

# Expected Delivery Date: Naegele's rule
edd_date = lmp_date + timedelta(days=280)
```

These fields are recomputed on every patient update where `lmp_date` changes, triggering schedule and risk reevaluation.

---

## Local vs Production Storage

| Aspect | Local (Demo/Dev) | Production (Databricks) |
|--------|-------------------|------------------------|
| **Database** | SQLite (`demo.db`, WAL mode) | Delta Lake tables |
| **Vector Store** | FAISS (in-memory) | Databricks Vector Search |
| **File Storage** | Local filesystem | Databricks Volumes |
| **Schema** | Flat tables (no catalog/schema prefix) | Unity Catalog: `asha_sahayak.{schema}.{table}` |
| **Batch Jobs** | Manual Python scripts | Databricks Workflows |
| **Access Control** | None (single user) | Unity Catalog RBAC |

The `db.py` abstraction layer handles all differences transparently — service code is identical in both modes.
