# End-to-End Workflows: Multi-Agent Execution, Cost Attribution & Optimization

This document details the operational and runtime workflows of the **Code Assistant Agent** system across five core lifecycles:
1. **Infrastructure & Retrieval Corpus Provisioning Workflow**
2. **Runtime Multi-Agent Query Fan-Out & Cost Attribution Sequence**
3. **Automated LLM-as-a-Judge Quality Evaluation Workflow**
4. **Iterative FinOps Cost-Quality Optimization Workflow**
5. **Graded Final Challenge Verification & Artifact Export Workflow**

---

## 1. Infrastructure & Retrieval Corpus Provisioning Workflow

Before any developer query can be answered, Part 0 provisions two retrieval backends:
1. **Managed Agent Search (`code_assistant-docs-engine`)** in `global` for official documentation ([`assets/docs.jsonl`](./assets/docs.jsonl)).
2. **Local ChromaDB Ephemeral Vector Store (`community`)** in memory for community Stack Overflow Q&As ([`assets/stackoverflow.jsonl`](./assets/stackoverflow.jsonl)).

```mermaid
sequenceDiagram
    autonumber
    actor Eng as Platform Engineer
    participant NB as Notebook / run_lab.py
    participant VAIS as code_assistant/vais.py
    participant DE as GCP Discovery Engine API
    participant Ret as code_assistant/retrieval.py
    participant Emb as Vertex AI Embeddings (gemini-embedding-001)
    participant Chroma as ChromaDB EphemeralClient
    participant Ledger as UsageLedger

    Eng->>NB: Execute Part 0 Provisioning
    NB->>VAIS: vais.ensure(PROJECT, "global", "code_assistant-docs", "code_assistant-docs-engine", docs)
    VAIS->>DE: search(probe="KeyError", top_k=1)
    alt Engine not yet provisioned or empty
        VAIS->>DE: create_data_store("code_assistant-docs", GENERIC, NO_CONTENT)
        VAIS->>DE: import_documents(20 inline JSON docs from assets/docs.jsonl)
        VAIS->>DE: create_engine("code_assistant-docs-engine", SEARCH_TIER_STANDARD)
        loop wait_until_searchable (every 20s up to 600s)
            VAIS->>DE: search(probe="KeyError", top_k=3)
        end
    end
    DE-->>NB: Ready: serving_config path (DOCS_SC)

    NB->>Ret: build_index(so, "gemini-embedding-001", PROJECT, LOCATION, name="community")
    Ret->>Emb: models.embed_content(20 SO posts)
    Emb-->>Ret: 20 Embedding Vectors + token_count statistics
    Ret->>Ledger: add(query_id="__index__", agent="index", call_type="embedding", embed_tokens=...)
    Ret->>Chroma: get_or_create_collection("community")
    Ret->>Chroma: col.add(ids, documents, embeddings)
    Chroma-->>NB: Ready: 20 Stack Overflow Q&As indexed
```

---

## 2. Runtime Multi-Agent Query Fan-Out & Cost Attribution Sequence

When `answer_query(coordinator, query, query_id, qid_holder)` runs on a single developer question, a multi-turn conversation unfolds across the `Coordinator`, `DocsAgent`, and `CommunityAgent`. Every LLM turn and retrieval lookup is intercepted and recorded in `UsageLedger`.

```mermaid
sequenceDiagram
    autonumber
    participant Caller as measure() Loop
    participant Ledger as UsageLedger
    participant Runner as ADK InMemoryRunner
    participant Coord as Coordinator (gemini-3.5-flash)
    participant Docs as DocsAgent (gemini-3.1-flash-lite)
    participant VAIS as Agent Search (vais.search)
    participant Comm as CommunityAgent (gemini-3.1-flash-lite)
    participant Chroma as ChromaDB + gemini-embedding-001

    Caller->>Ledger: qid_holder["v"] = query_id
    Caller->>Runner: runner.run(user_id="lab", session_id=query_id, new_message=query)
    Runner->>Coord: Turn 1: Plan sub-agent tool calls

    Note over Coord,Ledger: _model_cb fires for Coordinator Turn 1
    Coord-->>Ledger: add(agent="Coordinator", call_type="model", input_tokens, output_tokens, thought_tokens)

    Coord->>Docs: AgentTool Call: DocsAgent(request)
    Docs->>Docs: Turn 1: Formulate docsagent_lookup(query)
    Docs-->>Ledger: add(agent="DocsAgent", call_type="model", input_tokens, output_tokens, thought_tokens=0)

    Docs->>VAIS: docsagent_lookup(query, top_k=2)
    VAIS-->>Ledger: add(agent="DocsAgent", call_type="retrieval", model="vais_query", retrieval_queries=1)
    VAIS-->>Docs: Top 2 Official Docs Passages

    Docs->>Docs: Turn 2: Summarize retrieved official docs
    Docs-->>Ledger: add(agent="DocsAgent", call_type="model", input_tokens, output_tokens, thought_tokens=0)
    Docs-->>Coord: Grounded Official Docs Summary

    Coord->>Comm: AgentTool Call: CommunityAgent(request)
    Comm->>Comm: Turn 1: Formulate communityagent_lookup(query)
    Comm-->>Ledger: add(agent="CommunityAgent", call_type="model", input_tokens, output_tokens, thought_tokens=0)

    Comm->>Chroma: communityagent_lookup(query, top_k=2)
    Chroma-->>Ledger: add(agent="CommunityAgent", call_type="embedding", model="embedding", embed_tokens=N)
    Chroma-->>Comm: Top 2 Stack Overflow Threads

    Comm->>Comm: Turn 2: Summarize prior community discussion
    Comm-->>Ledger: add(agent="CommunityAgent", call_type="model", input_tokens, output_tokens, thought_tokens=0)
    Comm-->>Coord: Grounded Community Summary

    Coord->>Coord: Turn 2: Synthesize final cited answer
    Note over Coord,Ledger: _model_cb fires for Coordinator Turn 2
    Coord-->>Ledger: add(agent="Coordinator", call_type="model", input_tokens, output_tokens, thought_tokens)

    Coord-->>Runner: Final Cited Response
    Runner-->>Caller: Return Final Answer Text
```

### Why Baseline Fan-Out Is Expensive
As shown in the sequence above, **1 user query fans out into up to 6 LLM API calls + 1 Agent Search query + 1 Embedding API call**:
- **2 Coordinator LLM calls** (Turn 1 planning + Turn 2 synthesis over retrieved summaries).
- **2 DocsAgent LLM calls** (Turn 1 tool call formulation + Turn 2 extraction over retrieved docs).
- **2 CommunityAgent LLM calls** (Turn 1 tool call formulation + Turn 2 extraction over retrieved SO posts).

---

## 3. Automated LLM-as-a-Judge Quality Evaluation Workflow

After running a configuration across an evaluation dataset (`eval_dataset` or `eval_holdout`), `run_quality_eval(records, project, location)` scores every response against ground-truth reference answers.

```mermaid
flowchart LR
    subgraph Inputs["Evaluation Inputs"]
        Records["records<br/>([{'query', 'answer', 'reference'}])"]
        Rubric["CRITERIA & RATING_RUBRIC<br/>(1-5 Correctness & Completeness)"]
    end

    subgraph EvalPipeline["code_assistant/quality.py"]
        DF["Construct pandas.DataFrame<br/>(prompt, response, reference)"]
        Metric["CORRECTNESS_METRIC<br/>PointwiseMetric(question_answering_correctness)"]
        Task["Vertex AI EvalTask<br/>EvalTask(dataset=df, metrics=[CORRECTNESS_METRIC])"]
        Extract["Read summary_metrics<br/>question_answering_correctness/mean (mean5)"]
        Norm["Normalize to [0.0, 1.0]<br/>round((mean5 - 1) / 4, 4)"]
    end

    subgraph Output["Scorecard Output"]
        Result["quality = 0.9167"]
    end

    Records --> DF
    Rubric --> Metric
    DF --> Task
    Metric --> Task
    Task --> Extract
    Extract --> Norm
    Norm --> Result
```

---

## 4. Iterative FinOps Optimization Workflow

The lab guides the engineer through a disciplined Measure $\rightarrow$ Optimize $\rightarrow$ Verify loop across Parts 1–6:

```mermaid
stateDiagram-v2
    [*] --> Part1_Baseline
    Part1_Baseline: Part 1: Baseline (Pro + Flash)\nCoordinator: Pro | Sub-Agents: Flash\nthinking: HIGH | top_k: 3\nCost: ~$0.025+/q | Quality: ~0.94

    Part1_Baseline --> Part2_Tiering: Apply Lever 1 (Model Tiering)
    Part2_Tiering: Part 2: Sub-Agents -> Flash-Lite\nSub-agent thought_tokens = 0\nCoordinator now dominates total bill

    Part2_Tiering --> Part3_Caching: Analyze Lever 2 (Context Caching)
    Part3_Caching: Part 3: Context Caching\n90% discount on cached_tokens\nEvaluate break-even call volume

    Part3_Caching --> Part4_ReasoningRetrieval: Apply Lever 3 (Reasoning & Retrieval)
    Part4_ReasoningRetrieval: Part 4: thinking=LOW & top_k=2\nBounds Coordinator thought_tokens\nCuts retrieved passage volume

    Part4_ReasoningRetrieval --> Part6_FinalChallenge: Combine Levers on Held-Out Set (16 Queries)
    Part6_FinalChallenge: Part 6: Final Production Config\nCoordinator: gemini-3.5-flash (thinking=LOW)\nSub-Agents: gemini-3.1-flash-lite (top_k=2)\nCost: $0.0111/q (<= $0.0135) | Quality: 0.9167 (>= 0.90)

    Part6_FinalChallenge --> [*]: Verdict PASS & Export final_metrics.json
```

---

## 5. Graded Final Challenge Verification & Artifact Export Workflow

In Part 6, the grading workflow enforces architectural constraints, evaluates all 16 held-out queries in `assets/eval_holdout.jsonl`, verifies the budget and quality thresholds, and uploads `final_metrics.json` to the Qwiklabs grading bucket.

```mermaid
flowchart TD
    Start(["Start Part 6 Grading"]) --> CheckConfig{"Architectural Guardrails Valid?"}
    CheckConfig -->|"Coordinator == flash-lite<br/>or Embedding != gemini-embedding-001"| FailAssert["Raise AssertionError"]
    CheckConfig -->|"Valid"| BuildAgent["build_code_assistant()<br/>Coord: gemini-3.5-flash<br/>Docs/Comm: gemini-3.1-flash-lite<br/>thinking='LOW', top_k=2"]

    BuildAgent --> RunHoldout["Run 16 Held-Out Queries<br/>(assets/eval_holdout.jsonl)"]
    RunHoldout --> ComputeCost["Compute ledger.cost_per_query()<br/>Result: $0.011135 / query"]
    RunHoldout --> ComputeQuality["Run run_quality_eval()<br/>Result: 0.9167 quality"]

    ComputeCost --> CheckThresholds{"cost <= $0.0135 AND<br/>quality >= 0.90?"}
    ComputeQuality --> CheckThresholds

    CheckThresholds -->|"No"| VerdictFail["show_verdict(False, reasons)"]
    CheckThresholds -->|"Yes"| VerdictPass["show_verdict(True, [])<br/>Write final_metrics.json"]
    VerdictPass --> UploadGCS["Upload to GCS:<br/>gs://<PROJECT_ID>-metrics-result/results/final_metrics.json"]
    UploadGCS --> Done(["Lab Challenge Complete (100% Score)"])
```
