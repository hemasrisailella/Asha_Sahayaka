# Deployment Guide — ASHA Sahayak

> Step-by-step guide for deploying ASHA Sahayak locally, on Databricks, or in any Python environment.

---

## Table of Contents

- [Deployment Options](#deployment-options)
- [Local Development Setup](#local-development-setup)
- [Databricks Deployment — Method A: GitHub Repo](#method-a-github-repo--databricks-recommended)
- [Databricks Deployment — Method B: CLI Bundle](#method-b-databricks-cli-bundle-deploy)
- [Environment Variables](#environment-variables-reference)
- [Verify Deployment](#verify-deployment)
- [Production Checklist](#production-checklist)
- [What Gets Uploaded](#what-gets-uploaded)
- [Provider Configuration](#provider-configuration)
- [Databricks Free Edition Limits](#databricks-free-edition-limits)
- [Troubleshooting](#troubleshooting)

---

## Deployment Options

| Method | Best For | Internet Required | API Keys Required |
|--------|----------|-------------------|-------------------|
| **Local (SQLite + Mock)** | Development, demo, testing | No | No |
| **Local (SQLite + Sarvam)** | Full-featured demo | Yes (for Sarvam API) | Sarvam AI |
| **Databricks (GitHub)** | Production, team, CI/CD | Yes | Sarvam AI (optional) |
| **Databricks (CLI Bundle)** | Quick deploy from terminal | Yes | Sarvam AI (optional) |

---

## Local Development Setup

### Prerequisites

| Requirement | Version | Install |
|-------------|---------|---------|
| Python | 3.10+ | `brew install python` / `apt install python3` |
| pip | Latest | Included with Python |
| Tesseract OCR | 4.0+ (optional) | `apt install tesseract-ocr` / `brew install tesseract` |

### Step 1: Clone & Install

```bash
git clone https://github.com/hemasrisailella/Asha_Sahayak.git
cd Asha_Sahayak

# Create virtual environment
python -m venv .venv
source .venv/bin/activate   # Linux/Mac
# .venv\Scripts\activate    # Windows

# Install dependencies
pip install -r requirements.txt
```

### Step 2: Generate Data & Launch

```bash
# Generate synthetic data (30 patients, 82 observations, 12 facilities)
python -m tools.generate_all

# Launch the app
python app/main.py
```

Open **http://localhost:8080**. The app auto-seeds from synthetic data on first run.

### Step 3: Enable AI Providers (Optional)

```bash
cp .env.example .env
# Edit .env with your Sarvam API key:
#   SARVAM_API_KEY=your_key_here
#   REASONING_PROVIDER=sarvam
#   TRANSLATION_PROVIDER=sarvam
#   SPEECH_PROVIDER=sarvam
```

Without API keys, mock providers simulate all AI features for demo purposes.

### Step 4: Run Tests

```bash
pytest tests/ -v
# Expected: 53 tests passing
```

---

## Databricks Deployment

There are **two ways** to deploy to Databricks:

| Method | Best For | Needs |
|--------|----------|-------|
| **A. GitHub Repo → Databricks** | Teams, CI/CD, recommended | GitHub account |
| **B. Databricks CLI Bundle Deploy** | Quick deploy from local machine | Databricks CLI |

Both methods are described step-by-step below.

---

## Prerequisites

1. **Databricks Workspace** — Free Edition works ([signup](https://www.databricks.com/try-databricks))
2. **Sarvam AI API Key** (optional) — from [dashboard.sarvam.ai](https://dashboard.sarvam.ai/)
3. **Python 3.10+** on your local machine

---

## Method A: GitHub Repo → Databricks (Recommended)

### Step 1: Initialize Git and Push to GitHub

```bash
cd asha-sahayak

# Initialize git
git init
git add .
git commit -m "ASHA Sahayak — production-ready initial commit"

# Create a GitHub repo first at https://github.com/new, then:
git remote add origin https://github.com/YOUR_USERNAME/asha-sahayak.git
git branch -M main
git push -u origin main
```

The `.gitignore` ensures secrets (`.env`) and runtime files (`demo.db`) are excluded.
Synthetic data (`data/synthetic/`) and reference docs ARE included — they're needed on Databricks.

### Step 2: Connect GitHub to Databricks

1. Open your Databricks workspace
2. Navigate to **Workspace → Repos**
3. Click **Add Repo**
4. Paste your GitHub URL: `https://github.com/YOUR_USERNAME/asha-sahayak`
5. Click **Create Repo**

Databricks clones your repo. All files (code, data, configs) are now in the workspace.

### Step 3: Install Dependencies on a Cluster

1. Go to **Compute → Create Cluster** (or use existing)
2. Under **Libraries → Install New**:
   - Source: **PyPI**
   - Package: `gradio pydantic pyyaml httpx faiss-cpu sarvamai python-dotenv Pillow`
3. Click **Install**

Or run in a notebook attached to the cluster:

```python
%pip install gradio pydantic pyyaml httpx faiss-cpu sarvamai python-dotenv Pillow numpy pandas
dbutils.library.restartPython()
```

### Step 4: Seed the Database (First Time)

Open `notebooks/setup_demo_data.py` as a notebook and run it. This:
- Creates the SQLite database with the synthetic data
- Runs risk/schedule/ration engines for all 30 patients
- Ingests reference guidelines into the RAG index

### Step 5: Deploy as Databricks App

1. Go to **Workspace → Apps** (left sidebar)
2. Click **Create App**
3. Fill in:
   - **Name**: `asha-sahayak`
   - **Source**: select the repo path `/Repos/YOUR_EMAIL/asha-sahayak`
4. Under **Environment Variables**, add:

   | Variable | Value |
   |----------|-------|
   | `SARVAM_API_KEY` | your Sarvam API key (or leave blank for mock) |
   | `REASONING_PROVIDER` | `sarvam` (or `mock`) |
   | `TRANSLATION_PROVIDER` | `sarvam` (or `mock`) |
   | `SPEECH_PROVIDER` | `sarvam` (or `mock`) |
   | `VISION_PROVIDER` | `mock` |
   | `EMBEDDING_PROVIDER` | `mock` |
   | `DEMO_MODE` | `true` |

5. Click **Deploy**

The app starts at a URL like: `https://your-workspace.databricks.com/apps/asha-sahayak`

### Step 6: Update After Code Changes

```bash
# On your local machine, after making changes:
git add .
git commit -m "your change description"
git push

# In Databricks: Workspace → Repos → Pull to sync
# Then: Apps → asha-sahayak → Restart
```

---

## Method B: Databricks CLI Bundle Deploy

### Step 1: Install Databricks CLI

```bash
# macOS
brew install databricks/tap/databricks

# Linux / pip
pip install databricks-cli

# Verify
databricks --version
```

### Step 2: Configure Authentication

```bash
# Option 1: Interactive login (opens browser)
databricks configure

# Option 2: Set environment variables
export DATABRICKS_HOST=https://your-workspace.cloud.databricks.com
export DATABRICKS_TOKEN=your_personal_access_token
```

To generate a token:
1. Databricks → **Settings → Developer → Access Tokens**
2. Click **Generate New Token**
3. Copy the token

### Step 3: Validate the Bundle

```bash
cd asha-sahayak
databricks bundle validate
```

You should see: `Successfully validated`

### Step 4: Deploy the Bundle

```bash
databricks bundle deploy
```

This uploads all source code (respecting `.gitignore`) and creates:
- The Databricks App (`asha-sahayak`)
- The daily refresh job
- The weekly summary job

### Step 5: Set App Environment Variables

After the first deploy, set the Sarvam API key:

```bash
# Via Databricks workspace UI:
# Apps → asha-sahayak → Configuration → Add environment variables
```

Or update `databricks.yml` with your key before deploying (not recommended — keeps secrets out of code).

### Step 6: Start the App

```bash
databricks bundle run asha_sahayak
```

Or from the UI: **Apps → asha-sahayak → Start**

---

## Verify Deployment

Once the app is running:

1. Open the app URL shown in **Apps → asha-sahayak**
2. You should see the home page with 30 patients loaded
3. Navigate to **Patients** tab — should show patients from 3 villages
4. Open **AI Assistant** — select a patient and type a query
5. Check **Dashboard** — risk distribution chart and daily schedule

---

## What Gets Uploaded

Files that ARE deployed (tracked by git):

```
✓ app/                  # Gradio UI
✓ services/             # Business logic
✓ providers/            # AI provider adapters
✓ models/               # Pydantic data models
✓ pipelines/            # Batch jobs
✓ tools/                # Data generation scripts
✓ data/synthetic/       # 30 patients, 82 observations, 12 facilities
✓ data/sample_ehr/      # 5 sample EHR text files
✓ data/sample_reference/ # Maternal health guidelines (RAG source)
✓ data/seed/            # Fallback demo data
✓ config/               # App config (no secrets)
✓ sql/                  # Delta Lake DDL
✓ notebooks/            # Setup notebooks
✓ tests/                # Test suite
✓ requirements.txt
✓ databricks.yml
✓ app.yaml              # Databricks App entrypoint
✓ .env.example          # Template (no real keys)
```

Files that are NOT deployed (in `.gitignore`):

```
✗ .env                  # Secrets
✗ data/demo.db          # Runtime database (regenerated)
✗ __pycache__/          # Python cache
✗ venv/                 # Virtual environment
✗ .pytest_cache/        # Test cache
✗ .DS_Store             # macOS artifacts
```

---

## Provider Configuration

| Capability | Provider | Free? | Notes |
|------------|----------|-------|-------|
| Reasoning | Sarvam AI (`sarvam`) | Free tier | Best for Indian languages |
| Translation | Sarvam AI (`sarvam`) | Free tier | 22 Indian languages |
| Speech-to-Text | Sarvam AI (`sarvam`) | Free tier | saaras:v3 model |
| OCR | `mock` (default on Databricks) | Yes | Set `pytesseract` if Tesseract installed |
| Embeddings | `mock` (default on Databricks) | Yes | Set `local` if GPU/large memory available |

Set `REASONING_PROVIDER=mock` etc. to run entirely without any API keys.

---

## Databricks Free Edition Limits

| Resource | Free Limit | Our Usage |
|----------|-----------|-----------|
| Databricks Apps | 1 app | asha-sahayak |
| SQL Warehouse | 1 warehouse | For Delta Lake queries |
| Vector Search | 1 endpoint | Optional (FAISS fallback works) |
| Clusters | Time-limited | For setup notebooks |
| Storage | Community tier | SQLite + synthetic data ~2MB |

The app auto-stops after 24 hours of inactivity on Free Edition. Restart from **Apps → asha-sahayak → Start**.

---

## Troubleshooting

### "App failed to start"
- Check logs: **Apps → asha-sahayak → Logs**
- Common cause: missing dependency — run `%pip install` in a notebook first
- Verify `app.yaml` exists at repo root

### "Import error: No module named services"
- Check `source_code_path` in `databricks.yml` is `.` (root), not `./app`
- The app uses relative imports from the repo root

### "Sarvam API errors"
- Set `REASONING_PROVIDER=mock` etc. to bypass API calls
- Check your API key at [dashboard.sarvam.ai](https://dashboard.sarvam.ai/)

### "No patients showing"
- Delete `data/demo.db` (if exists) and restart the app
- The app auto-seeds from `data/synthetic/` on first run

### "App stopped after 24 hours"
- Free Edition limitation — restart from the Apps page
- Data persists in the database file

---

## Environment Variables Reference

| Variable | Default | Options | Description |
|----------|---------|---------|-------------|
| `SARVAM_API_KEY` | (unset) | API key string | Sarvam AI authentication |
| `REASONING_PROVIDER` | `mock` | `sarvam`, `databricks`, `mock` | LLM provider for chat/reasoning |
| `TRANSLATION_PROVIDER` | `mock` | `sarvam`, `mock` | Text translation provider |
| `SPEECH_PROVIDER` | `mock` | `sarvam`, `mock` | Speech-to-text provider |
| `VISION_PROVIDER` | `mock` | `pytesseract`, `sarvam`, `mock` | OCR/document extraction |
| `EMBEDDING_PROVIDER` | `local` | `local`, `mock` | Text embedding for RAG |
| `EMBEDDING_MODEL` | `intfloat/multilingual-e5-small` | Any sentence-transformers model | Embedding model name |
| `DEMO_MODE` | `false` | `true`, `false` | Use synthetic data |
| `DATABRICKS_HOST` | (unset) | URL | Databricks workspace URL |
| `DATABRICKS_TOKEN` | (unset) | Token | Personal access token |
| `WAREHOUSE_ID` | (unset) | ID string | SQL Warehouse ID |

---

## Production Checklist

Before deploying to production:

- [ ] **Security**: Set strong `SARVAM_API_KEY`, never commit `.env`
- [ ] **Data**: Migrate from synthetic data to real patient data
- [ ] **Database**: Move from SQLite to Delta Lake tables
- [ ] **Vector Search**: Create Databricks Vector Search endpoint (replace FAISS)
- [ ] **Authentication**: Configure Databricks App proxy authentication
- [ ] **HTTPS**: Ensure all traffic is encrypted (default on Databricks)
- [ ] **Consent**: Verify patient consent workflow before storing data
- [ ] **Backup**: Configure Delta Lake time travel / backup schedule
- [ ] **Monitoring**: Set up alerts for EMERGENCY risk evaluations
- [ ] **Jobs**: Schedule daily_refresh and weekly_summary as Databricks Workflows
- [ ] **Testing**: Run `pytest tests/ -v` and verify all 53 tests pass
- [ ] **Audit**: Verify audit_log is capturing all actions
