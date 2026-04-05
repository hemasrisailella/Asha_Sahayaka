# API Specification — ASHA Sahayak

> Complete REST API reference for the ASHA Sahayak application. All endpoints are served via **FastAPI** at the base URL (default: `http://localhost:8080`).

---

## Table of Contents

- [Overview](#overview)
- [Authentication](#authentication)
- [Health & System](#health--system-endpoints)
- [Patient Management](#patient-management)
- [Clinical Observations](#clinical-observations)
- [Scheduling](#scheduling)
- [Ration & Nutrition](#ration--nutrition)
- [AI Conversation](#ai-conversation)
- [Document Upload](#document-upload)
- [Dashboard](#dashboard)
- [Error Handling](#error-handling)
- [Service-Level Contracts](#service-level-contracts)

---

## Overview

| Property | Value |
|----------|-------|
| **Base URL** | `http://localhost:8080` (local) or `https://{workspace}.databricks.com/apps/asha-sahayak` |
| **Protocol** | HTTP/HTTPS |
| **Content-Type** | `application/json` (default), `multipart/form-data` (file uploads) |
| **Framework** | FastAPI 0.115+ |
| **Server** | Uvicorn (async) |

## Authentication

Currently, the API does not require authentication (demo mode). For production deployment on Databricks, authentication is handled by Databricks App proxy.

---

## Health & System Endpoints

### `GET /api/health`

Health check with provider connectivity status.

**Response** `200 OK`:
```json
{
  "status": "healthy",
  "version": "1.1.0",
  "providers": {
    "reasoning": "sarvam",
    "translation": "sarvam",
    "speech": "sarvam",
    "vision": "pytesseract",
    "embeddings": "local"
  },
  "database": "sqlite",
  "timestamp": "2026-04-05T10:30:00Z"
}
```

### `GET /api/stats`

Global statistics across all patients.

**Response** `200 OK`:
```json
{
  "total_patients": 30,
  "risk_distribution": {
    "NORMAL": 12,
    "ELEVATED": 8,
    "HIGH_RISK": 6,
    "EMERGENCY": 4
  },
  "villages": ["Hosahalli", "Kuppam", "Arjunpur"],
  "trimester_distribution": {
    "1st": 8,
    "2nd": 12,
    "3rd": 10
  }
}
```

### `GET /api/alerts`

Active alerts across all patients, sorted by severity (EMERGENCY first).

**Response** `200 OK`:
```json
{
  "alerts": [
    {
      "alert_id": "uuid",
      "patient_id": "uuid",
      "patient_name": "Meena Kumari",
      "severity": "EMERGENCY",
      "alert_type": "risk_evaluation",
      "message": "Severe hypertension detected (BP 165/112). Immediate FRU referral required.",
      "reason_codes": ["R002"],
      "created_at": "2026-04-05T08:00:00Z",
      "resolved": false
    }
  ]
}
```

---

## Patient Management

### `GET /api/patients`

List all registered patients with optional filtering.

**Query Parameters**:
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `village` | string | No | Filter by village name |
| `search` | string | No | Search by name, phone, or village |
| `risk_band` | string | No | Filter by risk band (NORMAL, ELEVATED, HIGH_RISK, EMERGENCY) |

**Response** `200 OK`:
```json
{
  "patients": [
    {
      "patient_id": "uuid",
      "full_name": "Lakshmi Devi",
      "age": 26,
      "village": "Hosahalli",
      "gestational_weeks": 28,
      "trimester": "3rd",
      "risk_band": "NORMAL",
      "risk_score": 5.0,
      "edd_date": "2026-06-15",
      "phone": "9876543210"
    }
  ],
  "total": 30
}
```

### `GET /api/patients/{patientId}`

Get full patient profile with all details.

**Path Parameters**:
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `patientId` | string (UUID) | Yes | Patient unique identifier |

**Response** `200 OK`:
```json
{
  "patient_id": "uuid",
  "asha_worker_id": "uuid",
  "full_name": "Lakshmi Devi",
  "age": 26,
  "village": "Hosahalli",
  "phone": "9876543210",
  "language_preference": "kn",
  "consent_status": "granted",
  "lmp_date": "2025-09-15",
  "edd_date": "2026-06-22",
  "gestational_weeks": 28,
  "trimester": "3rd",
  "gravida": 2,
  "parity": 1,
  "known_conditions": ["mild_anemia"],
  "current_medications": ["IFA"],
  "blood_group": "B+",
  "height_cm": 155.0,
  "risk_band": "NORMAL",
  "risk_score": 5.0,
  "created_at": "2025-10-01T10:00:00Z",
  "updated_at": "2026-04-05T08:00:00Z"
}
```

**Response** `404 Not Found`:
```json
{
  "detail": "Patient not found"
}
```

### `POST /api/patients`

Register a new pregnant woman. Pregnancy fields (gestational_weeks, trimester, edd_date) are **auto-computed** from `lmp_date`.

**Request Body**:
```json
{
  "full_name": "Geeta Kumari",
  "age": 24,
  "village": "Hosahalli",
  "phone": "9876543211",
  "language_preference": "kn",
  "lmp_date": "2026-02-01",
  "gravida": 1,
  "parity": 0,
  "blood_group": "O+",
  "height_cm": 158.0,
  "known_conditions": [],
  "current_medications": []
}
```

**Response** `201 Created`:
```json
{
  "patient_id": "generated-uuid",
  "full_name": "Geeta Kumari",
  "gestational_weeks": 9,
  "trimester": "1st",
  "edd_date": "2026-11-08",
  "risk_band": "NORMAL",
  "risk_score": 0.0,
  "schedule_generated": true,
  "ration_generated": true
}
```

**Side effects**:
1. ANC schedule (7 visits) auto-generated via ScheduleService
2. Initial risk evaluation run via RiskService
3. Ration recommendation generated via RationService

### `PUT /api/patients/{patientId}`

Update patient profile. If `lmp_date` changes, all pregnancy fields are recomputed and schedule is regenerated.

**Request Body** (partial update):
```json
{
  "phone": "9876543222",
  "known_conditions": ["mild_anemia", "prior_csection"],
  "lmp_date": "2026-01-15"
}
```

**Response** `200 OK`:
```json
{
  "patient_id": "uuid",
  "full_name": "Geeta Kumari",
  "gestational_weeks": 11,
  "trimester": "1st",
  "edd_date": "2026-10-22",
  "risk_band": "HIGH_RISK",
  "risk_score": 60.0,
  "message": "LMP changed — schedule and risk recomputed"
}
```

---

## Clinical Observations

### `GET /api/patients/{patientId}/observations`

Get all clinical observations for a patient, ordered by date (newest first).

**Response** `200 OK`:
```json
{
  "observations": [
    {
      "observation_id": "uuid",
      "patient_id": "uuid",
      "obs_date": "2026-03-15",
      "hemoglobin": 11.5,
      "systolic_bp": 118,
      "diastolic_bp": 76,
      "blood_sugar_fasting": 85.0,
      "blood_sugar_pp": null,
      "weight_kg": 62.5,
      "urine_protein": "nil",
      "urine_sugar": "nil",
      "edema": "no",
      "fetal_movement": "normal",
      "fetal_heart_rate": 142,
      "fundal_height_cm": 28.0,
      "pallor": "nil",
      "source_report_id": null,
      "notes": "Normal ANC-4 visit"
    }
  ]
}
```

---

## Scheduling

### `GET /api/schedule/today`

Get all visits scheduled for today across all patients.

**Query Parameters**:
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `village` | string | No | Filter by village |

**Response** `200 OK`:
```json
{
  "date": "2026-04-05",
  "visits": [
    {
      "schedule_id": "uuid",
      "patient_id": "uuid",
      "patient_name": "Lakshmi Devi",
      "village": "Hosahalli",
      "visit_type": "ANC-4",
      "visit_number": 4,
      "facility_name": "Hosahalli PHC",
      "tests_due": ["Hb", "BP", "Weight", "Urine", "FHR", "Fetal position"],
      "risk_band": "NORMAL",
      "is_pmsma_aligned": false,
      "suggested_slot": "09:00-11:00"
    }
  ],
  "total": 3
}
```

### `GET /api/schedule/overdue`

Get all overdue visits sorted by days overdue (most overdue first).

**Response** `200 OK`:
```json
{
  "overdue_visits": [
    {
      "schedule_id": "uuid",
      "patient_id": "uuid",
      "patient_name": "Anita Sharma",
      "visit_type": "ANC-3",
      "due_date": "2026-03-28",
      "days_overdue": 8,
      "tests_due": ["Hb", "BP", "GDM screening"],
      "risk_band": "ELEVATED"
    }
  ],
  "total": 5
}
```

### `GET /api/schedule/patient/{patientId}`

Get the full ANC schedule for a specific patient.

**Response** `200 OK`:
```json
{
  "patient_id": "uuid",
  "patient_name": "Lakshmi Devi",
  "schedule": [
    {
      "schedule_id": "uuid",
      "visit_type": "ANC-1",
      "visit_number": 1,
      "due_date": "2025-10-15",
      "status": "completed",
      "facility_name": "Hosahalli PHC",
      "tests_due": ["Blood group", "Hb", "Urine", "HIV", "VDRL", "BP", "Weight", "TT-1"],
      "is_pmsma_aligned": false,
      "notes": ""
    },
    {
      "schedule_id": "uuid",
      "visit_type": "ANC-5",
      "visit_number": 5,
      "due_date": "2026-04-20",
      "status": "scheduled",
      "facility_name": "Hosahalli PHC",
      "tests_due": ["BP", "Weight", "FHR", "Fetal position", "Birth prep review"],
      "is_pmsma_aligned": false,
      "notes": ""
    }
  ]
}
```

### `PUT /api/schedule/{scheduleId}/mark-completed`

Mark a scheduled visit as completed.

**Response** `200 OK`:
```json
{
  "schedule_id": "uuid",
  "status": "completed",
  "completed_at": "2026-04-05T10:30:00Z"
}
```

---

## Ration & Nutrition

### `GET /api/ration/patient/{patientId}`

Get the latest ration recommendation for a patient.

**Response** `200 OK`:
```json
{
  "ration_id": "uuid",
  "patient_id": "uuid",
  "trimester": "3rd",
  "calorie_target": 2700,
  "protein_target_g": 75,
  "ration_items": [
    {
      "item_name": "Rice/Wheat",
      "quantity": 650,
      "unit": "g",
      "frequency": "daily",
      "category": "cereal",
      "rationale": "Primary energy source"
    },
    {
      "item_name": "Dal/Pulses",
      "quantity": 150,
      "unit": "g",
      "frequency": "daily",
      "category": "protein",
      "rationale": "Protein and iron source"
    },
    {
      "item_name": "Milk",
      "quantity": 600,
      "unit": "ml",
      "frequency": "daily",
      "category": "dairy",
      "rationale": "Calcium and protein"
    }
  ],
  "supplements": ["IFA (Iron + Folic Acid)", "Calcium 500mg"],
  "special_adjustments": [],
  "rationale": "3rd trimester (28+ weeks): Increased calorie and protein needs for fetal growth. Standard POSHAN 2.0 baseline with IFA and Calcium supplementation.",
  "rule_basis": ["POSHAN_2.0_T3_BASELINE", "MCP_CARD_IFA", "MCP_CARD_CALCIUM"],
  "source_refs": ["POSHAN 2.0 Guidelines", "MCP Card 2018"]
}
```

### `POST /api/ration/generate`

Generate a new ration recommendation for a patient (replaces previous).

**Request Body**:
```json
{
  "patient_id": "uuid"
}
```

**Response** `200 OK`: Same structure as GET above.

### `GET /api/ration/village/{villageName}/summary`

Weekly ration aggregation for a village — useful for Anganwadi distribution planning.

**Response** `200 OK`:
```json
{
  "village": "Hosahalli",
  "week_start": "2026-03-31",
  "total_beneficiaries": 10,
  "aggregate_rations": {
    "Rice/Wheat": {"total_kg": 45.5, "unit": "kg"},
    "Dal/Pulses": {"total_kg": 12.0, "unit": "kg"},
    "Milk": {"total_liters": 42.0, "unit": "liters"}
  },
  "supplement_counts": {
    "IFA": 10,
    "Calcium": 8,
    "Vitamin C": 3,
    "Albendazole": 2
  },
  "avg_calorie_target": 2467,
  "avg_protein_target_g": 65
}
```

---

## AI Conversation

### `POST /api/chat`

Send a multilingual message (text, audio, or image) to the AI assistant. Returns a patient-aware, guideline-grounded response.

**Request Body** (`multipart/form-data` or `application/json`):
```json
{
  "patient_id": "uuid",
  "text": "मुझे सिर दर्द हो रहा है",
  "source_language": "hi",
  "mode": "general",
  "audio_bytes": null,
  "image_bytes": null
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `patient_id` | string | Yes | Patient to contextualize the response |
| `text` | string | Yes* | Text query in any supported language |
| `source_language` | string | No | ISO code (hi, en, kn, te, ta, mr, bn, gu, ml, pa, od). Auto-detected if absent. |
| `mode` | string | No | `general` (default), `ration`, `risk_check`, `schedule` |
| `audio_bytes` | bytes | No | Audio recording (WAV/MP3) for voice input |
| `image_bytes` | bytes | No | Image of a report/document for OCR |

*At least one of `text`, `audio_bytes`, or `image_bytes` is required.

**Response** `200 OK`:
```json
{
  "encounter_id": "uuid",
  "original_text": "मुझे सिर दर्द हो रहा है",
  "translated_query": "I am having a headache",
  "ai_response": "Based on Lakshmi's current profile (28 weeks, BP 118/76), a headache alone is not immediately alarming. However, given that she is in her 3rd trimester, please check:\n\n1. **Blood pressure** — if BP > 140/90, this could indicate pregnancy-induced hypertension\n2. **Visual disturbances** — ask if she has blurred vision or seeing spots\n3. **Edema** — check for swelling in face and hands\n\nIf headache is severe AND accompanied by vision changes, refer immediately to PHC for pre-eclampsia evaluation (MCP Card Rule R006).\n\n**Recommended action**: Measure BP now. If normal, advise rest and hydration. If elevated, escalate to PHC.",
  "translated_response": "लक्ष्मी की वर्तमान प्रोफ़ाइल (28 सप्ताह, BP 118/76) के आधार पर...",
  "retrieved_guidelines": [
    {
      "chunk_id": "uuid",
      "text": "Danger sign: Severe headache with visual disturbance may indicate pre-eclampsia...",
      "source": "danger_signs.md",
      "category": "danger_signs",
      "relevance_score": 0.89
    }
  ],
  "retrieved_patient_context": [
    {
      "chunk_id": "uuid",
      "text": "ANC-4 visit on 2026-03-15: BP 118/76, Hb 11.5, normal FHR...",
      "source": "observation",
      "relevance_score": 0.82
    }
  ],
  "risk_summary": "Current risk band: NORMAL. No emergency rules triggered.",
  "triggered_rules": [],
  "red_flag": false,
  "confidence": 0.85,
  "modality": "text",
  "source_language": "hi"
}
```

**Processing Pipeline** (internal):
1. Audio → transcribe (STT) / Image → extract (OCR) / Text → direct
2. Detect language → translate to English (pivot)
3. Build patient context (vitals, conditions, history)
4. RAG retrieve (5 guidelines + 3 patient memory chunks)
5. Risk evaluate (16 rules against patient + symptoms)
6. Compose prompt (system + context + guidelines + risk + query)
7. Generate response (LLM, temp=0.3)
8. Translate back to source language
9. Persist encounter + update patient memory

---

## Document Upload

### `POST /api/documents/upload`

Upload and process an EHR document (PDF or image). Automatically extracts clinical values, detects abnormalities, and creates an observation record.

**Request** (`multipart/form-data`):
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `patient_id` | string | Yes | Patient to associate the report with |
| `file` | binary | Yes | PDF or image file (JPEG, PNG) |
| `file_type` | string | No | MIME type (auto-detected if absent) |

**Response** `200 OK`:
```json
{
  "report_id": "uuid",
  "patient_id": "uuid",
  "file_type": "image/jpeg",
  "extraction": {
    "findings": {
      "hemoglobin": 8.5,
      "systolic_bp": 145,
      "diastolic_bp": 92,
      "blood_sugar_fasting": 98.0,
      "urine_protein": "positive",
      "report_type": "Antenatal Report"
    },
    "observations": [
      "Patient has moderate anemia",
      "Blood pressure elevated"
    ],
    "recommendations": [
      "Double IFA supplementation",
      "Monitor BP weekly"
    ],
    "raw_text": "Full OCR text output..."
  },
  "abnormality_flags": [
    "WARNING: Moderate anemia (Hb 8.5 < 10 g/dL)",
    "WARNING: Elevated BP (145/92 > 140/90)",
    "WARNING: Urine protein positive — pre-eclampsia risk"
  ],
  "extractor_confidence": 0.82,
  "observation_created": true,
  "observation_id": "uuid"
}
```

### `GET /api/documents/patient/{patientId}`

List all uploaded documents/reports for a patient.

**Response** `200 OK`:
```json
{
  "reports": [
    {
      "report_id": "uuid",
      "file_type": "image/jpeg",
      "report_date": "2026-03-15",
      "abnormality_flags": ["WARNING: Moderate anemia (Hb 8.5)"],
      "extractor_confidence": 0.82,
      "created_at": "2026-03-15T14:00:00Z"
    }
  ]
}
```

---

## Dashboard

### `GET /api/dashboard/village/{villageName}`

Complete village dashboard snapshot for operational planning.

**Response** `200 OK`:
```json
{
  "village": "Hosahalli",
  "generated_at": "2026-04-05T10:30:00Z",
  "summary": {
    "total_patients": 10,
    "risk_distribution": {
      "NORMAL": 4,
      "ELEVATED": 3,
      "HIGH_RISK": 2,
      "EMERGENCY": 1
    },
    "trimester_distribution": {
      "1st": 2,
      "2nd": 4,
      "3rd": 4
    }
  },
  "todays_visits": [
    {
      "patient_name": "Lakshmi Devi",
      "visit_type": "ANC-4",
      "tests_due": ["Hb", "BP", "FHR"],
      "risk_band": "NORMAL",
      "facility": "Hosahalli PHC"
    }
  ],
  "overdue_visits": [
    {
      "patient_name": "Anita Sharma",
      "visit_type": "ANC-3",
      "days_overdue": 8,
      "tests_due": ["GDM screening"],
      "risk_band": "ELEVATED"
    }
  ],
  "high_risk_queue": [
    {
      "patient_name": "Meena Kumari",
      "risk_band": "EMERGENCY",
      "risk_score": 100.0,
      "triggered_rules": ["R002: Severe Hypertension", "R012: Prior C-Section"],
      "known_conditions": ["prior_preeclampsia", "prior_csection"]
    }
  ],
  "active_alerts": [
    {
      "patient_name": "Meena Kumari",
      "severity": "EMERGENCY",
      "message": "Severe hypertension detected. FRU referral required.",
      "created_at": "2026-04-05T08:00:00Z"
    }
  ],
  "upcoming_deliveries": [
    {
      "patient_name": "Priya Sharma",
      "edd_date": "2026-04-28",
      "gestational_weeks": 37,
      "risk_band": "ELEVATED"
    }
  ],
  "ration_summary": {
    "total_beneficiaries": 10,
    "supplement_counts": {"IFA": 10, "Calcium": 8},
    "avg_calorie_target": 2467
  }
}
```

### `GET /api/dashboard/village/{villageName}/risk-queue`

High-risk patient queue, sorted by severity (EMERGENCY first, then by risk score).

**Response** `200 OK`:
```json
{
  "village": "Hosahalli",
  "high_risk_patients": [
    {
      "patient_id": "uuid",
      "patient_name": "Meena Kumari",
      "risk_band": "EMERGENCY",
      "risk_score": 100.0,
      "triggered_rules": ["R002", "R012"],
      "known_conditions": ["prior_preeclampsia"],
      "gestational_weeks": 34,
      "last_visit_date": "2026-03-20"
    }
  ]
}
```

### `GET /api/dashboard/village/{villageName}/upcoming-deliveries`

Patients with EDD in the next 30 days.

**Response** `200 OK`:
```json
{
  "village": "Hosahalli",
  "upcoming_deliveries": [
    {
      "patient_id": "uuid",
      "patient_name": "Priya Sharma",
      "edd_date": "2026-04-28",
      "gestational_weeks": 37,
      "risk_band": "ELEVATED",
      "delivery_plan": "Hospital delivery recommended (anemia)"
    }
  ]
}
```

---

## Error Handling

All errors follow a consistent format:

```json
{
  "detail": "Human-readable error message",
  "error_code": "PATIENT_NOT_FOUND",
  "timestamp": "2026-04-05T10:30:00Z"
}
```

| HTTP Status | When |
|------------|------|
| `200` | Successful request |
| `201` | Resource created (POST) |
| `400` | Invalid input (missing required fields, invalid data) |
| `404` | Resource not found (patient, schedule, report) |
| `422` | Validation error (Pydantic) |
| `500` | Internal server error |

---

## Service-Level Contracts

For direct Python service integration (bypassing REST), all services expose typed interfaces:

### PatientService
```python
create_patient(patient: Patient) -> Patient
get_patient(patient_id: str) -> Optional[Patient]
list_patients(village: Optional[str] = None) -> List[PatientSummary]
update_patient(patient_id: str, updates: dict) -> Patient
search_patients(query: str) -> List[PatientSummary]
update_risk(patient_id: str, risk_band: RiskBand, risk_score: float)
count_by_risk(village: Optional[str] = None) -> dict
```

### RiskService
```python
evaluate_patient(patient: Patient, latest_obs: Optional[Observation] = None,
                 reported_symptoms: Optional[List[str]] = None) -> RiskEvaluation
evaluate_all_patients() -> List[RiskEvaluation]
get_latest_observation(patient_id: str) -> Optional[Observation]
```

### ScheduleService
```python
generate_schedule(patient: Patient) -> List[ScheduleEntry]
get_patient_schedule(patient_id: str) -> List[ScheduleEntry]
get_due_today(village: Optional[str] = None) -> List[ScheduleEntry]
get_overdue(village: Optional[str] = None) -> List[ScheduleEntry]
get_daily_task_list(village: str, target_date: Optional[date] = None) -> DailyTaskList
mark_completed(schedule_id: str)
get_next_pmsma_date(from_date: Optional[date] = None) -> date
```

### RationService
```python
generate_recommendation(patient: Patient, latest_obs: Optional[Observation] = None) -> RationRecommendation
aggregate_village_rations(village: str) -> VillageRationSummary
```

### ConversationService
```python
process_message(patient_id: str, text: Optional[str] = None,
                audio_bytes: Optional[bytes] = None,
                image_bytes: Optional[bytes] = None,
                source_language: str = "hi",
                mode: str = "general") -> Dict
get_patient_history(patient_id: str, limit: int = 10) -> List[Dict]
```

### DocumentService
```python
process_upload(patient_id: str, file_bytes: bytes, file_type: str,
               file_name: str = "") -> Dict
get_patient_reports(patient_id: str) -> List
```

### DashboardService
```python
get_village_dashboard(village: str) -> Dict
create_snapshot(village: str, snapshot_type: str = "daily")
```

### RetrievalService
```python
retrieve(query: str, patient_id: Optional[str] = None,
         top_k_guidelines: int = 5, top_k_patient: int = 3) -> Dict
add_guideline_chunk(chunk_id: str, text: str, source: str, category: str)
add_patient_memory(patient_id: str, text: str, chunk_type: str, source_date: str)
initialize()
```

### AuditService
```python
log(action: str, entity_type: str, entity_id: str, actor: str, details: dict)
log_model_call(provider: str, model_name: str, action: str,
               patient_id: str, input_tokens: int, output_tokens: int, latency_ms: float)
get_recent_logs(limit: int = 50, entity_type: Optional[str] = None) -> List
```
