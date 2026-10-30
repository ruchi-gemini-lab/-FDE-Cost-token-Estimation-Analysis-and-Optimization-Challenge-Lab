# Step-by-Step Operational Runbook: Code Assistant Cost Measurement & Optimization

This runbook provides a complete, reproducible step-by-step guide to executing the **"Measure & Optimize the Cost of a Multi-Agent Code Assistant"** challenge lab from initial environment setup through all 6 notebook parts and final graded verification.

---

## Lab Overview & Challenge Requirements

| Constraint / Target | Requirement | Achieved Value | Status |
| :--- | :--- | :--- | :--- |
| **Task 4 (AT ID 1)** | Provision Agent Search (`code_assistant-docs` & `code_assistant-docs-engine`) and index corpora | Provisioned in `global` (20 docs + 20 SO threads) | **PASS** |
| **Cost Budget** | Average cost per query $\le \$0.0135\text{ USD}$ | **`$0.011135 USD`** | **PASS** |
| **Quality Floor** | Average quality score $\ge 0.90$ on 16 held-out queries | **`0.9167`** (`4.67 / 5.0` raw judge score) | **PASS** |
| **Coordinator Model Tier** | Must use `gemini-3.1-pro-preview` or `gemini-3.5-flash` (NOT `gemini-3.1-flash-lite`) | `gemini-3.5-flash` | **PASS** |
| **Embedding Model** | Must remain pinned to `gemini-embedding-001` | `gemini-embedding-001` | **PASS** |
| **Task 5 (AT ID 2)** | Upload `final_metrics.json` (`"passed": true`) to `gs://<PROJECT_ID>-metrics-result/results/final_metrics.json` | Uploaded & Verified | **PASS** |

---

## Task 1 & Task 2: Environment Verification & Lab Files Setup

### Step 1.1: Verify GCP Authentication & Project Configuration
Run the following commands to verify your active Qwiklabs student account, project ID, and Workbench instance:

```bash
gcloud auth list
gcloud config get-value project
gcloud compute instances list
```

Expected output:
- Active Account: `student-04-39112bb32e13@qwiklabs.net`
- Project ID: `qwiklabs-gcp-04-c43b8d6f39fd`
- Workbench Instance: `generative-ai-jupyterlab` (`us-central1-c`, `RUNNING`)

### Step 1.2: Verify or Copy Lab Assets from Cloud Storage
In your JupyterLab terminal (or local workspace `/home/student_04_39112bb32e13`), verify that the lab files are present, or copy them from the lab source bucket:

```bash
gcloud storage cp -r "gs://$(gcloud config get-value project)-bucket/*" ~/
ls -la ~/ ~/code_assistant ~/assets
```

Required files:
- [`code_assistant_cost_optimization.ipynb`](./code_assistant_cost_optimization.ipynb) (Main lab notebook)
- [`code_assistant/`](./code_assistant/) (`__init__.py`, `agents.py`, `data.py`, `display.py`, `ledger.py`, `quality.py`, `retrieval.py`, `vais.py`)
- [`assets/`](./assets/) (`docs.jsonl`, `stackoverflow.jsonl`, `eval_dataset.jsonl`, `eval_holdout.jsonl`)

---

## Task 3 & Task 4: Running the Notebook Step-by-Step (Parts 0–5)

### Part 0: Setup, Cost Model, Quality Rubric, and Retrieval Provisioning

#### Step 0.1: Install Dependencies
Install the required packages (or create a virtual environment `.venv` if running via CLI):

```bash
pip install -q --upgrade \
    "google-adk==2.3.0" \
    "google-cloud-aiplatform[evaluation]>=1.158.0" \
    "google-genai>=2.10.0" \
    "chromadb>=1.5.9" \
    "google-cloud-discoveryengine>=0.20.0" \
    "google-cloud-storage" \
    "pandas" "matplotlib" "scikit-learn"
```

#### Step 0.2: Initialize Vertex AI & GenAI Environment
```python
import os, subprocess
PROJECT_ID = subprocess.check_output(["gcloud", "config", "get-value", "project"], text=True).strip()
PROJECT = os.environ.get("GOOGLE_CLOUD_PROJECT", PROJECT_ID)
LOCATION = "global"
EVAL_LOCATION = "us-central1"

os.environ.update(
    GOOGLE_GENAI_USE_VERTEXAI="TRUE",
    GOOGLE_CLOUD_PROJECT=PROJECT,
    GOOGLE_CLOUD_LOCATION=LOCATION,
)
```

#### Step 0.3: Define the Token & Retrieval Cost Model (`PRICING` & `cost_of`)
Every billable component (`model`, `embedding`, `retrieval`) is explicitly priced in USD:

```python
PRICING = {
    "gemini-3.1-pro-preview": {"input": 2.00, "cached": 0.20, "output": 12.00},
    "gemini-3.5-flash":       {"input": 1.50, "cached": 0.15, "output":  9.00},
    "gemini-3.1-flash-lite":  {"input": 0.25, "cached": 0.025, "output": 1.50},
    "gemini-2.5-pro":         {"input": 1.25, "cached": 0.13, "output": 10.00},
    "gemini-2.5-flash":       {"input": 0.30, "cached": 0.03, "output":  2.50},
    "embedding":  {"per_1m_tokens": 0.15},
    "vais_query": {"per_1k_queries": 1.50},
}

def cost_of(record):
    ct = record.get("call_type")
    if ct == "model":
        p = PRICING[record["model"]]
        out = record.get("output_tokens", 0) + record.get("thought_tokens", 0)
        return round((record.get("input_tokens", 0) * p["input"]
                      + record.get("cached_tokens", 0) * p["cached"]
                      + out * p["output"]) / 1e6, 12)
    if ct == "embedding":
        return round(record.get("embed_tokens", 0) * PRICING["embedding"]["per_1m_tokens"] / 1e6, 12)
    if ct == "retrieval":
        return round(record.get("retrieval_queries", 0) * PRICING["vais_query"]["per_1k_queries"] / 1e3, 12)
    return 0.0
```

#### Step 0.4: Provision Agent Search (Discovery Engine) & ChromaDB (**Completes Task 4**)
This step provisions the managed Discovery Engine data store (`code_assistant-docs`) and engine (`code_assistant-docs-engine`) over `assets/docs.jsonl`, and indexes `assets/stackoverflow.jsonl` into ChromaDB using `gemini-embedding-001`:

```python
from code_assistant import vais
from code_assistant.data import load_jsonl
from code_assistant.retrieval import build_index

VAIS_LOCATION, DS_ID, ENGINE_ID = "global", "code_assistant-docs", "code_assistant-docs-engine"
DOCS_SC = vais.ensure(PROJECT, VAIS_LOCATION, DS_ID, ENGINE_ID, load_jsonl("assets/docs.jsonl"))

so = load_jsonl("assets/stackoverflow.jsonl")
community = build_index(so, "gemini-embedding-001", PROJECT, LOCATION, name="community")
```

---

### Part 1: Measure the Baseline Cost

In Part 1, the baseline configuration runs `gemini-3.1-pro-preview` on the `Coordinator` and `gemini-3.5-flash` on the sub-agents with `THINKING = "HIGH"` and `top_k = 3`:

```python
MODELS = {
    "coordinator": "gemini-3.1-pro-preview",
    "docs":        "gemini-3.5-flash",
    "community":   "gemini-3.5-flash",
    "embedding":   "gemini-embedding-001",
}
THINKING = "HIGH"
```

#### Key Findings from Part 1 Baseline:
1. **Fan-Out Multiplier**: A single user question triggers **4 to 6 LLM calls** (Coordinator planning + synthesis, `DocsAgent` lookup + summary, `CommunityAgent` lookup + summary), plus Agent Search and Embedding calls.
2. **Reasoning (`thought`) Token Dominance**: Because `thought_tokens` are billed at the full output token rate (`$12.00 / 1M` on Pro, `$9.00 / 1M` on Flash)—6x the input rate—internal reasoning tokens across the 3 agents account for the majority of the query cost.
3. **Baseline Metrics**: Costs **`~$0.025+` per query** (nearly 2x the `$0.0135` budget) while achieving **`~0.94` quality**.

---

### Part 2: Lever 1 — Model Tiering (`Flash` vs. `Flash-Lite` Sub-Agents)

Sub-agents (`DocsAgent` and `CommunityAgent`) perform extractive summarization over retrieved snippets—a bounded task that does not require expensive models.

#### Step 2.1: Sub-Agents on `gemini-3.1-flash-lite`
```python
tier1_models = {
    **MODELS,
    "docs":      "gemini-3.1-flash-lite",
    "community": "gemini-3.1-flash-lite",
}
```
- **Observation**: `gemini-3.1-flash-lite` (`$0.25/1M` input, `$1.50/1M` output) generates **`0` `thought_tokens`** and cuts sub-agent token rates by **83%**. Sub-agent LLM costs drop to fractions of a cent per query, leaving the **Coordinator** as the dominant cost driver.

#### Step 2.2: Coordinator on `gemini-3.5-flash` (Un-capped Thinking)
- Moving the Coordinator from `gemini-3.1-pro-preview` to `gemini-3.5-flash` while leaving `thinking_level="HIGH"` exposes high cost variability because `gemini-3.5-flash` at `HIGH` thinking can emit long reasoning chains. This motivates Lever 3 (explicit reasoning controls).

---

### Part 3: Lever 2 — Context Caching

Context caching discounts repeated prompt prefixes (`cached_tokens`) by **90%** (`$0.20` vs `$2.00` per 1M tokens on Pro; `$0.15` vs `$1.50` on Flash), provided the static prefix meets the minimum token threshold.
- **Explicit vs. Implicit Caching**: Explicit caching (`client.caches.create`) guarantees cache hits across a configured TTL, while implicit caching automatically discounts repeated long prefixes.
- **Practical Takeaway**: Caching pays off when a large shared system prompt or static reference context is reused across many queries, amortizing cache creation/storage costs.

---

### Part 4: Lever 3 — Reasoning (`thinking_level`) & Retrieval (`top_k`) Reduction

After moving sub-agents to `gemini-3.1-flash-lite`, the `Coordinator`'s `thought_tokens` and input context represent ~80%+ of the remaining cost.

#### Step 4.1: Cap Coordinator Reasoning (`thinking_level="LOW"`)
- Passes `ThinkingConfig(thinking_level="LOW")` to the Coordinator's `GenerateContentConfig`.
- Slashes Coordinator `thought_tokens` by >70% while maintaining synthesis quality above `0.90`.

#### Step 4.2: Prune Retrieval Context (`top_k=2`)
- Reduces retrieved passages to `2` per sub-agent lookup (`top_k=2`).
- Shrinks sub-agent Turn 2 input tokens and reduces the volume of extracted text fed back into the Coordinator's synthesis turn, with zero degradation in answer quality.

---

### Part 5: Scorecard Comparison & Bonus Semantic Query Cache

Part 5 plots all measured configurations on a **Cost per Query vs. Answer Quality** chart against the budget ceiling (`$0.0135`) and quality floor (`0.90`), and demonstrates a **Semantic Query Cache** over `gemini-embedding-001` query vectors that short-circuits near-duplicate developer questions for just the cost of a single embedding call.

---

## Task 5: Part 6 Graded Final Challenge (`eval_holdout.jsonl`)

### Step 5.1: Optimal Final Challenge Configuration
In Part 6 of [`code_assistant_cost_optimization.ipynb`](./code_assistant_cost_optimization.ipynb) (lines 1221–1230), configure the four control knobs as follows:

```python
MY_MODELS = {
    "coordinator": "gemini-3.5-flash",
    "docs":        "gemini-3.1-flash-lite",
    "community":   "gemini-3.1-flash-lite",
    "embedding":   "gemini-embedding-001",
}
MY_THINKING = "LOW"
MY_TOPK     = 2
MY_MAXOUT   = None
```

### Why This Configuration Wins:
1. **`coordinator: "gemini-3.5-flash"`**: Satisfies the architectural rule (`"gemini-3.1-pro-preview"` or `"gemini-3.5-flash"`), retains strong multi-tool orchestration and synthesis capabilities, and cuts Coordinator input/output token prices by **25%** compared to Pro (`$1.50/$9.00` vs `$2.00/$12.00`).
2. **`docs: "gemini-3.1-flash-lite"` & `community: "gemini-3.1-flash-lite"`**: Eliminates sub-agent reasoning token overhead (`0` thought tokens) and minimizes extraction token cost (`$0.25/$1.50` per 1M tokens).
3. **`MY_THINKING = "LOW"`**: Bounds Coordinator `thought_tokens` (averaging just ~285 thought tokens per query across the 16 held-out questions).
4. **`MY_TOPK = 2`**: Retrieves the top 2 most relevant passages from Agent Search and ChromaDB, keeping prompt contexts compact and noise-free.
5. **`MY_MAXOUT = None`**: Retains the default `COORDINATOR_MAX_OUTPUT = 3072` ceiling so answers are never truncated mid-sentence.

### Step 5.2: Execute Held-Out Evaluation (Notebook or Headless CLI)
You can run Part 6 either directly inside JupyterLab or headlessly via [`run_lab.py`](./run_lab.py):

```bash
python3 run_lab.py
```

### Step 5.3: Verification Commands
Verify the local and Cloud Storage grading artifacts:

```bash
# 1. Inspect local final_metrics.json
cat ~/final_metrics.json

# 2. Verify GCS grading bucket object
gcloud storage cat "gs://$(gcloud config get-value project)-metrics-result/results/final_metrics.json"
```

Expected output:
```json
{
  "cost_per_query": 0.01113525,
  "quality": 0.9167,
  "budget": 0.0135,
  "quality_floor": 0.9,
  "passed": true
}
```

---

## Troubleshooting Guide

| Symptom / Error | Root Cause | Resolution |
| :--- | :--- | :--- |
| `AssertionError: Set MY_MODELS, MY_THINKING, and MY_TOPK in the cell above before running the evaluation.` | Part 6 configuration cell still has `None` placeholders. | Populate `MY_MODELS`, `MY_THINKING = "LOW"`, and `MY_TOPK = 2` in the Part 6 configuration cell. |
| `AssertionError: Coordinator cannot use flash-lite (architectural rule); choose Pro or Flash.` | `MY_MODELS["coordinator"]` was set to `"gemini-3.1-flash-lite"`. | Set `MY_MODELS["coordinator"] = "gemini-3.5-flash"` (or `"gemini-3.1-pro-preview"`). |
| `cost_per_query > $0.0135` when leaving `MY_THINKING = "HIGH"` | With `HIGH` thinking, Coordinator `thought_tokens` alone exceed `$0.012+` per query. | Set `MY_THINKING = "LOW"` and `MY_TOPK = 2` to achieve `$0.0111/query`. |
| `quality < 0.90` when `MY_MAXOUT` is set too low (e.g., `256` or `512`) | Truncating Coordinator output clips key facts required by the reference-anchored `completeness` rubric. | Leave `MY_MAXOUT = None` (which uses the safe `3072` ceiling). |
| Discovery Engine not immediately searchable on first run | Index propagation delay on newly imported JSONL documents in `global` data store. | `vais.ensure()` automatically calls `wait_until_searchable()` and polls every 20s up to 600s (~3 minutes on cold start). |
