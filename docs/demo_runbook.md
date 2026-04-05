# Demo Runbook — ASHA Sahayak

> Step-by-step guide for demonstrating ASHA Sahayak at the BharatBricks Hackathon. Includes 6 interactive scenarios with talking points for judges.

---

## Table of Contents

- [Pre-Demo Checklist](#pre-demo-checklist)
- [Quick Setup](#quick-setup)
- [Demo Scenarios](#demo-scenarios)
  - [Scenario 1: New Registration + Normal Pregnancy](#scenario-1-new-registration--normal-pregnancy)
  - [Scenario 2: Moderate Anemia Detection + Ration Guidance](#scenario-2-moderate-anemia-detection--ration-guidance)
  - [Scenario 3: High-Risk Emergency Detection](#scenario-3-high-risk-emergency-detection)
  - [Scenario 4: Multilingual AI Chat](#scenario-4-multilingual-ai-chat)
  - [Scenario 5: Report Upload + OCR](#scenario-5-report-upload--ocr)
  - [Scenario 6: Village Dashboard](#scenario-6-village-dashboard)
- [Talking Points for Judges](#talking-points-for-judges)
- [Key Differentiators](#key-differentiators)
- [Common Questions & Answers](#common-questions--answers)

---

## Pre-Demo Checklist

- [ ] App is running and accessible (local or Databricks URL)
- [ ] 30 patients visible on the Patients page
- [ ] At least one High-Risk/Emergency patient exists (Meena Kumari)
- [ ] At least one anemic patient exists (Priya Sharma)
- [ ] Browser is open with good internet (for Sarvam AI calls, if enabled)
- [ ] Sample report image available for upload demo (optional)
- [ ] Speaker/audio setup if showing voice input demo

---

## Quick Setup

### Option A: Local Demo (Recommended for reliability)
```bash
cd Asha_Sahayak
pip install -r requirements.txt
python -m tools.generate_all   # Only needed first time
python app/main.py
```
Open **http://localhost:8080** in browser. Demo data auto-seeds on first run.

### Option B: Databricks Deployment
1. Import repo into Databricks workspace
2. Run `notebooks/setup_demo_data.py` on a cluster
3. Deploy as Databricks App → get public URL

### Verify Setup
- Home page shows total patients (should be 30)
- Risk distribution cards show NORMAL, ELEVATED, HIGH_RISK, EMERGENCY counts
- Active alerts section populated

---

## Demo Scenarios

### Scenario 1: New Registration + Normal Pregnancy

**Goal**: Show patient registration with auto-computed pregnancy fields and cascade effects.

**Duration**: ~2 minutes

**Steps**:
1. Go to **Patients** tab
2. Click **Register New Patient**
3. Fill in the form:
   - Name: **Geeta Kumari**
   - Age: **24**
   - Village: **Hosahalli**
   - Language: **Kannada (kn)**
   - LMP Date: **2026-02-01**
   - Gravida: **1**, Parity: **0**
   - Blood Group: **O+**
4. Click **Register Patient**
5. **Point out auto-computed fields**:
   - Gestational weeks: shows exact weeks since Feb 1
   - Trimester: shows "1st" (if < 14 weeks)
   - EDD: shows Nov 8, 2026 (LMP + 280 days, Naegele's rule)
6. **Point out cascade effects**:
   - Risk band: **NORMAL** (green badge)
   - ANC schedule auto-generated (7 visits)
   - Ration plan auto-generated (1st trimester baseline)

**Key talking points**:
- "One form → five automatic computations"
- "Naegele's rule used by OB-GYNs worldwide, now available for ASHA workers"
- "Schedule, risk, and ration all generated in one registration"

---

### Scenario 2: Moderate Anemia Detection + Ration Guidance

**Goal**: Show risk detection from clinical data and personalized nutrition guidance.

**Duration**: ~3 minutes

**Steps**:
1. Go to **Patients** tab
2. Find and select **Priya Sharma** (demo patient with Hb 8.5)
3. Navigate to her **Patient Detail** page
4. Show **Clinical Data**:
   - Hemoglobin: **8.5 g/dL** (highlighted yellow — below normal 11 g/dL)
   - Other vitals visible
5. Show **Risk Assessment**:
   - Risk Band: **ELEVATED** (yellow badge)
   - Triggered Rule: **R009 — Moderate Anemia (Hb 7-10 g/dL)**
   - Suggested action: "Double IFA supplementation, nutrition counseling"
6. Show **Ration Recommendation**:
   - Notice supplements: **Double IFA + Vitamin C** (not just standard IFA)
   - Iron-rich foods emphasized: beetroot, spinach, jaggery, amla
   - Calorie target adjusted: **+500 calories** vs baseline
   - Rationale text explains WHY each adjustment was made
   - Rule basis shows: `POSHAN_2.0_T2_BASELINE + MCP_CARD_ANEMIA_MODERATE`

**Key talking points**:
- "Every recommendation comes with a rule basis and source reference"
- "Not just a diet plan — it's scheme-aligned with POSHAN 2.0 and MCP Card"
- "The ASHA worker can explain to the patient AND the medical officer exactly why this diet was recommended"
- "If the Anganwadi asks 'Why does this patient need double IFA?' — the answer is right here"

---

### Scenario 3: High-Risk Emergency Detection

**Goal**: Show how multiple risk factors compound and trigger emergency escalation.

**Duration**: ~3 minutes

**Steps**:
1. Go to **Patients** tab
2. Find and select **Meena Kumari** (demo patient with BP 152/98, prior preeclampsia)
3. Navigate to her **Patient Detail** page
4. Show **Risk Assessment**:
   - Risk Band: **HIGH_RISK** or **EMERGENCY** (red badge)
   - Risk Score: **high (60-100)**
   - Multiple triggered rules:
     - **R007**: Pregnancy-Induced Hypertension (BP > 140/90)
     - **R012**: Prior C-Section
     - **R015**: Urine Protein Elevated (pre-eclampsia risk)
5. Show **Emergency Alert** banner at the top:
   - Clear escalation recommendation: "Refer to FRU immediately"
   - Reason codes with explanations
6. Show **ANC Schedule**:
   - Notice **extra biweekly visits** added for high-risk monitoring
   - Visits scheduled at higher-level facility (CHC/DH instead of PHC)

**Key talking points**:
- "Three risk rules triggered simultaneously — the system compounds them"
- "Deterministic rules from India's own MCP Card — not a black-box ML model"
- "ASHA worker gets clear, actionable next steps: where to refer, how urgently"
- "Every rule trace is auditable — if a medical officer asks 'why did you refer?', the evidence is here"

---

### Scenario 4: Multilingual AI Chat

**Goal**: Show the conversational AI with patient-aware RAG and multilingual support.

**Duration**: ~3 minutes

**Steps**:
1. Go to **AI Assistant** tab
2. Select any patient (e.g., Lakshmi Devi)
3. Set language to **Hindi (hi)** from the dropdown
4. Type: **"मुझे सिर दर्द हो रहा है"** (I am having a headache)
5. Wait for response
6. **Point out in the response**:
   - Original text preserved (Hindi)
   - Translated query shown (English)
   - AI response is contextual:
     - References the patient's current BP and gestational week
     - Mentions danger signs to check (MCP Card: headache + vision = pre-eclampsia risk)
     - Recommends measuring BP
   - Translated response back to Hindi
7. **Expand the Evidence & Context drawer** (if available):
   - Show **retrieved guideline chunks** — specific passages from danger_signs.md
   - Show **patient context** — recent vitals, conditions
   - Show **triggered rules** — any risk rules that fired

**Key talking points**:
- "The ASHA worker asked in Hindi — the system understood, processed in English, responded in Hindi"
- "The response isn't generic — it knows this patient is 28 weeks with normal BP"
- "Every response is grounded in government guidelines — see the evidence drawer"
- "Supports 10+ Indian languages: Hindi, Kannada, Telugu, Tamil, Marathi, Bengali, Gujarati, Malayalam, Punjabi, Odia"
- "Powered by Sarvam AI — India's own multilingual LLM"

---

### Scenario 5: Report Upload + OCR

**Goal**: Show document ingestion with automatic extraction and abnormality detection.

**Duration**: ~2 minutes

**Steps**:
1. Go to **Patient Detail** tab for any patient
2. Find the **Upload Report** section
3. Upload a sample report image/PDF (or use one of the pre-existing sample EHRs)
4. Click **Process Report**
5. Show the extraction results:
   - **Extracted findings**: Hb value, BP readings, blood sugar, urine protein
   - **Abnormality flags**: Yellow/red markers for out-of-range values
   - **Confidence score**: OCR extraction confidence
6. Show that an **Observation record was auto-created** with the extracted values
7. Show that patient's **risk was re-evaluated** with the new data

**Key talking points**:
- "ASHA worker takes a photo of a lab report → system reads and understands it"
- "No manual data entry — reduces errors and saves time"
- "Abnormalities are auto-detected against MCP Card thresholds"
- "Uses pytesseract (offline OCR) or Sarvam Vision (cloud OCR for 23 languages)"

---

### Scenario 6: Village Dashboard

**Goal**: Show the operational planning view that makes an ASHA worker's day productive.

**Duration**: ~2 minutes

**Steps**:
1. Go to **Dashboard** tab
2. Enter **"Hosahalli"** as the village name and click **Load Dashboard**
3. Walk through each section:
   - **Village Summary**: Total patients, risk distribution pie/bar
   - **High-Risk Queue**: EMERGENCY patients at the top, sorted by risk score
   - **Today's Visits**: Who to visit today, what tests to carry
   - **Overdue Visits**: Missed visits with days overdue
   - **Upcoming Deliveries**: Patients due in next 30 days
   - **Ration Summary**: Total beneficiaries, supplement counts, average calorie targets

**Key talking points**:
- "One screen = entire village's maternal health status"
- "ASHA worker starts her day here: who's the most urgent? Who do I visit today?"
- "Ration summary helps Anganwadi plan weekly THR distribution"
- "Overdue visits ensure no one falls through the cracks"
- "Upcoming deliveries help plan hospital referrals in advance"

---

## Talking Points for Judges

### 1. Databricks Usage
- **Delta Lake**: 5 schemas, 20+ tables under Unity Catalog for ACID transactions
- **Vector Search**: RAG retrieval for guidelines + patient memory (FAISS fallback locally)
- **Databricks Jobs**: Daily risk refresh, weekly ration aggregation, guideline ingestion
- **Databricks Apps**: One-click deployment with environment variable management
- **Unity Catalog**: RBAC-ready data governance for patient data

### 2. AI Centrality
- **RAG Retrieval**: Dual-index (guidelines + patient memory) for grounded responses
- **Multilingual Translation**: Sarvam Translate for 22 Indian languages
- **Risk Reasoning**: 16 deterministic rules + LLM-generated explanations
- **OCR/Vision**: Automatic extraction from lab reports in any language
- **Speech-to-Text**: Voice input for low-literacy ASHA workers

### 3. India Focus
- **Sarvam AI**: India-built LLM (sarvam-m 24B) for best Indic language performance
- **PMSMA**: 9th-of-month scheduling aligned with government scheme
- **MCP Card 2018**: All 16 risk rules sourced from India's maternal health card
- **POSHAN 2.0 / Saksham**: Nutrition norms from Ministry of Women & Child Development
- **JSSK / JSY / PMMVY**: Scheme awareness in AI responses

### 4. Safety & Explainability
- **Deterministic rules first**: No black-box ML for emergency detection
- **Evidence drawer**: Every response shows retrieved sources and triggered rules
- **Rule basis**: Every ration recommendation cites specific guidelines
- **Audit trail**: Complete log of all actions, model calls, and clinical decisions
- **Human-in-the-loop**: System recommends, ASHA worker decides

### 5. Technical Quality
- **53 automated tests** covering all 16 risk rules, scheduling, ration engine, RAG, dashboard
- **Provider abstraction**: Works offline (mock) or with real APIs — same codebase
- **Type safety**: Pydantic models for all data structures
- **Clean architecture**: 4 layers (Presentation → Service → Provider → Data)

---

## Key Differentiators

| What Others Build | What ASHA Sahayak Does Differently |
|-------------------|------------------------------------|
| Generic health chatbots | Patient-aware responses grounded in individual health history |
| English-only interfaces | 10+ Indian languages with voice input support |
| Black-box ML risk scores | 16 deterministic rules from India's own MCP Card, fully auditable |
| General nutrition advice | Scheme-aligned rations (POSHAN 2.0) with condition adjustments |
| Standalone features | Integrated workflow: registration → risk → schedule → ration → chat → dashboard |
| Cloud-only solutions | Works fully offline with SQLite + FAISS + mock providers |

---

## Common Questions & Answers

**Q: What happens if there's no internet?**
A: The app works fully offline. SQLite replaces Delta Lake, FAISS replaces Vector Search, and mock providers simulate all AI features.

**Q: How accurate is the OCR?**
A: pytesseract handles clear printed reports well. For handwritten or low-quality scans, Sarvam Vision (cloud) provides better accuracy with table extraction support in 23 languages.

**Q: Is the risk engine ML-based?**
A: No — by design. The 16 risk rules are deterministic, sourced from India's MCP Card and Safe Motherhood Booklet. This ensures predictability and auditability. ML scoring can be added as a complementary signal in the future.

**Q: How many languages are supported?**
A: 10+ Indian languages for the UI and chat (Hindi, Kannada, Telugu, Tamil, Marathi, Bengali, Gujarati, Malayalam, Punjabi, Odia, English). Sarvam Translate supports all 22 official Indian languages.

**Q: How does this integrate with existing government systems?**
A: Currently standalone. Future integration points: ABDM Health ID, eJanani portal, RCH portal. The data model is designed to be compatible with these systems.

**Q: What about data privacy?**
A: Patient consent is tracked (`consent_status` field). All audit logging excludes PII. Databricks Unity Catalog provides RBAC for production access control. No patient data is sent to external APIs — only anonymized queries.

**Q: How does the ration engine work?**
A: Trimester-specific baselines from POSHAN 2.0 (2200/2500/2700 cal) with condition-specific adjustments (anemia → +500 cal, double IFA; GDM → -300 cal, low GI diet). Every recommendation includes `rule_basis` and `source_refs` for auditability.
