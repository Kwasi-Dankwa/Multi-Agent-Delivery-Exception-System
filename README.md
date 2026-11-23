# AI-Powered Last-Mile Delivery Exception Handling System

An end-to-end, multi-agent AI proof of concept (POC) built with **LangGraph**, **LangChain**, and **ChromaDB** to automate last-mile delivery exception triage, policy-grounded resolution, quality validation, and personalized customer communications.

---

## Business Problem

Roughly 10% of last-mile shipments encounter delivery exceptions—such as failed delivery attempts, address mismatches, damaged items, refused deliveries, and weather delays. Manual exception handling relies on human dispatchers cross-referencing noisy status logs, driver notes, customer tiers, pickup locker constraints, and internal operations manuals under tight SLAs. This leads to high operational costs, inconsistent service quality, unnecessary supervisor escalations, and preventable customer churn.

This project automates the exception-handling pipeline from raw log ingestion to validated customer outreach while enforcing deterministic safety guardrails, PII redaction, and auditable reasoning traces.

---

## System Architecture

The workflow orchestrates **six nodes** in a hybrid **Supervisor + Sequential/Iterative Review** LangGraph state machine:

```text
START ──▶ Preprocessor ──▶ Supervisor ──┬──▶ Resolution Agent ──┐
                                        ├──▶ Critic Agent ──────┼──▶ (loops back to Supervisor)
                                        ├──▶ Communication Agent┘
                                        └──▶ Finalize ──▶ END
```

| Node | Type | Responsibility |
| :--- | :--- | :--- |
| **Preprocessor** | Deterministic Python | Deduplicates repeated scan events, consolidates multi-row shipments, filters routine noise (`DELIVERED`, `SCANNED`) without LLM calls, and retrieves context via domain tools. |
| **Supervisor** | Deterministic Python | Controls workflow routing, manages Critic revision loops (`MAX_REVISIONS`), and enforces mandatory supervisor escalation triggers in code. |
| **Resolution Agent** | LLM (`gpt-5.6-luna`, `reasoning_effort="none"`) | Classifies actionable exceptions (`YES`/`NO`), selects resolution actions (`RESCHEDULE`, `REROUTE_TO_LOCKER`, `REPLACE`, `RETURN_TO_SENDER`, `N/A`), and generates step-by-step rationale grounded in retrieved playbook rules. |
| **Critic Agent** | LLM (`gpt-5.6-luna`, `reasoning_effort="medium"`) | Validates the proposed decision against event facts, locker capacity constraints, escalation rules, and playbook policies, returning `ACCEPT`, `REVISE` (with feedback), or `ESCALATE`. |
| **Communication Agent** | LLM (`gpt-5.6-luna`, `reasoning_effort="none"`) | Generates personalized `SMS` or `EMAIL` notifications with `FORMAL` or `CASUAL` tone matched to customer tier, acknowledging active credits and locker pickup details. |
| **Finalize** | Deterministic Python | Packages the structured final decision, audit trajectory, and end-to-end latency metrics. |

---

## Key Features & Guardrails

- **Noise Filtering & Multi-Event Consolidation:** Removes duplicate scan rows and merges multi-attempt shipment logs while preserving historical driver notes before invoking any LLM.
- **RAG over Operational Playbook:** Indexes `exception_resolution_playbook.pdf` into a **ChromaDB** vector store using HuggingFace embeddings (`all-MiniLM-L6-v2`) and tracks page-level citations in evaluation outputs.
- **Privacy-by-Design (Tool-Level PII Redaction):** The `lookup_customer_profile` tool strips customer names (`include_pii=False`) for the Resolution and Critic agents, exposing PII (`include_pii=True`) exclusively to the Communication Agent.
- **Hybrid Deterministic + LLM Escalation Control:** Combines a deterministic rule engine (`check_escalation_rules` with regex-based delay parsing) with a Critic review loop so mandatory supervisor escalation triggers cannot be overridden by the LLM.
- **5-Dimensional Evaluation Framework:** Evaluates every shipment across **Task Completion Rate**, **Escalation Accuracy**, **Tool Call Accuracy**, **Reasoning Trajectory Coherence** (LLM-as-a-Judge, 1–5 scale), and **End-to-End Latency**.

---

## Evaluation Results

Evaluated across 10 curated last-mile shipment scenarios (`SHP-001` through `SHP-010`) covering routine noise, duplicate scans, multi-attempt failures, locker capacity constraints, damaged perishables/fragiles, refused deliveries, and discretionary escalations:

| Metric | Score |
| :--- | :--- |
| **Task Completion Rate** | **90.0%** |
| **Escalation Accuracy (excl. N/A)** | **75.0%** |
| **Tool Call Accuracy** | **100.0%** |
| **Reasoning Trajectory Coherence (1–5)** | **3.90 / 5.00** |
| **Mean End-to-End Latency** | **8.428s** |

---

## Tech Stack

- **Orchestration & Agents:** LangGraph, LangChain, Pydantic (structured outputs)
- **Models & Embeddings:** OpenAI (`gpt-5.6-luna`), HuggingFace (`all-MiniLM-L6-v2`)
- **Vector Store & Document Parsing:** ChromaDB, PyPDF
- **Data Processing:** Python, Pandas

---

## Project Structure

```text
├── Project_Full_Code_Notebook.ipynb   # End-to-end multi-agent pipeline, tools & evaluation suite
├── customers.csv                      # Customer profiles, tiers, channels, and active credits
├── lockers.csv                        # Pickup locker locations, sizes, hours, and capacity status
├── delivery_logs.csv                  # Multi-row shipment status logs with noise and duplicate scans
├── ground_truth.csv                   # Ground truth labels and rule-by-rule justifications
└── exception_resolution_playbook.pdf  # Natural-language operations manual indexed into ChromaDB
```

---

## Setup & Usage

1. **Install dependencies:**
   ```bash
   pip install openai langchain langchain-openai langchain-text-splitters langgraph langchain-huggingface sentence-transformers langchain-chroma chromadb pypdf pandas
   ```
2. **Configure API credentials:**
   Set `OPENAI_API_KEY` (and `OPENAI_BASE_URL` if using a proxy) in your environment or Google Colab Secrets (`openai_api_key`).
3. **Run the notebook:**
   Execute `Project_Full_Code_Notebook.ipynb` sequentially to build the vector index, compile the LangGraph workflow, run all 10 test scenarios, and generate the aggregated evaluation report.
